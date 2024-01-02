---
title: "Spring - Core(Container의 동작구조)"
categories: Spring
tags:
  - IoC
---  


스프링 프레임워크는 IoC 원칙을 구현한 IoC Container(이하 스프링 컨테이너)를 내장하고 있다. 이것은 스프링 프레임워크의 가장 중요한 개념이다. 복잡한 객체간의 관계를 관리하고 의존 객체를 생성하고 주입해주는 역할을 한다. 덕분에 **객체 생성과 관리에 대한 제어권을 외부(프레임워크)로 넘김으로써 확장성 높은 유연한 어플리케이션를 설계**{: .font-highlight}할 수 있다.  

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

### Container 설정정보
스프링 컨테이너는 컨테이너를 이루는 `Bean`들에 대한 정보를 받아 관리한다. 이를 위해서 컨테이너 생성 시 설정정보 작성하고 제공한다. 일반적으로 설정정보를 작성하는 방식에는 대표적으로 아래와 같은 것들이 있다.  
1. XML
2. Groovy
3. Annotaion(@Configuration, @Bean)

**스프링 어플리케이션은 `ApplicationContext`를 구현한 클래스에서 설정정보를 인자로 받아 스프링 컨테이너를 생성한다.** 설정정보가 XML 형식일 경우 `GenericXmlApplicationContext` 클래스를 통해, Annotation 형태로 작성했다면 `AnnotationConfigApplicationContext` 클래스를 통해 설정정보를 넘겨주고 `ApplicationContext` 객체(스프링 컨테이너)를 생성한다.  

<br />  

### Container의 Bean 생성
`ApplicationContext` 가 `Bean` 객체를 만들때, `BeanDefinition`(빈 메타정보)을 기반으로 실제 객체를 생성한다. `BeanDefinition`은 인터페이스로 `ApplicationContext` 객체 생성 시, 내부적으로 `BeanDefinitionReader` 를 이용하여 설정정보를 읽고 `Bean` 객체 하나당 하나의 `BeanDefinition`을 만들어낸다. 이 메타정보에는 빈의 클래스명, 빈을 생성하는 팩토리 메서드, 싱글톤 객체로 만들것인지 등의 정보가 들어가 있다.  


<br />  

### Singleton으로 Bean 관리
만약 `Bean` 을 싱글톤을 만들지 않는다면, 요청 또는 의존관계가 있을때마다 객체를 계속생성하게 된다. 이는 쓸데없은 메모리 낭비가 일어날 수도 있다.(물론 싱글톤이 필요없을 경우도 있다) 이러한 문제를 보완하기 싱글톤 패턴을 구현할 수 있다.(private 생성자와 static 변수에 객체 생성) 하지만 기본적으로 싱글톤 패턴에는 다음과 같은 문제들이 존재한다.

- 싱글톤 패턴을 구현하기 위한 코드를 작성해야함
- 생성자가 private 이므로 생성자로 의존성 주입이 불가능
- 생성자가 private 이므로 자식 클래스 구현이 어려움
- 구체 클래스에 의존함(DIP, OCP 위반 가능성이 높음)
- 객체가 미리 생성되어 있기때문에 테스트하기 어려움

**스프링 컨테이너는 위와 같은 문제를 해결하면서 `Bean`을 싱글톤으로 생성 및 관리**{: .font-highlight}한다. (물론 싱글톤 방식만 지원하는 것은 아니다.) 이렇듯 개발자 친화적인 방법을 제공해주지만 싱글톤 방식을 사용할땐 주의할 점 있다.  

1. 하나의 인스턴스를 공유해서 사용하기 때문에 **객체 내부가 stateful 하면 안된다.**
2. 공유되지 않는 지역변수나 파라미터, ThreadLocal을 사용해야함

그렇다면 스프링 컨테이너는 어떻게 각 `Bean`이 싱글톤임을 보장할까? `@Confiugraion` 어노테이션이 붙은 클래스를 상속받아(CGLIB라는 바이트 조작 라이브러리를 사용) 임의의 다른 클래스를 만든다. 그리고 해당 클래스가 `Bean`으로 등록된다. 그렇게 상속받아 만든어진 클래스 내부에서는 각 `Bean`들이 싱글톤으로 만들어지게 된다. 

<br />  

### 설정정보 자동등록 - ComponentScan
보통 설정정보에서 `Bean`을 등록하는 코드를 작성한다. 하지만 `Bean`이 늘어날때마다 이러한 작업을 하는 것은 생산성이 떨어지며, 자칫 누락할 확률도 존재한다. 따라서 `Bean`을 자동으로 설정정보에 등록할 수 있는 방법을 지원한다.  

**설정정보의 클래스에 `@ComponentScan` 어노테이션을 달면 해당 클래스는 자동으로 패키지 내의 `Bean` 들을 스캔하고 정보를 담는다.**{: .font-highlight}(스프링 부트에서는 `@SpringBootApplication` 어노테이션이 내부적으로 해당 역할을 한다.)
 물론 `Bean`으로 등록할 객체들에도 `@Component`(`@Service`, `@Controller`, `@Repository` 등) 어노테이션을 달아주어야 한다.  

또한 `@ComponentScan` 은 includeFilters, excludeFilters를 인자로 받아 스캔대상으로 포함/제외할 객체를 선택할 수 있다.(기본값으로 모두 include)  

먄약 스캔과정에서 Bean에 등록할 이름이 중복된다면 어떻게 할까?  
1. **자동 빈 등록으로 이름이 같은 Bean이 등록된다면?**  
`ConflictingBeanDefinitionStoreException` 발생
2. **수동으로 등록 및 자동으로 등록 했을때 중복된다면?**  
수동 빈 등록이 우선권을 가진다(빈을 오버라이딩 함, 스프링 부트는 기본값으로 오버라이딩하지 않고 오류를 냄)    

<br />  

### Bean의 의존성 자동주입
각 `Bean` 객체들은 다른 `Bean`들과의 의존관계를 가진다. `Bean` 객체의 생성 및 관리는 스프링 컨테이너가 담당하므로 의존성을 주입하는 역할 또한 스프링 컨테이너가 담당한다. 이때, 우리는 `@Autowired` 어노테이션으로 스프링 컨테이너에게 의존성을 자동으로 주입해달라고 부탁할 수 있다. 해당 어노테이션을 통해 주입할 수 있는 방식은 크게 3가지가 있다.  

1. **생성자 주입**{: .font-highlight}
  - Container에 Bean이 등록될 때 같이 일어남
  - 생성자 호출시점에 1번 호출된다는 것을 보장
  - 필수적이고 **불변**하는 의존성 객체를 주입받기 좋은 방법
  - 테스트 코드에서의 주입 누락을 방지할 수 있으며 의존필드에 `final` 을 사용가능
  - 생성자가 하나일때는 `@Autowired` 를 생략할 수 있음
2. **setter 주입**
  - Bean을 등록한 이후에 주입됨
  - 이후에 변경이 가능하다
3. **필드 주입**
  - 필드에 바로 주입
  - 외부에서 주입이 불가능해서 Container가 아닌 순수한 자바객체로 테스트하기 어려움
  - 테스트 코드나 설정을 모적으로 하는 곳에서만 사용하는 것을 권장  


기본적으로 `@Autowired` 는 주입되어야할 객체가 등록되어 있지 않다면 실행되지 않는다. 하지만 주입 대상이 등록되어 있지 않더라도 실행되어야하는 경우가 있다. 이를 위해 몇가지 방법이 존재한다.  

1. `@Autowired(required = false)` → 의존관계가 없다면 메서드자체가 실행되지 않음
2. 인자 앞에 `@Nullable` → 실행은 되지만 null
3. 타입 감싸기 `Optional<Type>` → 실행이 되고 Optional.empty 값이 들어감  

**💡lombok 라이브러리**  
getter, setter와 같이 빈번하게 만들고 사용하는 코드들을 어노테이션으로 자동으로 만들어주는 라이브러리이다. 이 중 `@RequiredArgsConstructor` 어노테이션은 final이 붙은 필드들을 초기화해주는 생성자를 자동으로 만들어준다. 따라서, 생성자가 하나인 경우, 해당 어노테이션을 이용해 생성자 주입을 손쉽게 구현할 수 있다.  
{: .notice--info}  

만약 의존성 주입 시 주입 가능한 `Bean`이 2개 이상이라면 어떻게 될까? 인터페이스에 의존하고 있으니 해당 인터페이스를 구현한 `Bean`이 2개 이상 존재할 수 있다. 이때는 지명 or 우선순위를 설정할 수 있다.  

1. Bean 이름과 인자(필드)명이 일치된다면 해당 Bean 주입
2. `@Qualifier` 어노테이션 사용(Bean 클래스 및 인자에 추가하여 매칭시키기)
3. `@Primary` 어노테이션 사용(Bean 클래스에 우선순위를 주는 것)  

만약 모든 방법을 다 사용한다면, 우선순위는 Qualifier  > Primary > 타입 순이다.  

스프링 부트는 기본적으로 ComponentScan을 사용한다. 그렇다면 언제 수동 등록을 할까?  
AOP, 공통 로그 처리, 데이터베이스 연결 등 광범위하게 영향을 미치는 기술지원 객체는 수동 빈 등록으로 명확하게 나타내는 것이 유지보수 하기 좋다. 또한, 다형성을 적극 활용하는 비즈니스 로직도 수동으로 등록하는 것이 좋다.

<br />  

### Annotation 커스텀
`@Qualifier` 같은 경우 인자로 스트링을 받기 때문에 실수가 일어날 확률이 높다. 따라서, 스프링이 지원하는 Annotation 조합방식을 사용하여 커스텀 Annotation을 만들어 사용하는 것이 좋다.  

```java
import java.lang.annotation.*;

@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER}) // 해당 Annotation을 사용할 수 있는 요소들
@Retension(RetentionPolicy.RUNTIME) // 어노테이션의 라이프사이클
@Inherited // 어노테이션을 사용한 클래스를 상속한 클래스도 어노테이션을 가짐
@Documented // javadoc으로 api문서를 만들때 annotation 설명을 포함하도록 지정
@Qualifier("customAnnotation")
public @interface CustomAnnotation {
}
```

<br />  

### Bean의 생명주기
`Bean`이 생성된 후 혹은 소멸되기 전 특정 로직이 실행되어야 할 수 있다.(ex. 데이터베이스 커넥션 풀, 소켓 연결) 이때 `Bean`의 생명주기를 활용할 수 있다. 전체적인 생명주기는 다음과 같다.  

> 컨테이너 생성 → 빈 생성 → 의존관계 주입 → 초기화 콜백 → 사용 → 소멸전 콜백 → 스프링 종료 (생성자 주입의 경우 빈 생성과 의존관계 주입이 동시에 일어난다.)  

**💡 객체 생성 시 초기화도 함께 처리하는게 좋지 않을까?**  
생성자는 필수 정보를 받고 메모리를 할당해 객체를 생성하는 책임을 가진다. 초기화는 이렇게 생성된 값을 활용해서 외부 커넥션을 연결하는 등의 무건운 동작을 수행한다. 생성자 안에서는 빠르게 생성하는 일만 처리하는 것이 역할을 나누는 것이므로 유지보수하기 좋다.  
{: .notice--info}  

`Bean` 이 초기화/소멸 콜백을 받을 수 있는 방법으로 아래 3가지 방식이 있다.  
1. **인터페이스 - `InitializingBean` , `DisposableBean`**  
    - 두 인터페이스의 메서드를 override
    - 스프링 프레임워크에 의존
    - 외부 라이브러리에 적용불가(내가 코드를 고칠 수 없으므로)
    - 따라서, 잘 사용하지 않음
2. **설정정보에 초기화, 소멸 메서드 지정**
    - @Bean 어노테이션에 initMethod, destroyMethod 에 메서드 이름 지정
    - destroyMethod는 기본값으로 “(inferred)”이다. 이 기능은 자동으로 close, shutdown 이라는 이름의 메서드를 호출해준다. 따라서, 만약 이러한 이름으로 지정하였다면 메서드 이름을 지정하지 않아도 된다.
    - **설정 정보를 사용하기 때문에 외부 라이브러리에도 적용가능**
3. **Annotaion - `@PostConstruct`, `@PreDestroy`**{: .font-highlight}
    - 가장 권장하는 방식
    - javax.annoation 에 패키지에 존재(java에서 공식적으로 지원. 즉, 자바 표준)  

<br />  

### Bean Scope
Bean Scope(빈 스코프)란 **생성된 Bean이 유지되는 기간**을 의미한다. 스프링은 다양한 스코프를 지원한다.

1. **싱글톤(기본 스코프)**
    - **스프링 컨테이너 생성 시점에 생성 및 초기화 됨.**{: .font-highlight}
2. **프로토타입**
    - **스프링 컨테이너에서 Bean을 조회할 때마다 생성 및 초기화**{: .font-highlight}
    - 스프링 컨테이너가 초기화 까지만 관여, 이후는 관리하지 않음(종료에 관여X)
3. **request(Web 관련)**
    - 요청이 들어오고 나갈때까지만 유지
4. **session(Web 관련)**
    - 웹 세션이 생성되고 종료될 때까지 유지
5. **application(Web 관련)**  
    - 웹의 서블릿 컨텍스트(ServletContext)와 같은 범위로 유지
6. **websocekt(Web 관련)**  
    - 웹 소켓과 동일한 생명주기를 가지는 스코프

하지만 만약 싱글톤 빈이 프로토타입 빈에 의존하는 경우 싱글톤 빈이 생성시점에 프로토타입 빈을 주입받기 때문에 프로토타입 빈으로써의 역할(새로운 객체 할당)을 하지 못한다.

이와 같은 경우를 해결하기 위해서 **사용시 의존관계에서 탐색하여 가져올 수 있도록 `ObjectProvider` 기능을 제공**한다. 이렇게 의존관계를 주입 받는게 아니라 직접 필요한 의존관계를 찾는 것을 DL(Dependency Lookup, 의존관계 조회) 이라한다.  

```java
class ClientBean {
	@Autowired
	private ObjectProvider<PrototypeBean> beanProvider;

	public int logig() {
		// 의존관계 탐색
		PrototypeBean bean = beanProvider.getObject();
	}
}
```  

하지만, `ObjectProvider` 는 스프링이 지원하는 기술로 스프링에 의존한다는 단점이 있다. 자바 표준에서도 이와 같은 기술을 제공해주는데 바로 `Provider` 이다.(javax.inject:javax.inject:1 라이브러리를 gradle에 추가해야함) 이 기능은 get() 메서드 하나만 존재하여 기능이 매우 단순하다는 장점이 있다. 

여기서 더 나아가서 Provider조차 사용하지 않는 프록시 기술이 등장하였다. 해당 클래스위에 다음과 같이 작성하여 프록시 클래스를 만들어 낼 수 있다.  
`@Scope(value = “request”, proxyMode = ScopedProxyMode.TARGET_CLASS)`

CGLIB라는 라이브러리로 해당 클래스를 상속받은 가짜 프록시 객체를 만들고 이 객체를 주입해준다. 이후, 실제 호출시 이 가짜 프록시에서 객체를 반환해주는 것이다.(다형성의 힘!) 이 방식은 장점은 다음과 같다.
- 싱글톤 쓰듯이 사용할 수 있음
- Provider든 프록시든 꼭 필요한 시점까지 지연처리
- 단지 어노테이션 설정 변경반으로 원본객체를 대체할수 있음. 그것도 원본객체를 고치지 않고! → **다형성과 DI 컨테이너가 가진 큰 강점**  


<br />  

**해당 글은 아래 강의 및 문서를 공부한 내용입니다.**  
- [인프런 - 스프링 핵심원리 기본편](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B8%B0%EB%B3%B8%ED%8E%B8)
- [Spring 공식문서 Core 파트](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#context-create)