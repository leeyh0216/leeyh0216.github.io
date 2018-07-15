---
layout: post
title:  "[Spring] Spring Core - Documentation"
date:   2017-08-19 21:14:00 +0900
author: leeyh0216
categories: dev framework spring
---

# Spring의 IoC Container

## IoC container란?

Spring Framework는 Inversion of Control(제어의 역전)이라는 개념을 가지고 있다. IoC는 Dependency Injection(의존성 주입)으로도 알려져 있다.

의존성 주입은 어떤 객체가 다른 객체에 의존성을 가지고 있을 때(다른 객체를 이용하는 경우), 이를 constructor, setter 등으로 주입해주는 방식이다.

예를 들어 다음과 같은 경우가 있다고 가정해보자.

{% highlight java %}
package com.leeyh0216.test;

public class A {
	private B b;
	public A(B b){
		this.b = b;
	}
}
{% endhighlight %}

위 코드에서 A클래스는 B클래스의 객체를 생성자를 통해 초기화한다. 이처럼 어떤 클래스가 다른 객체를 이용해야 할 때 이를 외부에서 주입(생성자, setter 등을 이용하여) 받는 과정을 의존성 주입이라 한다. 일반적으로는 아래와 같이 main 함수 등에서 객체를 초기화하며 의존성을 주입해 줄 것이다.

{% highlight java %}
public class Test{
	public static void main(String[] args) throws Exception{
		B b = new B();
		A a = new A(b);
	}
}
{% endhighlight %}

그렇지만 Spring에서는 이러한 의존성 주입을 Spring의 Container에서 대신해준다.
이렇게 개발자가 아닌 프레임워크에서 객체에 필요한 의존성들을 주입해주는 과정을 제어의 역전이라고 한다.

## Spring의 BeanFactory, ApplicationContext, Bean

Spring에서 제공하는 BeanFactory 인터페이스는 모든 타입의 객체를 관리/구성하는 기능을 제공한다. ApplicationContext는 BeanFactory 인터페이스의 서브 인터페이스로써 Spring에서 제공하는 AOP(Aspect Oriented Programming), 메시지 리소스 핸들링, 이벤트 발행, Application단에서의 특정 컨텍스트 기능을 구현한다.

Spring IoC Container에 의해 관리되는 모든 객체를 빈(bean)이라고 부른다. 빈은 Spring IoC Container에 의해 초기화되고 관리된다.

ApplicationContext는 메타데이터를 이용하여 Bean들을 초기화하고 관리한다. 메타데이터는 XML, 자바 어노테이션, 자바 코드로 구성할 수 있다.