# StateProvider解读

## 为什么研究

通过观察线上日志发现，NiFi每隔两分钟都会出现如下日志，将component状态日志物化在本地。

> 2020-04-04 10:16:09,075 INFO [Write-Ahead Local State Provider Maintenance] org.wali.MinimalLockingWriteAheadLog org.wali.MinimalLockingWriteAheadLog@242a3997 checkpointed with 2 Records and 0 Swap Files in 20 milliseconds (Stop-the-world time = 9 milliseconds, Clear Edit Logs time = 7 millis), max Transaction ID 5

如此频繁写磁盘，如果数据量大的话，一定会对性能造成很大影响。所以我们需要研究StateProvider具体功能是什么，是否可以少写甚至不写WAL，从而提升NiFi的整体性能。

## 研究范围

- StateProvider采用依赖注入的方式，将接口的实现定义在nifi.properties文件中，构造函数参数定义在state-management.xml文件中。

``` properties
####################
# State Management #
####################
nifi.state.management.configuration.file=./conf/state-management.xml
# The ID of the local state provider
nifi.state.management.provider.local=local-provider
# The ID of the cluster-wide state provider. This will be ignored if NiFi is not clustered but must be populated if running in a cluster.
nifi.state.management.provider.cluster=zk-provider
# Specifies whether or not this instance of NiFi should run an embedded ZooKeeper server
nifi.state.management.embedded.zookeeper.start=false
# Properties file that provides the ZooKeeper properties to use if <nifi.state.management.embedded.zookeeper.start> is set to true
nifi.state.management.embedded.zookeeper.properties=./conf/zookeeper.properties
```

``` xml
    <local-provider>
        <id>local-provider</id>
        <class>org.apache.nifi.controller.state.providers.local.WriteAheadLocalStateProvider</class>
        <property name="Directory">./state/local</property>
        <property name="Always Sync">false</property>
        <property name="Partitions">16</property>
        <property name="Checkpoint Interval">2 mins</property>
    </local-provider>

    <cluster-provider>
        <id>zk-provider</id>
        <class>org.apache.nifi.controller.state.providers.zookeeper.ZooKeeperStateProvider</class>
        <property name="Connect String"></property>
        <property name="Root Node">/nifi</property>
        <property name="Session Timeout">10 seconds</property>
        <property name="Access Control">Open</property>
    </cluster-provider>

    <cluster-provider>
        <id>redis-provider</id>
        <class>org.apache.nifi.redis.state.RedisStateProvider</class>
        <property name="Redis Mode">Standalone</property>
        <property name="Connection String">localhost:6379</property>
    </cluster-provider>
```

StateProvider接口目前有三种实现方式：

- 单机模式
1.存储在本地（默认./state/local目录）
2.默认实现：org.apache.nifi.controller.state.providers.local.WriteAheadLocalStateProvider

- 集群模式
1.基于Zookeeper的默认实现：org.apache.nifi.controller.state.providers.zookeeper.ZooKeeperStateProvider
2.基于redis实现：org.apache.nifi.redis.state.RedisStateProvider

考虑到我们主要面向预处理场景，使用的是单机模式，所以我们只关注StateProvider的local模式实现。

## WriteAheadLocalStateProvider 功能

- StateProvider使用WAL机制存储与恢复NiFi component（Processor就是一种component）的状态，所谓状态，可以是processor处理时长、processor中定义的PropertyDescriptor等等。默认使用已经废弃的org.wali.MinimalLockingWriteAheadLog，官方依然保留该实现主要是考虑到在某些场景下（比如存储恢复并发量并不大的componet状态），该实现足够轻量级。

- 我们在日志中看到其实是WriteAheadLocalStateProvider中定义的一个每2分钟调度一次的定时checkpoint任务。

``` Java
static final PropertyDescriptor CHECKPOINT_INTERVAL = new PropertyDescriptor.Builder()
        .name("Checkpoint Interval")
        .description("The amount of time between checkpoints.")
        .addValidator(StandardValidators.TIME_PERIOD_VALIDATOR)
        .defaultValue("2 mins")
        .required(true)
        .build();

long checkpointIntervalMillis = context.getProperty(CHECKPOINT_INTERVAL).asTimePeriod(TimeUnit.MILLISECONDS);

executor.scheduleWithFixedDelay(new CheckpointTask(), checkpointIntervalMillis, checkpointIntervalMillis, TimeUnit.MILLISECONDS);

private class CheckpointTask implements Runnable {
        @Override
        public void run() {
            try {
                logger.debug("Checkpointing Write-Ahead Log used to store components' state");

                writeAheadLog.checkpoint();
            } catch (final IOException e) {
                logger.error("Failed to checkpoint Write-Ahead Log used to store components' state", e);
            }
        }
    }
```

## 是否可优化

1. StateProvider保存的是当前运行期间NiFi Processor的状态，且重启时通过WAL恢复。考虑到目前处理场景为预处理，预处理最大的特点是**无状态**，且NiFi为容器化部署，完全可以考虑关闭此特性。但是，NiFi框架中并没有提供内存型StateProvider的实现，也没有开关可关闭。

2. 根据生产环境可知，state目录占比大小在100kb左右，虽然checkpoint周期较短，但是Processor无状态变更时，基本不会产生journal日志与周期性的checkpoint日志。

3. 所以综合上述因素，StateProvider对于性能的影响并不大，目前可不作为调优选项，后续可考虑自研内存型StateProvider，不再物化到磁盘。
