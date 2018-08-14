# HashMap 分析

## 面试描述

**Q: 什么是HashMap**

A: 实现了 Map 接口的 Hash table 类。以键值对的方式来保存数据，因为 hash table 算法的原因，查找时间极快。

**Q: HashMap 注意事项有什么？**

A: 非线程安全，多线程时应该使用 ConcurrentHashMap。

注意 Capacity 和 Load factor，默认是 16 与 0.75.

**Q: 描述HashMap put 和 get 方法的实现过程。**

A: put 过程：对 key 调用 hashCode，计算 bucket 下标。如果碰撞，那么放到 bucket 的链表 or 红黑树中。如果链表过长超过8，也和 table 大小有关，将链表转为红黑树。

get 过程，对 key hash，找到 bucket，如果无冲突，直接获取。当冲突时，需要从链表 or 红黑树中找到（使用 key.equals()。

**Q: resize 无需重新计算 hash 的原理。**

A：resize 是将容量加倍。由于 bucket 的下标是通过 (n - 1) & hash 得到。所以，在同一 bucket 的元素，在扩容前，比如 n = 16 时，最低位的4bit 是相同的。当扩容后，bucket 下标要用最低的5bit 计算。这是多出来的最高位可能是0，也可能是1（根据 hash 值决定）。当为0时，对应的元素停留在原来的 bucket。当为1时，移动到“原索引 + oldCap”的位置。

**Q：hash 值计算的过程。**

A：

h = key.hashCode();

h = h >>> 16 ^ h; // 高16位与低16位异或，原因是减少碰撞，使得在 bucket 很小时，高位也可以参与运算。

## 源码分析

首先分析一下父类 AbstractMap：

* 实现了 Map 接口的骨架，使得继承此类能够减少工作量。
* 如果要实现一个不可改变的 Map，那么只需要实现 entrySet 方法即可。同时返回的继承了 AbstractSet 且未实现 add, remove 方法的 Set 类型。
* 如果要实现一个可改变的 Map，那么需要实现 put 方法，并且 Iterator 所返回的 entrySet 中必须有 remove 方法。

### AbstractMap 方法分析

> containsValue(Object value)

判断是否包含 value，使用 Iterator 对 entrySet 进行遍历，线性时间复杂度。

```java
public boolean containsValue(Object value) {
    Iterator<Entry<K,V>> i = entrySet().iterator();
    if (value==null) {
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            if (e.getValue()==null)
                return true;
        }
    } else {
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            if (value.equals(e.getValue()))
                return true;
        }
    }
    return false;
}
```

> containsKey(Object key)

和上一个方法同理，都是对 entrySet 进行遍历。

> get(Object key)

仍然是通过遍历 entrySet，貌似 AbstractMap 中，都是通过遍历来查找元素的。

```java
public V get(Object key) {
    Iterator<Entry<K,V>> i = entrySet().iterator();
    if (key==null) {
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            if (e.getKey()==null)
                return e.getValue();
        }
    } else {
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            if (key.equals(e.getKey()))
                return e.getValue();
        }
    }
    return null;
}
```

> public Set<K> keySet()

返回 Set 对象，这里是即时构建一个匿名的 AbstractSet 对象。

```java
public Set<K> keySet() {
    if (keySet == null) {
        keySet = new AbstractSet<K>() {
            public Iterator<K> iterator() {
                return new Iterator<K>() {
                    private Iterator<Entry<K,V>> i = entrySet().iterator();
                    public boolean hasNext() {
                        return i.hasNext();
                    }
                    public K next() {
                        return i.next().getKey();
                    }
                    public void remove() {
                        i.remove();
                    }
                };
            }
            public int size() {
                return AbstractMap.this.size();
            }
            public boolean isEmpty() {
                return AbstractMap.this.isEmpty();
            }
            public void clear() {
                AbstractMap.this.clear();
            }
            public boolean contains(Object k) {
                return AbstractMap.this.containsKey(k);
            }
        };
    }
    return keySet;
}
```

> public Collection<V> values()

同 keySet 方法，也是即时构建一个 AbstractCollection 对象。

### HashMap 简介

使用 Hash table 来实现 Map 接口。允许 null 作为键值。

基本上和 Hashtable相同，除了非同步以及允许 null 之外。

不保证顺序。

虽然 hash 算法提供了constant-time 的性能表现，在 get， put方法上。但是仍然需要注意不要设置 capacity 过大，毕竟 Iterator 遍历的时候线性的。

影响 HashMap 性能的因素有两点：capacity，load factor。

* **capacity** HashMap 的容量，表示有多少个 bucket 来存放数据（必须是2的乘方）。
* **load factor** 表示当使用量到达总容量的多少时，进行自动扩容。此时会进行 rehash，并且 capacity 提升为2倍。

Load factor 默认是 0.75，过大会导致 HashMap 在进行 hash 时更慢，而过小会占用更多空间。

如果要存储的数据本身就很多，那么尽量在一开始就设置一个较大的 capacity，避免自动扩容所带来的时间消耗。

由于 HashMap 是非同步的，所以在发生了结构性改变的时候，必须要进行同步。add，remove 是结构性改变，因为 mapping 发生了改变。如果是修改一个已存在的 key 所对应的 value 值，不算机构性改变。

在创建 HashMap 的时候进行同步的方法：

```java
Map m = Collections.synchronizedMap(new HashMap());
```

在创建 Iterator 后，除了 Iterator.remove 外所有的结构性变化，都会使得该 Iterator 不可用，抛出 ConcurrentModificationException。

### HashMap 方法解析

