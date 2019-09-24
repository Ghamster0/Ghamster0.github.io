---
title: Java源码之排序算法
date: 2019-09-20 18:31:32
tags:
- Java
- 源码
---

常见的排序算法主要包括冒泡、插入、归并、希尔以及快排，教科书上也给出了算法的简单实现和原理解析。在实际使用中，编程语言提供的排序算法，通常是这些算法相结合的优化版本

Java的排序方法集中在`java.util.Arrays`类，主要有三大类：

- 对基本类型数组排序算法，封装在`java.util.DualPivotQuicksort`
- 对泛型数组的排序算法，封装在`java.util.TimSort`
- 多线程排序方法`parallelSort`

本文将结合源码，分析`DualPivotQuicksort`排序算法，源码为`Oracle jdk1.8 java.util.DualPivotQuicksort`

<!-- more -->

## DualPivotQuicksort 类

虽然这个类的名字叫做*双枢轴快排*，但实际上这个类封装了上面提到的大多数排序算法。排序方法`sort`可以根据一些列策略，自动选取合适的算法对数组排序

该类定义了几个常量：  

| constant                                  | value |
| ----------------------------------------- | ----- |
| QUICKSORT_THRESHOLD                       | 286   |
| INSERTION_SORT_THRESHOLD                  | 47    |
| COUNTING_SORT_THRESHOLD_FOR_BYTE          | 29    |
| COUNTING_SORT_THRESHOLD_FOR_SHORT_OR_CHAR | 3200  |

> 计数排序特别适合待排序数组特别长，类型本身可表示的值个数相对较少的情况，如`byte`。与其他类型数组排序相比，`byte`数组的排序多了一个判断步骤——当数组长度超过30（right-left+1）时，会触发计数排序。由于计数排序相对简单，在此不多做赘述。

以`int`型数组为例，`sort`方法有两种签名，分别是：

- `sort(int[] a, int left, int right, int[] work, int workBase, int workLen)`，访问权限为package，实现了归并排序算法。当数组长度较短时，调用下面的方法
- `sort(int[] a, int left, int right, boolean leftmost)`，访问权限private，实现了快排和插入排序

总结起来，排序算法选择策略如下（待排序数组长度L）：

1. L >= QUICKSORT_THRESHOLD：采用归并排序
2. L < QUICKSORT_THRESHOLD && L >= INSERTION_SORT_THRESHOLD：采用双枢轴快排
3. L < INSERTION_SORT_THRESHOLD：采用插入排序

## Merge Sort

使用归并排序的条件有三条：

1. 数组长度超过QUICKSORT_THRESHOLD
2. 数组局部有序
3. 数组不存在较多连续相等的元素

### 有序性判定

源码119~148行：

```java
int[] run = new int[MAX_RUN_COUNT + 1];
int count = 0; run[0] = left;

// Check if the array is nearly sorted
for (int k = left; k < right; run[count] = k) {
    if (a[k] < a[k + 1]) { // ascending
        while (++k <= right && a[k - 1] <= a[k]);
    } else if (a[k] > a[k + 1]) { // descending
        while (++k <= right && a[k - 1] >= a[k]);
        for (int lo = run[count] - 1, hi = k; ++lo < --hi; ) {
            int t = a[lo]; a[lo] = a[hi]; a[hi] = t;
        }
    } else { // equal
        for (int m = MAX_RUN_LENGTH; ++k <= right && a[k - 1] == a[k]; ) {
            if (--m == 0) {
                sort(a, left, right, true);
                return;
            }
        }
    }

    /*
     * The array is not highly structured,
     * use Quicksort instead of merge sort.
     */
    if (++count == MAX_RUN_COUNT) {
        sort(a, left, right, true);
        return;
    }
}
```

归并排序优化的基本思路是，将原数组中原本有序的子序列检测出来，以这些子序列为初始划分进行排序。从而避免像传统的归并排序算法，为保证初始子序列有序，子序列长度只能设为1

数组`run`用来记录序列中每段有序子序列的坐标范围，如`run[0]==0 && run[1]==3`表示第一段有序序列起始索引为0，终止索引为2；变量`count`记录已检出的有序子序列个数

for循环检索有序序列，并将值存储到`run`和`count`：

1. 前两个元素递增，检索后续递增元素

2. 前两个递减，检索后续递减，然后将这段子序列reverse

3. 前两个相等，检索后续相等。这里有个特殊情况，当连续相等的元素数量超过MAX_RUN_LENGTH时，认为相等元素过多，不再使用归并，而是调用快排方法解决

当`count`值超过MAX_RUN_COUNT时，同样退出归并，调用快排进行排序。因为`count`过大意味着平均有序子序列较短，即数组基本无序

> 源码152~156处理了一下边界情况，当以上for循环最后一次执行结束后，如果刚好`k==right`，执行递增`run[count] = k`，表示发现一组有序序列，索引是`*`到`right-1`；下一次循环条件`k<right`不满足，for循环退出。但实际上，数组元素`a[right]`是由一个元素组成的有序序列，因对应`run`数组末尾需要添加一个值为`right+1`的元素

### 归并算法

归并算法空间复杂度为O(n)，即必须有和待排序数组等大的辅助数组空间。入参的`work`变量就是用作辅助数组，数组可用空间可以不从0索引开始，但必须连续；`workBase`表示起始地址，`workLen`表示可用长度。传入的工作数组可能为空，此时可以自己创建一个。

`sort`方法没有返回值，所以我们希望归并结束时，刚好数据由辅助数组复制到原数组。因此需要根据有序子序列个数计算出需要归并的次数：

```java
byte odd = 0;
for (int n = 1; (n <<= 1) < count; odd ^= 1);
```

当`odd==1`，即`n`左移2<sup>k</sup>+1次，需要归并奇数次，第一次归并应从辅助数组到原数组；反之亦然。源码618~628行即处理这个问题

源码631~651根据`run`数组保存的信息执行归并，每轮归并后把新的有序序列信息覆盖到`run`中保存：

```java
for (int last; count > 1; count = last) {
    for (int k = (last = 0) + 2; k <= count; k += 2) {
        int hi = run[k], mi = run[k - 1];
        for (int i = run[k - 2], p = i, q = mi; i < hi; ++i) {
            if (q >= hi || p < mi && a[p + ao] <= a[q + ao]) {
                b[i + bo] = a[p++ + ao];
            } else {
                b[i + bo] = a[q++ + ao];
            }
        }
        run[++last] = hi;
    }
    if ((count & 1) != 0) {
        for (int i = right, lo = run[count - 1]; --i >= lo;
            b[i + bo] = a[i + ao]
        );
        run[++last] = right;
    }
    long[] t = a; a = b; b = t;
    int o = ao; ao = bo; bo = o;
}
```

这里有几个需要注意的点：由于归并后子序列数量减半，实际上`run`的后1/2空间还保存着原有的子序列信息，但for循环通过`count`确定循环边界，所以每次归并结束只要更新`count`即可；若本轮`count`为奇数，则最后一个有序序列需要原样复制到目标数组中

## Quick Sort

当数组长度小于QUICKSORT_THRESHOLD或上文列举的其他合适情况时，private的排序方法`sort(int[] a, int left, int right, boolean leftmost)`被调用

该方法大致有四种排序逻辑：insertion sort，pair insertion sort，quick sort，dual pivot quick sort。前两种用于待排序数组长度小于INSERTION_SORT_THRESHOLD的情况，后两种相反。由于快排是递归排序，实际在排序的最后阶段是由插入排序完成的。

### 插入排序

`leftmost`参数，顾名思义，待排序子序列是否在数组最左侧。`leftmost`为`true`时，使用传统的插入排序（这里不再展开描述），否则使用pair insertion sort。

分析代码，在方法触发快排逻辑后，得到被*枢*分隔开的2或3个子序列排序，随后执行递归，只有非最左侧的子序列递归方法入参`leftmost`为`false`，其余所有情况都为`true`。因此根据快排的性质可以确定，当`leftmost`为`false`时，子序列中的所以元素均大于等于其左侧的所有元素，所以其左侧元素均可充当守卫，而不必每轮循环检查索引越界

pair insertion sort源码在253~274：

```java
do {
    if (left >= right) {
        return;
    }
} while (a[++left] >= a[left - 1]);

for (int k = left; ++left <= right; k = ++left) {
    int a1 = a[k], a2 = a[left];

    if (a1 < a2) {
        a2 = a1; a1 = a[left];
    }
    while (a1 < a[--k]) {
        a[k + 2] = a[k];
    }
    a[++k + 1] = a1;

    while (a2 < a[--k]) {
        a[k + 1] = a[k];
    }
    a[k + 1] = a2;
}
int last = a[right];

while (last < a[--right]) {
    a[right + 1] = a[right];
}
a[right + 1] = last;
```

pair insertion sort改进的思想是，每次取两个元素，先对较大的进行插入排序；当找到插入位置后，由于较小插入位置一定更靠前，所以可以从当前位置继续执行插入排序算法，而不用回到有序序列的尾部重新比较。

代码在排序开始前`do...while`跳过了序列开头已有序的部分，同时`left++`，这是由于上面提到左侧元素可以充当守卫的原因，不再需要`left`记录待排序序列的起始位置

### 双枢轴快排

当待排序数组长度大于INSERTION_SORT_THRESHOLD，使用快排

源码279~312行处理枢轴选择逻辑：

- 取`seventh`为待排序序列长度近似1/7，`e3`为中点索引
- 取`e3`向前向后`seventh, 2*seventh`长度位置索引，最终得到`e1~e5`
- 对`e1~e5`位置的元素执行插入排序（这里没用for循环，直接手写完整插入排序，简直太暴力了）
- 判等，当`e1~e5`均不想等时，取`e2 e4`为枢，执行dual pivot quick sort；否则取`e3`为枢，执行普通快排（这里不在详细描述）

双枢轴快排原理如下：

- 快排需要两个枢`pivot1`和`pivot2`，三个指针(索引)`less k great`，三个指针将数组分为四个部分，参考源码：

```java
/*
* Partitioning:
*
*   left part           center part                   right part
* +--------------------------------------------------------------+
* |  < pivot1  |  pivot1 <= && <= pivot2  |    ?    |  > pivot2  |
* +--------------------------------------------------------------+
*               ^                          ^       ^
*               |                          |       |
*              less                        k     great
*
* Invariants:
*
*              all in (left, less)   < pivot1
*    pivot1 <= all in [less, k)     <= pivot2
*              all in (great, right) > pivot2
*/
outer:
for (int k = less - 1; ++k <= great; ) {
    int ak = a[k];
    if (ak < pivot1) { // Move a[k] to left part
        a[k] = a[less];
        a[less] = ak;
        ++less;
    } else if (ak > pivot2) { // Move a[k] to right part
        while (a[great] > pivot2) {
            if (great-- == k) {
                break outer;
            }
        }
        if (a[great] < pivot1) { // a[great] <= pivot2
            a[k] = a[less];
            a[less] = a[great];
            ++less;
        } else { // pivot1 <= a[great] <= pivot2
            a[k] = a[great];
        }
        a[great] = ak;
        --great;
    }
}
```

> 排序开始前，`less`从`left`开始向右移动，跳过`<pivot1`的元素；同理`great`向左跳过
> 注意边界情况，`less`指示的是center part的起始位置，即`pivot1<=a[less]<=pivot2`

- 初始`k`等于`less`，即center part长度为0；`k`和`great`之间的元素即为待排序元素；当`k==great`本轮快排结束
- `k`指示的值有三种情况：
  - `pivot1 <= a[k] <= pivot2`：`a[k]`属于center part，只需执行`k++`
  - `a[k]<pivot1`：`a[k]`属于left part，`a[less]`和`a[k]`交换，执行`less++, k++`
  - `a[k]>pivot2`：`a[k]`属于right part，此时情况有些复杂。首先将`great`向左移动，找到一个`a[great]<=pivot2`的元素，分两种情况：
    - `pivot1<=a[great]<=pivot2`，则交换`a[k]`和`a[great]`即可，然后`k++, great--`
    - `a[great]<pivot1`，首先`a[great]`赋值给`a[less]`，扩展left part区`less++`；`a[less]`原值赋给`a[k]`，`k++`相当于center part向右平移一个单位；`a[k]`原值赋给`a[great]`，`great--`
- 快排结束后，递归调用`sort`对子序列排序

源码中递归前，对于center part进行了特殊处理——判断`less < e1 && e5 < great`值，若该值为`true`则center part长度是否超出原数组长度4/7
理论上通过上述枢轴选择策略，`pivot1 pivot2`应当将原数组分割为基本相等的三份，如果center part较大，则问题很可能出现在边界条件`=`上，即center part中有大量相等元素。可以通过排除与`pivot1`和`pivot2`相等的元素，大幅减少center part中需要排序的长度。这部分算法可以看作变形版的快排，参考源码注释：

```java
/*
* Partitioning:
*
*   left part         center part                  right part
* +----------------------------------------------------------+
* | == pivot1 |  pivot1 < && < pivot2  |    ?    | == pivot2 |
* +----------------------------------------------------------+
*              ^                        ^       ^
*              |                        |       |
*             less                      k     great
*
* Invariants:
*
*              all in (*,  less) == pivot1
*     pivot1 < all in [less,  k)  < pivot2
*              all in (great, *) == pivot2
*/
```

随后只对[less, k-1]递归即可

## Conclusion

TimSort算法更神奇，有时间也写一下