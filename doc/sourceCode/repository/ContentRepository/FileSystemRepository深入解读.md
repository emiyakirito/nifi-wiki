# FileSystemRepository深入解读

## 特点

- FileSystemRepository是ContentRepository接口基于本地文件系统的一种实现。将FlowFile的content采用拼块的方式存储在不同的partition中，支持content的写入、合并、标记清除，不支持更新操作（更新操作通过写入新内容实现）。
- 对于老化content，支持标记清除与打包两种清理策略。
- 采用顺序写方式，提高IO性能。

## 初始化

- 使用“nifi.content.claim.max.flow.files”配置项初始化可写入ContentClaim的FileOutputStream阻塞队列大小，表示可同时并发写入的FileOutputStream流数量。

    ``` Java
    // Queue for claims that are kept open for writing. Ideally, this will be at
    // least as large as the number of threads that will be updating the repository simultaneously but we don't want
    // to get too large because it will hold open up to this many FileOutputStreams.
    // The queue is used to determine which claim to write to and then the corresponding Map can be used to obtain
    // the OutputStream that we can use for writing to the claim.
    private final BlockingQueue<ClaimLengthPair> writableClaimQueue;

    this.maxFlowFilesPerClaim = nifiProperties.getMaxFlowFilesPerClaim();
    this.writableClaimQueue  = new LinkedBlockingQueue<>(maxFlowFilesPerClaim);
    ```

- 采用container-section结构，nifi.properties配置文件中以"nifi.content.repository.directory."开头的目录看作是container，container中包含多个子目录（默认1024），看作是section。

- 根据nifi.properties中"nifi.content.repository.archive.enabled"决定是否启用打包。"nifi.content.repository.archive.max.retention.period"决定打包文件过期时间（默认12小时）。"nifi.content.repository.archive.max.usage.percentage"决定archive文件最大占用空间（默认50%）。**NiFi启动时，需要重新扫描section中所有打包文件，找出其中最早时间戳，从而继续计算老化时间。由于需要扫描container中每个文件，所以IO开销较大，NiFi使用异步线程实现此功能，找到最早老化时间后关闭该线程池。**

- "nifi.content.repository.always.sync"配置项决定是否每次content更新就强制flush到磁盘。

- 启动周期性调度（默认1s）的清理类线程。
    1. BinDestructableClaims单线程负责将全局待清理的ContentClaim对象按照container分别装入对应的阻塞队列中，超时装入时间为10分钟，如果10分钟都无法将待清理对象加入到队列中，则说明清理速率远远小于待清理对象生成速率，待清理对象会在NiFi下次重启时被重新清理。
    2. 每个container都会对应一个ArchiveOrDestroyDestructableClaims线程，其负责清理/打包对应container的阻塞队列中的ContentClaim对象。
    3. 每个container都会对应一个DestroyExpiredArchiveClaims线程，其负责清理对应container中已过期的archive文件。

## 创建ContentClaim对象

- 在FlowFileRepository中我们已经介绍过了ResourceClaim与ContentClaim之间的关系。
- 取出可用的ResourceClaim对象。阻塞队列writableClaimQueue记录了当前所有ClaimLengthPair对象，ClaimLengthPair对象中保存了ResourceClaim对象及其可用大小Length。
- 无可用ResourceClaim对象时，首先创建，创建完成后才会放入到writableClaimStreams中，保证不会有多个线程同时写入一个ContentClaim对象。
- 如果有可用ResourceClaim对象，则取出并增加引用计数。

    ``` Java
    @Override
    public ContentClaim create(final boolean lossTolerant) throws IOException {
        ResourceClaim resourceClaim;

        final long resourceOffset;
        final ClaimLengthPair pair = writableClaimQueue.poll();
        if (pair == null) {
            final long currentIndex = index.incrementAndGet();

            String containerName = null;
            boolean waitRequired = true;
            ContainerState containerState = null;
            for (long containerIndex = currentIndex; containerIndex < currentIndex + containers.size(); containerIndex++) {
                final long modulatedContainerIndex = containerIndex % containers.size();
                containerName = containerNames.get((int) modulatedContainerIndex);

                containerState = containerStateMap.get(containerName);
                if (!containerState.isWaitRequired()) {
                    waitRequired = false;
                    break;
                }
            }

            if (waitRequired) {
                containerState.waitForArchiveExpiration();
            }

            final long modulatedSectionIndex = currentIndex % SECTIONS_PER_CONTAINER;
            final String section = String.valueOf(modulatedSectionIndex).intern();
            final String claimId = System.currentTimeMillis() + "-" + currentIndex;

            resourceClaim = resourceClaimManager.newResourceClaim(containerName, section, claimId, lossTolerant, true);
            resourceOffset = 0L;
            LOG.debug("Creating new Resource Claim {}", resourceClaim);

            // we always append because there may be another ContentClaim using the same resource claim.
            // However, we know that we will never write to the same claim from two different threads
            // at the same time because we will call create() to get the claim before we write to it,
            // and when we call create(), it will remove it from the Queue, which means that no other
            // thread will get the same Claim until we've finished writing to it.
            final File file = getPath(resourceClaim).toFile();
            ByteCountingOutputStream claimStream = new SynchronizedByteCountingOutputStream(new FileOutputStream(file, true), file.length());
            writableClaimStreams.put(resourceClaim, claimStream);

            incrementClaimantCount(resourceClaim, true);
        } else {
            resourceClaim = pair.getClaim();
            resourceOffset = pair.getLength();
            LOG.debug("Reusing Resource Claim {}", resourceClaim);

            incrementClaimantCount(resourceClaim, false);
        }

        final StandardContentClaim scc = new StandardContentClaim(resourceClaim, resourceOffset);
        return scc;
    }

    ```

## 删除ContentClaim对象

- 删除ContentClaim对象实际上是删除ContentClaim对象所在的ResourceClaim对象。

    ``` Java
    @Override
    public boolean remove(final ContentClaim claim) {
        if (claim == null) {
            return false;
        }

        return remove(claim.getResourceClaim());
    }
    ```

- 如果ContentClaim对象为null/ContentClaim对象无所在的ResourceClaim对象/所在ResourceClaim对象正在被使用中，均删除失败

    ``` Java
    private boolean remove(final ResourceClaim claim) {
        if (claim == null) {
            return false;
        }

        // If the claim is still in use, we won't remove it.
        if (claim.isInUse()) {
            return false;
        }
        ...
    }
    ```

- 找到ResourceClaim对应的实际物理文件，关闭ResourceClaim对应的OutputStream，最终删除对应的实际物理文件。过程中若发生任何IO异常，均删除失败。**如果确实删除失败，只有等background清理线程清除。**

    ``` Java
    private boolean remove(final ResourceClaim claim) {
        ...
        Path path = null;
        try {
            path = getPath(claim);
        } catch (final ContentNotFoundException cnfe) {
        }

        // Ensure that we have no writable claim streams for this resource claim
        final ByteCountingOutputStream bcos = writableClaimStreams.remove(claim);

        if (bcos != null) {
            try {
                bcos.close();
            } catch (final IOException e) {
                LOG.warn("Failed to close Output Stream for {} due to {}", claim, e);
            }
        }

        final File file = path.toFile();
        if (!file.delete() && file.exists()) {
            LOG.warn("Unable to delete {} at path {}", new Object[]{claim, path});
            return false;
        }

        return true;
    }
    ```

## 合并多个ContentClaim对象

- 在合并过程中，首先会写入header（"HEADER"），在依次写入每个ContentClaim对象对应的字节，每个ContentClaim对象之间使用分隔符（"DEMARCATOR"）分割，最后写入footer（"FOOTER"）。

    ``` Java
    @Override
    public long merge(final Collection<ContentClaim> claims, final ContentClaim destination, final byte[] header, final byte[] footer, final byte[] demarcator) throws IOException {
        if (claims.contains(destination)) {
            throw new IllegalArgumentException("destination cannot be within claims");
        }

        try (final ByteCountingOutputStream out = new ByteCountingOutputStream(write(destination))) {
            if (header != null) {
                out.write(header);
            }

            int i = 0;
            for (final ContentClaim claim : claims) {
                try (final InputStream in = read(claim)) {
                    StreamUtils.copy(in, out);
                }

                if (++i < claims.size() && demarcator != null) {
                    out.write(demarcator);
                }
            }

            if (footer != null) {
                out.write(footer);
            }

            return out.getBytesWritten();
        }
    }
    ```

## importFrom过程

- importFrom过程其实是将外部文件内容采用拼块的方式追加到到ContentClaim对象中。

    ```Java
    @Override
    public long importFrom(final Path content, final ContentClaim claim) throws IOException {
        try (final InputStream in = Files.newInputStream(content, StandardOpenOption.READ)) {
            return importFrom(in, claim);
        }
    }

    @Override
    public long importFrom(final InputStream content, final ContentClaim claim) throws IOException {
        try (final OutputStream out = write(claim, false)) {
            return StreamUtils.copy(content, out);
        }
    }
    ```

## 读取ContentClaim

- 找到ContentClaim对应的文件
- 根据ContentClaim的起始offset和长度length，从对应的文件中将这部分的字节读取出来。

## 写入ContentClaim

- 找到ContentClaim对应的ResourceClaim的OutputStream
- 将ContentClaim内容写入到对应的OutputStream流中。
- 当写入的ContentClaim长度超过maxAppendableClaimLength（默认1MB），首先需要标记这个ContentClaim已经不可再写，并将其从writableClaimQueue队列中移除，并关闭对应的OutputStream。

    ``` Java
        // if we've not yet hit the threshold for appending to a resource claim, add the claim
        // to the writableClaimQueue so that the Resource Claim can be used again when create()
        // is called. In this case, we don't have to actually close the file stream. Instead, we
        // can just add it onto the queue and continue to use it for the next content claim.
        final long resourceClaimLength = scc.getOffset() + scc.getLength();
        if (recycle && resourceClaimLength < maxAppendableClaimLength) {
            final ClaimLengthPair pair = new ClaimLengthPair(scc.getResourceClaim(), resourceClaimLength);

            // We are checking that writableClaimStreams contains the resource claim as a key, as a sanity check.
            // It should always be there. However, we have encountered a bug before where we archived content before
            // we should have. As a result, the Resource Claim and the associated OutputStream were removed from the
            // writableClaimStreams map, and this caused a NullPointerException. Worse, the call here to
            // writableClaimQueue.offer() means that the ResourceClaim was then reused, which resulted in an endless
            // loop of NullPointerException's being thrown. As a result, we simply ensure that the Resource Claim does
            // in fact have an OutputStream associated with it before adding it back to the writableClaimQueue.
            final boolean enqueued = writableClaimStreams.get(scc.getResourceClaim()) != null && writableClaimQueue.offer(pair);

            if (enqueued) {
                LOG.debug("Claim length less than max; Adding {} back to Writable Claim Queue", this);
            } else {
                writableClaimStreams.remove(scc.getResourceClaim());
                resourceClaimManager.freeze(scc.getResourceClaim());

                bcos.close();

                LOG.debug("Claim length less than max; Closing {} because could not add back to queue", this);
                if (LOG.isTraceEnabled()) {
                    LOG.trace("Stack trace: ", new RuntimeException("Stack Trace for closing " + this));
                }
            }
        } else {
            // we've reached the limit for this claim. Don't add it back to our queue.
            // Instead, just remove it and move on.

            // Mark the claim as no longer being able to be written to
            resourceClaimManager.freeze(scc.getResourceClaim());

            // ensure that the claim is no longer on the queue
            writableClaimQueue.remove(new ClaimLengthPair(scc.getResourceClaim(), resourceClaimLength));

            bcos.close();
            LOG.debug("Claim lenth >= max; Closing {}", this);
            if (LOG.isTraceEnabled()) {
                LOG.trace("Stack trace: ", new RuntimeException("Stack Trace for closing " + this));
            }
        }
    ```

## 清除purge

- 将container中所有section子目录及对应的文件全部删除
- 并循环10次检查所有container目录是否可写入，防止其他线程锁定该目录

## shutdown

- 关闭所有清理类线程
- 关闭所有打开中的OutputStream对象。考虑到shutdown时，依然会有线程将content写入到对应的OutputStream流中，写入失败时，对应的ProcessSession会rollback写入过程，没有写入成功的FlowFile对象会被记录在WAL的journal日志中，当NiFi重启时，关闭前未正确写入的content会被重新写入，保证数据完整性。
