---
layout: post
title:  "Validation에 책임 연쇄 패턴 적용하기"
date:   2019-06-29 22:00:00 +0900
author: leeyh0216
categories: spring
---

# Validation에 책임 연쇄 패턴 적용하기

데이터를 저장하기 전에 데이터에 대한 검증(Validation)을 수행해야 하는 경우가 있다.

예전에는 아래와 같이 데이터를 관리하는 클래스 내에 validation 이라는 메서드를 정의해서 기본 validation을 수행하고, 필요한 경우 해당 메서드를 재정의하여 사용하거나 preValidate, postValidate와 같은 추상 메서드를 만들어놓고 하위 클래스에서 이를 구현하면, validate 함수 내에서 이를 호출해주는 방식으로 구현했었다.

**OldValidationExample.java**
{% highlight java %}
package com.leeyh0216.spring.examples.validation.plain;

import com.leeyh0216.spring.examples.validation.Metadata;

public class OldValidationExample {

    public static abstract class AbstractMetadataManager {

        public void validate(Metadata metadata) {
            if (metadata.getId() == null || metadata.getId().isEmpty())
                throw new IllegalArgumentException("ID must not be null or empty string");

            //하위 클래스에서 작성한 postValidate를 호출해준다.
            postValidate(metadata);
        }

        //자식 클래스들이 Metadata에 대한 Validation을 수행할 수 있도록 한다.
        protected abstract void postValidate(Metadata metadata);
    }

    public static class SampleMetadataManager extends AbstractMetadataManager {

        @Override
        public void postValidate(Metadata metadata) {
            if (metadata.getOwner() == null || metadata.getOwner().isEmpty())
                throw new IllegalArgumentException("Owner must not be null or empty string");
        }
    }

    public static void main(String[] args) {
        Metadata m1 = new Metadata();
        m1.setId("hello");
        m1.setOwner("world");

        SampleMetadataManager metadataManager = new SampleMetadataManager();
        metadataManager.validate(m1);

        m1.setOwner(null);
        metadataManager.validate(m1);
    }
}
{% endhighlight %}

이러한 방식으로 구현하다보니 아래와 같은 문제점이 있었다.

* Validation이 늘어날 수록 preValidate, postValidate 메서드의 코드 양이 많아진다.
* validate 호출 순서를 바꾸기가 어렵다.
* 재정의를 하다보면 반드시 호출되어야 하는 함수를 재정의한 메서드에서 호출하지 않는 경우가 발생한다.

어떻게 해결할까 하다가 Chain of responsibility 패턴을 이용하여 코드를 수정해보았다.

## 책임 연쇄(Chain of responsibility) 패턴

객체 지향 디자인에서 chain-of-responsibility pattern은 명령 객체와 일련의 처리 객체를 포함하는 디자인 패턴이다. 각각의 처리 객체는 명령 객체를 처리할 수 있는 연산의 집합이고, 체인 안의 처리 객체가 핸들할 수 없는 명령은 다음 처리 객체로 넘겨진다. 이 작동방식은 새로운 처리 객체부터 체인의 끝까지 다시 반복된다.

> 출처: [위키 백과: 책임 연쇄 패턴](https://ko.wikipedia.org/wiki/%EC%B1%85%EC%9E%84_%EC%97%B0%EC%87%84_%ED%8C%A8%ED%84%B4)

## Validation에 적용해보기

일반적인 책임 연쇄 패턴은 Handler 들이 Linked List 형식으로 구현되어 있고, 다음 Handler를 호출할 지 안할지를 이전 Handler에서 정할 수 있다.

그러나 Validation의 경우 현재 Handler에서 데이터에 대한 이상이 검출되는 경우 IllegalArgumentException을 호출하므로써 Chain의 진행을 끊을 수 있기 때문에 그냥 List에 Handler를 넣고 순차적으로 호출하는 방식으로 구현하였다.

**NewValidationExample.java**
{% highlight java %}
package com.leeyh0216.spring.examples.validation.plain;

import com.leeyh0216.spring.examples.validation.Metadata;

import java.util.ArrayList;
import java.util.List;

public class NewValidationExample {

    public interface IValidationHandler {

        void validate(Metadata metadata);

    }

    public static class HandlerEventChain {
        private List<IValidationHandler> handlerList = new ArrayList<>();

        public void addHandler(IValidationHandler handler) {
            handlerList.add(handler);
        }

        public void validate(Metadata metadata) {
            for (IValidationHandler handler : handlerList)
                handler.validate(metadata);
        }
    }

    public static class IdValidationHandler implements IValidationHandler {
        @Override
        public void validate(Metadata metadata) {
            if (metadata.getId() == null || metadata.getId().isEmpty())
                throw new IllegalArgumentException("ID must not be null or empty string");
        }
    }

    public static class OwnerValidationHandler implements IValidationHandler {
        @Override
        public void validate(Metadata metadata) {
            if (metadata.getOwner() == null || metadata.getOwner().isEmpty())
                throw new IllegalArgumentException("Owner must not be null or empty string");
        }
    }

    public static void main(String[] args) {
        Metadata m1 = new Metadata();
        m1.setId("hello");
        m1.setOwner("world");

        HandlerEventChain eventChain = new HandlerEventChain();
        eventChain.addHandler(new IdValidationHandler());
        eventChain.addHandler(new OwnerValidationHandler());

        eventChain.validate(m1);

        m1.setOwner(null);

        eventChain.validate(m1);
    }
}
{% endhighlight %}

## Spring에서 사용해보기

Spring에서는 `@Component`, `@Bean`, `@Service` 어노테이션 등으로 선언된 Handler들을 List 형태로 받아와서 EventChain Bean을 초기화해주면 된다.

`@Order` 어노테이션을 사용하면 검증 순서를 손쉽게 변경할 수 있다.

**SpringValidationExample.java**
{% highlight java %}
package com.leeyh0216.spring.examples.validation.spring;

import com.leeyh0216.spring.examples.validation.Metadata;
import com.leeyh0216.spring.examples.validation.plain.NewValidationExample;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Configuration
@SpringBootApplication
@ComponentScan("com.leeyh0216.spring.examples.validation.spring")
public class SpringValidationExample {

    @Bean
    public NewValidationExample.IdValidationHandler idValidationHandler() {
        return new NewValidationExample.IdValidationHandler();
    }

    @Bean
    public NewValidationExample.OwnerValidationHandler ownerValidationHandler() {
        return new NewValidationExample.OwnerValidationHandler();
    }

    @Bean
    public NewValidationExample.HandlerEventChain handlerEventChain(List<NewValidationExample.IValidationHandler> validationHandlers) {
        NewValidationExample.HandlerEventChain eventChain = new NewValidationExample.HandlerEventChain();
        for (NewValidationExample.IValidationHandler handler : validationHandlers)
            eventChain.addHandler(handler);

        return eventChain;
    }

    public static void main(String[] args) {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(SpringValidationExample.class);
        NewValidationExample.HandlerEventChain handlerEventChain = applicationContext.getBean(NewValidationExample.HandlerEventChain.class);

        Metadata m1 = new Metadata();
        m1.setId("hello");
        m1.setOwner("world");

        handlerEventChain.validate(m1);

        m1.setOwner(null);

        handlerEventChain.validate(m1);
    }
}
{% endhighlight %}