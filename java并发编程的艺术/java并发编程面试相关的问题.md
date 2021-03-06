# 面试相关问题

##   1.1 JMM 说说是什么？

​      **JMM是Java内存模型，是一种符合内存模型的规范，目的是解决由于多线程通过共享内存进行通信时，存在的本地内存数据不一致、编译器会对代码指令重排序、处理器会对代码乱序执行等带来的问题。目的是保证并发编程场景中的原子性、可见性和有序性。还提供了一系列原语，封装了底层实现后，供开发者直接使用**，**比如`volatile`、`synchronized`、`final`、`concurren`包等**



##   1.2 为什么要有 JMM？

​     **为了保证共享内存的正确性（可见性、有序性、原子性）**，内存模型定义了共享内存系统中多线程程序读写操作行为的规范。通过这些规则来规范对内存的读写操作，从而保证指令执行的正确性。它与处理器有关、与缓存有关、与并发有关、与编译器也有关。他解决了CPU多级缓存、处理器优化、指令重排等导致的内存访问问题，保证了并发场景下的一致性、原子性和有序性。



## 1.3 Happened-before是什么？

   在java中，一方面要让CPU和编译器可以灵活的进行重排序，另一方面也要告诉开发者，在什么情况下什么样的重排序不需要感知，需要感知什么样的重排序并作出处理。为了描述这个规范，JMM引入了happen-before，使用happen-before描述两个操作之间的内存可见性。**这是一套规则，** **简单来说，happen-before的意思就是，如果 操作A happen-before 操作B，那么操作A的执行结果必须对操作B可见。**

## 1.4  volatile和 synchronized 的区别是什么？

​     首先需要理解线程安全的两个方面：**执行控制**和**内存可见**。**执行控制**的目的是控制代码执行（顺序）及是否可以并发执行。**内存可见**控制的是线程执行结果在内存中对其它线程的可见性。根据**Java内存模型**的实现，线程在具体执行时，会先拷贝主存数据到线程本地（CPU缓存），操作完成后再把结果从线程本地刷到主存。

​    `synchronized`关键字解决的是执行控制的问题，它会阻止其它线程获取当前对象的监控锁，这样就使得当前对象中被`synchronized`关键字保护的代码块无法被其它线程访问，也就无法并发执行。更重要的是`synchronized`还会创建一个**内存屏障**，内存屏障指令保证了所有CPU操作结果都会直接刷到主存中，从而保证了操作的内存可见性，同时也使得先获得这个锁的线程的所有操作，都**happens-before**于随后获得这个锁的线程的操作。

​    `volatile`关键字解决的是内存可见性的问题，会使得所有对`volatile`变量的读写都会直接刷到主存，即保证了变量的可见性。这样就能满足一些对变量可见性有要求而对读取顺序没有要求的需求。

##   1.5 为什么要用 volatile 修饰？说说它的功能？

​         1：防止指令重排  2：保证可见性 ，  具体的功能如上图所示。

##   1.6 什么是 MESI 协议？CPU 原语是什么？什么是可见性？

​      (1)缓存一致性，具体的参考资料https://blog.csdn.net/muxiqingyang/article/details/6615199

​      (2)原语保证变量每次的修改都能从缓存中刷出到主存（至少是L3共享缓存中），读取的时候每次都从主存中读取。这样其他线程就可以见到变量的变化。

## 1.7 自旋锁是什么？偏向锁是什么？

​    所谓自旋锁，就是让该线程等待一段时间，不会被立即挂起，看持有锁的线程是否会很快释放锁。怎么等待呢？执行一段无意义的循环即可（自旋）。

​    偏向锁，顾名思义，它会偏向于第一个访问锁的线程，如果在运行过程中，同步锁只有一个线程访问，不存在多线程争用的情况，则线程是不需要触发同步的，这种情况下，就会给线程加一个偏向锁。如果在运行过程中，遇到了其他线程抢占锁，则持有偏向锁的线程会被挂起，JVM会消除它身上的偏向锁，将锁恢复到标准的轻量级锁。

## 1.8 锁消除了解吗？

​    为了保证数据的完整性，我们在进行操作时需要对这部分操作进行同步控制，但是在有些情况下，JVM检测到不可能存在共享数据竞争，这是JVM会对这些同步锁进行锁消除。锁消除的依据是逃逸分析的数据支持。 []

## 1.9 锁会被合并吗？什么时候会发生？

   通常情况下，为了保证多线程间的有效并发，会要求每个线程持有锁的时间尽可能短，但是大某些情况下，一个程序对同一个锁不间断、高频地请求、同步与释放，会消耗掉一定的系统资源，因为锁的讲求、同步与释放本身会带来性能损耗，这样高频的锁请求就反而不利于系统性能的优化了，虽然单次同步操作的时间可能很短。**锁粗化就是告诉我们任何事情都有个度，有些情况下我们反而希望把很多次锁的请求合并成一个请求，以降低短时间内大量锁请求、同步、释放带来的性能损耗。**

一种极端的情况如下：

```java
public void doSomethingMethod(){
    synchronized(lock){
        //do some thing
    }
    //这是还有一些代码，做其它不需要同步的工作，但能很快执行完毕
    synchronized(lock){
        //do other thing
    }
```

上面的代码是有两块需要同步操作的，但在这两块需要同步操作的代码之间，需要做一些其它的工作，而这些工作只会花费很少的时间，那么我们就可以把这些工作代码放入锁内，将两个同步代码块合并成一个，以降低多次锁请求、同步、释放带来的系统性能消耗，合并后的代码如下:

```java
public void doSomethingMethod(){
    //进行锁粗化：整合成一次锁请求、同步、释放
    synchronized(lock){
        //do some thing
        //做其它不需要同步但能很快执行完的工作
        //do other thing
    }

}
```

## 2.0 你刚才说了 CAS，你能说下它是什么东西吗？为什么要引入 CAS？ABA 问题是如何解决的

​    比较并替换。CAS需要有3个操作数：内存地址V，旧的预期值A，即将要更新的目标值B。CAS指令执行时，当且仅当内存地址V的值与预期值A相等时，将内存地址V的值修改为B，否则就什么都不做，然后就自旋等待，再重新尝试修改。整个比较并替换的操作是一个原子操作。

​    我们经常使用volatile关键字修饰某一个变量，表明这个变量是全局共享的一个变量，同时具有了可见性和有序性。但是却没有原子性。比如说一个常见的操作a++。这个操作其实可以细分成三个步骤：

（1）从内存中读取a（2）对a进行加1操作（3）将a的值重新写入内存中

​    在单线程状态下这个操作没有一点问题，但是在多线程中就会出现各种各样的问题了。因为可能一个线程对a进行了加1操作，还没来得及写入内存，其他的线程就读取了旧值。造成了线程的不安全现象。线程数较少、等待时间短可以采用自旋锁进行CAS尝试拿锁，较于synchronized高。

  用ATOMICSTAMPEDREFERENCE解决ABA问题，加个版本号来解决这个。

## 2.1 AQS 了解吗？它是如何实现的？

