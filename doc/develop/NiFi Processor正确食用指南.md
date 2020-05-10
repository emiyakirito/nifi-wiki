# NiFi Processor正确食用指南

## NiFi Processor编程模型

- 我们团队常用的多线程模型 VS NiFi多线程模型

> 1. 我们团队常用的多线程模型通过将handler与线程绑定实现无锁编程，减少或者屏蔽多线程冲突，从而提高处理性能并降低多线程开发门槛。
> 2. NiFi多线程模型采用的是“池化”思想，即在JVM运行周期内，生成且只生成一个handler对象，使用线程池调度执行该handler对象的**onTrigger**方法。
> **所以，NiFi多线程模型与我们常用的无锁多线程模型不一样**。

- Processor实现要求线程安全，其实是要求**onTrigger方法必须是线程安全的**。下面列举一些常用的线程安全方法

> 1. 使用线程安全的类，比如juc封装的线程安全类的Collection、Map等。
> 2. 按照DDD思想，强烈建议使用**无副作用**的函数方法，或者使用ThreadLocal线程本地变量。
> 3. 对于在onTrigger使用的类对象必须使用原子性引用（AtomicReference）。

- @onScheduled注解修饰的方法

> 根据我们之前的源码分析，我们知道NiFi在startProcessor方法中，使用反射机制调用Processor中@onScheduled注解修饰的方法，主要是用于Processor的一些一次性工作，比如初始化连接、初始化自己的线程池对象等。**所以，我们应该将资源的初始化工作放在@onScheduled注解修饰的方法中。**

- @onUnScheduled和@OnStopped注解修饰的方法

> 1.为什么将这两个注解放在一起说？是因为NiFi在stopProcessor方法中，会先将线程池shutdown，注意不是shutdownNow，使用shutdown一方面是为了优雅的等待执行中以及在排队中的Processor::onTrigger方法的完成，另一方面是为了保证onTrigger方法执行的完整性。

> 2.NiFi依然是使用反射机制首先调用Processor中@onUnScheduled注解修饰的方法，这个方法从名字上看是和@onScheduled相对的，所以在这个方法中我们应该去释放在@onScheduled申请的资源，比如关闭数据库连接、停止工作线程、关闭线程池等等。

> 3.那么为什么又会有@onStopped注解呢？因为有些Processor的停止工作可能要分成两阶段来做。通过源码分析我们可知，@onUnScheduled方法执行后，所有的activeThread都会被停止，保证当前只有一个用于控制Processor状态的monitorThread，我们可以将一些需要在@onUnScheduled之后做的工作放在该方法中。

- @onShutdown

> 1.这个注解本质上就是shutdownHook，会依次执行@onUnScheduled和@OnStopped注解修饰的方法，但是需要注意的是这个方法是quietly执行的，也就是不会输出任何日志信息，不过可以用标准输出流打印日志信息，因为标准输出流最终会被重定向到nifi-bootstrap.log日志中。

> 2.**千万不要特别依赖这个方法的执行**，因为当NiFishutdown超时或者被kill -9杀掉时，这个方法是会中断或者不会被执行的，所以这个方法中不要放强逻辑的代码。

## 编写高性能的NiFi Processor

- 编写高性能NiFi Processor之前，首先一定要保证功能的正确性，所以请一定在onTrigger方法中使用线程安全的类、方法与对象。

- 高性能的评判标准：**Processor单线程每秒处理 1w FlowFile对象**

> 通过一段时间生产系统中的使用，我们发现通过增加Queue的大小来解决性能问题是一种反模式。一方面，并没有真正解决Processor性能不足的问题；另一方面，堆积大量的FlowFile对象会造成频繁的FGC，影响稳定性。所以，官方建议Queue的大小为1w，交换阈值为2w，是很有道理的。

- 不是所有的逻辑都必须放在onTrigger方法中，因为如果所有逻辑都放在onTrigger方法中，那么我们的Processor永远只能串行工作，高性能无从谈起。所以，我们需要对Processor处理的问题进行解耦，分离出哪些是必须放在onTrigger方法中做的，哪些是可以脱离NiFi框架做的事情。
判定哪些必须放在onTrigger方法中，哪些不需要放在onTrigger方法中的一个最基本的原则就是：**需不需要依赖ProcessSession对象**

- 谨慎的使用锁，当我们在实现中感觉必须使用锁时，想一想能不能采用其他设计模式规避加锁。