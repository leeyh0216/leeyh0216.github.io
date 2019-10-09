---
layout: post
title:  "LeetCode - Generate Parentheses"
date:   2019-10-08 21:00:00 +0900
author: leeyh0216
tags:
- ps
- recursion
---

- 출처: [LeetCode - Generate Parentheses](https://leetcode.com/problems/generate-parentheses/)
- 난이도: 하
- 관련 기술: Recursion
- 문제 요약: n개의 (, )를 이용하여 완전히 닫힌 괄호 조합을 만들어내는 문제
- 풀이일
  - 2019년 10월 8일
  
#### 풀이 방법

재귀 함수를 이용하여 풀이할 수 있는 문제이다.

닫힌 괄호를 만들어내야 하는데, '(', ')'를 기존 문자열에 붙일 때 기존 문자열에서의 '('의 갯수는 ')'보다 같거나 많아야 한다. 또한 기존 문자열의 '('와 ')'의 갯수가 같은 경우 반드시 '('를 붙여주어야 한다.

예를 들어 '(()'가 기존 문자열인 경우에는 '(' 가 ')'보다 많으므로 뒤에 ')'를 붙여 닫힌 괄호를 만들어낼 수 있다.

반면 '())'가 기존 문자열인 경우에는 ')' 가 '('보다 많으므로 뒤에 어떠한 문자를 붙이더라도 닫힌 괄호를 만들어낼 수 없다.

'(' 문자를 -1, ')' 문자를 1이라고 생각해보면 기존 문자열의 합이 0보다 작거나 같은 경우에는 계속해서 닫힌 괄호를 만들어낼 수 있으며 0보다 큰 경우에는 만들 수 없는 경우이다. 

#### 코드
{% highlight java %}
class Solution {
    
   List<String> result = new ArrayList<>();

  private void generateParenthesis(int l, int r, int sum, String current){
    //쓸 수 있는 괄호의 갯수가 없는 경우
    if(l <0 || r < 0)
      return;

    //기존 문자열의 '('와 ')'의 갯수가 동일한 경우
    if(sum == 0) {
      if(l == 0 && r == 0)
        result.add(current);
      else
        generateParenthesis(l - 1, r, sum - 1, current + "(");
    }
    //기존 문자열의 '(' 갯수가 ')'보다 많은 경우
    else if(sum < 0){
      generateParenthesis(l-1, r, sum-1, current + "(");
      generateParenthesis(l, r -1 , sum + 1, current + ")");
    }
    //기존 문자열의 ')' 갯수가 '(' 보다 많은 경우
    else
      return;
  }

  public List<String> generateParenthesis(int n) {
    generateParenthesis(n,n,0,"");
    return result;
  }
}
{% endhighlight%}