---
layout: post
title:  "[Hadoop] MapReduce"
date:   2017-03-14 00:47:00 +0900
author: leeyh0216
categories: dev hadoop mapreduce
---

> 이 문서는 Hadoop의 MapReduce 프레임워크의 이해를 위해 작성하였습니다.

### MapReduce

일반적으로 Hadoop의 MapReduce Framework를 지칭한다. 하지만 병렬 컴퓨팅, 함수형 언어 관점에서는 이를 모두 다르게 해석한다.

- 병렬 컴퓨팅 관점에서의 map : 배열의 모든 요소에 병렬적으로 적용될 수 있는 간단한 연산. 서로 독립적인(연관되지 않은) 작업들을 분해할 수 있다. 
- 함수형 언어 관점에서의 map : 배열(리스트)의 모든 요소에 적용하여 동일한 순서로 결과를 반환받을 수 있는 고차함수.

- Reduce : 함수형 언어에서 일반적으로 fold라고 불리우며, 나누어진 결과들을 합치는(재귀적으로) 작업.

### Data Locality

컴퓨터 공학에서의 Data Locality는 시간 지역성, 공간 지역성으로 나뉘는데, 이는 캐시 메모리에서 쓰이는 개념이다.(시간 지역성 - 참조했던 데이터는 또 참조할 것이다, 공간 지역성 - 옆에 있는 것도 곧 참조할 것이다)
하지만 Hadoop에서의 Data Locality의 경우 이러한 개념이 아닌, 연산이 데이터가 있는 곳에서 발생하도록 하는 것이다. 기존의 프로그램들은 Driver 프로그램으로 데이터를 불러들여 연산을 수행했지만, Hadoop은 연산 프로그램을 Data가 위치한 곳으로 이동시킨다.

### MapReduce 구성요소

#### Job Tracker

MapReduce 프로그램은 Job이라는 하나의 작업 단위로 관리되며, Job Tracker는 전체 잡의 스케쥴링을 관리하고, 모니터링한다. Job을 처리하기 위해 몇 개의 맵과 리듀스를 실행할지 계산하고, 어떤 Task Tracker에서 실행할지 결정하고(Data Locality), 해당 Task Tracker에 Job을 할당한다.

#### Task Tracker
실제로 MapReduce 프로그램을 실행한다. 일반적으로 데이터노드가 설치된 서버에서 실행된다(Data Locality). Job은 Map Task와 Reduce Task로 분리된다. 각 Task는 단독 JVM으로 실행되지만, 이러한 JVM은 재사용이 가능하도록 설정할 수 있다.

### String hashcode에 대한 고찰
public int hashCode()
Returns a hash code for this string. The hash code for a String object is computed as
 s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
 
using int arithmetic, where s[i] is the ith character of the string, n is the length of the string, and ^ indicates exponentiation. (The hash value of the empty string is zero.)

Hash Partioner의 경우 치중되도록 데이터를 분류할 수 있으니 주의해야 할 듯하다.
