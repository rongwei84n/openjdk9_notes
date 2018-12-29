Title:HashMap

---



# 一. 问题切入点

## 1. 怎么根据key找到对应的存储地址?

## 2. 哈希因子是用来干嘛,默认多少?

## 3. 初始容量默认多少？

## 4. 冲突了怎么办?

## 5. 扩容在什么情况下发生?怎么扩容?

## 6. 为什么Hashmap的长度是2的次幂？

## 7. 为什么说重写equals的同时重写hashcode方法？

## 8. 多线程下面使用Hashmap会怎么样?

## 9. Hashmap使用了什么样的hash算法

如果你能回答这些问题了，那么就不用接着看了。如果不能的话，我们可以看，最后我会总结出这些问题的答案。



# 二. Hashmap介绍

hashmap采用键值对的方式存储数据，非线程安全，如果hash碰撞低的情况下，查询效率非常高，可以达到o(1).

如果发生hash碰撞，会转成链表。但是链表过长的话查询效率就会很低，所以当链表成员个数大于8的时候，就会转成红黑树。

扩容拆分红黑树的时候，如果拆分后的树节点小于6个，那么树就会重新转化成链表。



# 三. 代码分析

# 3.1 Hashmap成员变量

```
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

默认初始化容量，16
```



```
static final int MAXIMUM_CAPACITY = 1 << 30;
最大容量
```



```java
static final float DEFAULT_LOAD_FACTOR = 0.75f;

哈希因子，默认0.75,它的作用是当实际容量达到当前容量的多少时进行扩容，默认0.75就是当达到75%使用率的时候就需要扩容了。

那太大或者太小有什么影响呢？

1. 太小-频繁扩容，频繁rehash，影响性能。这个很好理解，比如你的空间16，哈希因子0.1，那用了1个元素就需要扩容了，其他还有15个空间没有使用呢。
2. 太大-迟迟不扩容，导致有限的空间存储了大量的key，那么肯定就很容易冲突。比如你的空间是4，然后哈希因子1千万，那我们插入一万个元素也不会扩容。
所以我们1万个元素就会放到这4个空间里面，那肯定会冲突很严重(平均一个空间存2500个元素)，而hashmap处理冲突是把冲突的元素放入一个链表(java8开始当冲突元素大于8个时改成红黑树)，那对插入和查询的性能影响还是很大的。

至于0.75则是一个经验值。
```



```java
static final int TREEIFY_THRESHOLD = 8;

冲突后元素链表转成红黑树的阈值，某个槽位当超过8个元素时，转化成红黑树，提高性能。
因为链表过长的话，查询效率低，链表的查询复杂度是o(n)，需要从头遍历到尾。
```



```
static final int UNTREEIFY_THRESHOLD = 6;
扩容后，需要把红黑树转化成链表的阈值，如果扩容拆分红黑树后节点小于6，那么就转化成链表。
```



```
static final int MIN_TREEIFY_CAPACITY = 64;
树形最小容量，当要转化成一个红黑树时，如果发现Hashmap的容量小于64，那么就要扩容。
```



```
transient Node<K,V>[] table;
存储的Hashmap元素
```



```
transient Set<Map.Entry<K,V>> entrySet;
缓存的键值对集合
```



```
transient int size;
Hashmap里面真正存储的元素个数
```



```
transient int modCount;
修改次数，用于快速失败，防止多线程操作Hashmap
```



## 3.2 put(K key, V value)操作

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

3.2.1 首先做hash操作，拿出key的hashcode()，然后执行>>>16操作。为什么？

我的理解

1. 作为系统设计者，它不能完全相信它的用户，万一用户传入的hash过于简单怎么办？所以它需要进行加工。
2. 然后拿高16位和低16位做亦或运算，这个算法的具体解释可以看最后4.9


```
^(亦或操作，如果两个数的二进制，相同位数只有一个是1，则该位结果是1，否则是0)
```




3.2.2 拿到hash之后，执行putVal操作，继续看.

```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0) { //@@ 1. HashMap has not init yet
        n = (tab = resize()).length;
    }

    //@@ 2. (n-1) & hash equals hash % n, but has better perference
    //if no value at point position, then put it here.
    if ((p = tab[i = (n - 1) & hash]) == null) {
        tab[i] = newNode(hash, key, value, null);
    } else {
        //@@ 3. if there is a value already, maybe has two chances:
        //a. same key, replease.
        //b. different key, hash collision!!!
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k)))) {
            //@@ 4. same key, replease.
            e = p;
        } else if (p instanceof TreeNode) {
            //@@ 5. It's a TreeNode already, add value to Tree.
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        } else {
            //@@ 6. collision-> add value into linked list
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) {// -1 for 1st
                        //@@ 7.If list size > threshold(8), change to tree.
                        treeifyBin(tab, hash);
                    }
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k)))) {
                    break;
                }
                p = e;
            }
        }
        if (e != null) { // @@ existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //@@ 8. used as fast failed, multi thread access hashmap or list map when add/delete map.
    ++modCount;
    //@@ 9. add size
    if (++size > threshold) {
        resize();
    }
    afterNodeInsertion(evict);
    return null;
}
```

3.2.3 如果hashmap是第一次使用，还没有初始化，那就初始化

```
if ((tab = table) == null || (n = tab.length) == 0) { //@@ 1. HashMap has not init yet
    n = (tab = resize()).length;
}
```

3.2.4 直接调用resize()进行扩容，那我们来看看resize()的逻辑。

```
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        } else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY) {
            newThr = oldThr << 1; // double threshold
        }
    }
    else if (oldThr > 0) {// initial capacity was placed in threshold
        newCap = oldThr;
    } else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

先明白几个变量：

oldCap-扩容前的容量

oldThr-扩容前的阈值

newCap-扩容后的容量

newThr-扩容后的阈值，如果到达了这个阈值就需要再次扩容

3.2.5 我们来看看扩容的判断逻辑

```
if (oldCap > 0) {
    if (oldCap >= MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return oldTab;
    } else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
             oldCap >= DEFAULT_INITIAL_CAPACITY) {
        newThr = oldThr << 1; // double threshold
    }
}
else if (oldThr > 0) {// initial capacity was placed in threshold
    newCap = oldThr;
} else {               // zero initial threshold signifies using defaults
    newCap = DEFAULT_INITIAL_CAPACITY;
    newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
}
```

如果oldCap>0，说明以前初始化，然后判断是否大于最大容量，如果是那么把阈值设置成Integer.MAX_VALUE;

如果还没达到最大容量，那么容量翻倍。

```
newThr = oldThr << 1;
```

如果oldThr > 0，说明初始化Hashmap的时候指定了容量，因为threshold在指定了容量初始化Hashmap就会初始化

```
public HashMap(int initialCapacity, float loadFactor) {
        ...
        this.threshold = tableSizeFor(initialCapacity);
    }
```

如果不指定参数初始化Hashmap，那么不会初始化threshold，如下:

```
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

所以如果这种情况的话，就把阈值设置成threshold

```
int oldThr = threshold;
else if (oldThr > 0) {// initial capacity was placed in threshold
    newCap = oldThr;
} 
```

最后就是默认初始化的逻辑，初始化成默认容量和阈值。

```
else {               // zero initial threshold signifies using defaults
    newCap = DEFAULT_INITIAL_CAPACITY;
    newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
}
```

newCap = 16，newThr = 16*0.75 = 12



3.2.6 如果newThr == 0

```
if (newThr == 0) {
    float ft = (float)newCap * loadFactor;
    newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
              (int)ft : Integer.MAX_VALUE);
}
```

如果newThr == 0 肯定是不对，大家想想newThr是干啥的？是扩容的阈值，如果等于0，那不是要一直扩容么。

所以加个判断newThr == 0，如果真的不幸发生了，那么就用新容量 * 哈希因子。

到这里的话，新的容量计算逻辑完成了，那么大家猜猜接下来需要干什么呢？没错，就是需要拷贝原来Hashmap的元素了。

3.2.7 先判断原来的Hashmap是不是null，然后for循环遍历里面的元素.

```
if (oldTab != null) {
    for (int j = 0; j < oldCap; ++j) {
    	Node<K,V> e;
        if ((e = oldTab[j]) != null) {
        	...
        }
    }
}
```

如果oldTab[j] == null，说明这个槽位没有数据，那就跳过，如果 != null才需要拷贝，所以继续往下看

```
oldTab[j] = null;
if (e.next == null)
    newTab[e.hash & (newCap - 1)] = e;
else if (e instanceof TreeNode)
    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
else {
	...
}
```

3.2.8 如果e.next == null，那说明什么？

说明当前这个槽位没有冲突，只有一个元素，那么拿e.hash和newCap-1做&运算，由于newCap是2的次幂，所以newCap - 1的二进制就是一串连续的1，比如

```
newCap = 2 (newCap - 1 = 1) 二进制

newCap = 4 (newCap - 1 = 11)

newCap = 8 (newCap - 1 = 111)

newCap = 16 (newCap - 1 = 1111)

newCap = 32 (newCap - 1 = 11111)

所以e.hash和newCap - 1做&运算的话，就是把过滤掉e.hash大于newCap的部分，保留低位。
比如e.hash = 22,二进制就是10110,newCap = 16，那么&之后就是

10110
01111
--------
00110
把高位的1过滤掉了，10110->00110，十进制就是6，所以22在newCap=16的时候就落在第6个槽位。

那在newCap扩容到32的时候，就会这样计算
10110
11111
--------
10110
由于32>22，所以高位被保留了，所以在newCap=32的时候，22就落在22槽位
```

那回到上面的话题，e.hash & newCap -1的意思就是如果在原来容量的hash就会保留在原来的槽位，如果大于原来的容量的hash，就会通过高位保留的方法，放到扩容的槽位上(比如22，原来在第6槽位，扩容后在22槽位)。

3.2.9 如果e instanceof TreeNode表示什么？

如果这种情况，那么说明当前节点冲突大于8，已经转化成红黑树了。

如果这种情况下，那应该怎么办呢？

这个时候需要处理红黑树里面每个元素，重新hash，因为扩容后，本来冲突的key可能不会冲突了。比如上面22和6在newCap=16的时候，都在6这个槽位。现在扩容到newCap = 32之后，那么22就到22这个槽位了，和6就不冲突了。

同时，如果拆分红黑树之后冲突的节点不到6个了，那么就需要把红黑树重新调整成链表。

所以需要拆分红黑树，jdk源代码的名字也是用split来表达这个意思。

```
else if (e instanceof TreeNode)
    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
```



3.2.10 拆分红黑树

```
/**
 * Splits nodes in a tree bin into lower and upper tree bins,
 * or untreeifies if now too small. Called only from resize;
 * see above discussion about split bits and indices.
 *
 * @param map the map
 * @param tab the table for recording bin heads
 * @param index the index of the table being split
 * @param bit the bit of hash to split on
 */
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    TreeNode<K,V> b = this;
    // Relink into lo and hi lists, preserving order
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (TreeNode<K,V>)e.next;
        e.next = null;
        if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }

    if (loHead != null) {
        if (lc <= UNTREEIFY_THRESHOLD)
            tab[index] = loHead.untreeify(map);
        else {
            tab[index] = loHead;
            if (hiHead != null) // (else is already treeified)
                loHead.treeify(tab);
        }
    }
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}
```

split以oldCap为基准，把红黑树拆分两部分(low和high)，low是指还会落在原来那个槽位的，high是指移动到新槽位的，判断依据是(e.hash & bit) == 0 , bit代表oldCap，如果==0就是低位

以oldCap = 16(10000)为例，有22和6两个key，那么

```
22:
10110
10000
----------
10000
所以22落在高位

6:
00110
10000
----------
00000
所以6落在低位
```

lc- low count，代表低位冲突元素的个数

hc-high count, 代表高位冲突元素的个数

```
if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
```

如果拆分后冲突的元素<=6，那么就还原成链表，否则继续当作一个红黑树。

```
if (loHead != null) {
    if (lc <= UNTREEIFY_THRESHOLD)
        tab[index] = loHead.untreeify(map);
    else {
        tab[index] = loHead;
        if (hiHead != null) // (else is already treeified)
            loHead.treeify(tab);
    }
}
if (hiHead != null) {
    if (hc <= UNTREEIFY_THRESHOLD)
        tab[index + bit] = hiHead.untreeify(map);
    else {
        tab[index + bit] = hiHead;
        if (loHead != null)
            hiHead.treeify(tab);
    }
}
```



3.2.11 如果槽位是一个链表怎么处理？

判断完上面的逻辑后(单个元素，红黑树)，剩余的就只能是一个链表了，那怎么处理呢？

我们思考下，与3.2.10对比，逻辑应该是类似的，拆分链表到高位和低位两个链表，但是不同的是，不需要进行红黑树和链表的转换了。

因为本来就是链表，拆分链表后长度肯定不会大于原来的长度，所以肯定达不到重构成红黑树的条件。

于是do…while循环遍历链表，然后拆分到高位和低位链表即可。

```
else { // preserve order
    Node<K,V> loHead = null, loTail = null;
    Node<K,V> hiHead = null, hiTail = null;
    Node<K,V> next;
    do {
        next = e.next;
        if ((e.hash & oldCap) == 0) {
            if (loTail == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
        }
        else {
            if (hiTail == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
        }
    } while ((e = next) != null);
    if (loTail != null) {
        loTail.next = null;
        newTab[j] = loHead;
    }
    if (hiTail != null) {
        hiTail.next = null;
        newTab[j + oldCap] = hiHead;
    }
}
```

到此，resize逻辑就讲完了，当然如果是第一次初始化的话，我们oldTab就是null了，所以不需要走拷贝元素的逻辑。

其实，我们也可以看出，resize的工程挺繁重的，特别是需要重构红黑树以及链表。

所以，我们要选择合理Hashmap大小以及哈希因子。

那，继续回到putVal的逻辑。



3.2.12 没有冲突的插入值

最开心的事情就是插入元素的时候发现没有冲突(指定的位置 == null)，直接插入即可。

```
if ((p = tab[i = (n - 1) & hash]) == null) {
    tab[i] = newNode(hash, key, value, null);
} 
```

算法是(n -1) & hash，n是Hashmap的容量，一般是2的次幂，比如16.

这个算法上面已经讲过了，所以不在重复(请参考上oldCap = 16, hash=22或者6的例子)



3.2.13 有冲突怎么办？

有冲突得分两种情况：

1.一种是插入同样的key。比如用户连续两次put同一个key，那简单，只要用后面的覆盖前面的。

2.还有一种就是真的冲突了，不同的key计算出同一个槽位，比如oldCap = 16的时候，22与6两个key，这个时候如果是红黑树那么就把节点插入红黑树；如果是链表，就插入链表但是如果插入后长度大于8，那么就要转化成红黑树。

代码如下:

```
else {
    //@@ 3. if there is a value already, maybe has two chances:
    //a. same key, replease.
    //b. different key, hash collision!!!
    Node<K,V> e; K k;
    if (p.hash == hash &&
        ((k = p.key) == key || (key != null && key.equals(k)))) {
        //@@ 4. same key, replease.
        e = p;
    } else if (p instanceof TreeNode) {
        //@@ 5. It's a TreeNode already, add value to Tree.
        e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
    } else {
        //@@ 6. collision-> add value into linked list
        for (int binCount = 0; ; ++binCount) {
            if ((e = p.next) == null) {
                p.next = newNode(hash, key, value, null);
                if (binCount >= TREEIFY_THRESHOLD - 1) {// -1 for 1st
                    //@@ 7.If list size > threshold(8), change to tree.
                    treeifyBin(tab, hash);
                }
                break;
            }
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                break;
            }
            p = e;
        }
    }
    if (e != null) { // @@ existing mapping for key
        V oldValue = e.value;
        if (!onlyIfAbsent || oldValue == null)
            e.value = value;
        afterNodeAccess(e);
        return oldValue;
    }
}
```



3.2.14 快速失败

众所周知，Hashmap是非线程安全的，如果允许多线程同时操作会产生不可预知的错误，比如两个线程同时put同一个key.

map.put("a", 1);

map.put("a",2);

那么最后结果到底是1呢还是2呢？那是不可预知的，得看线程的调度顺序。所以Hashmap用一个变量modCount(修改次数)来快速失败。

```
//@@ 8. used as fast failed, multi thread access hashmap or list map when add/delete map.
++modCount;
```

实现思路也很简单，需要快速失败的地方操作之前记录下modCount的值，操作之后再来读取modCount，如果两个不相等，说明这个过程中，有其他线程操作了Hashmap，那么就抛出ConcurrentModificationException。

比如遍历Hashmap的时候执行putVal操作：

```
public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
    Node<K,V>[] tab;
    ...
    if (size > 0 && (tab = table) != null) {
        int mc = modCount;
        for (Node<K, V> e : tab) {
            for (; e != null; e = e.next)
                action.accept(e);
        }
        if (modCount != mc)
            throw new ConcurrentModificationException();
    }
}
```



3.2.15 是否需要扩容

插入一个元素后，如果当前Hashmap的元素个数大于了阈值，那么就扩容，走resize逻辑。resize我们上面已经重点分析过，不再复述。

```
if (++size > threshold) {
    resize();
}
```



## 3.3 get操作

Hashmap有put操作就有get操作，我们来看看get是怎么完成的。

```
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

```
static final int hash(Object key) {
    int h;
    //@@ use h>>>16 maybe because the author don't beleave that key'hashCode has right value.
    //maybe some guy's hashcode is always similar.
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

首先肯定是根据key计算出hash,这个步骤已经在put操作的时候介绍过了。然后调用getNode方法

```
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```



3.3.1 getNode判断条件

首先会进行一个if判断

if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {

​	….

}

如果tab != null && tab.length >0 ，这个是必须判断的，如果Hashmap里面没有元素，肯定没有获取的必要了。

然后用(n-1) & hash操作，这个操作大家肯定不陌生了取到对应的槽位。



3.3.2 获取对应的值，判断第一个元素

如果这个槽位上有数据的话，那么就进行下面的操作。

如果有数据，一般有3种情况

1. 单个元素，没有冲突的
2. 一个红黑树，冲突的元素大于8个了
3. 链表，冲突的元素在8个以内。

所以，它首先判断第一个元素是不是我们要寻找的，因为上面3种情况，不管哪一种，都是可以先判断第一个元素。

如果第一个元素就是我们要寻找的，那么我们就不用遍历链表或者红黑树了，间接提升了效率。

```
if (first.hash == hash && // always check first node
    ((k = first.key) == key || (key != null && key.equals(k))))
    return first;
```

源代码里面也是注释了"always check first node"。

大家注意一点，判断是否是我们要找的元素有两点

1. hash是否相等
2. 是否equals

这就说明了我们重写一个对象的equals方法的时候一定要重写hash，不然这里匹配就可能会有问题。



3.3.3 遍历红黑树找到合适的元素

```
if (first instanceof TreeNode)
    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
```



3.3.4 从链表里面获取合适的元素

```
do {
    if (e.hash == hash &&
        ((k = e.key) == key || (key != null && key.equals(k))))
        return e;
} while ((e = e.next) != null);
```

这样就找到对应的元素，如果没有找到的话，返回null



那么，我们现在来回答最开始的那几个问题。



# 四. 问题答案

## 4.1 怎么根据key找到对应的存储地址?

```
1. 首先得到int hash = key.hashcode()
2. hash = hash >>> 16
3. (n-1) & hash得到正确的槽位
4. 如果槽位上有冲突，那么就遍历链表或者红黑树
```



## 4.2 哈希因子是用来干嘛,默认多少?

```
哈希因子默认0.75
作用是限定填充多满的时候需要扩容，不能过大或者过小。
过大: 导致难以扩容，那么在有限的空间里面就存储了很多的元素，难免就会增大冲突，从而增加查询和插入的复杂度。
过小: 导致频繁的扩容，影响性能，Hashmap利用率低。

举个例子，哈希因子就是一个麻袋外面套的一根松紧带，如果过大，就是松紧带太松。就会导致麻袋里面装进去很多很多东西，本来需要换个大麻袋而没有更换。
如果过小，也就是松紧带过紧，麻袋里面刚放一点东西就需要重新换个大的麻袋，从旧的麻袋换到新麻袋也是资源消耗，同时麻袋只是存储了一点东西就更换，也是资源浪费。
```



## 4.3 初始容量默认多少？

```
初始默认大小 16
```



## 4.4 冲突了怎么办?

```
冲突了，一般也说是哈希碰撞，这个时候会使用一个链表来存储冲突的key，如果冲突的节点超过8个，为了提高检索效率，就会转化成红黑树。
```



## 4.5 扩容在什么情况下发生?怎么扩容?

```
扩容在存储的元素大于阈值的时候发生。
扩容的时候一般是直接扩大到2倍
然后需要把原来的元素重新定位槽位，此外如果是红黑树和链表的话，还要进行拆分，拆分成高位和低位的两个部分。
如果是红黑树的话，还要进行红黑树转成链表的判断，如果红黑树的节点小于6了，那么就需要转化成链表。
```



## 4.6 为什么Hashmap的长度是2的次幂？

```
为了便于计算，不管是put还是get，或者resize都需要进行hash & (n-1)的与操作。
而n = 2的次幂的话，那么(n - 1)的二进制就是一连串的1，比如
1: 1
3: 11
7: 111
15: 1111
31: 11111
那么hash &(n-1)的时候就会保留hash低位1，丢弃高位的1。
```



## 4.7 为什么说重写equals的同时重写hashcode方法？

```
因为get操作的时候除了比较equals方法外，还要比较hash.
```



## 4.8 多线程下面使用Hashmap会怎么样?

```
可能会出现多线程不可预知的问题，比如
map.put("a",1);
map.put("a",2);
两个线程执行，那么最后的结果得看线程调度的结果。
所以Hashmap是非线程安全的，所以Hashmap提供了快速失败的操作。如果多线程操作，会报ConcurrentModificationException
```



## 4.9 Hashmap使用了什么样的hash算法

使用hash算法我的理解是：

1. 设计者不能容易相信用户传入的hashcode，所以他要做一些处理，但是这个处理还要兼顾性能和效果，不能因为这个处理而降低效率，同时还要有一定的效果。
2. 所以他把传入的hashcode的高16位和低16位进行了^操作。



详细分析如下:

**^(亦或操作，如果两个数的二进制，相同位数只有一个是1，则该位结果是1，否则是0)**


先看下他的整个代码算法步骤:
1. hashcode右移16位  (h >>> 16)

2. hashcode和右移16位的结果 ^运算

3. 上面两步计算出来的hash和(n-1)做&运算，n代表map容量，比如默认的16

   ​

设计者认为n一般是一个不会太大的数字，比如16，那么(n-1)的二进制就是1111，所以如果直接拿key的hashcode和n-1做&运算，那么基本就是低位在运算。



比如hashcode是1101111001111 0101

```
1101111001111 0101

0000000000000 1111

--------------------------- 做&操作

0000000000000 0101
```



所以前面高位就是被过滤掉的，**只有低位参与了运算**

所以！设计者想让高位也参与进来，于是，他在计算hashcode的时候，首先右移16位，前面补0，假设我们传入的hashcode是32位，那么高位16就变成低位16，而高16位全部补0

然后和原来的hashcode做^运算

由于右移后的hashcode高16是0，所以^运算对结果的高16位没有影响

根据上面讲的，右移后的低16位是之前的高16位，那么现在就是原来的高16位与低16位做^操作。


这样计算出来的结果，是hashcode的高16位和低16位共同计算出来的。



当然，这里我们会提一个问题，如果我的hashcode本来不足16位，那会发生什么呢？
由于不足16位，那么右移16后，就会全部是0，所以^的时候就不会有任何效果，**所以对于hashcode的重写，除非你有特殊原因，一般建议大于16位，这样就可以利用设计者这里做的优化。**



这个流程可以用具体的数字模拟下，比如hashcode 32位是1111 1111 1111 0001    0011 1100 1000 0101
那么这个流程就如下:

```
1. 右移16位
   0000 0000 0000 0000 1111 1111 1111 0001 
2. ^运算
   0000 0000 0000 0000    1111 1111 1111 0001  右移16位后的结果
   1111 1111 1111 0001    0011 1100 1000 0101  原来的hashcode
   ------------------------------------------------------------------ ^操作
   1111 1111 1111 0001    1100 0011 0111 0100
3. 和n-1做&运算，n=16
   1111 1111 1111 0001    1100 0011 0111 0100  计算后的hashcode
   0000 0000 0000 0000    0000 0000 0000 1111  n-1的二进制
   ------------------------------------------------------------------ &操作
   0000 0000 0000 0000    0000 0000 0000 0100

```



最终结果0100=4，也就是落在4槽位

再举个hashcode<=16位数的例子，比如和上面hashcode的低位一样
0011 1100 1000 0101

```
1. 右移16位
   0000 0000 0000 0000  0000 0000 0000 0000
2. ^运算
   0000 0000 0000 0000  0000 0000 0000 0000
   0000 0000 0000 0000  0011 1100 1000 0101 原来的hashcode
   ------------------------------------------------------------------ ^运算
   0000 0000 0000 0000  0011 1100 1000 0101

可以看到计算后的hashcode结果和计算前是一样的，因为我们的hashcode长度小于等于16，相当于没有高位，所以对最终结果没有影响

1. 和n-1 &操作
   0000 0000 0000 0000  0011 1100 1000 0101
   0000 0000 0000 0000  0000 0000 0000 1111
   ------------------------------------------------------------------ &操作
   0000 0000 0000 0000  0000 0000 0000 0101

```



0101 = 5
最终落到槽位5

其实设计者也会这个算法做了说明：

使用key.hashcode的高位和低位进行疑惑操作，设计者认为目前的hashmap很容易碰撞，因为hashmap的大小比如是16是，n-1就是1111，那就是只有4位做运算。
所以在考虑速度，质量，作用等因素后，就把高16位和低16做了亦或运算。
另外，设计者还认为目前hashcode基本分布的不错，而且就算冲突也有红黑树来提升冲突链表的性能，所以他只是简单的亦或一下，这样也不会造成什么性能消耗，同时还让高位也参与了运算。


可以看到设计者对二进制这种运算的运用已经是炉火纯青，这个也是后面自己写代码可以借鉴的。


具体大家看设计者的注释

/**
     * Computes key.hashCode() and spreads (XORs) higher bits of hash
     * to lower.  Because the table uses power-of-two masking, sets of
     * hashes that vary only in bits above the current mask will
     * always collide. (Among known examples are sets of Float keys
     * holding consecutive whole numbers in small tables.)  So we
     * apply a transform that spreads the impact of higher bits
     * downward. There is a tradeoff between speed, utility, and
     * quality of bit-spreading. Because many common sets of hashes
     * are already reasonably distributed (so don't benefit from
     * spreading), and because we use trees to handle large sets of
     * collisions in bins, we just XOR some shifted bits in the
     * cheapest possible way to reduce systematic lossage, as well as
     * to incorporate impact of the highest bits that would otherwise
     * never be used in index calculations because of table bounds.
     */
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }





参考:

https://www.cnblogs.com/chengxiao/p/6059914.html

https://blog.csdn.net/u011240877/article/details/53351188

https://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/