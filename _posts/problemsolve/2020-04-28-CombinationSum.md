---
layout: post
title:  "LeetCode - Combination Sum"
date:   2020-04-28 22:00:00 +0900
author: leeyh0216
tags:
- ps
- leetcode
---

- 출처: [LeetCode - Combination Sum](https://leetcode.com/problems/combination-sum/)
- 난이도: 중
- 관련 기술: Array
- 풀이일
  - 2020년 04월 28일

# 문제 내용

주어진 배열의 원소를 사용하여(중복 사용 가능) 만든 배열의 합이 주어진 목표 값이 되는 배열들을 만들어 반환하는 문제이다. 결과 리스트에는 중복된 배열이 존재하면 안된다.

# Back Tracking을 이용한 풀이

Back Tracking을 이용하여 풀이하면 된다. 다만 원소를 중복 사용할 수 있기 때문에, 다음 함수를 호출하기 전에 자신이 만들 수 있는 경우의 수를 모두 소모한 뒤 호출해야 한다.

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