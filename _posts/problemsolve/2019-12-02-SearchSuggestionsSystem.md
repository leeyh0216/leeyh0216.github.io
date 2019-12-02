---
layout: post
title:  "LeetCode - Search Suggestions System"
date:   2019-12-02 20:30:00 +0900
author: leeyh0216
tags:
- ps
- hashtable
- string
---

- 출처: [LeetCode - Search Suggestions System](https://leetcode.com/problems/search-suggestions-system/)
- 난이도: 중
- 관련 기술: Hashtable, String
- 문제 요약: 물건의 목록(products)과 검색 문자열(searchWord)가 주어진다. 검색 문자열을 타이핑 할 때 각 순간에서의 문자열을 접두어로 가지는 물건의 목록을 출력하는 문제이다. 단, 출력하는 물건의 목록은 Lexical Order로 3개로 제한한다.
- 풀이일
  - 2019년 12월 02일

#### 풀이 방법

* Product: \["mousepad", "monitor"\]
* Search Word: "mouse"

가 주어졌다고 가정하자. 

1. Product를 Lexical Order로 정렬한다 => \["monitor", "mousepad"\]
2. Product를 순회하며 아래 로직을 수행한다.
   * 선택된 Product의 \[0, 0\] ~ \[0, length - 1\] 까지 순회하며 아래 로직을 수행한다.
     * \[0, 0\] => "m"이며, 이를 Map에 추가한다 => Map("m" => \["monitor"\])
     * \[0, 1\] => "mo"이며, 이를 Map에 추가한다 => Map("m" => \["monitor"\], "mo" => \["monitor"\])
     * ...
     * \[0, 6\] => "monitor"이며, 이를 Map에 추가한다 => Map("m" => \["monitor"\], "mo" => \["monitor"\], "mon" => \["monitor"\] ..., "monitor" => \["monitor"\])
3. SearchWord의 \[0, 0\] ~ \[0, length - 1\] 까지 순회하며 아래 로직을 수행한다.
   * \[0, 0\] => "m"이며 이에 해당하는 값을 Map에서 찾아 결과 배열에 추가한다 => List(List("monitor","mousepad"))
   * \[0, 1\] => "mo"이며 이에 해당하는 값을 Map에서 찾아 결과 배열에 추가한다 => List(List("monitor","mousepad"), List("monitor", "mousepad"))
   * \[0, 2\] => "mou"이며 이에 해당하는 값을 Map에서 찾아 결과 배열에 추가한다 => List(List("monitor","mousepad"), List("monitor", "mousepad"), List("mousepad"))
   * ...

4. 결과 배열을 반환한다.


#### 코드

{% highlight java %}
class Solution {
    public List<List<String>> suggestedProducts(String[] products, String searchWord) {
        //Products를 Lexical Order로 정렬
        Arrays.sort(products);
        
        //Product의 suffix로 Map을 생성한다.
        Map<String, List<String>> map = new HashMap<>();
        for(int i = 0; i < products.length; i++){
            StringBuilder sb = new StringBuilder();
            for(int j = 0; j < products[i].length(); j++){
                if(j < searchWord.length() && products[i].charAt(j) != searchWord.charAt(j))
                    break;
                sb.append(products[i].charAt(j));
                if(map.containsKey(sb.toString())){
                    List<String> list = map.get(sb.toString());
                    if(list.size() < 3){
                        list.add(products[i]);
                        map.put(sb.toString(), list);
                    }
                }
                else{
                    List<String> list = new ArrayList<>();
                    list.add(products[i]);
                    map.put(sb.toString(), list);
                }
            }
        }
        
        List<List<String>> result = new ArrayList<>();
        StringBuilder sb = new StringBuilder();
        for(int i = 0; i <searchWord.length(); i++){
            sb.append(searchWord.charAt(i));
            if(map.containsKey(sb.toString())){
                result.add(map.get(sb.toString()));
            }
            else
                result.add(new ArrayList<>());
        }
        return result;
    }
}
{% endhighlight %}