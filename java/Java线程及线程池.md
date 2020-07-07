# Java 线程和线程池

-----

## 线程基本概念

> 要解释线程，就必须明白什么是 *进程*：
> 
> - 进程：
> 
> 是指运行中的应用程序，每个进程都有自己 **独立** 的地址空间(内存空间)，由操作系统直接分配。
> 
> - 线程：
>
> ***轻量级的进程、没有独立的地址空间、由进程创建、创建时开销小、进程可以拥有多个线程；***<br>
> 线程有5种状态：
>> **1. 新建(NEW)** new 关键字创建了 Thread 类；<br>
>> **2. 就绪(RUNNABLE)** 线程已经启动（调用了 `T.start()`），正在等待 CPU 资源；<br>
>> **3. 运行(RUNNING)** 已经获得 CPU 资源，正在运行；<br>
>> **4. 阻塞(BLOCKED)** 由于某些原因使处于运行中的线程 **让出** 当前的 CPU 资源，比如 `T.sleep()、t.join()`、等待用户输入等<br>
>> **5. 死亡(DEAD)** 处于 RUNNING 的线程，在执行完 `run()` 方法。
>> ![](./resource/thread-5.status.jpg)

## 几种关键的方法

- ***Thread.sleep(long millis)***：Thread 类下的方法，控制线程休眠，一定是当前线程调用此方法，当前线程进入阻塞，但不释放对象锁，millis 后线程自动苏醒进入就绪状态。作用：给其它线程执行机会的最佳方式。

- ***Thread.yield()***：一定是当前线程调用此方法，当前线程放弃获取的cpu时间片，由运行状态变会可运行状态，让OS再次选择线程。作用：让相同优先级的线程轮流执行，但并不保证一定会轮流执行。实际中无法保证yield()达到让步目的，因为让步的线程还有可能被线程调度程序再次选中。Thread.yield() 不会导致阻。

- ***t.join()/t.join(long millis)***：当前线程里调用其它线程1的 join() 方法，当前线程阻塞，但不释放对象锁，直到线程1执行完毕或者 millis 时间到，当前线程进入可运行状态。 

- ***obj.wait()***：Object 类下的方法，控制线程等待，由当前线程调，调用后释放对象锁，进入等待队列；依靠 notify()/notifyAll() 唤醒或者 wait(long timeout) timeout 时间到自动唤醒。 

- ***obj.notify()***：Object 类下的方法，唤醒在此对象监视器上等待的单个线程，选择是任意性的；notifyAll() 唤醒在此对象监视器上等待的所有线程。

## 线程创建方式

- 继承 Thread 类

> `run()` 为线程类的核心方法，相当于主线程的 main() 方法，是每个线程的入口；是由 jvm 创建完本地 *操作系统级线程* 后回调的方法，不可以手动调用；<br>
> `T.start()` 只可以被调用一次，如果调用两次 将会抛出线程状态异常；<br>
> `native` 生明的方法只有方法名，没有方法体，是本地方法，不是抽象方法，是调用 c 语言方法； ***registerNative()*** 方法包含了所有与线程相关的操作系统方法；<br>

- 覆写 Runnable 接口 (*推荐此方式*)

> 避免单继承局限性，便于共享资源；<br>
> 当子类实现 Runnable 接口，此时子类和 Thread 是代理模式关系。

- 覆写 Callable 接口

> 核心方法叫 `call()` 方法，`FutureTask` 配合拿到线程执行结果，并且可以抛出异常。

- 线程池启动

> 通过 `Executors` 工具类。

## 多线程

> 一个进程中同时运行多个线程，来完成不同的工作，***多线程实际上是交替占用 CPU 资源，而非我们表面看起来的并行执行***。

#### 使用多线程的好处

- 充分利用 CPU 的资源
- 简化编程模型
- 提高系统并发能力，高效地受理用户请求
- 部分逻辑异步处理，提高主业务流程的响应速度

#### 多线程的烦恼

- 提高程序复杂度
- 引入并发安全问题

## 线程池

#### 使用线程池的好处

- 降低资源消耗 重复利用已创建线程，降低线程创建与销毁的资源消耗；
- 提高响应效率 任务到达时，不需等待创建线程就能立即执行；
- 提高线程可管理性；
- 防止服务器过载 内存溢出、CPU耗尽。

#### 线程池应用范围

- 需要大量的线程来完成任务，并且完成所需时间较短，如Web服务器完成网页请求；
- 对性能有苛刻要求，如服务器实时响应；
- 突发性大量请求，且不至于在服务器产生大量线程。

#### 线程池的实现

- 代码架构

> Executors 静态工厂的功能，生产各种类型线程池；<br>
> Executor 线程池的顶级接口；<br>
> ***ExecutorService*** 真正的线程池接口；<br>
> AbstractExecutorService ExecutorService 的抽象实现，里面定义了许多线程池操作的方法；<br>
> ThreadPoolExecutor ExecutorService 的默认实现。

- Executors 详解

> 核心方法是 `E.execute(Runnable r) 或 E.submit(Callable<T> r)`，可以创建三种类型的普通线程池：
>> **FixThreadPool(int n)** 固定大小的线程池，适用于服务器负载较重、需要限制当前线程数量的场景；<br>
>> **SingleThreadPoolExecutor** 单线程池，适用于需要保证顺序执行各个任务的场景；<br>
>> **CachedThreadPool()** 缓存线程池，当提交任务速度高于线程池中任务处理速度时，缓存线程池会不断的创建线程，在执行结束后缓存60s，如果不被调用则移除线程，适用于服务器负载较轻、提交短期异步任务场景。

- ThreadPoolExecutor 详解

```java
/**
 * Creates a new {@code ThreadPoolExecutor} with the given initial
 * parameters.
 *
 * @param corePoolSize 核心池大小，也就是线程池中会维持不被释放的线程数量，在线程池刚创建
 * 			时，里面并没有建好的线程，只有当有任务来的时候才会创建
 * @param maximumPoolSize 线程池的最大线程数，代表着线程池中能创建多少线程池。
 * 			超出corePoolSize，小于maximumPoolSize的线程会在执行任务结束后被释放。
 * 		 	此配置在CatchedThreadPool中有效。
 * @param keepAliveTime 会被释放的线程缓存的时间
 * @param unit 参数keepAliveTime的时间单位。
 * @param workQueue 存储等待执行任务的阻塞队列：
 * 			1、SynchronousQueue——直接提交策略，适用于CachedThreadPool。它将任务直接提交给
 * 		 			线程而不保持它们。如果不存在可用于立即运行任务的线程，则试图把任务加入队列将
 * 		 		 	失败，因此会构造一个新的线程。此策略可以避免在处理可能具有内部依赖性的请求集
 * 		 		  	时出现锁。直接提交通常要求最大的 maximumPoolSize 以避免拒绝新提交的任务
 * 		 		   （正如CachedThreadPool这个参数的值为Integer.MAX_VALUE）。当任务以超过
 * 		 		   队列所能处理的量、连续到达时，此策略允许线程具有增长的可能性。吞吐量较高。
 * 		 		   
 * 		 	2、LinkedBlockingQueue——无界队列，适用于FixedThreadPool与
 * 		  			SingleThreadExcutor。基于链表的阻塞队列，创建的线程数不会超过
 * 		  			corePoolSizes（maximumPoolSize值与其一致），当线程正忙时，任务进入队列
 * 		  		 	等待。按照FIFO原则对元素进行排序，吞吐量高于ArrayBlockingQueue。
 * 		  		  
 * 		  	3、ArrayListBlockingQueue——有界队列，有助于防止资源耗尽，但是可能较难调整和控
 * 		   			制。队列大小和最大池大小可能需要相互折衷：使用大型队列和小型池可以最大限度地
 * 		   		 	降低 CPU 使用率、操作系统资源和上下文切换开销，但是可能导致人工降低吞吐量。
 * 		   		  	如果任务频繁阻塞（例如，如果它们是 I/O边界），则系统可能为超过您许可的更多
 * 		   		   线程安排时间。使用小型队列通常要求较大的池大小，CPU使用率较高，但是可能遇到
 * 		   		   不可接受的调度开销，这样也会降低吞吐量。
 * 		   
 * @param threadFactory the factory to use when the executor creates a new 
 * 			thread
 * 		 
 * @param handler the handler to use when execution is blocked because the 
 * 			thread bounds and queue capacities are reached. 
 * 		 	控制线程池任务超额处理策略，即任务拒绝策略，JDK提供4种：
 * 		   1、ThreadPoolExcutor.AbortPolicy()——直接抛出 RejectedExecutionException
 * 		   			异常，默认操作
 * 		   2、ThreadPoolExcutor.CallerRunsPolicy()——只用调用者所在线程来运行任务
 * 		   3、ThreadPoolExcutor.DiscardOldersPolicy()——丢弃队列里最近的一个任务，并执行
 * 		   			当前任务
 * 		   4、ThreadPoolExcutor.DiscardPolicy()——不处理，直接丢掉
 * 		   自定义：可以通过实现RejectedExceptionHandler接口，实现
 * 		   			rejectedException(ThreadPoolExecutor e, Runnable r)方法自定义操
 * 		   		 	作。
 * 
 * @throws IllegalArgumentException if one of the following holds:<br>
 *         {@code corePoolSize < 0}<br>
 *         {@code keepAliveTime < 0}<br>
 *         {@code maximumPoolSize <= 0}<br>
 *         {@code maximumPoolSize < corePoolSize}
 * @throws NullPointerException if {@code workQueue}
 *         or {@code threadFactory} or {@code handler} is null
 */
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler);                       
```

> excute()方法：
> 
> 1. 当少量的线程在运行，线程的数量还没有达到corePoolSize，那么启用新的线程来执行新任务。
> 2. 如果线程数量已经达到了corePoolSize，那么尝试把任务缓存起来，然后再次检查线程池的状态，看这个时候是否能添加一个额外的线程，来执行这个任务。如果这个检查到线程池关闭了，就拒绝任务。
> 3. 如果我们没法缓存这个任务，那么我们就尝试去添加线程去执行这个任务，如果失败，可能任务已被取消或者任务队列已经饱和，就拒绝掉这个任务。

- 合理配置线程池的大小

> 一般需要根据任务的类型来配置线程池大小：
> 
> - 如果是 CPU 密集型任务，就需要尽量压榨CPU，参考值可以设为 *N*(CPU)+1；
> - 如果是 IO 密集型任务，参考值可以设置为2**N*(CPU)。

