---
layout: post
title:  "[Scala] Scala의 클래스(기본)"
date:   2017-03-25 12:10:00 +0900
author: leeyh0216
categories: dev lang scala
---

> 이 문서는 Scala의 클래스 기본 내용에 대해 작성한 문서입니다.

### 클래스 기초
#### 클래스와 생성자

스칼라의 클래스는 자바 클래스와 유사한 형태를 가지고 있다. 자바와 동일하게 class 키워드를 이용하여 아래와 같이 클래스를 정의할 수 있다.
{% highlight scala %}
scala> class C{
     | }
defined class C
{% endhighlight %}

하지만 생성자의 경우 자바와는 다른 특성을 가지고 있다.
- 생성자 파라메터가 클래스 이름 옆에 나열된다.
- 클래스 내부에 메소드, 변수 선언이 아닌 모든 구문은 생성자의 구문이 된다.

다음 예를 보면, 스칼라 클래스의 생성자에 대해서 이해할 수 있다.
{% highlight scala %}
//클래스 이름 옆에 생성자 파라메터가 나열된다.
scala> class C(a : Int, b : Int){
//클래스 안의 메소드, 필드 선언이 아닌 구문은, 객체가 생성될 때 호출된다.
     | println("This is Constructor")
     | val c = a
     | val d = b
     | val e = a+b
     | def printe(){
     | println(e)
     | }
     | }
defined class C

scala> val c = new C(1,2)
//C 클래스의 생성자에 의해 출력된 구문
This is Constructor
c: C = C@2437c6dc

scala> c.printe()
3
{% endhighlight %}

1개 이상의 생성자를 만들고 싶을 때는 보조 생성자를 사용하면 된다. 보조 생성자는 자바 생성자와 유사하지만, 이름이 this이고 반환형이 존재하지 않는 함수를 선언하여 주 생성자를 호출하는 구문부터 시작하면 된다.
{% highlight scala %}
scala> class D(a : Int, b : Int){
	//보조 생성자는 이름이 this인 함수 형태로 선언한다.
     | def this(c : Int) = {
	//반드시 첫 부분에 주 생성자를 호출해야 한다.
     | this(1,c)
     | println("This is Second Constructor")
     | }
     | }
defined class D

scala> val d = new D(1,2)
d: D = D@5383967b

scala> val e = new D(2)
This is Second Constructor
e: D = D@3cef309d
{% endhighlight %}

#### 전제조건 확인

Scala에서는 require이라는 구문을 이용하여 생성 시에 잘못된 인자가 들어올 경우 IllegalArgumentException이 발생하도록 만들 수 있다.
{% highlight scala %}
scala> class PosNumAdder(a : Int, b : Int){
     | require(a>0)
     | require(b>0)
     | def add() : Int = {
     | a+b
     | }
     | }
defined class PosNumAdder

scala> val c = new PosNumAdder(1,2)
c: PosNumAdder = PosNumAdder@1e4a7dd4

scala> println(c.add)
3

scala> val d = new PosNumAdder(-1,0)
java.lang.IllegalArgumentException: requirement failed
  at scala.Predef$.require(Predef.scala:212)
  ... 33 elided
{% endhighlight %}
위와 같이 require문을 이용하여 해당 클래스가 생성될 때 필요한 전제조건을 설정해 놓고, 해당 전제 조건을 어기게 되면 아래와 같이 IllegalArgumentException이 발생하는 것을 볼 수 있다.

#### 필드와 메소드

필드와 메소드의 속성은 Java 와 거의 유사하고, 접근지정자를 명시하지 않았을 경우 public 으로 간주한다.
Java와 동일하게 메소드 오버로드 또한 가능하다.

{% highlight scala %}
scala> class Calculator(){
     | println("this is calculator class")
     | def add(addList : Array[Int]) = {
     | var sum = 0
     | for(i <- addList)
     | sum+=i
     | sum
     | }
	//위의 add 메소드를 오버로드한다.
     | def add(a : Int, b : Int) = {
     | a+b
     | }
     | }
defined class Calculator

scala> val c = new Calculator()
this is calculator class
c: Calculator = Calculator@368247b9

scala> c.add(1,2)
res0: Int = 3

scala> val arr = Array[Int](1,2,3,4)
arr: Array[Int] = Array(1, 2, 3, 4)

scala> println(c.add(arr))
10
{% endhighlight %}


