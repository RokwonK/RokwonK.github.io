---
title: "Body, Query, Entity에 포함된 Legacy Enum 효율적으로 관리하기(3) Entity편"
excerpt: "Jpa Entity에서 DB에 사용중인 Legacy Enum들을 깔끔하고 효율적으로 관리해보자"

categories:
  - Project

permalink: /project/legacy-enum-management-3/
---

<br />  

요청/응답 Body와 Query에 담긴 Legacy Enum을 다루는데 성공하였다. 남은 건 DB에 존재하는 Enum 뿐.

과거의 실수로 MySQL DB의 컬럼정의가 mysql enum타입인 것이 몇 개 존재한다. 그리고 그 값들도 전부 소문자이다. 현재 마이그레이션 되는 프로젝트에서는 JPA를 사용하고 있어 JPA Entity와 DB의 테이블 정의를 일치시켜 줘야한다.

현재로서는 mysql enum 타입을 다른 타입으로 마이그레이션하기에는 너무 수고롭다. 우선은 DB를 직접 건드리지 않고 어플리케이션에서 Enum으로 처리하기로 결정했다.

코틀린의 Enum 값은 대문자 사용을 권장한다. 때문에 DB와 소통시 대문자 Enum <-> 소문자 문자열로 변환할 필요가 있다. 이를 다루기 위해 구현한 방법을 공유해본다.

> Body, Query, Entity 부분을 각각 포스팅을 나누어 작성하였습니다.
> - [요청/응답 Body에 포함된 Legacy Enum 관리하기](https://rokwonk.github.io/project/legacy-enum-management-1/)
> - [Query에 포함된 Legacy Enum 관리하기](https://rokwonk.github.io/project/legacy-enum-management-2/)
> - **(Now) Entity에서 DB에 사용중인 Legacy Enum 관리하기**

<br/>

### JPA Entity에서 Enum 사용시
JPA Entity에서 Enum을 사용을 한다면 다음과 같은 방식을 이용한다. `@Enumerated(EnumType.STRING)`을 통해 Enum 값이 그대로 문자열로 변환되어 DB값에 적용된다.

`@Enumerated(EnumType.ORDINAL)` 사용할 수도 있지만 이는 Enum의 순서 값(1, 2)이 DB값으로 들어간다. 가독성도 확장성도 좋지 못해 많이 사용되지는 않는다.
```kotlin
@Entity  
@Table(name = "users")  
class UserEntity(  
    // ...
  
    @Column(name = "status")
    @Enumerated(EnumType.STRING)
    var status: UserStatus = UserStatus.ACTIVE,
    
    // ...
)
```

'개발 한 스푼' 어플리케이션에서 문제는 mysql enum 타입을 사용하고 있으며 값이 소문자라는 것이다. mysql enum 타입의 경우 다음과 같이 컬럼정의를 직접 넣어줘야 문제가 발생하지 않는다.
```kotlin
@Column(name = "status" , columnDefinition="ENUM('active','exit')")
var status: UserStatus = UserStatus.ACTIVE,
```

아이러니하게 Enum 값을 소문자로 작성해도 DB에서 Entity로 변환시 대문자 Enum값을 찾는다. MySQL 버전에 따라 달라지는 것 같기도..? 그래서 잠깐이나마 다음과 같이 쓸 생각도 해봤다... 돌아가긴 하더라..
```kotlin
enum class UserStatus {  
    ACTIVE, EXIT, active, exit;
}
```

<br/>

### 커스텀이 필요할때
방법을 강구하다 `AttributeConverter`의 존재를 알 수 있었다.(동시에 우아한 형제들의 테크블로그도 굴러 들어왔다.)

`AttributeConverter` 인터페이스는 Entity의 속성과 DB 컬럼값 간 커스텀 변환로직을 만들 수 있도록 도와준다. X타입은 Entity 속성의 타입이며 Y타입은 DB 커럼의 타입이다. 해당 인터페이스 하나로 양방향 변환을 해결할 수 있다.
```kotlin
public interface AttributeConverter<X,Y> {
	public Y convertToDatabaseColumn (X attribute);
    public X convertToEntityAttribute (Y dbData);
}
```

<br/>

### Legacy Enum 다루는 AttributeConverter 작성하기
대소문자간 변환이 필요한 Legacy Enum들이 N개가 존재한다. 모든 Enum들에 대소문자로 변환하는 로직을 반복해서 작성할 수는 없는법! [앞선 포스팅들](https://rokwonk.github.io/project/legacy-enum-management-1/)처럼 **공통 로직 구현체를 재사용하는 방향**으로 만들어보자.

우선 inteface를 만들어 Legacy Enum임을 표시하도록 하자. 필자의 로직에서 인터페이스를 만들지 않아도 상관없었다. 하지만 인터페이스로 제약조건을 걸어 Legacy가 아닌 다른 Enum에서 실수로 적용하는 일을 막기 위해 사용하기로 했다.
```kotlin
interface LegacyEntityEnum
```

Legacy Enum을 처리하는 공통로직을 가진 `AttributeConverter`는 다음과 같이 작성했다. Entity에서 DB 컬럼으로 보낼땐 단순히 소문자로 바꿔 처리하면 되지만, DB 컬럼 값을 Enum에 매핑하기 위해서는 값을 순회하여 컬럼의 대문자와 같은 값을 사용한다.

```kotlin
abstract class LegacyEntityEnumConverter<T>(  
    private var enumType: Class<T>,  
    private var nullable: Boolean = false  
) : AttributeConverter<T?, String?> where T: Enum<T>, T:LegacyEntityEnum {  
    override fun convertToDatabaseColumn(attribute: T?): String? {  
        if (!nullable && attribute == null) {  
            throw DomainInvalidAttributeException(type = enumType.simpleName)  
        }  
        
        return attribute?.name?.lowercase()  
    }  
    
    override fun convertToEntityAttribute(dbData: String?): T? {  
        if (!nullable && dbData == null) {  
            throw DomainInvalidAttributeException(type = enumType.simpleName)  
        } else if (nullable && dbData == null) {  
            return null  
        }  
        
        val data = dbData?.uppercase()  
        return enumType.enumConstants  
            .firstOrNull { it.name == data }  
            ?: throw DomainInvalidAttributeException(type = enumType.simpleName)  
    }  
}
```

아쉬운 점은 `AttributeConverter`를 사용시 구체적인 타입을 받아올 수 없다. 때문에 각 Legacy Enum에서 해당 Converter를 이용해 구현하면서 구체적인 타입(enumType)을 넘겨줘야 한다.

추가적으로 Enity의 속성이 null이 가능할 수 있기 때문에 nullable 이라는 파라미터도 추가하였다.  

<br/>

### 적용하기
각 Enum에서 해당 Converter를 이용하도록 한다. 구체적인 타입을 넘겨주고 @Converter 어노테이션과 autoApply 속성을 true로 두면 자동으로 등록되어 처리된다.
```kotlin
enum class UserStatus: LegacyEntityEnum {  
    ACTIVE, EXIT;  
  
    @Converter(autoApply = true)  
    class LegacyConverter: LegacyEntityEnumConverter<UserStatus>(UserStatus::class.java, true)  
}
```

Enum마다 Converter를 작성해야하긴 하지만 같은 변환 로직을 또 다시 구현하지 않아도 된다는 점이 매력만점이다. 이것으로 인터페이스를 이용한 3종류(Body, Query, Entity)의 Legacy Enum을 다뤘던 경험공유를 마친다.

<br/>

### 참고
우아한 형제들의 테크 블로그에서 많은, 대부분의 영감을 얻었다. 해당 블로그보다는 훨씬 쉬운 상황이라 구현하기에는 간편했던거 같다.
 - [우아한형제들 - Legacy DB의 JPA Entity Mapping](https://techblog.woowahan.com/2600/)

<br/>