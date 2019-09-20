---
title: Arrays数组判等 equals与mismatch
date: 2019-03-18 10:34:48
tags:
- Java
- 源码
---

## 概述  
- `Arrays`类是Java中用于数组操作的工具类，实现了比较、复制、查找和排序等常用操作
- `equals`方法用于判断数组是否相等，包含了各种基本类型和`Object`类型的重载版本，对于`Object`数组`Object.equals`进行判断
- `ArraysSupport`类是`jdk11`（似乎是在`jdk9`引入）的工具类，提供了`mismatch`方法，用于检查两个被判定的数组对象中，首个不相等元素的下标
- `jdk11`中优化了`equals`方法的实现逻辑，在数组元素为基本数据类型的情况下，使用`mismatch`方法替代逐个元素比较的方式

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

## ArraysSupport.mismatch
查看`ArraysSupport`源码，发现整体逻辑由`mismatch`和`vectorizedMismatch`两个方法实现

`vectorizedMismatch`方法对比数组内容，返回值有两种：`>0`表示首个不匹配元素；`<=0`剩余未处理元素个数(取反)。

`mismatch`返回两数组首个不匹配元素的索引，若匹配则返回-1。具体逻辑是调用`vectorizedMismatch`，如果得到正值则直接返回，否则逐个比较`vectorizedMismatch`剩余的未处理元素

`mismatch`方法源码如下：

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

## ArraysSupport.vectorizedMismatch

> 该方法标注了 `@HotSpotIntrinsicCandidate` 注解，即`jvm`可以有自己的优化实现方式

该方法是这项优化的核心所在，基本思路是增加单次比较处理的数据量

比如，`short[]`每个元素长度为2字节，`int[]`元素长度为4字节，`long[]`元素长度为8字节。对于一个长度为8的`short[]`，逐个元素比较需要8次，每次2字节；假如把这个数组在内存中的二进制结构看作`long[]`，则相当于长度为2的数组，只需两次比较就能处理完

这时候需要处理两个问题：1. 假如`short[]`的第7个元素不匹配，当按照`long[]`处理时，只能发现这个“长整形数组”的第2个元素不匹配，但这对应原数组的5~8位，需要根据系统的大小端排列顺序等信息进一步换算；2. 假如`short[]`长度为9，则剩余1位元素不能处理

方法的源码如下：

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
- `long aOffset`, `long bOffset`：数组中元素的起始`Offset`。由于数组对象在内存中存储需要保存额外信息，例如对象类型、长度等，所以数组中存放的值并不是从该对象所占内存的0地址开始。比如本例中，传入的参数为`Unsafe.ARRAY_INT_BASE_OFFSET`，通过debug发现在当前平台上该值为16
- `length`：需要对比的长度。该方法并不提供数组下标越界检查，因此`length`必须是合理的数值
- `log2ArrayIndexScale`：数组下标缩放，`long`长度为8字节，缩放为3(`8=1<<3`)；`int`长度为4字节，缩放为2；`short`缩放为1

方法操作逻辑如下，以`int[]`为例：
- 取`wi`为将被处理数组看作`long[]`的情况下，数组具有的元素个数。即用`long[]`相对原数组的缩放，对原数组的长度缩放
- `for`循环，以`long`的长度步进，使用`getIntUnaligned`方法以非对齐的方式直接读取内存中的8字节数据(`av`, `bv`)，对比两数组
- 当`av`和`bv`不等时，则发现不匹配。两值异或，得到形如`0000000000010000`的二级制数据，然后根据是大端存储还是小端存储决定两侧的0值哪一侧应排在原数组不匹配元素前面，对这一侧0的个数进行符合原数组的所以缩放，然后返回
- 如果剩余元素不够一次处理，则返回剩余元素取反
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

实测确实没有带来性能提升，但考虑到方法标注了`@HotSpotIntrinsicCandidate`，这个测试结果参考价值不大