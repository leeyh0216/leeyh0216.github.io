---
layout: post
title:  "LeetCode - Longest Common Prefix"
date:   2019-10-08 21:00:00 +0900
author: leeyh0216
tags:
- ps
- string
---

- 출처: [LeetCode - Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix/)
- 난이도: 하
- 관련 기술: String
- 문제 요약: 입력 문자열들의 가장 긴 Prefix를 찾는 문제이다.
- 풀이일
  - 2019년 10월 8일
  
#### 풀이 방법

단순한 for문을 통해 풀 수 있는 문제이다.

1. 가장 짧은 문자열의 길이를 찾는다.(모든 문자열의 Prefix는 가장 짧은 문자열의 길이를 넘어설 수 없기 때문)
2. 모든 문자열의 0 ~ 가장 짧은 문자열의 길이 - 1 까지의 값들을 비교하며 다른 문자가 1개라도 나올 때 중지한다.

#### 코드
{% highlight java %}
class Solution {
    public String longestCommonPrefix(String[] strs) {
        int commonLen = Integer.MAX_VALUE;
        for(int i = 0;i<strs.length; i++)
            commonLen = Math.min(commonLen, strs[i].length());
        
        String commonPrefix = "";
        if(strs.length == 0)
            return commonPrefix;
        
        for(int i = 0;i<commonLen; i++){
            char currentChar = strs[0].charAt(i);
            for(int j = 1;j<strs.length;j++){
                if(strs[j].charAt(i) != currentChar)
                    return commonPrefix;
            }
            commonPrefix += currentChar;
        }
        return commonPrefix;
    }
}
{% endhighlight%}