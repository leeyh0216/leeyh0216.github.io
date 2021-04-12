---
layout: post
title:  "LeetCode Weekly Contest 187"
date:   2020-05-03 20:00:00 +0900
author: leeyh0216
tags:
- ps
- leetcode
---

# 참여 결과 및 후기

| Rank        | Score | Finish Time | Q1      | Q2      | Q3      | Q4 |
|-------------|-------|-------------|---------|---------|---------|----|
| 860 / 9245  | 12    | 8:09:00	    | 7:36:30 | 7:40:23 | 8:09:00 | -  |

오전에 일정이 있어 정규 Contest 시간에는 참여하지 못하고 Mock Contest로 참석했다. 어차피 시간이 되었어도 요즘 꾸준히 문제를 풀지 않았었기 때문에 자신이 없어 참여하지는 않았을 것 같다.

4번 문제는 어려워서 풀지 못했고, 30분만에 3문제를 풀이했다. 다시 꾸준히 하루에 3문제씩 풀어 1달 뒤에는 정규 Contest에 참여해야겠다.

# Weekly Contest 187

## 1436. Destination City

### 문제 설명

2차원 배열 형태(실제로는 `List<List<String>>`으로 입력이 주어짐)로 두 도시 간 이동경로(`\["Seoul", "Busan"\]`과 같이)가 주어진다.

모든 이동경로는 따라가다보면 1개 도시로 끝나게 된다. 이 1개 도시를 찾는 것이 문제이다.

### 접근 방식

정답이 되는 마지막 도시는 목적지에만 등장하고 시작지에는 등장하지 않을 것이다. 모든 경로를 순회하며 시작, 종료 도시를 Set에 저장하고, 종료 Set에는 존재하지만 시작 Set에는 존재하지 않는 도시를 찾았다.

### 소스 코드

{% highlight java %}
class Solution {
    public String destCity(List<List<String>> paths) {
        Set<String> startCitySet = new HashSet<>();
        Set<String> endCitySet = new HashSet<>();
        
        for(int i = 0; i < paths.size(); i++){
            String start = paths.get(i).get(0);
            String end = paths.get(i).get(1);
        
            startCitySet.add(start);
            endCitySet.add(end);
        }
        
        Iterator<String> iter = endCitySet.iterator();
        while(iter.hasNext()){
            String city = iter.next();
            if(!startCitySet.contains(city))
                return city;
        }
        return "";
    }
}
{% endhighlight %}

## 1437. Check If All 1's Are at Least Length K Places Away

### 문제 설명

0, 1로 이루어진 배열과 정수 K가 주어진다. 배열 내 1 사이의 거리가 모두 K 이내인지 확인하는 문제이다.

### 접근 방식

이전 1의 위치를 저장해두고, 1이 등장할 때마다 거리를 측정하여 K 를 초과하는 거리가 계산될 경우 false를 반환하는 형식으로 풀이하였다.

### 소스 코드

{% highlight java %}
class Solution {
    public boolean kLengthApart(int[] nums, int k) {
        int lastPos = -1;
        
        for(int i = 0; i < nums.length; i++){
            if(nums[i] == 1){
                if(lastPos == -1)
                    lastPos = i;
                else if(i - lastPos - 1 >= k)
                    lastPos = i;
                else
                    return false;
            }
        }
        return true;
    }
}
{% endhighlight %}

## 1438. Longest Continuous Subarray With Absolute Diff Less Than or Equal to Limit

### 문제 설명

입력으로 배열과 limit 값이 주어진다. 가장 작은 값의 차의 절대 값이 limit보다 작거나 같은 부분 배열의 최대 길이를 반환하는 문제이다.

### 접근 방식

Sliding window와 TreeMap으로 풀이하였다. TreeMap은 `TreeMap<Integer, List<Integer>>` 타입을 써서 키를 배열의 값, 값을 배열의 인덱스 목록으로 지정하였다.

Sliding window의 시작과 끝을 leader와 follower라고 가정하고 아래 로직으로 접근하였다.

1. 현재 leader가 위치한 배열의 값(`nums[leader]`)을 TreeMap에 추가한다.
2. TreeMap의 가장 큰 값(`lastKey()`)과 가장 작은 값(`firstKey()`)의 차의 절대값이 limit보다 크다면 follower를 증가시킨다.
   * 이 때 TreeMap에서 follower에 해당하는 Key에 접근하여 List에서 해당 Index(0)을 제거하고, 배열이 비었다면 Key 자체를 삭제시킨다.
3. 최대 길이 값을 (leader - follower + 1)과 비교하여 갱신한다.

### 소스 코드

{% highlight java %}
class Solution {
    public int longestSubarray(int[] nums, int limit) {
        TreeMap<Integer, List<Integer>> map = new TreeMap<>();
        int leader = 0, follower = 0, maxLen = 0;
        
        while(leader < nums.length){
            int currentVal = nums[leader];
            if(map.containsKey(currentVal))
                map.get(currentVal).add(leader);
            else{
                List<Integer> list = new LinkedList<>();
                list.add(leader);
                map.put(currentVal, list);
            }
            // System.out.println(leader);
            // System.out.println(map.size());
            // System.out.println(map.firstKey());
            // System.out.println(map.lastKey());
            while(Math.abs(map.firstKey() - map.lastKey()) > limit){
                map.get(nums[follower]).remove(0);
                if(map.get(nums[follower]).isEmpty())
                    map.remove(nums[follower]);
                follower++;
            }
            maxLen = Math.max(maxLen, leader - follower + 1);
            leader++;
        }
        return maxLen;
    }
}
{% endhighlight %}