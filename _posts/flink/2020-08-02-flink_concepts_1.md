---
layout: post
title:  "Flink Concept - Operator, Task, Parallelism"
date:   2020-08-02 18:10:00 +0900
author: leeyh0216
categories:
- flink
---

# 개요

진행 중인 프로젝트에서 Flink를 사용할 기회가 생겼다. 처음 코드를 작성할 때는 'Spark과 거의 비슷하네?' 라는 생각을 했는데, 사용하면 할 수록 다른 부분을 많이 느끼게 되어 정리 차 글을 작성한다.

아래 자료들을 참고하여 작성하였다.

* [Apache Flink - Flink Architecture](https://ci.apache.org/projects/flink/flink-docs-release-1.11/concepts/flink-architecture.html)
* [삼성 SDS - 연산 처리의 성능 한계에 도전하는 병렬 컴퓨팅](https://www.samsungsds.com/global/ko/news/story/1203227_2919.html)
* [Streaming Processing with Apache Flink - O'REILLY](http://acornpub.co.kr/book/stream-processing-flink)
* [ZDNET - Understanding task and data parallelism](https://www.zdnet.com/article/understanding-task-and-data-parallelism-3039289129/)

## 병렬처리 개념

병렬 컴퓨팅을 사용하면 많은 데이터나 작업들을 동시 수행하여 빠르게 처리할 수 있다. 병렬 컴퓨팅은 하나의 방법이 아니며, 아래와 같이 여러 방식들이 존재한다.

* [Bit-Level Parallelism](https://en.wikipedia.org/wiki/Bit-level_parallelism)
* [Instruction Level Parallelism](https://en.wikipedia.org/wiki/Instruction-level_parallelism)
* [Data Parallelism](https://en.wikipedia.org/wiki/Data_parallelism)
* [Task Parallelism](https://en.wikipedia.org/wiki/Data_parallelism)

Flink는 Data Parallelism과 Task Parallelism을 결합하여 사용하기 때문에, 이 글에서는 Data Paralleism과 Task Parallelism만 다룬다.

### Data Parallelism

Data Parallelism은 데이터를 각기 다른 노드(혹은 프로세서)에 분산하여 병렬적으로 연산을 수행하는 방법이다.

아래와 같이 0 ~ 15까지의 합을 구하는 함수를 작성한다고 가정하자. 하나의 합을 구할 때 소요되는 시간은 약 1초이다.

{% highlight java %}
@Test
public void testSum() throws Exception {
  long start = System.currentTimeMillis();
  int sum = 0;
  for (int i = 0; i < 16; i++) {
    sum += i;
    Thread.sleep(1000);
  }
  long end = System.currentTimeMillis();
  Assert.assertEquals(120, sum);
  System.out.println(String.format("Elapsed: %d", (end - start)));
}
{% endhighlight %}

총 16초 가량이 소요된 것을 확인할 수 있다.

여기에 Data Parallelism을 적용한다고 하면 아래와 같은 코드를 작성할 수 있다.

{% highlight java %}
class SumRunnable implements Runnable {
  AtomicInteger sum;
  int start, end;

  public SumRunnable(AtomicInteger sum, int start, int end) {
    this.sum = sum;
    this.start = start;
    this.end = end;
  }

  @Override
  public void run() {
    for (int i = start; i < end; i++) {
      sum.addAndGet(i);
      try {
        Thread.sleep(1000);
      } catch (Exception e) {
        //Nothing to do
      }
    }
  }
}

@Test
public void testDataParallelism() throws Exception {
  long startMillis = System.currentTimeMillis();
  AtomicInteger sum = new AtomicInteger(0);
  List<Thread> threads = new ArrayList<>();
  for (int i = 0; i < 4; i++) {
    int start = i * 4;
    int end = start + 4;
    Thread t = new Thread(new SumRunnable(sum, start, end));
    t.start();
    threads.add(t);
  }
  for (Thread t : threads)
    t.join();
  long endMillis = System.currentTimeMillis();
  Assert.assertEquals(120, sum.get());
  System.out.println(String.format("Elapsed: %d", (endMillis - startMillis)));
}
{% endhighlight %}

0 ~ 16까지의 값을 \[0 ~ 3\], \[4 ~ 7\], \[8 ~ 11\], \[12 ~ 15\] 로 총 4개의 범위로 분할한 뒤, 각 범위를 하나의 프로세서(스레드)에게 할당하였다.

기존 N의 작업을 N/4 씩 나누어 동시에 수행하기 때문에 전체 소요 시간도 16초 -> 4초로 감소한 것을 확인할 수 있다.

### Task Parallelism

Task Parallelism은 데이터가 아닌 작업들을 나누어 수행하는 방법이다.

흔히 볼 수 있는 Task Parallelism은 Pipelining이 있다. 자동차 공장에서 자동차를 만들 때 수행해야 하는 공정이 아래와 같다고 하자.

* 부품 만들기(A)
* 차체 만들기(B)
* 조립하기(C)
* 도장하기(D)
* 내장 조립하기(E)
* 하부 조립하기(F)

이 중 A와 B, E와 F는 서로 동시에 진행될 수 있다고 할 때 공정을 아래와 같이 나눌 수 있을 것이다.(괄호 안의 공정은 동시에 수행될 수 있다는 의미)

(A, B) -> C -> D -> (E, F)

동일한 차(데이터)에 대해 A, B와 E, F 작업을 나누어 동시에 처리하였다. 또 다른 예제는 아래와 같다.

입력에 대해 합을 구하는 연산자(A)와 곱을 구하는 연산자(B)가 있고, 이를 처리할 수 있는 프로세서 2개(Processor1, Processor2)가 있다고 하자. Task Parallelism은 Processor1에 A 연산, Processor2에 B 연산을 할당하여 입력 데이터에 대해 각기 다른 연산을 동시에 처리한다.

예를 들어 아래와 같이 특정 구간의 합과 곱을 구하는 함수를 작성한다고 하자. 합 연산의 경우 1초, 곱 연산의 경우 2초가 소요된다고 가정하자.

{% highlight java %}
@Test
public void testSumAndMultiple() throws Exception {
  int sum = 0, multiple = 1;
  long start = System.currentTimeMillis();
  for (int i = 1; i <= 8; i++) {
    sum += i;
    Thread.sleep(1000);
  }
  for (int i = 1; i <= 8; i++) {
    multiple *= i;
    Thread.sleep(2000);
  }
  long end = System.currentTimeMillis();

  Assert.assertEquals(36, sum);
  Assert.assertEquals(40320, multiple);
  System.out.println(String.format("Elapsed: %d", (end - start)));
}
{% endhighlight %}

총 24초의 시간이 소요된 것을 확인할 수 있다.

여기에 Task Parallelism을 적용하면 아래와 같은 코드를 작성할 수 있다.

{% highlight java %}
class SumRunnable implements Runnable {
  AtomicInteger sum;
  int start, end;

  public SumRunnable(AtomicInteger sum, int start, int end) {
    this.sum = sum;
    this.start = start;
    this.end = end;
  }

  @Override
  public void run() {
    for (int i = start; i <= end; i++) {
      sum.addAndGet(i);
      try {
        Thread.sleep(1000);
      } catch (Exception e) {
        //Nothing to do
      }
    }
  }
}

class MultipleRunnable implements Runnable {
  AtomicInteger multiple;
  int start, end;

  public MultipleRunnable(AtomicInteger multiple, int start, int end) {
    this.multiple = multiple;
    this.start = start;
    this.end = end;
  }

  @Override
  public void run() {
    for (int i = start; i <= end; i++) {
      final int toMultiple = i;
      multiple.updateAndGet(v -> v * toMultiple);
      try {
        Thread.sleep(2);
      } catch (Exception e) {
        //Nothing to do
      }
    }
  }
}

@Test
public void testTaskParallelism() throws Exception {
  AtomicInteger sum = new AtomicInteger(0), multiple = new AtomicInteger(1);

  long start = System.currentTimeMillis();
  Thread sumThread = new Thread(new SumRunnable(sum, 1, 8));
  Thread multipleThread = new Thread(new MultipleRunnable(multiple, 1, 8));
  sumThread.start();
  multipleThread.start();
  sumThread.join();
  multipleThread.join();
  long end = System.currentTimeMillis();

  Assert.assertEquals(36, sum.get());
  Assert.assertEquals(40320, multiple.get());
  System.out.println(String.format("Elapsed: %d", (end - start)));
}
{% endhighlight %}

합 연산을 수행하는 작업(`sumThread`)과 곱 연산을 수행하는 작업(`multipleThread`)을 각 작업 당 하나의 프로세서(스레드)가 수행할 수 있도록 하였다.

결과적으로 총 소요 시간이 24초 -> 8초로 줄어든 것을 확인할 수 있었다.

### 병렬처리와 성능

병렬처리를 통해 얻을 수 있는 성능 향상은 [암달의 법칙(Amdahl's law)](https://ko.wikipedia.org/wiki/%EC%95%94%EB%8B%AC%EC%9D%98_%EB%B2%95%EC%B9%99)과 [구스타프슨의 법칙(Gustafson's Law)](https://ko.wikipedia.org/wiki/%EA%B5%AC%EC%8A%A4%ED%83%80%ED%94%84%EC%8A%A8%EC%9D%98_%EB%B2%95%EC%B9%99)에서 다룬다.

#### 암달의 법칙(Amdahl's law)

위 Task Parallelism의 자동차 예제에서도 나왔지만, 작업 파이프라인 중에는 병렬화가 가능한 부분도 있지만 병렬화가 불가능한 부분(직렬)도 있다.

암달의 법칙은 컴퓨터 시스템의 일부를 개선(병렬화)할 때 전체적으로 얼마만큼의 성능 향상(기존 직렬적인 부분 + 병렬화된 부분)이 있는지 계산하는데에 사용된다. 공식은 아래와 같다.

전체 성능 향상 = 1 / ((1 - P) + (P / S))

위 공식에서 P는 성능 향상을 이루어낸(병렬화된) 부분의 비율이고, S는 향상된 성능을 나타낸다. 예를 들어 작업의 80%에 대해 속도를 4배로 늘렸다 하면 아래와 같이 계산할 수 있다.

전체 성능 향상 = 1 / ((1 - 0.8) + (0.8 / 4)) = 2.5

작업의 80%에 대해 4배의 성능 향상을 이루어냈더라도 전체적으로는 고작 2.5배의 성능 향상밖에 이루어내지 못했다. 아래와 같이 작업의 99%에 대해 수천, 수만 배의 성능 향상이 이루어진다 해도 전체적으로는 최대 20배의 성능 향상 밖에 이루어내지 못한다는 절망적인 법칙이다.

![암달의 법칙, 출처: Wikipedia](https://upload.wikimedia.org/wikipedia/commons/thumb/e/ea/AmdahlsLaw.svg/600px-AmdahlsLaw.svg.png)

> 그림 출처: [Wikipedia - 암달의 법칙](https://ko.wikipedia.org/wiki/%EC%95%94%EB%8B%AC%EC%9D%98_%EB%B2%95%EC%B9%99)

#### 구스타프슨의 법칙(Gustafson's Law)

구스타프슨의 법칙은 암달의 법칙과 반대로 대용량 작업은 효율적으로 병렬화 할 수 있다는 가정 하에 만들어진 법칙이다. 공식은 아래와 같다.

S(P) = P - α(P - 1)

위 공식에서 P는 프로세서의 수, S는 성능 향상, α는 병렬화되지 않는 순차적인 부분의 비율이다. 자세한 공식 유도 방법은 [Wikipedia - 구스타프슨의 법칙](https://ko.wikipedia.org/wiki/%EA%B5%AC%EC%8A%A4%ED%83%80%ED%94%84%EC%8A%A8%EC%9D%98_%EB%B2%95%EC%B9%99)을 참고하면 좋다.

위 공식대로라면 순차적인 부분의 비율이 낮을 수록 거의 프로세서 수 만큼의 성능 향상을 이뤄낼 수 있다. 따라서 문제의 크기가 커진다 해도 프로세서의 수를 늘리면 동일한 시간 안에 처리할 수 있다는 결론이 나오게 된다.

---

> Spark나 Flink와 같은 병렬 처리 프레임워크를 사용하며 데이터 처리를 수행하며 느낀 것은 어느정도 구스타프슨의 법칙이 맞아 떨어진다는 것이다. 거의 대부분의 경우에서 프로세서(혹은 장비)의 수를 늘리면 전체적인 성능 향상이 크게 나타나는 것을 느꼈다.
>
> 다만 현실 세계 문제에서는 Shuffle 등에 사용되는 Network Overhead나 Source, Sink 과정에서 사용되는 Persistent Storage 들의 성능이 오히려 전체 파이프라인 성능에 크게 영향을 주는 것이 많았던 것 같다.

## Flink에서의 병렬 처리

![Apache Flink - Task and operators](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/tasks_chains.svg)
> 그림 출처: [Apache Flink - Task and operator chains](https://ci.apache.org/projects/flink/flink-docs-release-1.11/concepts/flink-architecture.html)

### Operator, Task

Operator는 Flink에서 수행되는 연산의 가장 작은 단위로 `map`, `flatMap` 등이 존재한다. 이러한 Operator들을 Chaining하여 Task로 만들 수 있다.

각 Task는 하나의 Thread에서 수행되며, Operator를 Task로 묶음으로써 Task 간 데이터 교환 오버헤드를 줄일 수 있다. Flink에서 Task는 동일한 TaskManager에서 수행될 수도 있지만, 다른 장비에서 수행되고 있는 TaskManager에 의해 실행될 수도 있기 때문에, Task 간 데이터 교환에는 아래와 같은 작업이 수행된다.

* Task A의 결과 데이터를 Serialization 한다.
* Serialization 된 결과 데이터를 TCP를 통해 다른 장비에서 실행 중인 TaskManager로 전달한다.
* 데이터를 수신한 TaskManager는 Deserialization을 수행하여 새로운 Task의 입력 데이터로 활용한다.

여러 개의 Operator를 한 개의 Task로 묶으면 하나의 Thread에서 Method Chaining 처럼 동작하기 때문에 위와 같은 Overhead의 발생을 피할 수 있다.

### TaskManager와 Task Slot

Flink에서 TaskManager는 Task의 실행을 담당하는 컴포넌트이다. TaskManager는 1개의 JVM Process로써 동작하며, 내부적으로 1개 이상의 스레드를 통해 1개 이상의 Task를 실행할 수 있다.(Task Parallelism)

위의 각 스레드를 Task Slot이라 한다. 1개의 TaskManager가 1개 이상의 Task Slot을 소유하므로써 가지는 특징은 아래와 같다.

#### JVM 메모리 공유

하나의 TaskManager에 속한 Task Slot들은 TaskManager의 메모리를 나누어 사용한다. 예를 들어 600MB의 Heap 공간을 가진 TaskManager에 3개의 Slot이 존재한다면, 각 Slot은 200MB 씩 메모리를 나누어 가진다.

> JVM 실행에 필요한 메모리가 존재하기 때문에, TaskManager에 Slot의 수가 많을 수록 이러한 메모리 오버헤드를 줄일 수 있다.

#### CPU Isolation 불가

다만 1개의 TaskManager에 속한 Slot들은 JVM Thread로써 동작하기 때문에 CPU Isolation은 이룰 수 없다. CPU Isolation이 필요하다면 TaskManager 별로 Slot을 1개씩 가지게 하고, 각 TaskManager를 통해 CPU Isolation을 달성해야 한다.

#### TaskManager에서 사용하는 자원 공유

TaskManager에는 데이터 교환을 위한 TCP 연결이나 내부 객체 등 다양한 리소스가 존재한다. 하나의 TaskManager에 속한 Slot들은 이러한 자원을 공유하게 된다.

### Task와 Subtask 그리고 Slot Sharing

![Apache Flink - Slot Sharing](https://ci.apache.org/projects/flink/flink-docs-release-1.11/fig/slot_sharing.svg)
> 그림 출처: [Apache Flink - Task Slots and Resources](https://ci.apache.org/projects/flink/flink-docs-release-1.11/concepts/flink-architecture.html#task-slots-and-resources)

1개의 Task는 1개 이상의 Subtask로 나뉠 수 있다. 즉, 동일한 연산(Task, Operator Set)을 수행하는 Processor들이 여러 개 존재하는 구조이다.(Data Parallelism)

위의 Task Slot에서 실행하는 작업의 단위는 사실 Task가 아닌 Subtask 단위이다. 만일 2개의 Task(Task A, Task B)가 존재하고, 각 Task마다 3개의 Subtask로 실행해야 한다고 가정하면, 총 6개의 Slot이 필요하다.

그런데 Task A는 Task B에 비해 매우 빠르게 수행되는 연산이라고 가정해보자. Task A를 담당하는 CPU는 Utilization이 낮고, Task B를 담당하는 CPU는 Utilization이 매우 높을 것이다. 이러한 비효율을 줄이기 위해 Flink에서는 Slot Sharing을 지원한다.

Slot Sharing은 Job을 구성하는 Task 중 가장 높은 Parallelism(Subtask의 수)만큼의 Slot을 만들고, 모든 Task들이 이 Slot을 공유하는 방식이다.

어차피 Subtask들은 Thread이기 때문에 Time slicing을 통해 균등하게 작업 시간을 할당받을 것이고, Task A 처리가 끝나고 Task B 처리만 남았을 때는 Task B가 모든 CPU 자원을 사용할 수 있기 때문에 매우 효율적인 자원 운용이 가능해진다.