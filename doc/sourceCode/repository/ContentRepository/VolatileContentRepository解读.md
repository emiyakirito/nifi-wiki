# VolatileContentRepository解读

- 内存型ContentRepository，FlowFile的content全部写入堆内存，写入总大小通过“nifi.volatile.content.repository.max.size”控制，同时可通过“nifi.volatile.content.repository.block.size”定义每个block的大小。maxSize默认大小为100MB，blockSize默认大小为32kb。
  
    ```Java
    public static final int DEFAULT_BLOCK_SIZE_KB = 32;
    maxBytes = (long) DataUnit.B.convert(100D, DataUnit.MB);

    public static final String MAX_SIZE_PROPERTY = "nifi.volatile.content.repository.max.size";
    public static final String BLOCK_SIZE_PROPERTY = "nifi.volatile.content.repository.block.size";
    ```

- VolatileContentRepository将block管理委派给MemoryManager，MemoryManager将maxSize划分成maxSize/blockSize个blockSize大小的字节数组，并使用阻塞队列BlockingQueue进行管理。默认为100 * 1024 / 32 = 3200个blocks。

    ``` Java
    public MemoryManager(final long totalSize, final int blockSize) {
        this.blockSize = blockSize;

        final int numBlocks = (int) (totalSize / blockSize);
        queue = new LinkedBlockingQueue<>(numBlocks);

        for (int i = 0; i < numBlocks; i++) {
            queue.offer(new byte[blockSize]);
        }
    }
    ```

- 当总内存大小超过maxSize时，VolatileContentRepository提供接口设置backupRepository，但是NiFi框架里并没有提供相关参数给用户设置，所以形同虚设。

    ```Java
    /**
     * Specifies a Backup Repository where data should be written if this
     * Repository fills up
     *
     * @param backup repo backup
     */
    public void setBackupRepository(final ContentRepository backup) {
        final boolean updated = backupRepositoryRef.compareAndSet(null, backup);
        if (!updated) {
            throw new IllegalStateException("Cannot change BackupRepository after it has already been set");
        }
    }
    ```

- VolatileContentRepository抽象出ContentBlock对象用于管理内存中的content。由于block是按照blockSize分桶的，所以ContentBlock对象通过ClaimSwitchingOutputStream实现内容数组的**跨桶追加写**。
- 其中跨桶追加写是通过ArrayManagedOutputStream::write实现的。我们具体来分析下源码。

    ``` Java
    public void write(final byte[] b, int off, final int len) throws IOException
    ```

    1. 计算当前桶能够写下待写入的content。如果可以写下，则往桶内追加写。

    ``` Java
        final int bytesFreeThisBlock = currentBlock == null ? 0 : currentBlock.length - currentIndex;
        if (bytesFreeThisBlock >= len) {
            System.arraycopy(b, off, currentBlock, currentIndex, len);
            currentIndex += len;
            curSize += len;

            return;
        }
    ```

    2. 若当前桶写不下，则首先计算所需的桶的个数。不能整除的取上整。

    ``` Java
        // Try to get all of the blocks needed
        final long bytesNeeded = len - bytesFreeThisBlock;
        int blocksNeeded = (int) (bytesNeeded / memoryManager.getBlockSize());
        if (blocksNeeded * memoryManager.getBlockSize() < bytesNeeded) {
            blocksNeeded++;
        }
    ```

    3. 从MemoryManager中取出所需要的的所有桶。

    ``` Java
        // get all of the blocks that we need
        final List<byte[]> newBlocks = new ArrayList<>(blocksNeeded);
        for (int i = 0; i < blocksNeeded; i++) {
            final byte[] newBlock = memoryManager.checkOut();
            if (newBlock == null) {
                memoryManager.checkIn(newBlocks);
                throw new IOException("No space left in Content Repository");
            }

            newBlocks.add(newBlock);
        }
    ```

    4. 采用**覆盖写**的方式，优先写满当前桶，并重置currentBlock和currentIndex。写满后继续写下一个桶，直至本次待写入内容全部写完，置currentBlock为当前最后一个写入的桶，currentIndex为当前桶已写入字节数，以便于下一次追加写。

    ``` Java
        // we've successfully obtained the blocks needed. Copy the data.
        // first copy what we can to the current block
        long bytesCopied = 0;
        final int bytesForCur = currentBlock == null ? 0 : currentBlock.length - currentIndex;
        if (bytesForCur > 0) {
            System.arraycopy(b, off, currentBlock, currentIndex, bytesForCur);

            off += bytesForCur;
            bytesCopied += bytesForCur;
            currentBlock = null;
            currentIndex = 0;
        }

        // then copy to all new blocks
        for (final byte[] block : newBlocks) {
            final int bytesToCopy = (int) Math.min(len - bytesCopied, block.length);
            System.arraycopy(b, off, block, 0, bytesToCopy);
            currentIndex = bytesToCopy;
            currentBlock = block;
            off += bytesToCopy;
            bytesCopied += bytesToCopy;
        }
    ```

- 当写入过程中发生任何异常，则会调用redirect操作，即将内容写入backupRepository中，但由于实际使用中并没有，所以写满后会一直抛出“Content Repository out of space” 异常。

- 循环写。ContentRepository每次写入ContentClaim都是无状态的，因为每次创建ContentClaim时都会重置ContentClaim对应的ContentBlock中的当前桶。

    ``` Java
        final ContentBlock content = getContent(claim);
        content.reset();

        ArrayManagedOutputStream::reset -> ArrayManagedOutputStream::destroy

        public void destroy() {
        writeLock.lock();
        try {
            memoryManager.checkIn(blocks);
            blocks.clear();
            currentBlock = null;
            currentIndex = 0;
            curSize = 0L;
        } finally {
            writeLock.unlock();
        }
    }
    ```

## 与FileSystemRepository对比

### 优势

- 全内存型ContentRepository，不产生文件IO（即使是内存顺序IO），也不占用虚盘空间。

### 风险点

- 通过测试与代码分析可知，VolatileContentRepository是有fatal级别的bug，民间从2017年首次发现至今（1.11.4版本）一直没有修复，猜测是使用的人非常少，也没有人提交此bug，所以掩盖了此问题。

> 当每次写入的内容都只需要一个block时，写入正常，而下一次再写入时，则生成了一个新的ContentBlock对象，持有一个新的ClaimSwitchingOutputStream对象，调用ArrayManagedOutputStream::write方法是永远都不会继续追加，而是重新选一个block继续写，最终造成还没有达到maxSize，就将block耗尽的缺陷。

> 我们用测试用例来证明。100MB可以分成51200个2KB块，所以当循环至51201时，发生“java.io.IOException: Content Repository is out of space”异常。

   ``` Java
    @Test
    public void test() throws IOException {
        System.setProperty(NiFiProperties.PROPERTIES_FILE_PATH, TestVolatileContentRepository.class.getResource("/conf/nifi.properties").getFile());
        final Map<String, String> addProps = new HashMap<>();
        addProps.put(VolatileContentRepository.BLOCK_SIZE_PROPERTY, "2 KB");
        final NiFiProperties nifiProps = NiFiProperties.createBasicNiFiProperties(null, addProps);
        final VolatileContentRepository contentRepo = new VolatileContentRepository(nifiProps);
        contentRepo.initialize(claimManager);
        // can write 100 * 1024 /1 = 102400, but after 51201, blocks exhausted
        for (int idx =0; idx < 51201; ++idx) {
            final ContentClaim claim = contentRepo.create(true);
            try (final OutputStream out = contentRepo.write(claim)){
                final byte[] oneK = new byte[1024];
                Arrays.fill(oneK, (byte) 55);

                out.write(oneK);
            }
        }
    }
   ```

- 目前该问题也已经由本人连同VolatileFlowFileRepository问题一并提交给NiFi官方，**修复时间未知**。

- 需要控制图中带Content的FlowFile数量，一旦超出maxSize则会发生OOM，影响运行稳定性。

### 总结

- 优势并没有风险点突出，而且存在明显bug，所以不建议切换成VolatileContentRepository。如果想减少ContentRepository的IO操作，建议仍然使用FileSystemRepository，同时将ContentRepository目录放在虚盘中。

