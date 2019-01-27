---
layout: post
title:  "Spring Core Technologies - Customizing the nature of a bean"
date:   2019-01-23 10:00:00 +0900
author: leeyh0216
categories: spring
---

# Customizing the nature of a bean

## Lifecycle callbacks

Spring에서 제공하는 `InitializingBean` 혹은 `DisposableBean` 인터페이스를 구현한다면, Bean의 Lifecycle을 Container에게 위임할 수 있다.

Container는 Bean 생성 과정에서는 `afterPropertiesSet` 함수를 호출하고 Bean의 소멸 과정에서는 `destroy` 함수를 호출한다.

> `InitializingBean`과 `DisposableBean`은 Spring Framework에서 제공하는 Interface이기 때문에, 해당 Interface를 구현한 코드들은 모두 Spring과 Coupling되는 문제를 가지고 있다. 만일 Spring Framework와의 의존 관계를 없애고 싶을 경우 JSR-250에 정의된 `@PostConstruct`, `@PreDestroy`를 사용하거나, `init-method`, `destroy-method`를 메타데이터에서 지정해주는 것이 좋다.

> 그러나 과연 위의 가이드라인을 따른다고 해서 완전히 Spring Framework에 Independent 한 코드를 짤 수 있을지, 그리고 이러한 코드를 재활용할 수 있을지에 대해서는 의문이 든다.

Spring Framework에서는 BeanPostProcessor 구현체가 Bean 초기화/소멸 과정에서 이러한 함수들을 찾아 적절히 실행시킨다. 만일 이러한 과정을 변경하고 싶다면, BeanPostProcessor를 스스로 구현해야 한다.

추가적으로 Bean의 Lifecycle을 Container의 Lifecycle에 연결하고 싶은 경우, `Lifecycle` Callback을 구현하면 된다.

### Initialization callbacks

`org.springframework.beans.factory.InitializingBean` 인터페이스를 구현하게 되면, Container에 의해 Bean에 필요한 모든 속성(Properties 혹은 Dependencies)을 주입받은 이후 Container에 의해 `afterPropertiesSet` 함수가 호출된다.

단, `InitializingBean` 인터페이스는 Spring과 Coupling되는 이슈가 존재하기 때문에 `@PostConstruct`를 사용하거나 Configuration 메타데이터에 `init-method`를 명시해주는 것이 좋다.

#### InitializingBean을 활용하는 방법

{% highlight java %}
package com.leeyh0216.springstudy.initializingbean;

import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Autowired;

public class InitializingBeanService implements InitializingBean {

    public InitializingBeanService(){
        System.out.println(String.format("%s's constructor called", getClass().getSimpleName()));
    }

    @Autowired
    public void setMyRepository(MyRepository myRepository){
        System.out.println("Set MyRepository Called");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println(String.format("%s's init method called", getClass().getSimpleName()));
    }
}
{% endhighlight %}

위와 같은 코드를 가진 Application을 실행하는 경우, 아래와 같은 출력이 발생한다.

{% highlight bash %}
InitializingBeanService's constructor called
Set MyRepository Called
InitializingBeanService's init method called
{% endhighlight %}

Container가 생성자 -> Setter -> 초기화 메서드 순으로 실행하는 것을 확인할 수 있다.

#### Init-Method 를 활용하는 방법

위의 코드에서 `InitializingBean` 인터페이스를 제거하고, 해당 인터페이스에서 구현해야 할 함수인 `afterPropertiesSet` 함수를 `initThis` 라는 순수한 함수로 변경하였다.

{% highlight java %}
package com.leeyh0216.springstudy.initmethod;

import org.springframework.beans.factory.annotation.Autowired;

public class InitMethodBeanService {

    public InitMethodBeanService(){
        System.out.println(String.format("%s's constructor called", getClass().getSimpleName()));
    }

    @Autowired
    public void setMyRepository(MyRepository myRepository){
        System.out.println("Set MyRepository Called");
    }

    public void initThis() throws Exception {
        System.out.println(String.format("%s's init method called", getClass().getSimpleName()));
    }
}
{% endhighlight %}

다만, Container가 해당 클래스를 Bean으로 만들 때 호출해야 할 `init-method`를 인지할 수 있도록 Configuration 클래스에서 Bean Annotation 속성에 `initMethod`를 기재해주어야 한다.

{% highlight java %}
package com.leeyh0216.springstudy.initmethod;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan("com.leeyh0216.springstudy.initmethod")
public class AppConfig {

    @Bean
    public MyRepository getMyRepository(){
        return new MyRepository();
    }

    @Bean(initMethod="initThis")
    public InitMethodBeanService getInitializingBeanService(){
        return new InitMethodBeanService();
    }
}
{% endhighlight %}

개인적으로 위와 같은 방식은 선호하지 않는다. 언제든 오타를 낼 수 있기에 `initMethod`에 잘못된 이름(혹은 오타가 발생)이 적히는 경우 Compile Time에 잡아낼 수 없기 때문이다(물론 테스트를 넣으면 당연히 잡을 수 있겠지만..).

#### @PostConstruct 를 활용하는 방법

단순히 Bean이 생성된 후 호출될 함수에 `@PostConstruct` 어노테이션만 붙여주면 된다.

{% highlight java %}
package com.leeyh0216.springstudy.postconstruct;

import org.springframework.beans.factory.annotation.Autowired;

import javax.annotation.PostConstruct;

public class PostConstructBeanService {

    public PostConstructBeanService(){
        System.out.println(String.format("%s's constructor called", getClass().getSimpleName()));
    }

    @Autowired
    public void setMyRepository(MyRepository myRepository){
        System.out.println("Set MyRepository Called");
    }

    @PostConstruct
    public void postConstructMethod() throws Exception {
        System.out.println(String.format("%s's init method called", getClass().getSimpleName()));
    }
}
{% endhighlight %}

#### Callback 메서드의 인자와 반환

Spring Framework 문서를 보면 아래와 같은 표현이 등장한다.

> In the case of XML-based configuration metadata, you use the init-method attribute to specify the name of the method that has a void no-argument signature.

즉, `init-method`는 인자가 없는 형태의 함수여야 한다는 것이다. 그래서 아래와 같이 인자를 주고 실행해 보았다.

{% highlight java %}
package com.leeyh0216.springstudy.initmethod;

import org.springframework.beans.factory.annotation.Autowired;

public class InitMethodBeanService {

    public InitMethodBeanService(){
        System.out.println(String.format("%s's constructor called", getClass().getSimpleName()));
    }

    @Autowired
    public void setMyRepository(MyRepository myRepository){
        System.out.println("Set MyRepository Called");
    }

    public void initThis(int a) throws Exception {
        System.out.println(String.format("%s's init method called", getClass().getSimpleName()));
    }
}
{% endhighlight %}

그랬더니 아래와 같이 오류가 발생한다.

{% highlight bash %}
Caused by: org.springframework.beans.factory.support.BeanDefinitionValidationException: Couldn't find an init method named 'initThis' on bean with name 'getInitializingBeanService'
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.invokeCustomInitMethod(AbstractAutowireCapableBeanFactory.java:1716)
	...
{% endhighlight %}

그럼 인자가 아니라 반환형이 있을 경우는 어떨까? 그런 경우도 테스트 해보았는데 정상적으로 동작하는 것을 확인하였다.

동일한 내용을 `@PostConstruct`에도 적용해보았는데, 좀 더 디테일한 오류 메시지가 발생한다.

{% highlight bash %}
Caused by: java.lang.IllegalStateException: Lifecycle method annotation requires a no-arg method: public java.lang.String com.leeyh0216.springstudy.postconstruct.PostConstructBeanService.postConstructMethod(int) throws java.lang.Exception
	at org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor$LifecycleElement.<init>(InitDestroyAnnotationBeanPostProcessor.java:349)
	...
{% endhighlight %}

반환값을 지정하는 경우에는 오류가 발생하지 않고 잘 실행되었지만, 어차피 반환형을 사용하는 곳이 없기 때문에 Spring 문서에서 말했던 것과 같이 인자가 없는 함수 형태로만 정의해서 사용해야겠다.

### Destruction callbacks

Destruction callback에서는 

* `DisposableBean` 인터페이스 구현
* `@PreDestroy` 어노테이션
* `destroy-method` 지정

방식을 제공하고 있다.

위의 내용은 Initialization callback과 동일한 방식으로 구현하기 때문에 별도로 코드를 첨부하지는 않는다.

다만 아래와 같은 3가지 특이사항/주의사항이 존재한다.

#### Java에서 제공하는 리소스 해제 인터페이스 호출

Java에서는 객체가 가지고 있는 리소스를 해제할 수 있도록 강제하는 인터페이스인 `java.lang.AutoClosable`과 `java.io.Closable` 을 제공한다.(`java.io.Closable`은 Java 1.5, `java.lang.AutoClosable`은 Java 1.7에 도입된 인터페이스이며, `java.io.Closable`은 `java.lang.AutoClosable`을 상속하므로써 Backward-Compatibility를 보장한다.)

만일 Bean에 위 2개 인터페이스 중 하나라도 구현되어 있다면 해당 인터페이스의 함수들을 호출하게 된다.

위의 인터페이스들은 try-with-resource 구문과 사용도 가능하기 때문에, 별도로 구현하는 것보다는 위 인터페이스를 사용하는 것이 좋지 않을까 생각한다.

#### Non-Web Application에서는 ApplicationContext의 registerShutdownhook()을 호출해야 한다.

Non-Web Application(주로 Pure Java Application)에서는 Container에서 Application의 종료 시점을 알 수 없으므로, `registerShutdownhook` 함수를 호출하여 현 JVM의 Shutdown Event를 확인할 수 있도록 해야 한다.

`ApplicationContext`의 `registerShutdownhook`은 내부적으로 `Runtime`의 `addShutdownHook`을 호출하여 Application 종료 이벤트를 수신한다.

{% highlight java %}
/**
	 * Register a shutdown hook with the JVM runtime, closing this context
	 * on JVM shutdown unless it has already been closed at that time.
	 * <p>Delegates to {@code doClose()} for the actual closing procedure.
	 * @see Runtime#addShutdownHook
	 * @see #close()
	 * @see #doClose()
	 */
	@Override
	public void registerShutdownHook() {
		if (this.shutdownHook == null) {
			// No shutdown hook registered yet.
			this.shutdownHook = new Thread() {
				@Override
				public void run() {
					synchronized (startupShutdownMonitor) {
						doClose();
					}
				}
			};
			Runtime.getRuntime().addShutdownHook(this.shutdownHook);
		}
	}
{% endhighlight %}

#### finalize는 사용하지 말자

이 부분은 Effective Java에 나오는 내용인데, Java의 Object 객체에는 `finalize`라는 함수를 오버라이딩 할 수 있게 되어 있다.

JavaDoc에는 아래와 같이 기술되어 있다.

> Called by the garbage collector on an object when garbage collection determines that there are no more references to the object. A subclass overrides the finalize method to dispose of system resources or to perform other cleanup.

그러나 실제로 해당 함수가 언제 호출될 지 알 수 없기때문에, 해당 함수의 사용을 권하지 않는다고 되어 있다.

### Combining lifecycle mechanisms

Spring 2.5 버전부터

* InitializingBean, DisposableBean
* Custom init, destroy methods
* @PostConstruct, @PreDestroy

등 Bean의 생애주기를 컨트롤 할 수 있는 방법이 제공된다.

위 메소드들은 아래와 같은 순서로 호출된다.

1. @PostConstruct Annotation이 붙은 메서드
2. InitializingBean을 상속받았을 때 구현하는 afterPropertiesSet 메서드
3. 커스텀 초기화 메서드
4. @PreDestroy Annotation이 붙은 메스더
5. DisposableBean을 상속받았을 때 구현하는 destroy() 메서드
6. 커스텀 소멸 메서드

