---
title: "Spring Boot - 스프링이 데이터베이스 통신을 지원하는 방법"
excerpt: "JDBC부터 JPA, 트랜잭션까지.. 스프링 컨테이너는 어떻게 데이터베베이스 통신을 지원할까?"

categories:
  - Spring

permalink: /spring/spring-boot-database-interaction/
---

<br />  

스프링은 데이터베이스와의 상호작용을 위한 여러가지 기술들을 제공해 주어 편리하게 개발을 할 수 있도록 도와준다. 워낙 간편하게 코드를 작성할 수 있도록 도와주다보니 간편한 로직들을 막 작성해도 알아서 잘 돌아가더라.. 

비교적 복잡한 로직을 작성할때도 대부분은 블로깅을 통해 쉽게 구현이 가능해 내부동작을 이해하지 못하고 사용하는 경우도 많았다. 매번 궁금증이 들기도 했지만 프로젝트의 완성이 우선이다보니 호기심은 구석에 던져놓는 경우가 다반사였다.

근래 코드를 다시 읽던 중에 과거의 들었던 궁금증이 다시 떠올라 이번 기회에 모두 정리하기로 마음먹었다. 내부 소스코드도 뜯어보면서 데이터베이스 통신의 기본이 되는 **JDBC부터 시작해서 트랜잭션 동기화, Spring Data JPA까지 모두 정리**해볼까 한다.

<br />  

## JDBC
JDBC(Java Database Connectivity)는 자바에서 데이터베이스와 상호작용할 수 있게 도와주는 표준 API이다. 추상화된 JDBC API를 이용해 우리는 데이터베이스와 연결하고 SQL을 실행 및 결과를 받을 수 있다. **이런 JDBC의 가장 큰 장점은 통일된 인터페이스를 이용해 다양한 데이터베이스와의 상호작용을 동일한 방식한 방식으로 통신할 수 있도록 도와준다는 것이다.**

Oracle, MySQL 등 각 데이터베이스마다 프로토콜(SQL전달 및 응답처리)이 조금씩 다르다. 때문에 통일된 인터페이스를 제공하지 않는다면 각 데이터베이스에 의존된 코드를 작성할 수 밖에 없다. JDBC 덕분에 개발자는 데이터베이스 종류에 따른 의존도를 줄일 수 있다. 

<br />  

### Driver - DB연결 추상화
JDBC는 `Driver`라는 인터페이스를 통해 데이터베이스 연결을 추상화한다. 각 데이터베이스용 Driver를 상속받는 구현체가 존재하고 개발자는 DriverManager 클래스를 통해 커넥션을 얻어 데이터베이스와 상호작용할 수 있다. 

![JDBC](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/149631c6-6221-4ec4-839c-0604759c7c38){: .align-center style="width: 50%;"}  
JDBC
{: .image-caption style="font-size: 14px;" }  


통신할 데이터베이스에 맞는 Driver를 다운받기만 하면 DriverManager에서 이를 자동으로 인식하고 Connection을 받아올 수 있다. 아래 코드는 JDBC를 이용하여 MySQL 데이터베이스와 통신하는 과정이다.

```java
public class JDBCDemo {

    // 데이터베이스 연결 정보
    private static final String JDBC_URL = "jdbc:mysql://localhost:3306/your_database_name";
    private static final String USERNAME = "your_username";
    private static final String PASSWORD = "your_password";

    public static void main(String[] args) {
        Connection connection = null;
        PreparedStatement preparedStatement = null;
        ResultSet resultSet = null;

        try {
            // 1. 데이터베이스 연결
            connection = DriverManager.getConnection(JDBC_URL, USERNAME, PASSWORD);

            // 2. SQL 쿼리 준비
            String sqlQuery = "SELECT * FROM your_table_name WHERE condition_column = ?";
            preparedStatement = connection.prepareStatement(sqlQuery);
            preparedStatement.setString(1, "some_value");

            // 3. SQL 실행 및 결과 얻기
            resultSet = preparedStatement.executeQuery();

            // 4. 결과 처리
            while (resultSet.next()) {
                // 결과 데이터 읽기
                int id = resultSet.getInt("id");
                String name = resultSet.getString("name");
                // 필요한 작업 수행
                System.out.println("ID: " + id + ", Name: " + name);
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            // 자원 해제
            try {
                if (resultSet != null) {
                    resultSet.close();
                }
                if (preparedStatement != null) {
                    preparedStatement.close();
                }
                if (connection != null) {
                    connection.close();
                }
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }
}

```

1. `Connection` 획득 - DriverManager를 통해 DB연결을 받아온다.
2. `Statement` 작성 - 실행할 쿼리를 준비한다.
3. `Statement` 실행 - 쿼리를 실행하여 DB를 통해 결과를 얻어온다.
4. `ResultSet` 읽기 - 쿼리결과는 ResultSet에 담기며 이를 읽어 필요한 작업을 수행한다.

모든 작업이 끝난후에는 사용하던 자원을 할당 해제시켜준다.

JDBC 덕분에 데이터베이스와의 의존성은 확실히 줄어들었지만 보다시피 한 번의 통신에 코드에 양이 방대하다. 연결 설정, 자원반납 부분에서 보일러 플레이트 코드들이 보이고 결과를 처리하는 부분에서도 자료형, 데이터 검증의 필요성이 느껴진다. 무엇보다도 **DB와 상호작용이 필요할 때마다 커텍션을 생성하는데 이에 대한 관리가 필요**해 보인다.

<br />  

## DataSource
JDBC API에서 DB와 통신이 필요할 때 `DriverManager`를 통해 커넥션을 얻어왔다. 이 방식은 요청시 마다 커넥션을 생성하는데 이런 경우에 몇 문제점이 존재한다.
1. DB와의 커넥션은 그 생성 비용이 크다. 
2. 무분별한 커넥션 생성은 데이터베이스에 큰 부하를 줄 수 있다.
3. 필요하다면 timeout과 같은 설정을 할 수 있다. 매 커넥션마다 이러한 커넥션 설정을 하는 것은 비효율적이다.

이러한 문제 때문에 **데이터베이스와의 커넥션을 관리하는 계층**을 하나 두는데 이 계층이 바로 `DataSource`이다. `DataSource` 또한 추상화된 계층으로 필요에 따라 다른 커넥션을 관리하는 방식을 사용할 수 있다. 대표적인 구현체로는 DriverManagerDataSource와 HikariDataSource가 있다.

<br />  

### HikariDataSource와  Connection Pool
`DriverManagerDataSource`는 기존 `DriverManager`에 비해 커넥션에 대한 설정을 중앙 관리해준다. 테스트용도로만 사용한다고 하니 참고만 해두자.

`HikariDataSource`는 Spring Boot에서 기본적으로 사용하는 DataSource이다. 이 DataSource의 가장 큰 매력으로는 **Connection Pool**이 있다.(줄여서 CP, HikariCP 라고 부른다)

Connection Pool은 어플리케이션이 실행될 때, **미리 설정된 갯수만큼의 커넥션을 만들어 저장해 두는 공간**이다. 만약 어플리케이션에서 커넥션을 사용하고자 한다면 이 Connection Pool에서 커넥션을 꺼내 사용한 뒤 할당을 해제하는게 아니라 Connection Pool에 반납한다. 반납한 커넥션은 다른 요청에서 재사용된다. Connection Pool 덕분에 커넥션 생성 비용을 줄일 수 있다. 

![Connection Pool](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/5a945227-2754-4b2c-a614-5b0fc5b80dd4){: .align-center style="width: 90%;"}  
Connection Pool
{: .image-caption style="font-size: 14px;" }  

실제로 `HikraiDataSource` 내부에서는 `HikariPool`을 유지하고 커넥션을 꺼내서 사용하는 걸 볼 수 있다.  

```java
public class HikariDataSource extends HikariConfig implements DataSource, Closeable  
{  
   private final HikariPool fastPathPool;  
   private volatile HikariPool pool;  
  
   public HikariDataSource(HikariConfig configuration) {  
      // 커넥션 풀 생성
      pool = fastPathPool = new HikariPool(this);  
   }  
  
   @Override  
   public Connection getConnection() throws SQLException  
   {  
      // 커넥션 풀에서 커넥션 가져오기
      if (fastPathPool != null) {  
         return fastPathPool.getConnection();  
      }  
      
      HikariPool result = pool;  
      // ...
  
      return result.getConnection();  
   }
   // ...
}
```

`HikraiDataSource`는 Connection Pool 외에도 여러가지 기능으로 커넥션을 관리해준다.
1. Connection Pool의 커넥션을 모두 사용했을 경우 추가적인 커넥션 생성 가능
2. 요청이 많아 사용할 수 있는 커넥션이 존재하지 않는 경우 대기하는 큐 존재
3. 요청 timeout 관리
4. 커넥션은 DB와의 연결 유효시간(maxLifeTime)이 존재. 커넥션 만료 및 새로운 커넥션 생성으로 Connection Pool 유지

사용방법이 크게 달라지지는 않지만 DataSource를 이용한 DB 통신 코드를 봐보자. DataSource에 대한 설정은 생략했다.

```java
@Component
public class DatabaseExample {

    private final DataSource dataSource;

    @Autowired
    public DatabaseExample(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public void fetchDataFromDatabase() {
        try (Connection connection = dataSource.getConnection();
             Statement statement = connection.createStatement();
             ResultSet resultSet = statement.executeQuery("SELECT * FROM your_table_name")) {

            while (resultSet.next()) {
                int id = resultSet.getInt("id");
                String name = resultSet.getString("name");
                System.out.println("ID: " + id + ", Name: " + name);
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

`getConnection()` 메서드에서 내부적으로는 커넥션을 재사용한다. 이러한 부분을 제외하고는 `DriverManager`를 사용할때와 사용방법은 똑같다. 커넥션을 받아오는 객체만 달라졌을 뿐이다. 코드가 간결해보이는 이유는 자원 할당 해제하는 부분을 Java7부터 지원되는 `try-with-resources`를 이용하였기 때문이다.

> **try-with-resources**  
> `AutoClosable` 인터페이스를 구현한 객체들을 try 구문이 끝난후 close() 메서드를 호출해준다. Connection, Statement, ResultSet 모두 `AutoClosable`를 구현한다.  

<br />  

### 정리(JDBC와 DataSource)
지금까지 `JDBC`와 `DataSource` 내용을 정리해보자.
1. `JDBC`는 **다양한 데이터베이스를 일관된 방식으로 접근할 수 있도록 도와**준다.
	- Dirver라는 인터페이스가 DB 연결에 대한 추상체이다.
	- 각 DB에 맞는 Driver 구현체 라이브러리를 다운받기만 하면 자동으로 인식하고 적용된다.
2. 개발자는 `DriverManager`를 통해 DB와 커넥션을 맺고 이를 사용할 수 있다.
	- 하지만 DB와의 커넥션은 직접적으로 생성하고 이용하는 것은 굉장히 민감한 문제이다.
3. `DataSource`라는 **커넥션 관리 레이어를 두고 이를 활용하는 것이 효과적**이다.
4. `HikariDataSource`는 `DataSource`의 구현체로 **커넥션 풀링으로 효율적으로 커넥션을 사용할 수 있도록 돕는다.** 이 DataSource는 기본으로 적용되는 구현체이며 여러가지로 커넥션 관리를 지원한다.
5. 따라서, 우리는 `DataSource` 타입으로 의존성을 주입받아 이를 통해 커넥션을 받아와 데이터베이스와 상호작용을 진행하면 된다.

![Datasource](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/7dbb3b48-a24c-4566-8454-63f6163e1226){: .align-center style="width: 70%;"}  
DataSource
{: .image-caption style="font-size: 14px;" }  

<br />

## JPA
JDBC가 편리한 데이터베이스 소통을 지원해주지만 실제로 어플리케이션을 개발할때 생산성이 떨어지는 문제점들이 존재한다. 어플리케이션이 추구하는 객체지향적인 로직과는 거리가 멀고, SQL 쿼리와 쿼리결과를 비선언적인 방식으로 직접 파싱해야 한다.

이러한 문제점들은 AOP, 여러가지 패턴들을 도입해서 어느정도 해결할 수 있지만 SQL 쿼리를 작성하는 것과 결과를 객체로 다시 매핑하는데 생길 수 있는 실수와 생산성 저하는 여전히 문제점으로 남는다.

ORM(Object-Relational Mapping)은 이러한 **데이터베이스 통신과 객체 지향 프로그래밍 간의 간극을 줄이기 위해 생겨났다.**  ORM을 사용하면 데이터베이스와의 상호작용을 객체지향적인 방식으로 이용할 수 있다.

JPA(Java Persistence API)는 이러한 ORM을 구현하기 위한 표준 인터페이스로 **객체와 관계형 데이터베이스간의 불일치를 해소하고 객체로 데이터를 다룰 수 있도록 도와**준다. JPA는 인터페이스(명세)로 다양한 구현체 프레임워크가 존재하는데 흔히 Hibernate를 사용한다.  

> **❗️ 이 파트에서 다루는 내용**  
> 이 포스팅의 JPA 파트에서는 **스프링 컨테이너가 '개발자가 JPA를 사용하는데 필요한 기술들'을 어떻게 제공하는지에 포커스를 두고 설명**한다. JPA 활용을 위한 여러 가이드라인과 관련된 포스팅은 따로 작성해 볼 예정이다.  


<br />  

### JPA로 DB와 통신하기
JPA는 관계형 데이터베이스의 테이블과 객체를 매핑하며 객체지향의 방식으로 데이터베이스 통신을 도와준다. 관계형 데이터베이스에서의 테이블은 Entity라고 부르는 객체에 매핑한며 해당 객체와 JPA가 제공하는 `EntityManager`를 통해 객체지향적인 방식으로 데이터베이스와 통신을 한다.

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
            member.setName("rokwon");
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
JPA - EntityManager로 통신
{: .image-caption style="font-size: 14px;" }  

<br />  

### EntityManager와 PersistenceContext
`EntityManager`에는 `PersistenceContext`(이하 영속성 컨텍스트)라는 공간이 존재한다. 이 공간은 Entity(RDB의 테이블과 매핑되는 객체)들의 상태를 저장하고 관리하는 컨테이너이다.  

우리는 이 Entity를 통해 데이터를 검색(select), 새로운 데이터를 생성(create), 기존의 데이터를 수정(update), 데이터를 삭제(delete) 하는데 **영속성 컨텍스트는 이 Entity의 상태를 추적하여 데이터베이스와 효율적으로 통신할 수 있도록 돕는다.**

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