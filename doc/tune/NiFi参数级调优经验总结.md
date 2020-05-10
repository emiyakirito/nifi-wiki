# NiFi 参数级调优经验总结

## 理性分析并看待NiFi劣势

- 分布式系统永远离不开CAP原理的讨论，从CAP角度分析NiFi系统，我们很明显的看出NiFi是一个CP系统，通过事务的方式保证数据一致性、完整性。当系统过载时，NiFi在可用性方面做了一定妥协。
- 在预处理场景中，我们期待的框架是AP系统，因为我们要保证系统的高可用性，即使过载时也不应该造成数据积压。同时，预处理场景对数据完整性做了一定妥协，从流式处理原语的角度来说，预处理框架实现的是at-most-once。
- 结合工程实践与源码分析，我们知道NiFi最大的性能瓶颈在于繁重的磁盘IO。NiFi的主要磁盘IO操作，我们列举如下：

>1. StateProvider使用WAL记录componet组件（Processor）状态变化。
>2. FlowFileRepository使用WAL记录所有FlowFile的增、删、改动作journal日志。
>3. FlowFileRepository定期checkpoint。
>4. FlowFileRepository保存Queue过载时swapout到磁盘上的FlowFile序列化对象。
>5. ContentRepository物化FlowFile的content。
>6. ContentRepository启动时扫描所有文件确定最早老化时间、定期将老化的FlowFile的content装箱、打包/清除。
>7. ProvenanceRepository记录FlowFile的血统、事件等。
>8. ProvenanceRepository使用lucene对FlowFile相关字段做索引，提供查询。

- 所以，我们的优化目标是将NiFi从CP系统尽量改造为AP系统，以满足我们预处理场景**无状态**的需求。除了操作系统、docker镜像这些物理层面的优化以及优化我们自定义Processor外，我们优化的策略还包括：（1）关闭不必要的IO；（2）减少或者用其他性能开销更小的方式替代必要的IO。

## 关闭不需要的IO

- 第二条与第三条（**存在BUG，暂时无法改进**）。这部分的IO在预处理场景下是没有必要的，我们使用VolatileFlowFileRepository，一种内存型FlowFileRepository实现，完全关闭写WAL的journal日志以及checkpoint。减少运行期间的IO消耗（即使是内存顺序IO），且重启时也不需要去加载日志恢复状态，**可提高约300%的启动速度。**
- 第六条。我们关闭了FlowFile的content打包功能，采用直接标记清除的策略。
- 第七条与第八条。我们使用VolatileProvenanceRepository，一种内存型ProvenanceRepository实现，完全关闭写FlowFile血统事件与做索引，同时将buffer_size缩减为1，减少内存消耗。

## 减少或替代必要的IO操作

- 第一条。通过线上系统发现，这部分的IO数据量开销较小，只要我们不要人为的经常去改变Processor的状态（包括启动、停止、修改属性等），基本不会触发journal日志，checkpoint文件基本也为空。
- 第四条。这部分的IO操作是没法完全避免的。在NiFi过载时，会出现大量的FlowFile交换，但是我们将FlowFileRepository目录定义在虚盘，保证内存顺序IO。NiFi处理稳定时不会产生交换操作。所以这方面的优化需要从两个层面展开：（1）提高交换IO速率；（2）提高Processor处理性能。
- 第五条。经过测算，采用文件rename的性能大约比NiFi的importFrom方式高一个数量级。所以，我们FlowFile对象的content都不写入ContentRepository中，文件大小不可控制的情况下，根据FlowFile自定义属性filepath获取文件全路径进行本地文件IO操作；文件大小可控的情况下，直接写入FlowFile的属性中（比如索引入库Processor）。
