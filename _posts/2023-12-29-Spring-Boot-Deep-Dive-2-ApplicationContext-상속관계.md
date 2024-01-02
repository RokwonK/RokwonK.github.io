---
title: "Spring Boot Deep Dive(2) - ApplicationContext 상속관계"
excerpt: "Spring Container의 전신인 ApplcationContext는 어떤 역할과 책임을 가지는지 살펴보자"

categories:
  - Spring

permalink: /spring/spring-boot-deep-dive-2/
---

<br />  

스프링은 스프링 컨테이너(혹은 DI 컨테이너) 라고 부르는 핵심 기술을 중심으로 동작한다. 이 컨테이너는 빈이라고 부르는 스프링 객체들의 생명주기를 관리하고 빈 간의 의존성을 관리해준다.

스프링 컨테이너의 실체는 코드에서 `ApplicationContext`의 구현체로 존재한다. 이 컨테이너는 [앞선 포스팅(SpringApplication 초기화 과정 Deep Dive)](https://rokwonk.github.io/spring/spring-boot-deep-dive-1/)에서 `SpringApplication`의 run 메서드를 통해 `ApplicationContext`가 생성 및 실행된다고 잠깐 이야기 했었다.

이번 포스팅에서는 실제 웹 어플리케이션에서 동작하는 `ApplicationContext`이 어떤 상속관계로 이루어져있는지, 우리의 웹 어플리케이션은 어떤 `ApplicationContext` 구현체로 동작하는지 소스코드를 들여다보자.

{: .important}
> Spring Boot 3.2.1 버전을 기준으로 작성되었습니다.

<br />  

## 최종 ApplicationContext 구현체
우리가 Spring Boot를 이용해 어플리케이션을 만들면 `ApplicationContext`의 최종 구현체는 다음 3개 중 한 가지이다.
1. `AnnotationConfigServletWebServerApplicationContext`
	- **전통적인 서블릿기반 Web Mvc 어플리케이션**
2. `AnnotationConfigReactiveWebServerApplicationContext`
	- **Reactive 기반 웹 어플리케이션**
3. `AnnotationConfigApplicationContext`
	- 그 외

[이전 포스팅](https://rokwonk.github.io/spring/spring-boot-deep-dive-1/)에서 `SpringApplication` 생성자에서 초기화했던 `WebApplicationType`을 이용하여 셋 중 하나를 인스턴스화한다. 실제로 코드에서 어떤 구현체가 선택되어 만들어지는 지는 다음 포스팅(SpringApplication run 메서드 실행)에서 자세히 알아보도록 하자. 

Servlet 기반 웹 어플리케이션의 구현체인 `AnnotationConfigServletWebServerApplicationContext`을 기준으로 상속관계를 들여다보자.

<br />  

## ApplicationContext 상속 구조
IntelliJ가 제공해주는 Diagram의 힘을 빌려 `AnnotationConfigServletWebServerApplicationContext`의 상속구조를 확인해보면 아래의 사진과 같다.

![AnnotationConfigServletWebServerApplicationContext 전체 상속구조](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/a5cb57ca-1aed-4dd1-8c23-9d57c72b84ba)

여러모로 복합해 보인다. 때문에 소스코드를 뒤적거리며 나름대로 핵심 class와 Interface만을 추려보았다.

![ApplicationContext 핵심 Class & Interface](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/dcefdb0d-bf01-4250-a80b-2ccad894f77c)

`AnnotationConfigServletWebServerApplicationContext`부터 거슬러 올라가면
- **ServletWebServerApplicationContext**
- **GenericWebApplicationContext**
- **GenericApplicationContext**
- **AbstractApplicationContext** (추상 클래스)  

까지 상속 받고 있다. 각 클래스마다 각 한 가지의 핵심 책임을 가지고 구현되고 있는 것을 볼 수 있다.(곧 살펴보도록 하자)

`AbstractApplicationContext`가 상속받고 있는 `DefatulResourceLoader`는 Url, FileUrl, ClassPath에서 데이터를 받아올 수 있는 데이터로더이다.

클래스 상속구조에서의 최상단인 `AbstractApplicationContext` 에서는 `ApplicationContext`(실행환경)로서 지녀야할 여러가지 설정, 역할이 정의된 인터페이스인 `CofigurationApplicationContext`를 구현하고 있다. 여기서 재밌었던 점은 **하나의 인터페이스가 한 종류의 역할만을 가지고 필요에 의해 여러 인터페이스를 합쳐 새롭게 필요한 인터페이스를 만들고 있다는 것이다.** ISP(인터페이스 분리 원칙)을 지키고 있음을 알 수 있다.

상속되는 각 인터페이스마다 어떤 역할을 가지고 있는지도 곧 알아보도록 하자. 위 사진에 나와있는 핵심 역할을 지닌 인터페이스를 정리하자면 다음과 같다.
- **ConfigurableApplicationContext**
- **ApplicationContext**
- **ListableBeanFactory**
- **HierarchicalBeanFactory**
- **BeanFactory**

<br />  

## ApplicationContext의 역할

### 단일 빈 검색을 담당하는 BeanFactory
상속 구조의 가장 꼭대기에는 `BeanFactory`가 있다. 보통 Spring Container에 대해 이야기하면 나오는 그 `BeanFactory`가 맞다. 왜 그런 이야기가 나오느냐 하면 `BeanFactory`의 메서드를 살펴보면 잘 알 수 있다. **단일 빈을 가져오거나 빈에 대한 정보를 확인할 수 있는 역할**을 가지고 있다. 말 그대로 빈 공장이 가져야할 역할을 가지고 있다.

```java
public interface BeanFactory {  
    String FACTORY_BEAN_PREFIX = "&";  

    Object getBean(String name) throws BeansException;  
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;  
    Object getBean(String name, Object... args) throws BeansException;  
    <T> T getBean(Class<T> requiredType) throws BeansException;  
    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;  
    <T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);  
    <T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);  
    boolean containsBean(String name);  
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;  
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;  
    boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;  
    boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;  
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;  
    Class<?> getType(String name, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException;  
    String[] getAliases(String name);  
}
```

BeanFactory는 일반적인 빈에 대한 정보를 가져올 수 있는 역할을 가지고 있다. 하지만 해당 역할만으로는 부족할 때가 있는다. 때문에 이를 보충하는 인터페이스가 대표적으로 2가지 존재하는데 `ListableBeanFactory`와 `HierarchicalBeanFactory`이다.  

<br />  

### 다중 빈 검색을 담당하는 ListableBeanFactory
`ListableBeanFactory`의 역할은 이름에서 알 수 있듯이 **빈에 대한 정보를 List로 제공**하는 것이다. 왜 굳이 List로 제공할 필요가 있을까? 우리가 개발할때도 그렇듯이 하나의 인터페이스 타입으로 여러가지 구현체를 만들 수 있다. 때문에 타입으로 불러올때 N개의 결과가 있을 수 있다. `ListableBeanFactory`는 이런 기능을 추가 확장시킨 인터페이스이다. 

그런데 이 뿐만 Annotation으로 빈을 찾거나 빈의 Annotation을 찾을 수 있는 기능도 존재한다.(이건 Annotation 관련 인터페이스로 따로 분할하는게 좋지 않을까? 하는 생각도 든다.)

```java
public interface ListableBeanFactory extends BeanFactory {  
    boolean containsBeanDefinition(String beanName);  
    int getBeanDefinitionCount();  
    String[] getBeanDefinitionNames();  
    <T> ObjectProvider<T> getBeanProvider(Class<T> requiredType, boolean allowEagerInit);  
    <T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType, boolean allowEagerInit);  
    String[] getBeanNamesForType(ResolvableType type);  
    String[] getBeanNamesForType(ResolvableType type, boolean includeNonSingletons, boolean allowEagerInit);  
    String[] getBeanNamesForType(@Nullable Class<?> type);  
    String[] getBeanNamesForType(@Nullable Class<?> type, boolean includeNonSingletons, boolean allowEagerInit);  
    <T> Map<String, T> getBeansOfType(@Nullable Class<T> type) throws BeansException;
    <T> Map<String, T> getBeansOfType(@Nullable Class<T> type, boolean includeNonSingletons, boolean allowEagerInit) throws BeansException;  
    String[] getBeanNamesForAnnotation(Class<? extends Annotation> annotationType);  
    Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType) throws BeansException;  
    <A extends Annotation> A findAnnotationOnBean(String beanName, Class<A> annotationType) throws NoSuchBeanDefinitionException;  
    <A extends Annotation> A findAnnotationOnBean(String beanName, Class<A> annotationType, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException;  
    <A extends Annotation> Set<A> findAllAnnotationsOnBean(String beanName, Class<A> annotationType, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException;
}
```  

<br />  

### 빈팩토리 간 연결 - HierarchicalBeanFactory
`BeanFactory`의 또다른 확장 인터페이스인 `HierarchicalBeanFactory`는 `BeanFactory`간의 계층을 관리할 수 있는 역할을 가진다. 처음에는 의아해했다. *'음? `BeanFactory` 간 계층 구조가 필요하다고? 하나의 어플리케이션에는 하나의 `BeanFactory`만 구동하는 것 아닌가?'* 라고 생각했는데 멀티 모듈과 같은 상황을 간과했다. 

멀티 모듈은 모듈을 조합하여 하나의 어플리케이션 만들어 `BeanFactory`가 여러 개이다. 때문에 빈 관리의 효율을 위해 빈을 통합해서 관리할 필요가 있다. 때문에 외부에서 하나의 `BeanFactory`에 요청을 보내도 내부에서 연관되어 있는`BeanFactory`들에서 빈을 찾아 리턴하는 것이 좋을 것이다. 때문에 `BeanFactory`간에 부모관계(연관관계)를 맺어 둔다.

```java
public interface HierarchicalBeanFactory extends BeanFactory {  
    BeanFactory getParentBeanFactory();  
    boolean containsLocalBean(String name);    
}
```

그런데 흥미로운 점이 있다. `HierarchicalBeanFactory`에는 연관된 `BeanFactory`를 get하는 메서드는 존재하지만 set하는 메서드는 존재하지 않는다. 그럼 누가 관계를 맺는 역할을 가진단 말인가?

빈팩토리 간 관계를 맺어주는 역할은 `ConfigurableApplicationContext`이 맡고 있다. 아무래도 `ApplicationContext`(실행환경)의 구성을 설정하는 부분은 모두 `ConfigurableApplicationContext`의 역할이 가지는 듯하다.(ConfigurableApplicationContext에 대한 설명은 잠시 후에 이어한다.)

여기까지가 Bean 및 BeanFactory 관리를 위한 역할 정의를 위한 인터페이스들이었다면 이제는 본격적으로 실행환경을 위한 역할들이 등장한다.  

<br />  

### 실행환경 - ApplicationContext
`ListableBeanFactory`와 `HierarchicalBeanFactory`를 상속받는 `ApplicationContext`는 실행환경이 마땅히 지녀야할 정보들을 가져오는 메서드들이 존재한다. 말 그대로 실행환경의 역할이다.

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory, MessageSource, ApplicationEventPublisher, ResourcePatternResolver {  
    String getId();
    String getApplicationName();  
    String getDisplayName();
    long getStartupDate();
    ApplicationContext getParent();
    AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException;  
}
```

위 코드에서 특별한게 없어보이지만 한 가지 흥미로운 점이 있다. 바로 `getAutowireCapableBeanFactory()` 메스드를 통해 리턴되는 `AutowireCapableBeanFactory`이다. `ApplicationContext`가 곧 `BeanFactory`인데 이 새로운 타입의 `BeanFactory`는 대체 뭘까?

우리가 **지금까지 알아본 `ApplicationContext`는 실제로 빈을 관리하지 않는다.** 외부적으로 빈을 관리하는 것처럼 보이지만 **내부적으로 빈을 생성하고 빈 간의 의존성 주입을 실제로 수행하는 기능 담당 `BeanFactory`를  따로 가지고 있다.**  이 포스팅의 타이틀에 있는 (BeanFactory는 사실 2개라고?)의 진실이 바로 이것이다.

`ApplicationContext`는 외부에 빈 정보를 제공해주기 위해 정보제공용 `BeanFactory` 역할만을 담당하고 내부적으로 실제 빈 관리를 담당하는 `BeanFactory(AutowireCapableBeanFactory)` Has-A 관계로 가진다.(놀랍도록 역할이 잘 나뉘어져 있다.)

`ApplicationContext` 클래스 상속관계를 살펴볼 때 어떤 클래스가 이 기능적 빈팩토리의 책임을 담당하는지 살펴보자.  

<br />  

### 실행환경에 대한 구성과 실행환경을 조작하는 ConfigurableApplicationContext
**실행환경 설정, 구성 대한 모든 역할**은 `ConfigurableApplicationContext` 전부 가지고 있다. 앞서 설명한 `ApplicationContext`의 3가지 최종 구현체 모두 이 인터페이스를 구현하고 있다. 

여러가지 get, set 메서드들이 존재하는지 get 메서드들은 제외하고 set 메서드들 위주로 가져와봤다. 위에서 이야기한 `BeanFactory` 연관관계 설정을 위한 `setParent` 부터 `ApplicationListener` 설정도 보인다.

주목해서 볼만한 점은 `refresh` 메서드와 `addBeanFactoryPostProcessor` 메서드이다. `refresh` 메서드는 `ApplicationContext` 초기화 과정 중 refresh 단계에 해당되는데 이 단계에서 모든 빈들이 객체화 되고 구성된다. `addBeanFactoryPostProcessor`는 refresh 단계 중 실행되는 `BeanFactoryPostProcessor`를 추가하는 메서드이다. 

refresh 단계에서 무슨 일들이 일어나는지에 대한 자세한 내용은 추후 refresh 단계 소스코드 Deep Dive 포스팅에서 알아본다.(컴포넌트 스캔과 자동구성이 바로 이 단계에서 일어난다. 이렇게 이야기하면 흥미가 생기겠지?ㅎㅎ)

```java
public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable {  
    void setParent(@Nullable ApplicationContext parent);
    void setEnvironment(ConfigurableEnvironment environment);  
    void setApplicationStartup(ApplicationStartup applicationStartup);
    void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor);  
    void addApplicationListener(ApplicationListener<?> listener);  
    void removeApplicationListener(ApplicationListener<?> listener);
    void setClassLoader(ClassLoader classLoader);  
    void addProtocolResolver(ProtocolResolver resolver);  
    void refresh() throws BeansException, IllegalStateException;  
    void registerShutdownHook();  
    ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
}
```  

<br />  

### ApplicationContext 역할 정리 
이렇게 `ApplicationContext`가 실행환경으로 동작 및 설정되기 위해 어떤 종류의 인터페이스들로 구성되어 있는지를 살펴보았다. 마지막으로 나름 위 내용들을 도식화 해보았다.

![ApplicationContext 역할 정리 ](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/a239a155-39d6-4773-b674-771d8bf6b7af)


<br /> <br />  

## 상속받는 각 클래스들이 지니는 책임

### AnnotationConfigServletWebServerApplicationContext
`AnnotationConfigServletWebServerApplicationContext`은 Web MVC에서 사용되는 `ApplicationContext` 최종구현체이다. [Docs](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/web/servlet/context/AnnotationConfigServletWebServerApplicationContext.html)에 따르면 `@Configuration`이나 `@Component`처럼 **어노테이션 기반의 빈들을 등록하는 기능을 수행한다고 소개**한다.

하지만 프로젝트 실행을 디버깅 해본 결과, 실제로 이 클래스에서 어노테이션을 스캔하는 로직들이 동작하는 걸 확인할 수는 없었다. 아무래도 개발자가 직접 사용하는 방식인 듯하다.

<br />  

### ServletWebServerApplicationContext
 `AnnotationConfigServletWebServerApplicationContext`가 상속받고 있는 클래스로 **내장 WebServer를 생성, 관리하는 기능을 수행하는 클래스**이다.

실행 단계 중 refresh 단계에서 `createWebServer`메서드를 실행하여 WebServer를 생성, 셋팅한다.

```java
public class ServletWebServerApplicationContext extends GenericWebApplicationContext implements ConfigurableWebServerApplicationContext {  
    private static final Log logger = LogFactory.getLog(ServletWebServerApplicationContext.class);  
    public static final String DISPATCHER_SERVLET_NAME = "dispatcherServlet";  
    private volatile WebServer webServer;  
    private ServletConfig servletConfig;
    private String serverNamespace;  

    private void createWebServer() {  
      // ...
    }
}
```  

<br />  

### GenericWebApplicationContext
`GenericWebApplicationContext` 는 **Servlet의 실행환경인 `ServletContext`을 관리**한다.

`WebServer` 위에서 `ServletContext`가 동작하므로 `WebServer`가 생성 셋팅될때 ServletContext도 생성되어 셋팅된다. 이러한 셋팅 로직은 `ServletWebServerApplicationContext`의 `createWebServer` 메서드 내 깊숙한 곳에 존재한다.

```java
public class GenericWebApplicationContext extends GenericApplicationContext implements ConfigurableWebApplicationContext, ThemeSource {  

    @Nullable  
    private ServletContext servletContext;  

    @Nullable  
    private ThemeSource themeSource;
}
```  

<br />  

### GenericApplicationContext
`GenericApplicationContext`는 Generic이라는 이름에서부터 유추할 수 있듯이 3종류의 최종 구현체가 모두에게 상속되는 클래스이다. 이 클래스가 바로 **실제 기능적인 빈 생성,관리를 담당하는 내부 빈팩토리를 관리하는 클래스**이다. 

실제로 내부 메서드들도 대부분 내부적인 빈팩토리를 조작하는 메서드들로 구성되어 있다. 또 `BeanDefinitionRegistry` 인터페이스를 구현해 BeanDefinition 저장소임을 표방하고 있다.

`DefaultListableBeanFactory` 가 내부 빈팩토리로 `ApplicationContext` 인터페이스에서 보았던 `AutoWireCapableBeanFactory`를 구현한 구현체이다.

```java
public class GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry {  
    private final DefaultListableBeanFactory beanFactory;  

    @Nullable  
    private ResourceLoader resourceLoader;  
    private boolean customClassLoader = false;  
    private final AtomicBoolean refreshed = new AtomicBoolean();

    @Override  
    public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)  
          throws BeanDefinitionStoreException {  
        this.beanFactory.registerBeanDefinition(beanName, beanDefinition);  
    }  
      
    @Override  
    public void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {  
        this.beanFactory.removeBeanDefinition(beanName);  
    }  
      
    @Override  
    public BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {  
        return this.beanFactory.getBeanDefinition(beanName);  
    }
}
```  

<br />  

### AbstractApplicationContext
마지막으로 `AbstractApplicationContext`이다. 이 클래스는 추상 클래스로 **`ApplicatioContext`로서 제 기능을 다하기 위한 기본적인 로직들이 구현**되어 있다. 특히 refresh()와 같은 프로젝트 실행에 대한 로직들이 **템플릿 메소드 패턴으로 되어 있어 서브클래스에서 이를 효과적으로 구현**할 수 있도록 돕는다.

아래코드는 템플릿 메소드 패턴이 적용된 refresh 코드이다. 각 단계의 메서드들을 서브클래스들이 필요에 따라 오버라이딩해 필요한 로직을 작성할 수 있다. 

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader implements ConfigurableApplicationContext {
	  // ...

    @Override  
    public void refresh() throws BeansException, IllegalStateException {  
        this.startupShutdownLock.lock();  
        try {  
          // ...

          // 템플릿 메소드 패턴
          prepareRefresh();  
          ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  
          prepareBeanFactory(beanFactory);  
        
          try {  
              postProcessBeanFactory(beanFactory);  
              StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");  

              invokeBeanFactoryPostProcessors(beanFactory);   
              registerBeanPostProcessors(beanFactory);  
              beanPostProcess.end();  

              initMessageSource();  
              initApplicationEventMulticaster();  
              onRefresh();  
              registerListeners();  
              finishBeanFactoryInitialization(beanFactory);  
              finishRefresh();  
          }  
          // ...
        }
    }
}
```  

<br />  

### 정리
복잡한 로직들을 이렇게 깔끔하고 단순하게 정리했다는게 놀랍다. 적절한 역할을 가지는 인터페이스로 나누고 각 클래스가 한 가지 책임만을 지니도록 깔끔한게 분리된 스프링 코드를 보니 새롭게 공부하는 것 같은 느낌이 든다.

마지막으로 지금까지 알아본 인터페이스들과 클래스들이 가지는 핵심 역할과 책임을 도식화해보았다.
![ApplicationContext상속구조](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/a6e3d218-e57e-48f5-aa01-725a545a99a8)  

다음 포스팅은 다시 `SpringApplication` 실행으로 돌아가서 SpringApplication의 `run` 메서드에서 Application이 실행되는 과정의 코드를 살펴보도록 하자.