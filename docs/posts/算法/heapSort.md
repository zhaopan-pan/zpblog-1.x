---
sidebar: false
date: "2022-1-5"
tag: 
 - 算法
title: 二叉堆排序
category: 
 - 算法
---

## 实现

有个上一篇的二叉堆前置知识，趁热打铁顺便温习一下堆排序

```js
// 交换两个节点
function swap(A: number[], i: number, j: number) {
    let temp = A[i]
    A[i] = A[j]
    A[j] = temp
}

/**
 * 比较移动操作
 * @param arr 
 * @param startIndex 开始下标
 * @param endIndex 结束下标
 * @returns 
 */
function shiftDown(arr: number[], startIndex: number, endIndex: number, compareTo: (l: number, r: number) => Boolean) {
    // 当前父节点值
    const temp = arr[startIndex]
    // i = 2 * startIndex + 1 左子节点，
    // i < endIndex 目的是对结点 endIndex 以上的结点全部做顺序调整
    // i = 2 * i + 1 左子节点的左子节点，代表当前节点的下一级，靠它循环
    for (let i = 2 * startIndex + 1; i < endIndex; i = 2 * i + 1) {
        // 如果右子节存在，且右子节点大于左子节点，就把i更新成右子节点的下标
        if (i + 1 < endIndex && compareTo(arr[i], arr[i + 1])) {
            i++
        }
        // 根据比较规则看是否需要父子替换，否则跳出
        if (compareTo(temp, arr[i])) {
            swap(arr, startIndex, i)
            // 更新父节点索引
            startIndex = i
        } else {
            break
        }
    }

}
/**
 * 创建大/小顶堆:一般升序采用大顶堆，降序采用小顶堆
 * @param arr 
 * @param compareTo 
 */
function createHeap(arr: number[], compareTo: (l: number, r: number) => Boolean) {
    // 最后一个非叶子节点下标
    const startIndex = Math.floor(arr.length / 2 - 1)
    for (let i = startIndex; i >= 0; i--) {
        //从第一个非叶子结点从下至上，从右至左调整结构
        shiftDown(arr, i, arr.length, compareTo)
    }
}

/**
 * 堆排序
 * @param arr 
 * @param compareTo 
 */
function startSort(arr: number[], compareTo: (l: number, r: number) => Boolean) {
    // arr.length - 1： 最后一个节点交换位置后是最大或最小的，shiftDown中遍历的时候会排除最后一个节点
    for (let j = arr.length - 1; j > 0; j--) {
        // 堆顶(最大值或最小值)和最后一个交换位置
        swap(arr, 0, j)
        // 从根节点开始对堆进行调整
        shiftDown(arr, 0, j, compareTo)
    }
}
/**
 * 堆排序方法
 * @param a 需要排序的数据
 * @param compare 比较方法，决定升降顺序
 */
function heapSortFn(a: number[], compare ? : (l: number, r: number) => Boolean) {
    // 默认降序，可以自定义成升序
    const compareTo = compare ? compare : (l: number, r: number) => l > r
    const arr = a
    // 创建大/小顶堆 
    createHeap(arr, compareTo)
    //推排序
    startSort(arr, compareTo)
}

export default function heapSort() {
    let arr1 = [5, 8, 0, 10, 4, 6, 1]
    heapSortFn(arr1)
    console.log("降序:", arr1) //[10, 8, 6, 5, 4, 1, 0]
    heapSortFn(arr1, (i: number, j: number) => i < j)
    console.log("升序:", arr1) // [0, 1, 4, 5, 6, 8, 10]
}
```

## 复杂度分析

主要围绕 `shiftDown` 函数分析， `shiftDown` 本身是在重建堆, 执行的次数是由完全二叉树的深度决定的, 由完全二叉树的性质可知 `n` 个节点的话深度为 `[log2n]+1` ，从 `i = 2 * i + 1` 也可以看出，所以 `shiftDown` 的复杂度为 `O(logN)` ; 它的外层循环次数都是n次，复杂度是 `O(n)` ; 所以最终复杂度是 `O(nlogN)`

## 总结

堆排是一种选择排序，以此处升序为例：  
1、将初始二叉树转化为大顶堆: 实际上是从第一个非叶子结点开始，从下至上，从右至左，对每一个非叶子结点做shiftDown操作  
2、【重点】: 此时根结点为最大值，将其与最后一个结点交换  
3、除开最后一个结点，将其余节点组成的新堆转化为大顶堆，此时根结点为次最大值，将其与最后一个结点交换  
4、重复步骤2、3，直到堆中元素个数为1（或其对应数组的长度为1），排序完成
