# nifi-wiki

NiFi源码阅读总结与调优经验(2019.12-2020.4)

## 核心架构

[Flow-Based Programming](./doc/core/Flow-Based Programming.md)

[NiFi核心架构](./doc/core/NiFi核心架构.md)

## 源码解读

- FlowFile Repository

[磁盘型FlowFileRepository](./doc/sourceCode/repository/FlowFileRepository/WriteAheadLogFlowFileRepository深入解读.md)

[WriteAheadLog](./doc/sourceCode/repository/FlowFileRepository/WriteAheadLogFlowFileRepository深入解读.md)

[内存型FlowFileRepository](./doc/sourceCode/repository/FlowFileRepository/VolatileFlowFileRepository解读.md)

- Content Repository

[磁盘型ContentRepository](./doc/sourceCode/repository/ContentRepository/FileSystemRepository深入解读.md)

[内存型ContentR](./doc/sourceCode/repository/ContentRepository/VolatileContentRepository解读.md)

- FlowFile

[FlowFile的swap-in和swap-out过程](./doc/sourceCode/flowfile/FlowFile%20的%20swapIn%20和%20swapOut%20过程.md)

- Session

[ProcessSession接口](./doc/sourceCode/session/ProcessSession接口解读.md)

[ProcessSession中commit、rollback、transfer过程](./doc/sourceCode/session/StandardProcessSession关键问题解析.md)

- Verbose

[另一位神秘的I/O大户--StateProvider](./doc/sourceCode/verbose/StateProvider.md)

## Processor二次开发

[Processor正确食用指南](./doc/develop/NiFi%20Processor正确食用指南.md)

## 调优经验

[系统级调优经验总结]()

[参数级调优经验总结](./doc/tune/NiFi参数级调优经验总结.md)

## 联系作者

