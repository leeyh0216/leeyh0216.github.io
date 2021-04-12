---
layout: post
title:  "LeetCode Weekly Contest 188"
date:   2020-05-10 19:00:00 +0900
author: leeyh0216
tags:
- ps
- leetcode
---

# 참여 후기

오늘은 Mock이 아니라 개별 문제를 풀었다. 개인 사정에 컨디션이 별로인 상태라서 Mock으로 Contest를 진행해버리면 더 Depressed 될 것 같아서..

# Weekly Contest 188

## 1441. Build an Array With Stack Operations

### 문제 설명

목표 값이 저장되어 있는 배열 `target`과 정수 `n`이 주어진다.

1 ~ n의 정수를 Push/Pop 연산을 이용하여 `target`을 만들기 위해 어떻게 Push와 Pop을 배치해야하는지 반환해야 한다.

### 접근 방식

1 ~ n까지 순회하며 현재 숫자가 `target`에 포함된 숫자인 경우 Push, 그렇지 않은 경우 Push/Pop을 결과에 집어넣는 방식으로 풀이하였다.

### 소스 코드

{% highlight java %}
class Solution {
    public List<String> buildArray(int[] target, int n) {
        List<String> result = new ArrayList<>();
        int targetIdx = 0;
        for(int i = 1; i <= n && targetIdx < target.length; i++){
            if(target[targetIdx] == i){
                result.add("Push");
                targetIdx++;
            }
            else{
                result.add("Push");
                result.add("Pop");
            }
        }
        return result;
    }
}
{% endhighlight %}

## 1442. Count Triplets That Can Form Two Arrays of Equal XOR

### 문제 설명

입력 배열 `arr`이 주어진다. 0 <= i < j <= k < arr.length 인 상황에서 i ~ j - 1까지의 XOR과 j ~ k까지의 XOR 연산의 값이 같아지는 i, j, k 쌍의 갯수를 구하는 문제이다. 

### 접근 방식

XOR 연산은 같은 숫자에 대해 수행했을 때 0이 나온다. i ~ j - 1까지 XOR한 값과 j ~ k 까지 XOR한 값이 같으려면 i ~ k까지 XOR 한 값이 0이 되어야 한다.

또한 해당 구간 내에 어떤 위치에 j가 있어도 전체 XOR은 0이 되므로 i ~ k까지의 구간만 구하면 (k - i) 내의 모든 j의 갯수를 결과로 반환하면 된다.

### 소스 코드

{% highlight java %}
class Solution {
    public int countTriplets(int[] arr) {
        int result = 0;
        for(int i = 0; i < arr.length; i++){
            int xor = arr[i];
            for(int j = i + 1; j < arr.length; j++){
                xor = xor ^ arr[j];
                if(xor == 0)
                    result += (j - i);
            }
        }
        return result++;
    }
}
{% endhighlight %}

## 1443. Minimum Time to Collect All Apples in a Tree

### 문제 설명

트리에서 사과가 있는 부분(빨갛게 칠해진 부분)을 최소 방문으로 순회하면 몇 번의 이동이 필요한지 구하는 문제이다.

### 접근 방식

일단 `edges` 배열을 통해 부모 <-> 자식 간의 관계를 나타내는 `Map<Integer, List<Integer>>` 객체를 만들었다. 부모 노드의 Key값이 Map의 Key 값이고, 자식 목록이 값(List)로 표현된다.

그 후 재귀호출(`getCost` 메서드)을 통해 비용을 계산하는데,

* 자기 자신이 사과가 있는 부분이라면 일단 2는 기본으로 소모된다(자신의 부모 -> 자신 -> 자신의 부모로 이어지는 2)
* 자기 자식들에 대해 `getCost`를 한 값을 더한다.

위의 두 값을 더한 값을 반환하는 메서드를 작성하면 된다. 단, 루트 노드인 0의 경우 2를 더하면 안된다(부모가 없으므로).

### 소스 코드

{% highlight java %}
class Solution {
    
    public int getCost(int n, Map<Integer, List<Integer>> map, List<Boolean> hasApple){
        List<Integer> childs = map.containsKey(n) ? map.get(n) : new ArrayList<>();
        int ret = 0;
        for(int i = 0; i < childs.size(); i++)
            ret += getCost(childs.get(i), map, hasApple);
        
        if(ret == 0)
            ret = hasApple.get(n) ? 2 : 0;
        else if(n != 0)
            ret = 2 + ret;
        return ret;
    }
    
    public int minTime(int n, int[][] edges, List<Boolean> hasApple) {
        Map<Integer, List<Integer>> map = new HashMap<>();
        
        for(int i = 0; i < edges.length; i++){
            if(!map.containsKey(edges[i][0]))
                map.put(edges[i][0], new ArrayList<>());
            map.get(edges[i][0]).add(edges[i][1]);
        }
        return getCost(0, map, hasApple);
    }
}
{% endhighlight %}