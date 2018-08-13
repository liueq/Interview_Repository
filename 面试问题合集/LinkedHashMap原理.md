# LinkedHashMap原理

**要点**：

使用 Dequeue 来保存 Entries 的顺序。

默认顺序是根据 insert 顺序，re-insert 不会影响该顺序。

推荐将其他任意类型的 Map 转换成 LinkedHashMap，这样能够保持该 Map 前后顺序一致。

```java
void foo(Map m){
  Map copy = new LinkedHashMap(m);
  ...
}
```

LinkedHashMap 所提供的 access-order，只会被 put, get 方法所影响。其他集合操作等都不会影响该顺序。

提供了和 HashMap 相同的方法，且性能相同。在此基础上，对于 Entry的遍历，LinkedHashMap 性能更高。

影响 LinkedHashMap 性能的两个参数是 capacity, load factor。相较于 HashMap 在初始化时会受 capacity 变大而影响性能，LinkedHashMap 几乎不会受影响。

非线程安全。

线程安全的构建方式：

```java
Map m = Collections.synchronizedMap(new LinkedHashMap());
```

对于插入顺序的LinkedHashMap来说，对于已存在 key-value 的值进行修改，并不是结构性修改。而对于访问顺序的 LinkedHashMap 来说，仅仅是 get 方法也会带来结构性修改。

对于 Iterator 采用的是 fail-fast 的策略。只要在生成 Iterator 后，Map 发生了结构性修改，再访问该 Iterator，那么就会抛出 ConcurrentModificationException。

默认的 load factor 是 0.75。

## 方法解析

> init()

在任何 Entry 被插入之前， 初始化 header。

```java
header = new Entry<>(-1, null, null, null);
header.before = header.after = header;
```

> transfer(HashMap.Entry[] newTable, boolean rehash)

将所有的 Entries 移动到一个新的  table array 中。相当于重新构建链表关系，一般用于提升遍历速度。

```java
int newCapacity = newTable.length;
for (Entry<K,V> e = header.after; e != header; e = e.after) {
    if (rehash)//是否要重新计算 hash 值
        e.hash = (e.key == null) ? 0 : hash(e.key);
    int index = indexFor(e.hash, newCapacity); //计算在 newTable 中的 index
    e.next = newTable[index];
    newTable[index] = e;
}
```

> containsValue(Object value)

判断是否是包含某个 value，这里值得注意的是对于 null 一视同仁。

```java
if (value==null) {
    for (Entry e = header.after; e != header; e = e.after)
        if (e.value==null) //即使是 null，也会返回 true
            return true;
} else {
    for (Entry e = header.after; e != header; e = e.after)
        if (value.equals(e.value))
            return true;
}
return false;
```

> get(Object key)

在 HashMap 的基础上增加 access 记录。

```java
public V get(Object key) {
    Entry<K,V> e = (Entry<K,V>)getEntry(key);
    if (e == null)
        return null;
    e.recordAccess(this); //记录下访问标记，这里的 Entry 是在 LinkedHashMap 中定义的一个新子类
    return e.value;
}
```

> Entry<K, V> extends HashMap.Entry<K, V>

在 Entry 中加入了链表结构。

```java
// 从链表中移除自己
private void remove() {
    before.after = after;
    after.before = before;
}

// existingEntry 之前，插入自己
private void addBefore(Entry<K,V> existingEntry) {
    after  = existingEntry;
    before = existingEntry.before;
    before.after = this;
    after.before = this;
}

// 将自己移动到 m 的最开头，如果 m 是根据 access-order 排序
// 移动的操作分为两步，先把自己从 m 中移除，然后把自己插入到 m header 之前
void recordAccess(HashMap<K,V> m) {
    LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
    if (lm.accessOrder) {
        lm.modCount++;
        remove();
        addBefore(lm.header);
    }
}
```

> LinkedHashIterator<T> implements Iterator<T>

自己实现了 Iterator 接口，使用链表的方式遍历。

使用 Iterator 不会对 LinkedHashMap 造成结构性变化。

```java
// 在创建 Iterator 的时候，给 exceptedModCount 赋值
// 如果在 Iterator 创建之后，LinkedHashMap 发生了任何结构性变化，那么 modeCount 就变化了，此时抛出异常。
int expectedModCount = modCount;

// 必须检测 modCount
// 移除操作成功后，对 expectedModCount 赋最新值
public void remove() {
    if (lastReturned == null)
        throw new IllegalStateException();
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    LinkedHashMap.this.remove(lastReturned.key);
    lastReturned = null;
    expectedModCount = modCount;
}

//获取下一个 Entry，同样必须先检查 modCount
Entry<K,V> nextEntry() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    if (nextEntry == header)
        throw new NoSuchElementException();
    Entry<K,V> e = lastReturned = nextEntry;
    nextEntry = e.after;
    return e;
}
```

> addEntry(int hash, K key, V value, int bucketIndex)

在 HashMap 的方法上，将 Entry 加入到链表头。

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    super.addEntry(hash, key, value, bucketIndex);
    // Remove eldest entry if instructed
    Entry<K,V> eldest = header.after;
    if (removeEldestEntry(eldest)) {
        removeEntryForKey(eldest.key);
    }
}
```

> createEntry(int has, K key, V value, int bucketIndex)

主要在 clone 和 反序列化时才使用，不会改变 table 的大小。同样是加到链表头。

```java
void createEntry(int hash, K key, V value, int bucketIndex) {
    HashMap.Entry<K,V> old = table[bucketIndex];
    Entry<K,V> e = new Entry<>(hash, key, value, old);
    table[bucketIndex] = e;
    e.addBefore(header);
    size++;
}
```

> removeEldestEntry(Map.Entry<K, V> eldest)

是否要在插入 Entry 时，删除一个最老的Entry。在 addEntry 中会被调用。

```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```



