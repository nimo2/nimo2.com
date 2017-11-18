---
title: 膜拜Doug.Lea系列 locks实现(part 1)
categories: 技术
tags: 膜拜Doug.Lea系列
date: 2017-08-22 20:43:21
---


Doug.Lea老爷子写的JDK的java.util.concurrent.locks包，提供了一组完善的锁机制。之前曾刷过这个包的代码，但是都没有留下什么干货，这一次做一个总结，也将此作为膜拜Doug.Lea系列的第一章。（希望还能有第二章。。。）



### locks包结构

locks包内的内容实际不多，结构也很清晰。

{% asset_img locks包结构.png locks包结构  %}

其中有一个最值得关注的，也是接下来我想重点分析的类—— AbstractQueuedSynchronizer。我们可以称之为所有Java锁实现的内核，通过对这个抽象类做不同的具体实现，可以提供不同锁特性。实际上JDK也是这样做的，在ReentrantLock和ReentrantReadWriteLock以及其他一些同步类实现中，都有子类（或自身）继承了该抽象类做具体的特性的实现。



### AbstractQueuedSynchronizer（AQS）

AQS的核心功能是一个"自旋+阻塞式"锁。而后在此基础上，逐步完善了异常处理，共享锁，Condition条件阻塞等特性。为了让整篇文章思路更清晰，我写了一个简易版的AQS——SpinLock。从这个简易版的AQS出发,逐步扩展功能，大家可以顺着思路最终看到一个完整的AQS。



#### 自旋队列锁——SpinLock

应该说自旋队列锁是AQS的最粗糙版本，可以实现一个基本的锁机制。基础的逻辑可以精简成两句话。

1. 竞争锁资源，如果竞争不成功，则进入队列自旋。
2. 释放锁资源，触发队首节点自旋的退出条件。队首节点重新尝试竞争锁资源，成功则退出自旋，退出队列。否则继续。

怎么去实现这两句话呢？可以想想有哪些必须要具备的元素。

1. 锁资源，我们需要一个能够代表锁的属性，什么都可以，只要保证在这个属性上，各个线程是竞争的即可。
2. 队列，我们要有一个保存竞争失败线程的队列。这个队列我们从队尾开始排队竞争锁失败的线程，从队首开始尝试反复竞争锁资源。所以我们这个队列需要有tail 和 head 属性。同时，为了保持队列结构，我们每个节点需要保持前一个节点的引用，即需要一个prev属性。
3. 锁资源持有者，我们要有一个属性标明锁资源的持有者，这样在释放的时候，我们能够确定当前想要释放锁的线程是不是真的持有锁。

好了，万事具备了吗？还没有，还有一点需要注意，因为锁资源、队列首尾的资源都是有可能多个线程竞争写入的，所以在对这些资源的写入操作时，需要注意是否是线程安全的，如果不是线程安全，那么必须采用CAS操作，只有当CAS操作成功的前提下，才能确保当前线程是竞争胜出者，可以对该资源所一系列写入操作。

OK，这下是万事具备了，现在来完成代码实现。

```java
/**
 * Created with IntelliJ IDEA.
 * <p>
 * 自旋锁
 * <p>
 *
 * 1. 竞争锁资源，如果竞争不成功，则进入队列自旋。
 * 2. 释放锁资源，触发队首节点自旋的退出条件。队首节点重新尝试竞争锁资源，成功则退出自旋。否则继续。
 *
 * @author nimo2
 * @since 2017/8/17
 */
public class SpinLock {

    /** 锁持有线程 */
    private transient Thread owner;

    /** 锁资源 */
    private volatile int state = 0;

    /** 队尾结点 用于管理节点入队列 */
    private volatile Node tail;

    /** 队首节点 用于管理节点出队列 */
    private volatile Node head;

    /** 状态属性原子更新器 */
    private static final AtomicIntegerFieldUpdater<SpinLock> STATE_UPDATER = AtomicIntegerFieldUpdater.newUpdater(SpinLock.class, "state");

    /** 队尾节点引用原子更新器 */
    private static final AtomicReferenceFieldUpdater<SpinLock, Node> TAIL_UPDATER = AtomicReferenceFieldUpdater.newUpdater(SpinLock.class, Node.class, "tail");

    /** 队首节点引用原子更新器 */
    private static final AtomicReferenceFieldUpdater<SpinLock, Node> HEAD_UPDATER = AtomicReferenceFieldUpdater.newUpdater(SpinLock.class, Node.class, "head");

    private class Node {

        /** 每个Node在prev节点上自旋 */
        private Node prev;

    }

    /**
     * 尝试获取锁资源，在这里做了锁资源的写入操作，并非是线程安全的，
     * 需要CAS操作，保证当前线程是竞争胜出者。
     * @return boolean
     */
    private boolean tryAcquire() {
        if(STATE_UPDATER.compareAndSet(this, 0, 1)) {
            owner = Thread.currentThread();
            return true;
        }
        return false;
    }

    /**
     * 尝试释放锁资源，如果当前线程不拥有锁资源，则抛出异常。
     * 否则，先释放owner属性，再释放锁资源。
     * 这里不需要担心锁资源的并发写入，因为此时当前线程还是锁持有者，别人无法与其竞争state的写入权限。
     * @return boolean
     */
    private boolean tryRelease() {
        if(owner != Thread.currentThread()) {
            System.out.println(Thread.currentThread().getName() + "释放锁失败，持有线程:" + owner.getName());
            throw new IllegalMonitorStateException();
        }
        owner = null;
        state = 0;//注意释放顺序
        return true;
    }

    /**
     * 尝试获取锁，如果获取失败，则进入队列自旋
     * @return boolean
     */
    private boolean acquire() {

        if(!tryAcquire()) {
            acquireQueued();
        }
        return true;
    }

    /**
     * 释放锁资源
     * @return boolean
     */
    private boolean release() {
        return tryRelease();
    }

    /**
     * 进入队列，并开始自旋。因为有可能有多个线程竞争head和tail资源，所以需要使用CAS操作确保竞争胜出者才可以继续执行后续代码。
     * 这里稍作调整，队首节点为一个虚节点，用于逻辑控制，
     * 当节点的前一个节点为对首节点，则尝试获取锁，若成功则退出自旋，若失败则继续自旋。
     * 其余阻塞节点因不再队首之后，所以始终保持自旋。
     */
    private void acquireQueued() {

        Node node = new Node();

        //入队列
        for(; ; ) {
            Node t = this.tail;
            if(t == null && HEAD_UPDATER.compareAndSet(this, null, new Node())) {
                tail = head;
            }

            if(t != null && TAIL_UPDATER.compareAndSet(this, t, node)) {
                node.prev = t;
                break;
            }
        }

        //队列自旋
        for(; ; ) {

            Node pred = node.prev;
            if(pred == head && tryAcquire()) {//前一节点为头节点，则认定为可以尝试获取锁 (退出条件)
                head = node;
                node.prev = null; // help GC
                break;
            }


        }

    }


    /**
     * 对外公布锁方法
     */
    public void lock() {
        acquire();
    }

    /**
     * 对外公布解锁方法
     */
    public void unlock() {
        release();
    }


    public static void main(String[] args) {

        final Counter c = new SpinLock.Counter();

        for(int j = 0; j < 10000; j++) {
            Thread t = new Thread(new Runnable() {
                @Override
                public void run() {
                    c.increment();
                }
            });

            t.start();
        }


    }

    public static class Counter {
        int i = 0;
        SpinLock lock = new SpinLock();

        public void increment() {
            try {
                lock.lock();
                i++;
                System.out.println(i);
            } finally {
                lock.unlock();
            }

        }
    }
}
```

测试运行后输出结果，当没有lock时，可能出现9999。加上lock后恢复正常。实际调试过程中还加了一些控制台输出，这里为了看着整洁去掉了。接下去进一步完善核心功能，因为自旋会占用cpu资源，如果需要利用自旋锁同步的代码块执行时间较长（如进行IO），则可能因为占用cpu时间太长，导致别的不需要同步的线程也受到影响，从而影响到整个系统。为了避免这个现象，接下去为SpinLock加上阻塞功能，进化为——SpinAndParkLock。



### "自旋+阻塞"队列锁——SpinAndParkLock

有了上面的SpinLock做基础，实现SpingAndParkLock变得简单很多，基本处理逻辑也可以精简成两句话：

1. 自旋时，当前Node尝试标记prev节点为Signal态，如果标记成功后，还是不能获取锁，则park（阻塞）当前线程，安心等待prev节点来通知unpark（取消阻塞）。
2. 当前Node释放锁资源时，如果发现自己的节点状态为Signal态，则说明后继节点期望被通知unpark,则此时做unpark操作。

为了实现这两句话，考虑一些必须添加的元素：

1. 每个Node节点需要增加thread属性，用于记录当前park的线程，这样当上一个节点通知的时候，才能找到对应的线程做unpark操作。
2. 每个Node节点增加waitStatus属性，用于表示当前节点的状态，如果被其后继节点更新为Signal态，则在释放锁资源时，需要对后继节点持有的线程做unpark。
3. 为了方便找到后继节点，而不用每次都从tail依据prev反向来确定后继节点，需要把原先的单向链表改造成双向链表，为每个Node节点增加next属性。

有了以上几个元素，就可以开始动手实现代码了。

```java
/**
 * Created with IntelliJ IDEA.
 * <p>
 * 自旋锁+线程park功能
 * <p>
 *
 * 1.基于SpinLock，我们要给Node添加三个属性，
 * thread：持有的线程
 * next:后继节点，因为需要通知下一个节点做unpark操作，如果从tail反向查询，效率比较低，所以加一个正向的链条
 * waitStatus:用于判断是否需要通知下一个节点做unpark操作
 *
 * @author nimo2
 * @since 2017/8/17
 */
public class SpinAndParkLock {

    /** 锁持有线程 */
    private transient Thread owner;

    /** 锁资源 */
    private volatile int state = 0;

    /** 队尾结点 用于管理节点入队列 */
    private volatile Node tail;

    /** 队首节点 用于管理节点出队列 */
    private volatile Node head;

    /** 状态属性原子更新器 */
    private static final AtomicIntegerFieldUpdater<SpinAndParkLock> STATE_UPDATER = AtomicIntegerFieldUpdater.newUpdater(SpinAndParkLock.class, "state");

    /** 队尾节点引用原子更新器 */
    private static final AtomicReferenceFieldUpdater<SpinAndParkLock, Node> TAIL_UPDATER = AtomicReferenceFieldUpdater.newUpdater(SpinAndParkLock.class, Node.class, "tail");

    /** 队首节点引用原子更新器 */
    private static final AtomicReferenceFieldUpdater<SpinAndParkLock, Node> HEAD_UPDATER = AtomicReferenceFieldUpdater.newUpdater(SpinAndParkLock.class, Node.class, "head");

    private class Node {

        public Node() {
        }

        public Node(Thread thread) {
            this.thread = thread;
        }

        /** waitStatus = -1 代表需要通知后继节点做unpark操作 */
        static final int SIGNAL = -1;

        /** 每个Node在prev节点上自旋 */
        private Node prev;

        /** 每个Node节点的next节点，用于正向推导需要unpark的节点 */
        private Node next;

        /** 持有线程 */
        private Thread thread;

        /** 节点自旋状态 -1代表正在阻塞，其他情况代表不阻塞 */
        private volatile int waitStatus;

    }

    /**
     * 尝试获取锁资源，在这里做了锁资源的写入操作，并非是线程安全的，
     * 需要CAS操作，保证当前线程是竞争胜出者。
     *
     * @return boolean
     */
    private boolean tryAcquire() {
        if(STATE_UPDATER.compareAndSet(this, 0, 1)) {
            owner = Thread.currentThread();
            return true;
        }
        return false;
    }

    /**
     * 尝试释放锁资源，如果当前线程不拥有锁资源，则抛出异常。
     * 否则，先释放owner属性，再释放锁资源。
     * 这里不需要担心锁资源的并发写入，因为此时当前线程还是锁持有者，别人无法与其竞争state的写入权限。
     *
     * @return boolean
     */
    private boolean tryRelease() {
        if(owner != Thread.currentThread()) {
            System.out.println(Thread.currentThread().getName() + "释放锁失败，持有线程:" + owner.getName());
            throw new IllegalMonitorStateException();
        }
        owner = null;
        state = 0;
        return true;
    }

    /**
     * 尝试获取锁，如果获取失败，则进入队列自旋
     *
     * @return boolean
     */
    private boolean acquire() {

        if(!tryAcquire()) {
            acquireQueued(addWaiter());
        }
        return true;
    }

    /**
     * 释放锁资源，如果发现头节点被标记为Signal态，则通知头节点的后继节点做unpark。
     *
     * @return boolean
     */
    private boolean release() {

        if(tryRelease()) {
            Node h = head;
            if(h != null && h.waitStatus == Node.SIGNAL) {
                LockSupport.unpark(h.next.thread);
                return true;
            }
        }
        return false;
    }

    /**
     * 进入队列，并开始自旋。因为有可能有多个线程竞争head和tail资源，所以需要使用CAS操作确保竞争胜出者才可以继续执行后续代码。
     * 这里稍作调整，队首节点为一个虚节点，用于逻辑控制，
     * 当节点的前一个节点为对首节点，则尝试获取锁，若成功则退出自旋，若失败则继续自旋。
     * 其余阻塞节点因不再队首之后，所以始终保持自旋。
     *
     * 自旋时，尝试标记prev节点为Signal态，如果标记成功后，
     * 还是不能获取锁，则park（阻塞）当前线程，安心等待prev节点来通知unpark（取消阻塞）。
     */
    private void acquireQueued(Node node) {


        //队列自旋
        for(; ; ) {

            Node p = node.prev;
            if(p == head && tryAcquire()) {//前一节点为头节点，则认定为可以尝试获取锁 (退出条件)
                head = node;
                node.prev = null;
                node.thread = null;
                p.next = null;// help GC
                break;
            }

            if(shouldParkAfterFailedAcquire(p)) {
                LockSupport.park(this);
            }
        }

    }

    /**
     * 添加node节点，构造方法持有当前线程，为unpark做准备
     * @return Node
     */
    private Node addWaiter() {
        Node node = new Node(Thread.currentThread());
        enq(node);
        return node;
    }

    /**
     * 入队列
     * @param node 需要入队列的节点
     */
    private void enq(Node node) {

        //入队列
        for(; ; ) {
            Node t = this.tail;
            if(t == null) { // Must initialize
                if(HEAD_UPDATER.compareAndSet(this, null, new Node())) {
                    tail = head;
                }
            } else {
                if(TAIL_UPDATER.compareAndSet(this, t, node)) {
                    node.prev = t;
                    t.next = node;
                    break;
                }
            }

        }
    }

    /**
     * 第一次标记成功后，返回false，这样能够多进行一次循环，保证调用LockSupport.park方法时，
     * 父亲节点的Signal状态一定是设置好的。
     *
     * @param pred 父亲节点
     * @return boolean
     */
    private boolean shouldParkAfterFailedAcquire(Node pred) {
        int ws = pred.waitStatus;
        if(ws == Node.SIGNAL) {
            return true;

        }
        pred.waitStatus = Node.SIGNAL;
        return false;
    }


    /**
     * 对外公布锁方法
     */
    public void lock() {
        acquire();
    }

    /**
     * 对外公布解锁方法
     */
    public void unlock() {
        release();
    }


    public static void main(String[] args) {

        final Counter c = new SpinAndParkLock.Counter();

        for(int j = 0; j < 10000; j++) {
            Thread t = new Thread(new Runnable() {
                @Override
                public void run() {
                    c.increment();
                }
            });

            t.start();
        }


    }

    public static class Counter {
        int i = 0;
        SpinAndParkLock lock = new SpinAndParkLock();

        public void increment() {
            try {
                lock.lock();
                i++;
                System.out.println(i);
            } finally {
                lock.unlock();
            }

        }
    }
}
```

写到这其实已经可以看出AQS的影子了，而且目前SpingAndParkLock应该可以投入使用了，但是AQS只做到这一步还不够。因为作为抽象实现，将tryRelease和tryAcquire方法都下放给子类实现，这样会引入新的问题。当当前线程已经作为Node节点进入队列，但是在acquireQueued方法执行tryAcquire时抛出异常，就会引起整个Node链条断裂，后继节点无法收到unpark通知，导致严重阻塞。所以AQS需要一个Node节点进入队列后线程出现异常，Node节点能够在不影响队列的情况下安全退出的方法。

### "自旋+阻塞"队列锁+异常处理

对于Node节点的安全退出策略，设计的逻辑可以描述为下面几步：

1. 当前Node尝试标记自己pred节点为Signal态时，主动检查pred节点是否是Cancel态，如果是则向前递归，直到找到一个不是Cancel态的节点，重新组合队列，剔除掉中间Cancel态的节点。
2. 当前Node节点释放锁资源，通知next节点做unpark操作时，主动检查next节点是不是Cancel态，如果是，则跳过该节点，递归直到找到一个不是Cancel态的后继节点，通知其unpark。
3. 当Node节点进入队列后的代码运行时出现异常，设置当前Node节点为Cancel态。

看似没问题了，但是漏了一个场景，假设一个Node节点A刚刚向它的prev节点B设置了Signal态，此时B节点因为tryAcquire方法异常，将自己设置成了Cancel态。结果就是，这个Cancel态的Node节点的prev节点C因为未被设置Signal态而不会知道需要通知A节点，而A节点正在安心的等待unpark。

为了应对这种特殊情况，需要在异常处理时采取一个相对保守的方案——即总是认为Cancel态的后继节点是需要做unpark操作的。当异常出现，当前Node节点被设置为Cancel态后，首先尝试为当前Node节点的next节点寻找合适的prev节点，并设置prev节点为Signal态。如果找不到合适的prev节点，则unpark next节点，让它自己按照逻辑1，重新找到一个可以承担通知责任的prev节点。

好了万事大吉，就差实现了。写到这已经写不动了，直接引用老爷子针对异常处理的代码片段。

```java
/**
 * 此方法里实现了逻辑1
 */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {//这个if块实现了重组队列，剔除掉中间Cancel态节点的作用。
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

/**
 * 此方法里实现了逻辑2
 * 让人困惑的是，这里并没有使用next属性去寻找非cancel的节点，而是通过prev逆向推导。
 * s==null的情况考虑到并发逆向推导可以理解，s.waitStatus>0也需要逆向推导就不可理解了。
 * 看了老爷子写的注释也没有说明清楚cancel态的情况为何要这么处理，如果有高人请赐教。
 * 
 */
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {//此处实现了跳过cancel态节点进行unpark通知
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}

/**
 * 此方法实现了逻辑3和为了应对特殊情况的"保守主义"实现。
 */
private void cancelAcquire(Node node) {
        // Ignore if node doesn't exist
        if (node == null)
            return;

        node.thread = null;

        // Skip cancelled predecessors
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        // predNext is the apparent node to unsplice. CASes below will
        // fail if not, in which case, we lost race vs another cancel
        // or signal, so no further action is necessary.
        Node predNext = pred.next;

        // Can use unconditional write instead of CAS here.
        // After this atomic step, other Nodes can skip past us.
        // Before, we are free of interference from other threads.
        node.waitStatus = Node.CANCELLED;

        // If we are the tail, remove ourselves.
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        } else {//这个if块中实现了"保守主义"算法，始终认为cancel态节点的后继节点
          	    //需要通知，这里为后继节点找好"继父"，如果不成功，就唤醒后继节点
          		//让它自己找。
            // If successor needs signal, try to set pred's next-link
            // so it will get one. Otherwise wake it up to propagate.
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
```

有了异常处理，这下安心多了。但是老爷子不安心于此，他又给AQS添加了一个非常常用的新特性——Share模式锁。

### "自旋+阻塞"队列锁+异常处理+Share模式锁

这一段落之前，我们一直都在描述的是Exclusive模式锁，一般被广泛的称之为互斥锁。互斥锁的特点是，当有一个线程占有锁之后，其他线程不能在占有该锁。这样，一个锁的释放机制，完全由当前占有锁的线程决定。在这一段落里，将引入一个新的模式锁——Share模式锁，它有一个大家更熟悉的术语——共享锁。在Share模式下，可以同时有多个线程占有锁，这样，一个锁的释放机制，需要由多个线程共同决定。这听起来比Exclusive模式锁稍微复杂一些，不过AQS将多线程如何协作完成锁的锁定和释放抽象为了tryAcquireShare和tryReleaseShare方法，并没有在AQS中实现，所以留在AQS中，量身为Share模式锁做的设计十分精简，概括起来就一句话——Ensure that a release propagates，确保一个Share模式的release操作能够传播下去。具体来说，可以描述为下面两步：

1. 当一个Share模式的release操作发生，如果head节点为SIGNAL态，则唤醒next节点。否则，则确保将head节点设置为PROPAGATE态。
2. 当一个Share模式的Node节点因为操作1的release操作占有锁成功，判断head节点是否为PROPAGATE态，如果为PROPAGATE态，则清楚的知道，之前发生过一个Share模式的release操作是需要确保能够传播下去的，则当前Node节点履行传播的职责，首先将自己设置为head节点，判断next节点是否是Share模式节点，如果是，则重复操作1。

这里引入的PROPAGATE态是在抽象AQS中保证健壮的传播链的必要状态。上面两句话描述的逻辑中关于PROPAGATE态的部分可以转化为下面这张示意图：

{% asset_img Share模式-Propagate态.png Share模式-Propagate态功能示意 %}

这里题外话，讨论一下PROPAGATE态存在的必要性。值得注意的是，AQS传播的是一个**release操作**。所以整个传播的触发点是必须有一个Share模式的release操作发生。如果不顾是否有release操作的发生（在操作2中不判断head的PROPAGATE态）直接进行传播工作，则Share模式下的节点会产生很多无意义的unpark操作。设计PROPAGATE态用以标记一个release操作，即是为了避免这种情况的产生。

好了，接下来看老爷子是如何实现的Share模式：

```java
	//增加了一整套xxxShared方法
    protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }

    protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }

	//这个方法实现了逻辑1的触发点，即一个Share模式的release操作
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        /*
                         * !!!!注意这一行，引入了unpark的传播，
                         * 其他代码都和SpinAndParkLock相似。!!!
                         */
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    /**
     * 这个方法实现了逻辑2，这里特殊的是，只是判断了h.waitStatus<0，因为这里包括了
     * SIGNAL和PROPAGATE两种状态，实际上都是需要传播的。具体可以看英文注释
     */
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        if (propagate > 0 || h == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }

	/**
	 * 这个方法实现了逻辑1&2中的传播细节，
	 * 确保一个即使在多个acquire和release操作并发的情况下，也能
	 * 保证release正确传播下去。代码逻辑可以解释为：
	 * 1. 如果head为SIGNAL态，不论如何，保证有且仅有一个线程正确执行
	 *    unparkSuccessor(h)操作。
	 * 2. 如果head为初始态，不论如何，确保在此方法结束时，头节点变更为
	 *    PROPAGATE态。
	 */
    private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

Share模式锁有两个经典的使用场景CountDownLatch和ReadWriteLock，我会在part 2中详细描述分析这两种场景下，线程是如何协作完成加锁和释放锁的操作。这里跳过暂且不表。AQS到目前为止已经有了自旋+阻塞通知+异常处理+共享模式，拼图终于到了最后一块。我们知道Java关键字Synchronized可以配合Object的notify和wait方法实现线程之间的协作，典型的就是生产者和消费者模型。现在，我们就要给AQS加上类似的功能——ConditionObject。

### "自旋+阻塞"队列锁+异常处理+Share模式锁+ConditionObject

CondtionObject实现的功能与Object的notify和wait相似，对应的分别是signal和wait方法。不同的是，ConditionObject是一个java实现。设计思路可以描述成以下几步：

1. CondtionObject维护一个“自旋+阻塞”队列之外的Condition队列。
2. 当线程执行wait方法，生成一个Condtion态的Node节点，进入ConditionObject维护的Condition队列。同时当前线程放弃自己占有的锁资源（包括重入的资源），之后做park操作。
3. 当线程执行signal方法，将从ConditionObject维护的Condtion队列中，取队首的Node节点压入“自旋+阻塞”队列，对Node节点对应的线程做unpark操作，调用acquireQueued方法，让其重新进入争取获得锁的流程。

上面描述的设计步骤中关于Condition队列使用的部分可以用下图解释：

{% asset_img Condition.png Condition队列工作示意图 %}

这里，需要说明的是，因为CondtionObject的使用场景限制在Exclusive模式，且只有当当前线程占有锁的情况下才可以使用CondtionObject的wait和signal方法。所以在使用signal和wait方法时，不存在并发安全问题，可以省去很多并发安全的设计。

这里截取CondtionObject中最关键的部分代码来说明CondtionObject的工作方法,其中有两处优化处理，将在代码注释中说明。

```java
static final class Node {
    ...
    //这里需要为Node新增加一个nextWaiter属性，用于维持Condition队列
    Node nextWaiter;
    ...
}
```

```java
public class ConditionObject implements Condition, java.io.Serializable {
    ...
    //首先需要firstWaiter和lastWaiter引用，来维护一个Condtion队列。
    /** First node of condition queue. */
    private transient Node firstWaiter;
    /** Last node of condition queue. */
    private transient Node lastWaiter;
    ...
    
    //实现了逻辑2
    public final void await() throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        AbstractQueuedSynchronizer.Node node = addConditionWaiter();
        int savedState = fullyRelease(node);//彻底放弃锁资源，不管重入了多少次
        int interruptMode = 0;
      	/*
      	 * 优化处理1：判断当前的Node节点是否已经在AQS列表里了，
      	 * 因为可能发生连续的wait(),和signal(),
      	 * 有可能在执行到这段代码时，当前Node节点已经计入了AQS队列，
      	 * 如果是这样就无需park操作，直接进去acquireQueued方法即可。
      	 */
        while (!isOnSyncQueue(node)) {
            LockSupport.park(this);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)//调用acquireQueued方法进入争取锁的流程。
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null) // clean up if cancelled
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
    }      
    
    //signal方法，调用doSignal方法实现逻辑3
    public final void signal() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        Node first = firstWaiter;
        if (first != null)
            doSignal(first);
    }
  	//调用transferForSignal方法实现逻辑3
    private void doSignal(AbstractQueuedSynchronizer.Node first) {
        do {
            if ( (firstWaiter = first.nextWaiter) == null)
                lastWaiter = null;
            first.nextWaiter = null;
        } while (!transferForSignal(first) &&
        (first = firstWaiter) != null);
    }
  
  	//实现逻辑3
    final boolean transferForSignal(AbstractQueuedSynchronizer.Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, AbstractQueuedSynchronizer.Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        AbstractQueuedSynchronizer.Node p = enq(node);//进入AQS队列
        int ws = p.waitStatus;
      	/*
      	 * 优化处理2: 尝试设置prev节点的Signal态,如果设置成功，
      	 * 就不需要进行unpark操作了，
      	 * 可以直接等待AQS队列中的prev节点来通知其进行unpark操作。
      	 */
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, AbstractQueuedSynchronizer.Node.SIGNAL))
            /*
             * 如果不成功，则通知wait线程去执行acquireQueud操作，
             * 进入争取获取锁的流程。
             */
            LockSupport.unpark(node.thread);
        return true;
    }

}
```

拼图终于完成了，现在，我们有了一个完成的AQS。

### 总结&预告

通过一步步做“加法”，我们拥有了一个功能完善的AQS，这个AQS具备优秀的特性，在同步块代码执行时间很短的情况下，因为存在自旋，所以不需要频繁的park/unpark线程，在同步块代码执行时间很长的情况下，因为存在阻塞，所以不会持续占用CPU资源。在自选锁和阻塞锁之间做了一个很好的权衡；做了完善的异常处理，妈妈再也不用担心线程"一睡不醒"；提供了共享锁的设计支持，能够通过AQS设计出很多优秀的共享锁；提供了Condition用于进行线程间通信，实现类似生产者和消费者模式的阻塞队列。当然，这里列出的代码并不是AQS代码的全部，但是最核心的部分，都已经在这里呈现。在part 2里，我想针对AQS的三个经典使用案例ReentrantLock,ReadWriteLock,CountDownLatch做一下代码分析。尽情期待。



