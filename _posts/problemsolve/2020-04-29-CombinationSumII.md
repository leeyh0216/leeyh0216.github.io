---
layout: post
title:  "LeetCode - Combination Sum II"
date:   2020-04-29 12:37:00 +0900
author: leeyh0216
tags:
- ps
- leetcode
---

- 출처: [LeetCode - Combination Sum II](https://leetcode.com/problems/combination-sum-ii/)
- 난이도: 중
- 관련 기술: Array, Back Tracking
- 풀이일
  - 2020년 04월 29일

# 문제 내용

주어진 배열의 원소를 사용하여 만든 배열의 합이 주어진 목표 값이 되는 배열들을 만들어 반환하는 문제이다.

[LeetCode - Combination Sum](https://leetcode.com/problems/combination-sum/)과 거의 동일한 문제인데, 조건이 약간 다르다.

Combination Sum에서는 원본 배열에 중복된 요소가 없지만, 각 요소를 여러번 사용해도 되는 반면, Combination Sum II에서는 원본 배열에 중복된 요소가 있지만, 각 요소를 한번씩만 사용할 수 있다.

# Back Tracking을 이용한 풀이

Back Tracking을 이용하여 풀이하면 된다. 다만 배열에 중복된 원소가 존재하고 한번씩만 사용해야 하기 때문에, 다음과 같이 접근해야 한다.

1. 원본 배열을 정렬한다.
2. 백트래킹 시 한 단계에서 동일한 원소를 1번 ~ N번까지 사용하고 다음 원소로 넘어가야 한다. 이렇게 안하고 한 단계에서 하나씩 꺼내쓰면 중복이 발생해보린다.

{% highlight java %}
class Solution {
    
    List<List<Integer>> result = new ArrayList<>();
    
    void makeCombination(int[] arr, int target, int idx, int sum, List<Integer> combination){
        if(arr.length == idx)
            return;
        
        //현재 값을 포함하지 않은 Combination 생성
        makeCombination(arr, target, idx + 1, sum, combination);
        
        //현재 값을 포함한 Combination 생성
        int i = 1;
        for(; sum + arr[idx] * i <= target; i++){
            combination.add(arr[idx]);
            if(sum + arr[idx] * i == target){
                List<Integer> tmp = new ArrayList<>();
                tmp.addAll(combination);
                result.add(tmp);
            }
            else{
                makeCombination(arr, target, idx + 1, sum + arr[idx] * i, combination);
            }
        }
        
        //넣었던 데이터를 제거해줌
        for(int j = i; j > 1; j--)
            combination.remove(combination.size() - 1);
    }
    
    public List<List<Integer>> combinationSum(int[] candidates, int target) {
        makeCombination(candidates, target, 0, 0, new ArrayList<>());
        return result;
    }
}
{% endhighlight %}