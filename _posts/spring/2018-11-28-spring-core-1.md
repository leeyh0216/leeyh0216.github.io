---
layout: post
title:  "Spring Core Technologies - The IoC Container(1)"
date:   2018-11-28 10:00:00 +0900
author: leeyh0216
categories: spring
---

# The IoC Container

## Introduction to the Spring IoC Container and Beans

Inversion of Control(IoC, 제어의 역전)은 Dependency Injection(DI, 의존성 주입)로도 알려져 있다. IoC가 동작하는 방식은 아래와 같다.

1. 객체는 자신이 동작하는데에 필요한 의존 객체들을 생성자 혹은 생성 이후에 설정할 수 있도록 **생성자의 인자**, **팩토리 메서드의 인자** 또는 **프로퍼티**에 정의한다.

2. Container는 Bean(빈)를 생성할 때, 생성하는 빈이 의존하는 객체들을 해당 Bean이 정의한 방식을 통해 주입(Inject)한다.

즉, Bean이 스스로의 초기화나 의존 객체 관리를 하는 것이 아니라 Container가 제어권을 얻어 해당 절차를 수행하는 것이다.

> **Bean** 이란? Spring에서 Bean은 Application을 구성하는 객체로써, IoC Container에 의해 관리(객체의 생성이나 의존 객체 구성 등)된다.

Spring에서 이러한 IoC Container의 역할을 수행하는 패키지가 `org.springframework.beans` 와 `org.springframework.context` 이다.

* `BeanFactory`: 객체의 관리를 수행하는 기능을 제공한다.(기본 기능만 제공)
* `ApplicationContext`: `BeanFactory`를 상속받아 아래와 같은 추가 기능을 제공한다.(Enterprise 수준의 기능 제공)
  * Spring AOP
  * Message resource handling
  * Event publication
  * Application-layer specific contexts

## Container Overview

`org.springframework.context.ApplicationContext` 는 Spring의 IoC Container를 표현하는 Interface이며, Configuration 메타데이터를 이용하여 Bean들을 초기화, 구성, 조합하는 역할을 담당한다.

Configurtion Metadata는

* XML
* Java Annotation
* Java Code

로 표현할 수 있다.

![Spring IoC Container](/assets/spring/spring_ioc_container.jpg)

위 그림과 같이 사용자는 비즈니스 로직이 포함된 POJO(Plain Old Java Object)와 POJO 들을 초기화, 조합하는 방식을 기술한 Configuration 메타데이터만을 작성하면, Spring Container는 이러한 내용을 조합하여 System 을 구성하게 된다.

## Bean Overview

Spring IoC Container는 Bean을 내부적으로 `BeanDefinition` 객체로 관리한다. `BeanDefinition` 객체에는 다음과 같은 메타데이터들이 포함되어 있다:

* A package-qualified class name: Bean 의 구현이 정의된 클래스 정보(즉, Bean 의 클래스 정보)

* Scope, life cycle callbacks 등 Bean 의 동작 구성

* Bean이 동작하기 위해 참조하는 다른 Bean 들. Collaborators 혹은 Dependencies 라고도 불리운다.

## Java-based Container Configuration

> 위의 내용까지는 Springframework 의 1. IoC Container에 있는 내용이며, 이후 내용은 XML 기반으로 설명되어 있어, Java Based Container Configuration이 기술된 1.12 장으로 넘어오게 되었다.

### Basic Concepts: `@Bean` and `@Configuration`

* `@Bean` : Spring IoC Container에 의해 관리(Instantiate, Configure, Initialize)될 객체를 표현하는 함수에 사용된다.

* `@Congiruation` : Bean 정의가 모여있는 클래스의 어노테이션으로 사용된다. 즉, Configuration 메타데이터 역할을 담당한다.

### Instantiating the Spring Container by Using `AnnotationConfigApplicationContext`

우선 비즈니스 로직이 담긴 MyService 클래스(POJO)를 선언한다.

{% highlight java %}
package com.leeyh0216.springstudy.basic;

public class MyService {

    private static final String SERVICE_NAME = "MY_SERVICE";

    public MyService(){

    }

    public void printServiceName(){
        System.out.println("My Service: " + SERVICE_NAME);
    }
}

{% endhighlight %}

위의 MyService 클래스를 초기화하여 Bean으로 만드는 메서드를 포함한 Configuration 클래스를 만든다.

{% highlight java %}
package com.leeyh0216.springstudy.basic;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    @Bean
    public MyService getMyService(){
        return new MyService();
    }

}
{% endhighlight %}

`@Bean` 어노테이션을 붙인 메서드는 MyService 클래스의 객체를 생성하여 반환하는 메서드이다.

그리고 마지막으로 main 함수를 가진 Application 클래스를 만드는데, main 함수에서는 Spring의 IoC Container를 생성하는데, 우리가 만든 Configuration 메타데이터 클래스인 AppConfig를 이용하여 초기화를 진행할 것이다.

{% highlight java %}
package com.leeyh0216.springstudy.basic;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Application {

    public static void main(String[] args) throws Exception {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        MyService myService = applicationContext.getBean(MyService.class);
        myService.printServiceName();
    }
}

{% endhighlight %}

위 Application 을 실행하면, 아래와 같은 로그가 찍히면서 프로그램이 실행된 것을 확인할 수 있다.

{% highlight text %}
20:59:23.234 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemProperties] PropertySource with lowest search precedence
20:59:23.241 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemEnvironment] PropertySource with lowest search precedence
20:59:23.675 [main] DEBUG org.springframework.context.annotation.AnnotationConfigApplicationContext - Unable to locate ApplicationEventMulticaster with name 'applicationEventMulticaster': using default [org.springframework.context.event.SimpleApplicationEventMulticaster@4bb4de6a]
...생략
20:59:23.816 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Returning cached instance of singleton bean 'getMyService'
My Service: MY_SERVICE

{% endhighlight %}

main 함수에서는 MyService 객체를 초기화하는 부분이 전혀 없지만, Spring의 ApplicationContext 객체는 우리가 제공한 Configuration Metadata를 이용하여 MyService 객체를 초기화 한 것을 확인할 수 있다.

> 여러 개의 Configuration Metadata를 등록하는 방법(register, refresh 함수 사용)도 있지만, 이러한 방식은 잘 사용하지 않기 때문에 정리하지 않도록 한다.

> 예제에서 ApplicationContext의 `T getBean(Class<T> requiredType)` 을 이용하여 메인 함수 내에서 Bean을 가져오는 방식을 사용하고 있는데, 1.2.3. Using the Container 에는 아래와 같은 내용이 나온다.

>> `getBean` 메소드를 이용하여 초기화된 Bean 객체를 가져올 수 있지만, 이상적으로는 어플리케이션의 코드에서는 `getBean`과 같은 Spring API를 호출하여 사용해서는 안된다.

> 완전히 제어 흐름을 Spring IoC Container에게 맡겨야 한다는 의미인 것이고, 회사 업무를 진행하며 느꼇던 부분은 적어도 Web Framework에서의 Spring Project에서는 ApplicationContext 자체를 써본 일이 없는 것 같다.

### Enabling Component Scanning with `scan(String...)`

> `@Configuration` 어노테이션이 붙은 클래스 내에 `@Bean` 어노테이션을 붙인 함수를 통해 Bean을 생성하는 방법 이외에도 여러가지 초기화 방식이 존재한다. 그 중 하나인 `@Configuration`, `@ComponentScan`, `@Component`를 이용하는 방식을 알아본다.

ApplicationContext는 `@Configuration` 을 통한 Bean 제공 이외에도 `@Component` 어노테이션이 붙은(혹은 JSR-330 스펙이 구현된) 클래스를 초기화할 수도 있다.

`@Configuration` 클래스에 `@ComponentScan` 어노테이션을 추가하여 `@Component` 어노테이션이 붙은 클래스가 속한 패키지를 명시해주면 `@Bean` 어노테이션과 메서드를 이용하지 않고도 Bean 초기화가 가능하다.

위의 MyService와 유사하게 MyComponentService 클래스를 만든다. 단, 위와 달리 이 클래스에는 `@Component` 라는 어노테이션을 붙였다. `@Component` 어노테이션은 이 클래스를 Bean으로 생성한다는 의미를 가진다.

{% highlight java %}
package com.leeyh0216.springstudy.componentscan;

import org.springframework.stereotype.Component;

@Component
public class MyComponentService {

    private static final String SERVICE_NAME = "MY_COMPONENT_SERVICE";

    public MyComponentService(){

    }

    public void printServiceName(){
        System.out.println("My Service: " + SERVICE_NAME);
    }
}
{% endhighlight %}

위의 Configuration 클래스 역할을 하는 AppConfig 클래스를 만든다. 위와 달리 `@Bean` 어노테이션이 붙은 MyService Bean 초기화 역할을 담당하는 메서드는 제거하고, `@ComponentScan` 어노테이션을 추가한다. 인자로는 `com.leeyh0216.springstudy.componentscan`을 넣어주었는데, 이는 해당 패키지 아래의 모든 클래스 중 `@Component` 어노테이션이 붙은(사실 @Service 나 @Repository 등이 붙은 클래스도 검색하지만, 향후에 나오기 때문에 자세히 설명하지는 않는다) 클래스를 검색한 뒤, 해당 클래스를 초기화하여 Bean으로 만든다.

#### @Configuration 도 Component Scan의 대상 !

`@Configuration`의 선언에도 아래와 같이 @Component 어노테이션이 붙어있는 것을 확인할 수 있다.

{% highlight java %}
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {

	String value() default "";

}
{% endhighlight %}

즉, Configuration 클래스 또한 ComponentScan의 대상이 된다. 따라서 Configuration 클래스 또한 Bean으로 만들어진다. 물론 Configuration 클래스는 빈을 선언하거나 자원을 선언하는 용도로 사용되어야 하겠지만, ComponentScan에 의해 Bean으로 만들어지는지 테스트하기 위해 아래와 같이 테스트 코드를 작성해보자.

{% highlight java %}
package com.leeyh0216.springstudy.configurationscan;

import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    public void printName(){
        System.out.println(getClass().getName());
    }
}

{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.configurationscan;

import com.leeyh0216.springstudy.componentscan.MyComponentService;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Application {

    public static void main(String[] args) throws Exception {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        ((AnnotationConfigApplicationContext) applicationContext).scan("com.leeyh0216.springstudy.configurationscan");
        ((AnnotationConfigApplicationContext) applicationContext).refresh();
        AppConfig appConfig = applicationContext.getBean(AppConfig.class);
        appConfig.printName();
    }
}

{% endhighlight %}

아래와 같은 출력 결과가 나타나는 것을 볼 수 있다.

{% highlight text %}
21:27:56.588 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemProperties] PropertySource with lowest search precedence
21:27:56.596 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemEnvironment] PropertySource with lowest search precedence
...생략
21:27:57.125 [main] DEBUG org.springframework.core.env.PropertySourcesPropertyResolver - Could not find key 'spring.liveBeansView.mbeanDomain' in any property source
21:27:57.129 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Returning cached instance of singleton bean 'appConfig'
com.leeyh0216.springstudy.configurationscan.AppConfig$$EnhancerBySpringCGLIB$$9282b585
{% endhighlight %}

#### Bean의 이름은 `@Bean` 어노테이션이 붙은 메서드 이름 !

Bean의 이름은 `@Bean` 어노테이션이 붙은 메서드의 이름과 동일하다. 아래와 같은 예제를 실행해보자.

{% highlight java %}
package com.leeyh0216.springstudy.beanname;

public class MyService {

    private static final String SERVICE_NAME = "MY_SERVICE";

    public MyService(){

    }

    public void printServiceName(){
        System.out.println("My Service: " + SERVICE_NAME);
    }
}
{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.beanname;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    @Bean
    public MyService myNameIsMyService(){
        return new MyService();
    }

}

{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.beanname;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Application {

    public static void main(String[] args) throws Exception {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        System.out.println(((AnnotationConfigApplicationContext) applicationContext).getBean("myNameIsMyService").getClass().getName());
    }
}

{% endhighlight %}

위 코드를 실행해보면 아래와 같은 로그가 출력되는 것을 확인할 수 있다.

{% highlight java %}
21:38:40.051 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemProperties] PropertySource with lowest search precedence
21:38:40.060 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemEnvironment] PropertySource with lowest search precedence
...생략
21:38:40.592 [main] DEBUG org.springframework.core.env.PropertySourcesPropertyResolver - Could not find key 'spring.liveBeansView.mbeanDomain' in any property source
21:38:40.594 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Returning cached instance of singleton bean 'myNameIsMyService'
com.leeyh0216.springstudy.beanname.MyService
{% endhighlight %}

만일 Bean에 이름을 특정하고 싶다면, `@Bean`의 name 혹은 value 프로퍼티에 이름으로 사용할 값을 선언하면 된다.

AppConfig 클래스의 myNameIsMyService의 `@Bean`을 `@Bean("servicebean")`으로 변경하면, 해당 이름으로 Bean을 찾을 수 있는 것을 확인할 수 있다.

