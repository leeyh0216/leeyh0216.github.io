---
layout: post
title:  "[Scala] Scala Basic(2)"
date:   2017-03-12 00:10:00 +0900
author: leeyh0216
categories: dev lang scala
---

> 이 문서는 Scala의 기본 개념(변수, 함수)을 이해하기 위해 작성된 문서입니다.

### 스칼라의 내장 제어구문

스칼라는 함수형 언어이다. 스칼라는 프로그램을 값을 계산하는 과정이라고 생각하며, 프로그램의 각 요소도 값을 계산하여 돌려줘야 한다는 개념으로 설계되었기 때문에, 일반적인 언어에서 값을 반환하지 않는 if, for 등도 값을 반환하도록 설계되어 있다.

### 스칼라의 if 표현식

스칼라의 if는 다른 언어와 동일하게 작동하지만, 값을 반환할 수 있다는 것이 차이점이다.
예를 들어 자바에 다음과 같은 구문이 있다고 생각해보자.

{% highlight java %}
public void static main(String[] args) throws Exception{
   String fileName = "";
   if(new File(args[0]).exists())
      fileName = args[0];
   else
      fileName = "Not Exist";
   
   System.out.println(fileName);
}
{% endhighlight %}

if문을 사용하여 해당 파일이 존재하는지 확인하고 fileName이라는 변수에 각 상황에 따라 다른 값을 넣어주는 예제이다. if문이 값을 반환하지 않기 때문에 각 코드블럭에서 fileName에 값을 할당해 주는 과정이 존재하고, fileName 변수가 final이 아니기 때문에 추후에 해당 값이 바뀔 수 있으므로(부수효과,Side Effect), 코딩 시에 주의해야 한다.

하지만 스칼라에서는 다음과 같이 값을 할당하여 이러한 문제를 방지할 수 있다.

{% highlight scala %}
import java.io.File;

val fileName = if(new File("/home/leeyh0216/scripts/test.scala").exists)
"/home/leeyh0216/scripts/test.scala"
else
"File Not Found"

println(fileName)
{% endhighlight %}

위와 같이 if문의 결과를 fileName 변수에 저장하므로써, 추후 fileName이 변경되지 않는다는 보장을 하여 부수효과(Side Effect)를 방지할 수 있다. 만일 if문을 만족하지 않아 else문을 실행해야 하는데 if문만 정의했을 경우에는 fileName에 Unit값이 할당된다.


### 스칼라의 for 표현식

#### 기본 사용법

스칼라에서 for 표현식에는 다양한 사용법이 존재한다.
일단 제너레이터라는 <- 문법을 사용하여 순회를 할 수 있다.

{% highlight scala %}
scala> val arr = Array[String]("hello","world","!!")
arr: Array[String] = Array(hello, world, !!)

scala> for(str <- arr)
     | println(str)
hello
world
!!
{% endhighlight %}
제너레이터 기준 오른쪽에 위치하는 순회 가능한 변수의 원소를 왼쪽의 값에 넣어주므로써 for문을 사용할 수 있다.
주의해야 할 점은 왼쪽에 값을 할당받는 변수는 val 변수라는 것이다. 따라서 코드 블록 상에서 해당 값을 변경할 수 없다.

또한 숫자 범위의 경우 Range 타입을 사용할 수 있다.
{% highlight scala %}
scala> val arr = Array[String]("hello","world","!!")
arr: Array[String] = Array(hello, world, !!)

scala> for(str <- arr)
     | println(str)
hello
world
!!

scala> for(i <- 1 until 3)
     | println(i)
1
2
{% endhighlight %}

to의 경우 우측에 명시된 값을 포함하는 것이고 until의 경우 우측에 명시된 값보다 작은 값까지를 의미한다.

#### 필터링

일반적으로 Java에서는 특정 값일 때는 for문 내에서 아무런 처리를 하지 않고 continue로 해당 Iter를 넘겨버리는 경우가 존재한다.
스칼라에서는 필터링이라는 기능을 이용해서 해당 값을 통과해버리는 것이 가능하다.
{% highlight scala %}
scala> for(i <- 1 to 3
     | if(i!=2)
     | )
     | println(i)
1
3
{% endhighlight %}

물론 여러 번의 필터링을 거치는 것도 가능하다
{% highlight scala %}
scala> for(i <- 1 to 5
     | if(i!=2)
     | if(i!=5)
     | )
     | println(i)
1
3
4
{% endhighlight %}

#### 중첩 순회

자바에서의 2중 for문, 3중 for문을 스칼라에서는 하나의 for문 안에서 처리할 수 있다.
다음은 간단한 구구단의 예이다.
{% highlight scala %}
scala> for(i <- 1 to 9;
     | j<- 1 to 9)
     | println(i+"x"+j+"="+(i*j))
{% endhighlight %}

계층이 1개 늘어날 때마다 세미콜론을 써주어야 하며, 세미콜론 생략을 위해서는 소괄호() 가 아닌 중괄호{} 를 사용해야 한다.
{% highlight scala %}
scala> for{i <- 1 to 9
     | j <- 1 to 9}
     | println(i*j)
{% endhighlight %}

#### 변수 바인딩

for문의 괄호 안에서 새로운 변수를 바인딩 할 수 있다.
{% highlight scala %}
scala> for(i <- 1 to 9;
     | j = i*2)
     | println(j)
{% endhighlight %}
바인딩하는 변수는 val로 선언되며 for 코드 블록 내에서만 사용할 수 있다.

다음과 같이 변수 바인딩과 필터링을 엮어 사용할 수 있는데, 필터링 시에 새로 계산한 값을 이용하여 필터링 하는 것도 가능하다
{% highlight scala %}
scala> for(i <- 1 to 9;
     | j = i*2
     | if(j>6)
     | )
     | println(i)
4
5
6
7
8
9
{% endhighlight %}

#### 새로운 컬렉션 만들기

yield 키워드를 사용하면 for문의 결과를 이용하여 새로운 컬렉션을 생성할 수 있다.
{% highlight scala %}
scala> val list = List("Hello","World","My","Name","is","yonghwan")
list: List[String] = List(Hello, World, My, Name, is, yonghwan)

scala> val list1 = for{str <- list
     | if(str.length>4)
     | rename = str+","
     | }yield rename
list1: List[String] = List(Hello,, World,, yonghwan,)
{% endhighlight %}

위와 같이 yield를 사용하면 for문에서 처리한 값을 반환할 수 있다. 결과 타입은 yield 다음에 오는 타입의 List형태이다.
예를 들어 다음과 같이 Int 형 결과를 yield했다면, 결과 타입은 List[Int]가 될 것이다.
{% highlight scala %}
scala> val list = List("Hello","World","My","Name","is","yonghwan")
list: List[String] = List(Hello, World, My, Name, is, yonghwan)

scala> val list1 = for{str <- list
     | len = str.length}
     | yield len
list1: List[Int] = List(5, 5, 2, 4, 2, 8)
{% endhighlight%}

만일 yield 뒤에도 값이 아닌 연산식이 와야한다면 중괄호{}를 사용할 수 있다.
{% highlight scala %}
scala> val list = List("hello","world")
list: List[String] = List(hello, world)

scala> val list1 = for{str <- list}
     | yield {
     | str.length
     | }
list1: List[Int] = List(5, 5)
{% endhighlight %}
