# RxJava2 操作符总览

| 分类            | 作用                                                         | 类型1                                                        | 类型2                                                        |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 创建操作符      | 创建 Observable ，发送事件                                   | just()<br />fromArrays()<br />fromIterable()<br />create()<br />timer()<br />interval() | intervalRange()<br />range()<br />rangeLong()<br />never()<br />empty()<br />error()<br />defer() |
| 变换操作符      | 变换 Observer 发送的事件                                     | Map()<br />FlatMap()<br />ConcatMap()<br />Buffer()          |                                                              |
| 组合/合并操作符 | 组合多个 Observable，合并需要发送的事件                      | concatArray()<br />concat()<br />mergeArray()<br />concatDelayError()<br />merge()<br />mergeDelayError()<br />combineLastest() | combineLastestDelayError()<br />reduce()<br />collect()<br />startWithArray()<br />startWith()<br />Zip()<br />count() |
| 功能性操作符    | 辅助 Observable 在发送事件时实现一些功能性需求（线程切换，错误处理） | subscribe()<br />subscribeOn()<br />observeOn()<br />delay()<br />onErrorReturn() | onErrorResumeNext()<br />onExceptionResumeNext()<br />retry()<br />retryUntil()<br />retryWhen() |
| 过滤操作符      | 过滤/筛选 Observable 发送的事件，Observer 接收的事件         | distinctUntilChanged()<br />takeLast()<br />throttleFirst()<br />throttleLast()<br />throttleWithTimeout()<br />firstElement()<br />lastElement()<br />elementAt() | elementAtOrError()<br />take()<br />debounce()<br />skipLast()<br />distinct()<br />sample()<br />ofType()<br />filter()<br />skip() |
| 条件/布尔操作符 | 通过设置函数，判断 Observable 发送的事件是否符合条件         | contains()<br />exists()<br />isEmtpy()<br />amb()<br />all() | takeWhile()<br />takeUntil()<br />skipWhile()<br />skipUntil()<br />defaultIfEmpty()<br />SequenceEqual() |

## 创建操作符

应用场景：完整，快速创建 Observable；定时操作；周期性操作；数组，集合遍历。按照此分类，可将创建操作符分为3类：

实际开发需求：网络请求轮询

**基本创建** 

* create()

**快速创建，发送事件**

* just()
* fromArray()
* fromIterable()
* never()
* empty()
* error()

**延迟创建**

* defer()
* timer()
* interval()
* intervalRange()
* range()
* rangeLong()

## 变换操作符

对事件序列中的事件进行加工处理，使其变成不同的事件或不同的事件序列。

应用场景：嵌套回调

实际应用场景：网络请求嵌套回调

**常见变换操作符**

* Map()
* FlatMap()
* ConcatMap()
* Buffer()

## 组合/合并操作符

组合多个 Observable，合并需要发送的事件。

应用场景：组合多个 Observable，合并多个事件，发送事件前追加事件，统计数量。

实际应用场景：合并数据源&同时展示，获取缓存数据，联合判断。

**组合多个 Observable**

* 按发送顺序：concat(), concatArray()
* 按时间：merge(), mergeArray()
* 错误处理：concatDelayError(), mergeDelayError()

**合并多个事件**

* 按数量：Zip()
* 按时间：combineLastest(), combineLatestDelayError()
* 合并成一个事件发送：reduce(), collect()

**发送事件前追加发送事件**

* startWith()
* startWithArray()

**统计发送事件数量**

* count()

## 功能性操作符

辅助完成 Observable 发送事件时的一些需求。

实际应用场景：线程操作，发送网络请求时的差错重试机制，轮询。

**连接 Observable，Observer**

* subscribe()

**线程调度**

* subscribeOn()
* observeOn()

**延迟操作**

* delay()

**在事件声明周期中的操作**

* do()

**错误处理**

* onErrorReturn()
* onErrorResumeNext()
* onExceptionResumeNext()
* retry()
* retryUntil()
* retryWhen()

**重复发送操作**

* repeat()
* repeatWhen()

## 过滤操作符

过滤/筛选 Observable 发送的事件，Observer 接收的事件。

实际应用场景：功能防抖，联想搜索优化。

**根据指定条件过滤事件**

* Filter()
* ofType()
* skip()
* skipLast()
* distinct()
* distinctUntilChanged()

**根据指定事件数量过滤事件**

* take()
* takeLast()

**根据指定事件过滤事件**

* throttleFirst()
* throttleLast()
* Sample()
* throttleWithTimeout()
* debounce()

**根据指定事件位置过滤事件**

* firstElement()
* lastElement()
* elementAt()
* elementAtOrError()

## 条件/布尔操作符

通过设置函数，判断 Observable 发送的事件是否符合条件。

应用场景：本地判断发送数据事件。

实际应用场景：对发送数据事件进行控制。