---
layout: post
title:  "ThreadPoolExecutor에 대한 오해와 진실"
date:   2020-02-21 01:00:00 +0900
author: leeyh0216
tags:
- java
---

# ThreadPoolExecutor에 대한 오해와 진실

회사에서 팀원 분이 코드 리뷰를 해주셨는데, `ThreadPoolExecutor`을 잘못 사용하고 있다는 내용이었다.

내가 작성한 원본 코드는 대략 아래와 같다.

```
int numTasks = 60;
CountDownLatch countDownLatch = new CountDownLatch(numTasks);
ThreadPoolExecutor threadPoolExecutor= new ThreadPoolExecutor(10, 50, 10, TimeUnit.SECONDS, new LinkedBlockingQueue<>());

for(int i = 0; i < numTasks; i++){
    threadPoolExecutor.submit(() -> {
        //Do something
        countDownLatch.countDown();
    });
}
```

단순히 `ThreadPoolExecutor`의 초기화 인자만 보고 난 다음과 같이 추측했다.(Do something은 충분히 처리 시간이 긴 작업이 수행된다고 가정)

1. `ThreadPoolExecutor` 초기화 직후에는 `Thread`가 존재하지 않는다.
2. Loop에서 10개의 작업이 submit 된 이후까지는 순차적으로 `Thread`가 생성되어 총 10개의 `Thread`가 존재한다.
3. 11번째 ~ 50번째의 작업은 `corePoolSize`를 넘어섰으나, `maxPoolSize`에 도달하지 않았기 때문에, 동적으로 `Thread`의 수가 증가한다. 즉, 50개의 `Thread`가 존재한다.
4. 51번째 ~ 60번째의 작업은 `ThreadPoolExecutor`의 Thread 수가 `maxPoolSize`를 넘어섰기 때문에 `Queue`에서 대기한다.

위의 추측은 **완전히 틀렸다**.

## 실제로는 어떻게 동작하는가?

동시에 실행되는 최대 작업의 갯수(Active Count)와 `Queue`에서 대기하는 `Runnable`의 갯수가 몇 개인지 확인해보기 위해 아래와 같이 500ms 마다 출력 구문을 추가하였다.

```
int numTasks = 60;
BlockingQueue<Runnable> blockingQueue = new LinkedBlockingQueue<>();
CountDownLatch countDownLatch = new CountDownLatch(numTasks);
ThreadPoolExecutor threadPoolExecutor= new ThreadPoolExecutor(10, 50, 10,
    TimeUnit.SECONDS, blockingQueue);

for(int i = 0; i < numTasks; i++){
    threadPoolExecutor.submit(() -> {
        try {
            Thread.sleep(1000);
        }
        catch(Exception e){
		}
        countDownLatch.countDown();
    });
}

for(int i = 0; i < 120; i++){
    Thread.sleep(500);
	//현재 실행 중인 Thread의 수 출력
    System.out.println("Active: " + threadPoolExecutor.getActiveCount());
	//Queue에서 대기 중인 작업 갯수 출력
    System.out.println("Queue: " + blockingQueue.size());
}

threadPoolExecutor.shutdown();
```

위 코드를 실행한 결과는 아래와 같다.

```
Active: 10
Queue: 50
Active: 10
Queue: 40
Active: 10
Queue: 40
Active: 10
Queue: 30
Active: 10
Queue: 30
Active: 10
...
```

한 번에 동시에 실행되는 갯수는 `ThreadPoolExecutor`를 초기화 할 때 설정해주었던 `corePoolSize`인 10개이고, 그 갯수를 넘어서면 `maxPoolSize`만큼 동적으로 확장되어 실행되는 것이 아니고, `Queue`에서 대기를 하고 있었다.

## 뭐가 잘못된 걸까?

그냥 내가 `ThreadPoolExecutor`의 JavaDoc을 제대로 읽지 않은게 실수였다.

오라클에서 제공하는 [ThreadPoolExecutor의 JavaDoc](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html)을 읽어보면, `corePoolSize`와 `maxPoolSize`에 대해 다음과 같이 기술되어 있다.

> A ThreadPoolExecutor will automatically adjust the pool size (see getPoolSize()) according to the bounds set by corePoolSize (see getCorePoolSize()) and maximumPoolSize (see getMaximumPoolSize()). When a new task is submitted in method execute(Runnable), and fewer than corePoolSize threads are running, a new thread is created to handle the request, even if other worker threads are idle. If there are more than corePoolSize but less than maximumPoolSize threads running, a new thread will be created only if the queue is full. By setting corePoolSize and maximumPoolSize the same, you create a fixed-size thread pool. By setting maximumPoolSize to an essentially unbounded value such as Integer.MAX_VALUE, you allow the pool to accommodate an arbitrary number of concurrent tasks. Most typically, core and maximum pool sizes are set only upon construction, but they may also be changed dynamically using setCorePoolSize(int) and setMaximumPoolSize(int).

정리해보자면 `execute` 혹은 `submit`을 통해 새로운 `Runnable` 실행을 `ThreadPoolExecutor`에게 요청했을 때, 아래와 같이 처리된다.

1. `corePoolSize`보다 적은 `Thread`가 수행되고 있었던 경우: 실행 요청한 `Runnable`을 수행하기 위한 `Thread`를 새로 생성하여 즉시 실행한다.
2. `corePoolSize`보다 많은 `Thread`가 수행되고 있지만, `maxPoolSize`보다 적은 수의 `Thread`가 수행되고 있는 경우:
   1. `Queue`가 가득 차지 않은 경우: 즉시 실행하지 않고 `Queue`에 `Runnable`을 넣는다.
   2. `Queue`가 가득 찬 경우: `maxPoolSize`까지 `Thread`를 더 만들어 실행한다.

즉, `maxPoolSize`만큼 확장되는 것보다 `Queue`를 채우는 작업이 우선한다. `Queue`의 크기를 넘어선 수의 작업들이 요청되었을 때만 `maxPoolSize`만큼 확장된다.

## `maxPoolSize`가 동작하게 하는 방법

`maxPoolSize` 옵션이 동작하게 하려면 `Queue` 초기화 시 `Capacity`를 고정시키면 된다.

위에서 `Queue`를 초기화 할 때는 `Capacity`를 지정하지 않은 `LinkedBlockingQueue`를 할당하였다.

```
BlockingQueue<Runnable> blockingQueue = new LinkedBlockingQueue<>();
```

위 기본 생성자의 경우 `Queue`의 `Capacity`를 `Integer.MAX_VALUE`로 설정해버린다.

```
public LinkedBlockingQueue() {
    this(2147483647);
}
```

그렇기 때문에 아래와 같이 고정된 `Capacity`로 설정해야 한다. 나는 10 크기를 가지는 `Queue`로 초기화했다.

```
BlockingQueue<Runnable> blockingQueue = new LinkedBlockingQueue<>(10);
```

이후 다시 코드를 수행하면 아래와 같은 출력이 발생한다.

```
Active: 50
Queue: 10
Active: 10
Queue: 0
Active: 10
Queue: 0
...
```

전체 제출 작업의 수가 60개, `corePoolSize`가 10개, `maxPoolSize`가 50개, `Queue`의 `Capacity`가 10이기 때문에, 아래와 같이 동작할 것이다.

1. 최초 제출된 10개의 작업은 바로 `Thread`가 생성되어 실행된다.(Active: 10)
2. 그 다음 제출된 10개의 작업은 `Queue`에 쌓인다.(Active: 10)
3. 그 다음 제출된 40개의 작업은 `Queue`가 가득 찼기 때문에 아래와 같이 수행된다.
   1. `Queue`에 있던 10개의 작업 실행(Active: 20)
   2. 또 10개의 작업이 `Queue`에 삽입(Active: 20)
   3. `Queue`에 있던 10개의 작업 실행(Active: 30)
   4. 또 10개의 작업이 `Queue`에 삽입(Active: 30)
   5. `Queue`에 있던 10개의 작업 실행(Active: 40)
   6. 또 10개의 작업이 `Queue`에 삽입(Active: 40)
   6. `Queue`에 있던 10개의 작업 실행(Active: 50)
   6. 또 10개의 작업이 `Queue`에 삽입(Active: 50)
   7. Active Thread가 `maxPoolSize`와 같아졌기 때문에 나머지 10개의 `Runnable`은 Queue에서 대기
4. 최초 10개의 작업이 완료되며 `Queue`에 있던 작업 10개가 실행됨(Active: 50). `Queue`는 비게 됨

## 그래서 나는 어떻게 했는가?

`ThreadPoolExecutor`를 사용한 이유는 가용 자원이 한정적이기 때문이었다. 다만, 사용자가 작업 실행 요청을 하면 자원이 없더라도 일단 대기 `Queue`에 넣어 놓고 가용 자원이 확보될 때까지 기다려야 했다.

문제는 사용자의 작업 요청이 몇 건이 들어올 지 모른다는 것이다. 최소 한건 ~ 최대 수십, 수백 건까지 들어올 수 있었다.

만일 위의 예제에서의 60건의 작업이 아닌 100건의 작업이 들어온다고 가정해보자.

```
//기존 60건에서 100건으로 증가. 다른 옵션들은 조정하지 않음
int numTasks = 100;
BlockingQueue<Runnable> blockingQueue = new LinkedBlockingQueue<>(10);
CountDownLatch countDownLatch = new CountDownLatch(numTasks);
ThreadPoolExecutor threadPoolExecutor= new ThreadPoolExecutor(10, 50, 10,
    TimeUnit.SECONDS, blockingQueue);

for(int i = 0; i < numTasks; i++){
    threadPoolExecutor.submit(() -> {
        try {
            Thread.sleep(1000);
        }
        catch(Exception e){
		}
        countDownLatch.countDown();
    });
}
```

위 코드를 실행하면 다음과 같은 오류가 발생한다.

```
Exception in thread "main" java.util.concurrent.RejectedExecutionException: Task java.util.concurrent.FutureTask@574caa3f[Not completed, task = java.util.concurrent.Executors$RunnableAdapter@59690aa4[Wrapped task = com.leeyh0216.jpa.ThreadPoolTest$$Lambda$1/0x0000000800060840@6842775d]] rejected from java.util.concurrent.ThreadPoolExecutor@64cee07[Running, pool size = 50, active threads = 50, queued tasks = 10, completed tasks = 0]
```

Active Thread가 50개, Queue의 갯수가 10개이기 때문에 나머지 40개의 작업은 Queue에 저장할 수 없고, 이에 `submit` 실행 시 `RejectedExecutionException`이 발생하는 것이다.

위의 요구사항은 "한정된 가용자원"을 위해 동시 실행 작업의 갯수를 제한시키고 "사용자의 요청"은 대기하다가 나중에는 꼭 실행시켜야했기 때문에, `Queue`의 크기를 넘는 요청을 받아들일 수 있어야 했다. 즉, 사용자 요청 시 `RejectedExecutionException`이 발생해서는 안되었다.

이에 기존과 같이 `Integer.MAX_VALUE`만큼의 `Queue` 크기를 사용하도록 설정하여 문제를 해결하였다(?? 처음이랑 똑같군... 허탈...).

자세한 상황은 Oracle 문서를 참고하자!!