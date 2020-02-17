---
layout: post
title:  "Spring Sleuth 사용 시 Thread 간 Trace ID가 공유되지 않는 문제 해결하기"
date:   2020-02-17 23:00:00 +0900
author: leeyh0216
tags:
- spring
---

> 두 줄 요약
>
> ApplicationContext와 BeanFactory는 다른 객체이다.
>
> TraceableExecutorService를 초기화할 때는 ApplicationContext가 아니라 BeanFactory를 활용하라.

# Spring Sleuth 사용 시 Thread 간 Trace ID가 공유되지 않는 문제 해결하기

기존에 Spring Sleuth를 사용할 때 `Runnable`을 구현한 클래스에 Zipkin의 `Tracer`를 의존성으로 주입받아 `run()` 메서드 시작 시 Span을 생성하는 로직을 추가하여 사용하였다.

클래스와 전혀 무관한 필드들이 늘어남에 따라, 이를 줄이고자 예제를 찾아보던 중 [baeldung - Spring Cloud Sleuth in a Monolith Application](https://www.baeldung.com/spring-cloud-sleuth-single-application)를 확인하게 되어 구현을 시작했다.

## 사용한 코드

baeldung의 예제를 참고하여 다음과 같이 Configuration, Service, Controller를 작성하였다.

### ExecutorServiceConfiguration 클래스

`@Async` 어노테이션이 붙은 비동기 메서드를 실행할 `ExecutorService`를 초기화하는 설정 클래스이다.

Sleuth에서 제공하는 `TraceableExecutorService`를 반환하도록 한다. 초기화 시 `BeanFactory`가 필요한데, 나는 `ApplicationContextAware` 인터페이스를 구현하여 멤버 필드인 `beanFactory` 객체를 초기화 해주었다(복선).

```
package com.leeyh0216.spring.exercise.springexercise.configuration;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.cloud.sleuth.instrument.async.TraceableExecutorService;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

@EnableAsync
@Configuration
public class ExecutorServiceConfiguration implements ApplicationContextAware {
    private static final Logger logger = LoggerFactory.getLogger(ExecutorServiceConfiguration.class);

    private BeanFactory beanFactory;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.beanFactory = applicationContext;
        logger.info("BeanFactory set in {}", getClass().getSimpleName());
    }

    @Bean("TaskThreadPool")
    public ExecutorService taskThreadPool(){
        ExecutorService executorService = new ThreadPoolExecutor(
                10, 50, 10, TimeUnit.SECONDS, new LinkedBlockingQueue<>());
        return new TraceableExecutorService(beanFactory, executorService);
    }
}
```

### ThreadService 클래스

간단한 비동기 메서드를 구현한 `ThreadService` 클래스이다.

`ExecutorServiceConfiguration` 클래스에서 초기화한 `ExecutorService`를 사용할 수 있도록 `@Async`의 value에 `ExecutorService`의 Bean 이름인 "TaskThreadPool"을 기재한다. 

```
package com.leeyh0216.spring.exercise.springexercise.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

@Service
public class ThreadService {
    private static final Logger logger = LoggerFactory.getLogger(ThreadService.class);

    @Async("TaskThreadPool")
    public void execute(){
        logger.info("hello world");
    }
}
```

### SampleController

간단한 RestController이다.

`hello()` 메서드에서 "hello world"를 로깅하고, `ThreadService`의 `execute()` 메서드를 수행한다.

package com.leeyh0216.spring.exercise.springexercise.controller;

import com.leeyh0216.spring.exercise.springexercise.service.ThreadService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class SampleController {
    private static final Logger logger = LoggerFactory.getLogger(SampleController.class);

    private ThreadService threadService;

    public SampleController(ThreadService threadService){
        this.threadService = threadService;
    }

    @GetMapping("/")
    public ResponseEntity<String> hello(){
        logger.info("hello world");
        threadService.execute();
        return new ResponseEntity<>(HttpStatus.OK);
    }
}

## 실행 결과 예측

`TraceableExecutorService`가 정상적으로 동작했다면, `SampleController`에서 로깅한 "hello world"와 `ThreadService`에서 로깅한 "hello world"의 Trace ID는 같고 Span ID는 다르게 출력되어야 한다.

그러나 실제로 실행된 결과는 다음과 같이 서로 다른 Trace ID를 출력하고 있었다.

```
2020-02-17 23:27:03.997  INFO [,d91b4fa6412259e6,d91b4fa6412259e6,false] 44100 --- [nio-8081-exec-1] c.l.s.e.s.controller.SampleController    : hello world
2020-02-17 23:27:04.023  INFO [,31108241f6f44deb,31108241f6f44deb,false] 44100 --- [pool-1-thread-1] c.l.s.e.s.service.ThreadService          : hello world

```

## 왜 이런 현상이 발생했을까?

`TraceableExecutorService`의 코드는 다음과 같다.

```
public class TraceableExecutorService implements ExecutorService {

	final ExecutorService delegate;

	private final String spanName;

	Tracing tracing;

	SpanNamer spanNamer;

	BeanFactory beanFactory;

	public TraceableExecutorService(BeanFactory beanFactory,
			final ExecutorService delegate) {
		this(beanFactory, delegate, null);
	}

	public TraceableExecutorService(BeanFactory beanFactory,
			final ExecutorService delegate, String spanName) {
		this.delegate = delegate;
		this.beanFactory = beanFactory;
		this.spanName = spanName;
	}

    @Override
	public void execute(Runnable command) {
		this.delegate.execute(ContextUtil.isContextInCreation(this.beanFactory) ? command
				: new TraceRunnable(tracing(), spanNamer(), command, this.spanName));
	}
    ... 생략
}
```

`ExecutorServiceConfiguration`에서 초기화 할 때 넣어준 `ExecutorService` 객체는 "delegate" 필드로 가지고, `BeanFactory` 객체는 "beanFactory" 필드로 가지고 있다.

실제 `Runnable`을 실행하는 `public void execute(Runnable command)` 메서드를 보면, `ContextUtil.isContextInCreation(this.beanFactory)`의 결과가 참일 경우에는 매개변수로 전달받은 `Runnable`을 그대로 실행하고, 해당 결과가 거짓일 경우에만 현재 `Thread`의 Trace ID를 활용하는 `TraceRunnable` 객체를 초기화하여 실행하게 된다.

`ContextUtil` 객체의 `isContextInCreation` 메서드는 다음과 같이 구현되어 있다.

```
static boolean isContextInCreation(BeanFactory beanFactory) {
	boolean contextRefreshed = ContextRefreshedListener.getBean(beanFactory).get();
	if (!contextRefreshed && log.isDebugEnabled()) {
		log.debug("Context is not ready yet");
	}
	return !contextRefreshed;
}
```

`contextRefreshed` 변수가 참일 경우에만 `TraceRunnable`을 초기화하게 된다. 해당 변수는 `ContextRefreshedListener`의 `getBean` 메서드의 결과를 `get()` 한 값을 받아오고 있다.

`ContextRefreshedListener`의 코드는 아래와 같다.

```
/*
 * Copyright 2013-2019 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.cloud.sleuth.instrument.async;

...생략

class ContextRefreshedListener extends AtomicBoolean implements SmartApplicationListener {

	static final Map<BeanFactory, ContextRefreshedListener> CACHE = new ConcurrentHashMap<>();

	private static final Log log = LogFactory.getLog(ContextRefreshedListener.class);

	ContextRefreshedListener(boolean initialValue) {
		super(initialValue);
	}

	ContextRefreshedListener() {
		this(false);
	}

	static ContextRefreshedListener getBean(BeanFactory beanFactory) {
		return CACHE.getOrDefault(beanFactory, new ContextRefreshedListener(false));
	}

	@Override
	public boolean supportsEventType(Class<? extends ApplicationEvent> eventType) {
		return ContextRefreshedEvent.class.isAssignableFrom(eventType);
	}

	@Override
	public void onApplicationEvent(ApplicationEvent event) {
		if (event instanceof ContextRefreshedEvent) {
			if (log.isDebugEnabled()) {
				log.debug("Context successfully refreshed");
			}
			ContextRefreshedEvent contextRefreshedEvent = (ContextRefreshedEvent) event;
			ApplicationContext context = contextRefreshedEvent.getApplicationContext();
			BeanFactory beanFactory = context;
			if (context instanceof ConfigurableApplicationContext) {
				beanFactory = ((ConfigurableApplicationContext) context).getBeanFactory();
			}
			ContextRefreshedListener listener = CACHE.getOrDefault(beanFactory, this);
			listener.set(true);
			CACHE.put(beanFactory, listener);
		}
	}

}

```

이 클래스는 `AtomicBoolean`을 상속해서 구현했는데, Spring Application 실행 시 발생하는 `ApplicationEvent` 이벤트를 수신하여 현재 Application의 `BeanFactory`를 Key, 값이 true인 `ContextRefreshedListener` 객체를 CACHE에 저장해놓는다. 

만일 `getBean` 함수를 호출했는데, Spring Application이 실행될 때 이벤트로 수신한 `BeanFactory`를 사용하여 호출했다면 값이 true인 `ContextRefreshedListener` 객체를 반환할 것이고, 다른 `BeanFactory`를 사용하여 호출했다면 값이 false인 `ContextRefreshedListener` 객체를 반환할 것이다.

즉, 나는 무언가 잘못해서 ApplicationEvent에 설정된 `BeanFactory`가 아닌 다른 `BeanFactory`를 사용했다는 결론이 된다.

## 정답은?

다시 `ExecutorServiceConfiguration` 코드를 보았다.

```
package com.leeyh0216.spring.exercise.springexercise.configuration;

...생략

@EnableAsync
@Configuration
public class ExecutorServiceConfiguration implements ApplicationContextAware {
    private static final Logger logger = LoggerFactory.getLogger(ExecutorServiceConfiguration.class);

    private BeanFactory beanFactory;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.beanFactory = applicationContext;
        logger.info("BeanFactory set in {}", getClass().getSimpleName());
    }

    @Bean("TaskThreadPool")
    public ExecutorService taskThreadPool(){
        ExecutorService executorService = new ThreadPoolExecutor(
                10, 50, 10, TimeUnit.SECONDS, new LinkedBlockingQueue<>());
        return new TraceableExecutorService(beanFactory, executorService);
    }
}
```

전에 Spring을 공부할 때 `ApplicationContext` 클래스는 `BeanFactory` 클래스를 상속했다고 읽었기 때문에 어떤 `BeanFactory` 객체라도 주입받아 초기화할 때 사용하면 된다고 생각했다.

그러나 `ContextRefreshedListener` 에서는 `ApplicationEvent`에 담겨 있는 `BeanFactory`를 기준으로 CACHE를 생성하기 때문에, 해당 `BeanFactory`가 무엇인지 찾아야 했다.

`setApplicationContext`에 Break Point를 걸고 `BeanFactory` 객체를 확인해본 결과 `AnnotationConfigServletWebServerApplicationContext` 클래스의 객체인 것을 확인할 수 있었다.

`ContextRefreshedListener` 클래스의 `onApplicationEvent`에서도 `ApplicationContext`는 위와 동일한 `AnnotationConfigServletWebServerApplicationContext` 클래스의 객체이지만, 바로 아랫 줄의 `BeanFactory`는 다시 이 객체의 `getBeanFactory()` 메서드를 호출하여 할당하고, 이는 `DefaultListableBeanFactory` 객체인 것을 확인하였다.

## 결론은?

`ContextRefreshedListener` 클래스의 코드를 활용하여 `ExecutorServiceConfiguration` 클래스의 `setApplicationContext` 메서드를 아래와 같이 변경하거나, 

```
@Override
public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
    if (applicationContext instanceof ConfigurableApplicationContext) {
        beanFactory = ((ConfigurableApplicationContext) applicationContext).getBeanFactory();
    }
    logger.info("BeanFactory set in {}", getClass().getSimpleName());
}
```

아예 `ApplicationContextAware`가 아닌 `BeanFactoryAware`를 상속하여 아래와 같이 코드를 사용하면 된다.

```
@Override
public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
    this.beanFactory = beanFactory;
}
```