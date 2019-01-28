---
layout: post
title:  "Spring Core Technologies - Annotation-based container configuration(1)"
date:   2019-01-28 10:00:00 +0900
author: leeyh0216
categories: spring
---

# Annotation-based container configuration

## 어노테이션 방식의 설정이 XML 방식의 설정보다 나은가?

어노테이션 방식과 XML 방식은 각각 장/단점이 있기 때문에, 어느 것이 낫다고 말할 수는 없다.

어노테이션 방식의 경우 명료하고 정확한 설정을 할 수 있도록 선언 내부에 많은 컨텍스트를 포함하고 있다는 장점이 있다.

반면 XML 방식의 경우 소스코드의 수정이나 재컴파일 없이도 XML만으로 프로그램의 동작을 제어할 수 있는 장점이 있다.

몇몇 개발자들은 소스 코드 내에 객체 간의 연결을 가져가는 것을 선호하는 반면, 다른 개발자들은 어노테이션이 붙은 클래스는 더이상 POJO가 아니며, 설정이 분산된다고 주장한다.

> 개인적으로 XML 기반 설정보다는 어토네이션 기반 설정을 선호한다. XML 설정보다 어노테이션 기반 설정이 더 Compile Time에서 오류를 찾아낼 확률이 높다고 생각하기 때문이다. 다만, 위의 주장 중 설정이 분산된다는 부분에는 동의하는 편이다.

XML 설정과 다르게 어노테이션 설정은 컴포넌트들을 엮는데(Wiring up) 바이트코드 메타데이터에 의존한다. XML에 Bean 와이어링 정보를 기입하는 대신 컴포넌트 클래스의 생성자, 함수, 필드 등에 어노테이션을 기재하여 와이어링을 수행한다.

> 어노테이션 Injection은 XML Injection 이전에 수행된다. 따라서 XML 설정이 어노테이션 설정을 덮어쓰게 된다.

## 어노테이션 설정에서 사용하는 어노테이션 목록

### @Required

`@Required`는 Bean의 Setter에 사용된다. Setter의 인자에 해당 Bean이 필수적으로 의존적이라는 것을 나타내며, Setter 인자에 해당하는 Bean을 찾을 수 없을 경우 예외가 발생한다. 이는 빠르고 명시적인 예외를 발생시켜 `NullPointerException`과 같은 오류가 발생하지 않도록 한다.

{% highlight java %}
package com.leeyh0216.springstudy.requiredannotation;

import org.springframework.beans.factory.annotation.Required;
import org.springframework.stereotype.Component;

@Component
public class MyService{

    MyRepository myRepository;

    @Required
    public void setMyRepository(MyRepository myRepository){
        this.myRepository = myRepository;
    }
}
{% endhighlight %}

다만 `@Required` 어노테이션은 `@Autowired`와 같이 해당 함수를 호출하여 의존성을 주입해주는 역할을 하지 않는다. 때문에 아래와 같이 `MyRepository`가 컴포넌트로 등록되어 있고, ComponentScan에 의해 Bean으로 등록되어 있어도 오류가 발생하게 된다.

{% highlight java %}
package com.leeyh0216.springstudy.requiredannotation;

import org.springframework.stereotype.Component;

@Component
public class MyRepository {
}
{% endhighlight %}

{% highlight bash %}
Exception in thread "main" org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'myService' defined in file [C:\Users\leeyh\Documents\projects\SpringStudy\out\production\classes\com\leeyh0216\springstudy\requiredannotation\MyService.class]: Initialization of bean failed; nested exception is org.springframework.beans.factory.BeanInitializationException: Property 'myRepository' is required for bean 'myService'
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:564)
...생략
{% endhighlight %}

위와 같은 오류가 발생하지 않도록 하기 위해서는 `@Autowired` 어노테이션을 같이 붙여주어야 한다.

{% highlight java %}
package com.leeyh0216.springstudy.requiredannotation;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Required;
import org.springframework.stereotype.Component;

@Component
public class MyService{

    MyRepository myRepository;

    @Autowired
    @Required
    public void setMyRepository(MyRepository myRepository){
        this.myRepository = myRepository;
    }
}
{% endhighlight %}

다만 `@Autowired` 어노테이션에 `@Required`와 같은 역할을 하는 `required` 프로퍼티가 존재한다.
{% highlight java %}
@Target({ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Autowired {

	/**
	 * Declares whether the annotated dependency is required.
	 * <p>Defaults to {@code true}.
	 */
	boolean required() default true;

}
{% endhighlight %}

따라서 아래와 같이 `@Autowired`와 `required` 프로퍼티만 사용하는 것이 더 좋을 듯 하다.

{% highlight java %}
package com.leeyh0216.springstudy.requiredannotation;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Required;
import org.springframework.stereotype.Component;

@Component
public class MyService{

    MyRepository myRepository;

    @Autowired(required=true)
    public void setMyRepository(MyRepository myRepository){
        this.myRepository = myRepository;
    }
}
{% endhighlight %}

### @Autowired

`@Autowired`는 클래스의 필드, 생성자, 메소드에 쓰일 수 있다.

#### 필드에 `@Autowired`를 사용하는 경우
{% highlight java %}
package com.leeyh0216.springstudy.autowired;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class MyService {

    @Autowired
    DependencyA dependency;
}
{% endhighlight %}

위와 같이 필드에 `@Autowired` 어노테이션을 사용할 수 있다. 객체가 초기화 된 이후, ApplicationContext에서 해당 타입(여기서는 DependencyA)의 Bean을 찾아 주입해준다.

다만, 해당 필드는 생성자가 호출된 후 주입된다.

따라서 아래와 같은 코드를 수행할 경우,

{% highlight java %}
package com.leeyh0216.springstudy.autowired;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class MyService {

    @Autowired
    DependencyA dependency;

    public MyService(){
        System.out.println("DependencyA in constructor: " + dependency);
    }

    public void checkDependency(){
        System.out.println("Dependency A after constructor: " + dependency);
    }
}
{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.autowired;

import org.springframework.stereotype.Component;

@Component
public class DependencyA {

    public String toString(){
        return "This is DependencyA";
    }
}
{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.autowired;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Application {

    public static void main(String[] args) throws Exception {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        MyService myService = applicationContext.getBean(MyService.class);
        myService.checkDependency();
    }
}
{% endhighlight %}

아래와 같은 결과 출력된다.

{% highlight bash %}
DependencyA in constructor: null
Dependency A after constructor: This is DependencyA
{% endhighlight %}

따라서 Field Injection 방식을 사용할 경우 생성자 코드 내에서는 해당 객체가 초기화되지 않은 상태이므로, 사용해서는 안된다.

#### 생성자에 `@Autowired`를 사용하는 경우

{% highlight java %}
package com.leeyh0216.springstudy.autowired;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class MyService {
    
    DependencyA dependency;

    @Autowired
    public MyService(DependencyA dependency){
        this.dependency = dependency;
        System.out.println("DependencyA in constructor: " + dependency);
    }

    public void checkDependency(){
        System.out.println("Dependency A after constructor: " + dependency);
    }
}
{% endhighlight %}

위와 같이 생성자에 `@Autowired`를 사용할 수 있다. 해당 클래스를 Bean으로 만들 때, ApplicationContext가 생성자 인자들에 일치하는 Bean을 주입해주는 방식이다.

> 생성자가 1개인 경우 별도로 `@Autowired`를 붙여주지 않아도 된다. 다만 2개 이상의 생성자가 존재하는 경우 사용할 1개의 생성자에만 `@Autowired`를 붙여주어야 한다.

#### Setter에 `@Autowired`를 사용하는 경우

{% highlight java %}
package com.leeyh0216.springstudy.autowired;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class MyService {

    DependencyA dependency;

    public MyService(){
        System.out.println("MyService's constructor called");
    }

    @Autowired
    public void setDependency(DependencyA dependency){
        this.dependency = dependency;
        System.out.println("DependencyA has setted");
    }

    public void checkDependency(){
        System.out.println("Dependency A after constructor: " + dependency);
    }
}
{% endhighlight %}

위와 같이 Setter에 `@Autowired`를 붙여주는 경우, ApplicationContext가 Setter의 인자에 일치하는 Bean을 이용하여 해당 함수를 호출하게 된다.

위의 코드를 실행한 결과는 아래와 같다.

{% highlight bash %}
MyService's constructor called
DependencyA has setted
Dependency A after constructor: This is DependencyA
{% endhighlight %}

#### Autowiring 되는 타입이 애매한(Ambigious) 경우

임의의 인터페이스 A를 상속하는 B와 C 클래스가 존재하고, D 클래스에서는 A 인터페이스를 Autowiring하는 경우 어떤 상황이 발생할까?

아래와 같이 예제 코드를 작성하였다.

{% highlight java %}
package com.leeyh0216.springstudy.ambigiousautowiring;

public interface A {
}
{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.ambigiousautowiring;

import org.springframework.stereotype.Component;

@Component
public class B implements A{
}
{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.ambigiousautowiring;

import org.springframework.stereotype.Component;

@Component
public class C implements A{
}
{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.ambigiousautowiring;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class D {

    @Autowired
    private A a;
    
}
{% endhighlight %}

위와 같은 코드를 실행하면 아래와 같은 오류가 발생한다.

{% highlight bash %}
Exception in thread "main" org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'd': Unsatisfied dependency expressed through field 'a'; nested exception is org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type 'com.leeyh0216.springstudy.ambigiousautowiring.A' available: expected single matching bean but found 2: b,c
{% endhighlight %}

만일 하나의 인터페이스 혹은 클래스를 2개 이상의 클래스가 상속하고, 이를 Autowiring 대상으로 삼는다면 위와 같은 오류가 발생할 수 있으므로, 정확한 클래스를 Autowiring 대상으로 사용하는 것이 좋다.

### @Order

`@Order` 어노테이션은 동일 타입의 Bean 들을 List나 Set, Array 타입으로 Aggregation 할 때, 정렬 기준을 제공한다.
예를 들어 아래와 같이 A 인터페이스를 구현한 B,C 클래스가 존재하고, 이 클래스들을 Array<A> 타입으로 가져오는 코드를 작성해보자.

{% highlight java %}
package com.leeyh0216.springstudy.order;

public interface A {
}
{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.order;

import org.springframework.stereotype.Component;

@Component
public class B implements A {
}
{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.order;

import org.springframework.stereotype.Component;

@Component
public class C implements A {
}
{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.order;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class D {

    public D(A[] dependencies){
        System.out.println("Dependenis size: "+ dependencies.length);
        for(A dependency: dependencies){
            System.out.println("Dependency: " + dependency.getClass().getSimpleName());
        }
    }

}
{% endhighlight %}

위 어플리케이션이 실행되면 아래와 같이 출력이 발생한다.

{% highlight bash %}
Dependenis size: 2
Dependency: B
Dependency: C
{% endhighlight %}

만일 D 클래스의 생성자로 들어오는 `dependencies` 내의 요소들을 정렬하고 싶다면 어떻게 해야할까?

`@Order` 어노테이션을 사용하면 정렬이 가능하다.

B와 C 클래스를 아래와 같이 수정한 후 어플리케이션을 실행해보면 다른 출력내용이 발생하는 것을 확인할 수 있다.

{% highlight java %}
package com.leeyh0216.springstudy.order;

import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(2)
public class B implements A {
}
{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.order;

import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(1)
public class C implements A {
}
{% endhighlight %}

{% highlight bash %}
Dependenis size: 2
Dependency: C
Dependency: B
{% endhighlight %}

`dependencies` 인자 내에서 B와 C의 순서가 변경된 것을 알 수 있다.

다만 `@Order` 어노테이션은 특정 타입의 Bean들을 리스트나 배열 형태의 인자로 받을 때의 순서를 정할 뿐, 객체가 초기화되는 순서를 보장하지는 않는다.

즉, 위와 같이 수정했더라도 꼭 C 클래스의 객체가 B 클래스의 객체보다 먼저 초기화된다는 것을 보장할 수는 없다는 의미이다.

만일 C 클래스의 객체가 B 클래스의 객체보다 먼저 초기화되어야 하는 경우에는 `@DependsOn` 어노테이션을 사용해야 한다.
