---
layout: post
title:  "[Spring] Spring 제어의 역전"
date:   2017-05-28 01:14:00 +0900
author: leeyh0216
categories: dev framework spring
---

> Spring 제어의 역전을 공부하기 위해 작성한 post 입니다.
 http://docs.spring.io/spring/docs/5.0.0.RC1/spring-framework-reference/core.html#beans 페이지를 참고하여 작성하였습니다.

## Spring의 IOC Container

### Dependency Injection(의존성 주입)과 IoC(제어의 역전)
일반적으로 자바 클래스는 다른 클래스의 객체들과 연관 되어 있다(종속적이라고도 표현할 수 있다). 이러한 종속된 객체들을 클래스의 생성자 매개변수, setter 함수 등을 통해 설정하는 과정을 DI(Dependency Injection)이라고 한다.

아래와 같이 2개의 클래스가 존재한다고 가정하자.
{% highlight java %}
package com.leeyh0216.test;

public class CD {
	private String artistName;
	private String songName;
	
	public CD(String artistName, String songName){
		this.artistName = artistName;
		this.songName = songName;
	}
	
	public void setArtistName(String artistName){
		this.artistName = artistName;
	}
	
	public String getArtistName(){
		return this.artistName;
	}
	
	public void setSongName(String songName){
		this.songName = songName;
	}
	
	public String getSongName(){
		return this.songName;
	}
}
{% endhighlight %}

{% highlight java %}
package com.leeyh0216.test;

public class CDPlayer {
	
	private CD cd;
	
	public CDPlayer(CD cd){
		this.cd = cd;
	}
	
	public void play(){
		System.out.println("Play Song : "+cd.getSongName()+", Artist : "+cd.getArtistName());
	}
}
{% endhighlight %}

CDPlayer 클래스는 CD 클래스와 종속적이므로, 실행 시에 CD 객체를 생성자, setter 등을 통해서 주입 받아야 한다.
따라서 CDPlayer의 main() 함수는 다음과 같은 형태가 될 것이다.
{% highlight java %}
public static void main(String[] args) {
	CD cd = new CD("James","Love Song");
	CDPlayer cdPlayer = new CDPlayer(cd);
	cdPlayer.play();
}
{% endhighlight %}

위와 같이 내부에 자신의 동작에서 필요한 객체(의존 객체)를 외부에서부터 세팅되는 방식을 의존성 주입(Dependency Injection)이라고 한다.
물론, 내부에서 해당 객체를 초기화 할 수 있지만, 매우 좋지 않은(결합도가 높고 응집도가 낮아지는) 디자인이므로 이는 지양하는 것이 좋다.

간단한 프로그램의 경우 위와 같이 main() 함수 내에서 필요한 객체들을 초기화하고 주입해줄 수 있지만, 규모가 큰 프로그램일 수록 관리하기가 어려워 진다.
의존 객체를 외부에서 주입하는 것이 아닌 객체 스스로 주입하는 방식을 제어의 역전이라고 한다.

### Spring Framework에서의 DI와 IoC

Spring Framework에서는 이러한 IoC 기능을 ApplicationContext라는 Interface를 구현한 클래스들이 수행한다.
Spring에서 자바 클래스로 생성되는 객체는 빈(Bean)이라고 부른다.
ApplicationContext는 이러한 자바 빈들을 Application 시작 시에 Singleton 객체(로 만들지 않도록 설정할 수도 있다)로 만들어놓고, 이 빈들이 필요한 객체들에 주입해준다.

일반 자바 클래스를 빈으로 만들게 해달라는 방법은 @Component 어노테이션을 붙이는 방법이다.
다음은 간단한 자바 클래스이다.
{% highlight scala %}
package com.leeyh0216.anydata.services

import org.springframework.stereotype.Service
import org.springframework.stereotype.Component

class TaskService {

  println("Task Service Initalized")

  var taskName : String = _
  
  def test() = {
    "hello test"
  }
}
{% endhighlight %}

위 자바 클래스는 Spring Framework에서 빈으로 만들 수 없다. Spring Framework에서 빈으로 만들 수 있도록 혀용하기 위해서는 다음과 같이 @Component 어노테이션을 붙여주어야 한다.
{% highlight scala %}
package com.leeyh0216.anydata.services

import org.springframework.stereotype.Service
import org.springframework.stereotype.Component

@Component
class TaskService {

  println("Task Service Initalized")  
  
  var taskName : String = _
  
  def test() = {
    "hello test"
  }
}
{% endhighlight %}

위와 같이 @Component 어노테이션이 붙여져 있는 클래스의 경우 Spring의 ApplicationContext에서 Singleton 객체로 만들어 관리하게 된다.

다음은 간단한 Spring Application의 main() 함수가 있는 클래스(스칼라 코드로 작성했다)이다.
{% highlight scala %}
package com.leeyh0216.anydata

import org.springframework.context.annotation.ComponentScan
import org.springframework.boot.autoconfigure.EnableAutoConfiguration
import org.springframework.context.annotation.Configuration
import org.springframework.boot.SpringApplication

@Configuration
@EnableAutoConfiguration
@ComponentScan
class Server

object Server {
  def main(args : Array[String]) {
    SpringApplication.run(Array(classOf[Server].asInstanceOf[Object]), args);
  }  
}
{% endhighlight %}

@ComponentScan 이라는 어노테이션이 붙어 있는데, 이는 자신의 패키지를 포함한 하위 패키지의 클래스 중 @Component 어노테이션이 붙어 있는 클래스들을 찾는다는 뜻이고, 이를 찾아 Singleton 빈 객체로 만들어 놓는다는 것을 의미한다.

위 main() 함수를 실행하면 다음과 같은 Console Log를 볼 수 있다.

{% highlight bash %}
2017-05-29 18:07:41.767  INFO 5084 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'requestContextFilter' to: [/*]
Task Service Initalized
2017-05-29 18:07:42.256  INFO 5084 --- [           main] s.w.s.m.m.a.RequestMappingHandlerAdapter : Looking for @ControllerAdvice: org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@3d8314f0: startup date [Mon May 29 18:07:39 KST 2017]; root of context hierarchy
2017-05-29 18:07:42.341  INFO 5084 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/sample/hello]}" onto public org.springframework.http.ResponseEntity<java.lang.Object> com.leeyh0216.anydata.controllers.SampleController.home()
2017-05-29 18:07:42.345  INFO 5084 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/error]}" onto public org.springframework.http.ResponseEntity<java.util.Map<java.lang.String, java.lang.Object>> org.springframework.boot.autoconfigure.web.BasicErrorController.error(javax.servlet.http.HttpServletRequest)
2017-05-29 18:07:42.346  INFO 5084 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/error],produces=[text/html]}" onto public org.springframework.web.servlet.ModelAndView org.springframework.boot.autoconfigure.web.BasicErrorController.errorHtml(javax.servlet.http.HttpServletRequest,javax.servlet.http.HttpServletResponse)
2017-05-29 18:07:42.386  INFO 5084 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/webjars/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2017-05-29 18:07:42.386  INFO 5084 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2017-05-29 18:07:42.441  INFO 5084 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**/favicon.ico] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2017-05-29 18:07:42.622  INFO 5084 --- [           main] o.s.j.e.a.AnnotationMBeanExporter        : Registering beans for JMX exposure on startup
2017-05-29 18:07:42.710  INFO 5084 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat started on port(s): 8080 (http)
2017-05-29 18:07:42.716  INFO 5084 --- [           main] com.leeyh0216.anydata.Server$            : Started Server. in 3.644 seconds (JVM running for 4.155)
{% endhighlight %}

두 번째 줄을 보면 Task Service Initalized 가 출력되어 있는 것을 볼 수 있다.
main() 함수 그 어느 곳에서도 TaskService 클래스의 객체를 만들어준 적이 없지만, Spring ApplicationContext에서 해당 클래스의 객체를 생성했음을 알 수 있다.

다음과 같은 클래스를 하나 더 만들어보자. 아래는 URL을 호출하여 실행 시킬 수 있는 Controller 클래스인데, 지금은 어노테이션과 함수의 의미를 정확히 알 필요는 없다.
{% highlight scala %}
package com.leeyh0216.anydata.controllers

import org.springframework.beans.factory.annotation.Autowired
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.stereotype.Controller
import org.springframework.web.bind.annotation.RequestMapping

import com.leeyh0216.anydata.services.TaskService

@Controller
@RequestMapping(Array("/sample"))
class SampleController {

  @Autowired
  var taskService : TaskService = null
  
  @RequestMapping(Array("/hello"))
  def home() = {
      new ResponseEntity[Object](taskService.test(), HttpStatus.OK)
  }
}
{% endhighlight %}

위의 home() 함수를 호출하기 위해서는 Application 실행 후 http://localhost:8080/sample/hello를 웹 브라우져에서 들어가보면 된다.
"hello test"라는 문자열이 웹 페이지에 찍히는 것을 볼 수 있다.

위 클래스의 taskService 객체를 보면 분명 null로 초기화 되어 있는 것을 알 수 있다.
하지만 프로그램 실행 시에 Spring Framework에서는 @Autowired 어노테이션을 보고 미리 생성된 TaskService 빈(Singleton)을 위 taskService에 주입해주게 된다.

## Spring의 DI, IoC 관련 Annotation

### @Component Annotation

@Component 어노테이션을 클래스의 상단에 넣어주게 되면, 이 클래스는 프로그램 실행 시에 Singleton 형태의 객체(빈)을 만들어 Spring에서 관리받겠다는 의미이다.
만일 이 어노테이션을 붙이지 않게 되면 Spring에서는 해당 클래스를 찾을 수 없고, Bean으로 만들지 못하게 된다.

### @ComponentScan Annotation

클래스 위에 @ComponentScan 어노테이션을 붙이게 되면, 프로그램 실행 시에 자신의 패키지를 포함한 하위 패키지의 모든 클래스 중 @Component 어노테이션이 붙은 클래스들을 빈으로 생성하여 관리한다.
일반적으로 Application 진입점의 클래스에 붙여준다.

### @Autowired Annotation

@Autowired 가 붙은 객체의 빈을 찾아 주입해준다.
위에서는 단순히 필드에만 붙여주었지만, 다음과 같이 함수에도 붙여줄 수 있다.

{% highlight scala %}
package com.leeyh0216.anydata.controllers

import org.springframework.beans.factory.annotation.Autowired
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.stereotype.Controller
import org.springframework.web.bind.annotation.RequestMapping

import com.leeyh0216.anydata.services.TaskService

@Controller
@RequestMapping(Array("/sample"))
class SampleController {

  
  var taskService : TaskService = null
  
  @Autowired
  def setTaskService(taskService : TaskService){
    this.taskService = taskService;
  }
  
  @RequestMapping(Array("/hello"))
  def home() = {
      new ResponseEntity[Object](taskService.test(), HttpStatus.OK)
  }
}
{% endhighlight %}

위의 setTaskService 함수는 @Autowired 어노테이션이 붙어 있기 때문에, 프로그램 실행 시에 ApplicationContext가 함수 매개변수에 필요한 자바 빈들을 확인한 후, 함수를 자동으로 호출해준다.
즉, 위와 같이 @Autowired 어노테이션이 붙은 함수를 호출하는 호출자(caller)는 프로그래머가 아닌 Spring Framework의 ApplicationContext이다.

의존 객체를 주입할 때, 필요에 의해 무엇인가 작동해줘야 하는 상황에서는 위와 같이 setter 함수를 통해 주입받을 수 있다(생성자도 가능하다)

### @Configuration Annotation
만일 외부 라이브러리 등을 사용할 때, 외부 라이브러리의 특정 클래스를 빈으로 등록하고 싶을 때는 어떻게 해야 할까?
외부 라이브러리에서는 @Component 어노테이션을 붙여넣지 않았기 때문에 @ComponentScan 에 의해 자바 빈이 초기화되지 않는다.
따라서 @Configuration 클래스를 만들어서 우리가 직접 빈을 만들어 주어야 한다.

다음과 같이 TaskService 클래스가 @Component 어노테이션을 붙이지 않았다고 가정하자.
{% highlight scala %}
package com.leeyh0216.anydata.services

import org.springframework.stereotype.Service
import org.springframework.stereotype.Component

class TaskService {
  
  println("Task Service Initalized")
  
  var taskName : String = _
  
  def test() = {
    "hello test"
  }
}
{% endhighlight %}

이럴 경우 다음과 같이 Configuration 클래스를 만들어 Spring Framework가 우리가 생성한 자바 빈을 찾도록 유도할 수 있다.
{% highlight scala %}
package com.leeyh0216.anydata.services

import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.Bean

@Configuration
class TaskServiceConfiguration {
  
  @Bean
  def getTaskService() : TaskService = {
    println("getTaskService Called")
    val taskService = new TaskService();
    taskService
  }
}
{% endhighlight %}

위 클래스를 보면, @Configuration이 붙어 있는 것을 확인할 수 있는데, 이 어노테이션은 Spring Framework에게 내가 정의한 빈들이 있으니 가져다 쓰거라~ 라는 의미를 가지고 있다.

getTaskService 함수를 보면 @Bean 어노테이션이 붙어 있는데, 이 함수의 결과로 자바 객체(빈)을 반환한다는 의미이다.
즉, Spring Application Context는 프로그램 실행 시에 위 TaskServiceConfiguration 클래스를 찾은 후 @Configuration 어노테이션이 붙어 있는 것을 보고 '아, 이 클래스에 자바 빈들이 있구나. 가져가야겠다' 하고 내부의 필드나 함수들을 확인하고, @Bean 어노테이션이 붙어있는 필드나 함수를 호출하여 얻어낸 객체를 빈으로써 Application Context에서 관리한다.

프로그램을 실행해보면 다음과 같이 getTaskService Called가 출력된 것을 확인할 수 있다.
{% highlight bash %}
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.5.3.RELEASE)

2017-05-29 18:34:57.407  INFO 5201 --- [           main] com.leeyh0216.anydata.Server$            : Starting Server. on leeyh0216ui-MacBook-Air.local with PID 5201 (/Users/leeyh0216/Documents/projects/AnyData/bin started by leeyh0216 in /Users/leeyh0216/Documents/projects/AnyData)
2017-05-29 18:34:57.413  INFO 5201 --- [           main] com.leeyh0216.anydata.Server$            : No active profile set, falling back to default profiles: default
2017-05-29 18:34:57.489  INFO 5201 --- [           main] ationConfigEmbeddedWebApplicationContext : Refreshing org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@3d8314f0: startup date [Mon May 29 18:34:57 KST 2017]; root of context hierarchy
2017-05-29 18:34:59.226  INFO 5201 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat initialized with port(s): 8080 (http)
2017-05-29 18:34:59.242  INFO 5201 --- [           main] o.apache.catalina.core.StandardService   : Starting service Tomcat
2017-05-29 18:34:59.244  INFO 5201 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/8.5.14
2017-05-29 18:34:59.370  INFO 5201 --- [ost-startStop-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2017-05-29 18:34:59.370  INFO 5201 --- [ost-startStop-1] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1884 ms
2017-05-29 18:34:59.551  INFO 5201 --- [ost-startStop-1] o.s.b.w.servlet.ServletRegistrationBean  : Mapping servlet: 'dispatcherServlet' to [/]
2017-05-29 18:34:59.561  INFO 5201 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'characterEncodingFilter' to: [/*]
2017-05-29 18:34:59.563  INFO 5201 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
2017-05-29 18:34:59.563  INFO 5201 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'httpPutFormContentFilter' to: [/*]
2017-05-29 18:34:59.563  INFO 5201 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'requestContextFilter' to: [/*]
getTaskService Called
{% endhighlight %}

단, @Component 가 붙어있는 클래스에 대해서 다시 @Configuration 클래스에서 @Bean이 붙은 형태로 함수를 만들었을 때는, 충돌이 일어나게 된다. 이는 Bean ID 설정(Qualifier)로 해결할 수 있다.
