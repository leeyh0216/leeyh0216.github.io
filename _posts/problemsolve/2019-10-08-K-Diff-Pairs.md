---
layout: post
title:  K diff pairs in array"
date:   2019-10-08 21:00:00 +0900
author: leeyh0216
categories: ps
---

- 출처: [LeetCode - K diff pairs in array](https://leetcode.com/problems/k-diff-pairs-in-an-array/)
- 난이도: 하
- 관련 기술: Two Pointer, Sort
- 문제 요약: 배열 내의 임의의 두 숫자의 조합의 차가 k인 조합의 갯수를 찾는 문제이다.
- 풀이일
  - 2019년 10월 8일
  
#### 풀이 방법

정렬과 투포인터를 이용하여 풀 수 있는 문제이다.

1. 정렬을 수행한다. 정렬이 이루어진 배열은 n-k번째 요소는 반드시 n번째 요소보다 작거나 같은 값을 가지게 된다.
2. 투포인터 알고리즘을 이용하여 차이가 2인 조합들을 찾아 카운팅한다. 이 때 중복인 조합은 1개만 카운팅한다.

#### 코드
{% highlight java %}
class Solution {
    public int findPairs(int[] nums, int k) {
    Arrays.sort(nums);
    if(k <0)
        return 0;
    int l = 0, r = 1, cnt = 0;
    while(r < nums.length){
      if(nums[r] - nums[l] == k) {
        if(l == r){
          r++;  
        }
        else{
          cnt++;
          //중복된 Pair를 생성하지 않기 위해 r을 이동시킨다
          int currentR = nums[r];
          r++;
          while(r < nums.length && nums[r]== currentR)
            r++; 
        }
      }
      else if(nums[r] - nums[l] > k){
        while(l < r && nums[r] - nums[l] > k)
          l++;
      }
      else{
        int currentR = nums[r];
        r++;
        while(r < nums.length && nums[r] == currentR)
          r++;
      }
    }
    return cnt;
  }
}
{% endhighlight%}