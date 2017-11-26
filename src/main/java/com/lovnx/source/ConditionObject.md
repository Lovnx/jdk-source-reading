> **小记**
> 好久没更博，窗外光芒万丈，冬日的晚晨，多么美好，就不浪费了，循着键盘上的点点星辰，开工！

####**啥子是条件队列？**
我们都知道，在万类之祖Object里面定义了几个监视器方法：wait()，notify
()，notifyAll()，配合synchronized语义来控制线程的一些状态，在JDK1.5之后，由Lock替代了synchronized，而这几个监视器由条件队列Condition来实现，以便在某个状态条件现在可能为 true 的另一个线程通知它之前，一直挂起该线程（即让其“等待”），以原子的方式释放锁，并挂起当前线程，所以，也可以叫它为`线程的条件队列`(自创的)。

####**来看一个应用示例**
在以前刚学习Java的时候写过一个题，题干大概是这样的：开启三个线程依次轮流打印出75个数，且次序不能乱。下面是代码：

```
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class CountTo_75 {

    public static void main(String[] args) {
        final Box box = new Box();

        new Thread(new Runnable() { // 线程1
            @Override
            public void run() {
                for (int i = 1; i <= 5; i++) {
                    box.main_1();
                }
            }
        },"Thread-1").start();
        new Thread(new Runnable() { // 线程2
            @Override
            public void run() {
                for (int i = 1; i <= 5; i++) {
                    box.main_2();
                }
            }
        },"Thread-2").start();
        new Thread(new Runnable() { // 线程3
            @Override
            public void run() {
                for (int i = 1; i <= 5; i++) {
                    box.main_3();
                }
            }
        },"Thread-3").start();
    }

    static class Box {
        Lock lock = new ReentrantLock();
        Condition condition_1 = lock.newCondition();
        Condition condition_2 = lock.newCondition();
        Condition condition_3 = lock.newCondition();
        private volatile int flag = 1;
        public static int count = 0;   //用于计数的变量

        public void main_1() {
            lock.lock();
            try {
                while(flag != 1){
                    try {
                        condition_1.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                for (int j = 0; j < 5; j++) {
                    count++;
                    System.out.println(Thread.currentThread().getName() +" "+ count);
                }
                flag = 2;
                condition_2.signal();
            } finally {
                lock.unlock();
            }
        }

        public void main_2() {
            lock.lock();
            try {
                while(flag != 2){
                    try {
                        condition_2.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                for (int j = 0; j < 5; j++) {
                    count++;
                    System.out.println(Thread.currentThread().getName() +" " + count);
                }
                flag = 3;
                condition_3.signal();
            } finally {
                lock.unlock();
            }
        }

        public void main_3() {
            lock.lock();
            try {
                while(flag != 3){
                    try {
                        condition_3.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                for (int j = 0; j < 5; j++) {
                    count++;
                    System.out.println(Thread.currentThread().getName() + " " + count);
                }
                flag = 1;
                condition_1.signal();
            } finally {
                lock.unlock();
            }
        }
    }
}
```
抛开当时混乱的逻辑和性能考虑不足不谈，这段丑陋的代码，终归是实现了功能，而且是运用了Lock和Condition来实现的，用在这里来说明Condition的语义再好不过了。代码中初始化了3个条件队列，分别来控制3个线程的挂起状态，flag变量则控制它们之间的关系。

####**与Lock之间的实现关系**
Condition是一个接口，其实现类只有两个：AQS和AQLS，都以内部类的形式存在，内部类叫做ConditionObject，这里有点纳闷，既然这个类是为Lock专属定制的，为什么不在ReentrantLock里面来实现呢？放在AQS不会太臃肿吗？不知道Doug Lea上神当时是怎么考虑的。
由于在AQS中已经实现，因此在ReentrantLock里面对其操作也是很简单的，创建一个条件队列：

```
    public Condition newCondition() {
        return sync.newCondition();
    }
```
sync中的实现：

```
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;
        final ConditionObject newCondition() {
            return new ConditionObject();
        }
    }
```
很简单吧？呵呵，下面来看ConditionObject

####**ConditionObject实现**
首先定义了两个核心成员变量，条件队列的头节点和尾节点：
```
/** First node of condition queue. */
private transient Node firstWaiter;

/** Last node of condition queue. */
private transient Node lastWaiter;
```

#####**1、核心方法：await() 不可中断的条件等待实现**

```
public final void await() throws InterruptedException {
     //当前线程被中断则抛异常
     if (Thread.interrupted()) throw new InterruptedException();
     //添加进条件队列
     Node node = addConditionWaiter();
     //释放当前线程持有的锁
     int savedState = fullyRelease(node);
     int interruptMode = 0;
     //在这里一直查找创建的Node节点在不在Sync队列，不在就一直禁用当前线程
     while (!isOnSyncQueue(node)) {
         //park当前线程，直到唤醒
         LockSupport.park(this);
         //如被中断也跳出循环
         if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) 
         break;
     }
     //此时已被唤醒，获取之前被释放的锁，
     if (acquireQueued(node, savedState) && interruptMode != THROW_IE)   
       interruptMode = REINTERRUPT;
     //移除节点，释放内存
     if (node.nextWaiter != null) // clean up if cancelled
       unlinkCancelledWaiters();
     //被中断后的操作
     if (interruptMode != 0)
       reportInterruptAfterWait(interruptMode);
}
```
简而言之，await()方法其实就是为当前线程创建一个Node节点，加入到Condition队列并释放锁，之后就一直查看这个节点是否在Sync队列中了(signal()方法将它移到Sync队列)，如果在的话就唤醒此线程，重新获取锁。此外，awaitNanos(long nanosTimeout) 方法和await(long time, TimeUnit unit) 方法的实现大同小异，只是在何时跳出while循环的时候加了一个超时罢了。
另外还有几个相关的方法也看一下：

```
/**
 *  添加一个Node节点到Condition队列中
 */
private Node addConditionWaiter() {
  Node t = lastWaiter;
  //如果尾节点被取消，就清理掉
  if (t != null && t.waitStatus != Node.CONDITION) {
      unlinkCancelledWaiters();
      t = lastWaiter;
  }
  //新建状态为的CONDITION节点，并添加在尾部
  Node node = new Node(Thread.currentThread(), Node.CONDITION);
  if (t == null)
      firstWaiter = node;
  else
      t.nextWaiter = node;
  lastWaiter = node;
  return node;
}
```

```
	/**
	 *  释放当前线程的state，实际还是调用tryRelease方法
	 */
   final int fullyRelease(Node node) {
       boolean failed = true;
       try {
           int savedState = getState();
           if (release(savedState)) {
               failed = false;
               return savedState;
           } else {
               throw new IllegalMonitorStateException();
           }
       } finally {
           if (failed)
               node.waitStatus = Node.CANCELLED;
       }
   }
```

```
	/**
	 *  检查当前节点在不在Sync队列
	 */
   final boolean isOnSyncQueue(Node node) {
       //如果当前节点状态为CONDITION，一定还在Condition队列
       //如果Sync队列的前置节点为null，则表明当前节点一定还在Condition队列
       if (node.waitStatus == Node.CONDITION || node.prev == null)
           return false;
       //有后继节点，也有前置节点，那么一定在Sync队列
       if (node.next != null) // If has successor, it must be on queue
           return true;
           
       //倒查Node节点，前置节点不能为null，第一个if已经做了判断，其前置节点为non-null,但是当前节点也不在Sync也是可能的,因为CAS操作将其加入队列也可能失败，所以我们需要从尾部开始遍历确保其在队列
       return findNodeFromTail(node);
   }
```

#####**2、核心方法：signal() 将等待时间最长的线程（如果存在）从Condition队列中移动到拥有锁的Sync队列**

```
   public final void signal() {
	   //当前线程非独占线程，报非法监视器状态异常
       if (!isHeldExclusively())
           throw new IllegalMonitorStateException();
       //头节点是等待时间最长的节点
       Node first = firstWaiter;
       if (first != null)
           doSignal(first);
   }
```

```
   private void doSignal(Node first) {
         do {
	         //如果头节点的下一节点为null，则将Condition的lastWaiter置为null
             if ( (firstWaiter = first.nextWaiter) == null)
                 lastWaiter = null;
             //将头结点的下一个节点设为null
             first.nextWaiter = null;
             //被唤醒并且头节点不为null则结束循环
         } while (!transferForSignal(first) &&
                  (first = firstWaiter) != null);
     }
```

```
   final boolean transferForSignal(Node node) {
       //如果无法改变节点状态，说明节点已经被唤醒
       if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
           return false;
	   //将当前节点添加到Sync队列尾部，并设置前置节点的waitStatus为SIGNAL，表明后继有节点（可能）将被唤醒，如果取消或者设置waitStatus失败，会唤醒重新同步操作，这时候waitStatus是瞬时的，出现错误也是无妨的
       Node p = enq(node);
       int ws = p.waitStatus;
       if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
           LockSupport.unpark(node.thread);
       return true;
   }
```

#####**3、核心方法：signalAll() 将所有线程从Condition队列移动到拥有锁的Sync队列中。**

```
   public final void signalAll() {
       if (!isHeldExclusively())
           throw new IllegalMonitorStateException();
       Node first = firstWaiter;
       if (first != null)
           doSignalAll(first);
   }
```

```
	/**
	 * 遍历所有节点，加入到拥有锁的Sync队列
	 */
   private void doSignalAll(Node first) {
       lastWaiter = firstWaiter = null;
       do {
           Node next = first.nextWaiter;
           first.nextWaiter = null;
           transferForSignal(first);
           first = next;
       } while (first != null);
   }
```