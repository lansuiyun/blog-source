title: Merge Sort
date: 2015-11-23 15:18:11
tags: [algorithm,sort]
categories: algorithm
---

归并排序（Merge sort），是创建在归并操作上的一种有效的排序算法，效率为O(n log n)。1945年由约翰·冯·诺伊曼首次提出。该算法是采用分治法（Divide and Conquer）的一个非常典型的应用，且各层分治递归可以同时进行。

归并操作（merge），也叫归并算法，指的是将两个已经排序的序列合并成一个序列的操作。归并排序算法依赖归并操作。
<!-- more -->

归并排序过程如下：
![归并排序][Merge-sort]

---
### java实现

```
/**
 * 合并array中[start,mid][mid+1,end]两个有序子列
 *
 * @param array
 * @param start
 * @param mid
 * @param end
 * @param <T>
 */
public static <T extends Comparable<T>> void merge(T[] array, int start, int mid, int end) {
    if (mid >= end) {
        return;
    }
    if (array[mid].compareTo(array[mid + 1]) <= 0) {
        return;
    }
    int len = end - start + 1;
    T[] copy = (T[]) Array.newInstance(array.getClass().getComponentType(), len);
    System.arraycopy(array, start, copy, 0, len);
    int low = 0;
    int high = mid + 1 - start;
    int lowMax = mid - start;
    int highMax = end - start;
    for (int i = start; i <= end; i++) {
        if (low > lowMax) {
            System.arraycopy(copy, high, array, i, highMax - high + 1);
            break;
        }
        if (high > highMax) {
            System.arraycopy(copy, low, array, i, lowMax - low + 1);
            break;
        }
        if (copy[low].compareTo(copy[high]) <= 0) {
            array[i] = copy[low];
            low++;
        } else {
            array[i] = copy[high];
            high++;
        }
    }
}

public static <T extends Comparable<T>> void sort(T[] array, int start, int end) {
    if (end <= start) {
        return;
    }
    int mid = (end - start) / 2 + start;
    sort(array, start, mid);
    sort(array, mid + 1, end);
    merge(array, start, mid, end);
}

public static <T extends Comparable<T>> void sort(T[] array) {
    sort(array, 0, array.length - 1);
}
```

---
### 分析
* 最差时间复杂度	θ(nlog n)
* 最优时间复杂度	θ(n)
* 平均时间复杂度	θ(nlog n)
* 最差空间复杂度	θ(n)

归并排序是稳定排序

[Merge-sort]: http://7u2sbw.com1.z0.glb.clouddn.com/Merge-sort-example-300px.gif
