---
layout: post
title:  "Spring Core Technologies - The IoC Container(3)"
date:   2018-12-03 10:00:00 +0900
author: leeyh0216
categories: spring
---

# The IoC Container

## Introduction to the Spring IoC Container and Beans

### Bean Dependencies

Application에 Service Layer 역할을 하는 MyService와 Persistent Layer 역할을 하는 MyRepository 클래스가 있다고 가정해보자.

MyService 클래스가 동작하기 위해서는 MyRepository 객체가 필요하다.(즉, MyService 클래스가 MyRepository 클래스에 의존적이다)

이러한 경우 아래와 같은 코드를 작성해야 할까?

{% highlight java %}
package com.leeyh0216.springstudy.beandependencies;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    @Bean
    public MyService getMyService(){
        return new MyService(new MyRepository());
    }

}
{% endhighlight %}

`getMyService` 함수에서 MyService 객체 초기화 시, MyRepository 객체 또한 같이 초기화하여 생성자로 전달하고 있다. 이러한 방식을 사용할 경우, MyService의 의존 객체가 많을 경우 `getMyService` 함수 또한 비대해진다.

ApplicationContext는 모든 Bean을 관리하고, @Bean 어노테이션이 붙은 함수의 인자와 일치하는 Bean을 찾아 주입시켜주는 기능을 가지고 있다.

즉, 아래와 같이 MyRepository Bean을 초기화하는 @Bean 어노테이션이 붙은 메소드를 만들고, `getMyService` 함수의 인자로는 MyService를 초기화하는데 필요한 MyRepository를 선언한다. 그러면 Spring의 ApplicationContext는 `MyRepository` Bean을 먼저 getMyRepository 함수를 호출하여 초기화하고, `MyService` Bean 초기화 함수인 `getMyService`의 인수로 이미 Bean으로 생성되어 있는 `MyRepository` 객체를 주입시켜준다.

{% highlight java %}
package com.leeyh0216.springstudy.beandependencies;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    @Bean
    public MyRepository getMyRepository(){ return new MyRepository(); }

    @Bean
    public MyService getMyService(MyRepository myRepository){
        return new MyService(myRepository);
    }

}
{% endhighlight %}

### Receiving Lifecycle Callbacks

@Bean 어노테이션으로 초기화되는 객체들은 JSR-250 스펙에 정의된 @PostConstruct와 @PreDestory 어노테이션을 통한 Lifecycle Callback을 호출받을 수 있다.

또한 Spring Framework의 `InitializingBean`, `DisposableBean`, `Lifecycle`을 상속받으면 Container에 의해 Lifecycle을 관리받을 수 있다.

#### InitializingBean을 상속하는 방법

`InitializingBean`는 Bean이 `BeanFactory`에 의해 생성되고, 모든 의존성이 주입되어졌을 때 한번 호출되는 함수를 필요로 할 때 구현하는 인터페이스이다.

그렇다면 생성자와 다른 점은 무엇인가?

객체는 생성자 호출 시 필요한 의존 객체를 주입받을 수도 있지만, 생성된 이후에도 Setter를 통해 의존 객체를 주입받을 수 있다. `InitializingBean`은 이렇게 객체 생성 이후에 Setter를 통해 의존 객체를 주입받는 경우 구현해야하는 인터페이스이다.

`InitializingBean` 인터페이스를 상속받은 클래스는 `void afterPropertiesSet()` 메서드를 구현해야 한다. 이 함수는 Bean의 의존 객체가 모두 설정된 후 호출된다.

다루지는 않았었지만 `@Autowired` 어노테이션을 Bean으로 만들 클래스의 Setter에 설정해주면, Bean 생성 시 Setter의 인자와 동일한 객체가 있는 경우 BeanFactory가 주입해준다.

아래와 같은 MyService 클래스가 있다고 생각해보자.

{% highlight java %}
package com.leeyh0216.springstudy.initializingbean;

import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Autowired;

public class MyService implements InitializingBean {

    private static final String SERVICE_NAME = "MY_SERVICE";

    private MyRepository myRepository;

    public MyService(){
        System.out.println("MyService Constructor Called");
    }

    @Autowired
    public void setMyRepository(MyRepository myRepository){
        System.out.println("setMyRepository Called");
        this.myRepository = myRepository;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("afterPropertiesSet Called");
    }

    public void printServiceName(){
        System.out.println("My Service: " + SERVICE_NAME);
    }
}
{% endhighlight %}

Spring Application을 호출하면 위 객체의 메서드(혹은 생성자)가 아래와 같은 순서로 호출된다.

1. `MyService` 클래스의 생성자
2. `setMyRepository` 메서드
3. `afterPropertiesSet` 메서드

즉, `InitializingBean` 인터페이스의 `afterPropertiesSet()` 함수는 생성자 뿐만 아니라, Bean의 의존 객체를 BeanFactory가 모두 Injection 해준 후 호출되는 함수이다.

#### `@PostConstruct` 어노테이션을 붙인 메서드를 만드는 방법

Bean 클래스 내에 @PostConstruct 어노테이션을 붙인 메서드를 만드는 경우, BeanFactory가 Bean을 생성한 후 해당 함수를 호출하게 된다.

{% highlight java %}
package com.leeyh0216.springstudy.initializingbean;

import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Autowired;

import javax.annotation.PostConstruct;

public class MyService {

    private static final String SERVICE_NAME = "MY_SERVICE";

    private MyRepository myRepository;

    public MyService(){
        System.out.println("MyService Constructor Called");
    }

    @Autowired
    public void setMyRepository(MyRepository myRepository){
        System.out.println("setMyRepository Called");
        this.myRepository = myRepository;
    }

    @PostConstruct
    public void postConstruct(){
        System.out.println("postConstruct Called");
    }

    public void printServiceName(){
        System.out.println("My Service: " + SERVICE_NAME);
    }
}
{% endhighlight %}

위와 같이 `@PostConstruct` 어노테이션을 함수 위에 붙이는 경우, 아래와 같은 순서로 호출이 진행된다.

1. `MyService` 클래스의 생성자
2. `setMyRepository` 메서드
3. `postConstruct` 메서드

그렇다면 `@PostConstruct` 어노테이션이 붙은 메서드에 인자를 추가할 수 있을까? 아래와 같이 `@PostConstruct` 어노테이션이 붙은 메서드에 인자를 추가해보았다.

{% highlight java %}
@PostConstruct
public void postConstruct(String author){
    System.out.println("postConstruct Called");
}
{% endhighlight %}

호출해보면 아래와 같은 오류가 발생한다.
{% highlight text %}
Exception in thread "main" org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'getMyService' defined in com.leeyh0216.springstudy.initializingbean.AppConfig: Post-processing of merged bean definition failed; nested exception is java.lang.IllegalStateException: Lifecycle method annotation requires a no-arg method: public void com.leeyh0216.springstudy.initializingbean.MyService.postConstruct(java.lang.String)
{% endhighlight %}

*Lifecycle method annotation requires a no-arg method*. 즉, Lifecycle 관련 Annotation이 붙은 메서드는 Argument를 가질 수 없다는 의미이다.

#### `@PostConstruct` vs `afterPropertiesSet`

그렇다면 `@PostConstruct`와 `afterPropertiesSet` 둘 모두를 구현하는 경우 어떻게 동작할까?

아래와 같이 MyService 클래스를 만든 후, Application을 동작시켜보았다.
{% highlight java %}
package com.leeyh0216.springstudy.initializingbean;

import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Autowired;

import javax.annotation.PostConstruct;

public class MyService implements InitializingBean {

    private static final String SERVICE_NAME = "MY_SERVICE";

    private MyRepository myRepository;

    public MyService(){
        System.out.println("MyService Constructor Called");
    }

    @Autowired
    public void setMyRepository(MyRepository myRepository){
        System.out.println("setMyRepository Called");
        this.myRepository = myRepository;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("afterPropertiesSet Called");
    }

    @PostConstruct
    public void postConstruct(){
        System.out.println("postConstruct Called");
    }

    public void printServiceName(){
        System.out.println("My Service: " + SERVICE_NAME);
    }
}
{% endhighlight %}

호출 결과 아래와 같은 로그가 발생하였다.

{% highlight text %}
MyService Constructor Called
setMyRepository Called
postConstruct Called
afterPropertiesSet Called
{% endhighlight %}

즉, `@PostConstruct` 어노테이션이 붙은 메서드가 `afterPropertiesSet` 메서드보다 우선순위가 높은 것을 확인할 수 있다.

#### `DisposableBean` 인터페이스를 상속하는 방법

Bean이 파괴될 때(대부분 Application이 종료될 때) 호출되는 메소드를 구현할 수 있는 인터페이스이다. `void destroy()` 메소드를 구현해야 한다.

아래와 같이 `DisposableBean` 인터페이스를 구현한 MyService 클래스를 구현해보았다.
{% highlight java %}
package com.leeyh0216.springstudy.initializingbean;

import org.springframework.beans.factory.DisposableBean;

public class MyService implements DisposableBean {

    private static final String SERVICE_NAME = "MY_SERVICE";

    private MyRepository myRepository;

    public MyService(){
        System.out.println("MyService Constructor Called");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("destroy Called");
    }

    public void printServiceName(){
        System.out.println("My Service: " + SERVICE_NAME);
    }
}
{% endhighlight %}

이대로 실행시켜보면 "destroyCalled" 라는 메시지가 남지 않고 Application이 종료되는 것을 확인할 수 있다.

ApplicationContext가 종료될 때 Bean들의 destroy 메소드를 호출하는데, ApplicationContext가 종료되는 시점을 인지시키기 위해서는, AbstractApplicationContext의 `registerShutdownHook()` 메서드를 호출해주어야 한다.

즉, 변경된 Application 클래스 코드는 아래와 같다.

{% highlight java %}
package com.leeyh0216.springstudy.initializingbean;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.support.AbstractApplicationContext;

public class Application {


    public static void main(String[] args) throws Exception {
        ApplicationContext applicationContext= new AnnotationConfigApplicationContext(AppConfig.class);
        MyService myService = applicationContext.getBean(MyService.class);
        myService.printServiceName();

        ((AbstractApplicationContext)applicationContext).registerShutdownHook();
    }
}
{% endhighlight %}

위와 같이 코드를 변경하면 destroy 메서드가 정상적으로 호출되는 것을 확인할 수 있다.

#### `@PreDestroy` 어노테이션을 붙인 메서드를 만드는 방법

Bean 클래스 내에 `@PreDestroy` 어노테이션을 붙인 메서드를 만드는 경우, BeanFactory가 객체를 파괴하기 전에(혹은 객체가 파괴되기 전에) `@PreDestory` 어노테이션을 붙인 함수를 호출하게 된다.

아래와 같이 MyService에 `@PreDestory` 어노테이션이 붙은 메서드를 구현해보자.

{% highlight java %}
package com.leeyh0216.springstudy.initializingbean;

import org.springframework.beans.factory.DisposableBean;

import javax.annotation.PreDestroy;

public class MyService{

    private static final String SERVICE_NAME = "MY_SERVICE";

    private MyRepository myRepository;

    public MyService(){
        System.out.println("MyService Constructor Called");
    }

    @PreDestroy
    public void destroy(){
        System.out.println("destroy Called");
    }

    public void printServiceName(){
        System.out.println("My Service: " + SERVICE_NAME);
    }
}
{% endhighlight %}

Application이 종료되기 전에 "destroy Called" 메시지가 발생하는 것을 볼 수 있다.

#### `@PreDestory` vs `destroy()`

그렇다면 `@PostConstruct`와 `afterPropertiesSet()` 과 같이 우선순위가 존재할까? 아래와 같이 코드를 작성한 후 실행해보았다.

{% highlight java %}
package com.leeyh0216.springstudy.initializingbean;

import org.springframework.beans.factory.DisposableBean;

import javax.annotation.PreDestroy;

public class MyService implements DisposableBean{

    private static final String SERVICE_NAME = "MY_SERVICE";

    private MyRepository myRepository;

    public MyService(){
        System.out.println("MyService Constructor Called");
    }

    @Override
    public void destroy(){
        System.out.println("destroy Called");
    }

    @PreDestroy
    public void preDestory(){
        System.out.println("preDestroy Called");
    }

    public void printServiceName(){
        System.out.println("My Service: " + SERVICE_NAME);
    }
}
{% endhighlight %}

호출 결과 아래와 같은 로그가 발생하였다.
{% highlight java %}
preDestory Called
destroy Called
{% highlight java %}

즉, `@PreDestory` 어노테이션이 `destroy()` 메서드보다 높은 우선순위를 갖는 것을 볼 수 있다.

**결론적으로 Lifecycle 메서드들은 인터페이스보다 어노테이션 붙은 메서드의 호출 우선순위가 더 높은 것을 확인할 수 있었다.**