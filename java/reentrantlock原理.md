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
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
	                //设置当前线程节点为 head 并让原 head 退出 队列
	                //然后在 gc 时被销毁
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
		//调用LockSupport.park()挂起线程，并在被唤醒后返回线程的中断状态
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
