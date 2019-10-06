---
layout: post
title:  "Median of Two Sorted Arrays"
date:   2019-10-07 05:00:00 +0900
author: leeyh0216
categories: ps
---

- 출처: [LeetCode - Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays/)
- 난이도: 상
- 관련 기술: Array, Binary Search, Divide and Conquer
- 문제 요약: 미리 정렬된 2개의 배열을 합친 배열의 중간 값(median)을 찾아내면 되는 문제이다.

#### 풀이 방법

입력으로 들어온 2개의 배열(nums1, nums2)이 이미 정렬되어 있는 상태이므로, 2개의 포인터(l = nums1의 요소를 가리킴, r = nums2의 요소를 가리킴)을 두고 작은 값부터 최종 배열에 추가하여 배열 병합을 수행하고 중간 값을 찾으면 된다.

#### 코드

{% highlight java %}
class Solution {
    public double findMedianSortedArrays(int[] nums1, int[] nums2) {
    int nums1Length = nums1.length, nums2Length = nums2.length, numsLength = nums1Length + nums2Length;
    int[] nums = new int[numsLength];
    
    int l = 0, r = 0, cursor = 0;
    
    while(l < nums1Length && r < nums2Length){
      if(nums1[l] < nums2[r]){
        nums[cursor] = nums1[l];
        l++;
      }
      else{
        nums[cursor] = nums2[r];
        r++;
      }
      cursor++;
    }
    
    while(l < nums1Length){
      nums[cursor] = nums1[l];
      cursor++;
      l++;
    }
    while(r < nums2Length){
      nums[cursor] = nums2[r];
      cursor++;
      r++;
    }
    
    if(numsLength % 2 != 0)
      return nums[numsLength / 2];
    else
      return ((double)nums[numsLength / 2 - 1] + (double)nums[numsLength / 2 ]) / 2.0d;
  }
}
{% endhighlight %}