---
layout: post
title:  "LeetCode - Find All Anagrams in a String"
date:   2019-11-06 21:30:00 +0900
author: leeyh0216
tags:
- ps
- hashtable
- sliding window
---

- 출처: [LeetCode - Find All Anagrams in a String](https://leetcode.com/problems/find-all-anagrams-in-a-string/)
- 난이도: 중
- 관련 기술: Hashtable, Sliding Window
- 문제 요약: 문자열 S, P가 주어질 때 S 내에 존재하는 P의 아나그램을 찾는 문제이다.
- 풀이일
  - 2019년 11월 06일
  
#### 풀이 방법

Anagram은 단어나 문장을 구성하고 있는 문자의 순서를 바꾸어 다른 단어나 문장을 만드는 것이다. 즉, 두 문장이 있을 때 문장을 이루는 각 문자의 갯수만 동일하면 두 문장은 아나그램으로 볼 수 있다.

예를 들어 "aabbab"와 "ababab"가 있을 때, "aabbab"는 'a' 3개, 'b' 3개로 구성되어 있고, "ababab"도 'a' 3개, 'b' 3개로 구성되어 있기 때문에 두 문장을 서로의 아나그램이다.

s 안에 p의 아나그램이 몇 개 있는지 확인하는 방법은 아래와 같다. 여기서 s는 "abcbccbb", p는 "bbc"로 가정하도록 한다.
s[a, b]는 s의 a ~ b까지의 문자로 구성된 문장을 표현한다. 

1. p[0, p.length() - 1]에 존재하는 소문자의 갯수를 카운트한다.
   * b: 2
   * c: 1
2. s[0, p.length() - 1]에 존재하는 소문자의 갯수를 카운트한다.
   * a: 1
   * b: 1
   * c: 1
3. s[0, p.length() - 1]과 p[0, p.length() - 1]에 존재하는 소문자의 갯수가 같지 않으므로 s[0, p.length() - 1]은 p의 아나그램이 아니다.
4. s[1, 1 + p.length() - 1]에 존재하는 소문자의 갯수를 카운트 해야 한다. 2단계에서 구해놓았던 것에서 s[0, 0]을 제외하고 s[1 + p.length() - 1, 1 + p.length() - 1]을 추가하면 된다.(Sliding Window)
5. 3 과정을 반복한다.
6. 4, 5 과정을 반복하며 아나그램인 경우 s의 시작 인덱스를 결과 목록에 추가한다.

#### 코드

{% highlight java %}

class Solution {
    public List<Integer> findAnagrams(String s, String p) {
        //결과 저장용 변수
        List<Integer> result = new ArrayList<>();
        
        //p의 길이가 s보다 긴 경우 p의 Anagram이 s 내에 존재할 수 없다.
        if(s.length() < p.length())
            return result;
        
        //문장 내 소문자의 등장 횟수 카운트를 위한 변수
        int[] pCnts = new int[26], sCnts = new int[26];
        //p에 존재하는 문자를 저장하기 위한 변수
        List<Character> pChars = new ArrayList<>();
        
        for(int i = 0; i < p.length(); i++){
            //p 안에 들어 있는 소문자의 갯수를 카운트
            pCnts[p.charAt(i) - 'a']++; 
            
            //p에 존재하는 문자를 저장
            if(!pChars.contains(p.charAt(i)))
                pChars.add(p.charAt(i));
        }
        
        //s에서 0 ~ p.length() 범위에 등장하는 소문자를 카운트
        for(int i = 0; i < p.length(); i++)
            sCnts[s.charAt(i) - 'a']++;
        
        //아나그램 여부 체크
        boolean isAnagram = true;
        for(int i = 0; i < pChars.size(); i++){
            if(pCnts[pChars.get(i) - 'a'] != sCnts[pChars.get(i) - 'a']){
                isAnagram = false;
                break;
            }
        }
        if(isAnagram)
            result.add(0);
        
        //s를 p 범위로 Sliding하며 Anagram 체크
        for(int i = 1; i <= s.length() - p.length(); i++){
            sCnts[s.charAt(i-1) - 'a']--;
            sCnts[s.charAt(i + p.length() - 1) - 'a']++;
            isAnagram = true;
            for(int j = 0; j < pChars.size(); j++){
                if(pCnts[pChars.get(j) - 'a'] != sCnts[pChars.get(j) - 'a']){
                    isAnagram = false;
                    break;
                }
            }
            if(isAnagram)
                result.add(i);
        }
        return result;
    }
}

{% endhighlight %}