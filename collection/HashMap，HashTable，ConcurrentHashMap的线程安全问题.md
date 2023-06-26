# HashMap

HashMap是线程不安全的。
- JDK 1.7 HashMap 采用数组 + 链表的数据结构，多线程背景下，在数组扩容的时候，存在 Entry 链死循环和数据丢失问题。
- JDK 1.8 HashMap 采用数组 + 链表 + 红黑二叉树的数据结构，优化了 1.7 中数组扩容的方案，解决了 Entry 链死循环和数据丢失问题。但是多线程背景下，put 方法存在数据覆盖的问题。

# HashTable
HashTable 是线程安全的。
HashTable 容器使用 synchronized 来保证线程安全，但在线程竞争激烈的情况下 HashTable 的效率非常低下。因为当一个线程访问 HashTable 的同步方法，其他线程也访问 HashTable 的同步方法时，会进入阻塞或轮询状态。如线程1使用 put 进行元素添加，线程2不但不能使用 put 方法添加元素，也不能使用 get 方法来获取元素，所以竞争越激烈效率越低。

```java
    public synchronized V get(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return (V)e.value;
            }
        }
        return null;
    }
```

# ConcurrentHashMap
**JDK 1.7 ConcurrentHashMap 采用Segment数组 + HashEntry数组实现。** Segment 是一种可重入锁（ReentrantLock），在 ConcurrentHashMap 里扮演锁的角色；HashEntry 则用于存储键值对数据。一个 ConcurrentHashMap 里包含一个 Segment 数组，一个 Segment 里包含一个 HashEntry 数组，每个 HashEntry 是一个链表结构的元素。
![[Pasted image 20230626202112.png]]
分段锁技术将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问，能够实现真正的并发访问。

从上面的结构我们可以了解到，ConcurrentHashMap 定位一个元素的过程需要进行两次 Hash 操作。第一次 Hash 定位到 Segment，第二次 Hash 定位到元素所在的链表的头部。

这一种结构写操作的时候只对元素所在的Segment进行加锁即可，不会影响到其他的 Segment，这样，在最理想的情况下，ConcurrentHashMap 可以最高同时支持Segment数量大小的写操作（刚好这些写操作都非常平均地分布在所有的Segment上）。

这一种结构的带来的副作用是Hash的过程要比普通的 HashMap 要长。
**JDK 1.8 ConcurrentHashMap** 采用数组 + 链表 + 红黑树的方式实现，结构基本上和 1.8 中的 HashMap 一样，不过大量的利用了 volatile，final，CAS 等 lock-free 技术来减少锁竞争对于性能的影响，从而保证线程安全性。