# volatile 关键字用法

> 自己立即想到的内容

```
volatile 所修饰的 field，对于所有线程都是可见的，总能返回最新值。

volatile 可以阻止JVM 重排序。
```

> 修正后

```
volatile 所修饰的 field，对于所有线程都是可见的，总能返回最新值。但是不具有原子性。

volatile 可以阻止JVM 重排序。

当一个变量没有涉及到其他状态变量的不变约束时，才声明为 volatile，否则是 synchronized。

原理是使用 memory barrier 来刷新缓存。

volatile 用来修饰数组的时候，仅仅是对数组的引用提供了 volatile 语义，其成员并没有。
```