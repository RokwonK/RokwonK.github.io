---
title: "Spring - 공식문서 Core(IoC Container)"
categories: Spring
tags:
  - IoC
---  

스프링 프레임워크는 IoC 원칙을 구현한 IoC Container(DI라고도 불리는)를 내장하고 있다. (객체를 생성자 인수나 팩터리 메서드에서 반환받아 종속성에 추가하는 것, 쉽게 말해, **의존 객체에 대한 제어권을 외부에 넘기는 방식**{: .font-highlight} )

- `BeanFactory` 인터페이스는 모든 타입의 객체를 관리(Bean 생성, 조회, 의존성 주입)할 수 있는 설정 메커니즘을 제공한다.
- `ApplicationContext` 는 `BeanFactory`을 상속받고 다음과 같은 추가 기능을 제공해준다.
  1. **Spring AOP 기능과의 쉬운 통합**
  2. **여러가지 Application-layer context(context란, bean들이 담겨있는 컨테이너)의 기반이 됨**  
  웹 어플을 위한 WebApplicationContext 등
  3. **메시지 소스(Message Source)**  
  국제화 기능을 지원하여 다국어 메시지를 처리(번역) 기능
  4. **이벤트 발행(ApplicationEventPublisher)**  
  어플리케이션에서 이벤트를 발행하고 구독할 수 있음(어플리케이션간 통신)
  5. **리소스 로딩(ResourceLoader)**  
  어플리케이션에서 사용되는 파일,이미지,문자열, 프로퍼티 파일 등의 정적 데이터를 효율적으로 관리하고 로딩할 수 있다.
  6. **환경 설정정보 관리(Environment)**  
  다양한 환경(OS, 로깅수준, 디버깅 모드 등)에 맞게 설정 정보를 관리  
  -> `PropertyResolver` 인터페이스를 구현하여 만들 수 있음, `@Value` 어노테이션을 이요하여 설정 정보를 주입받음  
  -> properties 파일이나 YAML 파일 등의 설정 파일을 로드하고 이를 이용하여 어플에서 사용되는 설정정보를 관리  

`BeanFactory`가 프레임워크로써의 **설정기능과 기본기능을 제공**한다면 `ApplicationContext` 는 더 **엔터프라이징 기능을 더해준다.**  

Spring에서 어플리케이션의 중추 역할을 하고, IoC Container에 의해 관리되는 객체를 `Bean` 이라고 부른다. `Bean` 은 Spring의 IoC Conatiner에 의해 관리될 뿐만 아니라 생성되고 다른 Bean들과 합쳐진다. 또한, 이들 간의 종속성은 IoC Container에서 사용하는 메타데이터 정보에 반영된다.