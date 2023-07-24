---
title: "Spring - mvc 설정을 위한 다양한 방법들"
categories: Spring
tags:
  - mvc
---  

Spring boot를 이용한 프로젝트 도중 WebMvc 설정에 문제가 발생하였다. `WebMvcConfigurationSupport`와 `WebMvcConfigurer`를 혼용하면서 발생한 문제였다. 가까스로 문제를 해결한 후 해당 개념에 대한 지식이 부족하다고 판단하여 이에 관한 내용을 정리해보았다.  

### WebMvc 설정을 위한 방법들
xml을 이용한 설정 방식을 제외하고 자바 코드를 이용한 WebMvc 설정방식에는 다음과 같이 3가지 방법을 지원한다.  

> 1. `@EnableWebMvc` 어노테이션 사용  
2. `WebMvcConfigurer` 인터페이스 구현
3. `WebMvcConfigurationSupport` 클래스 상속  

<br />  

### @EnableWebMvc
`@EnableWebMvc`는 Spring Framework에서 웹과 관련된 전략들을 자동으로 구성해주는 역할을 한다. 일일이 MessageConverter나 ViewResolver를 등록하지 않아도 `@EnableWebMvc`를 통해 **웹과 관련된 기능들을 자동 구성한다.**  

`@EnableWebMvc` 어노테이션을 까보면 `DelegatingWebMvcConfiguration` Import하는 것을 볼 수 있다. `DelegatingWebMvcConfiguration`는 `WebMvcConfigurationSupport`를 상속받고 있는 클래스이다. 따라서 **@EnableWebMvc 어노테이션은 `WebMvcConfigurationSupport`을 사용한다는 것을 알 수 있다.**(WebMvcConfigurationSupport 대한 내용은 다음 섹션에서 이야기한다.)  
```java
// EnableWebMvc
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}

// DelegatingWebMvcConfiguration
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
  // ...
}
```  

Spring Framework 가 제공하는 자동구성과 더불어 기능을 확장하고 싶다면 해당 클래스에 `WebMvcConfigurer` 인터페이스를 implements 시켜 필요한 기능을 추가할 수 있다.  
```java
@Configuration
@EnableWebMvc
public class WebMvcConfig implements WebMvcConfigurer {
  @Override
  public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
    // Method Argument Resolver 추가
    resolvers.add(customArgumentResolver);
  }
}
```

Spring boot가 등장하면서 Web과 관련되 더 많은 설정들을 `WebMvcAutoConfiguration`을 통해 자동설정하기 때문에 직접적으로 `@EnableWebMvc`를 사용하지 않는다. 만약 사용한다면 `WebMvcAutoConfiguration` 기능이 비활성화되고 `@EnableWebMvc`을 통한 자동구성을 진행하기 때문이다.  

`WebMvcAutoConfiguration`을 살펴보면 `WebMvcConfigurationSupport.class`가 존재하지 않을때 활성화된다는 의미의 `@ConditionalOnMissingBean` 어노테이션이 사용되고 있다.  
`@EnableWebMvc`를 사용한 자동구성과 `WebMvcAutoConfiguration`을 사용한 자동설정에서 큰 차이가 보이지는 않지만 Spring boot를 사용한다면 `@EnableWebMvc` 사용을 권장하지 않는다.(단, 개발자가 완전 제어가 필요한 Low한 수준의 작업의 경우 사용)
```java
@AutoConfiguration(after = { DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class, ValidationAutoConfiguration.class })
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
public class WebMvcAutoConfiguration {
}
```  

<br />  

### WebMvcConfigurationSupport
`WebMvcConfigurationSupport`는 Spring의 WebMvc 설정 커스텀을 도와주는 클래스이다. 이를 직접적으로 상속받아 Spring Framework의 자동구성과 더불어 필요한 설정을 추가할 수 있다.  

`@EnableWebMvc`에서는 해당 어노테이션이 붙은 클래스에 `WebConfigurer` 인터페이스를 구현하여 자동구성을 확장할 수 있다. 즉, @EnableWebMvc + WebConfigurer는 WebMvcConfigurationSupport 상속과 같은 의미이다.  

```java
@Configuration
public class WebMvcConfig extends WebMvcConfigurationSupport {
  @Override
  public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
    // Method Argument Resolver 추가
    resolvers.add(customArgumentResolver);
  }
}
```

마찬가지로 직접 `WebMvcConfigurationSupport`을 사용하기 때문에 `WebMvcAutoConfiguration`가 비활성화된다.  


<br />  

### WebMvcConfigurer
자동설정(AutoConfiguration)된 MVC 구성에 추가적인 정보를 제공할 때 쓰인다. 쉽게 말해 MVC 구성을 확장시킬 수 있는 것이다.
`WebMvcConfigurer`은 인터페이스로 `WebMvcAutoConfiguraion`의 정보를 default 메서드로 담고 있어 필요한 메서드만 가져와서 정보를 추가할 수 있다.
- addFormatters
- addInterceptors
- addResourceHandlers
- addArgumentResolvers

위 메서드를 이용해서 자동구성에 추가적인 설정들을 더할 수 있다.

<bt />  


**참고**  
- https://incheol-jung.gitbook.io/docs/q-and-a/spring/enablewebmvc
- http://honeymon.io/tech/2018/03/13/spring-boot-mvc-controller.html