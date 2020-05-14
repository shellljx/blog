### Rxjava 1 源码解读
冷 Observable 例子
````java
Observable.just(1, 2, 3)
        .map { it -> it + 0.1 }
        .filter { it -> it > 0 }
        .subscribe({
            System.out.println(it)
        }, {
            System.out.println(it)
        })
```

#### 1. Observable 与 OnSubscribe
Rxjava 中几乎所有的操作符都是返回一个 `Observable`，可以看到 observable 的构造方法需要传入一个 `OnSubscribe`
```java
protected Observable(OnSubscribe<T> f) {
    this.onSubscribe = f;
}
```
几乎所有的操作符都要自己对应的 `OnSubscribe` 实现，下面以 `map` 操作符为例，使用上游 `Observable` 和 map 操作符的 `Func1` 函数来构造一个 `OnSubscribeMap` 实例，从而构造一个新的 `Observable` 返回给下游
```java
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
    return unsafeCreate(new OnSubscribeMap<T, R>(this, func));
}
```
所以每个下游的 `Observable` 都会持有上游的 `Observable` 和自己的 `OnSubscribe`.
#### 2. subscribe()
而 subscribe 方法有点特殊，它根据传入的 `onNext`,`onError`,`onCompleted` 来生成一个 `Subscriber` 然后调用 `subscribe` 方法返回一个 `Subscription`
```java
public final Subscription subscribe(final Action1<? super T> onNext, final Action1<Throwable> onError, final Action0 onCompleted) {
    return subscribe(new ActionSubscriber<T>(onNext, onError, onCompleted));
}
```
简化后的 subscribe方法如下
```java
subscriber.onStart();

try {
    // allow the hook to intercept and/or decorate
    RxJavaHooks.onObservableStart(observable, observable.onSubscribe).call(subscriber);
    return RxJavaHooks.onObservableReturn(subscriber);
} catch (Throwable e) {
    subscriber.onError(RxJavaHooks.onObservableError(e))
}
```
从上面的代码可以发现，在 subscribe 方法里把最下游的 `observable` 和 `onSubscribe` 传到了 `onObservableStart` 方法里，最终调用了，最下游 `onSubscribe` 的 call 方法.里面做了3件事
1. 使用下游订阅者和自己的处理函数 Func 构造了一个自己的订阅者
2. 下游订阅者把上层的订阅关系添加到自己的订阅关系list中
3. 上游 `Observable` 数据源调用 `unsafeSubscribe` 方法继续重复 1 , 2 步骤
4. 所以 Observable 开始发射数据的地方是在订阅执行的地方
```java
public void call(final Subscriber<? super T> child) {
    FilterSubscriber<T> parent = new FilterSubscriber<T>(child, predicate);
    child.add(parent);
    source.unsafeSubscribe(parent);
}
```
最终通过层层调用，调到了最开始的发射数据的数据源,本例子中是 `just` 操作符的 `onSubscribe` 的 call 方法
```java
public void call(Subscriber<? super T> child) {
    child.setProducer(new FromArrayProducer<T>(child, array));
}
```
总结: 是一个从上往下再从下往上的执行过程
1. 经过上面的分析，发现 Rxjava 经过层层方法调用，是的下游 `Observable` 持有自己的 `onSubscribe` ，下游的 `onSubscribe` 持有上游 `Observable` 和自己的处理方法 Func.
2. 直到执行到 `subscribe()` 方法，下游的 `onSubscribe.call(subscriber)`开始执行，内部生成自己的 `subscriber` 持有下游的 `subscriber` 和 自己的处理函数 Func.
3. 然后重复上面的操作直到最上层发射数据的 `onSubscribe.call(subscriber)` 被调用
```java
public void call(Subscriber<? super T> child) {
    child.setProducer(new FromArrayProducer<T>(child, array));
}
```
从上面可以看到最上游的 `onSubscribe` 给下游的 subscriber 设置了一个数据生产者 `FromArrayProducer`,数据生产者持有下游监听者和数据，直到最下游的监听者调用了 `producer.request` 方法，producer 则发射数据，从最上游的监听者的 `onNext` 方法开始处理数据然后再调用下层监听者的 `onNext` 方法

> Observable向下onSubscribe向上subscriber向下