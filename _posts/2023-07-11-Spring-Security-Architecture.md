---
title: "Spring Security Architecture"
categories: Spring
tags:
  - security
---  

Spring Security는 Spring Framwork를 기반으로 한 인증과 인가, 웹 공격에 대한 방어를 위한 기술이다. Servlet Container, Webflux 기반 모두 지원한다. 이 중 Servlet Container 기반으로 Spring Security가 동작하는 방식에 대해서 알아보자.  

## Spring Security Architecture
Spring Security는 Servlet Container 기반의 웹 어플리케이션에서 **Servlet Filter로 동작**{: .font-highlight}한다.  

![Spring FilterChain](https://docs.spring.io/spring-security/reference/_images/servlet/architecture/filterchain.png){: .align-center style="width: 30%;"}  
출처: [Spring Security 공식문서](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
{: .image-caption style="font-size: 14px;" }  

Servlet Filter는 클라이언트 요청이 Servlet에 전달되기 전에 동작한다. 여러개의 Filter를 체인형태로 묶은 것을 FilterChain이라고 부른다. 요청뿐만 아니라 응답또한 FilterChain을 거치며 조작할 수 있다. 이때 Filter라는 이름답게 요청이나 응답을 거를 수 있다.  

<br />  

### DelegatingFilterProxy
Servlet Container에는 Servlet Filter를 등록할 수 있지만 Spring의 Bean들을 인식하지 못한다. **`DelegatingFilterProxy`은 Servlet Filter에 Bean을 등록할 수 있도록 도와주는데 그 역할이 바로**{: .font-highlight}이다.  

![DelegatingFilterProxy](https://docs.spring.io/spring-security/reference/_images/servlet/architecture/delegatingfilterproxy.png){: .align-center style="width: 30%;"}  
출처: [Spring Security 공식문서](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
{: .image-caption style="font-size: 14px;" }  

Servlet Container의 Filter로 `DelegatingFilterProxy`를 등록하지만 그에 대한 구현 작업은 Spring Bean으로 위임하는 것이다. 다시말해, **Servlet Container와 Spring Container 사이의 다리역할을 하는 Spring Security의 중요한 구성요소**이다.  

> 덕분에 Spring에서 Filter 작업에 대한 구현을 쉽게 할 수 있다.

Servlet Filter들은 Spring Container가 시작되기 전에 등록되어져야한다. 즉, `DelegatingFilterProxy`가 등록될때는 Bean이 로드되지 않고 지연 로딩이 된다.  

<br />  

### FilterChainProxy  
Spring Security에서는 **`DelegatingFilterProxy`의 Filter 작업 구현 Bean으로 `FilterChainProxy`를 사용**한다. `FilterChainProxy`는 요청에 대해 알맞은 `SecurityFilterChain`으로 요청을 위임한다.  

어차피 `SecurityFilterChain`으로 위임할 것인데 `FilterChainProxy`을 사용하는 것에 의아해 할 것이다. 이에 대해서는 `SecurityFilterChain`을 알아보고 난 뒤에 설명한다.  

![FilterChainProxy](https://docs.spring.io/spring-security/reference/_images/servlet/architecture/filterchainproxy.png){: .align-center style="width: 60%;"}  
출처: [Spring Security 공식문서](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
{: .image-caption style="font-size: 14px;" }  

<br />  

### SecurityFilterChain
이 부분부터가 개발자가 직접 만지는 부분이다. 앞선 개념들은 Spring Security가 Servlet Filter로써 동작하지만 Bean을 이용하여 구현할 수 있도록 만든 기술들에 대한 설명이었다.  

`SecurityFilterChain`은 이름과 같이 **여러개의 Security Filter을 묶어낸 객체**이다. 요청이 들어오면 Security Filter들이 순서대로 처리하고 마지막까지 통과가되면 다음 Servelet Filter가 동작한다. Servlet Container에서 동작하는 FilterChain과는 다른 것을 유의하자! 굳이 따르자면 SecurityFilterChain은 nested FilterChain이라고 볼 수 있다.  

![FilterChainProxy](https://docs.spring.io/spring-security/reference/_images/servlet/architecture/securityfilterchain.png){: .align-center style="width: 60%;"}  
출처: [Spring Security 공식문서](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
{: .image-caption style="font-size: 14px;" }  

앞서 살펴보았듯이, `SecurityFilterChain`은 `FilterChainProxy`에 등록된다. `FilterChainProxy`를 둠으로써 여러가지 이점이 존재하는데 다음과 같다.
1. **모든 Spring Security의 시작점**  
문제 발생시 디버깅을 하기 위한 시작 지점이 될 수 있다.  
2. **필수 작업 수행**  
공통적으로 처리 되는 로직이 수행된다.  
3. **SecurityFilterChain 결정**  
`RequestMatcher` 인터페이스를 이용하여 `HttpServletRequest`의 요소들로 호출을 결정할 수 있다.  
**`RequestMatcher`를 이용하여 URL 경로나, 헤더에 따라서 `SecurityFilterChain` 결정할 수 있다.** 등록된 각 패턴마다 독립적 설정하고 실행된다. 이러한 특성을 이용해서 Spring Security가 특정 요청을 무시하길 원하는 패턴이 있다면 Security Filter가 없을 수도 있다.

<br />  

### Security Exceptions
Security Filter 중 `ExceptionTranlationFilter`를 사용하면 `AccessDeniedException` 및 `AuthenticationException` 을 HTTP 응답으로 변환할 수 있다.  

![FilterChainProxy](https://docs.spring.io/spring-security/reference/_images/servlet/architecture/exceptiontranslationfilter.png){: .align-center style="width: 60%;"}  
출처: [Spring Security 공식문서](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
{: .image-caption style="font-size: 14px;" }  

- 사용자가 인증되지 않았거나, `AuthenticationException`인 경우에는 Start Authentication
- `AccessDeniedException`인 경우에는 `AccessDeniedHandler`가 실행  

`ExceptionTranlationFilter`의 의사코드는 다음과 같다.

```java
try {
  filterChain.doFilter(request, response); 
} catch (AccessDeniedException | AuthenticationException ex) {
  if (!authenticated || ex instanceof AuthenticationException) {
    startAuthentication(); 
  } else {
    accessDenied(); 
  }
}
```  
<br />  

**참고**
- [Spring Security 공식문서](https://docs.spring.io/spring-security/reference/5.7/servlet/architecture.html)