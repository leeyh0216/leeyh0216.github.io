---
layout: post
title:  "LeetCode Biweekly Contest 25"
date:   2020-05-05 19:50:00 +0900
author: leeyh0216
tags:
- ps
- leetcode
---

# 참여 결과 및 후기

| Rank        | Score | Finish Time | Q1      | Q2      | Q3      | Q4 |
|-------------|-------|-------------|---------|---------|---------|----|
| 1983 / 5867 | 12    | 1:14:28	    | 0:03:23 | 0:35:04 | 0:54:28 | -  |

이번 Contest도 Weekly Contest 187과 같이 Mock Contest로 진행하였다.

Hard 문제인 Q4는 오늘도 풀지 못했고, Q2에서 문제 조건 때문에 4번이나 제출해서 완전 망했다. 아마 저 Rank도 풀기를 포기한 시점의 Rank이기 때문에 더 밀릴 예정.

꾸준히 해야하는데... 요즘 알고리즘 뿐만 아니라 현업에서 사용하는 기술들 공부하느라 바빠서... 라는 핑계로 넘겨야겠다.

# Biweekly Contest 25

## 1431. Kids With the Greatest Number of Candies

### 문제 설명

아이들이 가진 캔디의 숫자 배열(`candies`)과 추가로 받을 수 있는 사탕의 갯수(`extraCandies`)가 입력으로 주어진다.

각 아이들의 사탕 갯수(`candies[i]`) + 추가로 받을 수 있는 사탕의 갯수(`extraCandies`)가 가장 많은 사탕을 가진 아이의 사탕 갯수(`max(candies)`)보다 크거나 같으면 `true`, 아니라면 `false`를 세팅하는 문제이다.

### 접근 방식

1. 배열을 순회하며 가장 많은 사탕을 가진 아이의 사탕 갯수(`maxVal`)를 찾는다.
2. 배열을 순회하며 `candies[i] + extraCandies >= maxVal`이라면 `true`, 아니라면 `false`를 설정한다.

### 소스 코드

{% highlight java %}
class Solution {
    public List<Boolean> kidsWithCandies(int[] candies, int extraCandies) {
        int maxVal = -1;
        for(int i = 0; i < candies.length; i++)
            maxVal = Math.max(maxVal, candies[i]);
        
        List<Boolean> result = new ArrayList<>();
        for(int i = 0; i < candies.length; i++){
            if(candies[i] + extraCandies >= maxVal)
                result.add(true);
            else
                result.add(false);
        }
        return result;
    }
}
{% endhighlight %}

## 1432. Max Difference You Can Get From Changing an Integer

### 문제 설명

양의 정수 `num`이 입력으로 주어진다. `num`에서 숫자 하나를 고르고, 이 숫자와 같은 `num` 내의 숫자들을 모두 임의의 수로 바꾼다. 이렇게 2벌(`a`, `b`)의 숫자를 만든다. 이 `a`, `b`의 차이가 가장 크도록 만드는 문제이다.

예를 들어 `num`이 123416이라고 가정하자.

* 가장 큰 수를 만드려면 1을 9로 바꾸면 된다 -> 923496
* 가장 작은 수를 만드려면 2를 1로 바꾸면 된다 -> 113416

두 수의 차인 810080이 결과값이 된다.

### 접근 방식

* 가장 큰 수를 만드는 것은 가장 큰 자리수부터 9가 아닌 숫자를 찾아 해당 숫자에 해당하는 숫자를 모두 9로 바꾸면 된다.
* 가장 작은 수를 만드는 것은 가장 큰 자리수부터 0이 아닌 숫자를 찾아 해당 숫자에 해당하는 숫자를 모두 0으로 바꾸면 된다. 단, 가장 큰 자리수를 바꾸는 경우는 0이 아닌 1로 바꾸어야 하는 것을 주의한다.

### 소스 코드

{% highlight java %}
class Solution {
    public int maxDiff(int num) {
        //최대값으로 변경
        char[] max = String.valueOf(num).toCharArray();
        char toChange = 'a';
        for(int i = 0; i < max.length; i++){
            if(toChange !='a'){
                if(toChange == max[i])
                    max[i] = '9';
                continue;
            }
            
            if(max[i] != '9'){
                toChange = max[i];
                max[i] = '9';
            }
        }
        //최소값으로 변경
        char[] min = String.valueOf(num).toCharArray();
        toChange = 'a';
        char change = ' ';
        Set<Character> used = new HashSet<>();
        for(int i = 0; i < min.length; i++){
            if(i == 0){
                if(min.length == 1)
                    min[i] = '1';
                else if(min[i] == '1')
                    continue;
                else{
                    toChange = min[i];
                    min[i] = '1';
                    change = '1';
                }
            }
            else{
                if(min[i] == '0')
                    continue;
                else if(toChange == 'a' && min[i] != min[0]){
                    toChange = min[i];
                    min[i] = '0';
                    change = '0';
                }
                else if(toChange == min[i])
                    min[i] = change;
            }
            
        }
        
        int maxInt = Integer.valueOf(new String(max));
        int minInt = Integer.valueOf(new String(min));
        return maxInt - minInt;
    }
}
{% endhighlight %}

## 1433. Check If a String Can Break Another String

### 문제 설명

두 문자열이 주어진다. 두 문자열의 조합들을 만든다.

조합들의 각 자리를 비교할 때, 한 문자열의 모든 자리 문자가 다른 문자열의 모든 자리 문자보다 크거나 같은 경우는 Break이기 때문에 true를 반환하고 그렇지 않은 경우는 false를 반환한다.

### 접근 방식

두 문자열을 문자 배열로 변환한 뒤, 정렬을 수행한다.

Break가 성립하기 위해서는 정렬된 두 문자 배열 중 하나의 문자 배열이 다른 문자 배열보다 무조건 크면 된다는 방식으로 풀이하면 된다.

### 소스 코드

{% highlight java %}
class Solution {
    public boolean checkIfCanBreak(String s1, String s2) {
        char[] s1Arr = s1.toCharArray();
        char[] s2Arr = s2.toCharArray();
        
        Arrays.sort(s1Arr);
        Arrays.sort(s2Arr);
        
        boolean isIncrease = false, find = false;
        for(int i = 0; i < s1Arr.length; i++){
            if(s1Arr[i] == s2Arr[i])
                continue;
            else if(find){
                if(s1Arr[i] > s2Arr[i]){
                    if(!isIncrease)
                        return false;
                }
                else{
                    if(isIncrease)
                        return false;
                }
            }
            else{
                if(s1Arr[i] > s2Arr[i])
                    isIncrease = true;
                else
                    isIncrease = false;
                find = true;
            }
        }
        return true;
    }
}
{% endhighlight %}

> 처음 풀 때는 List로 풀어서 118ms가 걸렸는데, 이를 배열로 변경해서 풀어서 8ms로 단축시켰음. 굳이 List로 풀지 않아도 됐었는데... 내가... 괜한짓을....