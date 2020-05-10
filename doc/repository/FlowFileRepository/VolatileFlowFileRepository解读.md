# VolatileFlowFileRepository解读

VolatileFlowFileRepository是内存型FlowFileRepository，实现方式非常简单，父类FlowFileRepository中申明的方法，只有updateRepository方法会将DELETE类型的FlowFile标记清除，而swapFlowFilesIn、swapFlowFilesOut方法均为空实现。

## 与WriteAheadLogFlowFileRepository的对比

### 优势

- 全内存型，运行期间不写journal日志，也不会定期checkpoint。在预处理场景下，对于某特定类别数据，2分钟checkpoint期间会产生4.1GB左右的journal日志文件，关闭WAL可减少大量IO操作（即使是内存顺序IO）与虚盘空间（占用虚盘空间依然算入docker镜像占用memory）。

- 重启后，内存中所有FlowFile对象均丢失，包括运行期间因Queue过载而被交换出去的FlowFile，不需要去加载WAL（包括checkpoint与journal日志），加速启动过程（无WAL启动时间大约20s，加载WAL完成启动时间大约80s，WAL日志越大启动越慢）。可满足**无状态**预处理框架需求。

### 存在的风险

- 需要控制NiFi的图复杂度，建议使用默认Queue大小（背压1w，swap2w）。

> 经过测试发现，一个拥有64个属性的FlowFile对象占用内存大小为800字节。目前NiFi的最大堆是8g，足够容纳10^9个FlowFile对象。

> 在最大堆为8g的情况下，经测试发现内存中的FlowFile个数小于100w时，不会出现FGC。我们以某特定类别数据的图复杂度为例，目前图中共有29个Queue，全部过载的情况下，每个Queue中最多存放2w（Queue最大size）+1w（swap queue）=3w个FlowFile对象（正常背压1w的情况下，每个Queue不会超过1w个FlowFile对象），不会超过100w。

- 被交换出去的FlowFile对象，一旦重启就永远不再会被处理，所以需要有清理机制。有以下两种解决方案：

> 提高Processor处理性能，不产生swap文件。即使因某时数据量太大过载而产生swap文件，在正常运行期间也能消化掉数据，从而尽量保证NiFi镜像重启时，不会残留swap文件。

> 可以在NiFi基础镜像运行脚本中增加清理机制，在镜像重启时首先清理FlowFileRepository目录下可能存在的swap文件。

### 切换到VolatileFlowFileRepository所需工作

- NiFi基础镜像中的nifi.sh启动脚本中增加清理FlowFileRepository目录功能。

### **特别注意**
截止NiFi-1.11.4版本，VolatileFlowFileRepository存在fatal级别的bug，当Processor过载将FlowFile对象swap out后，即使Queue已空，也无法将磁盘上的FlowFile对象swap in，且Queue在UI上显示一直为满，造成上游Processor一直背压阻塞，整条流无法正常工作。

具体可参见本人提交的Bug Report
> <https://lists.apache.org/list.html?users@nifi.apache.org:2020-4> 中 "
Bug reports about working with VolatileContentRepository and VolatileFlowFileRepository"

> <https://issues.apache.org/jira/projects/NIFI/issues/NIFI-7388?filter=allissues>
