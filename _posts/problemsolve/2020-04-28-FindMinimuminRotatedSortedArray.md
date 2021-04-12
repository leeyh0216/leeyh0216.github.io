---
layout: post
title:  "LeetCode - Find Minimum in Rotated Sorted Array"
date:   2020-04-28 21:45:00 +0900
author: leeyh0216
tags:
- ps
- leetcode
---

- 출처: [LeetCode - Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/)
- 난이도: 중
- 관련 기술: Array
- 풀이일
  - 2020년 04월 28일

# 문제 내용

오름차순으로 정렬된 배열을 특정 기준점으로 뒤집어놓은 배열이 존재한다(예를 들어 \[1, 2, 3, 4\]와 같은 배열(Zero-Based)이 있을 때 Index 2를 기준으로 뒤집으면 \[4, 1, 2, 3\] 이 된다). 이 배열에서 가장 작은 값을 찾는 문제이다.

# Sliding Window를 이용한 풀이

이진 탐색을 이용하여 풀이하려 했으나, Sliding Window를 이용해서 풀어도 O(n) 밖에 안걸리기 때문에 해당 방식으로 풀이하였다.

Window의 크기를 3으로 잡으면 가장 작은 값이 등장하는 패턴은 다음과 같다.

* \[3, 1, 2\]
* \[2, 3, 1\]

감소 -> 증가 패턴이거나 증가 -> 감소 패턴을 보이는 Window가 있을 경우 상황에 따라 중간 값이나 맨 마지막 값을 반환하면 된다는 접근으로 풀이하였다.

{% highlight java %}
class Solution {
    public int findMin(int[] nums) {
        if(nums.length == 1)
            return nums[0];
        else if(nums.length == 2)
            return Math.min(nums[0], nums[1]);
        else if(nums[0] < nums[nums.length - 1])
            return nums[0];
        
        int l = 0, m = 1, r = 2;
        while(r < nums.length){
            if(nums[l] < nums[m] && nums[m] > nums[r])
                return nums[r];
            else if(nums[l] > nums[m] && nums[m] < nums[r])
                return nums[m];
            else{
                l++;
                m++;
                r++;
            }
        }
        return 0;       
    }
}
{% endhighlight %}