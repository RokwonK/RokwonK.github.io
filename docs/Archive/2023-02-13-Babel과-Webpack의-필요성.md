---
layout: default
parent: Archive
title: "Babel과 Webpack의 필요성"
categories: Front-end
tags:
  - Javascript
---  

프론트 개발자들에게 Babel과 Webpack은 친숙한 도구들일 것이다. 상용 서비스를 개발한다면 적어도 한 번쯤은 들어봤을만큼 중요한 도구들이다. 빠르게 진화하는 프론트세계에서 과거의 유물(구형 브라우저)들에 대한 호환성을 제공하는 매우 유익한 도구들이다. 이 포스트에서는 `Babel`과 `Webpack`에 대해서 알아보자.  

본격적으로 설명하기에 앞서 두 도구의 핵심을 빠르게 훑어보자. 아래 핵심설명을 보자면 역할은 다르지만 **공통적으로 높은 개발생산성과 브라우저 호환성을 위한 도구**{: .font-highlight}라고 볼 수 있다.
- `Babel` : **브라우저 호환성을 위한 트랜스파일러**(최신 문법을 브라우저에 호환되는 문법으로 바꿔줌)  
- `Webpack` : 브라우저 호환을 위한 모듈시스템 구성, 다수의 의존관계에 있는 모듈들을 하나의 js파일로 만들어줌으로써 **응답속도를 향상 시켜주는 모듈번들러**  

<br />  

## Babel
ES6이상의 최신 문법은 구현 브라우저에서 동작하지 않는 경우가 있다. 이러한 문제로 인해 **이전 버전의 문법으로 전환이 필요한데 해당 기능을 해주는 도구가 Babel과 같은 트랜스파일러이다.**  

babel은 npm을 통해 설치할 수 있다. 빌드 시에만 사용되기 때문에 `--save-dev` 옵션으로 devDependencies에 넣어주자.  
```bash
$ npm install --save-dev @babel/core
```  

이후 루트경로에서 `babel.config.json`를 만들고 plugin, preset을 추가하여 빌드 시 필요한 동작을 수행하도록 만들어 줄 수 있다. plugin, preset은 조금있다 자세히 알아보도록 하자. 지금은 빌드 시 처리할 행동목록이라고 이해하면 편할 것 같다.  
  
```bash
$ npx babel [파일] --out-file [변환후파일]
```  
위 명령어를 통해 원하는 파일을 트랜스파일 할 수 있다. 이때 babel은 3단계를 거쳐 결과를 만든다.  
- **Parsing** : 코드 -> AST(추상구문트리)
- **Transforming** : AST -> 변환된 코드
- **Printing** : 변경완료된 결과를 다른 파일로 출력한다.  

만들어진 결과물은 지정한 문법수준으로 트랜스파일 되었을 것이다. 이제 Babel 사용법에대해 더 자세하게 알아보자.  

<br />  

### Babel 설정하기
`babel.config.json`, `.babelrc` 또는 `package.json` 내 `babel` 키를 이용해서 설정을 할 수 있다.  

`.babelrc`파일은 경로별 구성이 가능하다. 폴더 구조별로 각각 다른 설정이 가능하다.  
`babel.config.json`은 전역 프로젝트 구성용이다. 물론 `.babelrc`를 루트경로에 설정한다면 같은 효과를 낼 수 있다.  

파일 내부를 보면 아래와 같다. 다른 옵션도 있지만 크게 plugin과 preset이 존재한다. 이 둘은 리스트 형식으로 빌드 시 수행할 동작을 추가해 줄 수 있다.  
```json
// babel.config.json
{
  "plugins": []
  "presets": []
  ...
}
```  

<br />  

### Plugins
plugins은 **Babel이 코드변환(빌드) 시 수행할 동작 리스트**이다. 직접 만들 수도 만들어진 플러그인(npm으로 다운)을 사용할 수도 있다. ES6+ 문법들에 대해 이하 버전으로 트랜스파일링 동작을 수행하는 플러그인들이 존재하는데 문제는 각 문법들(화살표 함수, const 등)마다 plugin 하나씩 전부 추가해야한다. 이러한 문제점은 preset을 통해 해결할 수 있다.  

ES6 문법인 Arrow Function을 plugin을 통해 트랜스파일하는 예시이다. npm을 통해 plugin을 하고 config 파일에 작성해주면 끝이다.
```bash
$ npm install --save-dev @babel/plugin-transform-arrow-functions
```  
```json
// babel.config.json
{
  "plugins": ["@babel/plugin-transform-arrow-functions"]
}
```  

하지만 앞서 말했듯 ES6문법을 전부 트랜스파일하기 위해 각 문법마다 plugin을 작성하기에는 노가다성이 짙다. 여기서 preset이라는 개념이 등장한다.  

<br />  

### Presets
**preset은 여러 plugin들을 조합하여 만들어진 하나의 도구세트**이다. ES6+ 문법 중 에서 하위버전에 상응하는 문법이 있는 것들(화살표함수, const 등)을 전부 `@babel/preset-env`에 들어있다.  
해당 preset을 추가하는 것은 plugin와 별반 다를 바 없다. npm으로 설치 후 config파일에 넣어주기만 하면된다.
```json
// .babelrc 
{
  "presets": [
    ["@babel/preset-env", { "targets": { "browsers": ["last 2 versions", ">= 1% in KR"] } }]
  ]
}
```  
위 코드에서 `targets`이라는 옵션이 보일 것이다. 이 옵션은 Babel만이 가지는 큰 장점인데 **트랜스파일링 기준을 문법이 아닌 브라우저에 초점을 맞춘 것**이다. 브라우저와 그 버전마다 지원해주는 ECMAScript 문법버전이 상이한데 제공하고자 하는 브라우저리스트를 명시하여 보다 높은 호환성을 제공해 줄 수 있다.(위의 문법은 '모든 브라우저의 가장 최신 두 버전 중 한국에서 1% 이상의 점유률을 가지고 있는 브라우저'라는 의미)

<br />  

### Polyfill
`@babel/preset-env` preset에서 '하위 버전과 상응가능한 문법들' 이라고 얘기했다. ES6에서 처음 등장한 Promise, Map, Set과 같은 객체들은 이하 버전에서 상응하는 문법이 존재하지 않는다. 이렇게 **상응되는 하위 버전이 없는 문법들의 동작방식을 수정하도록 도와주거나 새롭게 구현을 해주는 라이브러리가 `@babel/polyfill`**이다.  

`@babel/polyfill`은 config파일에 작성하는 것이 아닌 실제 프로젝트 파일 중 가장 루트에 작성한다.(서로다른 파일에서 또 작성하면 충돌이 난다고 한다.)  
```js
import '@babel/polyfill';
```  

<br />  

### Babel의 특징
지금까지 살펴본 Babel의 특징을 정리해보자면(+ α)
- ECMAScript 버전이 아닌 **브라우저리스트를 통한 더 높은 호환성**
- **커스터마이징, 확장성에 강점**을 보임(플러그인, 프리셋 사용)  
- source-map 지원을 통해 **컴파일된 코드를 쉽게 디버깅**할 수 있다.
- 하지만 **컴파일 과정이 느림**(또 다른 트랜스 파일러인 `SWC`에 비해)

<br />  

## Webpack  
Webpack은 js의 모듈시스템을 브라우저에서 호환가능한 형태로 바꿔주며 **모듈로 이루어진 파일들을 번들링하여 하나의 파일로 만들어준다.**{: .font-highlight} 번들링한 파일을 브라우저로 로드함으로써 빠른 응답속도를 가진다.  

**💡 번들링이 응답속도가 빠른이유**  
번들링 하지 않는다면 연관된 모듈파일들을 브라우저상에서 로드하게 될 것이다. 특히 HTTP/1.1 버전의 경우 필요한 스크립트파일을 하나씩 로드할 것이다. 브라우저에서 모든파일을 로드하는 방법은 파일당 네트워크 통신시간 존재하기 때문에 상대작으로 느릴 수 밖에 없다.
{: .notice--info}  

Webpack을 이용하여 빌드 시 **모듈들을 IIFE(즉시 실행 함수) 형식으로 감싸준다. IIFE의 가장 큰 특징은 해당 함수만의 스코프가 존재**한다는 것이다. 그로 인해 모듈간의 스코프 문제로 인한 충돌이 일어나지 않는다.  

**💡 모듈을 IIFE로 감싸는 이유**  
Javascript는 파일이 달라도 하나의 전역 스코프를 지닌다.(ES6 이전) 모든 파일이 하나의 스코프를 지님으로써 서로 다른 파일에서 동일한 변수명이 존재할 수 있어 프로그램 동작에 치명적인 문제를 일으킨다. IIFE는 본인만의 스코프를 지녀 젼역 스코프로 인한 충돌을 예방할 수 있다.  
{: .notice--info}  

다른 IIFE 형식의 번들러 도구들(Make, Gulp 등)과 달리 Webpack은 의존성 그래프를 생성하여 빌드해준다. 이는 최종 결과물의 번들파일이 **어떤 파일에 의존되어 있는지 확인할 수 있다**는 장점을 가진다.  

이제 Webpack을 설치하고 구조를 확인해보자. 아래 명령어를 통해 devDependencies에 라이브러리를 설치할 수 있다. 
```bash
$ npm i -D webpack webpack-cli
```  

기본적으로 명령어와 옵션들로 설정을 해줄 수 있지만 사용하는 옵션이 많아질수록 명령어 입력하기가 번거로워지고 가독성도 좋지않다. `webpack.config.js`파일을 만들고 설정을 등록해주는 것이 편하다.  

webpack에서 핵심은 **entry, output, module, plugin**이라는 개념이다. 하나씩 살펴보자.
```js  
// webpack.config.js  
const webpack = require('webpack');
module.exports = {
  mode: 'development', // development, production
  entry: '',
  output: {
    path: '',
    filename: '',
  },
  module: {

  },
  plugins: [],
};
```  

<br />  

### Entry(진입접)
`Entry`는 webpack이 번들링을 진행시킬 진입점이다. Webpack은 entry에 명시된 파일을 기준으로 트리를 만들고 번들 파일을 뱉어낸다.  
```js
module.exports = {
  entry: "./src/index.js",
};
```  

<br />  

### Output(결과 번들파일)
결과로 만들어지는 파일의 경로와 이름을 지정할 수 있다.
```js
module.exports = {
  entry: "./src/index.js",
  output: {
    path: "/",
    filename: "bundle.js"
  }
};
```  

<br />  

### Loader
`module`에 작성하며 자바스크립트 파일이 아닌 HTML, CSS, Images, 폰트 등도 함께 번들링할 수 있다. `rules` 속성에 처리 규칙을 작성한다. `test`에 매칭되는 항목을 `use`에 등록된 loader로 처리한다는 의미이다. loader로는 `css-loader`, `babel-lodaer` 등이 있다.  
```js
module.exports = {
  entry: "./src/index.js",
  output: {
    path: "/",
    filename: "bundle.js"
  },
  module: {
    rules: [
      {
        test: /\.css$/,
        use: ["style-loader", "css-loader"],
      },
    ],
  },
};
```  

<br />  

### Plugin
추가적으로 기능을 더하고자 할때 사용한다. Loader와의 다른점은 **Plugin은 번들링이 되어있는 파일을 처리한다.** 이전 결과물을 제거하거나 결과물에 빌드 정보 등을 추가하는 플러그인 등이 존재한다.  

<br />  

### Webpack의 장점  
Webpack의 장점을 정리하자면 아래와 같다.
- 모듈 호환성 및 의존성 관리
- 빠른 컴파일
- [**코드 스플리팅**](https://webpack.kr/guides/code-splitting/)을 이용하여 원하는 모듈을 원하는 타이밍에 로딩 가능
- **소스 코드 최적화** - uglify(난독화)/minify(압축)
- **소스 맵 설정**을 사용함으로써 컴파일된 파일에서도 원래의 파일 구조를 확인 가능  

<br />  

**참고**
- [BabelJS(바벨)](https://www.zerocho.com/category/ECMAScript/post/57a830cfa1d6971500059d5a)
- [[Babel] 폴리필(polyfill) - @babel/preset-env](https://velog.io/@kwonh/Babel-%ED%8F%B4%EB%A6%AC%ED%95%84polyfill-babelpreset-env)
- [babel은 ts를 어떻게 해석할까?](https://janggulu.tistory.com/m/entry/babel%EC%9D%80-ts%EB%A5%BC-%EC%96%B4%EB%96%BB%EA%B2%8C-%ED%95%B4%EC%84%9D%ED%95%A0%EA%B9%8C)
- [우리는 Webpack이 왜 필요할까?](https://seogeurim.tistory.com/13)
- [JavaScript 모듈화 도구, webpack](https://d2.naver.com/helloworld/0239818)
- [웹팩(Webpack) 기본 설정법 (Entry/Output/Loader/Plugins)](https://www.daleseo.com/webpack-config/)