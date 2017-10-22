**前言**
最近开始读JDK源码，所有心得准备总结成一个专栏，JDK Analysis系列的第一篇，就从万众瞩目的ReentrantLock开始吧，而谈到ReentrantLock，就不得不说AQS，它是AbstractQueuedSynchronizer类的简称，Doug Lea上神在JDK1.5将其引入，这才有了现在的并发包java.util.concurrent，所以要理解ReentrantLock的原理，AQS也是必须要搞懂的。这篇就先阐述ReentrantLock最基本的公平锁和非公平锁的实现，以及部分涉及的AQS原理，AQS源码解读将在后续跟进。整个系列基于JDK1.8.0_92。

**公平锁与非公平锁**
大家都知道，在JDK1.5之前，我们在多线程的环境下要想保证线程安全，就必须要使用synchronized关键字来实现对象锁或者类锁，以此满足这样的需求，JDK1.5之后则使用Lock来实现更加细粒度的锁。在刚接触Java的时候，学到这两种方式的时候，粗略地知道后者更加贴近面向对象的思想，但是在工作中遇到一些奇奇怪怪的需求的时候，只是知道这个是远远不够的。synchronized其实是一种公平锁，所谓公平锁，就是线程按照执行顺序排成一排，依次获取锁，但是这种方式在高并发的场景下极其损耗性能；这时候，Lock带着非公平锁应运而生了，所谓非公平锁，就是不管执行顺序，每个线程获取锁的几率都是相同的，获取失败了，才会采用像公平锁那样的方式。这样做的好处是，JVM可以花比较少的时间在线程调度上，更多的时间则是用在执行逻辑代码里面。

公平锁、非公平锁的创建方式：
```
//创建一个非公平锁，默认是非公平锁
Lock lock = new ReentrantLock();
Lock lock = new ReentrantLock(false);

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

**NonfairSync 非公平锁**
在谈NonfairSync之前，首先要谈谈ReentrantLock类里面定义的一个类属性Sync，它才是ReentrantLock实现的精髓。它首先在属性里声明，然后以抽象静态内部类的形式实现了AQS，源码如下：

```
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;
        
		//声明的lock()方法，供子类实现
        abstract void lock();
        
		//非公平锁的获取方式，相较于公平锁的tryAcquire()
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
		
        //释放锁
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

		//判断当前线程是否是锁的持有者
        protected final boolean isHeldExclusively() {
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        //获取当前锁持有线程，如果在队列中等待获取锁，则返回null
        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }
        
		//返回当前线程status的状态，如果持有锁就读取status，没有就0
        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }
		
        //是否上锁
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
此后，NonfairSync继承它来实现非公平锁，FairSync继承它来实现公平锁，AQS提供一个tryAcquire()的模板方法来使得公平锁和非公平锁的实现方式显得灵活。我们来看看NonfairSync的源码：
```
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;
        
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```
如代码所示，在lock的时候，先是尝试将AQS的status从0设为1，成功的话就把当前线程设置为锁的持有者，如果尝试失败了，基于模板方法，实际调用的是Sync的nonfairTryAcquire(int acquires)方法，该方法源码如下：
```
       final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //ReentrantLock是可重入锁是这里实现的
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
首先获取当前线程，和当前AQS维护的锁的状态，如果状态为0，则尝试将AQS的status从0设为acquires(实际是1)，如果设置成功，则获取锁成功，把当前锁设置为锁的持有者，返回true；如果当前线程已经是锁的持有者，则把status+acquires，如果结果越界，抛出异常，如果成功，返回true。细心的同学可以发现，一共有两次原子设status从0到1，为什么呢？因为这样可以提高获取锁的概率，因为是非公平的，所以有必要进行这样的操作，而且这样的操作与锁相对来讲损耗微乎其微。

**FairSync 公平锁**
公平锁就是每个线程在获取锁时会先查看此锁维护的等待队列，如果为空，或者当前线程线程是等待队列的第一个，就占有锁，否则就会加入到等待队列中，以后会按照FIFO的规则从队列中获取，下面是FairSync 的源码：
```
static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

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
来看子类的 tryAcquire方法，与非公平锁比较，获取锁的操作只有一点不同，就是加入了hasQueuedPredecessors() 方法，该方法又大有来头，ctrl进去是这的：
```
    public final boolean hasQueuedPredecessors() {
        Node t = tail; 
        Node h = head;
        Node s;
        //head没有next ----> false
		//head有next，next持有的线程不是当前线程 ----> true
		//head有next，next持有的线程是当前线程 ----> false
        return h != t && ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```
该方法的签名是：查询是否有其他线程比当前线程等待获取锁花费了更多的时间。在AQS中对线程是做了一个FIFO队列，这里的tail是尾，head是头，具体的实现会在后续跟进，这里就不多做赘述，有意思的是return那一行，其中的意思在上面做了解答，查询是否有其他线程比当前线程等待获取锁花费了更多的时间，有就返回true，没有就返回false，也就是说该方法返回false，才进行addWaiter状态的更改尝试，其余和部分和非公平锁的部分一样。

ctrl点进acquire(1)是这样的：
```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
首先通过tryAcquire方法尝试获取锁，如果成功直接返回，否则通过acquireQueued()再次尝试获取。在acquireQueued()中会先通过addWaiter将当前线程加入到CLH队列的队尾，在CLH队列中等待。在等待过程中线程处于休眠状态，直到成功获取锁才会返回。