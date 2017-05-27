---
layout: post
title:  "[Spring] Spring 제어의 역전"
date:   2017-05-28 01:14:00 +0900
author: leeyh0216
categories: dev framework spring
---

> Spring 제어의 역전을 공부하기 위해 작성한 post 입니다.
> http://docs.spring.io/spring/docs/5.0.0.RC1/spring-framework-reference/core.html#beans 페이지를 참고하여 작성하였습니다.
## Spring의 IOC Container

### Dependency Injection(의존성 주입)과 IoC(제어의 역전)
일반적으로 자바 클래스는 다른 클래스의 객체들과 연관 되어 있다(종속적이라고도 표현할 수 있다). 이러한 종속된 객체들을 클래스의 생성자 매개변수, setter 함수 등을 통해 설정하는 과정을 DI(Dependency Injection)이라고 한다.
클래스가 생성될 때, 자신에게 필요한 의존 객체들을 컨트롤할 때, 이것을 IoC(제어의 역전)이라고 부른다.

