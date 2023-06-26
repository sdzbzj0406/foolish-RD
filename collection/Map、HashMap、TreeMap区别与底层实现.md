## 本质区别

先看HashMap的定义：

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```

再看TreeMap的定义：

```java
public class TreeMap<K,V>
    extends AbstractMap<K,V>
    implements NavigableMap<K,V>, Cloneable, java.io.Serializable
```

从类的定义来看，HashMap和TreeMap都继承自AbstractMap，不同的是HashMap实现的是Map接口，而TreeMap实现的是NavigableMap接口。NavigableMap是SortedMap的一种，实现了对Map中key的排序。

这样两者的第一个区别就出来了，==TreeMap是排序的而HashMap不是。==
再看看HashMap和TreeMap的构造函数的区别。

```java
public HashMap(int initialCapacity, float loadFactor) 
```

HashMap除了默认的无参构造函数之外，还可以接受两个参数initialCapacity和loadFactor。

HashMap的底层结构是Node的数组：

```java
transient Node<K,V>[] table
```

initialCapacity就是这个table的初始容量。如果大家不传initialCapacity，HashMap提供了一个默认的值：

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```

当HashMap中存储的数据过多的时候，table数组就会被装满，这时候就需要扩容，HashMap的扩容是以2的倍数来进行的。而loadFactor就指定了什么时候需要进行扩容操作。默认的loadFactor是0.75。

```java
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

再来看几个非常有趣的变量：

```java
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64;
```

上面的三个变量有什么用呢？在java 8之前，HashMap解决hashcode冲突的方法是采用链表的形式，为了提升效率，java 8将其转成了TreeNode。什么时候会发送这个转换呢？

这时候就要看这两个变量TREEIFY_THRESHOLD和UNTREEIFY_THRESHOLD。

TREEIFY_THRESHOLD为什么比UNTREEIFY_THRESHOLD大2呢？看源代码的话，用到UNTREEIFY_THRESHOLD时候，都用的是<=,而用到TREEIFY_THRESHOLD的时候，都用的是>= TREEIFY_THRESHOLD - 1，所以这两个变量在本质上是一样的。

MIN_TREEIFY_CAPACITY表示的是如果table转换TreeNode的最小容量，只有capacity >= MIN_TREEIFY_CAPACITY的时候才允许TreeNode的转换。

TreeMap和HashMap不同的是，TreeMap的底层是一个Entry：

```java
private transient Entry<K,V> root
```

他的实现是一个红黑树，方便用来遍历和搜索。**插入、查询和删除时间复杂度都是O(logn)**，但是删除和插入后可能需要调整树结构满足红黑树的规则，需要耗费性能

TreeMap的构造函数可以传入一个Comparator，实现自定义的比较方法。

```java
public TreeMap(Comparator<? super K> comparator) {
        this.comparator = comparator;
    }
```

如果不提供自定义的比较方法，则使用的是key的natural order。

## 排序区别
TreeMap输出的结果是排好序的，而HashMap的输出结果是不定的。

## Null值的区别
HashMap可以允许一个null key和多个null value。而TreeMap不允许null key，但是可以允许多个null value。

## 性能区别
HashMap的底层是Array，所以HashMap在添加，查找，删除等方法上面速度会非常快。而TreeMap的底层是一个Tree结构，所以速度会比较慢。

另外HashMap因为要保存一个Array，所以会造成空间的浪费，而TreeMap只保存要保持的节点，所以占用的空间比较小。

HashMap如果出现hash冲突的话，效率会变差，不过在java 8进行TreeNode转换之后，效率有很大的提升。

TreeMap在添加和删除节点的时候会进行重排序，会对性能有所影响。

# 共同点

两者都不允许duplicate key,两者都不是线程安全的。