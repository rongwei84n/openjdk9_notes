Title:BitSet

---

# 一. BitSet介绍

Bitset是Java中的一种数据结构。Bitset中主要存储的是二进制位，做的也都是位运算，每一位只用来存储0，1值，主要用于对数据的标记。

Bitset的基本原理是，用1位来表示一个数据是否出现过，0为没有出现过，1表示出现过。使用的时候可以根据某一个位是否为0表示此数是否出现过。

如果扩容的话，直接扩容成2倍

存储的方式是以64位为单位进行存储，也就是一个long的大小，所以它的大小是long类型大小的整数倍。

如下

```
public static void main(String[] args) {
    BitSet bs1 = new BitSet();
    System.out.println(bs1.size());
    	
    BitSet bs2 = new BitSet(65);
    System.out.println(bs2.size());
}
-------执行结果
64
128
```

默认一个long类型大小，也就是64位，如果传入指定大小65，那么就是2个long大小，128。



# 二. 成员变量

```
private static final int ADDRESS_BITS_PER_WORD = 6;

//BITS_PER_WORD=64， 1左移6位
private static final int BITS_PER_WORD = 1 << ADDRESS_BITS_PER_WORD;
private static final int BIT_INDEX_MASK = BITS_PER_WORD - 1;


内部存储结构，用long表示一个存储结构，存储64个字节。如果多个的话，也就是以64个字节为单位，多个64个字节存储。
/**
 * The internal field corresponding to the serialField "bits".
 */
private long[] words;


已经使用的数组长度，也就是已经使用了多少个64字节。
/**
 * The number of words in the logical size of this BitSet.
 */
private transient int wordsInUse = 0;
```



# 三. 初始化

## 3.1 默认初始化

```
public BitSet() {
    initWords(BITS_PER_WORD);
    sizeIsSticky = false;
}
```

上面说BITS_PER_WORD=64，所以initWords(64)

```
private void initWords(int nbits) {
    words = new long[wordIndex(nbits-1) + 1];
}
```

```
private static int wordIndex(int bitIndex) {
    return bitIndex >> ADDRESS_BITS_PER_WORD;
}
```

这里面有两个索引位置，一个是wordIndex，指的是第几个64位置，也就是long数组的索引。

还有一个是bitIndex，指的是具体某个64个字节里面的位置索引。

所以我们要确定一个位置，必须知道上面这两个索引位置。

而wordIndex(int bitIndex)指的就是wordIndex，计算方式是bitIndex右移6位，比如如果

bitIndex = 1->右移6位就是0，所以1就应该落在long数组序号0

如果bitIndex = 65

也就是1000001->右移6位就是0001，也就是1

所以65就应该落在long数组序号1



## 3.2 指定大小

```
public BitSet(int nbits) {
    // nbits can't be negative; size 0 is OK
    if (nbits < 0)
        throw new NegativeArraySizeException("nbits < 0: " + nbits);

    initWords(nbits);
    sizeIsSticky = true;
}
```



## 3.3 元素复制

```
private BitSet(long[] words) {
    this.words = words;
    this.wordsInUse = words.length;
    checkInvariants();
}
```



# 四. 设置元素值

## 4.1 set

```
public void set(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);

    int wordIndex = wordIndex(bitIndex);
    expandTo(wordIndex);

    words[wordIndex] |= (1L << bitIndex); // Restores invariants

    checkInvariants();
}
```

我们如果知道了wordIndex和bitIndex的区别，那么这段代码逻辑就很容易看懂。

首先根据bitIndex找到合适和wordIndex,也就是该去多少个wordIndex上面去找。

找到wordIndex之后，然后在这个wordIndex里面设置对应的bitIndex.

设置的方法是做或运算。

打个比方，传入的bitIndex是2，那么首先计算出wordIndex=0，也就是long[0]里面

然后1左移两位就是1->100，做或操作，如果原来的2这个位置是0，那就是这样的

原来的63->0

000000000000000 原来数组 

000000000000100 左移两位后

-----------------------------------------------或操作

000000000000100 

于是，数组里面变成000000000000100 ，所以long[0]里面的第2个位置是1



## 4.2 clear

与set相对应的是clear，具体逻辑就不分析了，与set类似。

```
public void clear(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);

    int wordIndex = wordIndex(bitIndex);
    if (wordIndex >= wordsInUse)
        return;

    words[wordIndex] &= ~(1L << bitIndex);

    recalculateWordsInUse();
    checkInvariants();
}
```





