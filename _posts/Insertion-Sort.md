title: Insertion Sort
date: 2015-11-13 16:28:25
tags: [algorithm,sort]
categories: algorithm
---

### 原理
插入排序的核心思想是把一个元素插入到已排序元素列表的正确位置，通常做法是：从右向左遍历已排序元素列表，小于则接换位置，大于等于则停止。
具体步骤：
1. 将待排序元素列表的第一个作为已排序元素
2. 取未排序列表的首个元素A（本次为第二个元素）与已排序元素的末尾元素B(本次为第一个元素比较)比较
3. A < B 双方交换位置，A继续与B上一个元素C比较
4. 直到A >= X则说明该元素已经找到相应位置，A元素已经被放置在正确位置
5. ...重复 2 3 4
6. 直到最后一个元素插入相应的位置，排序结束
<!-- more -->
![插入排序][Insertion_sort]

  <center><font size=2>图片来源于[维基百科][wiki]</font></center>

---
### 分析
* 最差时间复杂度	O(n^2)
* 最优时间复杂度	O(n)
* 平均时间复杂度	O(n^2)
* 最差空间复杂度	总共O(n) ，需要辅助空间O(1)


* 插入排序是原地排序
* 插入排序的运行时间与元素初始状态有关，如果元素初始状态与最终排序结果一致，则只需比较n-1次，无需交换；如果初始状态为最终排序结果的倒序，则需比较共有n(n-1)/2次
* 插入排序不适合对于数据量比较大的排序应用。但是，如果需要排序的数据量很小，例如，量级小于千，那么插入排序还是一个不错的选择

---
### 改进
插入排序的核心是找到新元素的位置，主要操作是比较、交换。
#### 基于交换改进：
1. 新元素小当已排序元素时，不交换位置，只将已排序元素后移，直到找到新元素位置后，将新元素插入,如下图

    ![][300px]

2. 不使用数组排序，使用双向列表，这样，新元素小当已排序元素时无需移动，找到新元素位置后修改前后节点的指向即可

#### 基于查找位置的改进：
在已排序的元素中查找元素从头到尾依次比较是低效的、耗时的，一般我们会使用二分查找法。同理，可以使用二分查找法查找新元素的位置。
1. AB相邻的两个元素满足A <= new && new < B，则将新元素插入到A B之间
2. 首个元素F >= new ，则将新元素插入到队首
3. 末尾元素L <= new ,则将新元素插入到队尾

---
### java语言实现
1. 普通插入排序

  ```
  /**
   * 普通插入排序
   *
   * @param array
   * @param <T>
   */
  public static <T extends Comparable<T>> void InsertionSort(T[] array) {
      for (int i = 1; i < array.length; i++) {
          for (int j = i; j > 0; j--) {
              if (array[j].compareTo(array[j - 1]) < 0) {
                  T temp = array[j];
                  array[j] = array[j - 1];
                  array[j - 1] = temp;
              } else {
                  break;
              }
          }
      }
  }
  ```
2. 减少交换次数的排序

  ```
  /**
   * 较少交换的插入排序
   *
   * @param array
   * @param <T>
   */
  public static <T extends Comparable<T>> void lessExchangeSort(T[] array) {
      for (int i = 1; i < array.length; i++) {
          T temp = array[i];
          int index = 0;
          for (int j = i; j > 0; j--) {
              if (array[j - 1].compareTo(temp) <= 0) {
                  index = j;
                  break;
              }
          }
          if (i == index) {
              continue;
          }
          System.arraycopy(array, index, array, index + 1, i - index);
          array[index] = temp;
      }
  }

  ```
3. 利用二分查找法的插入排序

  ```
  /**
   * 利用二分查找法的插入排序
   *
   * @param array
   * @param <T>
   */
  public static <T extends Comparable<T>> void sortWithBinarySearch(T[] array) {
      for (int i = 1; i < array.length; i++) {
          T value = array[i];
          int index = binarySearch(array, i - 1, value);
          if (i == index) {
              continue;
          }
          System.arraycopy(array, index, array, index + 1, i - index);
          array[index] = value;
      }
  }

  /**
   * 二分查找法寻找AB A <= value <= B 中B的索引
   *
   * @param array
   * @param endIndex 已排序的最后一个元素的索引
   * @param value
   * @param <T>
   * @return
   */
  private static <T extends Comparable<T>> int binarySearch(T[] array, int endIndex, T value) {
      if (array[0].compareTo(value) >= 0) {
          return 0;
      }
      if (array[endIndex].compareTo(value) <= 0) {
          return endIndex + 1;
      }
      int low = 0;
      int high = endIndex;
      while (low <= high) {
          int mid = (low + high) >>> 1;
          T v = array[mid];

          if (v.compareTo(value) < 0) {
              low = mid + 1;
          } else if (v.compareTo(value) > 0) {
              high = mid - 1;
          } else {
              return mid + 1;
          }
      }
      return low;
  }
  ```


[Insertion_sort]: http://7u2sbw.com1.z0.glb.clouddn.com/Insertion_sort.gif
[300px]: http://7u2sbw.com1.z0.glb.clouddn.com/Insertion-sort-example-300px.gif
[wiki]: https://zh.wikipedia.org/wiki/%E6%8F%92%E5%85%A5%E6%8E%92%E5%BA%8F
