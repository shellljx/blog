### ReentrantLock原理
#### 1.构造函数
构造参数,通过不同构造参数可以构造`公平锁`或者`非公平锁`
```java
//默认构造参数为非公平锁
ReentrantLock(){
    sync = new NonfairSync();
}

//带参构造函数支持创建公平锁
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```
#### 2.lock()
通过该方法获取锁，如果compareAndSetState返回false表示当前state的值已经不为0了，有线程(可能是当前线程本身)已经在前面获取到了锁,反之则表示还没有其他线程获取锁
```java
final void lock() {
    //当前线程成功获得了独占模式 
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        //在当前线程调用lock之前已经有线程(可能是当前线程)获得了锁
        acquire(1);
}
```
```java
public final void acquire(int arg) {
    //再次尝试获取独占模式并且如果获取失败则进入队列等待
    if (!tryAcquire(arg) &&acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
#### 3.tryAcquire
通过这个方法尝试进入独占模式，该方法`公平锁`和`非公平锁`有不同的实现:
1. 在非公平锁中，如果 `state==0` 不管在同步队列中有没有等候更长时间的线程，当前线程就会直接进入独占模式
2. 公平锁中，如果 `state==0` 并且在同步队列中没有等候时间更长的线程节点，则进入独占模式
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        //还没有线程独占该锁，当前线程直接进入独占模式
        if (compareAndSetState(0, acquires)) {
             etExclusiveOwnerThread(current);
            return true;
        }
    }
    //如果当前线程和独占模式的线程是同一个线程，则直接 state+1，这就是重入
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        //只有当前线程在运行，所以直接调用 setState 更新到最新值
        setState(nextc);
        return true;
    }
    return false;
}
```
```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        //如果在同步队列中没有等待的线程&&设置 state 成功则进入独占模式
        if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //如果当前线程是独占模式的线程，则直接 state+1
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
#### 4.addWaiter
如果`tryAcquire` 返回 false 表示尝试进入独占模式失败，则把当前线程生成节点添加到同步队列的队尾
#### 5.acquireQueued
线程节点添加到队列尾部之后传入该方法，这个方法里面是一个死循环，流程如下:
1. 获取前面一个节点，如果前面一个节点就是 head 则尝试进入独占模式(因为 head 节点就是当前你独占锁的线程节点，有可能head已经释放了锁，state = 0 了，所以当前线程要去尝试进入独占模式)，如果当前线程成功的进入独占模式，则把自己设置成 head.如果进入独占模式失败(在非公平锁的情况可能有其他线程抢先又进入了独占模式设置 state = 1)则经过一系列的判断最终调用`LockSupport.park()`将当前线程阻塞住，等待着中断或者`LockSupport.unpark()`再将自己唤醒重新进入循环，直到自己成为 head 退出循环
```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //当前线程能够执行for循环，说明当前线程还没进入阻塞或者被唤醒
                //如果是被唤醒说明当前线程节点的前一个就是 head
                //这个时候调用 tryAcquire 尝试进入独占模式
                //如果成功进入独占模式就变成了head并退出循环，lock()方法就执行结束了
                //如果进入独占模式失败说明有其他线程抢先进入独占模式了，当前线程再重新进入阻塞状态，这就是非公平锁的情况下线程可能会重复的被唤醒挂起
                //如果某一个阻塞线程超时退出阻塞也会唤醒下一个等待节点，下一个等待节点又会重新循环尝试进入独占模式，如果发现失败就把它前一个cancelled的节点移除队列并再次进入阻塞,这种也会造成阻塞线程被重复唤醒挂起
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    //设置当前线程节点为 head 并让原 head 退出 队列
                    //然后在 gc 时被销毁
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    //lock()对中断的处理并不会抛出异常，只是设置了一个标记位
                    //lockInterruptibly()方法，这里会抛出中断异常来响应中断
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
#### 6.shouldParkAfterFailedAcquire
```java
    //如果返回 true 则当前线程节点可以阻塞
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        //队列中前一个节点的状态
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            //如果前一个节点的状态是释放锁后唤醒后一个节点，则当前线程则可以安全的等待被唤醒
            return true;
        if (ws > 0) {
            //如果前一个节点是 cancelled 表示前面的等待线程可能因为超时或者响应了中断取消了继续阻塞，
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * 前一个线程节点是 0 或者 PROPAGATE.  表示当前线程节点
             * 需要前一个节点释放锁后来唤醒当前线程，所以把前一个节点设置成 SIGNAL
             * 状态，然后返回 false 重新进入循环尝试进入独占模式
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
#### 7.unlock()
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        //头节点不为 null && 头节点状态不为初始状态
        if (h != null && h.waitStatus != 0)
            //唤醒同步队列里的排队线程
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
#### 8.tryRelease()
```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        //如果 state 减少到 0 则退出了独占模式
        //返回 true 也就要释放锁了，要去唤醒同步队列中的下一个线程
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```
#### 9.lockInterruptibly()
```java
private void doAcquireInterruptibly(int arg){
    try {
        for (;;) {
            //省略部分代码
            //该方法进入阻塞后，如果该线程发生了中断，退出阻塞并且抛出InterruptedException异常
            //最后执行cancelAcquire()方法
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
#### 10.cancelAcquire()
```java
private void cancelAcquire(Node node) {
    // 过滤为null的节点
    if (node == null)
        return;
    //清除节点绑定的 thread
    node.thread = null;

    // 略过已经被cancelled的前继节点
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    Node predNext = pred.next;

    //把要取消等待的节点状态设置成CANCELLED
    node.waitStatus = Node.CANCELLED;

    // 如果要取消等待的节点是尾节点，则把前面得到的最后面非Cancelled节点通过CAS设置成尾结点
    if (node == tail && compareAndSetTail(node, pred)) {
        //如果成功，则通过CAS把尾节点的 next 置空
        compareAndSetNext(pred, predNext, null);
    } else {
        //如果取消等待的前一个节点不是 head则进一步判断(前节点状态是SIGNAL吗？如果不是则CAS设置状态为SIGNAL)&&前节点绑定的线程不为空
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
                (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            //满足外面的判断条件后直接把取消节点的next通过CAS拼接到pred后面
            //如果拼接失败可以不进行其他操作，说明和其他线程操作竞争失败了
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            //如果要取消等待的节点是head的后继节点||要取消等待节点的前节点状态不是SIGNAL，则尝试激活要取消节点的后继节点
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```
通过对`cancelAcquire`方法的分析可以得出如下结论:
1. 如果取消等待的节点是尾节点，则把前面非CANCELLED的节点设置成尾节点
2. 如果取消等待的节点不是头节点的后继节点或者取消等待的节点的前节点状态最终是SIGNAL则把待取消等待节点从队列中剔除
3. 如果待取消节点是head的后继节点或者待取消节点的前节点不是SIGNAL，则尝试激活待取消节点的后继节点(如果激活的后继节点仍没能独占锁则再次进入阻塞状态)，所以会造成线程反复激活阻塞
#### 9.unparkSuccessor()
```java
private void unparkSuccessor(Node node) {

    int ws = node.waitStatus;
    if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

    //大部分情况待唤醒的后继节点就是node的下一个节点
    //为什么要从尾遍历呢？是为了在极端情况下能够遍历整个队列而不出现遗漏
    //比如enq方法中新节点入队compareAndSetTail(t, node)只能保证设置tail是原子性的但是不能保证入队的整个操作是原子性的 t.next = node 还没有执行的时候就遍历不到新入队的tail
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        //最后调用unpark来唤醒线程
        LockSupport.unpark(s.thread);
}
```
#### 10.总结
1. `Reentrant` 实现锁是通过 `AQS` 实现的，通过设置 state 的值来保持独占模式，重入的话 state 的值加 1
2. 必须调用 `unlock` 来手动释放锁
3. 支持响应中断，超时，尝试获取锁
4. 支持公平锁和非公平锁，公平锁. 非公平锁如果当前线程发现没有其他线程独占就会直接进去独占模式。公平锁还要检查有没有后继者，如果有就先入队