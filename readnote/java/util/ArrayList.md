Title:ArrayList

-----

# 一. 问题切入点

## 1.1 ArrayList怎么存储数据的？

## 1.2 ArrayList什么时候扩容，怎么扩容?

## 1.3 ArrayList线程安全吗？

## 1.4 ArrayList有快速失败吗？

## 1.5 ArrayList支持RandomAccess快速访问吗？

## 1.6 ArrayList默认大小是多少，使用默认大小时有什么优化？





# 二. ArrayList简介

ArrayList是Java集合框架中的一个重要的类，它继承于AbstractList，实现了List接口，是一个长度可变的集合，提供了增删改查的功能。集合中允许null的存在。

ArrayList类还是实现了RandomAccess接口，可以对元素进行快速访问。

实现了Serializable接口，说明ArrayList可以被序列化，还有Cloneable接口，可以被复制。

和Vector不同的是，ArrayList不是线程安全的。

ArrayList使用数组来存储数据



# 三. 代码阅读

## 3.1 基本属性

```java
/**
 * Default initial capacity.
 */
private static final int DEFAULT_CAPACITY = 10;
默认ArrayList大小
```



```java
/**
 * Shared empty array instance used for empty instances.
 */
private static final Object[] EMPTY_ELEMENTDATA = {};

用户指定大小为0容量的数组
```



```java
/**
 * Shared empty array instance used for default sized empty instances. We
 * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
 * first element is added.
 */
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
用户不指定容量大小的数组，默认大小容量是10; 其实可以看到DEFAULTCAPACITY_EMPTY_ELEMENTDATA和上面EMPTY_ELEMENTDATA里面初始数组大小都是0，那这里为什么又说初始值容量是10呢？
其实这是JDK的一种优化，如果是默认的大小，那么其实最开始是不会初始化一个容量为10的数组的，只有在add第一个元素的时候，才会生成一个大小是10的数组，然后存入第一个元素。

由于上面的优化，默认的大小也没有创建默认容量为10的数组，而是和初始指定大小0是一样的。所以必须申明两个空的数组
private static final Object[] EMPTY_ELEMENTDATA = {};
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
便于区分，因为这两种情况，在扩容的时候需要区分的。
```



```java
/**
 * The array buffer into which the elements of the ArrayList are stored.
 * The capacity of the ArrayList is the length of this array buffer. Any
 * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
 * will be expanded to DEFAULT_CAPACITY when the first element is added.
 */
transient Object[] elementData; // non-private to simplify nested class access

ArrayList里面存储数据的数组
```



```java
/**
 * The size of the ArrayList (the number of elements it contains).
 *
 * @serial
 */
private int size;
ArrayList里面元素的个数
```



# 3.2 初始化 

```
/**
 * Constructs an empty list with the specified initial capacity.
 *
 * @param  initialCapacity  the initial capacity of the list
 * @throws IllegalArgumentException if the specified initial capacity
 *         is negative
 */
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

/**
 * Constructs an empty list with an initial capacity of ten.
 */
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

如果没有指定大小初始化ArrayList，那么就用DEFAULTCAPACITY_EMPTY_ELEMENTDATA，默认大小的就是10。

```
this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
```



如果指定大小=0，那么就是EMPTY_ELEMENTDATA

```
this.elementData = EMPTY_ELEMENTDATA;
```



## 3.3 add

### 3.3.1 添加一个元素到数组末尾

```java
/**
 * Appends the specified element to the end of this list.
 *
 * @param e element to be appended to this list
 * @return {@code true} (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    modCount++;
    add(e, elementData, size);
    return true;
}
```

### 3.3.2 修改次数modCount++，用于快速失败，然后调用add方法

```
/**
 * This helper method split out from add(E) to keep method
 * bytecode size under 35 (the -XX:MaxInlineSize default value),
 * which helps when add(E) is called in a C1-compiled loop.
 */
private void add(E e, Object[] elementData, int s) {
    if (s == elementData.length)
        elementData = grow();
    elementData[s] = e;
    size = s + 1;
}
```

在add方法里面，如果size == elementData.length，就需要进行扩容 - grow().

那我们来思考下，什么情况下会满足size == elementData.length呢？

1. 初始化大小为0时add第一个元素
2. 初始化大小为默认大小时add第一个元素，因为初始化大小为默认时做了优化，并没有初始化大小为10的数组。
3. 大小真正达到容量时，比如容量20，我们想插入第21个元素时。

那，继续往下么看看grow扩容方法。



### 3.3.3 grow扩容

```
private Object[] grow() {
    return grow(size + 1);
}
```

```
private Object[] grow(int minCapacity) {
    return elementData = Arrays.copyOf(elementData,
                                       newCapacity(minCapacity));
}
```

```
/**
 * Returns a capacity at least as large as the given minimum capacity.
 * Returns the current capacity increased by 50% if that suffices.
 * Will not return a capacity greater than MAX_ARRAY_SIZE unless
 * the given minimum capacity is greater than MAX_ARRAY_SIZE.
 *
 * @param minCapacity the desired minimum capacity
 * @throws OutOfMemoryError if minCapacity is less than zero
 */
private int newCapacity(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity <= 0) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return minCapacity;
    }
    return (newCapacity - MAX_ARRAY_SIZE <= 0)
        ? newCapacity
        : hugeCapacity(minCapacity);
}
```

grow扩容里面基本逻辑就两点，那我们主要看计算新的数组大小，因为数据拷贝没什么看的。

1. 计算出新的数组大小-newCapacity
2. 调用数组拷贝-Arrays.copyOf



### 3.3.4 newCapacity计算扩容大小 

minCapacity 是原来大小+1

newCapacity 是计算出新的大小，是原来的1.5倍

```
int newCapacity = oldCapacity + (oldCapacity >> 1);
```

然后开始一些判断，如下:

```
if (newCapacity - minCapacity <= 0) {
	...
}
```

如果newCapacity<=minCapacity，什么时候newCapacity会小于或者等于minCapacity呢？

1. 默认的容量初始化时，也就是DEFAULTCAPACITY_EMPTY_ELEMENTDATA

2. 其他一些容量比较小的时候，比如2个元素的时候，

   minCapacity = 2+1=3

   newCapacity = 2 * 2>>1 = 3

所以，我们看它的流程

```java
if (newCapacity - minCapacity <= 0) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return minCapacity;
}
```

如果是默认大小初始化的时候，那么就会返回默认大小DEFAULT_CAPACITY - 10

如果其他情况，那么就会返回minCapacity

然后和MAX_ARRAY_SIZE比较，有没有达到最大大小，如果没有的话就返回扩容后的大小newCapacity。

```
return (newCapacity - MAX_ARRAY_SIZE <= 0)
        ? newCapacity
        : hugeCapacity(minCapacity);
```



## 3.3.5 元素拷贝

```
return elementData = Arrays.copyOf(elementData,
                                       newCapacity(minCapacity));
```



## 3.6 get

get方法的实现就很简单了，就是从数组指定索引位置读取值。

```
public E get(int index) {
    Objects.checkIndex(index, size);
    return elementData(index);
}
```

```
E elementData(int index) {
    return (E) elementData[index];
}
```



# 四. 问题解答



## 4.1 ArrayList怎么存储数据的？

```
使用数组来存储
```



## 4.2 ArrayList什么时候扩容，怎么扩容?

```
插入元素的时候如果发现size等于列表的容量，就需要扩容；扩容的大小是原来的1.5倍
```



## 4.3 ArrayList线程安全吗？

```
非线程安全
```



## 4.4 ArrayList有快速失败吗？

```
有，跟Hashmap一样，使用modCount来记录
```



## 4.5 ArrayList支持RandomAccess快速访问吗？

```
RandomAccess 是一个标记接口，用于标明实现该接口的List支持快速随机访问，主要目的是使算法能够在随机和顺序访问的list中表现的更加高效。
ArrayList支持快速访问。
```





## 4.6 ArrayList默认大小是多少，使用默认大小时有什么优化？

```
默认大小是10，不过刚初始化并不会申请10个空间的数组出来，在添加第一个元素的时候才会分配空间10的数组，这是JDK做的优化。
```







