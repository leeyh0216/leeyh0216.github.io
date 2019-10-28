---
layout: post
title:  "ClosureCleaner는 무슨 역할을 할까?"
date:   2019-09-09 21:00:00 +0900
author: leeyh0216
categories: spark
---

# ClosureClean는 무슨 역할을 할까?

Spark 논문을 보다가 아래와 같은 구문을 보았다.

> Scala represents each closure as a Java object, and these objects can be  serialized and loaded on another node to pass the closure across the network. Scala also saves any variables bound in the closure as fields in the Java object.
[출처: Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing](http://scholar.google.co.kr/scholar_url?url=https://www.usenix.org/system/files/conference/nsdi12/nsdi12-final138.pdf&hl=ko&sa=X&scisig=AAGBfm3la54b60i9jD8YgYCGOEbrdGzaSA&nossl=1&oi=scholarr)

위 내용이 의미하는 바가 정확히 이해되지 않아, 이 글을 작성한다.

## Scala의 Closure

### 바운드 변수와 자유 변수

아래는 일반적인 스칼라 함수 리터럴의 예시이다.

{% highlight scala %}
scala> val inc = (x: Int) => x + 1
inc: Int => Int = $$Lambda$781/1014824123@65bb9029

scala> inc(1)
res0: Int = 2
{% endhighlight %}

`inc`라는 이름의 함수의 인자로 전달된 `x`라는 변수는 inc 내부에서만 의미가 있다. 인자 `x`가 함수 `inc`에 종속되어 있다는 의미에서 **바운드 변수(bound variable)**라고 부른다.

위 예제를 아래와 같이 약간 바꾸어 보았다.

{% highlight scala %}
scala> var more = 1
more: Int = 1

scala> val inc = (x: Int) => x + more
inc: Int => Int = $$Lambda$792/1415937490@706eab5d
{% endhighlight %}

함수 리터럴 본문의 `more`라는 변수는 `x`와 달리 함수 리터럴에서 주어진 변수가 아니며, 함수 밖에서도 의미를 가진다. 이러한 변수를 **자유 변수(free variable)**라고 부른다.

### 