---
title: 找出第n大的数
date: 2019-09-20 18:07:28
tags: 算法杂谈
---

## Problem

给定一个整数数组，数组长度L，找出数组中第n大的数

<!-- more -->

## Solution

思路如下：

- 遍历整个数组，使用一个大小为n的列表T保存已遍历的元素中最大的n个数
- 为减少一个元素T中元素的比较次数，对T中元素排序
- 为减少维持T有序的成本，将T实现为小顶堆

成本：

- 时间复杂度：O(L*lg n)
- 空间复杂度：O(n)

## Code

```java
import java.util.Arrays;

public class FindN {

    private int find(int[] list, int n) {
        // 基于数组的小顶堆，初始化为Integer.MIN_VALUE
        int[] tree = new int[n];
        Arrays.fill(tree, Integer.MIN_VALUE);
        for (int i = 0; i < list.length; i++) {
            if (list[i] > tree[0]) {
                tree[0] = list[i];
                adjustTree(tree, 0);
            }
        }
        return tree[0];
    }

    private int leftChildIndex(int length, int i) {
        i = i * 2 + 1;
        return i < length ? i : -1;
    }

    private int rightChildIndex(int length, int i) {
        i = i * 2 + 2;
        return i < length ? i : -1;
    }

    // 简易的堆调整算法
    private void adjustTree(int[] tree, int subRoot) {
        int l = tree.length;
        int leftIndex = leftChildIndex(l, subRoot);
        int rightIndex = rightChildIndex(l, subRoot);
        int next;
        if (leftIndex > 0 && rightIndex > 0) {
            next = tree[leftIndex] < tree[rightIndex] ? leftIndex : rightIndex;
        } else if (leftIndex > 0) {
            next = leftIndex;
        } else {
            next = rightIndex;
        }
        if (next > 0) {
            if (tree[next] < tree[subRoot]) {
                swap(tree, next, subRoot);
                adjustTree(tree, next);
            }
        }
    }

    private void swap(int[] tree, int i, int j) {
        int tmp = tree[i];
        tree[i] = tree[j];
        tree[j] = tmp;
    }

    public static void main(String args[]) {
        FindN f = new FindN();
        System.out.println(f.find(new int[]{1, 2, 3, 13, 5, 6, 7, 8, 9, 10, 11, 12}, 5));
    }

}

```

