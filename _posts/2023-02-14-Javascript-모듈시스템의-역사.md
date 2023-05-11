---
title: "Javascript 모듈 시스템의 역사"
categories: JavaScript
tags:
  - nodejs
  - Javascript
---  

모듈이란 무엇인가? 모듈은 애플리케이션을 구성하는 개별적 요소로서 **재사용 가능한 코드 조각**을 말한다. 보통 모듈은 기능을 기준으로 파일로 분리한다. 단순 분리뿐 아니라 각 모듈파일은 **자신만의 파일 스코프를 가질 수 있어야 하며, 필요한 부분만을 노출시켜 외부에서의 접근범위를 제한할 수 있어야한다.(복잡성 줄이기)** 과거 Javascript 생태계에는 모듈의 개념이 존재하지 않았다. html파일의 `<script>` 태그로 각 파일을 불러 올 수 있지만 하나의 전역 스코프에서 관리되었다. **즉, 모듈별로 파일 스코프를 가지지 않았다는 의미**이다. 이는 파일이 달라도 변수들이 하나의 스코프에 존재하기 때문에 변수명이 겹치는 문제가 발생하고 프로젝트가 커질수록 제어하기 힘들어졌다. 이러한 불편함은 자연스럽게도 Javascript 생태계에 모듈의 개념을 도입하고자 하는 시도로 이어졌다.  

<br />  

## JS 모듈 시스템의 역사
ECMAScript에서 ES6(ES Module)를 발표하기 전까지 표준 모듈 시스템이 존재하지 않았다. 하지만 **모듈 시스템은 재사용성 및 확장성, 유지보수성 등 개발 생산성에 지대한 영향을** 끼치기 때문에 개발자들은 자체적으로 생태계에 모듈시스템을 만들기 시작했다.  

<br />  

### IIFE(즉시실행함수)
결국 전역 스코프에서 모든 파일이 추가되니 최대한 전역 스코프의 오염을 방지하기 위해 IIFE를 사용하였다. IIFE는 정의 즉시 실행되기 때문에 IIFE 내부로의 접근을 막을 수 있다.(접근범위 제한) 오직 이 함수에서 리턴되는 객체를 이용해서만 내부에 접근할 수 있다.  
```js
var example = (function() {
  function executeExample(value) { console.log(value); }
  return { execute: executeExample };
})()
```  
하지만 독자적인 파일 스코프를 가지지 않아 사실상 파일분리 수준밖에 되지 않는다.  

<br />  

### 형식이 존재하는 모듈시스템 - AMD  
AMD(Asynchronous Module Definition, 비동기적 모듈 선언)는 모듈로써 사용하기 위해 형식이 존재한다. `define` 함수를 이용해 선언하고 `require`를 이용해 사용하는 형식이다. **비동기적으로 스크립트를 실행하기 때문에 브라우저에서 효율이 좋다.**(DOM 구성 중 블록되지 않기 때문에) `defind()` 함수가 파일 스코프의 역할을 대신하여 전역 스코프에서 모듈을 분리해준다.  
(ex. AMD 구현 모듈로더 - RequireJS)

require.js를 다운받고 `define`으로 모듈을 정의, `require`로 모듈을 불러와 사용할 수 있다. 
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <title>Document</title>
</head>
<body>
  <script src="require.js"></script>
</body>
</html>
```

```js
// aModule.js
define(['a', 'b'], function(a, b) {
  function printValue() {
    console.log('hello');
    console.log(a);
    console.log(b);
  }
  return { hello : printValue, a: a, b: b };
});

// 사용할때
require(['aModule'], function (aModule) {
  aModule.printValue();
})
```  

<br />  

### 서버사이드를 위한 모듈시스템 - CommonJS  
CommonJS는 서버사이드(node.js)에서 사용하기 위해 만들어졌다. `module.exports`를 이용하여 모듈화 시키고 싶은 값들만 묶어서 내보내고 `require`로 불러와서 사용할 수 있다. CommonJS는 모든 파일이 로컬에 존재한다는 것을 전제로 한다.(동기적인 실행) `curl.js`를 이용해서 브라우저 환경에서도 사용할 수 있다.  

node.js 백엔드 개발자라면 `require`로 모듈을 불러오는 방식은 매우 익숙할 것이다. CommonJS는 ECMAScript가 지정한 표준은 아니지만 node.js 진영에서는 사실상 표준으로 사용하고 있다. 
```js
// aModule.js
module.exports = {
  a: 'a',
  b: 'b',
};

// 사용시 
const aModule = require('aModule');
console.log(aModule.a);
```

<br />  

### 통합을 위한 모듈시스템 - UMD
AMD, CommonJS와 같이 모듈시스템이 등장하였지만 문제는 **두 시스템이 호환이 되지 않는다는 것**이었다. 이 문제를 해결하고자 UMD(Universal Module Definition)가 등장하게 되었다. 어떤 시스템을 쓰던 간에 동작하게 한 것이다.  
사실 UMD는 디자인 패턴에 가깝다고 얘기를 한다. **모든 경우의 시스템을 커버할 수 있도록 모듈을 작성하는 것**이다.  

```js
// root : 전역, factory : 모듈을 선언하는 함수
(function (root, factory) {
	// AMD
  if (typeof define === 'function' && define.amd) {
    define(['a'], factory);
	// CommonJS
  } else if (typeof exports === 'object' && module.exports) {
    module.exports = factory(require('a'));
	// Browser globals(window 객체)
	} else {
    root.returnExports = factory(root.a);
  }
}(this, function (a) {  
  return { a : a };
}));
```  

<br />  

### 표준의 등장 - ESM(ES6)  
드디어 ECMAScript에서는 모듈 시스템을 통합시키고자 표준 모듈 시스템을 만들었다. 바로 ESM(ES6)이다. 프론트에서는 `<script>` 태그에 `type="module"` 어트리뷰트를 추가하면 로드된 Javascript 파일은 ESM으로써 동작한다. 또한 일반적인 Javascript 파일이 아닌 ESM임을 명확히 하기위해서 파일 확장자는 `.mjs`를 사용할 것을 권장하고 있다.  

ESM은 파일 자체의 **독자적이 모듈 스코프를 제공**하며 `import`, `export` 구문을 사용하여 모듈을 가져오고 내보낸다. 

```js
// hello.mjs
function printHello() {
	console.log("hello");
}
const text = "test text";

export { hello : printHello };
```  

해당 모듈을 불러오기 위해 html에서는 아래와 같이 작성한다.
```html
<html>
<body>
  <script type="module" src="hello.mjs"></script>
</body>
</html>
```  

또다른 Javascript 파일에서 해당 모듈을 불러오기 위해선 `import` 구문을 사용한다.
```js
import { hello } from './hello.mjs'
hello();
```  

<br />  

### 아직 남은 문제
ESM의 등장으로 모듈시스템이 통합되려는 움직임이 보여지긴 하지만 문제가 남아있다. **표준이 존재하지만 브라우저가 따라주지 않는다면 사용할 수 없다.** 구형 브라우저가 그런 경우이다. ESM을 지원하지 않는 브라우저 버전에서는 해당 모듈시스템을 사용할 수 없다.(크로스브라우징 이슈) 하지만 이러한 문제를 해결하기위해서 여러 방법이 등장하였다.  
(브라우져 호환성의 문제를 해결하기 위함이라 node.js 진영은 이상무)  

<br />  

### 모듈로더의 문제점
위에서 알아본 RequireJS, curlJS 들은 모듈의 형식을 가진 모듈들을 해석해주는 모듈로더이다.(AMD, UMD, CommonJS는 모듈로더 방식이다) js의 표쥰 모듈시스템(ESM)이 없던 시절 모듈로더를 통해서 모듈시스템을 구축했다. 모듈 로더의 가장 큰 특징은 **런타임(브라우저)에서 실행**된다는 것이다. 동작방식은 아래와 같다.  
- 브라우저에서는 **모듈로더로 사용할 스크립트(RequireJS, curlJS, SystemJS 등)를 로드**한다.
- **모듈 로더는 필요에따라 모듈로써 필요한 파일들을 다운받고 해석**한다.  
- 따라서 실행 도중에 많은 파일들을 로드한다.(개발자 콘솔에서도 확인이 가능하다.) 

모듈로더는 런타임 중에 파일을 불러오기때문에 응답성능에 큰 영향을 미친다는 단점이 존재한다.  

<br />  

### 문제해결 - 트랜스파일과 모듈번들러
모듈로더의 역할을 **런타임이 아닌 빌드타임으로 가져와 결과물을 미리 번들링 파일로 만드는 방법**이다. 모듈번들링 도구를 이용하여 빌드 시 모듈의 의존관계를 파악하여 번들파일을 만든다. 브라우저에서는 번들파일만 로드하면 되기 때문에 보다 응답시간이 단축된다. 하지만 ESM과 같은 모듈시스템은 구형 브라우저에서 인식이 불가할 수도 있다고 했다. 이러한 ES6+ 문법을 특정 버전 이하로 만들어주는 트랜스파일링 도구도 함께 사용한다.   
**모듈번들러 - webpack**  
**트랜스파일링 - babel**  