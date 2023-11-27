---
layout: default
parent: Archive
title: "Typescript 배경지식"
categories: JavaScript
tags:
  - nodejs
  - Javascript
  - Typescript
---  


Javascript는 프론트에서나 백엔드에서나 유용하게 쓰이는 프로그래밍 언어이다. 하지만 Javascript는 동적 타입 언어이기 때문에 변수에 타입이 지정되어 있지않아 **타입에러가 쉽게 발생하며 에러를 발견하기도 쉽지 않다.** 다수의 개발자가 협업 시 더욱 자주 발생하며 결국 안정성과 개발속도를 모두 떨어트리는 원인이 된다. 이러한 단점으로 인해 Javascript를 확장한 Typescript 사용이 각광받고 있다.  

TypeScript란 무엇일까? Javascript에 타입을 부여한 언어이다. **타입을 사용함으로써 타입관련된 에러를 사전에 예방**할 수 있다. 뿐만 아니라 개발 툴의 기능(자동완성 및 가이드 기능)을 활용하여 **개발 생산성을 향샹** 시킬 수 있다.  

## Typescript의 컴파일, 동작방식
기본적으로 node.js는 `.ts`파일을 이해하지 못한다. 따라서 node.js가 이해할 수 있는 `.js`파일로 변경해줘야 한다. 이 이러한 과정을 **`compile(컴파일)`**이라고 부른다. Typescript 컴파일러는 코드를 분석하여 `.d.ts` 타입 정의 파일, `.js.map`, `.js` 파일로 컴파일한다. Typescript 컴파일러로는 [tsc](https://github.com/microsoft/TypeScript), [swc](https://github.com/swc-project/swc)가 있고 [Babel에서 플러그인 형태](https://babeljs.io/docs/en/babel-plugin-transform-typescript)로 지원한다.  

보통 컴파일러는 **타입 체크(문법 검사)**와 Typescript에서 **Javascript로의 컴파일**뿐만 아니라 **브라우져 호환성을 위한 ES6+ 문법을 이하 버전으로 바꿔주는 트랜스파일링 기능도 지원**해준다.  

**💡 컴파일**  
C언에서 컴파일은 소스코드를 CPU가 이해할 수 있는 저수준 언어로 바꿔주는 것을 의미한다. Typescript의 컴파일은 node.js가 이해가능한 js 언어로의 변환을 의미한다. 두 행동의 결과는 다르지만 **인식 가능한 코드로의 변환**한다는 점에 공통점을 갖는다.
{: .notice--info}  

**💡 트랜스파일**  
앞서 설명한 컴파일보다는 좁은 의미를 가진다. 컴파일이 저수준으로의 코드변환이라면 트랜스 파일은 **비슷한 수준의 추상화를 가진 다른언어**로의 변환을 말한다.(ex. ES6에서 ES5로)
{: .notice--info}  

TypeScript 컴파일 시 동작과정(tsc의 동작과정)을 알아보자.([블로그 참고](https://velog.io/@sehyunny/how-ts-compiler-compiles)) 먼저, 전체적인 흐름은 보자면 아래와 같다.  
1. **Program 객체를 생성, `tsconfig.json`에 작성된 설정을 기반으로 컴파일할 코드를 로드한다.**
2. **Program 객체를 이용해 Parser 과정을 거쳐 AST를 만든다.**
3. **만들어진 AST를 이용해 Binder 과정을 거쳐 Symbol Table(Type checking과정에서 사용)을 만든다.**
4. **이후 Program 객체가 emit 메서드를 실행하면서 Type checking이 이뤄진 후 `.js` 등 결과물을 내뱉는다.**  

![](https://raw.githubusercontent.com/huytd/everyday/master/_meta/tsc-overview.png){: .align-center style="width: 50%;"}  
출처: [TypeScript / How the compiler compiles](https://www.huy.rocks/everyday/04-01-2022-typescript-how-the-compiler-compiles)
{: .image-caption style="font-size: 14px;"}  


### 1. Program 객체 생성
Program 객체는 `tsconfig.json` 파일에 명시한 컴파일 옵션과 소스코드를 가지는 객체이다. 이 객체를 기준으로 아래 과정들이 실행된다.  


### 2. Parser 과정
Parser에서 두 과정이 일어난다. Program 객체로 로드는 **소스코드를 scanner를 통해 Token Stream**을 만들고 **parser를 통해 AST(Abstract Syntax Tree)를 만든다.**  
쉽게 코드를 Token으로 분리하고 분리된 Token을 연관관계에 따라 Tree 형식으로 만드는 과정이다. 


### 3. Binder 과정
AST를 이용하여 타입 검사단계에서 사용될 Symbol Table을 구성하는 단계이다. AST에 존재하는 각 Node들의 정보를 가지고 있다. `Node : Node의정보(Symbol)`에 대한 정보를 테이블(map) 형태로 가지고 있는다.  


### 4. Emit 과정
`Program`객체의 `emit` 메서드에 의해서 실행된다. Emit 과정도 두 단계로 나눠서 설명할 수 있다. 바로 **check와 emit이다.** 먼저, `getDiagnostics` 메서드가 호출되며 check를 시작한다. 이 과정에서 Symbol Table을 참조하여 타입검사를 한다.  
emit 과정에서는 ts파일을 `.js`파일로 바꾸어주고 ts파일에 대한 `.d.ts` 선언파일을 만들어 준다. 설정에 따라 `.js.map`파일(ts와 js 소스코드의 관계, json형식) 도 만들어준다.  

각 과정별 객체들의 더 자세한 역할이 궁금하다면 [이 블로그](https://basarat.gitbook.io/typescript/overview)를 참고해보길 권한다.  

<br />  

### 여러가지 컴파일러
Typescript를 js로 바꿔주는 컴파일러는 tsc 뿐만아니라 babel, swc 등이 존재한다. 어떤 컴파일러가 더 좋은가?에 정답은 없지만 선택은 해야한다.  간략하게나마 각 컴파일러의 특징을 정리해보았다.  

**tsc**
- 가장 많이 사용하는 컴파일러
- js변환 시 타입체킹은 물론 설정에 따라 ES6+ 문법을 트랜스파일링해줌
- 백엔드 프레임워크에서 대부분 사용(브라우저 호환성, DOM 구성 등의 문제가 있지 않으므로)


**babel**
- babel 7버전 이상부터는 타입스크립트의 컴파일러 역할을 할 수 있게 되었다.
- 단, 컴파일 시 **타입검사를 하지 않는다는 단점**이 존재한다.(플러그인이 존재하긴 함)  
- ECMAScript의 버전이 아닌 **브라우저의 리스트를 기준으로 트랜스파일링**(더 세분화된 하위 호환성을 제공, polyfill) 
- 타입체킹 문제로 tsc와 bable을 병렬로 사용하는 경우가 많음(tsc로 타입체킹 babel로 트랜스파일링)
- 7.16 버전이상부터는 typescript의 namespace, const enum의 기능도 트랜스파일이 가능함
- workflow의 확장성이 좋음(플러그인, 프리셋을 추가함으로써 역할 추가가 수월)


**swc**
- 현존하는 **가장 빠른 Typescript 컴파일러**
- babel과 마찬가지로 트랜스파일을 중점으로 하는 컴파일러
- 매우빠른 빌드속도를 장점으로 가지지만 확장성은 babel에 비해 상대적으로 떨어짐  

<br />  

## tsconfig.json
`tsconfig.json`은 typescript 설정파일이다. ts파일들을 `.js`파일로 변환시에 어떤것들을 어떻게 변환할 것인지 컴파일 속성에대해 세부적으로 설정이 가능하다. 상당히 많은 수의 옵션이 존재하는데 프로젝트 상황에 맞춰 옵션을 추가하면 좋을 것 같다. 아래에 정리된 옵션 이외에도 더 많은 수의 옵션이 존재한다. [참고](https://typescript-kr.github.io/pages/compiler-options.html)
```json
{
  "compilerOptions": {
    "module": "commonjs", // 사용할 js 모듈시스템 
    "declaration": true, // .d.ts 파일 생성
    "removeComments": true, // 주석삭제
    "emitDecoratorMetadata": true, // 데코레이터를 위한 메타데이터 지원 여부
    "experimentalDecorators": true, // 데코레이터 지원 여부
    "allowSyntheticDefaultImports": true, // default export가 없는 모듈에서 default imports를 허용
    "target": "es2017", // 사용할 ECMAScript 버전
    "sourceMap": true,  // 컴파일시 sourceMap 생성
    "outDir": "./dist", // 컴파일 결과물 디렉터리
    "baseUrl": "./",  // 기본 경로 설정
    "paths": {
      "@src/*": ["./src/*"] // 절대경로 Alias
    },
    "incremental": true, // 증분 컴파일을 활성화
    "skipLibCheck": true, // 선언파일(*.d.ts)의 타입검사를 건너뜀
    "strictNullChecks": true, // null, undefined에 대해 엄격한 검사
    "noImplicitAny": false, // any 타입에 대한 오류여부
    "strictBindCallApply": false, // bind, call, apply 메서드에 대해 엄격한 검사
    "forceConsistentCasingInFileNames": false, // 동일파일참조에 대해 일관성 검사
    "noFallthroughCasesInSwitch": false, // 스위치 문의 fallthrough 케이스에 대한 오류 보고
    "esModuleInterop": true // ES6 모듈 사양을 준수하여 Common JS 모듈을 가져올 수 있음, export default가 없는 라이브러리도 * as를 사용하지 않고 불러올 수 있음
  },
  "include" : [
    "src/**/*"
  ],
  "exclude": [
    "node_modules/**/*",
    "**/*.spec.ts"
  ]
}
```  
`compilerOptions`는 컴파일러의 기본값을 지정한다. `include`는 컴파일에 포함할 파일규칙이다. 지정하지 않으면 `exclude`에 명시하지 않은 모든 파일을 대상으로 한다. `exclude`은 컴파일에 제외할 파일규칙이다. 지정하지 않으면 기본적으로 outDir 디렉터리, node_modules, jspm_packages, bower_components를 제외한다.  
