# J.U.C之重入锁：ReentrantLock

​    **ReentrantLock，可重入锁，是一种递归无阻塞的同步机制。**它可以等同于synchronized的使用，但是ReentrantLock提供了比synchronized更强大、灵活的锁机制，可以减少死锁发生的概率。 

**API介绍如下：**

> **一个可重入的互斥锁定 Lock，它具有与使用 synchronized 方法和语句所访问的隐式监视器锁定相同的一些基本行为和语义，但功能更强大。**
>
> **ReentrantLock 将由最近成功获得锁定，并且还没有释放该锁定的线程所拥有。当锁定没有被另一个线程所拥有时，调用 lock 的线程将成功获取该锁定并返回。如果当前线程已经拥有该锁定，此方法将立即返回。可以使用 isHeldByCurrentThread() 和 getHoldCount() 方法来检查此情况是否发生。**

​    ReentrantLock还提供了公平锁也非公平锁的选择，构造方法接受一个可选的公平参数（默认非公平锁），当设置为true时，表示公平锁，否则为非公平锁。公平锁与非公平锁的区别在于公平锁的锁获取是有顺序的。但是公平锁的效率往往没有非公平锁的效率高，在许多线程访问的情况下，公平锁表现出较低的吞吐量（***如果一个线程组里，能保证每个线程都能拿到锁，那么这个锁就是公平锁。相反，如果保证不了每个线程都能拿到锁，也就是存在有线程饿死，那么这个锁就是非公平锁***）。

## 1 获取锁

我们一般都是这么使用ReentrantLock获取锁的：

```java
ReentrantLock lock = new ReentrantLock();
lock.lock();
```

lock方法：

```java
public void lock() {   
    sync.lock();
}
```

​    Sync为ReentrantLock里面的一个内部类，它继承AQS（AbstractQueuedSynchronizer），**它有两个子类：公平锁FairSync和非公平锁NonfairSync。**

​    ReentrantLock里面大部分的功能都是委托给Sync来实现的，同时Sync内部定义了lock()抽象方法由其子类去实现，默认实现了nonfairTryAcquire(int acquires)方法，可以看出它是非公平锁的默认实现方式。下面我们看非公平锁的lock()方法：

```java
final void lock() {   
    //尝试获取锁  
    if (compareAndSetState(0, 1))     
        setExclusiveOwnerThread(Thread.currentThread());    
    else        
    //获取失败，调用AQS的acquire(int arg)方法      
        acquire(1);
}
```

​    首先会第一次尝试快速获取锁，如果获取失败，则调用acquire(int arg)方法，该方法定义在AQS中，如下：

```java
public final void acquire(int arg) {    
    if (!tryAcquire(arg) 
        && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))       
        selfInterrupt();
}
```

​    这个方法首先调用tryAcquire(int arg)方法，在AQS中讲述过，tryAcquire(int arg)需要自定义同步组件提供实现，非公平锁实现如下：

```java
protected final boolean tryAcquire(int acquires) {
        //当前线程
            final Thread current = Thread.currentThread();
    //获取同步状态
            int c = getState();
    //state == 0,表示没有该锁处于空闲状态
            if (c == 0) {
                //获取锁成功，设置为当前线程所有
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
//线程重入 
//判断锁持有的线程是否为当前线程
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

​    该方法主要逻辑：首先判断同步状态state == 0 ?，如果是表示该锁还没有被线程持有，直接通过CAS获取同步状态，如果成功返回true。如果state != 0，则判断当前线程是否为获取锁的线程，如果是则获取锁，成功返回true。成功获取锁的线程再次获取锁，这是增加了同步状态state。



## 2 **释放锁**

​    获取同步锁后，使用完毕则需要释放锁，ReentrantLock提供了unlock释放锁：

```java
public void unlock() {   
    sync.release(1);
}
```

​    unlock内部使用Sync的release(int arg)释放锁，release(int arg)是在AQS中定义的：

```java
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

​    与获取同步状态的acquire(int arg)方法相似，释放同步状态的tryRelease(int arg)同样是需要自定义同步组件自己实现：

```java
protected final boolean tryRelease(int releases) {
    //减掉releases
            int c = getState() - releases;
    //如果释放的不是持有锁的线程，抛出异常
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
    //state == 0 表示已经释放完全了，其他线程可以获取同步状态了
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

​    只有当同步状态彻底释放后该方法才会返回true。当state == 0 时，则将锁持有线程设置为null，free= true，表示释放成功。

## 3 **公平锁与非公平锁**

​    公平锁与非公平锁的区别在于获取锁的时候是否按照FIFO的顺序来。释放锁不存在公平性和非公平性，上面以非公平锁为例，下面我们来看看公平锁的tryAcquire(int arg)：

```java
protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

  比较非公平锁和公平锁获取同步状态的过程，会发现两者唯一的区别就在于公平锁在获取同步状态时多了一个限制条件：hasQueuedPredecessors()，定义如下：

```java
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; //尾节点
    Node h = head;  //头节点
    Node s;
    //头节点 != 尾节点
//同步队列第一个节点不为null 
//当前线程是同步队列第一个节点
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

​    该方法主要做一件事情：主要是判断当前线程是否位于CLH同步队列中的第一个。如果是则返回true，否则返回false。

## 4 **ReentrantLock与synchronized的区别**

**首先他们肯定具有相同的功能和内存语义。**

1. 与synchronized相比，ReentrantLock提供了更多，更加全面的功能，具备更强的扩展性。例如：时间锁等候，可中断锁等候，锁投票。
2. ReentrantLock还提供了条件Condition，对线程的等待、唤醒操作更加详细和灵活，所以在多个条件变量和高度竞争锁的地方，ReentrantLock更加适合（以后会阐述Condition）。
3. ReentrantLock提供了可轮询的锁请求。它会尝试着去获取锁，如果成功则继续，否则可以等到下次运行时处理，而synchronized则一旦进入锁请求要么成功要么阻塞，所以相比synchronized而言，ReentrantLock会不容易产生死锁些。
4. ReentrantLock支持更加灵活的同步代码块，但是使用synchronized时，只能在同一个synchronized块结构中获取和释放。注：ReentrantLock的锁释放一定要在finally中处理，否则可能会产生严重的后果。
5. ReentrantLock支持中断处理，且性能较synchronized会好些。