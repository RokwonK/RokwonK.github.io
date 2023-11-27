---
layout: default
parent: Archive
title: "nest.js의 Execution Context"
categories: nestjs
tags:
  - backend
---



Nest에서 **현재 진행중인 환경에 대한 정보를 가져올 수 있는 유틸리티 클래스들을 제공**해준다. 대표적으로 `ArgumentsHost`와 `ExecutionContext` 클래스가 존재한다.  

### ArgumentsHost
`ArgumentsHost` 클래스는 **핸들러에 전달되는 인수를 확인할 수 있는 메서드들을 제공**해준다. 여기서 인수는 context(실행환경 e.g., HTTP, RPC, WebSocket 등), request, response 등을 포함한다. 이 클래스는 `exception filter`을 구현할때 `catch()`메서드의 인자로 받는것을 볼 수가 있다.  

이 클래스는 **멀티 컨택스트 환경에서 매우 유용하게 사용**할 수 있다. 멀티 컨택스트란, HTTP, RPC, GraphQL 등의 환경을 1개이상 사용하는 환경을 말한다. `exception filter`에서 사용하는 예시를 봐보자.

```ts
import { ExceptionFilter, Catch, HttpException, ArgumentsHost } from '@nestjs/common';

@Catch(CustomException)
export class CustomExceptionFilter implements ExceptionFilter {
  catch(exception: Error, host: ArgumentsHost) {
    if (host.getType() === 'http') {
      // http error response
    } else if (host.getType() === 'rpc') {
      // rpc error response
    } else if (host.getType<GqlContextType>() === 'graphql') {
      // graphql error response
    }
  }
}
```
멀티 컨택스트에서는 각 context마다 응답방법이 다를 것이다. 다수의 context에서 발생하는 예외를 처리할때 위와 같이 `ArgumentsHost`에서 context를 받아와 분기처리를 할 수 있다.  

`ArgumentsHost`로부터 핸들러(Controller 메서드)로 전달되는 인자들을 받아오는 몇가지 방법이 있다.
1. `getArgs()` 메서드를 이용하여 모든 인자를 배열로 받는 방법
2. `getArgByIndex()` 메서드를 이용하여 인자배열 중 해당 Index의 값만 받는 방법
3. `switchToHttp()` 메서드로 해당 context로 전환 후 각 conext에 맞는 메서드로 인자를 받는 방법  

위 방법들을 코드로 보자면 아래와 같다. 각 context을 알고 있다면 3번의 경우가 선언적이기에 가독성, 안정성이 뛰어나다.
```ts
// 인자 배열로 받기
catch(exception: Error, host: ArgumentsHost) {
  const [reqest, response, next] = host.getArgs();
}
```  
```ts
// 인자 배열 중 Index로 값 추출
catch(exception: Error, host: ArgumentsHost) {
  const request = host.getArgByIndex(0);
  const response = host.getArgByIndex(1);
}
```  
```ts  
// 각 context로 전환 후 메서드 이용하기
catch(exception: Error, host: ArgumentsHost) {
  const context = host.switchToHttp(); // HttpArgumentsHost
  const response = context.getResponse<Response>(); // HttpArgumentsHost의 메서드를 이용
}
```  

**Notice :** switchToHttp()뿐 아니라 switchToWs(), switchToRpc() 메서드도 지원한다.
{: .notice}


<br />  


### ExecutionContext  
`ExecutionContext` 클래스는 `ArgumentsHost`를 상속받은 클래스이다. 말 그대로 `ArgumentsHost` 보다 확장된 기능을 제공한다. **인자뿐 아니라 어떤 클래스(Controller)의 어떤 메서드(핸들러)에서 실행되고 있는지의 정보**까지 확인가능하다. 이 클래스는 `guard` 나 `interceptor`를 구현할때 인자로 받는 것을 확인할 수 있다.  

```ts  
// Guard에서 ExecutionContext
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';

@Injectable()
export class AuthenticatorGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const excuteMethodName = context.getClass().name; // 실행될 메서드 이름 
    const excuteClassName = context.getHandler().name; // 실행될 클래스 이름 

    return true;
  }
}
```  

이러한 기능은 `Reflector`를 이용하여 **클래스나 메서드에 주입된 메타데이터 정보에도 접근**할 수 있게 한다. 각 클래스나 메서드에는 `@SetMetadata()` 데코레이터를 이용해서 메타데이터를 설정할 수 있는데 `Reflector`와 `ExecutionContext`로 이러한 메타데이터에도 접근할 수 있다.  

**💡 Reflector**  
'@nestjs/core' 패키지에서 제공하는 유틸리티 클래스이다. Nest 어플리케이션의 프로퍼티, 파라미터, 클래스의 메타데이터를 가져올 수 있는 방법을 제공해준다.
{: .notice--info}  

`Reflector` 클래스는 `contructor`에 추가함으로써 쉽게 주입받을 수 있다.  
```ts
@Injectable()
export class AuthenticatorGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean { 
    return true;
  }
}
```  

`Reflector` 클래스를 이용해서 메타데이터를 가져올 수 있는 방법은 아래와 같다. 인자로 메타데이터 키와 가져올 클래스/메서드의 참조값을 주면 받아올 수 있다. 이때 참조값을 `ExecutionContext`에서 `getHandler()` 메서드를 통해 가져올 수 있다.  
```ts
@Injectable()
export class AuthenticatorGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get<string[]>('roles', context.getHandler());
    return true;
  }
}
```  
