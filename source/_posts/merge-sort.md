---
layout: post
title: "归并排序Java实现"
date: 2016-03-10 15:54
tags: 算法
category: 随笔
description: 归并排序的java实现。

---
具体算法是参考算法导论，这里只是我自己的实现的记录。
```java
package sort;

import org.junit.Test;

import java.util.Arrays;

/**
 * 归并排序
 *
 * @auth gongxufan
 * @Date 2017/8/2
 **/
public class MergeSort {
    /**
     * p<=q<r,A[p..q]和A[q+1..r]都已排好序
     *
     * @param A 输入
     * @param p 待合并数组的下界位置
     * @param q 分割位置
     * @param r 待合并数组的上界位置
     */
    public void mergeWithGuard(int[] A, int p, int q, int r) {
        //A[p..q]的数组长度
        int n1 = q - p + 1;
        //A[q+1..r]的数组长度
        int n2 = r - q;
        //构建两个数组
        int[] L = new int[n1 + 1];
        int[] R = new int[n2 + 1];
        int i = 0, j = 0;
        //初始化
        for (i = 0; i < n1; i++)
            L[i] = A[p + i];
        for (j = 0; j < n2; j++)
            R[j] = A[q + j + 1];
        //设置哨兵,防止越界
        L[n1] = Integer.MAX_VALUE;
        R[n2] = Integer.MAX_VALUE;
        //索引清零
        i = j = 0;
        //最多执行执行r-p次,就可以合并完成
        for (int k = p; k <= r; k++) {
            //如果L或者R复制完毕，则下一个元素就是哨兵位置,可以避免数组越界访问
            if (L[i] <= R[j]) {
                A[k] = L[i++];
            } else {
                A[k] = R[j++];
            }
        }
        System.out.println("L:" + Arrays.toString(L) + ",R:" + Arrays.toString(R));
        System.out.println("Merge Result:" + Arrays.toString(A));
    }

    /**
     * 不使用哨兵
     * p<=q<r,A[p..q]和A[q+1..r]都已排好序
     *
     * @param A 输入
     * @param p 待合并数组的下界位置
     * @param q 分割位置
     * @param r 待合并数组的上界位置
     */
    public void mergeWithoutGuard(int[] A, int p, int q, int r) {
        //A[p..q]的数组长度
        int n1 = q - p + 1;
        //A[q+1..r]的数组长度
        int n2 = r - q;
        //构建两个数组
        int[] L = new int[n1];
        int[] R = new int[n2];
        int i = 0, j = 0;
        //初始化
        for (i = 0; i < n1; i++)
            L[i] = A[p + i];
        for (j = 0; j < n2; j++)
            R[j] = A[q + j + 1];
        int k = p;
        i = j = 0;
        //分别复制L R,其中一个复制完成立即结束
        while (i < n1 && j < n2){
            if (L[i] <= R[j]) {
                A[k++] = L[i++];
            } else {
                A[k++] = R[j++];
            }
        }
        //L或者R复制完毕，剩下的元素再复制。已经复制完成的会跳过
        while (i < n1)
            A[k++] = L[i++];
        while (j < n2)
            A[k++] = R[j++];
        System.out.println("L:" + Arrays.toString(L) + ",R:" + Arrays.toString(R));
        System.out.println("Merge Result:" + Arrays.toString(A));
    }

    public void mergeSort(int[] A, int p, int r) {
        if (p < r) {
            System.out.println("当前排序的数组:" + Arrays.toString(Arrays.copyOfRange(A, p, r)));
            int q = (p + r) / 2;
            mergeSort(A, p, q);
            mergeSort(A, q + 1, r);
            mergeWithoutGuard(A, p, q, r);
        }
    }

    @Test
    public void testMerge() {
        int[] A = new int[]{3};
        int p = 0, r = 0, q = (p + r) / 2;
        mergeWithGuard(A, p, q, r);
    }

    @Test
    public void testMergeSort() {
        int[] A = new int[]{10, 4, 1, -1, 8, 3, 5, 0, -5};
        mergeSort(A, 0, A.length - 1);
        System.out.println("Sort Result:" + Arrays.toString(A));
    }
}
```
合并的实现采用了两种方式，一种是使用哨兵，一种是合并分两步。
