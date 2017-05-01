---
layout: post
title:  "[Spark] 동일성, 동등성, Spark의 distinct"
date:   2017-04-30 16:50:00 +0900
author: leeyh0216
categories: dev lang spark
---

> 이 문서는 Scala의 equals, hashcode와 Spark의 distinct에 관한 내용을 서술한 문서입니다.

## 동등성(Equality)와 동일성(Identity)

자바에서 객체 비교를 위해 사용되는 2개의 함수(equals, hashcode)는 매우 다른 의미를 가지고 있지만, 혼동하여 사용하는 경우가 매우 많다.
동등성은 비교 대상인 두 객체의 내용이 같은지를 비교하는 행위로써, 두 객체가 다른 객체일지라도 내용은 같을 수 있으므로 동등성의 결과가 true일 수 있다. 이는 자바 객체들이 기본적으로 가지고 있는 함수인 equals 함수로 구현한다.

하지만 동일성의 경우 비교 대상인 두 객체가 동일한 객체인지 비교하는 행위로써, 두 객체가 반드시 같은 객체여야 한다. 하지만 내용은 다를 수 있다. 따라서 동일성의 비교 결과가 true일지라도 동등성의 비교 결과는 false일 수 있다(객체의 상태를 관리하는 필드의 내용이 달라질 수 있으므로).

주의해야 할 점은, hashcode의 경우 함수이므로 매회 호출 될 때마다 사용자가 구현한 함수를 실행하여 결과를 출력하게 된다.
따라서 사용자가 hashcode 함수를 구현 시 mutable한 Field를 이용하여 hashcode를 구현하게 되면 프로그램 상의 오류가 발생할 수 있다(ex. Map, HashTable 등)

예를 들어 다음과 같은 Class가 존재한다고 가정하자.

{% highlight scala %}
class TestClass(_word : String, _comment : String) extends Serializable{
  val word = _word
  val comment = _comment
}
{% endhighlight %}

위 클래스는 case 클래스도 아니고 hashcode, equals를 구현하지 않았다.
위 클래스의 객체 2개를 생성하여 동등성, 동일성을 비교해보자.

{% highlight scala %}
val w1 = new TestClass("hello","say hi")
val w2 = new TestClass("hello","say hi")
println("동등성 비교 결과 : "+(w1.equals(w2)))
println("동일성 비교 결과 : "+(w1 == w2))

/*출력결과
동등성 비교 결과 : false
동일성 비교 결과 : false
*/
{% endhighlight %}

동일성, 동등성 비교 결과가 모두 false로 나왔다.
equals method를 다음과 같이 구현한 뒤 다시 한번 동일한 실험을 하면 동등성의 경우 true가 나오고, 동일성의 경우 false가 나온다.

{% highlight scala %}
override def equals(obj: scala.Any): Boolean = {
  if(obj.isInstanceOf[TestClass]){
    val transformedObj = obj.asInstanceOf[TestClass]
    transformedObj.word == this.word
  }
  else
    false
}
{% endhighlight %}

hashcode를 구현해주면, 동일성 비교 결과 또한 true가 나오게 된다.

{% highlight scala %}
override def hashCode(): Int = word.hashCode
{% endhighlight %}

주의해야 할 점은 String 객체의 hashCode 함수의 경우 중복될 가능성이 존재하므로, hashCode 구현 시에는 Integer 등의 숫자 타입 필드를 사용하고, 반드시 immutable Field를 활용해야한다는 점이다.

## Spark의 Distinct

Spark의 Distinct는 객체의 hashCode를 이용하여 수행된다.
따라서 equals만을 구현한 내용이 동일한 두 객체에 대해 Distinct를 수행하면 두 객체 모두 RDD내에 존재하지만, hashcode가 동일한(사실상 내용도 동일) 객체로 이루어진 RDD에 대해 distinct를 수행하면 하나의 객체만이 남아있게 된다.
