---
layout: post
title:  "LeetCode - Super Reduced String"
date:   2019-10-07 21:40:00 +0900
author: leeyh0216
tags:
- ps
- string
- recursion
---

- 출처: [HackerRank - Super Reduced String](https://www.hackerrank.com/challenges/reduced-string/problem)
- 난이도: 하
- 관련 기술: String, Recursion
- 문제 요약: 입력으로 주어진 문자열 내의 연속으로 2번 등장하는 문자를 제거하는 문제이다. 예를들어 aaabb가 나온다면 aaabb -> abb -> a가 된다.
- 풀이일
  - 2019년 10월 7일
  
#### 풀이 방법

재귀 함수를 만들어 풀이하면 된다. 각 Step에서는 문자열을 순회하며 연속 2회 등장하는 문자를 확인한다.

- 2회 연속 등장하는 문자가 발생한 경우: 해당 문자 2개를 빈 문자열로 치환한 뒤 재귀 함수 호출
- 2회 연속 등장하는 문자가 발생하지 않는 경우: 입력 문자열을 그대로 반환(빈 문자열인 경우 "Empty String" 반환)

#### 코드
{% highlight java %}
public class Solution {
  String superReducedString(String s) {
    if(s.isEmpty())
      return "Empty String";
    for(int i = 0;i<s.length() - 1; i++){
      char front = s.charAt(i), end = s.charAt(i+1);
      if(front == end) {
        String newStr = s.replace("" + front + front, "");
        return superReducedString(newStr);
      }
    }
    return s;
  }
}
{% endhighlight%}