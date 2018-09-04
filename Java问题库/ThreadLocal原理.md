# ThreadLocal 原理

ThreadLocal 内部实际上是通过 ThreadLocalMap 来进行变量的保存。其中使用了当前线程 Thread t 作为 key，而要保存的变量作为 value。这样虽然 ThreadLocalMap 是多线程共有的。但是每个线程只取出自己的 value。

应该是每一个 ThreadLocal 变量都对应一个 ThreadLocalMap。
