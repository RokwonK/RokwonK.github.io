---
title: "Spring Boot DB통신(2) - 스프링 컨테이너는 어떻게 JPA를 제공하는가?"
excerpt: "JPA에서 사용되는 구성요소들과 이 구성요소들을 스프링 컨테이너는 어떻게 제공하는걸까?"

categories:
  - Spring

permalink: /spring/spring-boot-db-2/
---  

<br />  

우리 개발자들이 JPA를 사용하기 위해 필요한 여러가지 구성요소들이 있다. 우리는 그 중 `EntityManager`를 주입 받고 이를 이용해 객체지향의 방식으로 DB와 소통한다. 우리가 편하게 `EntityManager`를 주입 받아서 사용할 때까지 스프링 컨테이너는 몇 가지 일을 수행하여 이를 효과적으로 사용할 수 있도록 돕는다.

먼저, JPA의 구성요소에 대해 살펴보고 **이 구성요소들을 스프링 컨테이너가 어떻게 제공**하는지 알아보자.

> **❗️ 이 포스팅이 다루는 내용**  
> **스프링 컨테이너가 '개발자가 JPA를 사용하는데 필요한 것'을 어떻게 제공하는지에 포커스를 둔다.** JPA의 사용과 관련된 내용은 초반에 JPA를 설명하기 위해 잠시 다루고 이후 거의 다루지 않는다.

<br />  

## JPA와 구성요소
JDBC가 편리한 데이터베이스 소통을 지원해주지만 실제로 어플리케이션을 개발할때 생산성이 떨어지는 문제점들이 존재한다. 어플리케이션이 추구하는 객체지향적인 로직과는 거리가 멀고, SQL 쿼리와 쿼리결과를 비선언적인 방식으로 직접 파싱해야 한다.

이러한 문제점들은 AOP, 여러가지 패턴들을 도입해서 어느정도 해결할 수 있지만 SQL 쿼리를 작성하는 것과 결과를 객체로 다시 매핑하는데 생길 수 있는 실수와 생산성 저하는 여전히 문제점으로 남는다.

ORM(Object-Relational Mapping)은 이러한 **데이터베이스 통신과 객체 지향 프로그래밍 간의 간극을 줄이기 위해 생겨났다.**  ORM을 사용하면 데이터베이스와의 상호작용을 객체지향적인 방식으로 이용할 수 있다.

JPA(Java Persistence API)는 이러한 ORM을 구현하기 위한 표준 인터페이스로 **객체와 관계형 데이터베이스간의 불일치를 해소하고 객체로 데이터를 다룰 수 있도록 도와**준다. JPA는 인터페이스(명세)로 다양한 구현체 프레임워크가 존재하는데 흔히 Hibernate를 사용한다.

<br />  

### JPA로 DB와 통신하기
JPA는 관계형 데이터베이스의 테이블과 객체를 매핑하며 객체지향의 방식으로 데이터베이스 통신을 도와준다. 관계형 데이터베이스에서의 테이블은 Entity라고 부르는 객체에 매핑한며 해당 객체와 JPA가 제공하는 `EntityManager`를 통해 객체지향적인 방식으로 데이터베이스와 통신을 한다.

`EntityManager`는 `EntityManagerFactory`를 통해 만들어낸다. 이 둘은 이후에 이어서 설명하니 지금은 객체지향의 방식으로 DB와 통신하는 것이 EntityManager이고 이러한 EntityManager를 만들어내는 공장이 `EntityManagerFactory`라고만 이해해두자.

```java
public class Main {

    public static void main(String[] args) {
        // EntityManagerFactory 생성, EntityManager 생성
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("myPersistenceUnit");
        EntityManager em = emf.createEntityManager();

        // 트랜잭션 시작
        EntityTransaction tx = em.getTransaction();
        tx.begin();

        try {
            // 데이터베이스에 저장할 엔티티 생성
            Member member = new Member();
            member.setId(1L);
            member.setName("Rok");
            member.setAge(25);

            // 엔티티를 영속화(Persistence Context에 추가)
            em.persist(member);
			
			// Select 쿼리도 객체 이용
	        TypedQuery<Member> query = entityManager.createQuery("SELECT m FROM Member m", Member.class);
	        List<Member> memberList = query.getResultList();
	        
            tx.commit();
        } catch (Exception e) {
            // 예외 발생 시 롤백
            tx.rollback();
        } finally {
            // EntityManager 종료
            em.close();
        }

        // EntityManagerFactory 종료
        emf.close();
    }
}
```

`EntityManagerFactory`로 `EntityManager`를 생성하고 이를 이용한다. `Member`는 Entity로 관계형 데이터베이스의 테이블과 매핑된다. create를 할때도 객체(Entity)를 사용하고 select 쿼리에서도 객체를 사용하는 것을 볼 수 있다. 

`EntityManager`는 이름에서 유추할 수 있듯이 Entity를 관리하며 내부적으로 JDBC API를 사용하여 데이터베이스와 통신을 한다. 또한  **객체(Entity)와 관계형 데이터베이스 사이의 패러다임 불일치를 해결**해준다. 덕분에 JPA 사용한다면 이 `EntityManager`가 제공하는 기능으로 대부분의 데이터베이스 통신을 객체지향적인 방식으로 사용할 수 있다.

> **Hibernate에서 EntityManager**  
> Hibernate에서는 EntityManager를 Session(EntityManager를 사용받은 인터페이스)이라는 이름으로 사용한다. EntityManagerFactory는 SessionFactory라는 이름으로 사용한다. 이들의 구현체는 이름 뒤에 Impl을 붙혀서 사용하는 것을 볼 수 있다.

![JPA](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/f54608c9-6c43-4d5f-9962-ca80e0b32095){: .align-center style="width: 90%;"}  
JPA EntityManager
{: .image-caption style="font-size: 14px;" }  

<br />  

### EntityManager와 PersistenceContext
`EntityManager`에는 `PersistenceContext`(이하 영속성 컨텍스트)라는 공간이 존재한다. 이 공간은 Entity(RDB의 테이블과 매핑되는 객체)들의 상태를 저장하고 관리하는 컨테이너이다.  

우리는 이 Entity로 데이터를 검색(select), 새로운 데이터를 생성(create), 기존의 데이터를 수정(update), 데이터를 삭제(delete) 하는데 **영속성 컨텍스트는 이 Entity의 저장공간이자 상태를 추적하여 데이터베이스와 효율적으로 통신할 수 있도록 돕는다.**

영속성 컨텍스트의 존재로 다음과 같은 이점을 얻을 수 있다.
1. **쓰기 지연**
	- Entity 저장(영속화)시에 곧바로 DB와 통신하지 않고 커밋 시(정확히는 flush)에 통신한다.
	- 이후에 Entity의 데이터가 변경될 수 있다. 만약 즉시 통신한다면 2번의 쿼리가 발생되는데 쓰기 지연으로 인해 통신을 1번으로 줄일 수 있다.
2. **변경 감지(더티 체킹)**
	- Entity의 정보가 수정된다면 후에 커밋(정확히는 flush)시에 변경을 감지하고 수정 쿼리를 만든다.
2. **1차 캐시**
	- 만약 검색하고자하는 정보가 영속성 컨텍스트 내부에 있다면 이를 사용한다.
3. **지연로딩**
	- Entity와 연관된 객체는 한 번에 불러오지 않고 사용시에 불러올 수 있는 지연로딩을 제공한다.

<br />  

### EntityManager를 생성하는 Factory
`EntityManager`를 사용할때는 상황마다 개별로 생성해서 사용한다. 이유는 `EntityManager`는 **쓰레드 안정성을 보장하지 않기 때문이다.** 여러 쓰레드 하나를 공유해서 사용하면 내부에서 관리하는 Enitty에 동시성 문제가 생길 수 있다. 동시성 문제로 DB의 실제 데이터와의 일관성이 깨지면 매우 치명적일 것이다.

또한 하나의 `EntityManager`는 **하나의 트랜잭션 범위를 담당**한다. 커밋을 하거나 문제가 생겼을 때의 롤백을 하는 범위가 된다. 쓰레드간에 를 공유해서 사용한다면 커밋과 롤백의 범위를 지정하기 어려울 것이다.

이러한 문제로 필요할 때 새로운 `EntityManager`를 생성해서 사용하는데 이 `EntityManager`를 생성하는 객체가 `EntityManagerFactory`이다. 이 **팩토리 객체는 생성 비용이 비싸고 Thread-safe하기 때문에 하나만 만들어서 공유하는 것이 좋다.**

우리가 직접 `EntityManagerFactory`를 생성하고 공유하는 과정을 구현하지는 않는다. 역시나 스프링 컨테이너가 알아서 만들어준다. 하지만 추후 이 팩토리를 직접 등록해야하는 경우도 있다.(DataSource가 2개 이상, 즉 2개 이상의 DB를 사용할때) 때문에 어떤 과정을 통해 등록되는지 알아두면 좋지 않을까한다.  

<br />  

## JPA를 위한 컨테이너의 지원
`application.yml` 프로퍼티 파일에 datasource에 대한 설정만 있으면 `EntityManager`를 주입받아 사용할 수 있다. 스프링 컨테이너는 실행 시 `EntityManagerFactory`와 `EntityManager`를 셋팅하고 필요에 따라 상황마다 `EntityManager`를 만들어 준다.  

<br />  

### EntityManagerFactory를 생성하는 팩토리 빈
`EntityManagerFactory`는 컨테이너 수준에서 싱글톤 빈으로 만들어 어플리케이션 전반에 하나의 객체를 공유한다. 다만, `EntityManagerFactory`는 그 자체가 빈으로 등록되는 것이 아닌 **팩토리빈에 의해 만들어져 빈으로 등록**된다.

> **FactoryBean< T > 타입**  
> 스프링의 빈 종류에는 일반 Bean과 FactoryBean이 있다. FactoryBean은 **스프링 컨테이너에서 T 타입 빈을 생성하는 방법을 제어하고 빈의 초기화 및 설정을 담당한다**. 여러가지 외부 의존성, 외부 설정을 연결해야하는 등 복잡한 빈 설정 및 커스터마이징을 해야하는 경우에 FactoryBean을 구현하여 특정 Bean을 만들어낸다.

`EntityManagerFactory`는 `LocalContainerEntityManagerFactoryBean`이라는 팩토리빈에 의해 싱글톤 빈으로 생성된다. `LocalContainerEntityManagerFactoryBean`은 **DataSource, Entity 정보를 받아 Vendor사(Hibernate 등)에 맞는 `EntityManagerFactory`를 생성**한다. 가장 많이 쓰이는 Hibernate의 구현체는 `SessionFactoryImpl`이다. 

```java
public class LocalContainerEntityManagerFactoryBean extends AbstractEntityManagerFactoryBean implements ResourceLoaderAware, LoadTimeWeaverAware {

	// Entity 정보 셋팅
	public void setManagedTypes(PersistenceManagedTypes managedTypes) {
		this.internalPersistenceUnitManager.setManagedTypes(managedTypes);  
	}
	
	// DataSource 셋팅
	public void setDataSource(DataSource dataSource) {  
	   this.internalPersistenceUnitManager.setDataSourceLookup(new SingleDataSourceLookup(dataSource));  
	   this.internalPersistenceUnitManager.setDefaultDataSource(dataSource);  
	}
	
	@Override  
	protected EntityManagerFactory createNativeEntityManagerFactory() throws PersistenceException {  
		// ... EntityManagerFactory 생성
	}

}
```

여기서 볼 수 있듯이 한 `EntityManagerFactory`당 하나의 DataSource와 연결되어 있기 때문에 만약 DB를 2개 이상 사용한다면 스프링 부트를 사용한다고 해도 `LocalContainerEntityManagerFactoryBean`를 따로 빈으로 등록해줘야한다.

<br />  

### 공유 EntityManager 생성과 의존성 주입
JPA를 사용할 때, `@PersistenceUnit`, `@PersistenceContext` 어노테이션을 이용해 EntityManagerFactory와 EntityManager를 주입받는다. 이 두 어노테이션은 어플리케이션 생성 중 `PersistenceAnnotationBeanPostProcessor` 빈후처리기에 의해 처리된다.

`@PersistencUnit` 어노테이션이 달리 필드에는 `LocalContainerEntityManagerFactoryBean`로 생성된 공유 EntityManagerFactory(Hibernate에서 SessionFactoryImpl)를 주입한다.

`@PersistenceContext` 어노테이션에는 EntityManager(SessionImpl)를 주입하는데 여기서 **특이한 점은 SeesionImpl의 프록시 객체를 생성하여 주입한다는 것**이다. 해당 빈후처리기에서는 이 프록시 객체를 공유 EntityManager로 하여 모든 필드에 이 공유 프록시 객체를 주입한다.

`PersistenceAnnotationBeanPostProcessor`의 내부 코드를 보면 먼저, 필드가 `@PersistenceContext` 어노테이션을 가지는지 확인하고 `SharedEntityManagerCreator` 객체의 createSharedEntityManager 메서드를 통해 프록시 객체를 생성한다. 

```java
/* SharedEntityManagerCreator.java */

public static EntityManager createSharedEntityManager(EntityManagerFactory emf, @Nullable Map<?, ?> properties,  
      boolean synchronizedWithTransaction, Class<?>... entityManagerInterfaces) {  
   // ...
   // 프록시 객체 생성
   return (EntityManager) Proxy.newProxyInstance(  
         (cl != null ? cl : SharedEntityManagerCreator.class.getClassLoader()),  
         ifcs, new SharedEntityManagerInvocationHandler(emf, properties, synchronizedWithTransaction));  
}

```

`EntityManager`는 트랜잭션별로 다른 EntityManager를 생성해야한다. 하지만 객체가 주입되는 시점에서 어떤 트랜잭션을 사용하게 될지 모른다. 때문에 **프록시 객체를 두고 해당 객체를 호출할때 현재 트랜잭션에서 사용되고 있는 EntityManager를 리턴하는 형식**이다.

정리해보자면,
1. 빈후처리기가 동작하는 단계에서 `PersistenceAnnotationBeanPostProcessor` 후처리기가 동작
2. `@PersistencUnit`, `@PersistenceContext` 어노테이션 필드에 값을 주입함
3. `@PersistenceContext` 경우엔 공유 프록시 `EntityManager`를 만들어 주입
4. 주입받은 `EntityManager`를 사용할때 내부에서 상황(트랜잭션)에 맞는 실제 `EntityManager`를 리턴한다.

> **트랜잭션별 EntityManager**  
> 트랜잭션 내에서 어떤 `EntityManager`가 사용되고 있는지 어떻게 알고 리턴할 수 있는 걸까? 이는 트랜잭션 동기화와 관련된 내용으로 다음 포스팅에서 설명할 예정이다.

<br />  

### 주입 받은 EntityManager 사용하기
 `LocalContainerEntityManagerFactoryBean` 팩토리빈을 통해 공유 `EntityManagerFactory`를 만들어 내고 이를 이용해서 모든 `@PersistenceContext` 어노테이션이 달린 `EntityManager` 필드에 공유 프록시 `EntityManager`를 주입하는 것까지 살펴보았다.

이제 DB통신을 하기 위해 이 `EntityManager`를 사용해보자. createQuery 메서드를 이용하여 쿼리를 만들면 프록시 객체를 만들때 넘겨준 `SharedEntityManagerInvocationHandler` 객체의 `invoke` 함수를 호출한다. 

![invoke 함수 호출](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/e24b2689-6b25-478b-ae0b-7e516671f41b){: .align-center style="width: 100%;"}  
SharedEntityManagerInvocationHandler invoke 함수
{: .image-caption style="font-size: 14px;" }  

이 invoke 함수는 내부에서 `EntityManagerFactoryUtils`를 통해서 현재 트랜잭션에서 사용중인 실제 `EntityManager`를 가져와서 메서드를 처리하는 것을 볼 수 있다. 

```java
/* SharedEntityManagerInvocationHandler 클래스 */

@Override  
@Nullable  
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
	// ...
	
	// 현재 트랜잭션에서 사용하는 EntityManager 가져오기
	EntityManager target = EntityManagerFactoryUtils.doGetTransactionalEntityManager(this.targetFactory, this.properties, this.synchronizedWithTransaction);  
	
	// ...
   
	try {  
		// 메서드(createQuery) 실행
		Object result = method.invoke(target, args);  
		// ...
		return result;  
	}  
}
```

여기서 적절한 `target`을 가져오기 위해 트랜잭션 동기화가 필요한데 이는 다음 포스팅에서 더 자세하게 알아보자.

<br />  

### 정리
스프링 부트 어플리케이션을 실행하게 되면 어떤 과정을 거쳐서 우리가 사용하는 JPA가 셋팅되는지 정리해보자.

1. 스프링 어플리케이션을 실행하면 AutoConfiguration을 통해 `JpaBaseConfiguration`이 빈으로 등록됨
2. 해당 Configuration에서 `LocalContainerEntityManagerFactoryBean` 빈을 생성함
3. 빈 생성 이후 빈후처리기가 동작 - `PersistenceAnnotationBeanPostProcessor` 동작
4. Persistence 어노테이션들을 읽고 주입
	- 이때 `@PersistenceContext`에는 공유 프록시 `EntityManager` 객체가 주입됨
	- Spring Data JPA를 사용할때 JpaRepository의 구현체인 SimpleJpaRepository에서도 `EntityManager`가 존재하는데 여기에도 이 프록시 객체가 주입됨
5. 스프링 어플리케이션 구동 완료
6. 요청이 들어오고 트랜잭션 시작, 프록시 `EntityManager`를 통한 DB 통신 시작
7. 내부에서 invoke 함수 호출
	- `EntityManagerFactoryUtils`를 통해 현재 트랜잭션에서 사용중인 `EntityManager` 불러옴
	- 실제 `EntityManager`를 통해 DB 통신
 8. 트랜잭션 완료 및 요청에 대한 응답 완료
