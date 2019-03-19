title: Selection Sort
date: 2015-11-13 08:40:33
tags: [algorithm,sort]
categories: algorithm
---

### 原理
选择排序（Selection sort）是一种简单直观的排序算法。它的工作原理如下：
1. 遍历所有元素，选出最大或者最小的元素，与第一个元素（最后一个亦可）交换位置
2. 遍历剩余的元素，继续选出最大或者最小元素，与第二个元素交换位置
3. ...
4. 直到最后一个元素，至此，排序完毕
<!-- more -->

  ![选择排序][Selection_sort_animation]

  <center><font size=2>图片来源于[维基百科][wiki]</font></center>

---
### 示例
以[2,3,9,5,6,4,1]为例，由小到大排序。i代表当前要选出的第X个元素，m代表当次最小元素

    i           m
    2,3,9,5,6,4,1
2,1交换，变为

    1,3,9,5,6,4,2
i前移，同时继续选出剩余元素中的最小元素

      i         m
    1,3,9,5,6,4,2
3,2交换

    1,2,9,5,6,4,3
如此重复，知道最后一个元素

        i       m
    1,2,9,5,6,4,3

          i   m
    1,2,3,5,6,4,9  

            i m
    1,2,3,4,6,5,9  

              i
    1,2,3,4,5,6,9
              m

                i
    1,2,3,4,5,6,9
至此，排序结束

---
### 分析
* 分类	排序算法
* 数据结构	数组
* 最差时间复杂度	О(n²)
* 最优时间复杂度	О(n²)
* 平均时间复杂度	О(n²)
* 最差空间复杂度	О(n) total, O(1) auxiliary

* 选择排序是原地排序，仅需要一个额外的元素存放当次的最大或最小值，不需要其他空间。
* 选择排序需要的交换次数较少，元素已经是有序的情况下不需要交换操作。但比较次数固定，与元素的初始状态无关，总的比较次数N=(n-1)+(n-2)+...+1=n*(n-1)/2。
* 选择排序的核心思想与冒泡排序一样，都是选出未排序元素中的最大值或最小值，比较次数与冒泡排序一样，但交换次数少于冒泡排序。

---
### java语言实现
    public static <T extends Comparable<T>> void selectionSort(T[] array) {
        int end = array.length - 1;
        for (int i = 0; i < end; i++) {
            int maxIndex = i;
            for (int j = i + 1; j < array.length; j++) {
                if (array[j].compareTo(array[maxIndex]) < 0) {
                    maxIndex = j;
                }
            }
            if (maxIndex != i) {
                T temp = array[i];
                array[i] = array[maxIndex];
                array[maxIndex] = temp;
            }
        }
    }
[Selection_sort_animation]: http://7u2sbw.com1.z0.glb.clouddn.com/Selection_sort_animation.gif
[wiki]: https://zh.wikipedia.org/wiki/%E9%80%89%E6%8B%A9%E6%8E%92%E5%BA%8F
