---
layout: post
title: 排序算法
categories: [算法]
description: Redis总结
keywords: 算法
typora-root-url: ../../
---



### 时间复杂度

### 空间复杂度

### 排序

#### 冒泡

每次拿当前位置`j`的值和下一个位置`j+1`的值做比较，如果`arr[j]>arr[j+1]`则交换到后面，然后`j++`指向下一个位置继续比较。

```java
private static void bubbleSort(int[] arr) {
        for (int i = 0; i < arr.length; i++) {
            for (int j = 0; j < arr.length - 1 - i; j++) {
                if (arr[j] > arr[j + 1]) {
                    swap(arr, j, j + 1);
                }
            }
        }
    }

    private static void swap(int[] arr, int i, int j) {
        int tmp = arr[i];
        arr[i] = arr[j];
        arr[j] = tmp;
    }
```

#### 选择

第一次从待排序区域选择一个最小的值，放到数组的第一个位置。 第二次从待选择区域选择一个最小的值放到数组的第二个位置。以此类推，直到待排序区域为空。

```java
	public static void selectSort(int[] arr) {
        if (arr == null || arr.length < 2) {
            return;
        }

        for (int i = 0; i < arr.length; i++) { //总共需要选择的次数
            int minIndex = i;                  //每次默认的最小值的下标 =i
            for (int j = i + 1; j < arr.length; j++) { //从剩余的待排序区域选择一个最小的值
                if (arr[j] < arr[minIndex]) {
                    minIndex = j;
                }
            }
            swap(arr, i, minIndex); //做交换
        }
    }
```



#### 插入排序

类似扑克牌的玩法，依次从右边剩余的区域里面拿一个，插入到左边已排序的合适位置， 直到右边剩余区域为空。

```java
    public static void insertSort(int[] arr) {
        if (arr == null || arr.length < 2) {
            return;
        }

        for (int i = 1; i < arr.length; i++) {
            if (arr[i] < arr[i - 1]) { //i-1是左边已排好序的
                for (int j = i; j > 0 && arr[j] < arr[j - 1]; j--) { //只要小于前一个值，就交换
                    swap(arr, j, j - 1);
                }
            }
        }
    }
```



### 递归

#### 归并排序

假如一个数组，从中间位置起，左边部分已排好序，右边部分也已排好序。那么只需要再次处理左右两边的部分则整体就有序了。递归进行左右两边的内容，化解为子问题。

```java
    public static void mergeSort(int[] arr) {
        if (arr == null || arr.length < 2) {
            return;
        }
        mergeSort(arr, 0, arr.length - 1);
    }

    public static void mergeSort(int[] arr, int l, int r) {
        if (l == r) {
            return;
        }

        //将数组分为左边和右边
        int mid = l + (r - l) / 2;
        mergeSort(arr, l, mid);
        mergeSort(arr, mid + 1, r);
        
        //左右两边都排好序后，进行合并
        merge(arr, l, mid, r);
    }

    private static void merge(int[] arr, int l, int mid, int r) {
        //申请一个临时数组，用来放左右两边再次排序后的结果
        int[] tmp = new int[r - l + 1];
        int i = l;
        int j = mid + 1;
        int idx = 0;
        while (i <= mid && j <= r) {
            if (arr[i] <= arr[j]) {
                tmp[idx++] = arr[i];
                i++;
            } else {
                tmp[idx++] = arr[j];
                j++;
            }
        }
      	
      	//下面的2个while同时只会执行其中的1个，因为上面while结束肯定有一个到尾部了
      	//左边是否有剩余
        while (i <= mid) {
            tmp[idx++] = arr[i];
            i++;
        }

      	//右边是否有剩余
        while (j <= r) {
            tmp[idx++] = arr[j];
            j++;
        }

        //拷贝回源数组
        for (int k = 0; k < tmp.length; k++) {
            arr[l++] = tmp[k];
        }
    }
```



##### 求小和

##### 求逆序列

### 快速排序

#### 数组左右切分

给定一个数组arr, 和一个数num, 把小于等于num的数放在左边，大于num的数放到右边。

#### 荷兰国旗问题

给定一个数组arr, 和一个数num, 把小于num的数放在左边，等于num的放在中间，大于num的数放到右边。

要求时间复杂度O(N)， 额外空间复杂度O(1)

#### 普通快排

#### 随机快排

## 桶排序

### 计数排序

### 基数排序

### 面试题

给定一个数组，求如果排序后，相邻两数的最大差值。要求时间复杂度O(N)，且不能用非基于比较的排序？

### 堆排序

#### 堆的结构

#### 大根堆

#### 小根堆

#### 堆排序

## 排序稳定性总结

## Hash函数

## Hash表

## 一致性Hash

## 布隆过滤器

## 并查集

