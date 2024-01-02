---
title: "Spring 웹 어플리케이션의 실행과정과 Container 구조"
categories: Spring
tags:
  - web
---  

Spring을 이용하여 만들어진 웹 어플리케이션은 어떤 과정을 거쳐 실행되고 요청을 받아들이는 걸까? Spring 웹 어플리케이션의 실행과정에 대해서 자세하게 알아보자.  

<br />  

## Spring Web 어플리케이션의 실행과정  
![spring_web_execute](https://github.com/kids-ground/mentos-infra/assets/52196792/759be230-70b1-41bc-874b-3bcfca51a636){: .align-center style="width: 70%;"}  
Spring Web 실행과정
{: .image-caption style="font-size: 14px;" }  

위 이미지는 Spring을 실행시 과정을 나타내었다. 한 단계씩 자세하게 살펴보자.  

### Servlet Container 실행 및 설정 로딩  
Servlet Container가 실행되고 웹 구성정보를 로딩한다. Spring에서는 `web.xml`, Spring boot에서는 `application.yml`에 작성된 웹 관련 설정들을 로딩한다.  
Servlet Container가 실행되고 곧바로 `ServletContext`가 실행된다.  


{% capture ServletContext %}
**💡 ServletContext**  
웹 어플리케이션의 전체 셍명주기, 실행환경에 대한 정보와 공통자원을 가지며 서블릿들이 공유하여 사용한다. ServletContext는 다음과 같은 특징이 있다.
- 웹 어플리케이션 하나에 하나만 생성
- **서블릿 및 필터 등록정보, 세션정보, 서버정보 등의 정보**를 가지고 있다.
- ServletContext를 통해 **ApplicationContext에 접근하여 빈에 대한 정보**를 얻을 수 있다.
- 요청마다 생성되는 HttpServletRequest, HttpServletResponse들도 ServletContext를 통해 접근할 수 있다
{% endcapture %}
<div class="notice--info">{{ ServletContext | markdownify }}</div>  

<br />  

### 3. ContextLoaderListener 실행  
`ContextLoaderListner`는 `ServletContextListener` 인터페이스를 구현한 구현체로 `ServletContext`의 생명주기를 관찰한다. `ServletContext` 시작 시 WebApplicationContext를 실행시킨다.  

**💡 Spring boot에서의 ServletContextListener**  
Spring에서는 web.xml에 등록된 `ContextLoaderListener`가 Java Class 파일로 생성하고 이후 AppcliationContext를 실행시키지만 Spring Boot에서는 SpringApplication 클래스가 내부적으로 `ServletWebServerApplicationContext`를 사용하여 웹 어플리케이션의 `ApplicationContext`를 실행, 관리한다.  
{: .notice--info}  

<br />  

### 4. ApplicationContext 실행
ApplicationContext는 흔히 Spring Container라고 불린다. 다시 말해, 이 단계에서 Spring Container가 구성된다. 컴포넌트스캔을 통해 패키지 및 하위 패키지를 스캔하여 여러 객체들을 빈으로 등록되고 의존성을 주입(DI)한다. 스프링 컨테이너는 웹 어플리케이션에서 아래와 같은 상속관계를 가진다. 

1. **BeanFactory**  
빈 객체의 생성(지연로딩 - 요청시 객체 생성), 관리, 검색 등의 기능을 제공하는 인터페이스  
2. **ApplicationContext**  
`BeanFacotry`를 확장한 인터페이스로 국제화, 메시지 처리, 이벤트 발행 및 처리, 리소스 로딩 및 처리, AOP(Aspect-Oriented Programming) 지원, 트랜잭션 관리 등의 기능을 제공한다.  
3. **WebApplicationContext**{: .font-highlight}  
`ApplicationContext`를 확장한 인터페이스로 `getServletContext` 메서드를 추가적으로 더 가지고 있다. `ContextLoaderListener`로 실행되는 스프링 컨테이너는 실질적으로 이 인터페이스의 구현체이다.  

이후 Servlet들은 `ServletContext`를 통해서 스프링 컨테이너로 접근할 수 있으며 스프링의 빈들도 `getServletContext` 메서드를 통해서 `ServletContext`에 접근할 수 있다.  

<br />  

### 5. 클라이언트의 요청 시 DispatchServlet 실행  
Servlet Container와 Spring Container가 실행되면 요청을 받을 준비가 끝난다.  
이후 클라이언트 요청이 오면 `DispatcherServlet`이 실행되고 요청을 받아 처리한다.  

<br /><br />  

## DispatcherServlet의 요청처리 과정
**DispatcherServlet는 Servlet Container로 들어오는 요청을 관리, 제어하는 컨트롤러**{: .font-highlight}이다. `DispatcherServlet`은 처음부터 생성되는 것이 아니라 서버가 띄워지고 처음 요청이 들어오면 생성된다. 이후에는 생성되어 있는 객체를 이용한다. 즉, 지연 생성되는 것이다.  

`DispatcherServlet`는 Front Controller 패턴을 이용하여 구현한 서블릿이다. 가장 앞단에서 **모든 요청을 받아 라우팅을 결정해주고 공통 로직(인코딩, 권한 검사 등)을 처리하여 유연하고 확장성 있는 웹 어플리케이션은 개발**할 수 있도록 도와준다.  

`DispatcherServlet`이 요청을 처리하는 과정을 알아보자

![dispatcherservlet](https://github.com/kids-ground/mentos-infra/assets/52196792/8c682d28-5a7b-41e9-921e-a8c02b6a1705){: .align-center style="width: 70%;"}  
DispatcherServlet 동작과정
{: .image-caption style="font-size: 14px;" }  

### 1. HandlerMapping 조회  
요청이 들어오면 DispatcherServlet이 요청을 받는다. 그리고 `HandlerMapping` 을 조회하여 해당 요청을 처리할 수 있는 핸들러가 존재하는지 찾는다. 여기서 핸들러란, 우리가 Controller에 `GetMapping`, `PostMapping` 어노테이션을 붙은 메서드들을 말한다.  

<br />  

### 2. HandlerAdapter 조회 & 요청  
핸들러를 처리할 수 있는 Adapter를 찾는다. 핸들러들은 Get이냐, Post냐 폼데이터를 받는냐, 해당 핸들러가 무엇을 인자로 받느냐 등 여러가지 형태로 존재한다. **다양한 형태로 존재하는 핸들러들을 일관된 방법으로 처리할 수 있는 것이 핸들러어댑터**이다.  

이 어댑터 패턴 덕분에 우리가 **Controller와 핸들러들을 필요한, 원하는 형태로 작성할 수 있고 결과적으로 유연하고 확장성 있는 웹 어플리케이션을 작성할 수 있는 것**{: .font-highlight}이다.  

어댑터들은 어댑터 인터페이스를 구현한 클래스들로 핸들러를 인자로 받아 처리할 수 있는지를 boolean값으로 리턴하는 메서드와 처리방식을 구현한 메서드 두가지가 존재한다.  

<br />  

### 3. Controller 핸들러에서 요청처리 
어댑터가 핸들러를 실행시켜 요청을 처리한다. 이 과정에서 Service, Repository 빈들을 이용해서 DB나 네트워크로 데이터를 주고받아 데이터를 조작한다.  

<br />  

### 4. 데이터 반환  
요청을 다 처리하고 나서 데이터를 반환하는데 데이터의 유형은 크게 2가지로 나눠질 수 있다. 첫번째는 View 그리고 String, JSON, XML과 같은 데이터 형식이다.  

`@RestController` 어노테이션이 붙은 클래스이거나 핸들러에 `@ResponseBody` 어노테이션이 붙어있다면 메세지컨버터에 의해 데이터형식으로 바꾸어준다. 그렇지 않다면 View 이름으로 인식한다.

<br />

### 5. 뷰 리졸버를 이용한 뷰 처리
`DispatcherServlet`이 반환받은 값이 View 이름이라면 뷰 리졸버를 통해 뷰 템플릿을 반환받고 Model을 주입한다.

<br />  

### 6. 클라언트에 응답
마지막으로 뷰, 데이터를 클라이언트로 응답하게 된다.  

<br /><br />  

## Servlet Container와 Spring Container의 구조  
앞서 Spring 웹 어플리케이션이 실행되는 과정과 요청을 처리하는 과정을 알아보았다. 그 과정 속에서는 여러가지 객체들이 존재하는데 이를 도식화하면 다음과 같다.  

![servlet_spring_container_구조](https://github.com/kids-ground/mentos-infra/assets/52196792/34ac201d-21c7-40de-9966-025f331a3c22){: .align-center style="width: 70%;"}  
Servlet Container와 Spring Container의 구조
{: .image-caption style="font-size: 14px;" }  

`DispatcherServlet` 요청처리 과정에서 사용되던 `HandlerMapping`, `HandlerAdpater` 등은 스프링 컨테이너의 `ServletWebApplicationContext` 에 저장되어 있다. `DispatcherServlet`은 `ServletContext`을 통해 스프링 컨테이너의 정보에 접근한다.  

또 Spring boot에서는 Servlet Container가 내재화 되어있다. 덕분에 `ContextLoaderListener`에 의해서 스프링 컨테이너가 만들어지지않고 Spring boot가 알아서 스프링 컨테이너를 띄워준다.  

또한 Spring boot에서는 `DispatcherServlet`와 `Filter`가 스프링 컨테이너의 빈으로 등록되어 스프링 컨테이너가 동작시킨다. 또한 ServletContext에도 등록되어 ServletContainer와 Servlet들이 접근할 수도 있다.  

위에 보이는 그림에서 색이 들어간 부분만 개발자들이 신경써서 개발하면 된다. 나머지는 Spring boot가 자동으로 구성해준다! 물론 필요에 따라 추가적으로 기능을 넣을 수는 있다. 말이 나온김에 Spring boot가 자동으로 구성해주는 기능들을 보자면 다음과 같다. 
- Embedded Servlet Container 설정
- DispatcherServlet 등록
- View Resolver 등록 - 여러 템플릿 엔진을 사용할 수 있음
- 정적 자원 처리
- Exception Resolver 구현(예외 처리)
- 보안 설정 - 웹 보안을 위한 기본적인 설정을 자동을 처리하여 인증과 권한 부여를 간편하게 설정할 수 있음
- Autuator - 상태 모니터링과 관리
- 한 번에 배포 - 배포과정이 간단해짐

<br />  

**참고**  
- https://gowoonsori.com/spring/architecture/
- https://sigridjin.medium.com/servletcontainer%EC%99%80-springcontainer%EB%8A%94-%EB%AC%B4%EC%97%87%EC%9D%B4-%EB%8B%A4%EB%A5%B8%EA%B0%80-626d27a80fe5
- https://velog.io/@dyunge_100/Spring-%EA%B8%B0%EB%B3%B8%EC%A0%81%EC%9D%B8-%EC%84%A4%EC%A0%95%EC%97%90-%EB%8C%80%ED%95%98%EC%97%AC
- https://mangkyu.tistory.com/221