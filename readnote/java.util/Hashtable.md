title:Hashtable

-----

# 一. 问题切入点

## 1. Hashtable怎么实现线程安全的

## 2. Hashtable默认容量是多少？

## 3. Hashtable扩容后是原来的多大？

## 4. Hashtable默认的哈希因子是多少，用来干嘛的？

## 5. Hashtable的hash算法和Hashmap有什么不同？



# 二. Hashtable简介

Hashtable是用一个数组维持的索引，然后在数组的每个槽位存储了放在这个槽位的数据，用链表来组织。如下

![image-20181229223958779](http://thyrsi.com/t6/645/1546164429x2728329023.png)

> 图来自
>
> https://www.cnblogs.com/chengxiao/p/6059914.html







# 三. 源代码



## 3.1 成员变量

```
private transient Entry<?,?>[] table;
Hashtable存储数据的结构，每个Entry包含一个hash,key,value,next.多个entry组成数组。
```



```
private transient int count;
Hashtable中元素的总个数
```



```
private int threshold;
扩容阈值，存储的元素个数大于这个值就需要被扩容,resize
```



```
private float loadFactor;
哈希因子，也可以理解成Hashtable装填率
threshold = 总容量 * loadFactor
这个值不能太大或者太小

太小-频繁扩容，频繁rehash，影响性能。这个很好理解，比如你的空间16，哈希因子0.1，那用了1个元素就需要扩容了，其他还有15个空间没有使用呢。
2. 太大-迟迟不扩容，导致有限的空间存储了大量的key，那么肯定就很容易冲突。比如你的空间是4，然后哈希因子1千万，那我们插入一万个元素也不会扩容。
```



```
private transient int modCount = 0;
修改次数，用于快速失败，比如遍历的时候又插入元素。
实现方法也很简单，操作前记录下modCount的值，操作完成后再次读取modCount，如果两个不相同，就说明做了其他操作，可以抛出ConcurrentModificationException。
```



可以看到Hashtable的逻辑相比于Hashmap实在是有点少的寒酸，感觉像是阉割版.

比如它没有Hashmap的红黑树逻辑





## 3.2 Hashtable初始化

Hashtable有3个重载的构造函数，可以给调用者指定初始容量以及哈希因子loadFactor，当然你也可以不指定，使用默认的值，默认大小是11，loadFactor是0.75

**默认大小是11？**



```java
public Hashtable(int initialCapacity, float loadFactor) {
    ...
    if (initialCapacity==0)
        initialCapacity = 1;
    this.loadFactor = loadFactor;
    table = new Entry<?,?>[initialCapacity];
    threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
}

public Hashtable(int initialCapacity) {
    this(initialCapacity, 0.75f);
}

public Hashtable() {
    this(11, 0.75f);
}
```

这是个有意思的地方，我们讲过Hashmap的大小必须是2的次幂，因为为了方便进行hash位运算。

那现在Hashtable的默认大小是11，它又采用什么样的hash算法呢？

继续往下面看。



## 3.3 put

```
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    if (value == null) {
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }

    addEntry(hash, key, value, index);
    return null;
}
```

首先, put方法加上了synchronized关键字，说明它是线程安全的，这也是它和Hashmap的区别之一。

然后，它拿hashcode和0x7FFFFFFF做&运算

```
0x7FFFFFFF = 1111111111111111111111111111111 长度31
```

那就是说过滤掉hashcode的31之前的位数

然后和tab.length求余得到索引号

```
int index = (hash & 0x7FFFFFFF) % tab.length;
```

对比下hashmap，他是用hash和(n-1)做&运算代替求余，这种做法比Hashtable更加高效，所以Hashtable这里的处理有点粗糙的感觉。

当然这也是它不要容量是2的次幂原因(因为只有n = 2的次幂，n-1的二进制才全部是1，才能做&操作得到正确的index)。

拿到index之后，它首先从对应的数组找到Entry首节点，然后遍历这个槽位上的链表。

那大家思考下，为什么要遍历这个链表呢？

那是因为Hashtable得知道这个槽位有没有已经存储了这个key，如果已经存储了，那么就直接返回了，代码如下:

```
Entry<K,V> entry = (Entry<K,V>)tab[index];
for(; entry != null ; entry = entry.next) {
   if ((entry.hash == hash) && entry.key.equals(key)) {
       V old = entry.value;
       entry.value = value;
       return old;
   }
}    
```

**判断是否是这个key的方法是比较hash和equals，这也是我们多次强调重写一个类的equals方法最好也重写hashcode方法的原因！**



如果没有在这个槽位上面没有找到这个key，说明这个key还没有存储，需要添加到Hashtable，于是继续往下走添加节点-addEntry

```
addEntry(hash, key, value, index);
```

```
private void addEntry(int hash, K key, V value, int index) {
    Entry<?,?> tab[] = table;
    if (count >= threshold) {
        // Rehash the table if the threshold is exceeded
        rehash();

        tab = table;
        hash = key.hashCode();
        index = (hash & 0x7FFFFFFF) % tab.length;
    }

    // Creates the new entry.
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>) tab[index];
    tab[index] = new Entry<>(hash, key, value, e);
    count++;
    modCount++;
}
```

添加之前，Hashtable会判断当前容量是否大于阈值threshold，如果是，那么需要扩容rehash，这个我们可以待会再讲。

我们先假设它的容量还没有达到阈值，那么将跳过if判断

```
if (count >= threshold) {
	...
}
```

到这里，那么就创建一个节点，把原来槽位数据添加到新节点的next部分，然后添加到指定的槽位上，

```
// Creates the new entry.
Entry<K,V> e = (Entry<K,V>) tab[index];
tab[index] = new Entry<>(hash, key, value, e);
```

```
protected Entry(int hash, K key, V value, Entry<K,V> next) {
    this.hash = hash;
    this.key =  key;
    this.value = value;
    this.next = next;
}
```



其实这里还有两种情况需要考虑

1. 这个槽位上有数据
2. 这个槽位上没有数据

不过这两种情况都可以作为一种处理，就是新建一个Entry，然后把现有槽位的数据(没有就是null)，加到Entry的next部位。

如果没有数据，那么Entry的next就是null，如果有数据，那么Entry的next就指向原来的数据，形成链表。

数据结果参考第二章图。

最后把计数和修改次数+1.

```
count++;
modCount++;
```



## 3.4 rehash

现在我们回头看看3.3 put值的时候，当超过阈值时rehash的操作

```
if (count >= threshold) {
    // Rehash the table if the threshold is exceeded
    rehash();

    tab = table;
    hash = key.hashCode();
    index = (hash & 0x7FFFFFFF) % tab.length;
}
```

```
protected void rehash() {
    int oldCapacity = table.length;
    Entry<?,?>[] oldMap = table;

    // overflow-conscious code
    int newCapacity = (oldCapacity << 1) + 1;
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        if (oldCapacity == MAX_ARRAY_SIZE)
            // Keep running with MAX_ARRAY_SIZE buckets
            return;
        newCapacity = MAX_ARRAY_SIZE;
    }
    Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

    modCount++;
    threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    table = newMap;

    for (int i = oldCapacity ; i-- > 0 ;) {
        for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
            Entry<K,V> e = old;
            old = old.next;

            int index = (e.hash & 0x7FFFFFFF) % newCapacity;
            e.next = (Entry<K,V>)newMap[index];
            newMap[index] = e;
        }
    }
}
```

在分析代码之前，我们先自己想想，rehash需要做什么？

可能

1. 扩容，Hashmap是扩大成2倍，不知道Hashtable扩容成多少
2. 重新定位槽位，原来在一个槽位的key，扩容之后也许就不会在一个槽位了，如果原来在一个链表里面的话，还的从链表里面拆出来。

那继续看代码

首先，获取新的容量

int newCapacity = (oldCapacity << 1) + 1;

那新容量就是原来的翻倍，然后再加1，比如原来的是11，扩容之后就是23

```
11->23->47
```

然后它还进行了一个最大容量的判断 "MAX_ARRAY_SIZE"

```
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

```
if (newCapacity - MAX_ARRAY_SIZE > 0) {
        if (oldCapacity == MAX_ARRAY_SIZE)
            // Keep running with MAX_ARRAY_SIZE buckets
            return;
        newCapacity = MAX_ARRAY_SIZE;
    }
```

如果已经达到最大容量了，那么就用最大容量，不再扩容

// Keep running with MAX_ARRAY_SIZE buckets

然后重新计算阈值

```
threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
```

接着，创建新的数组来重新存储数据，然后指向新的数组。

```
Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];
table = newMap;
```



接下来就是要不断重新移动原来key到新的Hashtable里面去了

思路是首先挨个遍历一个数组里面的每一个槽位，然后遍历每个槽位上的链表，把每个key根据新的大小newCapacity重新计算槽位，所以需要两层循环，如下:

```java
for (int i = oldCapacity ; i-- > 0 ;) {
    for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
        Entry<K,V> e = old;
        old = old.next;

        int index = (e.hash & 0x7FFFFFFF) % newCapacity;
        e.next = (Entry<K,V>)newMap[index];
        newMap[index] = e;
    }
}
```

**所以，从这里看出，rehash的操作实在是过于繁重，需要重新处理所有的key，所以应该设置合适的大小，尽量避免rehash！**



## 3.5 get

根据key返回存储的对应的值

```
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

操作的步骤是

1.根据key.hashcode和tab.length求余得到索引

2.根据索引找到槽位，然后遍历槽位上的链表

3.遍历链表的每个元素，对比equals和hashcode，如果相等，那就意味着找到了指定的元素。



# 四. 问题解决

## 4.1 Hashtable怎么实现线程安全的

```
synchronized关键字get,put等方法
```



## 4.2. Hashtable默认容量是多少？

```
11，因为它没有像Hashmap一样需要hash & (n-1)来获取槽位，而是通过求余的方法，所以它的大小不用限于2的次幂。
```



## 4.3. Hashtable扩容后是原来的多大？

```
2倍+1
比如11->23->47
```



## 4.4. Hashtable默认的哈希因子是多少，用来干嘛的？

```
默认哈希因子0.75，跟Hashmap一样，是用来描述Hashtable的填充率的，不能太大或者太大。
```



## 4.5. Hashtable的hash算法和Hashmap有什么不同？

```
Hashmap
1. hash = hash ^ hash >>> 16
2. int index = hash & (n-1)
---------------------------------------

Hashtable
(hash & 0x7FFFFFFF) % tab.length; 
```



# 五. 总结

从代码看，Hashtable的实现逻辑没有Hashmap完善，比如Hashtable没有红黑树，完全用链表来存储冲突的元素。

所以如果可能的话，应该尽量使用Hashmap，就算有线程安全的需求，也可以自己去实现线程安全。因为Hashtable的线程安全限定也是在方法上，粒度也是比较粗糙。







