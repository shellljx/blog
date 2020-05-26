### Thread Join 方法

在线程 A 上调用线程 B 的 `join` 方法会使得线程 A 阻塞直至线程 B 执行完毕

从 `join` 方法的源码可以看出，该方法使用是一个同步方法，持有线程 B 实例的锁，调用 `this.wait` 方法阻塞线程 A，`this.isAlive` 是退出循环的条件，当线程 B 终止后自动调用 `notifyAll` 方法唤醒线程 A

```java
public final synchronized void join(long millis)
throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (millis == 0) {
        while (isAlive()) {
            wait(0);
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }
}
```

 