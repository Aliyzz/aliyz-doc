# Java 神奇的锁

-----

> ***锁*** 用来解决线程安全问题，线程安全产生的根本原因是 由于 `编译重排` 或 `指令重排` 的存在以及 `JMM` 固有的特点，使得程序执行在时间和空间上不可控。

## 背景知识

#### 编译器重排序（指令重排）

> 简称 编译重排，也是我们常说的 Java 指令重排序；编译器会对高级语言的代码进行分析，在不改变单线程语义的情况下，编译器会选择对代码进行优化然后生成汇编代码。
> 
> 目的：提升程序在 CPU 上的运行性能，尽可能减少寄存器的读取、存储次数，充分复用寄存器的存储值。
> 
> 禁止：多线程情况下编译器优化会导致一些问题的出现，使用 ***`编译器屏障`*** 实现。
> 
> **注意**：这里的编译器指的是将 *高级语言* 编译成 *汇编语言*（机器码）的编译器（比如说 GCC、clang、甚至JIT），而不是 Javac 这种编译成 中间语言（JVM 字节码）的编译器

#### 处理器（CPU）重排序

> 属于硬件优化范畴，会遵守 ***`as-if-serial`*** 语义
>
> 指令并行重排序：在单线程语义内，处理器对 *不存在数据依赖关系的操作* 做重排序；
>
> 指令乱序重排序：从硬件架构上讲，处理器将多条指令不按程序规定的顺序分开发送给各个相应的电路单元进行处理，如 加法指令走加法器，除法指令走除法器；一般情况下是 “***顺序流入，乱序流出***”。

#### Java 内存模型（JMM）

> Java 内存模型规定所有的变量都是存在 **主存** 当中，每个线程都有自己的 **工作内存**；
> 
> 线程对变量的所有操作都必须在工作内存中进行，而不能直接对主存进行操作；
> 
> 线程不能访问其他线程的工作内存。

https://www.lagou.com/lgeduarticle/88558.html


## 问题引入

- 原子性：一个操作或者多个操作，要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行；
- 可见性：当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值；
- 顺序性：程序执行的顺序按照代码的先后顺序执行。


## synchronized

#### 如何理解

> Java 内置锁，对象锁，特点是 排他的、可重入（避免死锁）的锁；
> 
> 可以把任何一个非 `NULL` 对象作为锁，叫做 ***对象监视器（Object Monitor）***；
> 
> 锁释放 正常退出释放 和 异常自动释放。

#### 三种用法

> 实例方法：监视器锁（monitor）便是对象实例（this）；
> 
> 静态方法：监视器锁（monitor）便是对象的 Class 实例（因为 Class 数据存在于元空间，因此静态方法锁相当于该类的一个全局锁）；
> 
> 对象实例：监视器锁（monitor）便是括号括起来的对象实例。

#### 同步原理

> 在软件层面依赖 JVM。

- 对象同步

> ***monitorenter*** 指令：
> 
> 每个对象都是一个监视器锁（monitor），尝试获取 monitor 的所有权
>> 1. 如果 monitor 的 **进入数** 为 0，则该线程进入 monitor，然后将进入数设置为 1，该线程即为 monitor 的所有者；<br>
>> 2. 如果线程已经占有该 monitor，只是重新进入，则进入 monitor 的进入数加1；<br>
>> 3. 如果其他线程已经占用了 monitor，则该线程进入阻塞状态，直到 monitor 的进入数为 0，再重新尝试获取 monitor 的所有权。

> ***monitorexit*** 指令：
> 
> 执行 monitorexit 的线程必须是 objectref 所对应的 monitor 的所有者
>> 指令执行时，monitor 的进入数减 1，如果减 1 后进入数为 0，那线程退出 monitor，不再是这个monitor 的所有者，其他被这个 monitor 阻塞的线程可以尝试去获取这个 monitor 的所有权。

> 扩展 wait/notify
>> Synchronized的语义底层是通过一个 monitor 的对象来完成，其实 wait/notify 等方法也依赖于 monitor 对象，这就是为什么只有在同步的块或者方法中才能调用 wait/notify 等方法，否则会抛出 `java.lang.IllegalMonitorStateException` 的异常的。

- 方法同步

> JVM 根据标示符 ***`ACC_SYNCHRONIZED `*** 来实现方法的同步
>> 当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取 monitor，获取成功之后才能执行方法体，方法执行完后再释放 monitor；<br>
>> 在方法执行期间，其他任何线程都无法再获得同一个 monitor 对象。

- 比较

> 两种同步方式本质上没有区别，只是方法的同步是一种 *隐式* 的方式来实现，无需通过字节码来完成。两个指令的执行是 JVM 通过调用操作系统的 ***互斥原语 mutex*** 来实现，被阻塞的线程会被挂起、等待重新调度，会导致“用户态和内核态”两个态之间来回切换，对性能有较大影响。

https://www.cnblogs.com/aspirant/p/11470858.html

#### 可见性保证

> 线程得到锁时，从内存里重新读入数据，并刷新缓存；释放锁时将数据回写内存。

https://www.cnblogs.com/www-123456/p/10970048.html

https://blog.csdn.net/weixin_41950473/article/details/92080488?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase

## volatile

`https://www.cnblogs.com/dolphin0520/p/3920373.html`

#### 如何理解

> volatile 关键字是与Java的内存模型有关。
> 
> volatile 不保证原子性。


#### 可见性保证

> 1. 对变量的操作会立即被更新到主存，读取变量时会去主存读取；
> 2. 对变量的操作会禁止指令重排序。

#### 同步原理

> 从汇编代码发现，加入 volatile 关键字时，会多出一个 ***lock 前缀指令***，相当于一个内存屏障，提供3个功能：
> 
> 1. 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；
> 2. 它会强制将对缓存的修改操作立即写入主存；
> 3. 如果是写操作，它会导致其他CPU中对应的缓存行无效。

#### 最佳场景

> 1. 对变量的写操作不依赖于当前值；<br>
> 2. 该变量没有包含在具有其他变量的不变式中。
> 
> ***简单说就是在保证操作原子性的场景下使用***

## CAS

参考：https://blog.csdn.net/u011506543/article/details/82392338

#### 如何理解

> CAS（Compare and swap）比较和替换是设计并发算法时用到的一种技术。简单来说，比较和替换是使用一个 **期望值** 和一个变量的 **当前值** 进行比较，如果当前变量的值与我们期望的值 **相等**，就使用一个新值替换当前变量的值。

#### 原子性保证

> CAS 通过调用 `JNI` 的代码实现的，JNI：Java Native Interface 为 JAVA 本地调用；而`compareAndSwapInt`（实现位于 unsafe.cpp 包）就是借助 C 来调用 CPU 底层指令实现的。


#### CAS 缺点

> ABA问题，解决思路就是在变量前面追加上版本号；<br>
> 循环时间长开销大；<br>
> 只能保证一个共享变量的原子操作

#### Java 实现

```java
package java.util.concurrent.atomic;

class AtomicInteger {

	/**
     * Atomically sets the value to the given updated value
     * if the current value {@code ==} the expected value.
     *
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that
     * the actual value was not equal to the expected value.
     */
	public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
}

class AtomicReference<V> {

	/**
     * Atomically sets the value to the given updated value
     * if the current value {@code ==} the expected value.
     * 
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that
     * the actual value was not equal to the expected value.
     */
    public final boolean compareAndSet(V expect, V update) {
        return unsafe.compareAndSwapObject(this, valueOffset, expect, update);
    }
}
```

## LOCK

参考：https://blog.csdn.net/qpzkobe/article/details/78586619

#### 如何理解

> Lock 完全用 Java 写成，在 Java 这个层面是无关JVM实现的。
> 
> 在 `java.util.concurrent.locks` 包中有很多 Lock 的实现类，常用的有：
>> ReentrantLock --- <br>
>> ReadWriteLock --- 实现类 ReentrantReadWriteLock 
>
> 其实现都依赖 ***`java.util.concurrent.AbstractQueuedSynchronizer`***：

```java
// 抽象内部类
abstract static class Sync extends AbstractQueuedSynchronizer{
	...
}

// 非公平锁 默认方式
static final class NonfairSync extends Sync {
	...
}

// 公平锁
static final class FairSync extends Sync {
    ...
}
```

#### AQS 抽象队列同步器

> AQS 会把所有的请求线程构成一个 `CLH` 队列，当一个线程执行完毕（lock.unlock()）时会激活自己的后继节点，但正在执行的线程并不在队列中，而那些等待执行的线程全部处于阻塞状态；
> 
> 调查线程的显式阻塞是通过调用 `LockSupport.park()` 完成，而它又调用`sun.misc.Unsafe.park()` 本地方法；再进一步，HotSpot 在 Linux 中中通过调用 `pthread_mutex_lock` 函数把线程交给系统内核进行阻塞。


#### 同步原理（待确认！！）

> 在硬件层面依赖特殊的 CPU 指令。