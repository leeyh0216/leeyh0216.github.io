---
layout: post
title:  "[Scala] Scala의 클래스(심화)"
date:   2017-03-25 22:10:00 +0900
author: leeyh0216
categories: dev lang scala
---

> 이 문서는 Scala의 클래스 심화 내용에 대해 작성한 문서입니다.

### 클래스 심화
#### 추상 클래스

스칼라에서도 자바와 마찬가지로 추상 클래스가 존재하고, abstract 키워드를 이용하여 추상 클래스를 정의할 수 있다. 추상 메서드의 경우 abstract 키워드를 붙이지 않아도 함수 본문이 존재하지 않으면 추상 메서드로 인식한다.
{% highlight scala %}
//abstract 키워드를 이용한 추상 클래스 선언
scala> abstract class Human(name : String, sex : String){
     | val _name = name
     | val _sex = sex
     | def sayHello(){
     | println("Hello, My Name is "+_name+"!!")
     | }
	//추상 메서드. 본문이 존재하지 않는다.
     | def work() : String
     | }
defined class Human

//Human 클래스를 상속받은 Programmer 클래스. 부모 클래스의 초기화는 extends문 옆의 부모 클래스명 옆에 매개변수를 채워준다.
scala> class Programmer(name : String , sex : String, lang : String) extends Human(name,sex){
	//추상 메서드를 구현한다.
     | def work() : String = {
     | "My First Languate is "+lang+"!!"
     | }
     | }
defined class Programmer

scala> val p = new Programmer("leeyh0216","male","scala")
p: Programmer = Programmer@64f6106c

scala> p.sayHello()
Hello, My Name is leeyh0216!!

scala> println(p.work())
My First Languate is scala!!
{% endhighlight %}

필드의 경우에도 상속하는 클래스가 해당 필드를 정의할 수 있도록 남겨둘 수 있다.
{% highlight scala %}
scala> abstract class A{
//val인데도 불구하고 값을 할당하지 않아도 된다. 이는 abstract 클래스이기 때문이다.
     | val a : Int
     | }
defined class A

//A 클래스를 상속했으므로, A 클래스에서 할당해주지 않은 a의 값을 할당해주어야 한다.
scala> class B extends A{
     | val a = 10
     | }
defined class B

//A 타입의 변수에 B 타입의 객체를 넣었다
scala> val a : A = new B
a: A = B@4ac68d3e

//B 클래스는 A 클래스의 a 필드를 상속받아 값을 설정하였으므로, A 클래스의 객체라고 인식된 a의 a 필드를 출력해도 10이 나오게 된다.
scala> println(a.a)
10
{% endhighlight %}

만일, 추상 필드로 남겨놓은 값의 접근 수식자를 private로 설정할 경우 오류가 발생하게 된다. 이는 추상 클래스를 상속 받는 서브 클래스에서 해당 필드에 값을 할당할 수 없기 때문이다.
{% highlight scala %}
scala> abstract class A{
     | private val a : Int
     | }
<console>:12: error: abstract member may not have private modifier
       private val a : Int
                   ^
{% endhighlight %}

#### override 키워드

이미 슈퍼 클래스에서 정의한 메소드를 재정의하고 싶다면, override 키워드를 사용하여 해당 메서드를 구현하면 된다.
{% highlight scala %}
scala> class A{
     | def hello(){
     | println("hello world")
     | }
     | }
defined class A

scala> class B extends A{
	//override 키워드를 이용하여 슈퍼클래스 A에서 선언한 함수를 재정의한다.
     | override def hello(){
     | println("this is override hello world")
     | }
     | }
defined class B

scala> val a = new B
a: B = B@7a46a697

scala> a.hello()
this is override hello world
{% endhighlight %}
만일, 슈퍼 클래스의 메서드도 호출하고, 부가적으로 자신의 기능도 추가하고 싶다면, super.메서드명 을 재정의한 메소드에서 호출하면 된다.
{% highlight scala %}
scala> class A{
     | def hello(){
     | println("hello world")
     | }
     | }
defined class A

scala> class B extends A{
     | override def hello(){
     | println("This is additional Hello world")
	//super.hello()를 호출하여 슈퍼클래스에서 정의한 hello() 메서드를 호출한다.
     | super.hello()
     | }
     | }
defined class B

scala> val b = new B
b: B = B@7a46a697

scala> b.hello()
This is additional Hello world
hello world
{% endhighlight %}
