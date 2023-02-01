---
title: "nest.js의 핵심개념과 life cycle"
categories: nestjs
tags:
  - backend
---

![nest](https://camo.githubusercontent.com/5f54c0817521724a2deae8dedf0c280a589fd0aa9bffd7f19fa6254bb52e996a/68747470733a2f2f6e6573746a732e636f6d2f696d672f6c6f676f2d736d616c6c2e737667){: .align-center style="width: 30%;"}  
출처: [nestjs](https://github.com/nestjs/nest)
{: .image-caption style="font-size: 14px;" }  

기존 Node.js를 이용한 서버 프레임워크는 대부분 Express를 많이 사용하였다. 하지만 Express의 최대 장점이자 단점은 자유도가 너무 높다는 것이었다. 프레임워크단에서 관리하는 레벨보다는 **개발자가 관리해야하는 영역이 너무 넓다**보니 개발팀마다 다양한 형태를 보였고 성능 문제나 효율성이 떨어지는 경우가 많았다. 이러한 문제를 보완하기위해 Nest는 **DI와 IoC의 개념을 차용, 프레임워크 레벨에서 목적별로 기능을 나누고 관심사를 분리**하였고 그렇게 *테스트하기 좋은 환경, 느슨한 결합, 확장성 있으며 높은 유지보수성*{: .gray}을 가지는 프레임워크로 탄생하였다.  

본격적으로 nest가 내제하고 있는 기능들을 알아보자. nest는 요청이 들어오고 응답을 처리할때까지 여러 과정을 거친다. 각 과정에서는 개발자가 처리할 내용을 정의할 수 있다.  
아래 그림은 nest가 제공하는 목적별 기능들을 흐름순대로 보여주는 이미지이다. 각 기능들은 모든 route에 글로벌하게 적용할 수도 있으며 특정 route에만 적용시킬 수도 있다. 이제 요청/응답의 사이클을 순서대로 알아보면서 nest의 핵심철학을 이해해보자.  

![nest_life_cycle](https://user-images.githubusercontent.com/52196792/214865870-77bdcdd5-6596-40d7-8bd3-256c8034ab69.png){: .align-center style="width: 90%;"}  
nest life cycle
{: .image-caption style="font-size: 14px;" }  

<br />  

## Nest request-response cycle  
[Nest 공식문서](https://docs.nestjs.com/)은 각 개념에 대해 친절하고 자세하게 알려주고 있다. 하지만 필자는 nest의 전체적인 흐름을 한 눈에 이해하고 확인하기가 어려웠다. 해서 흐름 순대로 개념을 정리하여 보다 쉬운 이해를 목표로 글을 작성하였다. 각 개념별 더 깊은 학습을 위해서 공식문서를 꼭 읽어보기를 바란다!

### 시작점
```bash
$ npm i -g @nestjs/cli
$ nest new project-name
```  
위 명령어를 통해 nest 프로젝트를 시작하면 `src`폴더 아래 `main.ts`파일이 보일 것이다. 해당 파일을 열면 아래와 같은 코드가 보인다.  

{% highlight ts linenos %}
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
{% endhighlight %}  

5번째 줄에 `NestFactory.create()` 메서드는 **Nest 어플리케이션 인스턴스를 생성**한다. 그리고 `listen()` 메서드를 통해 **지정한 포트로 웹서버를 구동**시킨다. 즉, 이 파일은 Nest 어플을 생성하고 진입점을 열어주는 역할을 한다.

추가적으로 `NestFactory.create()` 메서드의 인자로 AppModule을 넣어주는데 이에 관해서는 **실질적으로 데이터처리를 담당하는 `Module`, `Controller`, `Provider`의 개념을 설명할 때 다시 이야기하겠다.**

<br />  

### Middleware
Middleware(이하 미들웨어)는 요청이 들어와 엔드포인트로 접근 전 **가장 먼저 호출되는 함수**이다. 미들웨어를 이용해 아래와 같은 행위를 할 수 있다.  
- 미들웨어를 이용해서 **요청 객체를 조작하거나 응답객체를 미리 조작**할 수 있다.
- 그 자리에서 해당 **요청을 종료**시킬 수도 있다.
- 함수실행 후 다음미들웨어를 호출 또는 엔드포인트로 넘길 수도 있다.
- 인증이나 로깅, 데이터의 유효성 검사등을 미들웨어 처리한다.  

개발자들은 미들웨어를 두 가지 방식으로 만들 수 있다. 바로 클래스형과 함수형이다.
```ts
// 클래스형 미들웨어
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';


@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`Request made to ${req.url}`);
    next();
  }
}
```
`NestMiddleware` Interface를 implements하여 use 함수를 구현하였다. 여기서 `@Injectable()`데코레이터를 사용했다는 것을 기억하자. 해당 클래스는 의존성을 주입하여 사용될 것을 명식하는 데코레이터이다.  

```ts
// 함수형 미들웨어
import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`Request...`);
  next();
};
```
로깅과 같은 간단한 미들웨어는 받는 인자만 명시하여 함수형으로도 구현이 가능하다.  

이렇게 작성한 미들웨어들은 전역으로 적용할 수도 있으며 특정 엔드포인트에만 적용할 수도 있다. 각 사례를 예시코드를 통해 살펴보자.  
```ts
// 특정라우트 범위에 적용
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
```  
`NestModule` interface를 구현하는 클래스에서 `configure` 메서드 내에 `apply` 메서드를 통해서 미들웨어를 적용시킬 수 있다. 이 때 `forRoutes` 메서드로 적용시킬 라우트를 지정할 수 있다(와일드 카드 사용 가능) 여기서 함수형 미들웨어를 적용시키고자 한다면 `LoggerMiddleware` 대신 `logger`를 넣으면 된다.

**💡 여러 미들웨어 적용하기**  
apply 메서드 내에 ' , ' 로 구분하여 나열하면 된다.  
-> consumer.apply(logger1, logger2, logger2).forRoutes('cats')
{: .notice--info}  

```ts
// main.ts
const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(3000);
```  
미들웨어를 전역으로 적용시키고자 한다면 `main.ts` 파일에서 `listen` 메서드 전에 `use`메서드를 이용하여 미들웨어를 적용시킬 수 있다.  

<br />  

### Exception Filters
**코드에서 처리하지 않은 예외가 발생했을때**, 적절한 상태코드, 값으로 처리해서 응답해준다. 서론에 작성한 이미지를 보면 미들웨어를 제외하고 다른 기능들을 감싸고 있는 것을 볼 수 있다. 해당 기능들 안에서 **작업을 처리하다 예외가 발생하면 Exception Filter에서 정의된 예외들을 처리하여 응답한다.** 궁긍적으로 핵심은 **예외/에러 핸들링을 중앙집중화**시킬 수 있다는 것이다.  

사용방법은 미들웨어와 크게 다르지 않다.  
```ts
import { ExceptionFilter, Catch, HttpException, ArgumentsHost } from '@nestjs/common';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: Error, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```  
`ExceptionFilter` interface를 구현하는 클래스를 선언하고 `@Catch` 데코레이터를 통해 어떤 예외를 잡을 것인지를 정의한다. 그리고 `catch()` 메서드를 구현하여 해당 예외에 대해 어떻게 응답할 것인지를 작성한다. 이렇게 **예외처리 코드를 본래 처리 코드에서 분리하여 작성할 수 있다.** 효율적인 재사용성, 데이터처리 코드와의 디커플링을 가져갈 수 있다.  

Exception Filter도 마찬가지로 전역적으로 혹은 부분적으로 적용시킬 수 있다. 부분적으로 적용하고 싶다면 `@UseFilters()` 데코레이터를 사용하고 전역적으로 사용하고자 한다면 nest 어플리케이션에 `useGloblaFilters()`메서드에 등록하면 된다.  

```ts
// 부분적으로 사용하기
@UseFilters(new HttpExceptionFilter())
export class NewController {}
```  
```ts
// 전역으로 사용하기 - main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(3000);
}
bootstrap();
```  

**🤔 여러 Exception Filter 적용하기**  
' , '를 이용하여 각 Filter를 나열하면 된다.
{: .notice--info}  


<br />  


### Guards  
오직 **접근 권한(권한, 역할, ACL)에 관한 책임**만을 가지고 있다. 내/외부적인 인증, 인가, 보안과 관련된 확인을 진행한다. 각 Guard는 boolean 값을 리턴하는데 false일 경우 인증실패로 block된다.  

**💡 Guard와 미들웨어의 Authorization(권한 부여)에서의 차이점**  
미들웨어는 그 다음에 호출되는 코드에 대해 알 수가 없다. 그러나 Guard는 Execution Context에 접근이 가능하여 다음 실행될 항목을 알고 있다. request-response cycle 중 정확한 지점에 로직을 삽입/실행할 수 있다.  
{: .notice--info}  

Guard는 다음과 같은 방법으로 작성할 수 있다.  
```ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';

@Injectable()
export class AuthenticatorGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    return request.headers.authorization === 'SECRET_TOKEN';
  }
}
```  

`CanActivate` interface를 구현하는 class를 선언하고 `@Injectable()` 데코레이터를 사용한다. 경로로 요청이 들어갈때 Nest는 Guard의 `canActivate` 메서드를 호출한다. 이 메서드에서 인증이 성공인지 실패인지에 대해 boolean 값으로 리턴한다.  

Guard 또한 부분적, 전역적으로 사용가능하다.  
```ts  
// 경로별 부분적으로 사용하기
@Controller('cats')
@UseGuards(AuthenticatorGuard) // or @UseGuards(new AuthenticatorGuard())
export class CatsController {}
```  
```ts  
// 전역으로 사용하기 - main.ts
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new AuthenticatorGuard());
```  
```ts  
// 전역으로 사용하기 & 의존성 주입을 이용
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: AuthenticatorGuard,
    },
  ],
})
export class AppModule {}
```  

Guard는 Role(역할)을 위해서 강력한 기능을 제공한다. 역할을 지닌 사용자에게만 해당경로로 접근 가능하게하는 방법을 잘 지원해주고 있기에 [공식문서](https://docs.nestjs.com/guards)를 꼭 읽기를 추천한다.  


<br />  


### Interceptors  
요청 본처리 바로직전에, 응답 바로 직후에 액세스할 수 있는 기능이다. Interceptor는 **AOP의 기술에서 영감을 받은 유용한 기능들을 가지고 있는데** 공식문서에서 이야기하는 특징은 아래와 같다.  
- 요청, 응답에 대한 **로깅**
- 요청, 응답을 유저 친화적으로 **변형**시키기
- 기본 기능에서 **확장된 기능 추가**
- 특정 조건에 따라 함수를 **overriding**(캐시의 목적)  

Interceptors 아래와 같이 작성가능하다.  
```ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    const now = Date.now();
    return next.handle().pipe(map((data) => data));
  }
}
```  
`NestInterceptor` interface를 구현하고 `intercept()` 메서드 내에 로직을 작성한다. interceptor는 두번째 인자로 `CallHandler`을 받는데 이 인자의 `handle()` 메서드를 이용하여 원하는 지점에 route handler를 실행시킬수 있다. `handle()` 메서드는 Observable을 반환한다. 이를 이용해서 응답값을 조작할 수 있다. `handle()` 이후 체이닝을 통해 응답값을 조작, 에러처리 등을 진행할 수 있다.  

Interceptor는 `@UseInterceptors()` 데코레이터를 이용해서 특정 라우트에만 적용할 수 있고 `useGlobalInterceptors()` 메서드내에 작성하여 전역적으로 적용할 수도 있다.  

<br />  

### Pipes
**요청 데이터의 변형 및 평가 역할**을 한다. data validation과 data transformation을 위해 사용한다. 이 작업 후 요청을 처리하게 된다. 빌트인으로 제공되는 pipe도 있으며(대개 원시타입의 변형/평가) 직접 작성할 수도 있다.  

요청 `Body`나 `Query`, `Param`에 대해서 올바른 key, type으로 요청을 받았는지 확인하는데 사용한다.  
```ts
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```  

<br />  

### Controllers  
Controller는 **특정 라우트에 대한 요청과 응답을 처리**한다. Controller내부는 각 라우트들을 처리할 메서드들로 이루어져있다. Nest 프레임워크에 Controller라는 것을 알리고 의존성, 생명주기를 관리하기위해 클래스 위에 `@Controller()` 데코레이터를 붙인다.  

```ts
import { Controller, Get, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('data')
export class DataController {
  @Get()
  findAll(@Req() request: Request): string {
    return 'return data';
  }
}
```  
각 특정 라우트마다 HTTP 메서드, Param, Query, Body 등을 데코레이터를 이용해 정의할 수 있다. Controller 내 하나의 함수는 하나의 라우트를 의미한다.  

<br />  

### Providers
비즈니스 로직을 구현하고 수행하는 역할을 맡는 모든 개체들이다. service, repository, factory, helper 등 모든 기능들이 Provider가 될 수 있다. 이렇게 역할을 나누어 객체들을 쪼개지 않고 Controller에서 모든 로직을 처리할 수 있지만 코드가 길어지고 유지/보수, 재사용성이 매우 떨어진다. 반대로 역할별로 나누고 필요에 따라 연계하여 사용한다면 재사용성이 높아지고 유지/보수가 보다 수월해진다. 위와 같은 장점과 더불어 Nest의 Provider가 가지는 장점은 **의존성을 주입할 수 있다**는 것이다. 

**💡 의존성 주입(DI, Dependency Injection)**  
의존관계의 객체를 **외부에서 생성 및 주입함**으로써 결합도를 줄이고 유연한 코드를 만들 수 있다. 단순히 객체를 외부에서 주입하는 것 뿐만아니라 **객체가 아닌 인터페이스에 의존관계를 가지는 것**까지 DI라고 부른다.  
(Typescript에서는 컴파일 시 interface가 사라진다. Nest에서 인터페이스에 의존성을 가지기 위해선 Custom Provider를 사용해야한다.)  
{: .notice--info}  

**💡 제어의 역전(IoC, Inversion Of Control)**  
개발자가 직접 의존관계의 객체를 생성하고 주입하는 형식이 아닌 **생성과 주입의 제어권을 프레임워크(컨테이너)에 넘겨주는 것**을 의미한다. 관리에 대한 제어권이 개발자 -> 프로그램으로 역전된다고 하여 '제어의 역전'이라고 한다.
{: .notice--info}  


Provider를 구현하는 방법은 아래와 같다. 클래스에 `@Injectable()` 데코레이터는 DI가 가능하다는 것을 Nest IoC에게 알리는 역할을 한다.  
```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class DatabaseService {
  getData(): string {
    return 'some data';
  }
}
```  

<br />  

### Modules  
**Provider들간의 연관관계, 의존성을 구성해주는 역할**을 한다. 앞서 작성한 Controller와 Provider들을 Module내에 정의하여 연관관계를 Nest 프레임워크에 알린다. Nest 프레임워크는 이들을 조합해 **서로의 의존관계를 생성/주입**해준다. 모듈은 다른 모듈의 provider 또한 참조할 수도 있다. Module의 특징은 아래와 같다.
- `controllers`로 생성할 Controller 세트를 받는다
- `providers`로 주입될 Provider 세트를 인자로 받는다.
- `imports`로 다른 Module을 받는다(다른 Module에서 `exports`한 provider를 사용할 수 있다.)
- `exports`로 다른 Module에서 사용가능한 이 모듈의 provider를 정의한다.
- 위에 받은 인자들을 생성/주입하여 하나의 모듈을 만든다.  

아래는 Controller에서 Provider를 주입받고 Provider를 외부에서 사용가능하게 export하는 예시코드다.
```ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports:[CatsService]
})
export class CatsModule {}
```

<br />  

### Middleware vs Guard vs Interceptor
**Middleware** -> Authentication(인증 - 토큰 유효성검사), cors  
**Guard** -> Authorization(권한부여) 역할, ExecutionContext로 접근 가능  
**Interceptor** -> request/response 데이터 조작, Stream overriding(캐시처리)  

<br />  

### Decorators 참고
[Exploring EcmaScript Decorators](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841)  