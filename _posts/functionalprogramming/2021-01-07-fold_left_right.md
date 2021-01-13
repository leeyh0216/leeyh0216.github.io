---
layout: post
title:  "FoldLeft와 FoldRight 제대로 알고 사용하기"
date:   2021-01-07 00:40:00 +0900
author: leeyh0216
tags:
- fp
---

# Fold

Fold는 주어진 결합 함수(Combining Operation)를 데이터 구조에 대해 재귀적으로 호출하여 결과 값을 만들어내는 고계 함수(Higher-Order Function)이다.

잠시 List에 대해서 생각해보자. List는 아래와 같이 두 개로 분류할 수 있다.

* 빈 List(보통 Nil이라고 부르며, []로 표현한다. Initial Value로도 사용될 수 있다)
* Prefix 역할을 하는 Element가 다른 List와 결합된 List

위 두 분류를 결합하면 List를 재귀적으로 표현할 수 있다.

예를 들어 \[1, 2, 3, 4, 5\]의 경우 아래와 같이 재귀적으로 표현할 수 있다.

1. 빈 List가 존재한다.(\[\])
2. 1의 List에 Prefix인 5를 붙여 새로운 List를 만든다.(\[5\])
3. 2의 List에 Prefix인 4를 붙여 새로운 List를 만든다.(\[4, 5\])
4. 3의 List에 Prefix인 3을 붙여 새로운 List를 만든다.(\[3, 4, 5\])
5. 4의 List에 Prefix인 2를 붙여 새로운 List를 만든다.(\[2, 3, 4, 5\])
6. 5의 List에 Prefix인 1을 붙여 새로운 List를 만든다.(\[1, 2, 3, 4, 5\])

Element와 다른 List의 결합은 [Cons](https://en.wikipedia.org/wiki/Cons)라는 함수로 대치해보면, \[1, 2, 3, 4, 5\]는 아래와 같은 함수로 표현할 수 있다.

\[1, 2, 3, 4, 5\] = Cons(1, Cons(2, Cons(3, Cons(4, Const(5, nil)))))

Fold는 이러한 Cons를 전달받은 함수로 대치하여 결과 값을 만들어 내는 고계함수이다. 예를 들어 Cons가 아닌 두 값을 더하는 Add를 Fold의 인자로 받았다면, 아래와 같이 동작할 것이다.

Result = Add(1, Add(2, Add(3, Add(4, Add(5, nil)))))

## Fold의 방향(Left, Right)

Fold는 어떤 방향으로 함수를 적용할 지에 따라 Left Fold와 Right Fold로 나뉜다.

예를 들어 1 + 2 + 3 + 4 + 5 의 경우 Add라는 재귀 함수로 Fold를 표현해보면 Left와 Right는 아래와 같이 나뉜다.

* Left Fold: Add(Add(Add(Add(1, 2), 3), 4), 5)
* Right Fold: Add(1, Add(2, Add(3, Add(4, 5))))

엄밀히 말하자면 Fold Left와 Fold Right는 아래와 같이 정의할 수 있다.

* Fold Left: **마지막 요소를 제외한** 요소들을 재귀적으로 처리한 결과와 마지막 요소를 결합
* Fold Right: **첫번째 요소를 제외한** 요소들을 재귀적으로 처리한 결과와 첫번째 요소를 결합

### Fold의 구현

FoldLeft와 FoldRight의 정의에 따라 각각의 함수를 구현해본다.

**Fold Left**

{% highlight scala %}
def foldLeft[A, B](combine: (B, A) => B, initialValue: B)(list: List[A]): B = {
    if (list.isEmpty)
      initialValue
    else
      combine(foldLeft(combine, initialValue)(list.slice(0, list.length - 1)), list.last)
  }
{% endhighlight %}

**Fold Right**

{% highlight scala %}
def foldRight[A, B](combine: (A, B) => B, initialValue: B)(list: List[A]): B = {
    if (list.isEmpty)
        initialValue
    else
        combine(list.head, foldRight(combine, initialValue)(list.tail))
}
{% endhighlight %}

#### Scala에서의 FoldLeft와 FoldRight의 구현

위와 같이 구현하는 경우 slice 함수나 재귀 함수 호출로 인한 속도 저하 문제가 발생한다.

Scala에서는 FoldLeft와 FoldRight을 아래와 같이 처리한다.

**Fold Left**

재귀함수 호출보다는 while문을 통해 head부터 tail까지 차례대로 처리한다. 함수형 언어이지만 실제 내부는 최적화를 위해 명령형으로 구현되었다.

{% highlight scala %}
def foldLeft[B](z: B)(op: (B, A) => B): B = {
  var acc = z
  var these = this
  while(!these.isEmpty) {
    acc = op(acc, these.head)
    these = these.tail
  }
  acc
}
{% endhighlight %}

**Fold Right**

Fold Right는 List를 Reverse 한 뒤 Fold Left를 적용해버린다.

{% highlight scala %}
def foldRight[B](z: B)(op: (A, B) => B): B = reverse.foldLeft(z)((right, left) => op(left, right))
{% endhighlight %}

### Combining Function별 차이(asymmetrical vs symmetrical)

Combining Function이 asymmetrical(비대칭)인지, symmetrical(대칭)인지에 따라 Folding 방식(Linear, Tree-like)이 달라지게 된다.

asymmetrical의 경우 Element와 결과의 타입이 다른 것을 의미한다. 예를 들어 Combining Function이 (A, B) => B(단, A는 B와 같지 않음)와 같이 이루어지는 함수이다. 이 경우 Tree 형식으로 계산하면 중간 결과가 (A, A) => B와 같이 표현되기 때문에 무조건 Linear한 방식을 선택해야 한다.

symmetrical의 경우 Element와 결과의 타입이 동일한 것을 의미한다. 예를 들어 Combining Function이 (A, A) => A와 같이 이루어지는 함수이다. 이 경우 Tree 형식으로 표현이 가능하다. 다만 Combining Function의 결합 법칙이 성립해야 정확한 결과를 기대할 수 있다. 

### 결합 법칙이 성립하지 않는 연산 주의하기

연산 방향이 바뀌었기 때문에, 결합 법칙이 성립하지 않는 함수의 경우 Left Fold와 Right Fold가 다른 결과를 반환할 수 있다. 예를 들어 두 수를 나누는 Div를 \[8, 7, 3\]에 적용해보자.

* Left Fold: Div(Div(8, 7), 3) = (8 / 7) / 3 = 0.38095238095
* Right Fold: Div(8, Div(7 / 3)) = 8 / (7 / 3) = 3.42857142857

이와 같이 결합 법칙이 성립하지 않는 함수에 대해 Fold를 적용할 때는 주의해야 한다.

### Lazy Evaluation과 무한수열 처리

FoldLeft는 무한수열을 처리할 수 없지만, FoldRight는 특정 조건을 만족하는 무한수열을 처리할 수 있다.

예를 들어 아래와 같은 데이터에 대해 FoldLeft와 FoldRight를 각각 수행한다고 가정해보자.

* Initial Value는 False
* Element는 무한 수열이며, 모든 값은 True
* Combining Function은 OR 연산: (a: Boolean, b: Boolean) => a || b

#### FoldRight의 경우

{% highlight scala %}
def combine(x: Boolean, y: Boolean) = x || y

def foldRight[A, B](combine: (A, B) => B, initialValue: B)(stream: Stream[A]): B = {
    if (stream.isEmpty)
        initialValue
    else
        combine(stream.head, foldRight(combine, initialValue)(stream.tail))
}

foldRight(combine, false)(Stream.continually(true))
{% endhighlight %}

위의 코드를 실행하면 어떻게 될 지 예상해보자.

1. FoldRight에서 전달된 Stream은 무한하기 때문에 `if (stream.isEmpty)` 구문은 건너뛰고 else 문으로 들어갈 것이다.
2. combine 함수가 실행되어 `true || foldRight(combine, initialValue)(stream.tail)` 구문이 완성될 것이다.
3. 2의 구문은 true 에서 이미 결과가 정해진 것이기 때문에 `foldRight(combine, initialValue)(stream.tail)` 구문을 실행할 필요 없이 반환된다.

즉, 무한수열이 있더라도 첫번째 파라메터만으로 함수의 결과를 얻을 수 있다면, 두번째 파라메터를 사용하지 않고 결과를 반환할 수 있게 된다.

그러나 위의 코드는 StackOverflow를 발생하며 끝나는데, 이는 combine 함수 호출 시 B 값이 계산되어 넘어가는 형태(Call by value)이기 때문이다. 이를 개선하기 위해 아래와 같이 Call by name parameter 방식으로 수정하면 정상적으로 동작하는 것을 확인할 수 있다.

{% highlight scala %}
def combine(x: Boolean, y: => Boolean) = x || y

def foldRight[A, B](combine: (A, =>B) => B, initialValue: B)(stream: Stream[A]): B = {
    if (stream.isEmpty)
        initialValue
    else
        combine(stream.head, foldRight(combine, initialValue)(stream.tail))
}

foldRight(combine, false)(Stream.continually(true))
{% endhighlight %}

#### FoldLeft의 경우

{% highlight scala %}
def foldLeft[A, B](combine: (B, A) => B, initialValue: B)(stream: Stream[A]): B = {
    if (list.isEmpty)
      initialValue
    else
      combine(foldLeft(combine, initialValue)(stream.slice(0, stream.length - 1)), stream.last)
  }
{% endhighlight %}

foldRight와 다르게 foldLeft는 combine에 쓰이는 파라메터가 둘 다 Stream의 마지막까지 접근해야 하기 때문에 무한 수열의 처리가 불가능하다.

# 참고 자료

* [Fold_(higher-order_function) - Wikipedia](https://en.wikipedia.org/wiki/Fold_(higher-order_function))
* [Folding Stream with Scala - void-main-args](http://voidmainargs.blogspot.com/2011/08/folding-stream-with-scala.html)