---
layout: post
title:  "[Scala] Scala의 trait"
date:   2017-03-25 22:46:00 +0900
author: leeyh0216
categories: dev lang scala
---

> 이 문서는 Scala의 trait에 대해 작성한 문서입니다.

#### Trait이란?

Programming in scala에서 Trait은 코드 재사용의 근간을 이루는 단위라고 나와 있다. Trait은 자바와 달리 다중 상속(c++에서 나오는 그 다중 상속 맞다)과 쌓을 수 있는 변경을 가능하게 한다. 단순히 Trait의 상속 기능만 사용하면, 클래스와 동일하게 사용하는 것이지만, 쌓을 수 있는 변경을 사용하면, 자바의 상속으로 이룰 수 없는 일들을 가능하게 한다.

#### Trait의 기초

Trait은 class 대신 trait 키워드를 사용해야 한다는 점 이외에는 일반적인 클래스와 다르지 않다.
{% highlight scala %}
//class 대신 trait 키워드를 이용하여 클래스와 동일하게 선언할 수 있다.
scala> trait T{
     | def hello(){
     | println("hello")
     | }
     | }
defined trait T

//trait T를 상속받아 클래스 S를 선언할 수 있다.
scala> class S extends T{
     | }
defined class S

scala> val s = new S
s: S = S@3f3afe78

//trait T에서 정의한 메소드를 해당 trait을 상속받은 클래스 S의 객체에서 사용할 수 있다.
scala> s.hello
hello
{% endhighlight %}

trait은 extends 가 아닌 with 키워드로도 상속이 가능하며, with는 extends 를 사용한 이후에는 몇 번이고 사용할 수 있으므로 1개 이상의 trait을 상속할 수 있다.
{% highlight scala %}
//A trait 정의
scala> trait A{
     | def hello(){
     | println("hello")
     | }
     | }
defined trait A

//B trait 정의
scala> trait B{
     | def bye(){
     | println("bye")
     | }
     | }
defined trait B

//A trait을 상속하고 B trait을 믹스인한 C 클래스를 선언한다.
scala> class C extends A with B{
     | }
defined class C

scala> val c = new C
c: C = C@27082746

//A trait에서 정의한 hello와 B trait에서 정의한 bye를 사용할 수 있다.
scala> c.hello
hello

scala> c.bye
bye
{% endhighlight %}

#### class와 trait의 차이점

- trait에서는 생성자 파라메터를 사용할 수 없다.
물론 내부에 메서드/필드 이외의 구문을 사용하여 생성 시에 특정 함수를 실행시킬 수는 있지만, class와 같이 생성자 파라메터를 받을 수 없다.
{% highlight scala %}
//생성자 파라메터를 사용할 경우 컴파일이 불가능하다.
scala> trait T(a : Int , b: Int){
<console>:1: error: traits or objects may not have parameters
trait T(a : Int , b: Int){
       ^

scala> trait T{
	//trait 안에 생성자가 호출될 때 호출할 구문을 넣는다.
     | println("this is constructor")
     | }
defined trait T

scala> class A extends T{
     | }
defined class A

scala> val a = new A
//Trait에서 호출한 println 문이 생성자 호출 시 호출된다.
this is constructor
a: A = A@277c0f21

{% endhighlight %}

- 일반적인 클래스 상속에서, 특정 메소드를 오버라이드 할 경우 자신의 슈퍼 클래스가 어떤 것인지 이미 아는 상태이므로, 오버라이드가 매우 제한적일 수 있다. 하지만 트레이트 믹스인을 여러번 사용하면 매우 동적인 오버라이드를 사용할 수 있다.

{% highlight scala %}
//추상 클래스 Adder를 선언하고, 내부 메서드로 Int형 변수를 매개변수로 가지고 반환형이 존재하지 않는 추상 메서드를 선언한다.
scala> abstract class Adder{
     | def add(a : Int)
     | }
defined class Adder

//추상 클래스 Adder를 상속받고, add 메서드를 오버라이드하여 기존 들어온 입력에 2를 곱한 후 슈퍼 클래스의 add를 호출하도록 한다.
scala> trait DoubleAdder extends Adder{
     | abstract override def add(a : Int){
     | super.add(a*2)
     | }
     | }
defined trait DoubleAdder

//추상 클래스 Adder를 상속받고, add 메서드를 오버라이드하여 기존 들어온 입력에 1을 더한 후 슈퍼클래스의 add를 호출하도록 한다.
scala> trait MoreOneAdder extends Adder{
     | abstract override def add(a : Int){
     | super.add(a+1)
     | }
     | }
defined trait MoreOneAdder

//추상 클래스 Adder를 상속 받고, add 메서드를 오버라이드하여 필드 result에 매개변수 a를 더한다.
scala> class DefaultAdder extends Adder{
     | var result = 0
     | def add(a : Int){
     | result+=a
     | }
     | }
defined class DefaultAdder

//DefaultAdder를 상속받고, DoubleAdder와 MoreOneAdder 순으로 믹스인한다.
scala> class First extends DefaultAdder with DoubleAdder with MoreOneAdder
defined class First

//DefaultAdder를 상속받고, MoreOneAdder와 DoubleAdder 순으로 믹스인한다.
scala> class Second extends DefaultAdder with MoreOneAdder with DoubleAdder
defined class Second

scala> val a = new First
a: First = First@63440df3

scala> a.add(1)

scala> val b = new Second
b: Second = Second@68ceda24

scala> b.add(1)

scala> println(a.result)
4

scala> println(b.result)
3

{% endhighlight %}

위에서 출력된 결과를 보면 두 결과가 다름을 알 수 있다.
일단 DefaultAdder, DoubleAdder, MoreOneAdder는 모두 추상 클래스인 Adder를 상속받았으며, 모두가 add 메서드를 오버라이드 해야 하는 상황이었다.
DefaultAdder의 경우에는 내부 필드인 result에 매개변수 a를 더하는 방식으로 오버라이드 하였고, DoubleAdder와 MoreOneAdder의 경우에는 자신의 부모 클래스의 super.add를 a*2, a+1을 매개변수로 입력하므로써 호출하였다.

결과적으로 First 클래스의 객체인 a의 add 메서드를 호출할 경우, 맨 마지막에 믹스인 된 MoreOneAdder의 add메서드가 호출되어 a의 값에 1을 더한 2를 자신보다 먼저 믹스인 된 DoubleAdder의 add메서드의 매개변수로 넘기게 된다. DoubleAdder의 add 메서드는 2를 매개변수로 받아 *2 연산을 수행한 후, 자신보다 먼저 믹스인(상속) 된 DefaultAdder의 add를 호출하게 되어 결과적으로 result값이 4가 된다.

하지만 Second 클래스의 객체인 b는 add 메서드를 호출할 경우, 맨 마지막에 믹스인 된 DoubleAdder -> MoreOneAdder -> DefaultAdder 의 add 메소드가 호출되는 형식이 되어 result가 3이 되게 된다.

각 Trait들은 자신이 슈퍼 클래스의 메소드를 오버라이드 할 당시에는 자신의 믹스인 순서를 모르기 때문에, 믹스인 순서에 따라 동적인 변경이 이루어지게 되고 이러한 과정을 스칼라에서는 쌓을 수 있는 변경이라고 부른다.
물론, 추상 클래스가 아닌 이미 동작이 정의된 클래스를 상속받은 클래스가 메서드를 abstract override로 재정의하고, 이를 특정 클래스에서 믹스인하여 사용할 수도 있다.


