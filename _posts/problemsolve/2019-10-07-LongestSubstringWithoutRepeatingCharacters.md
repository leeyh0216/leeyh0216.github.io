---
layout: post
title:  "Longest Substring Without Repeating Characters"
date:   2019-10-07 04:00:00 +0900
author: leeyh0216
categories: ps
---

- 출처: [LeetCode - Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/)
- 난이도: 중
- 관련 기술: Sliding Window
- 문제 요약: 입력으로 주어진 문자열(s)에서 **반복된 문자를 가지지 않는 최대 길이의 부분 문자열**을 찾는 문제이다.
- 풀이일
  - 2019년 10월 7일
  
#### 풀이 방법

s(l, r)이 반복된 문자를 가지지 않고, s(r+1)이 s(l, r)에 포함되지 않는 경우 s(l, r+1) 또한 반복된 문자를 가지지 않는 최대 길이 문자열로 볼 수 있다.

다만 s(r+1)이 s(l, r)에 포함되는 경우엔 l ~ r 구간에서 s(r+1)과 동일한 문자인 인덱스 x를 찾아내어 l을 x+1 로 변경해주어야 한다. l이 x+1로 변경된 후에는 s(l, r+1) 내에 반복된 문자가 존재하지 않으므로 다시 윈도우 크기를 키우며 로직을 수행하면 된다.

> * s(l, r): 문자열 s의 부분 문자열(인덱스: l ~ r)
> * s(x): 문자열 s의 x번째 문자

"abcabcbb"를 예시로 들어보자면 다음과 같다.

* s(0, 2)까지는 반복된 문자열이 없으므로 최대 길이는 3이 된다.
* s(3)이 s(0, 2)에 포함되므로 s(l)이 s(3) 값과 다를 때까지 증가시킨다.
* s(4)이 s(1, 3)에 포함되므로 s(l)이 s(4) 값과 다를 때까지 증가시킨다.
* ...

#### 코드

{% highlight java %}
public int lengthOfLongestSubstring(String s) {
    int l = 0, r = 0, len = s.length(), currentLen = 0, maxLen = 0;
    Set<Character> charSet = new HashSet<>();

    while(r < len){
        if(!charSet.contains(s.charAt(r))){
            currentLen++;
            charSet.add(s.charAt(r));
        }
        else{
            while(l < r && s.charAt(l) != s.charAt(r)){
                charSet.remove(s.charAt(l));
                l++;
            }
            l++;
            currentLen = r - l + 1;
        }
        maxLen = Math.max(currentLen, maxLen);
        r++;
    }
    return maxLen;
}
{% endhighlight%}