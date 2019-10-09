---
layout: post
title:  "Spring Core Technologies - Bean Scopes"
date:   2019-01-22 10:00:00 +0900
author: leeyh0216
tags:
- spring
---

# Bean Scopes

Bean Definition을 만든다는 것은, Bean으로 생성할 클래스를 통해 어떻게 객체를 만들어 내는지에 대한 방법(Recipe)을 만들어 내는 것이다.

Bean Definition에는 생성할 Bean의

* 의존성(Dependency)
* 설정값(Configuration values)
* **Scope**

이 포함된다.

설정을 통해 객체의 Scope을 지정하는 방식은 자바의 클래스 레벨에서 Scope을 제어하는 것보다 강력하고 유연하다. 

Spring에서는 7개의 Scope을 지원하며, Non Web Application에서는 이 중 2개만 사용이 가능하다.

## Singleton Scope

Spring의 기본 Scope. Spring IoC Container 당 1개의 객체를 유지한다.

![Singleton Scope](/assets/spring/singletonscope.jpg)

Spring의 Singleton Bean의 개념은 GoF 패턴에서 나오는 Singleton Pattern과 차이가 있다.

GoF 패턴에서 나오는 Singleton Pattern이 적용된 클래스는 Java ClassLoader에 단 1개의 객체밖에 존재할 수 없지만, Spring에서의 Singleton Bean은 Spring IoC Container에서만 1개의 객체를 유지한다. 즉, 임의로 Singleton Bean을 만들어낼 수 있다.

### 일반적인 Singleton Pattern

{% highlight java %}
package com.leeyh0216.others;

//Final 클래스로 만들어 상속이 불가하게 함
public final class SingletonExample {

    //JVM 내에서 1개만 유지되는 SingletonExample 객체
    private static SingletonExample instance = null;

    //synchronized 키워드를 통해 Thread-Safe 보장
    private static synchronized SingletonExample getInstance(){
        if(instance == null)
            instance = new SingletonExample();
        return instance;
    }

    private SingletonExample(){
        //Do something
    }

    public static void main(String[] args) throws Exception {
        SingletonExample s1 = SingletonExample.getInstance();
        SingletonExample s2 = SingletonExample.getInstance();

        System.out.println(s1 == s2);
    }
}
{% endhighlight %}

위와 같이 생성자를 `private`으로 선언하여 `new`를 통한 객체 생성을 막고, `getInstance` 함수를 통해서만 객체를 생성/참조할 수 있도록 하여 JVM 내에 1개의 객체만을 유지할 수 있도록 한다.

Reflection을 사용하지 않고서는 일반적인 방법으로 해당 클래스의 객체를 2개 유지하는 것은 불가능하다.

위 프로그램의 결과는 `true`가 나오게 된다.

### Spring에서의 Singleton Scope

테스트를 위해 3개의 파일을 작성한다.

* Program Entry Point 역할을 담당하는 Application 클래스
* Configuration 역할을 담당하는 AppConfig 클래스
* 테스트 클래스인 MyService

#### MyService.java

{% highlight java %}
package com.leeyh0216.springstudy.singletonscope;

public class MyService{

    public MyService() {
        //Do something
    }

}
{% endhighlight %}

아무 기능도 없이 기본 생성자만 존재하는 클래스이다.

#### AppConfig.java

{% highlight java %}
package com.leeyh0216.springstudy.singletonscope;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan("com.leeyh0216.springstudy.singletonscope")
public class AppConfig {

    @Bean
    public MyService getMyService(){
        return new MyService();
    }
}
{% endhighlight %}

MyService 타입의 Bean을 반환하는 `getMyService` 함수가 정의된 Configuration 클래스이다.

#### Application.java

{% highlight java %}
package com.leeyh0216.springstudy.singletonscope;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Application {

    public static void main(String[] args) throws Exception {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        MyService m1 = applicationContext.getBean(MyService.class);
        MyService m2 = applicationContext.getBean(MyService.class);
        MyService m3 = new MyService();

        System.out.println(m1 == m2);
        System.out.println(m2 == m3);
        System.out.println(m1 == m3);
    }
}
{% endhighlight %}

메인 함수가 들어 있는 Application 클래스이다.

위 프로그램의 출력은 아래와 같다.

{% highlight bash %}
true
false
false
{% endhighlight %}

위의 `m1`과 `m2` 객체는 Spring IoC Container에서 관리하는 Bean 객체이다. Spring에서의 기본 Scope은 Singleton이라 했기 때문에 `m1`과 `m2` 객체는 완전히 같은 객체이다. 그렇기 때문에 첫번째 출력은 `true`가 된다.

그런데 `m3` 객체는 MyService의 생성자를 직접 호출하여 생성한 객체이다. 즉, 이 객체는 Spring의 IoC Container의 관리를 받지 않는 객체이며, 기존에 생성된 `m1`, `m2` 객체와는 완전히 다른 객체이다. 그렇기 때문에 2,3번째 출력은 `false`가 되는 것이다.

## Prototype Scope

Prototype Bean은 Bean을 참조하는 요청(`getBean`과 같은 함수를 호출할 때)을 할 때마다 새로운 객체가 생성된다.

Prototype Scope을 가진 Bean은 Stateful한 Bean이 필요할 때 사용하고, Singleton Scope을 가진 Bean은 Stateless한 Bean이 필요할 때 사용하면 된다.

![Prototype Scope](/assets/spring/prototypescope.jpg)

다른 Scope과 다르게 Prototype Scope을 가진 Bean의 Life Cycle은 일부만 관리된다. Spring은 Prototype Bean을 생성하여 Client에게 넘겨주지만, 해당 객체를 기록(Record라고 나와 있는데, Container가 별도로 해당 Bean에 대한 참조를 가지고 있지 않다는 것을 의미하는 것 같다.)하고 있지 않다. Prototype Scope을 가진 Bean의 initialization 관련 Callback들은 모두 호출되지만, Destruction 관련 Callback은 호출되지 않기 때문에, 해당 Bean이 비싼 자원(ex. socket, file)을 가지고 있다면 클라이언트 코드에서 별도로 해당 자원을 소멸시켜야 한다.

### Prototype Scope 테스트 코드

위의 Spring Singleton 과 동일한 코드이지만, AppConfig 클래스의 `getMyService` 메서드에 `@Scope("prototype")` 어노테이션이 추가된 점만 다르다.

#### AppConfig.java

{% highlight java %}
package com.leeyh0216.springstudy.prototypescope;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;

@Configuration
@ComponentScan("com.leeyh0216.springstudy.prototypescope")
public class AppConfig {

    @Bean
    @Scope("prototype")
    public MyService getMyService(){
        return new MyService();
    }
}
{% endhighlight %}

위에서는 명시적으로 `@Scope` 어노테이션을 통해 prototype Scope을 지정하였으나, 별도로 명시하지 않는 경우 `@Scope("singleton")`과 동일한 효과를 가지게 된다.

위 코드를 기준으로 Application을 실행하였을 때 아래와 같은 결과가 발생한다.

{% highlight bash %}
false
false
false
{% endhighlight %}

Singleton과 다르게 Spring IoC Container에서 `getBean` 함수를 호출할 때마다 MyService의 객체를 새로 생성하여 반환하기 때문에, 모든 객체가 다를 수밖에 없다.

### Lifecycle Callback 비교하기(Singleton vs Prototype)

Singleton에서는 모든 Lifecycle Callback이 동작하고 Prototype에서는 Initialization 관련 Callback만 동작한다고 나와 있다.

아래와 같이 테스트를 수행해보았다.

#### SingletonService.java

{% highlight java %}
package com.leeyh0216.springstudy.singletonvsprototype;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;

public class SingletonService {

    public SingletonService() {
        //Do something
    }

    @PostConstruct
    public void onCreate(){
        System.out.println("Singleton has created");
    }

    @PreDestroy
    public void onDestroy(){
        System.out.println("Singleton is destroying");
    }
}
{% endhighlight %}

#### PrototypeService.java

{% highlight java %}
package com.leeyh0216.springstudy.singletonvsprototype;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;

public class PrototypeService {

    public PrototypeService() {
        //Do something
    }

    @PostConstruct
    public void onCreate(){
        System.out.println("Prototype has created");
    }

    @PreDestroy
    public void onDestroy(){
        System.out.println("Prototype is destroying");
    }
}
{% endhighlight %}

#### AppConfig.java

{% highlight java %}
package com.leeyh0216.springstudy.singletonvsprototype;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;

@Configuration
@ComponentScan("com.leeyh0216.springstudy.singletonvsprototype")
public class AppConfig {

    @Bean
    @Scope("singleton")
    public SingletonService getSingletonService(){
        return new SingletonService();
    }

    @Bean
    @Scope("prototype")
    public PrototypeService getPrototypeService() { return new PrototypeService(); }
}
{% endhighlight %}

#### Application.java

{% highlight java %}
package com.leeyh0216.springstudy.singletonvsprototype;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Application {

    public static void main(String[] args) throws Exception {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        SingletonService singletonService = applicationContext.getBean(SingletonService.class);
        PrototypeService prototypeService1 = applicationContext.getBean(PrototypeService.class);
        PrototypeService prototypeService2 = applicationContext.getBean(PrototypeService.class);
        ((AnnotationConfigApplicationContext) applicationContext).registerShutdownHook();
    }
}
{% endhighlight %}

위 프로그램의 출력은 아래와 같다.

{% highlight bash %}
Singleton has created
Prototype has created
Prototype has created
Singleton is destroying
{% endhighlight %}

위와 같이 Prototype Bean에 대해서는 "Prototype is destroying"이라는 문구가 출력되지 않은 것을 볼 수 있다.