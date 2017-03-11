---
layout: post
title:  "[Scala] Scala Basic(1)"
date:   2017-03-11 22:07:00 +0900
author: leeyh0216
categories: dev lang scala
---

> 이 문서는 Scala의 기본 개념(변수, 함수)을 이해하기 위해 작성된 문서입니다.


### 스칼라의 변수 정의

스칼라에는 val과 var 변수가 존재합니다.
val은 자바의 final 변수와 비슷한데, 한번 할당한 이후에는 할당된 값을 변경할 수 없습니다.
var의 경우 값을 할당 한 이후에도 값 변경이 가능합니다.

{% highlight scala %}
scala> val a = 3
a: Int = 3

//val 변수인 a에 다시 값을 할당하려 하면 reassignment 오류가 발생합니다
scala> a = 4
<console>:12: error: reassignment to val
       a = 4
         ^
{% endhighlight %}

자바와는 다르게 스칼라에서는 타입 정보를 생략할 수 있습니다. 이는 스칼라에 타입 추론 기능이 존재하기 때문입니다.
> val b : String = "Hello World" 와 같이 :를 활용하여 타입을 명시적으로 지정할 수 있습니다.

{% highlight scala %}
//타입 정보 없이 val 변수에 값을 대입해도, 변수 타입을 추론하여 String 타입으로 인식합니다.
scala> val b = "Hello World"
b: String = Hello World

//명시적으로 :String을 사용하여 변수 타입을 지정할 수 있습니다.
scala> val c : String = "Hello World"
c: String = Hello World
{% endhighlight %} 


### 스칼라의 함수 정의

스칼라에서 함수는 다음과 같이 작성할 수 있습니다.
{% highlight scala %}
def 함수명(매개변수) : 반환형 = {
  //함수 내용
}
{% endhighlight %}

스칼라에서 매개변수는 val 타입이며 반드시 타입을 명시해주어야 합니다.
두 Int값을 매개변수로 받아 큰 값을 리턴하는 함수를 작성해 보겠습니다.

{% highlight scala %}
scala> def max(x : Int, y : Int) : Int = {
     | if(x>y)
     | x
     | else
     | y
     | }
max: (x: Int, y: Int)Int
//스칼라에서는 명시적으로 return해주지 않아도 마지막에 쓰여있는 값이 리턴된다.

scala> val maxVal = max(3,5)
maxVal: Int = 5

scala> println(maxVal)
5
{% endhighlight %}

자바의 void는 Scala에서 Unit과 동일한 의미이다. 따라서 특정 값을 리턴하지 않는 함수는 다음과 같이 정의할 수 있다.
{% highlight scala %}
scala> def printSomething(a : String) = {
     | println(a)
     | }
printSomething: (a: String)Unit

scala> printSomething("Hello World")
Hello World
{% endhighlight %}


### 스칼라의 배열

스칼라에서 배열은 Array라는 클래스로 구현되어 있다.
일반적으로 자바에서는 배열을 다음과 같이 선언하고 사용한다.

{% highlight java %}
String[] arr = new String[3];
arr[0] = "Hello";
System.out.println(arr[0]);
{% endhighlight %}

하지만 스칼라에서는 다음과 같이 배열을 선언하고 사용할 수 있다.

{% highlight scala %}
scala> val a = new Array[String](3)
a: Array[String] = Array(null, null, null)

scala> a(0) = "Hello"

scala> a(1) = "World"

scala> println(a(0))
Hello

scala> println(a(1))
World
{% endhighlight %}

스칼라에서는 배열도 다른 클래스와 동일하다.
배열의 각 인덱스에 접근할 때에는 소괄호를 사용해서 접근하는데, 스칼라에서 객체 이름 뒤에 괄호를 사용하면 컴파일러에서 .apply()라는 함수로 변경하여 호출한다. 다른 클래스에서도 apply 함수를 정의하여 객체 이름 뒤에 호출하여 사용할 수 있다.
또한 a(0) = "Hello"와 같이 할당을 하는 부분은 컴파일러에서 해당 구문을 a.update(0,"Hello") 와 같이 변경하기 때문에 가능하다.


### 스칼라의 List, Tuple

#### 스칼라의 List

스칼라의 List는 자바의 List와 다르다. 자바에서는 List 선언 후 List에 원소를 추가하고 삭제하고 인덱스에 존재하는 원소의 값을 바꾸는 행위가 쉽다. 하지만 스칼라의 List는 변경 불가능한 객체이다. 따라서 List 선언 후 원소를 추가할 수 없고 인덱스에 존재하는 원소의 값을 변경할 수 없다.

만일 List의 앞에 새로운 원소를 추가하고 싶다면 ::(콘즈)라는 연산자를 사용하면 된다. 하지만 이런 방식보다는 ListBuffer라는 클래스를 사용하여 리스트를 조작한 후 나중에 List로 변경하는 작업이 더 빠를 것이다.

{% highlight scala %}
//리스트를 선언한다
scala> val list = List(1,2,3)
list: List[Int] = List(1, 2, 3)

scala> println(list(0))
1

//리스트에 값을 대입할 수 없다
scala> list(0) = 4
<console>:13: error: value update is not a member of List[Int]
       list(0) = 4
       ^

//::(콘즈)를 사용하여 리스트 앞에 원소를 추가하여 새로운 리스트를 생성할 수 있다
scala> val list1 = 1 :: list
list1: List[Int] = List(1, 1, 2, 3)

scala> println(list1.size)
4
{% endhighlight %}

#### 스칼라의 Tuple

자바에서와 달리 스칼라는 파이썬과 동일하게 Tuple이라는 클래스를 사용할 수 있다.
자바에서는 여러 개의 값으로 이루어진 구조체를 만들기 위해서는 클래스를 생성해야 했지만, 스칼라에서는 Tuple을 사용하여 이러한 문제를 극복 할 수 있다.

{% highlight scala %}
scala> val a = (1,"Hello",true)
a: (Int, String, Boolean) = (1,Hello,true)

scala> val b = ("Hello",false,2)
b: (String, Boolean, Int) = (Hello,false,2)

scala> println(a._1)
1

scala> println(a._2)
Hello
{% endhighlight %}

위와 같이 () 안에 값들을 넣어 Tuple을 선언할 수 있고, _(인덱스)를 사용하여 값을 출력할 수 있다. 주의해야 할 점은 배열과 달리 Tuple은 0이 아닌 1부터 값 접근을 해야한다는 점이다.
또한 Tuple의 각 원소는 변경할 수 없는 val값이다.

Tuple은 Tuple2, Tuple3... 등 Tuple을 이루는 원소의 갯수를 Tuple뒤에 붙인 이름으로 클래스가 존재한다. 명시적으로 Tuple 클래스를 사용하기 위해서는 다음과 같이 사용할 수 있다.

{% highlight scala %}
scala> val a = Tuple2(1,"Hello")
a: (Int, String) = (1,Hello)
{% endhighlight %}
