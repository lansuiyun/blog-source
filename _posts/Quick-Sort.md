title: Quick Sort
date: 2015-11-16 20:10:11
tags: [algorithm,sort]
categories: algorithm
description: 快速排序
toc: true
---

[快速排序][quickSort](Quicksort或 partition-exchange sort)是Tony Hoare Hoare于1961年提出的一种划分交换排序。它采用了一种分治的策略，通常称其为[分治法(Divide and conquer algorithms)][divide-and-conquer]。

### 原理
快速排序使用分治法（Divide and conquer）策略来把一个序列（list）分为两个子序列（sub-lists）。

![快速排序][quicksort_anim]
<center><font size=2>图片来源于[维基百科][quickSort]</font></center>
步骤为：

1. 从数列中挑出一个元素，称为"基准"（pivot）。
2. 重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准值大的摆在基准的后面（相同的数可以到任一边）。在这个分区结束之后，该基准就处于数列的中间位置。这个称为分区（partition）操作。此时，"基准"也处于最终的位置。
3. 对基准值元素的左右子数列重复进行1 2操作，直到每个子数列的长度为1或0。

递归的最底部情形，是数列的大小是零或一，也就是永远都已经被排序好了。虽然一直递归下去，但是这个算法总会结束，因为在每次的迭代（iteration）中，它至少会把一个元素摆到它最后的位置去。
<!-- more -->


### 示例
一次分区的具体步骤：
1. 取数组`a`最后一个元素作为基准`pivot`
2. 对数组由左至右循环，设置`s(storeIndex)=0`为下一个小于基准元素的位置，`i=0`为循环变量
3. 如果`a[i] < pivot`,a[s]、array[i]交换位置，且`s++`
4. 直到循环到最后一个元素，也就是pivot，交换a[s]与pivot的位置，pivot最终位置确定

本次分区结束

以[38，65，97，76，13，27，49]为例，一次分区过程如下：
![快速排序一次分区][quick_sort_parition]

一次分区结束后，取基准左侧子数列(38,13,27),右侧子数列(65,97,76)继续分区，直到每个子数列的长度为1或0，也即所有元素都已经处于正确的位置，排序结束

---
### java 实现
```java
/**
 * 数组内元素交换
 *
 * @param array
 * @param index1
 * @param index2
 * @param <T>
 */
private static <T extends Comparable<T>> void swap(T[] array, int index1, int index2) {
    if (index1 == index2) {
        return;
    }
    T temp = array[index1];
    array[index1] = array[index2];
    array[index2] = temp;
}

/**
 * 快速排序
 *
 * @param array
 * @param start
 * @param end
 * @param <T>
 */
public static <T extends Comparable<T>> void quickSort(T[] array, int start, int end) {
    if (end <= start) {
        return;
    }
    int storeIndex = start;
    int loopIndex = start;
    for (; loopIndex < end; loopIndex++) {
        if (array[loopIndex].compareTo(array[end]) <= 0) {
            swap(array, loopIndex, storeIndex);
            storeIndex++;
        }
    }
    swap(array, end, storeIndex);
    quickSort(array, start, storeIndex - 1);
    quickSort(array, storeIndex + 1, end);
}

public static <T extends Comparable<T>> void quickSort(T[] array) {
    quickSort(array, 0, array.length - 1);
}
```
---
### 改进
快速排序每次确定一个元素（基准pivot）的最终位置，并将小于、大于等于基准的元素分别放置在左右两侧。但与基准相等元素并没有放在最终的位置，之后还要参与分区。可以在一次分区的过程中同时将与基准相等的元素放置在最终位置，确保左右分区只有小于、大于基准的元素，减少运行时间。
#### java实现
```java
/**
 * 快速排序,同时处理相等元素
 *
 * @param array
 * @param start
 * @param end
 * @param <T>
 */
private static <T extends Comparable<T>> void quickSortWithSamePartition2(T[] array, int start, int end) {
    if (end <= start) {
        return;
    }
    int lt = start;
    int gt = end;
    T pivot = array[end];
    int i = end - 1;
    while (i >= lt) {
        int compareResult = array[i].compareTo(pivot);
        if (compareResult < 0) {
            swap(array, lt, i);
            lt++;
        } else if (compareResult > 0) {
            swap(array, gt, i);
            gt--;
            i--;
        } else {
            i--;
        }
    }

    quickSortWithSamePartition2(array, start, lt - 1);
    quickSortWithSamePartition2(array, gt + 1, end);
}

public static <T extends Comparable<T>> void quickSortWithSamePartition2(T[] array) {
    quickSortWithSamePartition2(array, 0, array.length - 1);
}

/**
 * 快速排序,同时处理相等元素
 *
 * @param array
 * @param start
 * @param end
 * @param <T>
 */
private static <T extends Comparable<T>> void quickSortWithSamePartition(T[] array, int start, int end) {
    if (end <= start) {
        return;
    }
    int storeIndex = start;
    int loopIndex = start;
    int sameCount = 0;
    for (; loopIndex < end; loopIndex++) {
        int compareResult = array[loopIndex].compareTo(array[end]);
        if (compareResult < 0) {
            swap(array, loopIndex, storeIndex);
            storeIndex++;
        } else if (compareResult == 0) {
            sameCount++;
        }
    }
    swap(array, end, storeIndex);
    int pivotIndex = storeIndex;
    int rightStart = storeIndex + 1;
    if (sameCount > 0) {
        for (int i = rightStart; i <= end && sameCount > 0; i++) {
            if (array[i].compareTo(array[pivotIndex]) == 0) {
                swap(array, i, rightStart);
                rightStart++;
                sameCount--;
            }
        }
    }

    quickSortWithSamePartition(array, start, pivotIndex - 1);
    quickSortWithSamePartition(array, rightStart, end);
}


public static <T extends Comparable<T>> void quickSortWithSamePartition(T[] array) {
    quickSortWithSamePartition(array, 0, array.length - 1);
}
```
另外，在分区内元素较少时，可以使用其他的排序方法，而不是继续递归调用快速排序。

### 分析
* 最差时间复杂度	θ(n^2)
* 最优时间复杂度	θ(n log n)
* 平均时间复杂度	θ(n log n)

在平均状况下，排序n个项目要Ο(n log n)次比较。在最坏状况下则需要Ο(n2)次比较，但这种状况并不常见。事实上，快速排序通常明显比其他Ο(n log n)算法更快，因为它的内部循环（inner loop）可以在大部分的架构上很有效率地被实现出来。



[quickSort]: https://zh.wikipedia.org/wiki/%E5%BF%AB%E9%80%9F%E6%8E%92%E5%BA%8F
[divide-and-conquer]: https://zh.wikipedia.org/wiki/%E5%88%86%E6%B2%BB%E6%B3%95
[quicksort_anim]: http://7u2sbw.com1.z0.glb.clouddn.com/Sorting_quicksort_anim.gif
[quick_sort_parition]: http://7u2sbw.com1.z0.glb.clouddn.com/quick_sort_parition.png
