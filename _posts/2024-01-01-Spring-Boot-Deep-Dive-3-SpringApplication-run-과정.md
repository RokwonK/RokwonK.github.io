---
title: "Spring Boot Deep Dive(3) - SpringApplication run 과정"
excerpt: "SpringApplication 의 run 메서드는 어떤 과정을 거쳐 어플리케이션을 실행하는지 알아보자"

categories:
  - Spring

permalink: /spring/spring-boot-deep-dive-3/
---

<br />  

[SpringApplication 초기화 과정](https://rokwonk.github.io/spring/spring-boot-deep-dive-1/)에서 `SpringApplication` 객체가 생성되면서 어떤 것들이 초기화 되는지 살펴보았다. `SpringApplication`은 인스턴스화를 후에 곧받로 인스턴스 `run` 메서드를 실행한다.

`run` 메서드 내부에서는 단계별로 여러가지 일을 수행한다. (ApplicationContext이 수행하는 역할은 [ApplicationContext 상속구조](https://rokwonk.github.io/spring/spring-boot-deep-dive-2/)를 확인해보자.) `run` 메서드 내부에서 수행하는 작업들을 소스코드를 통해 알아보자.

> Spring Boot 3.2.1 버전을 기준으로 작성되었습니다.


`run` 메서드에는 다음의 로직들이 실행되는데 먼저 가볍게 주석으로 각 역할에 대해 설명해두었다.
```java
/* SpringApplication.java */
public ConfigurableApplicationContext run(String... args) {  
	// 시간초 타이머
	Startup startup = Startup.create();  

	// ShutdownHook 준비
	if (this.registerShutdownHook) {  
	  SpringApplication.shutdownHook.enableShutdownHookAddition();  
	}  

	// BootStrapContext 생성
	DefaultBootstrapContext bootstrapContext = createBootstrapContext();  
	ConfigurableApplicationContext context = null;  

	// AWT Headless 관련 프로퍼티 설정
	configureHeadlessProperty();  

	// 리스너 이벤트 뿌려주기
	SpringApplicationRunListeners listeners = getRunListeners(args);  
	listeners.starting(bootstrapContext, this.mainApplicationClass);  
	
	try {  
		// 시작 시 Argument, Environment 설정
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);  
		ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);  
		
		// 시작 시 배너 출력
		Banner printedBanner = printBanner(environment);  
		
		// ApplicationContext 생성
		context = createApplicationContext();  
		
		// Startup 셋팅
		context.setApplicationStartup(this.applicationStartup);  
		
		// ApplicationContext 준비단계
		prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);  
		
		// ApplicationContext refresh단계
		refreshContext(context);  
		
		// ApplicationContext refresh 이후
		afterRefresh(context, applicationArguments);  
		
		startup.started();  
		if (this.logStartupInfo) {  
		 new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), startup);  
		}
		listeners.started(context, startup.timeTakenToStarted());  
		
		// ApplicationRunner, CommandLineRunner 실행
		callRunners(context, applicationArguments);  
	}  
	catch (Throwable ex) {  
		if (ex instanceof AbandonedRunException) {  
		 throw ex;  
		}  
		handleRunFailure(context, ex, listeners);  
		throw new IllegalStateException(ex);  
	}  
	try {  
		if (context.isRunning()) {  
		 listeners.ready(context, startup.ready());  
		}  
	}  
	catch (Throwable ex) {  
		if (ex instanceof AbandonedRunException) {  
		 throw ex;  
		}  
		handleRunFailure(context, ex, null);  
		throw new IllegalStateException(ex);  
	}  
	return context;  
}
```

핵심로직을 기준으로 그루핑시켜 보면 다음과 같다.
1. **사전 준비**
	- 실행시간 체크할 Startup
	- ShutDownHook 설정
	- BootStrapContext 생성
	- AWT 설정 및 Run Listener 생성
2. **환경 설정 셋팅**
	- 실행될 때 넘긴 인자(Argument)와 우리가 작성 `applicaiton.properties`와 같은 환경설정 셋팅
3. **`ApplicationContext` 생성 & 실행**
	- `ApplicationContext` 생성
	- Startup들 셋팅
	- prepare
	- refresh
4. **실행 완료 - 마무리**
	- 설정된 Runner들을(개발자가 설정함) 실행

<br />  

## 사전 준비
이 단계에서는 준비 단계이다. 본격적으로 `ApplicationContext`을 실행하기 전 필요한 것들을 셋팅한다.

```java
/* SpringApplication.java */
public ConfigurableApplicationContext run(String... args) {  
	// 1. 시간초 타이머
	Startup startup = Startup.create();  

	// 2. ShutdownHook 준비
	if (this.registerShutdownHook) {  
	  SpringApplication.shutdownHook.enableShutdownHookAddition();  
	}  

	// 3. BootStrapContext 생성
	DefaultBootstrapContext bootstrapContext = createBootstrapContext();  
	ConfigurableApplicationContext context = null;  

	// 4. AWT Headless 관련 프로퍼티 설정
	configureHeadlessProperty();  

	// 5. 리스너 셋팅
	SpringApplicationRunListeners listeners = getRunListeners(args);  

	// 6. staring 이벤트 호출
	listeners.starting(bootstrapContext, this.mainApplicationClass);  

	// ...
}
```  

<br />  

### 실행 시간 체크 - Startup 셋팅
처음 시작은 실행시간을 체크할 타이머 역할을 하는 Startup 셋팅하는 걸로 시작한다. 구현체를 보면 시간을 체크하기 위한 메서드들로만 이루어져 있다.  

<br />  

### ShutDownHook 사용 설정
두 번째로 `ShutDownHook`을 준비하다. `registerShutdownHook`은 기본값으로 true로 설정되어 있기때문에 활성화된다.

> **ShutDownHook**
>
> 프로그램이 예기치 못하게 종료되는 상황에서 별도의 쓰레드를 통해 연결 종료, 및 자원 반납의 처리를 할 수 있도록 작업을 처리해주는 역할을 해준다.

<br />  

### 임시 컨텍스트 BootStrapContext 생성
세 번째로 `BootStrapContext`를 생성하고 `ApplicationContext`의 변수를 선언한다. `BootStrapContext`는 `ApplicationContext` 생성 전에 **임시로 사용되는 Context이다. 어플리케이션 설정 및 환경설정 시에 사용**된다. `SpringApplicationRunListeners`의 `starting`과 `environmentPrepared` 두 이벤트를 처리하는데 관여한다. 각 이벤트에 대해선 이벤트 발생 로직을 설명할 때 자세히 알아보도록하자.

`createBootstrapContext()`를 통해 `BootStrapContext`을 생성하는데 이때 SpringApplication 생성자에서 초기화하였던 `bootstrapRegistryInitializers`들을 실행한다. [SpringApplication 초기화 과정](https://rokwonk.github.io/spring/spring-boot-deep-dive-1/)에서 보았듯 현재 버전(3.2.1)에서는 `bootstrapRegistryInitializers`들은 1개도 존재하지 않는다. 레거시로 남아있는 듯하다.  

<br />  

```java
/* SpringApplication.java */
private DefaultBootstrapContext createBootstrapContext() {  
	DefaultBootstrapContext bootstrapContext = new DefaultBootstrapContext();  
	this.bootstrapRegistryInitializers.forEach((initializer) -> initializer.initialize(bootstrapContext));  
	return bootstrapContext;  
}
```

이후에 생성할 `ApplicationContext`를 담을 변수도 미리 선언해 준다.  

<br />  

### 그래픽 설정 끄기 AWT 설정
네 번째로 AWT Headless 관련 프로퍼티를 설정한다. `java.awt.headless` 시스템 프로퍼티를 설정하는 코드만 들어가는데 설정된 기본값이 true이다. true면 AWT 기능을 비활성화된다.

> **Java AWT(Advanced Window Toolkit)**
>
> AWT는 Java에 포함된 그래픽 사용자 인터페이서(GUI) 개발을 위한 라이브러리이다. 서버와 같이 GUI가 필요없는 환경에서는 이 기능을 비활성화하는 것이 자원을 절약 할 수 있다. 

```java
/* SpringApplication.java */
private void configureHeadlessProperty() {  
	// SYSTEM_PROPERTY_JAVA_AWT_HEADLESS == java.awt.headless
	// this.headless 초기값은 true로 설정되어 있음
	System.setProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS,  
		 System.getProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS, Boolean.toString(this.headless)));
}
```  

<br />  

### 이벤트 브로커 설정
다섯 번째로 `SpringApplicationRunListeners`를 셋팅한다. Listeners는 각 단계마다 이벤트 메서드를 실행하는데 내부에 셋팅된 `SpringApplicationRunListener` 타입의 listener들을 실행한다. 현재 버전(3.2.1)을 기준으로는 `EventPublishingRunListener` 하나만 존재한다.

`SpringApplicationRunListeners`를 셋팅하는 로직을 보면 `getSpringFactoriesInstances`를 통해 정의된 `SpringApplicationRunListener` 들을 가져온다. 해당 메서드는 [SpringApplication 초기화 과정](https://rokwonk.github.io/spring/spring-boot-deep-dive-1/)에서 설명했듯이 `META-INF/spring.factories`에서 정의된 타입의 구현체 클래스를 가져와 인스턴스화 해준다. 

마지막에는 모든 listener들을 하나의 클래스(`SpringApplicationRunListeners`)로 묶는다. **이벤트 실행과 관련된 책임을 하나의 클래스로 캡슐화, 위임**하는 모습을 볼 수 있다.

```java
/* SpringApplication.java */
private SpringApplicationRunListeners getRunListeners(String[] args) {  
	ArgumentResolver argumentResolver = ArgumentResolver.of(SpringApplication.class, this);  
	argumentResolver = argumentResolver.and(String[].class, args);  
	
	// META_INF/spring.factories 에 정의된 클래스 인스턴스화
	// 현재는 EventPublishingRunListener 하나만 존재
	List<SpringApplicationRunListener> listeners = getSpringFactoriesInstances(SpringApplicationRunListener.class,  
		 argumentResolver);  
	SpringApplicationHook hook = applicationHook.get();  
	SpringApplicationRunListener hookListener = (hook != null) ? hook.getRunListener(this) : null;  
	if (hookListener != null) {  
	  listeners = new ArrayList<>(listeners);  
	  listeners.add(hookListener);  
	}  
	return new SpringApplicationRunListeners(logger, listeners, this.applicationStartup);
```

이번엔 한 층 더 들어가 `EventPublishingRunListener`의 내부를 살펴보면 `initialMulticaster`를 Has-A 관계로 가지고 있다. 이벤트 관련 메서드가 실행되면 이 멀티캐스터를 통해 이벤트와 관련된 `ApplicationListener`을 실행하는 형태이다.

일종의 **Pub-Sub 패턴으로 각 단계별로 후처리 되어야 할 작업을 처리**한다. 덕분에 느슨한 결합(Sub 쪽에 무엇인가 코드작업이 일어나도 Pub 쪽의 코드에는 전혀 영향을 끼치지 않는다.)을 가지고 단계별 후처리 작업들을 진행할 수 있다.

조금 흥미로운 점은 SpringApplication 생성 시 초기화했던 `ApplicationListener` 를 어디서 사용하나 했더니 `EventPublishingRunListener`가 사용하고 있었다. `initialMulticaster`가 각 이벤트에 맞는 `ApplicationListener`를 선별하여 실행시킨다.

```java
class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {  
	private final SpringApplication application;  
	private final String[] args;  
	private final SimpleApplicationEventMulticaster initialMulticaster; 
	
	EventPublishingRunListener(SpringApplication application, String[] args) {  
	  this.application = application;  
	  this.args = args;  
	  this.initialMulticaster = new SimpleApplicationEventMulticaster();  
	}
	
	@Override  
	public void starting(ConfigurableBootstrapContext bootstrapContext) {  
	   multicastInitialEvent(new ApplicationStartingEvent(bootstrapContext, this.application, this.args));  
	}  

	// 이벤트 메서드들 ...
	
	// mulitcaster를 이용하여 등록된 ApplicationListener를 실행시킨다.
	private void multicastInitialEvent(ApplicationEvent event) {  
	   refreshApplicationListeners();  
	   this.initialMulticaster.multicastEvent(event);  
	}  
	  
	private void refreshApplicationListeners() {
		this.application.getListeners().forEach(this.initialMulticaster::addApplicationListener);  
	}
}
```

`SpringApplicationRunListeners`의 구조를 정리해서 보자면 다음과 같은 구조를 띈다. 중첩 Pub-Sub 구조라는게 흥미롭다.

![SpringApplicationRunListeners구조](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/e119dff7-8b1f-4cc2-89df-ec5dee20f058)  

<br />  


### starting 이벤트에서 벌어지는 일
다시 돌아와서, 여섯번째로 방금 셋팅한 `SpringApplicationRunListeners`에 starting 이벤트를 발생시킨다. 이 이벤트로 최종적으로 실행되는 `ApplicationListener` 중 `BackgroundPreinitializer`은 **시간이 오래걸리는 작업들을 백그라운드 스레드를 이용해 미리 초기화**시킨다. 대표적으로 Validation, JDK, TomcatInitializer 등이 존재하다. TomcatInitializer에서는 쿠키 프로세서와 인증 되지 않은 유저 처리를 위한 프로세서를 초기화한다.

```java

public class BackgroundPreinitializer implements ApplicationListener<SpringApplicationEvent>, Ordered {

	// 이 메서드 실행됨
	@Override  
	public void onApplicationEvent(SpringApplicationEvent event) {  
		if (event instanceof ApplicationEnvironmentPreparedEvent  
			 && preinitializationStarted.compareAndSet(false, true)) {  
			performPreinitialization();  
		}
	   // ...
	}  
	  
	private void performPreinitialization() {  
		try {
			Thread thread = new Thread(new Runnable() {  
				@Override  
				public void run() {  
					runSafely(new ConversionServiceInitializer());  
					runSafely(new ValidationInitializer());  
					if (!runSafely(new MessageConverterInitializer())) {  
					   runSafely(new JacksonInitializer());  
					}  
					runSafely(new CharsetInitializer());  
					runSafely(new TomcatInitializer());  
					runSafely(new JdkInitializer());  
					preinitializationComplete.countDown();  
				}  
					
				boolean runSafely(Runnable runnable) {  
					try {  
					   runnable.run();  
					   return true;  
					}  
					catch (Throwable ex) {  
					   return false;  
					}  
				}  
			}, "background-preinit");  
			thread.start();  
		}  
		catch (Exception ex) {  
		}  
	}
}
```  

<br />  

### 정리
이제 사전 준비가 완료되었다. 살펴볼게 많아 길어졌는데 정리해보면 다음과 같다.
1. 타이머 설정
2. ShutDownHook 설정
	- 예기치 못한 오류로 어플리케이션 종료 시 자원 반납을 실행할 쓰레드 설정
3. 임시 컨텍스트 생성
	- `ApplicationContext` 생성 전 어플리케이션 설정과 환경설정을 담당할 `BootStrapContext` 생성
4. 그래픽 설정 끄기
	- 자원 절약을 위한 JAVA의 그래픽 라이브러리 `AWT` 설정 끄기
5. 후 처리를 담당할 이벤트 브로커 생성
	- `ApplicationContext`의 각 단계마다 후처리를 담당할 리스너들을 등록
6. 백그라운드 쓰레드를 이용해 오래걸리는 작업 미리 처리
	- starting 메서드의 뒤에 존재하는 리스너들이 해당 작업을 처리
	- 흥미로운 중첩 Pub-Sub 구조

<br />  

## 환경 설정값 셋팅
본격적으로 `ApplicationContext`를 구성하는 단계이다. 실행시 넘기는 Argument와 우리가 설정한 환경설정 값들을 불러와 준비시키는 단계이다.

```java
/* SpringApplication.java */
public ConfigurableApplicationContext run(String... args) {  

	// ...
	
	try {
		// 1. 시작 시 Argument, Environment 설정
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);  
		ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);  
		
		// 2. 시작 시 배너 출력
		Banner printedBanner = printBanner(environment);  
		
		// ...
	}
	// ...
}
```  

<br />  

### 환경설정(Environment ) 불러오기
Argument와 통합해 Environment 준비하는 만들어주는 단계이다. Environment라 함은 `Profile`과 `PropertySource`들을 의미한다. 알다시피 Properties를 설정할 수 있는 방법이 굉장히 많다. 

`prepareEnvironment()`에서 **모든 Properties들이 `PropertySource` 객체로 매핑된고 environment에 저장**된다. SystemProperties, SystemEnvironment, ConfigurationProperties, ServletConfigInitParams, ServletContextInitPrams, 설정파일(application.yml 등)과 같은 것들이 `PropertySoucre`과 `Profile` 객체로 매핑된다.

```java
/* SpringApplicaiton.java */

private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments) {  

	// 1. Create configure the environment
	ConfigurableEnvironment environment = getOrCreateEnvironment();  
	configureEnvironment(environment, applicationArguments.getSourceArgs());  
	ConfigurationPropertySources.attach(environment);
	
	// 2. Listener에 의해 설정파일(application.yml)을 읽고 셋팅함
	listeners.environmentPrepared(bootstrapContext, environment);
	DefaultPropertiesPropertySource.moveToEnd(environment);  
	Assert.state(!environment.containsProperty("spring.main.environment-prefix"),  
		 "Environment prefix cannot be set via properties.");  
		
	// 3. SpringApplication에 바인딩
	bindToSpringApplication(environment);  
	if (!this.isCustomEnvironment) {  
	  EnvironmentConverter environmentConverter = new EnvironmentConverter(getClassLoader());  
	  environment = environmentConverter.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());  
	}  
	ConfigurationPropertySources.attach(environment);  
	return environment;  
}
```

처음 `getOrCreateEnvironment()` 메서드가 실행되면 ServletConfigInitParams, ServletContextInitPrams, SystemProperties,SystemEnvironment를 읽어 `PropertySource`로 만든다.

실제로 이 부분이 실행된 뒤에 environment에는 4가지 `PropertySource`가 셋팅되는 것을 볼 수 있다.

![getOrCreateEnvironment](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/a78bbb61-398b-4ed0-bf2f-9d96867443f2)

다음으로 위에서 생성한 Listener를 이용해 `environpreared` 메서드를 발생시키게 되면 `EventPublishingRunListener`가 실행하는 `AppliationListener` 중 
`EnvironmentPostProcessorApplicationListener`이 실행이 된다.

여기서 총 7가지의 `EnvironmentPostProcessor`(후처리기)가 동작한다. 
![EnvironmentPrepared이벤트로 실행되는 PostProcessor](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/8b348c34-c416-4d95-b540-bc8433fe23e3)

이 후처리기들은 추가적인 `SystemProperties`들을 추가하고 **우리가 설정한 설정파일 `application.yml`파일들을 불러와 `PropertySource`로 셋팅**해준다. 이것들은 `OriginTrackedMapPropertySource`라는 구현체로 만들어진다. `environmentPrepared` 이벤트가 실행한 이후를 보면 필자가 작성한 설정파일 2개가 등록되어 있는 것을 볼 수 있다.

![EnvironmentPrepared이벤트 실행 이후](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/2dbc05ce-7f0e-4ec3-bf06-3b3de56bf64a)

마지막으로 셋팅된 `Environment`를 바인딩시키며 마무리한다.

<br />  

### 배너 출력
스프링 부트 실행 시 커맨드 라인에 출력되는 화면을 출력하는 과정이다. 필요에 따라 출력 내용을 바꿀 수도 있다고 한다. 주요 내용은 아니므로 패스하겠다.

<br />  

## ApplicationContext 생성 및 실행
본격적으로 실행환경을 셋팅하는 단계이다. `WebServer`부터 Bean 구성까지 스프링 어플리케이션이 실행되기 위한 모든 과정이 여기서 일어난다.

```java
/* SpringApplication.java */
public ConfigurableApplicationContext run(String... args) {  
	// ...
	try { 
		// ... 
		
		// 1. ApplicationContext 생성
		context = createApplicationContext();  
		
		// 2. Startup 셋팅
		context.setApplicationStartup(this.applicationStartup);  
		
		// 3. ApplicationContext 준비단계
		prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);  
		
		// 4. ApplicationContext refresh단계
		refreshContext(context);  
		
		// 5. ApplicationContext refresh 이후
		afterRefresh(context, applicationArguments);  
		
		// ...
	}
	// ...
}
```  

<br />  

### ApplicationContext 생성
가장 먼저, `WebApplicationType`따라 맞는 `ApplicationContext`를 생성한다. [ApplicationContext 상속관계](https://rokwonk.github.io/spring/spring-boot-deep-dive-2/)에서 알아보았던 **3가지 최종 구현체 중 하나를 생성**한다. 지금은 Servlet을 이용한 웹 어플리케이션이므로 `AnnotationConfigServletWebServerApplicationContext`를 생성한다.

```java
/* SpringApplication.java */

// DefaultApplicationContextFactory 
private ApplicationContextFactory applicationContextFactory = ApplicationContextFactory.DEFAULT;

protected ConfigurableApplicationContext createApplicationContext() {  
   return this.applicationContextFactory.create(this.webApplicationType);  
}
```  

<br />  

### Startup 셋팅
생성한 `ApplicationContext`에 `ApplicationStartup`을 셋팅한다. `ApplicationStartup`은 각 단계(Step)으로 나누어 지표를 기록해주는 역할을 한다. 이후에 `ApplicationContext` refresh 단계에서 startup을 이용해 단계마다 태깅을 하는 것을 볼 수 있다.

<br />  

### ApplicationContext 준비
`ApplicationContext`을 본격적으로 실행하는 단계인 refresh 단계를 시작하기 전에 필요한 설정을 셋팅하는 단계이다. 
1. **setEnvironment**
	- 생성한 `ApplicationContext`에 `Environment` 를 셋팅한다.
2. **postProcessApplicationContext**
	- `ClassLoader`, `ResourceLoader` 등 필요한 자원을 셋팅
3. **addAotGeneratedInitializerIfNecessary**
	- Spring boot 3.0 부터는 AOT를 사용하여 빌드시간을 단축할 수 있다. 만약 AOT를 사용한다면 그에 따른 Initializer를 셋팅한다.
4. **applyInitializers**
	- `SpringApplication` 생성 시 초기화했던 Initializer들을 적용한다.
5. **contextPrepared**
	- `ApplicationContext` 준비완료에 따른 후처리를 진행한다.
6. **bootstrapContext.close(context)**
	- `ApplicationContext`이 생성되어 필요없어진 `BootStrapContext`를 종료한다.
7. **싱글톤 빈 생성**
	- `SpringApplication`에서 등록할 수 있는 빈들을 등록, 생성한다.
8. **설정정보 셋팅**
	- 순환참조 허용, 빈 이름 중복 허용에 관한 셋팅을 한다.(기본값은 둘 다 불가능이다)
9. **BeanFactoryPostProcessor 등록**
	- BeanFactory 설정이 완료된 후에 사용될 `BeanFactoryPostProcessor`를 등록한다.
	- 지연생성, `PropertySource` 의 우선순위를 정하는 후처리기를 등록하는 것 같다.

```java
/* SpringApplication.java */

private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context, ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
	// 1. 환경설정 셋팅
	context.setEnvironment(environment);  
	
	// 2. 주입하기 
	postProcessApplicationContext(context);  

	// 3. AOT 셋팅
	addAotGeneratedInitializerIfNecessary(this.initializers);  
	
	// 4. Initializer 적용
	applyInitializers(context);  
	
	// 5. 준비완료 후처리
	listeners.contextPrepared(context);  

	// 6. 임시 Context 종료
	bootstrapContext.close(context);  
	if (this.logStartupInfo) {  
		logStartupInfo(context.getParent() == null);  
		logStartupProfileInfo(context);  
	}  
	
	// 7. 필요한 싱글톤 
	ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();  
	beanFactory.registerSingleton("springApplicationArguments", applicationArguments);  
	if (printedBanner != null) {  
		beanFactory.registerSingleton("springBootBanner", printedBanner);  
	}  
	
	// 8. 설정정보 셋팅
	if (beanFactory instanceof AbstractAutowireCapableBeanFactory autowireCapableBeanFactory) {
			autowireCapableBeanFactory.setAllowCircularReferences(this.allowCircularReferences);  
		if (beanFactory instanceof DefaultListableBeanFactory listableBeanFactory) {
					listableBeanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);  
		}  
	}  
	
	// 9. BeanFactoryPostProcessor 등록
	if (this.lazyInitialization) {  
		context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());  
	}  
	if (this.keepAlive) {  
		context.addApplicationListener(new KeepAlive());  
	}  
	context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));  
	
	// ...
}
```

<br />  

### 빈 구성단계 - Refresh
본격적으로 **개발자들이 등록한 빈들과 자동구성될 빈들이 등록되고 인스턴스화 되는 단계**이다. `ApplicationContext`의 refresh() 메서드를 통해 이 과정들이 차례대로 실행된다. 컴포넌트 스캔과 자동구성, 프록시 빈들이 구성되는 중요한 과정으로 그 과정이 상당히 길어 따로 포스팅하겠다.(이 과정이 메인단계라고 볼 수 있으며 여담이지만 이 과정이 궁금해 Deep Dive 시리즈를 작성하게 되었다.)

```java
/* SpringApplication.java */

private void refreshContext(ConfigurableApplicationContext context) {  
	if (this.registerShutdownHook) {  
		shutdownHook.registerApplicationContext(context);  
	}  
	refresh(context);  
}

protected void refresh(ConfigurableApplicationContext applicationContext) {  
	// 
	applicationContext.refresh();  
}
```

마지막으로 `afterRefreshContext`를 통해 실행환경 구성을 마무리하는데 이 메서드는 현재는 비어 있다.

<br />  

## 실행 완료
`ApplicationContext`가 실행되고 난 후의 프로세스들이다. 대표적으로 개발자가 등록한 Runner(`CommandLineRunner`, `ApplicationRunner`)를 실행해 `ApplicationContext`가 완료되었음을 알린다.

```java
/* SpringApplication.java */
public ConfigurableApplicationContext run(String... args) {  
	// ...
	try { 
		// ... 

		startup.started();  
		if (this.logStartupInfo) {  
		 new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), startup);  
		}
		listeners.started(context, startup.timeTakenToStarted());  
		
		// ApplicationRunner, CommandLineRunner 실행
		callRunners(context, applicationArguments);  
	}
	// ...
}
```

<br />  

### 마무리
디테일한 Bean 구성단계(refresh)를 제외하고 전체적인 SpringApplication의 실행을 살펴보았다. 코드를 잘 알지 못하는 상태에서 하나하나 뜯어봐야 하다보니 디버깅 능력이 날로 성장하는 것 같다. 물론 스프링 코드 자체가 깔끔하게 잘 짜여진 덕분이 크긴하다. 네이밍 뿐만 아니라 객체지향 패턴을 적절히 잘 사용하다보니 배울점도 많은 것 같다. 

지금까지 살펴본 실행흐름을 중요한 기능을 중심으로 도식화한 그림으로 마무리한다.

![SpringApplication_run_실행](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/bcbb76a3-09c7-4432-b246-117b62a87f15)