---
layout: post
title:  "Spring Core Technologies - The IoC Container(4)"
date:   2018-12-23 10:00:00 +0900
author: leeyh0216
tags:
- spring
---

# The IoC Container

## Dependencies

간단한 어플리케이션부터 기업형 어플리케이션까지 하나의 객체로만 동작하는 프로그램은 없다. 적어도 몇개의 객체들이 서로 상호작용하며 어플리케이션을 구성하고 있다.

### Dependency Injection

의존성 주입(Dependency Injection, D.I)은 객체들이 자신의 의존성(의존 객체)을

* 생성자 인자
* 팩토리 메서드의 인자
* Setter

를 통해 정의하여 Container가 해당 객체를 생성할 때, 필요한 의존 객체들을 주입해주는 방식을 말한다. 

의존성 주입을 이용하면 코드가 깔끔해지고, 객체와 의존 객체들을 효과적으로 Decoupling 시킬 수 있다.

의존성 주입은 크게 **생성자를 통한 의존성 주입**과 **Setter 기반의 의존성 주입**으로 존재한다.

#### 생성자를 통한 의존성 주입

생성자를 통한 의존성 주입은 객체가 의존성을 생성자의 인자에 표현하고, Container가 해당 인자들로 생성자를 호출하는 방식이다. 이 방식은 Static Factory Method를 사용하는 방식과 거의 비슷하다.

##### Constructor Argument Resolution

Container는 생성자의 인자 타입을 이용하여 의존성을 주입한다. 생성자 인자들 간의 모호성이 없는 경우는 생성자에 정의된 순서대로 의존성들이 초기화 된 후 주입된다.

아래와 같이

* DependencyA
* DependencyB
* MyService

3개의 컴포넌트가 존재하고, `MyService` 객체가 `DependencyA`와 `DependencyB`에 의존한다고 생각해보자.

**DepencencyA.java**
{% highlight java %}
package com.leeyh0216.springstudy.constructbaseddi;

import org.springframework.stereotype.Component;

@Component
public class DependencyA {

    public DependencyA(){
        System.out.println("DependencyA initialized");
    }

    public void printName(){
        System.out.println("This is DependencyA");
    }

}
{% endhighlight %}

**DependencyB.java**
{% highlight java %}
package com.leeyh0216.springstudy.constructbaseddi;

import org.springframework.stereotype.Component;

@Component
public class DependencyB {

    public DependencyB(){
        System.out.println("DependencyB initialized");
    }

    public void printName(){
        System.out.println("This is DependencyB");
    }
}
{% endhighlight %}

**MyService.java**

아래의 `MyService` 객체는 생성자를 통해 `DependencyA`와 `DependencyB`를 주입받고 초기화되는 것을 볼 수 있다.

{% highlight java %}
package com.leeyh0216.springstudy.constructbaseddi;

import org.springframework.stereotype.Component;

@Component
public class MyService {

    private DependencyA depA;
    private DependencyB depB;

    public MyService(DependencyA depA, DependencyB depB){
        System.out.println("MyService initialized");
        this.depA = depA;
        this.depB = depB;
    }

    public void printAllDependencies(){
        depA.printName();
        depB.printName();
    }
}
{% endhighlight %}

**Application.java**
{% highlight java %}
package com.leeyh0216.springstudy.constructbaseddi;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Application {

    public static void main(String[] args) throws Exception {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        MyService myService = applicationContext.getBean(MyService.class);
        myService.printAllDependencies();
    }
}
{% endhighlight %}

위의 코드를 실행해보면 

{% highlight text %}
DependencyA initialized
DependencyB initialized
MyService initialized
This is DependencyA
This is DependencyB
{% endhighlight %}

와 같이 출력되는 것을 확인할 수 있다.

우리는 명시적으로 `MyService`, `DependencyA`, `DependencyB` 객체를 
MyService는 의존성으로 DependencyA, DependencyB 가 필요하기 때문에, Container는 DependencyA -> DependencyB -> MyService Bean들을 초기화하게 된다.

`int`, `long`, `double` 등의 Primitive Type이나 `Integer`, `Long`, `Double` 등의 Object Type들도 Bean으로 만들 수 있다.

위의 예제를 아래와 같이 변경해보자.

{% highlight java %}
package com.leeyh0216.springstudy.constructbaseddi;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan("com.leeyh0216.springstudy.constructbaseddi")
public class AppConfig {

    @Bean("age")
    public Integer getAge(){
        return Integer.valueOf(27);
    }
}
{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.constructbaseddi;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class MyService {

    private DependencyA depA;
    private DependencyB depB;

    public MyService(DependencyA depA, DependencyB depB, Integer age){
        System.out.println("MyService initialized");
        this.depA = depA;
        this.depB = depB;
        System.out.println("Age: " + age);
    }

    public void printAllDependencies(){
        depA.printName();
        depB.printName();
    }


{% endhighlight %}

위의 예제를 실행시켜보면 아래와 같이 출력되는 것을 확인할 수 있다.

{% highlight text %}
DependencyA initialized
DependencyB initialized
MyService initialized
Age: 27
This is DependencyA
This is DependencyB
{% endhighlight %}

`@Configuration` 어노테이션이 붙은 AppConfig 클래스에서 age Bean을 초기화시켰기 때문에 `MyService` 생성자의 age라는 이름의 Integer 타입의 인자에 해당 값이 주입된 것을 확인할 수 있다.

#### Setter를 통한 의존성 주입

Setter를 통한 의존성 주입은 Container 인자가 없는 생성자 혹은 인자가 없는 Static Factory Method를 통해 초기화되는 Bean의 의존성을 Setter를 통해 의존 객체를 주입해주는 방식을 말한다.

아래와 같이 `MyService` 클래스를 변경해보자.

{% highlight java %}
package com.leeyh0216.springstudy.setterbaseddi;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class MyService {

    private DependencyA depA;
    private DependencyB depB;

    public MyService(){
        System.out.println("MyService initialized");
    }

    @Autowired
    public void setDependencyA(DependencyA depA){
        System.out.println("DependencyA setted");
        this.depA = depA;
    }

    @Autowired
    public void setDependencyB(DependencyB depB){
        System.out.println("DependencyB setted");
        this.depB = depB;
    }

    public void printAllDependencies(){
        depA.printName();
        depB.printName();
    }
}
{% endhighlight %}

해당 예제를 실행해보면 아래와 같은 결과가 출력되는 것을 볼 수 있다.

{% highlight text %}
DependencyA initialized
DependencyB initialized
MyService initialized
DependencyB setted
DependencyA setted
This is DependencyA
This is DependencyB
{% endhighlight %}

Container는 위와 같이 DependencyA, DependencyB, MyService 객체를 모두 초기화한 후, `@Autowired` 어노테이션이 붙어 있는 함수를 호출하여 의존성을 주입해주게 된다.

> 여기서 잠깐! 분명 Spring 문서에는
>> Container는 인자가 없는 생성자나 인자가 없는 Static Factory Method 를 통해 초기화되는 Bean에 대해서만 Setter를 통한 의존성 주입을 한다.
라고 쓰여 있다.
> 그러나 아래와 같은 방식으로 Constructer Based D.I 와 Setter Based D.I 를 혼합하여 사용할 수도 있었다.

{% highlight java %}
package com.leeyh0216.springstudy.setterbaseddi;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class MyService {

    private DependencyA depA;
    private DependencyB depB;

    public MyService(@Autowired DependencyA depA){
        System.out.println("MyService initialized");
        this.depA = depA;
    }

    @Autowired
    public void setDependencyB(DependencyB depB){
        System.out.println("DependencyB setted");
        this.depB = depB;
    }

    public void printAllDependencies(){
        depA.printName();
        depB.printName();
    }
}
{% endhighlight %}

#### 생성자를 통한 의존성 주입 or Setter를 통한 의존성 주입

생성자를 통한 의존성 주입은 반드시 필요한 의존성을 주입 받을 때 사용하고 Setter를 통한 의존성 주입은 추가적으로 필요한 의존성을 주입받을 때 사용하면 좋다.

* 생성자를 통한 의존성 주입의 경우 생성자로 전달되는 인자가 많은 경우 코드 악취를 풍길 수 있다.

* Setter를 통한 의존성 주입을 `@Required` 어노테이션과 함께 사용하는 경우(일치하는 Bean이 존재하지 않을 경우 NULL이 들어옴) NPE가 발생할 수 있으므로 해당 의존성을 사용하는 모든 코드에서 Not-Null Check를 해주어야 한다.

* 써드파티 라이브러리의 경우 Setter를 제공하지 않는 클래스가 존재할 수 있기 때문에 대부분 생성자를 통한 의존성 주입을 사용할 수 밖에 없다.
