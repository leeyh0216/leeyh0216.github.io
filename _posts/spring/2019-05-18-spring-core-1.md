---
layout: post
title:  "Spring IoC Container - Container, Bean overview"
date:   2019-05-18 10:00:00 +0900
author: leeyh0216
categories: spring-core
---

# The IoC Container

## Introduction to the Spring IoC Container and Beans

Inversion of Control(IoC, 제어의 역전)은 Dependency Injection(DI, 의존성 주입)로도 알려져 있다.

객체는 자신이 동작하는데에 필요한 의존성(객체)을
* 생성자
* 팩토리 메서드의 인자
* Setter
를 통해 설정할 수 있도록 한다.

Container는 객체를 생성할 때 의존성들을 위에서 정의한 방법으로 주입해준다.

객체가 자신의 초기화나 의존성의 설정을 스스로 하는 것이 아니라 Container에 위임하는 방식이기 때문에 제어의 역전이라고 불리우는 것이다.

---

`org.springframework.beans`와 `org.springframework.context` 패키지가 Spring Framework의 IoC Container의 기반을 제공한다.

이 중 `BeanFactory` 인터페이스는 객체를 관리하는 매커니즘을 제공하고, `ApplicationContext`는 `BeanFactory` 를 상속받은 하위 인터페이스로써,

* AOP
* Message Resource Handling
* Event Publication
* WebApplicationContext와 같은 Application-Layer Specific Context

등의 엔터프라이즈 기능을 추가적으로 제공한다.

Spring에서 Application을 구성하는 객체들은 Spring IoC Container에 의해 관리되며, Bean이라고 불린다. Bean은 Spring IoC Container에 의해 초기화, 조합, 관리된다.

또한 Bean과 Bean을 구성하는 의존성들은 Container에게는 Configuration metadata로 표현된다.

## Container Overview

![Spring IoC Container](/assets/spring/spring_ioc_container.jpg)

`org.springframework.context.ApplicationContext`는 Spring IoC Container를 대표하는 인터페이스로써 앞에서 언급한 Bean의 생성, 구성, 조합을 담당한다.

Container는 어떤 객체를 초기화하고 구성하고 조합할지를 Congiruation metadata를 사용해서 수행한다. Configuration metadata는 `XML`, `Java Annotations`, `Java Code` 등의 다양한 포맷으로 존재한다.

`XML` 방식이 전통적인 Congiruation metadata 표현 방식이지만, `Java Annotations`나 `Java Code`로도 Configuration metadata를 표현할 수 있다.

> Spring Boot Application을 공부하려는 목적이 크기 때문에, `XML`이나 `Application Context`를 직접 초기화하는 방법은 시도하지 않은 예정이다.

## Bean Overview

Container는 Bean을 관리할 때 `BeanDefinition`이라는 객체를 이용하여 관리한다. `BeanDefinition` 안에는 다음과 같은 속성들이 있다.

* `package-qualified class name`: Bean을 정의할 실제 구현 클래스
* Container에서의 Bean의 동작 상태(scope, lifecycle callbacks 등)
* Bean이 동작하는데 필요한 다른 Bean(collaborators 나 dependencies로 불림)

### `@Component`로 생성된 Bean의 Bean Definition

"HelloWorld!!" 를 출력하는 서비스를 아래와 같이 인터페이스와 구현 클래스를 만들었다.

`IHelloService.class`
```
package com.leeyh0216.springframeworkstudy.beandefinition;

public interface IHelloService {

    void printHelloWorld();
}
```

`HelloServiceImpl.class`
```
package com.leeyh0216.springframeworkstudy.beandefinition;

import org.springframework.stereotype.Component;

@Component("HelloService")
public class HelloServiceImpl implements IHelloService{

    @Override
    public void printHelloWorld() {
        System.out.println("Hello World!!");
    }
}
```

그리고 위에서 정의된 `HelloServiceImpl` Bean을 스캔할 수 있도록 Configuration 클래스를 아래와 같이 생성하였다.

`ApplicationConfiguration.class`
```
package com.leeyh0216.springframeworkstudy.beandefinition;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan("com.leeyh0216.springframeworkstudy.beandefinition")
public class ApplicationConfiguration {
}
```

아래와 같이 메인 함수를 작성하여 실행시켜보면
```
package com.leeyh0216.springframeworkstudy.beandefinition;

import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Application {

    public static void main(String[] args){
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(ApplicationConfiguration.class);
        BeanDefinition beanDefinition = applicationContext.getBeanDefinition("HelloService");

        System.out.println("Bean class name: " + beanDefinition.getBeanClassName());
        System.out.println("Bean Scope: " + beanDefinition.getScope());
        System.out.println("Bean constructor argument values: " + beanDefinition.getConstructorArgumentValues());
        System.out.println("Bean depends on " + beanDefinition.getDependsOn());
        System.out.println("Is lazy init bean? " + beanDefinition.isLazyInit());
        System.out.println("Has bean property values? " + beanDefinition.hasPropertyValues());
    }
}
```
다음과 같은 출력이 발생한다.
```
Bean class name: com.leeyh0216.springframeworkstudy.beandefinition.HelloServiceImpl
Bean Scope: singleton
Bean constructor argument values: org.springframework.beans.factory.config.ConstructorArgumentValues@cb
Bean depends on null
Is lazy init bean? false
Has bean property values? false
```

대부분의 정보가 정상적으로 출력되는 것을 알 수 있다.

### Factory 메소드로 생성된 Bean의 Bean Definition

위의 예제를 약간 수정하여 Factory Method로 Bean을 생성하도록 하였다.

`HelloServiceImpl.class`
```
package com.leeyh0216.springframeworkstudy.beandefinition;

public class HelloServiceImpl implements IHelloService{

    @Override
    public void printHelloWorld() {
        System.out.println("Hello World!!");
    }
}
```
`HelloServiceImpl` 클래스의 경우 `@Component` 어노테이션을 없애서 자동으로 Bean으로 생성되는 것을 방지하였다.

`ApplicationConfiguration.class`
```
package com.leeyh0216.springframeworkstudy.beandefinition;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan("com.leeyh0216.springframeworkstudy.beandefinition")
public class ApplicationConfiguration {

    @Bean("HelloService")
    public IHelloService getIHelloService(){
        return new HelloServiceImpl();
    }

}
```
`ApplicationConfiguration.class`에서는 `IHelloService` Bean을 생성할 수 있는 Factory Method를 만들었다.

위 변경된 사항으로 다시 메인 함수를 실행했을 경우 아래와 같은 출력이 발생하는 것을 알 수 있다.

```
Bean class name: null
Bean Scope: 
Bean constructor argument values: org.springframework.beans.factory.config.ConstructorArgumentValues@cb
Bean depends on null
Is lazy init bean? false
Has bean property values? false
```

### Factory Method로 생성한 Bean 정보의 행방

Factory Method로 생성한 Bean의 경우 기본 정보(Bean의 클래스명, Scope)가 null로 표기된다. 이러한 정보들은 어디에 있을까? Breakpoint를 잡아 BeanDefinition 객체를 확인해 보았다.

![Factory Method Bean Definition](/assets/spring/factory_method_beandefinition.png)

`beanDefinition` 객체의 `factoryMethodMetadata` 내부에 어느정도 Bean Definition에 관련된 내용을 확인할 수 있는 것을 알 수 있다.

다만, Scope 등의 정보는 여기에도 없다. 하지만 Scope를 설정해주지 않고도 정상 동작하는 것과, 여러 번의 객체 생성을 시도하여도 동일 객체가 반환되는 것을 보면 역시 기본 Scope인 Singleton으로 동작하는 것을 확인할 수 있었다.

다만 `@Component`, `@Service` 등의 어노테이션을 붙여 Bean으로 만든 경우에는 아래와 같이 BeanDefinition에 설정한 정보들이 정상적으로 들어있는 것을 알 수 있다.

![Annotated Bean Bean Definition](/assets/spring/annotated_bean_beandefinition.png)

---
### Naming Beans

모든 Bean들은 1개 이상의 식별자를 가진다. Container가 Bean들을 관리하기 위해서는 이 식별자가 유일해야 한다.

`@Component` 어노테이션을 확인해보면 다음과 같이 value가 이름을 나타내는 것을 볼 수 있다.
```
/**
 * The value may indicate a suggestion for a logical component name,
 * to be turned into a Spring bean in case of an autodetected component.
 * @return the suggested component name, if any (or empty String otherwise)
 */
String value() default "";
```

아래와 같은 예제를 작성해 보았다.

`ISimpleService.class`
```
package com.leeyh0216.springframeworkstudy.namingbeans;

public interface ISimpleService {

    void printVersion();
}
```

`SimpleServiceImplV1.class`
```
package com.leeyh0216.springframeworkstudy.namingbeans;

import org.springframework.stereotype.Component;

@Component
public class SimpleServiceImplV1 implements ISimpleService {
    @Override
    public void printVersion() {
        System.out.println("V1");
    }
}
```

`SimpleServiceImplV2.class`
```
package com.leeyh0216.springframeworkstudy.namingbeans;

import org.springframework.stereotype.Component;

@Component("SimpleServiceImpl")
public class SimpleServiceImplV2 implements ISimpleService{
    @Override
    public void printVersion() {
        System.out.println("V2");
    }
}
```

`SimpleServiceImplV3.class`
```
package com.leeyh0216.springframeworkstudy.namingbeans;

public class SimpleServiceImplV3 implements ISimpleService {
    @Override
    public void printVersion() {
        System.out.println("V3");
    }
}
```

`ApplicationConfiguration.class`
```
package com.leeyh0216.springframeworkstudy.namingbeans;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan("com.leeyh0216.springframeworkstudy.namingbeans")
public class ApplicationConfiguration {
    @Bean(name={"NewSimpleService","SimpleServiceV3"})
    public SimpleServiceImplV3 getSimpleServiceImplV3(){
        return new SimpleServiceImplV3();
    }
}
```

`Application.class`
```
package com.leeyh0216.springframeworkstudy.namingbeans;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Application {
    public static void main(String[] args){
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(ApplicationConfiguration.class);

        ISimpleService b1 = applicationContext.getBean("simpleServiceImplV1", SimpleServiceImplV1.class);
        b1.printVersion();

        ISimpleService b2 = applicationContext.getBean("SimpleServiceImpl", SimpleServiceImplV2.class);
        b2.printVersion();

        ISimpleService b3_1 = applicationContext.getBean("NewSimpleService", SimpleServiceImplV3.class);
        b3_1.printVersion();

        ISimpleService b3_2 = applicationContext.getBean("SimpleServiceV3", SimpleServiceImplV3.class);
        b3_2.printVersion();
    }
}
```

`SimpleServiceImplV1` 클래스와 같이 이름을 지정하지 않은 경우 클래스명의 첫번째 문자를 소문자로 변경하고, Camel Case화 시켜서 이름으로 간주한다.

`SimpleServiceImplV2` 클래스와 같이 이름을 명시적으로 지정하는 경우, 해당 이름을 사용하여 Bean을 찾을 수 있다.

`SimpleServiceImplV3` 클래스의 경우 다른 클래스와 달리 팩토리 메서드로 생성했으며, `@Bean` 어노테이션이 사용되었다. `@Bean` 어노테이션의 경우 1개 이상의 이름을 지정할 수 있도록 되어 있다. 때문에 NewSimpleService, SimpleServiceV3 등으로 Bean을 접근하여도 동일한 Bean이 반환되는 것을 확인할 수 있다.

`@Bean` 어노테이션의 name이 아래와 같이 지정되어 있기 때문에 1개 이상의 식별자를 사용할 수 있는 것으로 보인다.

```
/**
 * The name of this bean, or if several names, a primary bean name plus aliases.
 * <p>If left unspecified, the name of the bean is the name of the annotated method.
 * If specified, the method name is ignored.
 * <p>The bean name and aliases may also be configured via the {@link #value}
 * attribute if no other attributes are declared.
 * @see #value
 */
@AliasFor("value")
String[] name() default {};
```
