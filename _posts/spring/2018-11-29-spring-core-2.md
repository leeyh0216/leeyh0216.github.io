---
layout: post
title:  "Spring Core Technologies - The IoC Container(2)"
date:   2018-11-29 21:00:00 +0900
author: leeyh0216
categories: spring
---

# The IoC Container

## Introduction to the Spring IoC Container and Beans

### Bean 선언 시의 Interface 활용

Bean 객체를 초기화하여 반환하는 메소드(`@Bean` 어노테이션이 붙은) 만들어 ApplicationContext에서 찾아 사용하는 예제를 이전 글에서 만들어 보았다.

해당 예제에서는 초기화하여 반환하는 객체의 타입과 반환 타입이 완전히 일치했는데, 반환 타입은 구체화된 클래스가 아닌 Interface 혹은 Abstract Class로 설정할 수 있다.

먼저 아래와 같이 하위 클래스가 구현해야하는 인터페이스를 만든다.

{% highlight java %}
package com.leeyh0216.springstudy.interfacebean;

public interface IMyService {

    void printServiceName();
    
}

{% endhighlight %}

위의 인터페이스를 상속한 MyServiceV1을 구현한다.

{% highlight java %}
package com.leeyh0216.springstudy.interfacebean;

public class MyServiceV1 implements IMyService {

    private static final String SERVICE_NAME = "MY_SERVICE_V1";

    public MyServiceV1(){

    }

    @Override
    public void printServiceName(){
        System.out.println("My Service: " + SERVICE_NAME);
    }

}

{% endhighlight %}

그 후, IMyService Bean을 초기화할 Configuration 클래스와 메서드를 아래와 같이 구현한다.

{% highlight java %}
package com.leeyh0216.springstudy.interfacebean;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    @Bean
    public IMyService getMyService(){
        return new MyServiceV1();
    }

}
{% endhighlight %}

위 getMyService 함수에서 반환 형은 IMyService이지만, 실제 반환되는 객체는 IMyService를 상속한 클래스인 MyServiceV1의 객체인 것을 확인할 수 있다.

이를 테스트하는 Application 클래스를 아래와 같이 생성한다.

{% highlight java %}
package com.leeyh0216.springstudy.interfacebean;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Application {

    public static void main(String[] args) throws Exception {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        IMyService myService = applicationContext.getBean(IMyService.class);
        myService.printServiceName();
    }
}
{% endhighlight %}

실행 결과는 아래와 같다.

{% highlight text %}
21:12:32.878 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemProperties] PropertySource with lowest search precedence
21:12:32.886 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemEnvironment] PropertySource with lowest search precedence
...생략
21:12:33.427 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Returning cached instance of singleton bean 'getMyService'
My Service: MY_SERVICE_V1
{% endhighlight %}

위 코드의 장점은 무엇일까? 인터페이스 기반으로 작성되었기 때문에, 추후 기능 개선 혹은 추가를 위해 새로운 버전의 클래스인 MyServiceV2를 만들었을 때, Bean을 반환하는 메소드의 초기화 부분만을 수정하면, 이외의 코드는 수정하지 않고 그대로 사용할 수 있다.

아래와 같이 MyServiceV2를 IMyService 인터페이스를 상속받아 구현하고,

{% highlight java %}
package com.leeyh0216.springstudy.interfacebean;

public class MyServiceV2 implements IMyService {

    private static final String SERVICE_NAME = "MY_SERVICE_V2";

    public MyServiceV2(){

    }

    @Override
    public void printServiceName(){
        System.out.println("My Service: " + SERVICE_NAME);
    }

}

{% endhighlight %}

아래와 같이 Configuration 클래스의 getMyService 메소드만 살짝 수정해주면

{% highlight java %}
package com.leeyh0216.springstudy.interfacebean;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    @Bean
    public IMyService getMyService(){
        return new MyServiceV2();
    }

}
{% endhighlight %}

동일한 Application를 실행했을 때, 아래와 같이 다른 코드의 수정 없이도 정상적으로 동작하는 것을 확인할 수 있다.

{% highlight java %}
21:21:04.147 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemProperties] PropertySource with lowest search precedence
21:21:04.157 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemEnvironment] PropertySource with lowest search precedence
...생략
21:21:04.929 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Returning cached instance of singleton bean 'getMyService'
My Service: MY_SERVICE_V2
{% endhighlight %}

### 여러 개의 인터페이스를 구현한 하나의 클래스를 통해 초기화된 객체

아래와 같이 2개의 인터페이스(IMyService, IAnotherService)와 이 둘을 구현한 MyService 클래스가 있다고 생각해보자.

{% highlight java %}
package com.leeyh0216.springstudy.manyinterfacebean;

public interface IMyService {

    void printServiceName();
    
}
{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.manyinterfacebean;

public interface IAnotherService {

    void printAnotherName();

}
{% endhighlight %}

{% highlight java %}
package com.leeyh0216.springstudy.manyinterfacebean;

public class MyServiceV1 implements IMyService, IAnotherService {

    private static final String SERVICE_NAME = "MY_SERVICE_V1";

    public MyServiceV1(){

    }

    @Override
    public void printServiceName(){
        System.out.println("My Service: " + SERVICE_NAME);
    }

    @Override
    public void printAnotherName() { System.out.println(String.format("I also implement %s", IAnotherService.class.getName()));}

}
{% endhighlight %}

Configuration 클래스는 어떻게 구성해야 할까? 일단 아래와 같이 각각 IMyService, IAnotherService 를 반환타입으로 가지는 메서드를 포함한 Configuration 클래스를 만들어 보았다.

{% highlight java %}
package com.leeyh0216.springstudy.manyinterfacebean;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    private static final MyServiceV1 myService = new MyServiceV1();

    @Bean
    public IMyService getMyService(){ return myService; }

    @Bean
    public IAnotherService getAnotherService(){ return myService; }

}
{% endhighlight %}

테스트를 위해 아래와 같이 IMyService, IAnotherService 타입의 Bean을 ApplicationContext로부터 가져오려 했지만, 

{% highlight java %}
package com.leeyh0216.springstudy.manyinterfacebean;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Application {

    public static void main(String[] args) throws Exception {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        IMyService myService = applicationContext.getBean(IMyService.class);
        IAnotherService anotherService = applicationContext.getBean(IAnotherService.class);
        myService.printServiceName();
        anotherService.printAnotherName();
        System.out.println(myService == anotherService);
    }
}
{% endhighlight %}

다음과 같은 오류 메시지가 발생하며 실행에 실패하는 것을 확인할 수 있었다.

{% highlight text %}
Exception in thread "main" org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type 'com.leeyh0216.springstudy.manyinterfacebean.IMyService' available: expected single matching bean but found 2: getMyService,getAnotherService
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveNamedBean(DefaultListableBeanFactory.java:1041)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBean(DefaultListableBeanFactory.java:345)
...생략
{% endhighlight %}

IMyService 타입의 Bean에 만족하는 Bean이 getMyService와 getAnotherService 2개가 발견되었다는 메시지가 발생한다.

분명 getAnotherService 함수는 IAnotherService 인터페이스를 반환했는데도 이러한 오류가 발생하는 것을 확인할 수 있었다.

Stacktrace 첫번째 라인의 함수의 2번째 줄을 보면,

{% highlight java %}
String[] candidateNames = getBeanNamesForType(requiredType);
{% endhighlight %}

와 같이, Application에 등록된 Bean 중 우리가 인자로 전달한 IMyService 타입을 가진 Bean을 반환하는 getBeanNamesForType을 반환하는 것을 볼 수 있으며, 실제 오류가 나는 부분을 보면

{% highlight java %}
	private <T> NamedBeanHolder<T> resolveNamedBean(Class<T> requiredType, Object... args) throws BeansException {
		Assert.notNull(requiredType, "Required type must not be null");
		String[] candidateNames = getBeanNamesForType(requiredType);

		if (candidateNames.length > 1) {
			List<String> autowireCandidates = new ArrayList<String>(candidateNames.length);
			for (String beanName : candidateNames) {
				if (!containsBeanDefinition(beanName) || getBeanDefinition(beanName).isAutowireCandidate()) {
					autowireCandidates.add(beanName);
				}
			}
			if (!autowireCandidates.isEmpty()) {
				candidateNames = autowireCandidates.toArray(new String[autowireCandidates.size()]);
			}
		}

		if (candidateNames.length == 1) {
			String beanName = candidateNames[0];
			return new NamedBeanHolder<T>(beanName, getBean(beanName, requiredType, args));
		}
		else if (candidateNames.length > 1) {
			Map<String, Object> candidates = new LinkedHashMap<String, Object>(candidateNames.length);
			for (String beanName : candidateNames) {
				if (containsSingleton(beanName)) {
					candidates.put(beanName, getBean(beanName, requiredType, args));
				}
				else {
					candidates.put(beanName, getType(beanName));
				}
			}
			String candidateName = determinePrimaryCandidate(candidates, requiredType);
			if (candidateName == null) {
				candidateName = determineHighestPriorityCandidate(candidates, requiredType);
			}
			if (candidateName != null) {
				Object beanInstance = candidates.get(candidateName);
				if (beanInstance instanceof Class) {
					beanInstance = getBean(candidateName, requiredType, args);
				}
				return new NamedBeanHolder<T>(candidateName, (T) beanInstance);
			}
			throw new NoUniqueBeanDefinitionException(requiredType, candidates.keySet());
		}

		return null;
	}
{% endhighlight %}

거의 맨 아랫줄의 throw new NoUniqueBeanDefinitionException에서 발생하는 것을 확인할 수 있다.

원인은 위 함수의 거의 맨 윗 줄에 있는 
{% highlight java %}
String[] candidateNames = getBeanNamesForType(requiredType);
{% endhighlight %}

의 `getBeanNamesForType` 함수이다.

이 함수는 `org.springframework.beans.factory` 패키지에 선언된 `ListableBeanFactory`의 getBeanNamesForType을 구현한 것인데, 해당 함수는 아래와 같이 설명되어 있다.

> 주어진 타입(**SubClass를 포함하여**)과 일치하는 Bean 목록을 반환합니다.

SubClass를 포함했다는 사실이 매우 중요하다.

우리가 IMyService를 반환하는 getMyService와 IAnotherService를 반환하는 getAnotherService를 구현했어도, 결과적으로 반환되는 객체는 MyService 타입이다.

즉, IMyService 클래스를 `getBeanNamesForType`에 넘긴다 해도 구체화 클래스인 `MyService` 클래스의 객체인 `getMyService` Bean과 `getAnotherService` Bean이 반환된다.

두 개의 Bean을 반환할 수는 없기 때문에, Springframework에서 제시하는 기준에 맞춰지는 Bean을 반환하려고 candidate를 찾는 과정이 위의 `resolveNamedBean` 메소드에 구현되어 있는데,

1. `getBeanNamesForType`에서 반환한 Bean 이름이 1개인 경우 해당 이름을 가진 Bean을 반환
2. `getBeanNamesForType`에서 반환한 Bean 이름이 여러개인 경우
   1. `@Primiary` 어노테이션 등을 통해 Bean의 우선 순위를 지정한 경우 가장 높은 우선순위를 가지는 Bean을 반환
   2. 우선 순위가 명확하지 않은 경우 NoUniqueBeanDefinitionException 예외를 throw

와 같은 과정을 가지고 있다.

위 과정을 우리의 코드에 적용해보자면 선택지는 3개가 된다.

1. Class가 아닌 Bean 이름을 통해 Bean을 선택하는 방법
2. `IMyService`, `IAnotherService`를 모두를 상속받는 인터페이스를 `MyService` 클래스가 구현하여 Bean을 1개로 만드는 방법
3. `@Primary`와 같은 우선순위 어노테이션을 이용하여 우선순위에 따라 선택되게 만드는 방법

각 방법을 테스트해보도록 하겠다.

#### Class가 아닌 Bean 이름을 통해 Bean을 선택하는 방법

아래와 같이 기존에 getBean의 인자를 Class를 전달했던 방식에서 실제 Bean 이름을 전달하는 방식으로 변경한다.

{% highlight java %}
package com.leeyh0216.springstudy.manyinterfacebean;

import org.springframework.beans.factory.NoUniqueBeanDefinitionException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Application {

    public static void main(String[] args) throws Exception {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        System.out.println(applicationContext.getBeanNamesForType(IMyService.class).length);
        IMyService myService = (IMyService)applicationContext.getBean("getMyService");
        IAnotherService anotherService = (IAnotherService)applicationContext.getBean("getAnotherService");
        myService.printServiceName();
        anotherService.printAnotherName();
        System.out.println(myService == anotherService);
    }
}
{% endhighlight %}

이 경우 또한 ApplicationContext에 `getMyService`와 `getAnotherService` Bean 모두가 등록되어 있지만 `getMyService` 이름을 가진 Bean만을 가져오기 때문에 위와 같은 오류가 발생하지 않는 것이다.

#### `IMyService`, `IAnotherService` 를 상속받은 인터페이스를 MyService가 구현하는 방법

아래와 같이 `IMyService`, `IAnotherService`를 상속하는 `ITotalService` 인터페이스를 만든다.

{% highlight java %}
package com.leeyh0216.springstudy.manyinterfacebean;

public interface ITotalService extends IMyService, IAnotherService{
}
{% endhighlight %}

그 후 MyService가 해당 Interface를 구현하도록 한다.

{% highlight java %}
package com.leeyh0216.springstudy.manyinterfacebean;

public class MyServiceV1 implements ITotalService {

    private static final String SERVICE_NAME = "MY_SERVICE_V1";

    public MyServiceV1(){

    }

    @Override
    public void printServiceName(){
        System.out.println("My Service: " + SERVICE_NAME);
    }

    @Override
    public void printAnotherName() { System.out.println(String.format("I also implement %s", IAnotherService.class.getName()));}

}
{% endhighlight %}

또한 Configuration 클래스 또한 아래와 같이 `ITotalService`를 반환하도록 수정해주고

{% highlight java %}
package com.leeyh0216.springstudy.manyinterfacebean;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    @Bean
    public ITotalService getTotalService(){ return new MyServiceV1(); }

}
{% endhighlight %}

main 함수 또한 아래와 같이 `IMyService`, `IAnotherService`를 따로 가져오는 것이 아닌 `ITotalService` 하나만을 가져오도록 수정하면 정상적으로 동작하는 것을 확인할 수 있다.

{% highlight java %}
package com.leeyh0216.springstudy.manyinterfacebean;

import org.springframework.beans.factory.NoUniqueBeanDefinitionException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Application {

    public static void main(String[] args) throws Exception {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        ITotalService totalService = applicationContext.getBean(ITotalService.class);
        totalService.printServiceName();
        totalService.printAnotherName();
    }
}
{% endhighlight %}

이 경우는 Bean은 1개가 등록되어 있고, 타입에 따라 가져올 수 있도록 구현된 경우이다.

#### `@Primary` 어노테이션을 이용하여 우선순위에 따라 선택되게 만드는 방법

우선 MyService 클래스를 아래와 같이 수정하여, `getMyService`와 `getAnotherService`에서 반환되는 객체를 구분할 수 있도록 하자.

{% highlight java %}
package com.leeyh0216.springstudy.manyinterfacebean;

public class MyServiceV1 implements IMyService, IAnotherService {

    private String serviceName;

    public MyServiceV1(String serviceName){
        this.serviceName = serviceName;
    }

    @Override
    public void printServiceName(){
        System.out.println("My Service: " + serviceName);
    }

    @Override
    public void printAnotherName() { System.out.println(String.format("I also implement %s", IAnotherService.class.getName()));}

}
{% endhighlight %}

Configuration 클래스의 `getMyService` 메서드에 아래와 같이 `@Primary` 어노테이션을 붙여준다. 또한 두 객체를 구분할 수 있도록 생성자에 각각 "primary"와 "no priority"를 넣어준다.

{% highlight java %}
package com.leeyh0216.springstudy.manyinterfacebean;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@Configuration
public class AppConfig {

    @Primary
    @Bean
    public IMyService getMyService(){ return new MyServiceV1("primary"); }

    @Bean
    public IAnotherService getAnotherService(){ return new MyServiceV1("no priority"); }

}
{% endhighlight %}

그 후 main 함수를 아래와 같이 작성하여 돌려보면 `getMyService` 메서드에서 반환한 Bean이 우선적으로 선택되는 것을 볼 수 있다.

{% highlight java %}
package com.leeyh0216.springstudy.manyinterfacebean;

import org.springframework.beans.factory.NoUniqueBeanDefinitionException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Application {

    public static void main(String[] args) throws Exception {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        IMyService myService = applicationContext.getBean(IMyService.class);
        IAnotherService anotherService = applicationContext.getBean(IAnotherService.class);
        myService.printServiceName();
        anotherService.printAnotherName();
        System.out.println(myService == anotherService);
    }
}
{% endhighlight %}

이 경우 또한 ApplicationContext에 `getMyService`와 `getAnotherService` Bean 모두가 등록되어 있지만 우선순위에 의해 1개만 선택되는 경우이다.

여기까지 Interface 타입을 활용하는 방법과, Bean의 우선순위에 대해 알아보았다.