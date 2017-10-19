#### 前言
JDK Analysis系列的第一篇，就从万众瞩目的ReentrantLock开始吧，而谈到ReentrantLock，就不得不说AQS，它是AbstractQueuedSynchronizer类的简称，Doug Lea上神在JDK1.5将其引入，这才有了现在的并发包java.util.concurrent，所以要理解ReentrantLock的原理，AQS也是必须要搞懂的。

##### 公平锁与非公平锁
大家都知道，在JDK1.5之前，我们在多线程的环境下要想保证线程安全，就必须要使用synchronized关键字来实现对象锁或者类锁，以此满足这样的需求，JDK1.5之后则使用Lock来实现更加细粒度的锁。在刚接触Java的时候，学到这两种方式的时候，粗略地知道后者更加贴近面向对象的思想，但是在工作中遇到一些奇奇怪怪的需求的时候，只是知道这个是远远不够的。synchronized其实是一种公平锁，所谓公平锁，就是线程按照执行顺序排成一排，依次获取锁，但是这种方式在高并发的场景下极其损耗性能；这时候，Lock带着非公平锁应运而生了，所谓非公平锁，就是不管执行顺序，每个线程获取锁的几率都是相同的，获取失败了，才会采用像公平锁那样的方式。这样做的好处是，JVM可以花比较少的时间在线程调度上，更多的时间则是用在执行逻辑代码里面。

公平锁、非公平锁的创建方式：
```
//创建一个非公平锁，默认是非公平锁
Lock lock = new ReentrantLock();

//创建一个公平锁，构造传参true
Lock lock = new ReentrantLock(true);
```
相关源码：
```
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```
这部分源码比较简单，这里对于源码就不赘述。

##### NonfairSync 非公平锁
在谈NonfairSync之前，首先要谈谈ReentrantLock类里面定义的一个类属性Sync，它才是ReentrantLock实现的精髓。它首先在属性里声明，然后以抽象静态内部类的形式实现了AQS，源码如下：

```
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * Performs {@link Lock#lock}. The main reason for subclassing
         * is to allow fast path for nonfair version.
         */
        abstract void lock();

        /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        // Methods relayed from outer class
        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        final boolean isLocked() {
            return getState() != 0;
        }

        /**
         * Reconstitutes the instance from a stream (that is, deserializes it).
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }
```
此后，NonfairSync继承它来实现非公平锁。
