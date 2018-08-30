# RxJava2.0 的改进

**1.对于 backpress 的支持**

RxJava 1.x 中，不能对 backpress 很好的支持，在 2.0 中得到了改进。

Rxjava 2.x 中，新增了 Flowable 的 Observable 类型，对 backpress 有了很好的支持。

**2.对于命名的更改，更加符合 Java 8**

比如 Action0, Action1, Action2, ActionN 对应的是 Action, Consumer, BiConsumer, Consumer。

**3.RxJava2 的接口都允许抛出异常**

意味着不再需要 try-cache，但是同时不再支持 null。

```java
// Action接口
public interface Action {
    void run() throws Exception;
}

// Consumer接口
public interface Consumer<T> {
    void accept(T t) throws Exception;
}

// 注：
  // 1. 这意味着，在这些方法里调用会发生异常的方法不需要try-catch
  // 2. RxJava 2.0 不再支持 null 值，如果传入一个null会抛出 NullPointerException
```

**4.其他操作符的改变**

比如 compose, subscribe 等。

**5.新增 Processor，对于 backpress 的支持**

类似于 Subject，支持 backpress。

```java
//Processor
    AsyncProcessor<String> processor = AsyncProcessor.create();
    processor.subscribe(o -> Log.d("JG",o)); //three
    processor.onNext("one");
    processor.onNext("two");
    processor.onNext("three");
    processor.onComplete();

//Subject
    AsyncSubject<String> subject = AsyncSubject.create();
    subject.subscribe(o -> Log.d("JG",o));//three
    subject.onNext("one");
    subject.onNext("two");
    subject.onNext("three");
    subject.onComplete();
```

**6.更改的 SingleObservable，Completable**

命名更加规范。

