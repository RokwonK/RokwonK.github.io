---
title: "Spring - Core(Container의 동작구조)"
categories: Spring
tags:
  - IoC
---  


스프링 프레임워크는 IoC 원칙을 구현한 IoC Container(이하 스프링 컨테이너)를 내장하고 있다. 이것은 스프링 프레임워크의 가장 중요한 개념이다. 복잡한 객체간의 관계를 관리하고 의존 객체를 생성하고 주입해주는 역할을 한다. 덕분에 **객체 생성과 관리에 대한 제어권을 외부(프레임워크)로 넘김으로써 확장성 있고 유연한 어플리케이션를 설계**{: .font-highlight}할 수 있다.  

이러한 스프링 컨테이너는 어떻게 구성되고 만들어진걸까? 스프링 프레임워크의 핵심이라고 불리는 스프링 컨테이너에 대해서 자세하게 살펴보자.  

스프링 컨테이너는 두가지 핵심 인터페이스를 구현한다. `BeanFactory`와 이를 확장한 `ApplicationContext` 인터페이스이다. 사실상  `ApplicationConext` 을 구현한 것으로 스프링 컨테이너를 `ApplicationConext`라고 부르기도 한다. 두 인터페이스는 다음과 같은 역할을 가진다.  

- `BeanFactory` 인터페이스는 모든 타입의 객체를 관리(Bean 생성, 조회, 의존성 주입)할 수 있는 설정 메커니즘을 제공한다.
- `ApplicationContext` 는 `BeanFactory` 역할 외에 다음과 같은 부가적인 기능을 가진다.
    - **메시지 소스(MessageSource 상속)**  
    → 국제화 기능을 지원하여 다국어 메시지를 처리(번역) 기능
    - **이벤트 발행(ApplicationEventPublisher 상속)**  
    → 어플리케이션에서 이벤트를 발행하고 구독할 수 있음(어플리케이션간 통신)
    - **리소스 로딩(ResourceLoader 상속)**  
    → 어플리케이션에서 사용되는 파일,이미지,문자열, 프로퍼티 파일 등의 정적 데이터를 효율적으로 관리하고 로딩할 수 있다.
    - **환경 설정정보 관리(EnvironmentCapable 상속)**  
    → 다양한 환경(OS, 로깅수준, 디버깅 모드 등)에 맞게 설정 정보를 관리  
    → `PropertyResolver` 인터페이스를 구현하여 만들 수 있음, `@Value` 어노테이션을 이용하여 설정 정보를 주입받음  
    → properties 파일이나 YAML 파일 등의 설정 파일을 로드하고 이를 이용하여 어플에서 사용되는 설정정보를 관리

`BeanFactory`가 프레임워크로써의 **설정기능과 기본기능을 제공**한다면 `ApplicationContext` 는 **엔터프라이징 기능을 더해준다.**  

**스프링 어플리케이션의 비즈니스 로직을 담당하고 스프링 컨테이너에 의해 관리되는 객체를 `Bean`**{: .font-highlight} 이라고 부른다. `Bean` 은 Spring의 컨테이너에 의해 관리, 생성되고 다른 `Bean`들과 연관관계를 가질 수 있다. 스프링 어플리케이션은 이러한 `Bean` 간 상호작용을 통해 동작하게 된다.


<br />  

### Container 설정
우리가 정의하여 제공한 객체들의 설정정보를 이용하여 Spring Container 가 생성, 초기화되어 실행가능한 어플리케이션이 만들어진다. 일반적으로 설정정보를 작성하는 방식에는 대표적으로 아래와 같은 것들이 있다.  
1. XML
2. Groovy
3. Annotaion(@Configuration, @Bean)

ApplicationContext를 생성할때 설정정보를 함께 보내준다. 설정정보가 XML 형식일 경우 `GenericXmlApplicationContext` 클래스를 통해, Annotation 형태로 작성했다면 `AnnotationConfigApplicationContext` 클래스를 통해 설정정보를 넘겨주고 `ApplicationContext` 를 생성한다.

정리하자면, **스프링 어플리케이션은 `ApplicationContext` 를 구현한 클래스에서 인자로 받은 설정정보를 통해 `Bean` 객체를 만들고 관리하는 Container를 생성한다. 이를 IoC Container 또는 Spring Container라고 부른다.**  

<br />  

### Container의 Bean 생성

`ApplicationConext` 가 `Bean` 객체를 만들때, `BeanDefinition`(빈 메타정보)을 기반으로 실제 객체를 생성한다. `BeanDefinition`은 인터페이스로 `ApplicationContext` 객체 생성 시, 내부적으로 `BeanDefinitionReader` 를 이용하여 설정정보를 읽고  `Bean` 객체 하나당 하나의 `BeanDefinition` 을 만들어낸다. 이 메타정보에는 빈의 클래스명, 빈을 생성하는 팩토리 메서드, 싱글톤인지 등의 정보가 들어가 있다.

생성된 BeanDefinition을 가져오기 위해선 `AnnotationConfigApplicationContext` 와 같이 클래스에서 불러올 수 있다.(ApplicationContext 인터페이스에서는 BeanDefinition을 가져오는 메서드가 정의되어 있지 않다)

### Singleton Container

싱글톤 객체를 사용하지 않는다면, 객체를 참조할때마다 새로운 객체를 생성하게 된다. 이를 보완하기 위해 직접 싱글톤 패턴을 구현한다.(private 생성자와 static 변수에 객체 생성) 하지만 기본적으로 싱글톤 패턴에는 다음과 같은 문제들이 존재한다.

- 싱글톤 패턴을 구현하기 위한 코드를 작성해야함
- 생성자가 private이므로 생성자로 의존성 주입이 불가능
- 생성자가 private이므로 자식 클래스 구현이 어려움
- 구체 클래스에 의존함(DIP, OCP 위반 가능성이 높음)
- 객체가 미리 생성되어 있기때문에 테스트하기 어려움

Spring Container는 위와 같은 문제를 해결하면서 `Bean`을 싱글톤으로 관리한다. (물론 싱글톤 방식만 지원하는 것은 아니다.) 이렇듯 개발자 친화적인 방법을 제공해주지만 싱글톤 방식을 사용할땐 주의할 점 있다.

1. 하나의 인스턴스를 공유해서 사용하기 때문에 객체 내부가 stateful 하면 안된다.
2. 공유되지 않는 지역변수, 파라미터, ThreadLocal 을 사용해야함(즉, stack 영역에 저장되는 것들만 사용)

그렇다면 Spring Container는 어떻게 각 `Bean`이 싱글톤임을 보장할까? `@Confiugraion` 어노테이션이 붙은 클래스를 상속받아(CGLIB라는 바이트 조작 라이브러리를 사용) 임의의 다른 클래스를 만든다. 그리고 해당 클래스가 `Bean`으로 등록된다. 그렇게 상속받아 만든어진 클래스 내부에서는 각 `Bean`들이 싱글톤으로 만들어지게 된다. 

### Component Scan

Component Scan(컴포넌트 스캔)이란 설정 정보에 직접적으로 수동 등록하지 않고도 **자동으로 스프링 빈을 등록할 수 있는 기능**이다.

설정정보에 `@ComponentScan` 어노테이션을 붙이면 `@Component` ( `@Service`, `@Controller`, `@Repository` 등)이 붙은 모든 클래스들을 전부 찾아서 `Bean`에 등록해준다.(스프링 부트에서는 `@SpringBootApplication` 어노테이션이 그 역할을 한다.)

`@ComponentScan` 은 includeFilters, excludeFilters를 인자로 받아 스캔대상으로 포함/제외할 객체를 선택할 수 있다.(기본값으로 모두 include)

컴포넌트 스캔에서 Bean 이름이 중복된다면 어떻게 할까?

1. **자동 빈 등록으로 이름이 같은 Bean이 등록된다면?**
`ConflictingBeanDefinitionStoreException` 발생
2. **수동으로 등록, 자동으로 등록 했을때**
수동 빈 등록이 우선권을 가진다(빈을 오버라이딩 함, 스프링 부트는 기본값으로 오버라이딩하지 않고 오류를 냄)

### 의존관계 자동주입

각 `Bean`에서는 `@Autowired`를 이용해서 **의존관계를 자동으로 주입받을 수 있다.** 

1. **생성자 주입**
→ Container에 Bean이 등록될 때 같이 일어남
→ 생성자 호출시점에 1번 호출된다는 것을 보장
→ 필수적이고 **불변**하는 객체를 주입하기 좋은 방법
→ 테스트 코드에서의 주입 누락을 방지할 수 있으며 의존필드에 `final` 을 사용가능
→ 생성자가 하나일때는 `@Autowired` 를 생략할 수 있음
2. **setter 주입**
→ Bean을 등록한 이후에 주입됨
→ 이후에 변경이 가능하다
3. **필드 주입**
→ 필드에 바로 주입
→ 외부에서 주입이 불가능해서 Container가 아닌 순수한 자바객체로 테스트하기 어려움
→ 테스트 코드나 설정을 모적으로 하는 곳에서만 사용하는 것을 권장

기본적으로 `@Autowired` 는 주입되어야할 객체가 등록되어 있지 않다면 실행되지 않는다. 하지만 주입 대상이 등록되어 있지 않더라도 실행되어야하는 경우가 있다. 이를 위해 몇가지 방법이 존재한다.

1. `@Autowired(required = false)` → 의존관계가 없다면 메서드자체가 실행되지 않음
2. 인자 앞에 `@Nullable` → 실행은 되지만 null
3. 타입 감싸기 `Optional<Type>` → 실행이 되고 Optional.empty 값이 들어감

생성자를 만들고 `@Autowired` 를 붙이는 것조차 귀찮아 하는 많은 개발자들을 위해 자동으로 주입시켜주는 기능이 존재한다. 바로 lombok 라이브러리이다.(gradle 설정, plugin 다운, Annotaion Processors Enable 필수) lombok은 자주 사용되는 코드들을 어노테이션을 붙이는 것만으로도 자동으로 만들어준다.(getter, setter 등)

클래스에 lombok이 지원하는 `**@RequiredArgsConstructor` 어노테이션을 붙이면 final이 붙은 필드들이 있는 생성자를 자동으로 만들어준다.**

만약 Container 의존성 주입 시 주입 가능한 `Bean`이 2개 이상이라면 어떻게 될까? 인터페이스에 의존하고 있으니 해당 인터페이스를 구현한 `Bean`이 2개 이상 존재할 수 있다. 이때는 지명 or 우선순위를 설정할 수 있다.

1. Bean 이름과 인자(필드)명이 일치된다면 해당 Bean 주입
2. `@Qualifier` 어노테이션 사용(Bean 클래스 및 인자에 추가하여 매칭시키기)
3. `@Primary` 어노테이션 사용(Bean 클래스에 우선순위를 주는 것)

만약 모든 방법을 다 사용한다면, 우선순위는 Qualifier  > Primary > 타입 순이다.



**참고**
- 인프런 - 스프링 핵심원리 기본편
- Spring 공식문서 Core 파트