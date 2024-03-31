---
title: "ResponseBodyAdvice로 공통 응답 형식 적용시키기"
excerpt: "공통 응답 형식을 적용하고 싶은데 어떤 기능을 사용해야할까?"

categories:
  - Project

permalink: /project/apply-common-response/
---  

<br/>  

'개발 한 스푼'에서는 다음과 같은 형식으로 응답 Body를 보낸다.
```json
// 성공 - 객체 리턴
{
	"code": 200,
	"message": "Success",
	"data" : { } // 성공시 데이터
}

// 실패시
{
	"code": 400_001_003, // 내부적으로 유지하는 실패코드
	"message": "뭔가 잘못되었습니다."
}
```

성공과 실패 시 같은 응답형식을 취하고 `code`와 `message`를 통해 보다 자세한 실패 이유를 알려주기 위함이다.

실패시에는 등록된 ExceptionResovler에서 공통으로 처리하지만 정상적인 응답 시에는 수 많은 Controller 메서드에서 `SuccessResponse(data = ResponseObject)`와 같이 매번 래핑해서 사용할 수는 없는 노릇이다. 잘못하고 작성을 빼먹으면 더 큰일이다.(물론 테스트 코드가 남아있지만)

Controller에서는 Controller의 일만을 수행하게 냅두고 **공통형식은 Controller 밖에서 처리해보자.**

<br/>  

### 공통 응답 형식, 어디서 처리할 수 있을까?
가장 먼저 떠올랐던 것은 `Filter`와 `Interceptor`였다.

`Filter`는 유저의 요청을 가장 먼저 맞이하고 응답 시에는 가장 마지막에 통과하는 스프링 부트의 게이트웨이이다. **Filter에서 응답을 받았을때는 이미 응답 데이터가 모두 쓰여진 상태**다. 이는 Controller에서 리턴한 객체가 JSON으로 직렬화가 끝난 상태란 뜻이다. 이 데이터를 수정하기 위해서는 Response 캐싱부터 역직렬화, 수정, 다시직렬화까지 해줘야할 것들이 많다.

또한 공식설정상 Spring Container 외부에서 수행하기 때문에 단순 응답 Body 조작에서 사용하기에는 적절치 못하다 판단했다.

`Interceptor`의 경우에는 Spring Container 내에서 동작하지만 Controller에서 return값을 null로 넘기지 않는이상 **응답 데이터를 조작할 수 없다.**

또 다른 방법이 없을까하여, 요청부터 응답까지의 흐름에 사고의 템포를 맞춰 고민하다 보니 **HandlerAdapter 레벨에서 처리할 수 있는 방법**이 없을까? 하는 생각이 들었다. 유레카! 역시나? `ResponseBodyAdvice`를 통해 Controller 응답값을 제어할 수 있는 방법이 존재하였다.(~~GPT에게 물어봤다면 더 빨리 찾았을테지만!~~)

<br/>  

### ResponseBodyAdvice
`ResponseBodyAdvice`는 인터페이스로 두가지 메서드를 필수로 구현해야한다. 첫번째는 해당 객체를 적용할 대상인지를 체크하는 `supports`이고 다른 하나는 body를 조작할 수 있는 `beforeBodyWrite` 메서드이다.
```kotlin
boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType);

@Nullable  
T beforeBodyWrite(@Nullable T body, MethodParameter returnType, MediaType selectedContentType,  
      Class<? extends HttpMessageConverter<?>> selectedConverterType,  
      ServerHttpRequest request, ServerHttpResponse response);
```

해당 인터페이스를 구현하고 `@ControllerAdvice` 어노테이션을 붙이면 자동으로 등록된다. 글로벌 예외처리 클래스에 사용하던 그 어노테이션과 똑같다. Controller와 AOP에서 익숙한 용어인 Advice의 합성어로 Controller에 적용되는 부가적인 기능이라고 생각하면 된다.

등록된 `ResponseBodyAdvice`는 HandlerAdapter에서 사용된다. Controller 메서드(핸들러메서드)에서 응답값을 리턴 받으면 해당 응답에 적용가능한지(supports 메서드 실행) 확인 후 적용한다.

정리하자면, FilterChain에서 `Filter`가, DispatcherServlet에서 `Interceptor`가, HandlerAdapter에서 `Request/ResponseBodyAdvice`가 요청시(응답시 역순) 처리되는 것이다. (BodyAdvice에 대한 보다 자세한 동작흐름은 다음 포스트에 작성되어있다.)

ResponseBodyAdvice를 사용하여 응답 데이터를 조작할 때 이점은 다음과 같다고 생각한다.
1. **Spring Container 내에서 응답 조작가능**
2. **제공 기능과 필자의 의도가 매칭된다. -> 가독성이 좋다.**

<br/>  

### ResponseBodyAdvice로 공통 응답형식 적용하기
공통 응답형식을 가지는 class를 먼저 정의해보자. Controller에서 리턴되는 응답값들은 모두 data 필드로 들어갈 것이다.
```kotlin
data class SuccessResponse(  
    val code: Int = 200,  
    val message: String = "Success",  
    val data: Any?  
)
```

이제 `ResponseBodyAdvice`를 정의하자. 우선 모든 응답에 대해 `SuccessResponse`로 래핑할 예정이기 때문에 supports 는 무조건 true로 지정한다.(어떤 문제가 벌어질지도 모른 채.. )

`beforeBodyWrite`에서는 status가 성공인 경우에만 `SuccessResponse`로 래핑해준다. 사실 비즈니스 로직에서 실패의 경우, 예외를 던지는 케이스만 있기 때문에 status 확인 로직이 필요할까 고민했지만, 혹시나 명시적 실패를 200번 응답에 작성할 수도 있음을 고려하여 확인로직도 추가하였다.
```kotlin
@RestControllerAdvice
class ResponseAdviceHandler: ResponseBodyAdvice<Any> {  
    override fun supports(returnType: MethodParameter, converterType: Class<out HttpMessageConverter<*>>): Boolean {  
        return true
    }  
  
    override fun beforeBodyWrite(  
        body: Any?,  
        returnType: MethodParameter,  
        selectedContentType: MediaType,  
        selectedConverterType: Class<out HttpMessageConverter<*>>,  
        request: ServerHttpRequest,  
        response: ServerHttpResponse  
    ): Any? {  
        val servletResponse = (response as ServletServerHttpResponse).servletResponse  
        val status = servletResponse.status  
        val resolve = HttpStatus.resolve(status) ?: return body  
  
        return if (resolve.is2xxSuccessful) {  
            SuccessResponse(data = body)  
        } else body  
    }  
}

```

잘 작동한다!

<br/>  

### 예상치 못한 문제
잘 동작하는 와중 Controller에서 객체가 아닌 문자열을 리턴하는 API를 만들었다. API를 테스트하는 도중 왠열 에러가 발생했다. 다음과 같은 에러 코드와 함께..

```text
message: class com.*.*.*.SuccessResponse cannot be cast to class java.lang.String (com.*.*.*.SuccessResponse is in unnamed module of loader 'app'; java.lang.String is in module java.base of loader 'bootstrap')
type: ClassCastException
```

문자열 리턴 후 공통응답형식을 적용하는 것으로 인해 발생하는 문제였다. 이에 대한 해결과정은 다음 포스트에서 작성한다.  

<br/>