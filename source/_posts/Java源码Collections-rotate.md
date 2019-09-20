---
title: Java源码Collections.rotate
date: 2019-09-20 17:14:26
tags:
- Java
- 源码
---

`Collections.rotate`用于列表的旋转(平移)操作，即列表中所有元素左移n位。rotate操作有两种实现方式，分别适用不同的情况：

- 当列表实现了`RandomAccess`，如列表底层为数组，或列表底层为双向链表，但元素不太多时：

  - 选取一个元素a，选取a后第n位元素b，将a放入b所在位置；选取b后第n位元素c，将b放入c所在位置......直到所有元素都被移动

  - 为确保所有元素都被移动，需要记录当前已移动次数，和起始元素。当下一个位置刚好与起始元素位置相同且移动次数小于总元素个数时，说明出现列表长度为n的整数倍，需要重新选取还未移动的位置，作为新的起始点，继续操作

<!-- more -->

- 当列表底层为双向链表且较长时：
  
  - 使用翻转操作实现，首先对前`l-n`位翻转，然后对后n位翻转，最后整体翻转
  - 翻转通过`Collections.reverse`方法实现。对于双向链表，翻转操作需要一个首指针和一个尾指针，操作整体时间复杂度为O(n)

相关源码如下：

```java
/*
 * Copyright (c) 1997, 2014, Oracle and/or its affiliates. All rights reserved.
 * ORACLE PROPRIETARY/CONFIDENTIAL. Use is subject to license terms.
 */
package java.util;

public class Collections {
    public static void reverse(List<?> list) {
        int size = list.size();
        if (size < REVERSE_THRESHOLD || list instanceof RandomAccess) {
            for (int i=0, mid=size>>1, j=size-1; i<mid; i++, j--)
                swap(list, i, j);
        } else {
            ListIterator fwd = list.listIterator();
            ListIterator rev = list.listIterator(size);
            for (int i=0, mid=list.size()>>1; i<mid; i++) {
                Object tmp = fwd.next();
                fwd.set(rev.previous());
                rev.set(tmp);
            }
        }
    }
    
    public static void rotate(List<?> list, int distance) {
        if (list instanceof RandomAccess || list.size() < ROTATE_THRESHOLD)
            rotate1(list, distance);
        else
            rotate2(list, distance);
    }

    private static <T> void rotate1(List<T> list, int distance) {
        int size = list.size();
        if (size == 0)
            return;
        distance = distance % size;
        if (distance < 0)
            distance += size;
        if (distance == 0)
            return;

        for (int cycleStart = 0, nMoved = 0; nMoved != size; cycleStart++) {
            T displaced = list.get(cycleStart);
            int i = cycleStart;
            do {
                i += distance;
                if (i >= size)
                    i -= size;
                displaced = list.set(i, displaced);
                nMoved ++;
            } while (i != cycleStart);
        }
    }

    private static void rotate2(List<?> list, int distance) {
        int size = list.size();
        if (size == 0)
            return;
        int mid =  -distance % size;
        if (mid < 0)
            mid += size;
        if (mid == 0)
            return;

        reverse(list.subList(0, mid));
        reverse(list.subList(mid, size));
        reverse(list);
    }
}
```