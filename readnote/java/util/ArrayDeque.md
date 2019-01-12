Title:ArrayDeque

----

# 一. 问题切入点

## 1.1 ArrayDeque线程安全吗？

## 1.2 ArrayDeque提供了什么样的方式实现了双端队列？

## 1.3 ArrayDeque支持null元素吗？为什么？

## 1.4 ArrayDeque默认容量多少？如何扩容？

## 1.5 transient关键字作用是什么？



# 二. ArrayDeque简介

Deque即双端队列。是一种具有队列和栈的性质的数据结构。双端队列中的元素可以从两端弹出，其限定插入和删除操作在表的两端进行。

而ArrayDeque顾名思义就是用数组实现的双端队列。



# 三. 代码分析

## 3.1 成员变量

```
transient Object[] elements; // non-private to simplify nested class access

元素存储结构，数组
```

```
transient int head;
数组头的索引
```

```
transient int tail;
数组尾部的索引
```



## 3.2 初始化

```
/**
 * Constructs an empty array deque with an initial capacity
 * sufficient to hold 16 elements.
 */
public ArrayDeque() {
    elements = new Object[16];
}
```

初始化一个数组，大小为16，16是2的次幂，**2的次幂这是一个很神奇的数字**，因为length-1的二进制全是1，那么做&运算的时候，就可以找出数字对应的位置。 

具体效果就相当于%求余，不过&操作比%速度更快。

比如20和(16 -1)做&操作

```
10101 - 21
01111 - 15
---------------------
00101 - 5
结果就是5
```

20和16求余% = 5

所以结果&和%是一样的



## 3.3 push

```
public void push(E e) {
    addFirst(e);
}
```

```
public void addFirst(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[head = (head - 1) & (elements.length - 1)] = e;
    if (head == tail)
        doubleCapacity();
}
```

push操作是把元素插入到数组头部，它插入的方式是从尾部开始插入，以默认elements.length=16为例，elements.length-1=15

head初始是0，那么(0-1)&15 = 15，于是插入的位置就是15，也就是在数组的最尾部。

然后addFirst再次插入元素的话，那么head就是15，于是就变成了

(15-1) & (16-1) = 14

如果head等于1的时候，再次addFirst插入元素，那么head就会等于tail，此时就需要扩容，扩容的大小是扩大一倍。

我们用表格来描述下这个过程，假设addFirst("a"), addLast("b"),"-"表示没有插入数据

|             | 序号0 | 序号1         | 序号2   | 序号3   | 序号4   | 序号5   | 序号6   | 序号7   | 序号8   | 序号9   | 序号10  | 序号11  | 序号12  | 序号13  | 序号14  | 序号15  |
| ----------- | ----- | ------------- | ------- | ------- | ------- | ------- | ------- | ------- | ------- | ------- | ------- | ------- | ------- | ------- | ------- | ------- |
| 1.addFirst  | -     | -             | -       | -       | -       | -       | -       | -       | -       | -       | -       | -       | -       | -       | -       | a(head) |
| 2.addFirst  | -     | -             | -       | -       | -       | -       | -       | -       | -       | -       | -       | -       | -       | -       | a(head) | a       |
| 3.addFirst  | -     | -             | -       | -       | -       | -       | -       | -       | -       | -       | -       | -       | -       | a(head) | a       | a       |
| 4.addFirst  | -     | -             | -       | -       | -       | -       | -       | -       | -       | -       | -       | -       | a(head) | a       | a       | a       |
| 5.addLast   | b     | -(tail)       | -       | -       | -       | -       | -       | -       | -       | -       | -       | -       | a(head) | a       | a       | a       |
| 6.addFirst  | b     | -(tail)       | -       | -       | -       | -       | -       | -       | -       | -       | -       | a(head) | a       | a       | a       | a       |
| 7.addFirst  | b     | -(tail)       | -       | -       | -       | -       | -       | -       | -       | -       | a(head) | a       | a       | a       | a       | a       |
| 8.addFirst  | b     | -(tail)       | -       | -       | -       | -       | -       | -       | -       | a(head) | a       | a       | a       | a       | a       | a       |
| 9.addFirst  | b     | -(tail)       | -       | -       | -       | -       | -       | -       | a(head) | a       | a       | a       | a       | a       | a       | a       |
| 10.addFirst | b     | -(tail)       | -       | -       | -       | -       | -       | a(head) | a       | a       | a       | a       | a       | a       | a       | a       |
| 11.addFirst | b     | -(tail)       | -       | -       | -       | -       | a(head) | a       | a       | a       | a       | a       | a       | a       | a       | a       |
| 12.addFirst | b     | -(tail)       | -       | -       | -       | a(head) | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       |
| 13.addFirst | b     | -(tail)       | -       | -       | a(head) | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       |
| 14.addFirst | b     | -(tail)       | -       | a(head) | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       |
| 15.addFirst | b     | -(tail)       | a(head) | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       |
| 16.addFirst | b     | a(tail, head) | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       | a       |

这个时候，tail==head，所以也是数组被填满的时候，需要扩容，于是走doubleCapacity逻辑

这个算法说实话还是蛮奇妙的，很简单的几行代码，实现了head和tail循环的问题。



继续看扩容doubleCapacity

```
private void doubleCapacity() {
    assert head == tail;
    int p = head;
    int n = elements.length;
    int r = n - p; // number of elements to the right of p
    int newCapacity = n << 1;
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");
    Object[] a = new Object[newCapacity];
    System.arraycopy(elements, p, a, 0, r);
    System.arraycopy(elements, 0, a, r, p);
    elements = a;
    head = 0;
    tail = n;
}
```

这段代码的意思是把数组扩容成2倍，然后原来数组的元素拷贝到新数组的前面部分，然后head指向0，tail指向原来数组size。以原来是length=16为例，扩容后head=0，tail=16，元素存储在0-15位置。



## 3.4 pop

```
public E pop() {
    return removeFirst();
}
```

```
public E removeFirst() {
    E x = pollFirst();
    if (x == null)
        throw new NoSuchElementException();
    return x;
}
```

```
public E pollFirst() {
    final Object[] elements = this.elements;
    final int h = head;
    @SuppressWarnings("unchecked")
    E result = (E) elements[h];
    // Element is null if deque empty
    if (result != null) {
        elements[h] = null; // Must null out slot
        head = (h + 1) & (elements.length - 1);
    }
    return result;
}
```

就是获取第一个元素，然后把head++



# 四. 问题解答

## 4.1 ArrayDeque线程安全吗？

```
不是线程安全
```



## 4.2 ArrayDeque提供了什么样的方式实现了双端队列？

```
通过head和tail两个索引标记头尾元素，从而实现队列和栈的特性.
```



## 4.3 ArrayDeque支持null元素吗？为什么？

```
不支持，如果传入null,会抛出NullPointerException
```



## 4.4 ArrayDeque默认容量多少？如何扩容？

```
默认16，双倍扩容
```



## 4.5 transient关键字作用是什么？

```
java 的transient关键字为我们提供了便利，你只需要实现Serilizable接口，将不需要序列化的属性前添加关键字transient，序列化对象的时候，这个属性就不会序列化到指定的目的地中。

但是如果你是通过实现Externalizable接口来实现的序列化，那么transient关键字将不起作用。
```





参考:

http://www.importnew.com/21517.html