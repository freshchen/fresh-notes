# Java读源码之ReentrantLock（2）

## 前言

本文是 ReentrantLock 源码的第二篇，第一篇主要介绍了公平锁非公平锁正常的加锁解锁流程，虽然表达能力有限不知道有没有讲清楚，本着不太监的原则，本文填补下第一篇中挖的坑。

[Java读源码之ReentrantLock](https://gitlab.aihaisi.com/qiexr/docs/issues/689)

## 源码分析

### 感知中断锁

如果我们希望检测到中断后能立刻抛出异常就用 lockInterruptibly 方法去加锁，还是建议用 lock 方法，自定义中断处理，更灵活一点。

- **ReentrantLock#lockInterruptibly**

我们只需要把 **ReentrantLock#lock** 改成 **ReentrantLock#lockInterruptibly** 方法就可以获得内部检测中断的锁了

```java
public void lockInterruptibly() throws InterruptedException {
  sync.acquireInterruptibly(1);
}
```

- **AbstractQueuedSynchronizer#acquireInterruptibly**

主要流程和前文介绍的类似

```java
public final void acquireInterruptibly(int arg)
  throws InterruptedException {
  // 一上来就检查下中断，中断直接异常，就没必要抢锁排队了
  if (Thread.interrupted())
    throw new InterruptedException();
  if (!tryAcquire(arg))
    doAcquireInterruptibly(arg);
}
```

- **AbstractQueuedSynchronizer#doAcquireInterruptibly**

和正常加锁唯一区别就是这个方法，但是定睛一看是不是似曾相识？最大区别就是把中断标识给去掉了，检测到中断直接抛异常

```java
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
  	// 大神也偷懒了，因为这个方法，只有独占锁且检查中断这一个应用场景，把节点入队的步骤也揉了进来
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
              	// 当线程拿到锁苏醒过来，发现自己挂起过程被中断了，直接抛出异常
                throw new InterruptedException();
        }
    } finally {
      	// 只要发生了中断异常，就会进取消加锁方法
        if (failed)
            cancelAcquire(node);
    }
}
```

- **AbstractQueuedSynchronizer#cancelAcquire**

此方法很有东西，只保证该节点失效，然后延迟移出等待队列

```java
private void cancelAcquire(Node node) {
    if (node == null)
        return;
		// 把节点里登记等待的线程去掉，完成这一步此节点已经没有作用了
    node.thread = null;

    // 下面的三步其实可以放到一个CAS中，直接设置 CANCELLED 状态 ，拿前一个节点，predNext 也必然是自己，但是吞吐量就下来了
    // 这里大神，没有这样做也是出于了性能考虑，因为我们已经把等待线程设置成 null 了，所以此节点已经没有任何意义，没有必要去保证节点第一时间被释放，只要设置好 CANCELLED 状态
    // 就算后面 CAS 调整等待队列失败了，下次取消操作也会帮着回收。相应地代码复杂度提高了。
  
    /* ----------------------------------------- */
    // 找到自己前面第一个没取消的节点，
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    // 主要是为了下面把链表接上
    Node predNext = pred.next;
    // 这里逻辑上把当前节点的状态设置成取消，便于检测释放
    node.waitStatus = Node.CANCELLED;
  	/* ----------------------------------------- */
  
    // 如果当前节点是尾节点，就把前一个没取消的节点设成新尾巴
    if (node == tail && compareAndSetTail(node, pred)) {
      	// 把新尾巴的 next 设置成空
        compareAndSetNext(pred, predNext, null);
    } else {
        // 进到这里说明当前节点肯定不是尾节点了
        int ws;
      	// 条件1: 如果前一个非取消节点不是头，也就是还需要排队
      	// 条件2: 如果前一个节点为 SIGNAL，也就是说后面肯定还有线程等待被唤醒
      	// 条件3: 如果前一个节点也取消了，说明前一个节点也取消了，还没来得及设置状态
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
              	// 当前节点后一个没取消的话，就接到前一个正常的节点后面
                compareAndSetNext(pred, predNext, next);
        } else {
          	// 前一篇文章解锁部分讲过，会把下一个节点中的线程恢复，然后把后继节点接上
            unparkSuccessor(node);
        }

      	// 有点花里胡哨，直接 = null不行么，
        node.next = node; // help GC
    }
}
```

来张图说明下，假如我们目前等待队列里有7个线程。我们尝试中断线程7我们假设 CAS 操作都能成功：

![](https://cdn.jsdelivr.net/gh/freshchen/resource/img/draw/aqs-2-1.png)



### 等待条件锁

上篇文章看源码过程中，AQS中有个 CONDITION 状态没有研究

```java
static final int CONDITION = -2;
```

ReentrantLock 中的 newCondition 等 Condition 相关方法正是基于 AQS 中的实现的，让我们先大致了解一波作用和用法

#### Condition简介

Condition 类似于 Object 中的 wait 和 notify ，主要用于线程间通信，最大的优势是 Object 的 wait 是把线程放到当前对象的等待池中，也就是说一个对象只能有一个等待条件，而 Condition 可以支持多个等待条件，举个例子，商品要等至少三个人预定了才开始发售，第一个预定的减500，第二三两个减100。正式发售之后恢复原价。

```java
public class ReentrantLockConditionDemo {

    private final ReentrantLock reentrantLock = new ReentrantLock();
    private final Condition wait1 = reentrantLock.newCondition();
    private final Condition wait2 = reentrantLock.newCondition();
    private int wait1Count = 0;
    private int wait2Count = 0;

    public void buy() {
        int price = 999;
        reentrantLock.lock();
        try {
            while (wait1Count++ < 1) {
                System.out.println(Thread.currentThread().getName() + "减500");
                wait1.await();
                price -= 500;
            }
            wait1.signal();
            while (wait2Count++ < 2) {
                System.out.println(Thread.currentThread().getName() + "减100");
                wait2.await();
                price -= 100;
            }
            wait2.signal();
            System.out.println(Thread.currentThread().getName() + "到手价" + price);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            reentrantLock.unlock();
        }
    }

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        ReentrantLockConditionDemo reentrantLockConditionDemo = new ReentrantLockConditionDemo();
        IntStream.rangeClosed(0, 4)
                .forEach(num -> executorService
                        .execute(reentrantLockConditionDemo::buy)
                );
    }
  
    /**
     * 输出：
     *
     * pool-1-thread-1减500
     * pool-1-thread-2减100
     * pool-1-thread-3减100
     * pool-1-thread-4到手价999
     * pool-1-thread-5到手价999
     * pool-1-thread-1到手价499
     * pool-1-thread-2到手价899
     * pool-1-thread-3到手价899
     */
}
```

- **ReentrantLock#newCondition**

先来看条件的创建，需要基于锁对象使用 newCondition 去创建

```java
public Condition newCondition() {
  return sync.newCondition();
}

final ConditionObject newCondition() {
  // ConditionObject 是 AQS 中对 Condition 的实现
  return new ConditionObject();
}
```

#### ConditionObject结构

上一篇文章中介绍了 Node 结构，这里条件也使用了这个节点定义了一个单链表，统称为条件队列，上一篇介绍统称同步队列。条件队列结构相当简单就不单独画图了。

```java
// 条件队列头
private transient Node firstWaiter;
// 条件队列尾
private transient Node lastWaiter;

// 因为默认感知中断，需要考虑如何处理
// 退出条件队列时重新设置中断位
private static final int REINTERRUPT =  1;
// 退出条件队列时直接抛异常
private static final int THROW_IE    = -1;
```

#### 条件队列入队

- **AbstractQueuedSynchronizer.ConditionObject#await**

```java
public final void await() throws InterruptedException {
  if (Thread.interrupted())
    throw new InterruptedException();
  // 到条件队列中排队
  Node node = addConditionWaiter();
  // 此方法比较简单，就是调用前一篇讲过的 release 方法释放锁（调用 await 时必定是锁的持有者）
  // savedState 是进入条件队列前，持有锁的数量
  // 失败会直接抛出异常，并且最终把节点状态设置为 CANCELLED
  int savedState = fullyRelease(node);
  int interruptMode = 0;
  // 判断在不在同步队列（当调用signal之后会从条件队列移到同步队列）
  // 此判断很简单：节点状态是 CONDITION 肯定 false，否则就到同步队列中去找
  while (!isOnSyncQueue(node)) {
    // 挂起
    LockSupport.park(this);
    // 检查是不是因为中断被唤醒的
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
      break;
  }
  // 
  if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
    interruptMode = REINTERRUPT;
  if (node.nextWaiter != null) // clean up if cancelled
    unlinkCancelledWaiters();
  if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
}
```


