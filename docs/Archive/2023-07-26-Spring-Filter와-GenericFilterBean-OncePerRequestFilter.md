---
layout: default
parent: Archive
title: "Spring - Filter와 GenericFilterBean, OncePerRequestFilter"
categories: Spring
tags:
  - filter
---  

Spring boot에서 Filter를 만들기 위한 방법에는 여러가지가 존재한다. 그 중 자주 사용하는 방식은 다음과 같다.  
> 1. Filter 인터페이스 이용
2. GenericFilterBean 추상 클래스 이용
3. OncePerRequestFilter 추상 클래스 이용  

필터를 만들고 등록하는 과정은 편리하다. 위 방식을 이용해 만든 class를 빈으로 등록하기만 하면 된다. 그렇다면 각 방식들은 어떤 점이 다른 것일까?


### Filter 인터페이스
`Filter` 인터페이스는 모든 Filter의 최상위 인터페이스다. Filter의 동작을 정의하는 `doFilter` 메서드와 default 메서드로 정의되어 있는 `init`, `destroy` 메서드가 존재한다.  
```java
public interface Filter {
  default void init(FilterConfig filterConfig) throws ServletException {
  }

  void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
          throws IOException, ServletException;

  default void destroy() {
  }
}
```  

`Filter` 인터페이스를 구현한 Filter들은 서블릿에 진입할때마다 호출된다. 클라이언트의 요청당 한 번 실행되는 것이 아닌 서블릿에 진입할 때마다 호출되는 것이다.  

서블릿은 요청당 한 번만 실행되는거 아닌가? 라고 생각할 수 있다.(필자도 그렇게 생각했다.) 서블릿을 두 번 이상 호출하는 예로 forwarding이 있다. forwarding은 Servlet Container로 재요청하는 것과 마찬가지여서 FilterChain이 다시 실행된다.  

가장 중요한 점은 `Filter` 인터페이스 직접 구현체는 **Servlet의 스펙으로 빈으로 등록할 수 없다는 것**이다. 따라서, 이후에 나올 `GenericFilterBean`이나 `OncePerRequestFilter` 을 이용하여 빈으로 등록할 수 있다.

**💡 forwarding이란?**  
요청에 대한 처리를 같은 자원 내 다른 컨트롤로에게 전달하는 방법이다. 요청을 받은 컨트롤러에서 "forward:경로"를 리턴하면 해당 경로로 자원 내에서 리디렉션된다.
{: .notice--info}  

<br />  

### GenericFilterBean 추상클래스
`GenericFitlerBean` 은 `Filter` 인터페이스를 구현한 추상 클래스이다. 일반 `Filter` 인터페이스와의 차이점은 **스프링의 정보를 가져올 수 있도록 확장되어 있다는 점**이다. 뿐만 아니라 초기화(init) 시 수행할 작업이 정의되어 있고 빈으로 등록할 수 있어 커스텀 Filter를 구현하기 편리하다. 하지만 마찬가지로 하나의 클라이언트 요청 내 서블릿 진입시마다 호출된다는 특징을 가진다.

```java
public abstract class GenericFilterBean implements Filter, BeanNameAware, EnvironmentAware, EnvironmentCapable, ServletContextAware, InitializingBean, DisposableBean {
  // ...
}
```


<br />  


### OncePerRequestFilter 추상클래스
`OncePerRequestFilter`는 `GenericFilterBean` 추상클래스를 상속받은 추상클래스이다. 따라서, `GenericFilterBean`의 기능을 모두 가지고 있다.  

이에 더해서 이름에서 알 수 있듯 **클라이언트의 요청당 한 번만 실행됨을 보장**해준다. 상황에 따라 `GenericFilterBean`을 사용할 것인지 `OncePerRequestFilter`를 사용할 것인지 잘 판단해서 구현해야 할 것 같다.

```java
public abstract class OncePerRequestFilter extends GenericFilterBean {
  // ...
}
```