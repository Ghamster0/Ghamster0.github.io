---
title: Arrays.equals与ArraysSupport.vectorizedMismatch方法
date: 2019-03-18 10:34:48
tags:
- Java
- 源码
---

## 概述  
- `Arrays`类是Java中用于数组操作的工具类，实现了比较、复制、查找和排序等常用操作
- `equals`方法用于判断数组是否相等，包含了各种基本类型和`Object`类型的重载版本，对于`Object[]`会判断对象非空，并调用`Object.equals`判断对象是否相等
- `ArraysSupport`类是`jdk11`（似乎是在`jdk9`引入）的工具类，提供了`mismatch`方法和`vectorizedMismatch`。`mismatch`方法可以返回两个被判定的数组对象中，首个不相等元素的下标
- `jdk11`中使用`mismatch`方法修改了`equals`方法的实现逻辑

<!-- more -->

## Arrays类的equals方法实现  
`jdk8`中，数组判等的逻辑是：遍历数组中的元素，依次对每个元素判等
以`int[]`为例，源码如下：

```java
    public static boolean equals(int[] a, int[] a2) {
        if (a==a2)
            return true;
        if (a==null || a2==null)
            return false;

        int length = a.length;
        if (a2.length != length)
            return false;

        for (int i=0; i<length; i++)
            if (a[i] != a2[i])
                return false;

        return true;
    }
```
`jdk11`中，修改了所有用于基本数据类型的数组的`equals`方法，使用`mismatch`方法实现
以`int[]`为例，源码如下：
```java
    public static boolean equals(int[] a, int[] a2) {
        if (a==a2)
            return true;
        if (a==null || a2==null)
            return false;

        int length = a.length;
        if (a2.length != length)
            return false;

        return ArraysSupport.mismatch(a, a2, length) < 0;
    }
```

## mismatch和vectorizedMismatch  
为了探究`jdk11`中`equals`的魔法，必须看看`mismatch`方法到底做了什么。`mismatch`方法也拥有各种基本数据类型数组的重载版本，以`int[]`版本为例

### mismatch方法 
`mismatch`方法返回给定的两个数组中，首个不相同元素的下标，当完全匹配（0到length-1）时，返回-1，源码如下：
```java
    public static int mismatch(int[] a,
                               int[] b,
                               int length) {
        int i = 0;
        if (length > 1) {
            if (a[0] != b[0])
                return 0;
            i = vectorizedMismatch(
                    a, Unsafe.ARRAY_INT_BASE_OFFSET,
                    b, Unsafe.ARRAY_INT_BASE_OFFSET,
                    length, LOG2_ARRAY_INT_INDEX_SCALE);
            if (i >= 0)
                return i;
            i = length - ~i;
        }
        for (; i < length; i++) {
            if (a[i] != b[i])
                return i;
        }
        return -1;
    }
```
方法主要可以分为两个部分：
1. 调用`vectorizedMismatch`方法，对两个数组对象匹配，若方法返回正值，则表示找到了未匹配的元素，返回值为该元素在数组中的索引；若方法返回负值，则表示未找到不相等元素，但需要注意的是，`vectorizedMismatch`方法可能没有处理完数组中的所有元素，对返回值取反，得到还需处理的数组元素个数
2. 对`vectorizedMismatch`返回值取反，对剩余元素逐个验证

### vectorizedMismatch

> 该方法标注了 @HotSpotIntrinsicCandidate 注解，即jvm可以有自己的优化实现方式，所以...懂吧...?

该方法的核心思想是以`long`数据格式对比对象（不一定是数组）中的数据，即每次循环处理数据长度为8个字节，而不考虑对象是`long[]`还是`int[]`或`short[]`等。方法的源码如下：
```java
@HotSpotIntrinsicCandidate
    public static int vectorizedMismatch(Object a, long aOffset,
                                         Object b, long bOffset,
                                         int length,
                                         int log2ArrayIndexScale) {
        // assert a.getClass().isArray();
        // assert b.getClass().isArray();
        // assert 0 <= length <= sizeOf(a)
        // assert 0 <= length <= sizeOf(b)
        // assert 0 <= log2ArrayIndexScale <= 3

        int log2ValuesPerWidth = LOG2_ARRAY_LONG_INDEX_SCALE - log2ArrayIndexScale;
        int wi = 0;
        for (; wi < length >> log2ValuesPerWidth; wi++) {
            long bi = ((long) wi) << LOG2_ARRAY_LONG_INDEX_SCALE;
            long av = U.getLongUnaligned(a, aOffset + bi);
            long bv = U.getLongUnaligned(b, bOffset + bi);
            if (av != bv) {
                long x = av ^ bv;
                int o = BIG_ENDIAN
                        ? Long.numberOfLeadingZeros(x) >> (LOG2_BYTE_BIT_SIZE + log2ArrayIndexScale)
                        : Long.numberOfTrailingZeros(x) >> (LOG2_BYTE_BIT_SIZE + log2ArrayIndexScale);
                return (wi << log2ValuesPerWidth) + o;
            }
        }

        // Calculate the tail of remaining elements to check
        int tail = length - (wi << log2ValuesPerWidth);

        if (log2ArrayIndexScale < LOG2_ARRAY_INT_INDEX_SCALE) {
            int wordTail = 1 << (LOG2_ARRAY_INT_INDEX_SCALE - log2ArrayIndexScale);
            // Handle 4 bytes or 2 chars in the tail using int width
            if (tail >= wordTail) {
                long bi = ((long) wi) << LOG2_ARRAY_LONG_INDEX_SCALE;
                int av = U.getIntUnaligned(a, aOffset + bi);
                int bv = U.getIntUnaligned(b, bOffset + bi);
                if (av != bv) {
                    int x = av ^ bv;
                    int o = BIG_ENDIAN
                            ? Integer.numberOfLeadingZeros(x) >> (LOG2_BYTE_BIT_SIZE + log2ArrayIndexScale)
                            : Integer.numberOfTrailingZeros(x) >> (LOG2_BYTE_BIT_SIZE + log2ArrayIndexScale);
                    return (wi << log2ValuesPerWidth) + o;
                }
                tail -= wordTail;
            }
            return ~tail;
        }
        else {
            return ~tail;
        }
    }
```
首先简单看一下方法的入参：
- `Object a`, `Object b`：需要对比的对象
- `long aOffset`, `long bOffset`：起始`Offset`，即从对象的第几个字节开始对比。比如本例中，被`mismatch`方法调用时，传入的参数为`Unsafe.ARRAY_INT_BASE_OFFSET`，通过debug发现在当前平台上该值为16，因为从第16字节开始，才是数组中的数据（0-15字节存储的应该是数组的length等信息）
- `length`：需要对比的长度。该方法并不提供数组下标越界检查，因此`length`必须是合理的数值
- `log2ArrayIndexScale`：数组下标缩放，与`length`共同确定需要对比的字节长度。以`int[]`为例，`int`型数据占4个字节，即`1<<2`，所以`log2ArrayIndexScale`为2，方法需要处理的字节长度为`length*(1<<2)=4*length`

方法操作逻辑如下(逐行；接上节，以`int[]`为例)：
- 12行： `log2ValuesPerWidth`可以理解为相对缩放。`LOG2_ARRAY_LONG_INDEX_SCALE`长整型数组下标缩放为3，即长8字节；`int[]`缩放为2，即长4字节；所以相对缩放为1
- 13行：`wi`，将对象视为`long[]`遍历时，当前数组下标
- 14行： `for`循环，每次`wi++`，即每次循环步进8个字节。`length`为数组包含的`int`数据个数，将素组视为`long`时，循环`length>>log2ArrayIndexScale`即可遍历数组
- 15行： `bi`数组下标为`wi`的元素在`long`数组中的相对位置（字节）
- 16-17行： 调用`UnSafe.getLongUnaligned`方法，读取一个`long`值。  
  在Java中，数据是对齐存放的，即`long`类型的数据存放时，起始地址只能是8的整数倍，`int`类型数据存放的起始地址只能是4的整数倍……  
  而`getLongUnaligned`方法顾名思义，可以读取非对齐存放的数据。该方法接收一个Object对象和一个long类型的offset：若offset为8的整数倍，即`(offset & 7) == 0`,则直接调用`getLong`返回长整型数据；若offset为4的整数倍,则直接分别对`offset`和`offset+4`调用`getInt`返回两个整型数据；若offset为2的整数倍……  
- 18-24行： 判定两个数组的值`av==bv`是否成立，相同则继续下一次循环；不同是使用`^`按位与或，并根据系统采用*大端存储*还是*小端存储*决定从头(numberOfLeadingZeros)还是尾(numberOfTrailingZeros)找到第一个不匹配的位。然后根据`log2ArrayIndexScale`的值，还原这个位在原数组`int[]`中对应的索引。`LOG2_BYTE_BIT_SIZE`值为3，表示一个字节共8位
- 27-50行： 用于处理数组类型为`byte[]`和`char[]`，且剩余未处理长度大于4字节（一个`int`）的情况，原理与前面类似。
  `tail`表示剩余未处理的元素个数，由于`UnSafe.getIntUnaligned`只能一次处理4个字节，因此可能剩余1-3个`byte`或1个`char`
  
## 性能测试  
下面通过简单代码测试一下相对于`jdk8`，`jdk11`是否有性能的提升，测试代码如下：
```java
package main.test;

import java.util.Arrays;
import java.util.Random;

public class TestLegacyArrays {

    private static Random r = new Random();

    private static class LegacyArrays {
        private LegacyArrays() {}

        public static boolean equals(int[] a, int[] a2) {
            if (a==a2)
                return true;
            if (a==null || a2==null)
                return false;

            int length = a.length;
            if (a2.length != length)
                return false;

            for (int i=0; i<length; i++)
                if (a[i] != a2[i])
                    return false;

            return true;
        }
    }

    private static void test(int size, int fill){
        int[] a = new int[size];
        Arrays.fill(a, fill);
        int[] b = new int[size];
        Arrays.fill(b, fill);

        boolean isEqual;
        long now, finish;
        System.out.println("=> TestLegacyArrays: array size "+size);

        now = System.nanoTime();
        isEqual = LegacyArrays.equals(a,b);
        finish = System.nanoTime();
        System.out.println("  LegacyArrays: "+isEqual+" time: "+(finish-now));

        now = System.nanoTime();
        isEqual = Arrays.equals(a,b);
        finish = System.nanoTime();
        System.out.println("  Arrays: "+isEqual+" time: "+(finish-now));
        System.out.println();
    }

    public static void main(String[] args) {
        
        test(10, r.nextInt());
        test(100, r.nextInt());
        test(1000, r.nextInt());
        test(10000, r.nextInt());
        test(100000, r.nextInt());

    }
}
```
代码输出如下：
```console
=> TestLegacyArrays: array size 10
  LegacyArrays: true time: 373700
  Arrays: true time: 193700

=> TestLegacyArrays: array size 100
  LegacyArrays: true time: 2000
  Arrays: true time: 10200

=> TestLegacyArrays: array size 1000
  LegacyArrays: true time: 15400
  Arrays: true time: 186500

=> TestLegacyArrays: array size 10000
  LegacyArrays: true time: 148900
  Arrays: true time: 286800

=> TestLegacyArrays: array size 100000
  LegacyArrays: true time: 1233700
  Arrays: true time: 2424000


Process finished with exit code 0
```

emmmmmmmmmmmmmmm...................
所以这次的更新，或许是出于流式计算或者多线程或者其他因素的考量，又或者我的测试方法有问题？。。。