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
**lock()**
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
**tryAcquire**
通过这个方法尝试进入独占模式，该方法`公平锁`和`非公平锁`有不同的实现:
1. 在非公平锁中，如果 `state==0` 不管在同步队列中有没有等候更长时间的线程，当前线程就会直接进入独占模式
	```java
	final boolean nonfairTryAcquire(int acquires) {
	    final Thread current = Thread.currentThread();
	    int c = getState();
	    if (c == 0) {
		    //还没有线程独占该锁，当前线程直接进入独占模式
	        if (compareAndSetState(0, acquires)) {
	            setExclusiveOwnerThread(current);
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
2. 公平锁中，如果 `state==0` 并且在同步队列中没有等候时间更长的线程节点，则进入独占模式
	```java
	protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
    if (!hasQueuedPredecessors() &&compareAndSetState(0, acquires)) {
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
	```
