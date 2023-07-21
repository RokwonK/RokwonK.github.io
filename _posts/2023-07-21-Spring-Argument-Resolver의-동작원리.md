---
title: "Spring - Argument Resolver"
categories: Spring
tags:
  - security
---  

Controller의 메서드에 파라미터로 사용하는 `@RequestParam`, `@RequestBody` 는 어떻게 동작하는 것일까? Controller의 파라미터로 클라이언트의 요청정보가 아닌 추가적인 정보를 넘길 수 있을까? 이러한 의문들을 해결하기 위해 `HandlerMethodArgumentResolver`에 대해서 알아보자.  


### HandlerMethodArgumentResolver
`HandlerMethodArgumentResolver`는 **Controller 메서드의 매개변수에 바인딩 되는 인자들을 처리하는 역할**{: .font-highlight}을 맡는다. 우리가 Controller 메서드에서 자주 사용하는 `@RequestParam`, `@RequestBody`, `@PathVariable` 등의 어노테이션들은 미리 구현된 `HandlerMethodArgumentResolver`로서 HTTP 요청에서 매개변수를 추출하여 Controller의 메서드로 바인딩되는 것이다.  

이를 통해 우리는 코드의 중복을 줄이고 매개변수를 처리하는 일반적인 로직을 재사용할 수 있다.  

<br />  

### 커스텀 ArgumentResolver 만들기  
`@RequestParam`이나 `@RequestBody` 같이 제공되는 어노테이션이 아닌 커스텀 MethodArgumentResolver를 만들기 위해서는 크게 두 가지 단계가 필요하다.  
> 1. `HandlerMethodArgumentResolver`를 구현하는 클새스 생성, 구현
2. WebMvc에 resolver로 등록  

클라이언트측에서 `Authorization` HTTP 헤더로 보낸 Jwt토큰을 파싱하고, 얻은 유저정보를 메서드 인자로 넘기는 Resolver를 구현해보자.  

먼저 `HandlerMethodArgumentResolver`를 구현하는 객체를 만들자.
```java
// 파라미터에 사용할 어노테이션 추가
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface RequestUser {
}

// `HandlerMethodArgumentResolver`의 구현체
@Component
public class UserInfoArgumentResolver implements HandlerMethodArgumentResolver {
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.getParameterAnnotation(RequestUser.class) != null && parameter.getParameterType().equals(UserInfo.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();

        if (!(principal instanceof UserInfo)) {
            throw new RuntimeException();
        }

        return principal;
    }
}
```  

`HandlerMethodArgumentResolver`를 구현하는 클래스는 두가지 메서드를 구현해야한다.
> 1. `supportsParameter` - 어떤 파라미터를 처리할 수 있는지
2. `resolveArgument` - 해당 파라미터로 넘겨줄 값 지정  

위 코드에서는 `@RequestUser`이며 `UserInfo` 타입인 파라미터만을 처리할 수 있도록 지정하였다. 그리고 Spring Security를 통해 밀 SecurityContext에 저장된 인증정보를 꺼내 값을 넘겨주었다.  

다음으로 이렇게 만들어진 Resolver를 등록해주어야 한다.  
```java
@Configuration
@RequiredArgsConstructor
public class WebMvcConfig implements WebMvcConfigurer {
    private final UserInfoArgumentResolver userInfoArgumentResolver;

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(userInfoArgumentResolver);
        WebMvcConfigurer.super.addArgumentResolvers(resolvers);
    }
}
```  
WebMvc 환경설정에서 작성한 Resolver를 추가해주면 된다.  

이제 Controller에서 `@RequestUser` 어노테이션을 이용하는 파라미터를 작성하면 요청 시 해당 파라미터에 값이 담겨진다.  

```java
@RestController
@RequestMapping("/members")
@RequiredArgsConstructor
public class MemberController {
  private final MemberService memberService;

  @GetMapping
  public Long getMemberId(@RequestUser UserInfo userInfo) {
    System.out.println(userInfo.getMemberId()); // memberId가 출력됨
    return userInfo.getMemberId();
  }
}
```

<br />  

### 처리시점
사용법을 알아봤으니 동작원리를 살펴보자 대체 어느시점에서 `HandlerMethodArgumentResolver`가 동작하는 것일까?  

`DispatcherServlet`의 동작방식을 기억하는가? `DispatcherServlet`은 클라이언트요청에 맞는 컨트롤러를 `HandlerMapping`에서 찾은 후 `HandlerAdapter`를 이용해 요청을 처리한다. (`HandlerAdapter`의 역할은 여러개의 처리방식을 동일한 인터페이스로 처리할 수 있도록 도와준다.)  

여기서 `HandlerAdapter`가 handler의 인자들을 보며 처리가능한 MethodArgumentResolver를 선택하여 처리한다. 

![HandlerMethodArgumentResolver](https://github.com/kids-ground/shout-iOS/assets/52196792/4b73de6e-9683-4767-b3cb-f571742a27cd){: .align-center style="width: 70%;"}  
MethodArgument의 처리시점
{: .image-caption style="font-size: 14px;" }  
