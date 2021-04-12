---
layout: post
title:  "LeetCode - Maximum Points You Can Obtain from Cards"
date:   2020-04-26 22:30:00 +0900
author: leeyh0216
tags:
- ps
- leetcode
---

- 출처: [LeetCode - Maximum Points You Can Obtain from Cards](https://leetcode.com/problems/maximum-points-you-can-obtain-from-cards/)
- 난이도: 중
- 관련 기술: Array
- 풀이일
  - 2020년 04월 26일

# 문제 내용

임의의 배열과 정수 K가 주어진다. 배열의 양 끝에서 하나씩 K개의 숫자를 뽑아 만들 수 있는 최대 값을 반환하는 문제이다.

# 1차 풀이

Recursion을 통해 접근한 뒤, Memoization을 통해 풀이하려 했다.

결과는 실패였다. Memoization 시 2차원 배열을 만들었는데 입력 배열의 크기가 100,000까지 늘어날 수 있었기 때문에, 2차원 `int`형 배열을 만드는 순간 Memory Limit Exceed 가 발생하였다.

{% highlight java %}
class Solution {
    
    int[][] memo;
    
    public int maxScore(int[] cardPoints, int left, int right, int k){
        if(memo[left][right] != 0)
            return memo[left][right];
        if(k == 1)
            return memo[left][right] = Math.max(cardPoints[left], cardPoints[right]);
        return memo[left][right] = Math.max(cardPoints[left] + maxScore(cardPoints, left + 1, right, k - 1), cardPoints[right] + maxScore(cardPoints, left, right - 1, k - 1));
    }
    
    public int maxScore(int[] cardPoints, int k) {
        memo = new int[cardPoints.length][cardPoints.length];
        
        return maxScore(cardPoints, 0, cardPoints.length - 1, k);
    }
}
{% endhighlight %}

# 2차 풀이

배열의 좌측 혹은 우측에서 하나씩 뽑아가는 방식이기 때문에, 아래와 같은 방식으로 뽑는다면 O(K) 만큼의 시간복잡도로 풀이할 수 있었다.

1. 좌측에서 0개, 우측에서 K개
2. 좌측에서 1개, 우측에서 K-1개
3. 좌측에서 2개, 우측에서 K-2개
...
K. 좌측에서 K개, 우측에서 0개

길이 K의 배열 2개를 만들어 좌측으로부터의 부분합과 우측으로부터의 부분합을 더한 뒤, K번 순회하여 최대 값을 찾아내면 된다.

{% highlight java %}
class Solution {
    
    public int maxScore(int[] cardPoints, int k) {
        int[] fromStart = new int[cardPoints.length];
        int[] fromEnd = new int[cardPoints.length];
        
        int start = 0, end = 0, sum = 0;
        for(int i = 0; i < cardPoints.length; i++){
            start += cardPoints[i];
            sum += cardPoints[i];
            fromStart[i] = start;
        }
        
        for(int i = cardPoints.length - 1; i >= 0; i--){
            end += cardPoints[i];
            fromEnd[cardPoints.length - i - 1] = end;
        }
        
        int max = 0;
        for(int i = 0; i <= k; i++){
            int s = i, e = k - s;
            int ssum = s == 0 ? 0 : fromStart[s - 1];
            int esum = e == 0 ? 0 : fromEnd[e - 1];
            max = Math.max(max, ssum + esum);
        }
        return max;
    }
}
{% endhighlight %}