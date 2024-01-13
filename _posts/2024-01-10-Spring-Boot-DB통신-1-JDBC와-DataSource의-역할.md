---
title: "Spring Boot DB통신(1) - JDBC와 DataSource의 역할"
excerpt: "JDBC와 DataSource은 왜 필요할까? 커넥션과 커넥션을 회수하는 방법"

categories:
  - Spring

permalink: /spring/spring-boot-db-1/
---

<br />  

스프링은 데이터베이스와의 상호작용을 위한 여러가지 기술들을 제공해 주어 편리하게 개발을 할 수 있도록 도와준다. 워낙 간편하게 코드를 작성할 수 있도록 도와주다보니 간편한 로직들을 막 작성해도 알아서 잘 돌아가더라.. 

비교적 복잡한 로직을 작성할때도 대부분은 블로깅을 통해 쉽게 구현이 가능해 내부동작을 이해하지 못하고 사용하는 경우도 많았다. 매번 궁금증이 들기도 했지만 프로젝트의 완성이 우선이다보니 호기심은 구석에 던져놓는 경우가 다반사였다.

근래 코드를 다시 읽던 중에 과거의 들었던 궁금증이 다시 떠올라 이번 기회에 모두 정리하기로 마음먹었다. 데이터베이스 통신의 기본이 되는 JDBC부터 하나씩 살펴보자.

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
            // 1. 데이터베이스 연결 생성
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

1. `Connection` 생성 - DriverManager를 통해 DB연결을 생성한다.
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

실제로 `HikraiDataSource` 내부에서는 `HikariPool`을 유지하고 이 내부에서 커넥션을 꺼내서 사용하는 걸 볼 수 있다.
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
1. Connection Pool의 커넥션을 모두 사용했을 경우 **추가적인 커넥션 생성 가능**
2. 요청이 많아 사용할 수 있는 커넥션이 존재하지 않는 경우 **대기하는 큐 존재**
3. 요청 **timeout 관리**
4. 커넥션은 DB와의 연결 유효시간(maxLifeTime)이 존재. **커넥션 만료 및 새로운 커넥션 생성**으로 Connection Pool 유지

사용방법이 크게 달라지지는 않지만 DataSource를 이용한 DB 통신 코드도 봐보자. DataSource에 대한 설정은 생략했다.

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

`getConnection()` 메서드에서 내부적으로는 커넥션을 재사용한다. `DriverManager`를 사용할때와 사용방법은 똑같다. 어디서 커넥션을 받아오는지만 달라졌으며, 내부적으로는 커넥션을 관리를 제공해줄 뿐이다.

자원 할당 해제하는 부분만 Java7부터 지원되는 `try-with-resources`를 이용하여 간결하게 표현하였다.

> **try-with-resources**  
> `AutoClosable` 인터페이스를 구현한 객체들을 try 구문이 끝난후 close() 메서드를 호출해준다. Connection, Statement, ResultSet 모두 `AutoClosable`를 구현한다.

<br />  

### DataSource의 Connection은 Proxy 객체
위 코드에서 재미있는 점은 DataSource로 받은 Connection을 `try-with-resources`로 자원 할당 해제한다는 것이다. try 구문이 종료되면 `close()` 메서드가 실행되고 `Connection`을 해제하는 것이다. 

그런데 이상하다. 지금까지 살펴본바 `Connection`은 자원할당해제가 아닌 Connection Pool로 자원반납이 이루어져야 한다. 사실 DataSource로 받아온 `Connection`은 Proxy 객체이다. 때문에 **Proxy 객체의 close() 메서드를 호출하고 이 메서드 내부에서 자원반납이 이루어진다.**

코드 내부를 살펴보자. 위에서 살펴보았듯 DataSource(HikariDataSoure)로 `getConnection()` 메서드를 호출하면 `HikariPool`로부터 커넥션을 받아온다. `HikariPool`의 `getConnection()` 메서드를 보면 아래와 같다. 

```java
public Connection getConnection() throws SQLException {  
   return getConnection(connectionTimeout);  
}  
  
public Connection getConnection(final long hardTimeout) throws SQLException {  
   // ... 코드 생략
   try {  
      var timeout = hardTimeout;  
      do {  
         var poolEntry = connectionBag.borrow(timeout, MILLISECONDS);
         // ...
         return poolEntry.createProxyConnection(leakTaskFactory.schedule(poolEntry));  
      } while (timeout > 0L);  
 
   }  
}
```

poolEntry는 Connection Pool에 관리되는 요소이며, 실제 Connection을 가지고 있는 객체이다. `createProxyConnection`이라는 이름답게 poolEntry를 통해 `Connection` 프록시 객체를 만들어 리턴한다. 이름도 `HikariProxyConnection`으로 아주 직관적이다.

![HikariProxyConnection](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/78239b68-9aee-418b-82e3-4477e6d147e4)

그럼 이제 `HikariProxyConnection`의 `close()` 메서드가 실제로 자원해제가 아닌 Connection Pool 커넥션을 반납하는지 살펴보자. `HikariProxyConnection`은 `ProxyConnection`을 상속받고 있는데 . 이 클래스에  `close()` 메서드가 구현되어 있다.

```java
// ProxyConnection.java

@Override  
public final void close() throws SQLException  
{  
   if (delegate != ClosedConnection.CLOSED_CONNECTION) {  
      try {  
      // ...
      }  
      catch (SQLException e) {  
         // ... 
      }  
      finally {  
         delegate = ClosedConnection.CLOSED_CONNECTION;  
         poolEntry.recycle();  
      }  
   }  
}
```

HikariPool(Connection Pool)에 관리되고 있는 poolEntry(내부적으로 Connection을 가지고 있음)의 `recycle()` 메서드를 통해 Connection을 재활용하고 있다는 것을 알 수 있다.

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
	- 받아온 Connection은 Proxy객체로 close 시 Connection Pool로 반납이 이루어진다.

![Datasource](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/7dbb3b48-a24c-4566-8454-63f6163e1226){: .align-center style="width: 70%;"}  
DataSource
{: .image-caption style="font-size: 14px;" }  