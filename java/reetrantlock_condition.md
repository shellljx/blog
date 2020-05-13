### ReentrantLock 与 Condition
#### 得到 Condition对象
可以看出 `condition` 对象也是通过底层的 AQS 来获取的，而且还和锁对象使用的同一个 AQS 实例
```java
public Condition newCondition() {
    return sync.newCondition();
}
```
#### 阻塞await()
1. 首先判断当前线程是否中断，如果是则抛出`InterruptedException`
2. 把节点添加到条件等待队列
3. 把 state 值设置为 0 释放锁
4. 如果当前节点没有在同步队列(锁的等待队列)中，则调用 `park()` 方法阻塞当前线程
5. 如果线程被唤醒发现已经在同步队列中则尝试恢复锁
```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```
#### 添加节点到条件等待队列 addConditionWaiter()
1. 可以发现，每一个条件 condition 都有一个等待队列，使用 `firstWaiter` 来表示第一个节点，`lastWaiter` 来表示队列中最后一个节点
2. 如果发现最后一个节点的状态不是 `CONDITION` 则从头遍历整个等待队列删除退出等待的节点
3. 当前线程生成新的节点，并把新的节点拼接到等待队列的尾部
```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```
#### 释放锁
1. 拿到当前线程 `state` 的值然后释放并唤醒继任线程节点，这也是要调用 `await()` 方法需要先获得锁的原因之一吧
```java
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
#### 唤醒线程 signal()
1. 传入doSignal的 first 是条件等待队列的第一个节点
2. 首先把 first 的下一个节点赋值给 firstWaiter 此时指向 first 的下一个节点，也就是第二个节点,first 还是第一个节点，如果发现第二个节点为 null 说明整个等待队列就只有 first 一个节点
3. 把 first 节点的 nextWaiter 置空，移除出队列
4. 如果 first 节点不能转移到锁同步队列&&把first节点指向 firstWaiter 节点(既第二个节点)发现不为 null，则重新回到循环(第三个节点，第四个节点....)往下遍历

那现在看下什么时候 transferForSignal 方法返回 false 呢？
1. 如果原子设置节点状态从 `CONDITION` 到 0 失败，则条件等待节点 Cancell 了(可能超时可能中断)
2. 执行拼接到锁同步队列的操作(这里又回到了上一篇文章关于 ReentrantLock 获取锁的步骤)并返回前继节点
3. 如果发现前继节点是取消状态或者把前继节点状态设置成 SIGNAL 失败了，则激活这个被 signal 的线程重新执行进入锁同步队列的操作(激活后，此时被激活的线程还在await()方法体内,跳回到 `isOnSyncQueue` 方法循环)
4. 返回 true 退出第 2 步的循环
5. 而此时执行完 `signal()` 方法的线程最终释放锁，激活后继节点来尝试获得锁`acquireQueued`这又回到了锁同步队列循环体内，如果还有前继者会获取失败再次被挂起
```java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
            (first = firstWaiter) != null);
}
//把一个节点从条件等待队列移到锁的同步队列中去
final boolean transferForSignal(Node node) {

    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```