title: Shell Sort
date: 2015-11-14 13:05:04
tags: [algorithm,sort]
categories: algorithm
---

### 原理
希尔排序，也称递减增量排序算法，是插入排序的一种更高效的改进版本。希尔排序是非稳定排序算法。

具体步骤：
1. 取一个大于1的整数n1,将待排序元素分为n1组，间隔为n1的元素为同一组
2. 各组分别进行插入排序
3. 取正整数n2,满足n2 < n1,继续分组，并各组排序
4. 重复上述过程，直到nx = 1,此时，所有的待排序元素都为同一组，进行插入排序，至此，排序完成。

<!-- more -->
希尔排序是基于插入排序的以下两点性质而提出改进方法的：
* 插入排序在对几乎已经排好序的数据操作时，效率高，可以达到线性排序的效率
* 插入排序一般来说是低效的，因为插入排序每次只能将数据移动一位

希尔排序通过将比较的全部元素分为几个区域来提升插入排序的性能。这样可以让一个元素可以一次性地朝最终位置前进一大步。然后算法再取越来越小的步长进行排序，算法的最后一步就是普通的插入排序，但是到了这步，需排序的数据几乎是已排好的了（此时插入排序较快）。
  ![希尔排序][shellsort]

  <center><font size=2>图片来源于[维基百科][wiki]</font></center>

  ---
  ### 示例
  以[49，38，65，97，76，13，27，49，55，04]为例。
  第一次，取n1=5，则分为ABCDE五组

    A       C       E   A       C       E
    49  38  65  97  76  13  27  49  55  04
        B       D           B       D
ABCDE五组各自进行插入排序，结果为：

    A       C       E   A       C       E
    13  27  49  55  04  49  38  65  97  76
        B       D           B       D

第二次，取n2=3，则分为FGH三组

    F       H   F       H   F       H   F
    13  27  49  55  04  49  38  65  97  76
        G           G           G   
FGH各自进行插入排序，结果为：

    F       H   F       H   F       H   F
    13  04  49  38  27  49  55  65  97  76
        G           G           G   

 最后，取n1=1，此时，所有元素为一组，进行最终的插入排序

    04  13  27  38  49  49  55  65  76  97  

排序结束

[动画演示][shell_flash]

---
### 分析
* 最差时间复杂度	根据步长序列的不同而不同。已知最好的：O(n log^2 n)
* 最优时间复杂度	O(n)
* 平均时间复杂度	根据步长序列的不同而不同。
* 最差空间复杂度	O(n)
* 步长的选择是希尔排序的重要部分。只要最终步长为1任何步长序列都可以工作。算法最开始以一定的步长进行排序。然后会继续以一定步长进行排序，最终算法以步长为1进行排序。当步长为1时，算法变为插入排序，这就保证了数据一定会被排序。
* 已知的最好步长序列是由Sedgewick提出的(1, 5, 19, 41, 109,...)，该序列的项来自9 \* 4^i - 9 \* 2^i + 1和2^{i+2} \* (2^{i+2} - 3) + 1这两个算式[1]。这项研究也表明“比较在希尔排序中是最主要的操作，而不是交换。”用这样步长序列的希尔排序比插入排序和堆排序都要快，甚至在小数组中比快速排序还快，但是在涉及大量数据时希尔排序还是比快速排序慢。

---
### java语言实现
```
/**
 * 希尔排序
 *
 * @param attr
 * @param <T>
 */
public static <T extends Comparable<T>> void sort(T[] attr) {
    int length = attr.length;
    int gap = 1;
    while (gap < length / 3) {
        gap = gap * 3 + 1; // <O(n^(3/2)) by Knuth,1973>: 1, 4, 13, 40, 121, ...
    }
    while (gap >= 1) {
        for (int i = gap; i < length; i++) {
            for (int j = i; j - gap >= 0; j -= gap) {
                if (attr[j].compareTo(attr[j - gap]) < 0) {
                    T temp = attr[j];
                    attr[j] = attr[j - gap];
                    attr[j - gap] = temp;
                } else {
                    break;
                }
            }
        }
        gap /= 3;
    }
}
```

[shellsort]: http://7u2sbw.com1.z0.glb.clouddn.com/Sorting_shellsort_anim.gif
[wiki]: https://zh.wikipedia.org/wiki/%E5%B8%8C%E5%B0%94%E6%8E%92%E5%BA%8F
[shell_flash]: http://student.zjzk.cn/course_ware/data_structure/web/flashhtml/shell.htm
