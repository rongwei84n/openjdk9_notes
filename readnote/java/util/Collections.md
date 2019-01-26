Title:Collections

---

#一. 简介

Collections提供了一系列针对Collection类的静态方法，比如排序，打散，比较，倒序等方法



# 二. 举例

## 2.1 随机打散列表

> *shuffle*

```
List<String> lists = new ArrayList<String>();
    	for (int i = 0; i < 5; i++) {
			lists.add("i" + i);
		}
    	
    	for (String string : lists) {
			System.out.println(string);
		}
    	System.out.println("------------");
    	Collections.shuffle(lists);
    	for (String string : lists) {
			System.out.println(string);
		}
```

结果

```
i0
i1
i2
i3
i4
------------
i4
i1
i2
i0
i3
```



## 2.2 倒序

```
List<String> lists = new ArrayList<String>();
    	for (int i = 0; i < 5; i++) {
			lists.add("i" + i);
		}
    	
    	for (String string : lists) {
			System.out.println(string);
		}
    	System.out.println("------------");
    	Collections.reverse(lists);
    	for (String string : lists) {
			System.out.println(string);
		}
```

结果：

```
i0
i1
i2
i3
i4
------------
i4
i3
i2
i1
i0
```



## 2.3 排序

```
List<String> lists = new ArrayList<String>();
    	for (int i = 0; i < 5; i++) {
			lists.add("i" + i);
		}
    	
    	for (String string : lists) {
			System.out.println(string);
		}
    	System.out.println("------------");
    	Collections.sort(lists, new Comparator<String>() {
			@Override
			public int compare(String o1, String o2) {
				return o2.compareTo(o1);
			}
		});
    	for (String string : lists) {
			System.out.println(string);
		}
```

结果(倒序)

```
i0
i1
i2
i3
i4
------------
i4
i3
i2
i1
i0
```

顺序的话就修改下compare方法



## 2.4 交换元素位置

```
List<String> lists = new ArrayList<String>();
    	for (int i = 0; i < 5; i++) {
			lists.add("i" + i);
		}
    	
    	for (String string : lists) {
			System.out.println(string);
		}
    	System.out.println("------------");
    	Collections.swap(lists, 2, 4);
    	for (String string : lists) {
			System.out.println(string);
		}
```

结果

```
i0
i1
i2
i3
i4
------------
i0
i1
i4
i3
i2
```



## 2.5 同步

```
Collections.synchronizedList(List)
```



**其他的还有很多方法，大家可以自己探索。**