---
title: HashMap实现原理
date: 2018-05-24 22:02:36
tags:
---

#### 摘要
-

HashMap是我们日常比较常用的数据结构，put(key, value),可以通过key来查找对象，JDK 1.8 中引入红黑树来优化查询效率，接下来就底层算法实现和原理做出一个详细的分析

#### HashMap 基本组成元素
-

如下图：

{% asset_img img/HashMap.png %}

其基本组成部分是由一个2的n次幂长度的容量数组，最基本组成单元是Node节点，节点之间可以组成链式结构。如果节点数量大于8，会将其转成红黑树。
  
(1) HashMap put(key, value) 处理步骤
  
  1. 需要找到key的hashcode所对应的数组下标位置</br>
  2. 如果数组中存在Node节点，key相同值覆盖，不同则追加链式链起来</br>
  3. 如果链中的Node节点大于8 转为红黑树

  步骤就是上面三步，我们先围绕如何找到哈希桶数组下标位置，并且使用了哪些手段。
  
#### 寻找key的hashcode对应哈希桶数组的index
-

- 记住重点，HashMap中数组的长度总是2的n次幂
- Java中32位字节


1. 确定哈希桶数组索引位置

先看一段源码：

```
static final int hash(Object key) {
     int h;
     // h = key.hashCode() 为第一步 取hashCode值
     // h ^ (h >>> 16)  为第二步 高位参与运算
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

static int indexFor(int h, int length) {
     return h & (length-1);  //第三步 取模运算
}

```

解释下：
hash(Object key)中是取得key对象的hashcode，
优化了高位运算的算法，通过hashCode()的高16位异或低16位实现的：
(h = k.hashCode()) ^ (h >>> 16)
右移16位，自己的高半区和低半区异或，就是为了混合原始哈希码的高位和低位，以此来加大低位随机性。
，主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销。

{% asset_img img/HashMap_hashcode.png %}

- 得到hashcode值，换算成2进制
- 将值右移16位，就成了高16位与低16位进行异或运算
- 得到异或后的hashcode

这样获取的hashcode的随机性更强，也就是说产生hash碰撞的几率更低。

获取哈希桶数组的index
计算机中下面2个是等价的

```
a%(2^n) 等价于 a&(2^n-1)
```

举个例子：

n = 4；
a = 23；

23 % 16 = 7 (模运算)

我们转换成 & 运算，先转换成二进制

23                   ->     16 + 4 + 2 + 1   ->  10111

2^n-1 = 16-1 = 15.   ->     8 + 4 + 2 + 1    ->  01111

进行 &  运算

```
10111
01111
-----
00111  -> 2^2 + 2^1 + 2^0 = 4 + 2 + 1 = 7

```

经过以上步骤，我们就能给从一个key的hashcode值，通过(h = key.hashCode()) ^ (h >>> 16) & (length-1)
计算key对应哈希桶数组的index。

HashMap是利用了数组的取值的时间复杂度为O(1),然后通过算法，利用hashcode来找到对应的index，最好的算法是key均匀的分布在数组桶上。最差的算法是所有的key都分布在一个index里面。


#### Java 1.8 中的HashMap resize


使用的是2次幂的扩展(指长度扩为原来2倍)，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。看下图可以明白这句话的意思，n为table的长度，图（a）表示扩容前的key1和key2两种key确定索引位置的示例，图（b）表示扩容后key1和key2两种key确定索引位置的示例，其中hash1是key1对应的哈希与高位运算结果。

{% asset_img img/hashMap1.8Resize.png %}


扩展成一倍后，我们的高位增加了1个bit，新的key的hashcode计算后对应的index就发生新的变化：

{% asset_img img/hashMap1.8cap.png %}

因此，我们在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，可以看看下图为16扩充为32的resize示意图：

{% asset_img img/HashMap1.8exchang.png %}

所以在Java 1.8中HashMap的效率更高，省去了重新计算的时间


#### HashMap的线程不安全



  
