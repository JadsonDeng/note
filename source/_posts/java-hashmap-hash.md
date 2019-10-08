---
title: HashMap中hash(Object key)的原理，为什么这样做?
date: 2019-10-08 14:36:42
tags: Java
---
HashMap中计算数组下标是HashMap的核心算法,今天在看HashMap源码中看到了hash(Object key)方法百思不得其解。查找相关博客，但是都只是说了为了尽量均匀,没有详细讲。

## HashMap中hash(Object key)的原理，为什么这样做?
先看下hash(Object key)方法
```
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

详细大家基本都能看懂，但是知道这一步(h = key.hashCode()) ^ (h >>> 16)原因的人很少。

首先这个方法的返回值还是一个哈希值。为什么不直接返回key.hashCode()呢？还要与 (h >>> 16)异或。首先要了解以下知识点：

必备知识点.：^ 运算  >>>运算  &运算。

讲到这里还要看一个方法indexFor，在jdk1.7中有indexFor(int h, int length)方法。jdk1.8里没有，但原理没变。下面看下1.7源码
1.8中用tab[(n - 1) & hash]代替但原理一样。

```
static int indexFor(int h, int length) {
    return h & (length-1);
}
```

这个方法返回值就是数组下标。我们平时用map大多数情况下map里面的数据不是很多。这里与（length-1）相&,

但由于绝大多数情况下length一般都小于2^16即小于65536。所以return h & (length-1);结果始终是h的低16位与（length-1）进行&运算。如下例子（hashcode为四字节）

例如1. 

length = 8;  （length-1） = 7；转换二进制为111；

hashcode = 78897121 转换二进制：100101100111101111111100001

    0000 0100 1011 0011 1101 1111 1110 0001

&运算

   0000 0000 0000 0000 0000 0000 0000 0111

= 0000 0000 0000 0000 0000 0000 0000 0001 （就是十进制1，所以下标为1）

由于和（length-1）运算，length 绝大多数情况小于2的16次方。**所以始终是hashcode 的低16位参与运算。要是高16位也参与运算，会让得到的下标更加散列。**

所以这样高16位是用不到的，如何让高16也参与运算呢。所以才有hash(Object key)方法。让他的hashCode()和自己的高16位^运算。所以(h >>> 16)得到他的高16位与hashCode()进行^运算。

重点来了，为什么用^：**因为&和|都会使得结果偏向0或者1 ,并不是均匀的概念,所以用^。**

这就是为什么有hash(Object key)的原因。
