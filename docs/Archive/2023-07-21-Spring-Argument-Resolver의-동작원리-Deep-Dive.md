---
layout: default
parent: Archive
title: "Spring - Argument Resolver와 동작원리 Deep Dive"
categories: Spring
tags:
  - deep dive
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

### 동작원리
사용법을 알아봤으니 동작원리를 살펴보자 대체 어느시점에서 `HandlerMethodArgumentResolver`가 동작하는 것일까?  

![HandlerMethodArgumentResolver](https://github.com/kids-ground/shout-iOS/assets/52196792/65cfce1b-981d-458d-a079-c309c7780a0f){: .align-center style="width: 80%;"}  
HandlerMethodArgumentResolver의 동작원리
{: .image-caption style="font-size: 14px;" }  


`DispatcherServlet`의 동작방식을 기억하는가? `DispatcherServlet`은 클라이언트요청에 맞는 컨트롤러를 `HandlerMapping`에서 찾은 후 `HandlerAdapter`를 이용해 요청을 처리한다. (`HandlerAdapter`의 역할은 여러개의 처리방식을 동일한 인터페이스로 처리할 수 있도록 도와준다.)  
```java
// DipatcherServlet 클래스 내부 - doDispatch 메서드
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
  // ...중략
  try {
    try {
      processedRequest = checkMultipart(request);
      multipartRequestParsed = (processedRequest != request);

      // Request와 알맞는 HandlerMapping 가져오기
      mappedHandler = getHandler(processedRequest);
      if (mappedHandler == null) {
        noHandlerFound(processedRequest, response);
        return;
      }

      // HandlerAdapter 가져오기
      HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
      // ...
      // Adapter 실행
      mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
    }
  }
  // ...
}
```

이때 우리가 요청처리 handler를 구성할때 `@RequestMapping` 어노테이션을 이용했다면(아마도 REST 서버를 구성했다면) `HandlerAdapter` 중 `RequestMappingHandlerAdapter` 가 동작한다.  

여기서 재밌는 점은 우리가 작성한 핸들러들은 보통 메서드 단위이다. 때문에 메서드를 가지는 컨트롤러의 정보를 포함하여 리턴 타입 정보, Method의 Parameter 정보를 담고 있는 `HandlerMethod`를 `DispatcherServlet` 으로부터 제공받는다.  

`RequestMappingHandlerAdapter`에서는 HandlerMethod를 `InvocableHandlerMethod`로 래핑한다. 그리고 이 객체의 인스턴스 메서드인 `invokeAndHandle()`를 통해 요청을 실행한다.  
```java
// RequestMappingHandlerAdapter 클래스 내 - invokeHandlerMethod 메서드
protected ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
  // ...
  try {
    ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);

    // ...

    // InvocableHandlerMethod 인스턴스의 invokeAndHandle
    invocableMethod.invokeAndHandle(webRequest, mavContainer);
    
    return getModelAndView(mavContainer, modelFactory, webRequest);
  }
  // ...
}
```

`invokeAndHandle()` 메서드에서는 `invokeForRequest()`를 메서드를 실행한다. 이 메서드에서 드디어 우리가 원하던 `HandlerMethodArgumentResolver`들이 동작하게 된다. 메서드가 가지고 있는 Parameter 정보들을 순회하며 각 Parameter마다 처리가능한 `HandlerMethodArgumentResolver`를 선택하고 수행한다.  
```java
// ServletInvocableHandlerMethod은 클래스 내 - invokeAndHandle 메서드
// ServletInvocableHandlerMethod은 invokeHandlerMethod을 상속받은 클래스이다.
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
  // 요청처리 실행
  Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);

  // ...
}
```
```java
// InvocableHandlerMethod 클래스 내 - invokeForRequest
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
  // Method Argument들을 Resolver로 처리하고 return 받음
  Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
  if (logger.isTraceEnabled()) {
    logger.trace("Arguments: " + Arrays.toString(args));
  }
  return doInvoke(args);
}
```  

처리할때는 `InvocableHandlerMethod`가 가지고 있는 `resolvers` 인스턴스 변수를 사용한다. 이 `resolvers` 변수는 `HandlerMethodArgumentResolverComposite` 타입을 가진다. 이 타입이 무었이냐면 이름에서 유추할 수 있듯이 `HandlerMethodArgumentResolver`의 모음집들이다. 즉 `resolvers` 변수를 통해서 메서드 Parameter와 매핑되는 `HandlerMethodArgumentResolver`를 고르고 알맞은 타입으로 변환시켜 준다.  
```java
// InvocableHandlerMethod 클래스 내 - getMethodArgumentValues
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {

  MethodParameter[] parameters = getMethodParameters();
  if (ObjectUtils.isEmpty(parameters)) {
    return EMPTY_ARGS;
  }

  // 파라미터들을 순회하며 Resolver 실행
  Object[] args = new Object[parameters.length];
  for (int i = 0; i < parameters.length; i++) {
    MethodParameter parameter = parameters[i];
    parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
    args[i] = findProvidedArgument(parameter, providedArgs);
    if (args[i] != null) {
      continue;
    }

    // 알려진것처럼 support되는지 확인하고 Argument를 Resolve
    if (!this.resolvers.supportsParameter(parameter)) {
      throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
    }
    try {
      args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
    }
    catch (Exception ex) {
      // ...
    }
  }
  return args;
}
```

`InvocableHandlerMethod` 객체는 위와 같이 실제 요청처리의 전처리(메서드 파라미터 바인딩 등)를 모두 마친 뒤 실제 요청메서드를 실행시킨다.  




메서드 호출 수준까지 내려가 이야기하다보니 내용이 길어졌는데 요약하자면 다음과 같다.
> 1. **DispatchQueue**
  - `doDispach()` 메서드 내부에서 요청에 대한 적절한 `HandlerMapping` 및 `HandlerAdapter`를 찾음
  - `@RequestMapping` 어노테이션을 이용하여 어플리케이션을 구성했다면 `RequestMappingHandlerAdapter`를 이용하게 됨
  - `HandlerAdapter`의 `handle()` 메서드 실행
2. **RequestMappingHandlerAdapter**
  - 위 설명부분 에서는 빠졌지만 `handle()` 메서드는 `handleInternal()` 메서드를 실행하고 이 메서드에서 `invokeHandlerMethod()` 실행
  - `HandlerMethod`를 `InvocableHandlerMethod`로 래핑 후 `invokeAndHandle()` 메서드 실행
3. **InvocableHandlerMethod**
  - `invokeAndHandle()`에서는 `invokeForRequest()` 메서드를 실행하면서 요청처리
  - `invokeForRequest()`에서는 실제 요청처리 전 메서드 인자들을 처리
  - `resolvers` 변수를 통해 메서드 인자마다 처리가능한 `HandlerMethodArgumentResolver`를 사용하여 메서드 인자와 바인딩
  - 이후 메서드 실행하여 요청 처리
