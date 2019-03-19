title: Heap Sort
date: 2015-11-16 10:22:34
tags: 
  - algorithm
  - sort
categories: algorithm
---


堆排序（Heapsort）是指利用二叉堆这种数据结构所设计的一种排序算法。
### 二叉堆
二叉堆是一种特殊的堆，其有以下特性：
1. 二叉堆是完全二叉树或者是近似完全二叉树
2. 每个节点的左子树和右子树都是一个二叉堆
3. 父节点的键值总是与子节点的键值保持固定的序关系（parent>=son || parent<=son）

当父节点的键值总是大于或等于任何一个子节点的键值时为 **最大堆**。 当父节点的键值总是小于或等于任何一个子节点的键值时为 **最小堆**。
<!-- more -->
二叉堆一般用数组来表示。如果根节点在数组中的位置是1，则：
* leftSon(n) = 2n
* rightSon(n) = 2n+1
* parent(n) = n/2

如果存储数组的下标基于0，则：
* leftSon(n) = 2n+1
* rightSon(n) = 2n+2
* parent(n) = (n-1)/2

以[49，38，65，97，76，13，27，55，04]为例，构建二叉堆初始状态：
![][heap-sort1]

<center><font size=2>图A</font></center>

图A展示的并不是真正的二叉堆，因为它不满足二叉堆的特性： `父节点的键值总是与子节点的键值保持固定的序关系`。

---
### 二叉堆的操作
**以下说明基于最大堆**
1. #### 从下至上的重新建堆（sift-up）
  在一个 **最大堆** 中，如果一个节点的值大于其父节点的值(son > parent),不满足`固定的序关系（parent >= son）`，
  那么该节点就需要上移，直到满足该节点大于其两个子节点，而小于其父节点为止，从而达到使整个堆实现二叉堆的要求。
  简单的说，比较子节点与父节点，不满足堆性质则交换。从而使得满足二叉堆的性质。

  在二叉堆中插入节点时需要用到此操作，如下：
  ![][sift-up1]
  ![][sift-up2]
  ![][sift-up3]

  代码实现
  ```java
  /**
   * 二叉堆从下至上的重新建堆（sift-up）
   *
   * @param array 堆数组，index前的元素已经构成最大堆
   * @param index 新插入节点的索引
   * @param <T>
   */
  private static <T extends Comparable<? super T>> void siftUp(T[] array, int index) {
      int son = index;
      T key = array[son];
      while (son > 0) {
          int parent = getParent(son);
          if (key.compareTo(array[parent]) <= 0) {
              break;
          }
          array[son] = array[parent];
          son = parent;
      }
      array[son] = key;
  }
  ```
2. #### 由上至下的重新建堆（sift-down）
在一个 **最大堆** 中，如果一个节点的值小于其子节点的值（parent < son），不满足`固定的序关系(parent >= son)`，就违反了二叉堆的定义，那么该节点就需要下移，直到满足该节点大于其两个子节点为止，从而达到使整个堆实现二叉堆的要求。需要注意的是，`要与两个子节点中较大的节点进行比较`。
简单的说，比较父节点与子节点，不满足堆性质则交换。从而使得满足二叉堆的性质。
在二叉堆中删除根元素时，会用到此操作。`a = 100`

  删除根元素的步骤：
  1. 删除根元素
  2. 将堆中最后一个元素放到根元素位置
  3. 对新的根元素进行sift-down操作

![二叉堆sift-down][sfit-down]

代码实现：

```java
/**
 * 二叉堆从上至下的重新建堆（sift-down）
 *
 * @param array
 * @param index    要执行向下操作的父节点id
 * @param heapSize 堆的节点个数
 * @param <T>
 */
private static <T extends Comparable<? super T>> void siftDown(T[] array, int index, int heapSize) {
    int parent = index;
    T key = array[parent];
    int left;
    while ((left = getLeftSon(parent)) < heapSize) {
        int right = getRightSon(parent);
        int largerSon;
        if (right < heapSize && array[left].compareTo(array[right]) < 0) {
            largerSon = right;
        } else {
            largerSon = left;
        }
        if (array[largerSon].compareTo(key) > 0) {
            array[parent] = array[largerSon];
            parent = largerSon;
        } else {
            break;
        }
    }
    array[parent] = key;
}
```



### 堆排序
利用二叉堆的性质`根节点必定是最大值(最大堆)或者最小值(最小堆)`可以很容易实现排序。
1. 将给定元素集合构成一个二叉堆
2. 将根节点与最后一个节点替换位置，二叉堆的size - 1
3. 对新的根节点进行sift-down操作
4. 重复2 3 操作，直到二叉堆内仅剩一个元素

至此，排序完成

![][heapsort_anim]
<center><font size=2>图片来源于[维基百科][wiki]</font></center>




### java实现
```java

    private static int getParent(int son) {
        return (son - 1) >>> 1;
    }

    private static int getLeftSon(int parent) {
        return (parent << 1) + 1;
    }

    private static int getRightSon(int parent) {
        return (parent << 1) + 2;
    }

    private static void swap(Object[] array, int index1, int index2) {
        Object temp = array[index1];
        array[index1] = array[index2];
        array[index2] = temp;
    }

    /**
     * 二叉堆从下至上的重新建堆（sift-up）
     *
     * @param array 堆数组，index前的元素已经构成最大堆
     * @param index 新插入节点的索引
     * @param <T>
     */
    private static <T extends Comparable<? super T>> void siftUp(T[] array, int index) {
        int son = index;
        T key = array[son];
        while (son > 0) {
            int parent = getParent(son);
            if (key.compareTo(array[parent]) <= 0) {
                break;
            }
            array[son] = array[parent];
            son = parent;
        }
        array[son] = key;
    }


    /**
     * 二叉堆从上至下的重新建堆（sift-down）
     *
     * @param array
     * @param index    要执行向下操作的父节点id
     * @param heapSize 堆的节点个数
     * @param <T>
     */
    private static <T extends Comparable<T>> void siftDown(T[] array, int index, int heapSize) {
        int parent = index;
        T key = array[parent];
        int left;
        while ((left = getLeftSon(parent)) < heapSize) {
            int right = getRightSon(parent);
            int largerSon;
            if (right < heapSize && array[left].compareTo(array[right]) < 0) {
                largerSon = right;
            } else {
                largerSon = left;
            }
            if (array[largerSon].compareTo(key) > 0) {
                array[parent] = array[largerSon];
                parent = largerSon;
            } else {
                break;
            }
        }
        array[parent] = key;
    }

    /**
     * 构建二叉堆
     *
     * @param array
     * @param <T>
     */
    private static <T extends Comparable<? super T>> void heapify(T[] array) {
        int heapSize = array.length;
        for (int i = getParent(heapSize - 1); i >= 0; i--) {
            siftDown(array, i, heapSize);
        }
    }

    /**
     * 对array进行堆排序
     * @param array
     * @param <T>
     */
    public static <T extends Comparable<? super T>> void sort(T[] array) {
        int heapSize = array.length;
        heapify(array);
        while (heapSize > 1) {
            swap(array, 0, heapSize - 1);
            heapSize--;
            siftDown(array, 0, heapSize);
        }
    }
```



### 分析
* 最差时间复杂度	O(n log n)
* 最优时间复杂度	O(n log n)
* 平均时间复杂度	θ(n log n)
* 最差空间复杂度	O(n) total, O(1) auxiliary
* 堆排序属于就地排序
* 堆排序可以在集合中查找最大或者最小的N个值，而不需要对全部数据进行排序


### 优先级队列
利用二叉堆的性质`根节点必定是最大值(最大堆)或者最小值(最小堆)`同样可以很容易实现优先级队列。
* 队列添加元素：二叉堆对新元素进行sift-up
* 队列获取(poll)元素：二叉堆删除根元素 sift-down

JDK中的PriorityQueue、PriorityBlockingQueue都是基于二叉堆实现的。

[wiki]: https://zh.wikipedia.org/wiki/%E5%A0%86%E6%8E%92%E5%BA%8F
[heapsort_anim]: http://7u2sbw.com1.z0.glb.clouddn.com/Sorting_heapsort_anim.gif
[heap-sort1]: http://7u2sbw.com1.z0.glb.clouddn.com/heap-sort-init.png
[sift-up1]: http://7u2sbw.com1.z0.glb.clouddn.com/heap-sort-shit-up1.png
[sift-up2]: http://7u2sbw.com1.z0.glb.clouddn.com/heap-sort-shit-up2.png
[sift-up3]: http://7u2sbw.com1.z0.glb.clouddn.com/heap-sort-shit-up3.png
[sfit-down]: http://7u2sbw.com1.z0.glb.clouddn.com/heap-sort-sfit-down2.png
