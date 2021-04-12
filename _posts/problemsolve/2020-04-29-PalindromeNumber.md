---
layout: post
title:  "LeetCode - Palindrome Number"
date:   2020-04-29 12:57:00 +0900
author: leeyh0216
tags:
- ps
- leetcode
---

- 출처: [LeetCode - Palindrome Number](https://leetcode.com/problems/palindrome-number/)
- 난이도: 중
- 관련 기술: Array
- 풀이일
  - 2020년 04월 29일

# 문제 내용

주어진 정수가 Palindrome인지 아닌지 판별하는 문제이다. 음수는 '-' 부호 때문에 절대 Palindrome이 될 수 없다는 것이 포인트인듯 하다.

# mod 연산을 통한 풀이

주어진 정수를 10으로 나눈 나머지를 리스트에 저장하고, 이 리스트에 저장된 1 ~ (N/2) - 1번째 값과 N ~ (N/2)까지의 값이 같은지 확인하면 된다.

{% highlight java %}
class Solution {
    public boolean isPalindrome(int x) {
        if(x < 0)
            return false;
        
        List<Integer> list = new ArrayList<>();
        
        while(x != 0){
            list.add(x % 10);
            x /= 10;
        }
        
        int listSize = list.size();
        for(int i = 0; i < listSize / 2; i++)
            if(list.get(i) != list.get(listSize - i - 1))
                return false;
        return true;
    }
}
{% endhighlight %}