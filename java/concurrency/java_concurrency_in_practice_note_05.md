# 任务执行
大多数并发应用程序都是围绕“任务执行”来构造的。通过把应用程序的工作分解到多个任务中，可以简化程序的组织结构。

## 在线程中执行任务
在生产环境中，为提高性能，可能会为每个任务都分配一个线程，不过这种方法存在一些缺陷，尤其是需要创建大量线程的时候：
- 线程生命周期的开销非常高。线程的创建与销毁并不是没有代价的，如果请求量非常大且请求的处理过程都是轻量级的，那么为每个请求创建一个新线程将消耗大量的计算资源。
- 资源消耗。活跃的线程会消耗系统资源，尤其是内存。如果可运行的线程数量多于可用处理器的数据，那么有些线程将闲置，大量空闲的线程会占用大量内存，给垃圾回收器带来压力。
- 稳定性。在可创建线程的数量上存在一个限制。这个限制随不同平台的不同而不同，并受 JVM 的启动参数，Thread 构造函数中请求的栈大小以及底层系统对线程的限制等制约。

在一定范围内，增加线程可提高系统的吞吐率，但如果超出这个范围，再创建更多的线程只会降低程序的执行速度。

## Executor 框架
串行执行会产生糟糕的响应和吞吐，而为每个任务都分配一个线程会带来资源管理的复杂性，且可能带来不利影响。通过线程池可以避免这两方面的问题。线程池简化了线程的管理工作，并且 `java.util.concurrent` 提供了一种灵活的线程池实现作为 Executore 框架的一部分。在 Java 类库，任务执行的主体不是 Thread，而是 Executore，其接口如下所示：
```
public interface Executore {
    void execute(Runnable command);
}
```
Executor 基于生产者-消费者模式，提交任务的操作相当于生产者，执行任务的线程相当于消费者。如果要在程序中实现一个生产者-消费者的设计，那么最简单的方式通常就是使用 Executor。

每当看到如下形式的代码时：
```
new Thread(runnable).start();
```
并且你希望获得一种更灵活的执行策略时，请考虑使用 Executor 来代替Thread。

### 线程池
从字面含义来看，线程池是指管理一组同构工作线程的资源池。在线程池中执行任务比为每个任务分配一个线程优势更多。通过重用现有的线程而不是创建新线程，可以在处理多个请求时分摊在线程创建和销毁过程中产生的巨大开销。另外一个额外好处是，当请求到达时，工作线程通常已经存在，不会由于等待创建线程而延迟任务的执行，从而提高了响应性。通过适当调整线程池的大小，可以创建足够多的线程以便使处理器保持忙碌状态，同时还可以防止过多线程相互竞争资源而使应用程序耗尽内存或失败。

Java 类库提供了如下静态工厂方法来创建一个线程池：
- `newFiexedThreadPool`。创建一个固定长度的线程池。
- `newCachedThreadPool`。创建一个可缓存的线程池，如果线程池的当前规模超过了处理需求时，那么将回收空闲的线程，而当需求增加时，则可以添加新的线程，线程池的规模不存在任何限制。
- `newSingleThreadExecutor`。创建单个工作线程来执行任务，如果这个线程异常结束，会创建另一个线程来替代。该 Executor 能确保依照任务在队列中的顺序来串行执行（例如 FIFO, LIFO, 优先级）。单线程的 Executor 还提供大量的内部同步机制，从而确保任务执行的任何内存写入操作对于后续任务来说都是可见的。
- `newScheduledThreadPool`。创建一个固定长度的线程池，而且以延迟或定时的方法来执行任务，类似于 Timer。

从为每个任务分开一个线程的策略到基于线程池的策略，将对应用程序的稳定性产生重大的影响：服务器不会创建数千个线程来争夺有限的 CPU 和内存资源。通过使用 Executor，可以实现各种调优、管理、监视、记录日志、错误报告和其他功能，如果不使用任务执行框架，那么要增加这些功能是非常困难的。

### Executor 的生命周期
Executor 的实现通过会创建线程来执行任务。但 JVM 只有在所有线程全部停止后都会退出。因此，如果无法正确地关闭 Executor，那么 JVM 将无法结束。

由于 Executor 以异步方式来执行任务，因此在任何时刻，之前提交任务的状态不是立即可见的。为了解决执行服务的生命周期问题，Executor 扩展了 ExecutorService 接口，添加了一些用于生命周期管理的方法，如下：
```java
public interface ExecutorService extends Executor {
    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
}
```
ExecutorService 的生命周期有 3 种状态：运行、关闭和终止。ExecutorService 在初始创建时处于运行状态。shutdown 方法将执行平缓的关闭过程：不再接收新的任务，同时等待已经提交的任务执行完成-- 包括那些还未开始执行的任务。shuwdownNow 方法将执行粗暴的关闭过程：尝试取消所有运行中的任务，并不再启动队列中尚未开始执行的任务。

### 延迟任务与周期任务
Timer 类负责管理延迟任务与周期任务。然而 Timer 存在一些缺陷，因此应该考虑使用 ScheduledThreadPoolExecutor 来代替它。Timer 在执行所有定时任务时只会创建一个线程，如果某个任务的执行时间过长，那么将破坏其他 TimerTask 的定时精确性。如某个周期 TimerTask A 需要每 10ms 执行一次，而另外一个 TimerTask B 需要执行40ms，那么周期任务 A 或者在周期任务执行完后快速连续调用 4 次，或者彻底“丢失” 4 次调用。而线程池能弥补这个缺陷，它可以提供多个线程来执行延时任务和周期任务。
另外 Timer 线程并不捕获异常，因此当 TimerTask 抛出未检查的异常时将终止定时线程。这种情况下，Timer 也不会恢复线程的执行，而错认为整个 Timer 都被取消了。这个问题称之为“线程泄漏”（Thread Leakage）。在 Java 5.0 及以上 JDK 中，很少使用到 Timer。

如果要构建自己的调度服务，可使用 DelayQueue，它实现了 BlockingQueue，并为 ScheduledThreadPoolExecutor 提供调度服务。

## 找出可利用的并行性
如果向 Executor 提交了一组计算任务，并希望在计算完成后获得结果，那么可以保留与每个任务关联的 Future，然后反复使用 get 方法，同时将参数 timeout 指定为 0，从而通过轮询来判断任务是否完成。这种方法虽然可行，但却有些繁琐，不过还有一种更好的方法：CompletionService。

CompletionService 将 Executor 和 BlockingQueue 的功能整合在一起，可以将 Callable 任务提交给它来执行，然后使用类似于队列操作的 take 和 poll 等方法来获得已完成的结果，而这些结果会在完成时被封装为 Future。

### 为任务设置时限
如果某个任务无法在指定时间内完成，那么将不再需要它的结果，此时可以放弃这个任务。在有限时间内执行任务的主要困难在于，要确保得到答案的时间不会超过限定的时间，或在限定的时间内无法获得答案，在支持时间限制的 Future.get 中支持这种需求：当结果可用时，它将立即返回，如果在指定时限内没有计算结果，那么将抛出 TimeoutException。在使用限时任务时需注意，当这些任务超时后，应该立即停止这些超时任务，以避免继续计算一个不再使用的结果而浪费资源。

## 小结
本章通过叙述串行执行和为每个任务都启动一个线程的缺点引出了线程池的概念，然后介绍了线程池相关的基础知识，并介绍了 Java 类库提供的四种创建线程池的静态工厂方法：`newFiexedThreadPool`、`newCachedThreadPool`、`newSingleThreadExecutor` 和 `newScheduledThreadPool`。接着介绍了 Timer 管理延迟任务和周期任务的弊端，可使用 ScheduledThreadPoolExecutor 来代替 Timer 来执行周期任务；而对于反复使用 get 方法的一组计算需求，可使用 CompletionService。另外对于设置时限的任务，需在任务超时后将其取消，以节约资源。
