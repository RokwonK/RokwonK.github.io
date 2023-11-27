---
layout: default
parent: Archive
title: "Spring Security Authentication Architecture"
categories: Spring
tags:
  - security
---  

Spring Security은 인증 기능을 지원해준다. 인증방식에는 Username/Password를 이용한 방식, OAuth2.0을 이용한 방식 등 여러가지 메커니즘들이 존재하는데 Spring Security는 여러가지 인증방식에 대해 모두 지원해준다.  

Spring Security가 이러한 인증방식을 어떤 방식으로 처리하는지 동작방식을 자세하게 살펴보자.  

**💡 인증(Authentication)이란?**  
회원가입/로그인 과정이라고 생각하면 쉽다. 서비스에 접근하는 유저가 정당한 사용자인지 확인하는 과정이다. 인증을 통해 접근 권한(Authorization)을 부여할 수 있다.  
{: .notice}  

<br />  

## Authentication Architecture  
Spring Security은 많은 인증방식들(OAuth2.0, Username/Password)을 쉽게 사용할 수 있도록 도와준다. 우리는 내부동작을 몰라도 `formLongin()`, `oauth2Login()` 등을 통해 여러가지 인증 방식을 쉽게 구현할 수 있다.  
각 인증방식들은 동작하는 방식이 조금씩 다르지만 동일한 아키텍쳐를 기반으로 동작한다.  

<br />  

### SecurityContextHolder
`SecurityContextHolder`는 **유저들의 인증정보들을 가지고 있는 인증정보 바구니**{: .font-highlight}이다. 현재 요청에 대한 유저의 인증정보을 가지고 있는 `SecurityContext`를 담는 그릇이다.  

만약 현재 유저의 인증정보를 얻으려면 이 객체에 접근하면 된다. `SecurityContextHolder`은 ThreadLocal을 이용하기 때문에 현재 쓰레드에서는 항상 동일한 `SecurityContext`를 가져올 수 있다.  

<br />  

### SecurityContext
`SecurityContextHolder` 내부에 존재하며 `Authentication` 객체를 가지고 있다.  

![SecurityContextHolder](https://docs.spring.io/spring-security/reference/5.7/_images/servlet/authentication/architecture/securitycontextholder.png){: .align-center style="width: 50%;"}  
출처: [Spring Security 공식문서](https://docs.spring.io/spring-security/reference/5.7/servlet/authentication/architecture.html)
{: .image-caption style="font-size: 14px;" }  

<br />  

### Authentication
**실질적으로 유저정보를 받아올 수 있는 인터페이스**{: .font-highlight}이다. 이 객체는 Spring Security에서 두 가지 목적으로 사용된다.
1. `AuthenticationManager`의 input으로 사용 - 사용자가 제공한 인증정보를 바탕으로 자격증명을 하기 위함
2. 인증된 유저임을 나타내기 위함

`Authentication`를 구현한 객체는 다음의 것들을 가지고 있다.  
1. **principal** - 유저식별자(Username/Password 인증방식의 경우 `UserDetails` 객체)
2. **credentials** - 비밀번호(보통 보안을 위해 유저가 인증되면 사라진다)
3. **authorities** - 해당 유저에게 보여된 role 이나 scope

<br />  

### AuthenticationManager
**인증을 수행하는 방법을 정의한 인터페이스**{: .font-highlight}이다. `authenticate()` 메서드가 정의되어있다. 해당 메서드는 파라미티로 받은 `Authentication` 객체를 이용해서 인증을 수행하고 인증 처리된 `Authentication` 객체를 반환한다.  

다음은 실제로 정의되어 있는 `AuthenticationManager` 인터페이스이다.

```java
public interface AuthenticationManager {
	/**
	 * @param authentication the authentication request object
	 * @return a fully authenticated object including credentials
	 * @throws AuthenticationException if authentication fails
	 */
	Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
```  

<br />  

### ProviderManager
**`AuthenticationManager`을 구현한 객체로 Spring Security에서 사용되는 기본 구현체**이다. 이 객체는 `AuthenticationProvider`(AuthenticationManager 아님 주의) 리스트를 가지고 있다. 각각의 `AuthenticationProvider`는 `support` 메서드를 지원하는데 이를 통해 인증처리를 진행할 수 있는지를 확인한다. 리스트 중 처리가능한 `AuthenticationProvider`가 존재하면 해당 `AuthenticationProvider`를 통해 인증을 처리하게 된다. 만약 처리가능한 것이 없거나 실패한 경우 예외를 발생시킨다.  

**`ProviderManager`가 `AuthenticationProvider`를 선택하는 방식은 주로 인증방식에 따라 갈리게 된다.**{: .font-highlight} 즉, OAuth2.0이냐 Username/Password이냐에 따라 다른 `AuthenticationProvider`을 실행하게되는 것이다.  

**이를 통해 여러 유형의 인증을 지원하고 단일 `AuthenticationProvider` bean만을 노출하여 특정한 유형의 인증을 수행할 수 있다.**  

![SecurityContextHolder](https://docs.spring.io/spring-security/reference/5.7/_images/servlet/authentication/architecture/providermanager.png){: .align-center style="width: 60%;"}  
출처: [Spring Security 공식문서](https://docs.spring.io/spring-security/reference/5.7/servlet/authentication/architecture.html)
{: .image-caption style="font-size: 14px;" }  


ProviderManager는 다음과 같이 구현되어 있다. 아래적힌 코드뿐만 아니라 여러 기능을 하지만 그중 위에서 설명한 부분만 가져왔다. 
`AuthenticationProvider` 리스트를 초기화하고 클라이언트의 요청이 들어오면 `authenticate` 메서드가 실행되어 적절한 `AuthenticationProvider`를 찾아 실행한다.  
```java
public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean {

  private List<AuthenticationProvider> providers = Collections.emptyList();

  public ProviderManager(AuthenticationProvider... providers) {
    // ... providers 초기화
  }

  @Override
  public Authentication authenticate(Authentication authentication) throws AuthenticationException {
    Class<? extends Authentication> toTest = authentication.getClass();
    Authentication result = null;

    /*
      모든 AuthenticationProvider 순회
      - support 여부 확인
      - 지원가능하다면 처리 후 Authentication 받음
      - 처리가능한 provider가 있을때까지 반복
    */  
    for (AuthenticationProvider provider : getProviders()) {
      if (!provider.supports(toTest)) {
        continue;
      }

      try {
        result = provider.authenticate(authentication);
        if (result != null) {
          copyDetails(authentication, result);
          break;
        }
      }
      catch ( /*...*/ ) {
        // ...
      }
    }
    // ...
  }
}
```

<br />  

### AuthenticationProvider  
`ProviderManager`에서 설명했듯 **인증종류별 처리방식을 제공**{: .font-highlight}한다. 예를 들어, `DaoAuthenticationProvider`는 Username/Password 인증방식을 처리한다.  

<br />  

### AuthenticationEntryPoint
클라이언트가 능동적으로 Username/Password와 같이 인증정볼를 보내줄때는 상관이 없지만 인증되지 않은 유저가 권한이 없는 리소스에 접근을 한다면 자격 증명을 요청하는 응답을 주어야 한다. 이때 `AuthenticationEntryPoint` 인터페이스를 구현한 객체를 사용한다. **이 객체는 로그인 페이지로 리디렉션을 수행하거나 WWW-Authenticate 헤더등을 응답하여 인증을 요구**한다.  

즉, `AuthenticationEntryPoint`는 **인증되지 않은 클라이언트의 요청에 인증을 유도하는 역할**{: .font-highlight}을 한다. 인증되지 않은 채로 리소스에 접근 시 적절한 응답을 전달하여 인증을 유도한다.  

일반적으로 인증이 필요한 리소스에 접근할 때 호출되는 AuthenticationFilter에서 사용되어 인증여부를 확인하고 인증되지 않은 경우 `AuthenticationEntryPoint`를 호출한다.

<br />

### AbstractAuthenticationProcessingFilter
**인증 프로세스를 진행하는 모든 Security Filter의 부모 클래스**{: .font-highlight}이다. 사용자명과 패스워드를 사용한 폼 로그인을 처리하는 필터인 `UsernamePasswordAuthenticationFilter`, HTTP Basic 인증을 처리하는 `BasicAuthenticationFilter`의 부모가 `AbstractAuthenticationProcessingFilter` 이다.  

<br />  

### 요약
요약하자면 인증관련 Filter들은 다음과 같이 구현되어 있다.  

![spring_security_authentication_architecture](https://github.com/Nexters/keyme-backend/assets/52196792/d4bc4368-9891-4d19-a961-acb67ee47962){: .align-center style="width: 80%;"}  
Authentication Architecture
{: .image-caption style="font-size: 14px;" }  


<br />  

**참고**
- [Spring Security 공식문서 - Authentication Architecture](https://docs.spring.io/spring-security/reference/5.7/servlet/authentication/architecture.html#servlet-authentication-authentication)  
