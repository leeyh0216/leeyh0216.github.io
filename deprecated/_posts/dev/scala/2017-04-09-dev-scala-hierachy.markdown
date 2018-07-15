---
layout: post
title:  "[Scala] Scala의 계층구조"
date:   2017-04-09 12:55:00 +0900
author: leeyh0216
categories: dev lang scala
---

> 이 문서는 Scala의 계층구조에 대해 서술한 문서입니다.

### 스칼라의 계층구조
스칼라의 모든 클래스는 Any 라는 클래스를 상속한다. 또한 Nothing은 모든 클래스의 서브 클래스이다. 
Any 클래스는 다시 AnyRef와 AnyVal 클래스로 분기된다. AnyRef 클래스의 경우 참조형 클래스들이 상속받는 클래스이며 Java의 Object 클래스와 동일하며, AnyVal의 경우 미리 정의되어 있는 값 타입(Int, Long, Float, Double, Boolean, Char, Byte, Short, Unit)이 상속받는 클래스이다. AnyVal을 상속받은 클래스의 경우 AnyRef를 상속받은 클래스와 달리 new 키워드를 이용하여 초기화하지 않고 리터럴을 이용하여 초기화하게 된다.


