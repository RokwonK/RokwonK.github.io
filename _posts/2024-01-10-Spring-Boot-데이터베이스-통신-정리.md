---
title: "Spring Boot - 데이터베이스 통신 정리(정리중)"
excerpt: "스프링 어플리케이션에서 데이터베이스 통신을 지원하기 위한 여러 기술들은 어떻게 동작하는가"

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
JDBC가 편리한 데이터베이스 소통을 지원해주지만 실제로 어플리케이션을 개발할때 여러가지로 생산성이 떨어지는 문제점들이 있다.
- 다양한 트랜잭션 처리에 대한 복잡도
	- DB 통신로직과 비즈니스로직의 강한 결합 트랜잭션처리를 위해 Connection을 주고 받아야함
- 생산성
	- SQL 쿼리를 직접 작성하고 관리해야하므로 더 많은 코드, 작업이 필요하다.
- 보일러 플레이트 코드 작성
	- 예외처리와 가튼 보일러 플레이트성 코드가 많음

위 문제점들도 AOP, 여러가지 패턴들을 도입해서 어느정도 해결할 수 있지만 SQL 쿼리를 직접 작성하는 것과 쿼리 결과를 객체로 매핑하는데 생기는 생산성 저하가 여전히 문제로 남는다.

이러한 **객체지향과 관계형 데이터베이스간의 불일치를 해소하여 DB와 객체지향적인 코드로 상호작용이 가능하게 만드는 것이 객체-관계 매핑(ORM)** 이다. 스프링 진영에서는 ORM의 표준 명세로 JPA(Java Persistence API)를 지원해주고 있다.  

<br />

### JPA의 동작
JPA는 관계형 데이터베이스의 테이블과 객체를 매핑하며 객체지향의 방식으로 데이터베이스 통신을 도와준다. 관계형 데이터베이스에서의 테이블은 Entity라고 부르는 객체에 매핑한다. 해당 객체와 JPA가 제공하는 `EntityManager`를 통해 객체지향적인 방식으로 데이터베이스와 통신을 한다.

`EntityManager`는 이름에서 유추할 수 있듯이 Entity를 관리하며 내부적으로 JDBC API를 사용하여 데이터베이스와 통신을 한다. 또한  **객체(Entity)와 관계형 데이터베이스 사이의 패러다임 불일치를 해결**해준다. 덕분에 JPA 사용시 이 `EntityManager`가 제공하는 인터페이스만으로 대부분의 데이터베이스 통신을 객체지향적인 방식으로 사용할 수 있다.

`EntityManager`에는 `PersistenceContext`라는 공간이 존재한다. 간단히 말해서, Entity 인스턴스들을 저장하고 관리하는 컨테이너이다. 이 컨테이너는 **Entity의 생명주기를 관리할 뿐 아니라 1차 캐시의 역할, 지연 로딩과 같은 장점도 제공한다.**(여러가지 생명주기 메서드에 대한 설명과 어떤 방식으로 지연로딩, 쓰기 지연이 일어나는지는 생략했다)

![JPA](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/f54608c9-6c43-4d5f-9962-ca80e0b32095){: .align-center style="width: 90%;"}  
JPA
{: .image-caption style="font-size: 14px;" }  

