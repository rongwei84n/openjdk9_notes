title:LinkedList

----

# 一. 问题切入点

## 1.1 LinkedList怎么存储数据的？

## 1.2 LinkedList需要扩容吗？如果有怎么扩容的？

## 1.3 LinkedList线程安全吗？

## 1.4 LinkedList有快速失败吗？







# 二. LinkedList简介

LinkedList和ArrayList一样，作为List的一种实现。不过它和ArrayList采用数组来存储数据不同的是，它采用链表来存储数据。

使用数组存储和链表存储的对比如下:

|              | ArrayList | LinkedList |
| ------------ | --------- | ---------- |
| 遍历速度     | 快        | 慢         |
| 插入元素速度 | 慢        | 快         |
| 删除元素速度 | 慢        | 快         |



# 三. 代码解读

## 3.1 LinkedList属性

```
transient int size = 0;
LinkedList里面元素的总个数
```



```
/**
 * Pointer to first node.
 */
transient Node<E> first;
指向第一个元素
```



```
/**
 * Pointer to last node.
 */
transient Node<E> last;
指向最后一个元素，为了提高性能，避免遍历链表
```



## 3.2 add

我们用指定位置add元素的方法做代码分析，因为指定位置的插入更有代表性。

```
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}
```

```
private void checkPositionIndex(int index) {
    if (!isPositionIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```



### 3.1.1 checkPositionIndex 

先看checkPositionIndex方法，这是一个预检查的方法，检查准备要插入的index是否已经超出了范围。

```
private boolean isPositionIndex(int index) {
    return index >= 0 && index <= size;
}
```

index肯定只能在0~size之间，比如LinkedList大小是10，那插入的index肯定只能在0-10之间，如果你想插入20，那就要报错-IndexOutOfBoundsException



### 3.2.2 是否插入队尾

```java
if (index == size)
    linkLast(element);
else
    linkBefore(element, node(index));
```

这里又做了一个优化，如果在队尾(index == size)，那么就去队尾插入数据。

```java
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

linkLast方法也做了优化，它并没有从头开始遍历链表，而是记录了链表最后一个元素last，这样就省去了遍历链表的过程。

下面我们来看看这个过程，首先找到最后一个记录元素last，然后把要插入的元素包装下成一个Node对象，然后判断last元素是否等于null

1. 如果等于null，那就说明当前链表是空，还没有节点，那就把第一个节点first指向newNode
2. 如果链表里面有元素，那就把最后一个节点的next指向新的元素newNode，然后把原来的last节点指向newNode

最后就是size++, modCount++，modCount用于快速失败。



## 3.2.3 插入元素到列表中间linkBefore 

```
linkBefore(element, node(index));
```

```
Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

```
/**
 * Inserts element e before non-null Node succ.
 */
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

在linkBefore里面分成两步，

1. node方法找到合适的位置

2. 插入元素

   ​

### 3.2.3 node寻找合适的插入位置

```
Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

首先判断index < (size >>1)，size>>1意思是size的1/2，所以这里使用简单的二分法-只进行一步二分。

如果index < (size >>1)，那就是在前半部分，循环遍历前半部分

否则，循环遍历后半部分。

找到合适的插入点元素x



### 3.2.4 插入元素 

```
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```





## 3.3 get(int index)

```
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```

checkElementIndex 3.1.1里面说过，不再重复。

node(index)逻辑在3.2.3厘米说过，不再重复。



# 四. 问题解答

## 4.1 LinkedList怎么存储数据的？

```
使用链表来存储数据
```



## 4.2 LinkedList需要扩容吗？如果有怎么扩容的？

```
不需要，因为是链表存储数据，所以不需要扩容
```



## 4.3 LinkedList线程安全吗？

```
非线程安全
```



## 4.4 LinkedList有快速失败吗？

```
有，如果一边读取LinkedList一边修改LinkedList，会报java.util.ConcurrentModificationException
```









