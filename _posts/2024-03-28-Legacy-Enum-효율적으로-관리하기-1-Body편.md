---
title: "Body, Query, Entity에 포함된 Legacy Enum 효율적으로 관리하기(1) Body편"
excerpt: "요청, 응답 Body 내 Legacy Enum들을 깔끔하고 효율적으로 관리해보자"

categories:
  - Project

permalink: /project/legacy-enum-management-1/
---

<br />  

운영중이던 '개발 한 스푼' 서비스의 API 서버를 서버리스 아키텍쳐에서 스프링 부트로 마이그레이션중이다.  
마이그레이션을 진행하면서 API 요청이나 응답 Body, Query 그리고 JPA Entity에 포함된 Enum들이 전부 소문자로 작성되어 있는 것이 문제가 되었다.

코틀린의 경우 Enum 값을 대문자로 작성하는 것을 권장하는데 기존 정보들만 소문자로 두기엔 뭔가 마음 한 편이 불편하고 그렇다고 모든 Enum의 값을 소문자로 쓸 수도 없는 노릇이다. 

이러한 Legacy Enum들을 조금은 선언적이고 효율적으로 관리하기 위해서 필자가 적용한 방식을 공유해본다.

> Body, Query, Entity 부분을 각각 포스팅을 나누어 작성하였습니다.
> - **(Now) 요청/응답 Body에 포함된 Legacy Enum 관리하기**
> - [Query에 포함된 Legacy Enum 관리하기]()
> - [Entity에서 DB에 사용중인 Legacy Enum 관리하기]()

<br />  

### 기존 Legacy Enum을 다룬 방법
REST API의 요청 body와 응답은 보통 JSON을 이용한다. 스프링 부트에서는 ObjectMapper(Jackson)를 통해서 요청 JSON을 정의한 객체로 역직렬화해주거나 응답객체를 JSON으로 직렬화시켜준다.

Jackson은 개발자가 직렬화/역직렬화 과정에 직접 참여하여 커스텀할 수 있는 기능을 제공해 주는데 직렬화시에는 `@JsonValue`를 역직렬화시에는 `@JsonCreator` 어노테이션을 사용하면 된다.
```kotlin
// 요청 객체
data class LoginRequest(
	val token: String,
	val type: LoginType,
)

// type enum
enum class LoginType(
	@JsonValue val value: String
) {
	KAKAO("kakao"),
	APPLE("apple");
	
	companion object {
		@JvmStatic
		@JsonCreator
		fun from(lower: String) =
			values().find { it.value == lower }
			?: throw IllegalArgumentException("Unknown LoginType: $lower")
	}
}
```

하지만 Legacy Enum의 수가 많다면 Enum마다 해당 코드를 반복 작성해야하기 때문에 퍽 난감하다. 때문에 더 나은 방법을 찾다 인터페이스를 활용하여 처리해보았다.

<br />  

### Legacy인 Enum들 인터페이스로 전환 로직 처리하기
Legacy Enum들은 (외부)소문자 <-> 대문자(앱 내) 간 변환이라는 공통점을 가진다. 동일한 행동을 취한다면 하나로 묶어주는게 간편하다. 인터페이스를 만들고 해당 인터페이스를 상속하는 Enum들이 취해야하는 공통 로직을 따로 작성하면 좋을 것 같다.

우선, `LegacyDtoEnum` 이라는 인터페이스를 선언한다. 코틀린에서는 인터페이스나 클래스에 구현부가 없다면 `{ }`를 생략할 수 있다.
```kotlin
interface LegacyDtoEnum
```

이후, Legacy Enum들에 해당 인터페이스를 상속받는다.
```kotlin
enum class LoginType: LegacyDtoEnum {  
    KAKAO,  
    APPLE;  
}
```

이제 이 인터페이스를 구현하는 enum값들이 역/직렬화시 어떤 로직이 실행될지를 직접 정의해야 한다.

<br />  

### 커스텀 Serializer, Deserializer 구현하기
Jackson에서는 역/직렬화를 처리하는 커스텀 클래스를 만들 수 있도록 지원해준다. `StdDeserializer`와 `StdSerializer`를 상속받아 각각 역직렬화, 직렬화하는 커스텀 로직을 작성하고 적용하면 된다.

적용 시에는 필드나 클래스에 `@JsonDeserialize(using = 적용할 커스텀 클래스타입)` 어노테이션을 붙여주면 된다. 필자는 커스텀 직렬화 로직을 적용할 Enum들이 `LegacyDtoEnum`을 상속 받도록 만들었기 때문에 해당 인터페이스에 작성하면 모든 Enum들에 적용된다. 깔끔하면서 선언적이라 가독성이 좋다! 만들면서 내심 기분이 좋았다.

역직렬화 시에는 소문자를 대문자 Enum 매핑하고 직렬화시에는 대문자 Enum을 소문자 문자열로 교체하는 로직을 작성하면 된다. 
```kotlin
class LegacyDtoEnumCombinedSerializer {  
	// 역직렬화 커스텀 로직
    class LegacyDtoEnumDeserializer<T>(vc: Class<*>?): StdDeserializer<T>(vc), ContextualDeserializer where T : LegacyDtoEnum, T: Enum<*> {  
        constructor() : this(null)  
  
        @Suppress("UNCHECKED_CAST")  
        override fun deserialize(p: JsonParser, ctxt: DeserializationContext): T {  
            val jsonNode: JsonNode = p.codec.readTree(p)  
            val value: String = jsonNode.asText().uppercase()  
            val enumValue = (_valueClass as Class<out Enum<*>>)  
                .enumConstants  
                .firstOrNull { it.name == value }  
                ?: throw ApiInvalidEnumException(p.currentName)  
  
            return enumValue as T  
        }  
  
        override fun createContextual(ctxt: DeserializationContext, property: BeanProperty): JsonDeserializer<*> {  
            return LegacyDtoEnumDeserializer(property.type.rawClass)  
        }  
    }  

	// 직렬화 커스텀
    class LegacyDtoEnumSerializer<T>(vc: Class<*>?): StdSerializer<T>(vc, true), ContextualSerializer where T : LegacyDtoEnum, T: Enum<*> {  
        constructor() : this(null)  
  
        override fun serialize(value: T, gen: JsonGenerator, provider: SerializerProvider) {  
            gen.writeString(value.name.lowercase())  
        }  
        
        override fun createContextual(prov: SerializerProvider, property: BeanProperty): JsonSerializer<*> {  
            return LegacyDtoEnumSerializer(property.type.rawClass)  
        }  
    }  
}
```

`LegacyDtoEnumDeserializer`와 `LegacyDtoEnumSerializer` 클래스가 각각 역직렬화, 직렬화 로직을 수행한다. 제네릭 T 타입이 `LegacyDtoEnum` 타입이자 Enum일 경우에만 사용가능하도록 제약을 걸었다.(`where T : LegacyDtoEnum, T: Enum<*>`)

한 가지 더 주목할 점은 `ContextualDeserializer` 인터페이스를 구현한다는 점이다. 특정한 타입이 아닌 추상화된 타입(인터페이스 & Enum)을 이용하기 때문에 `_valueClass`에 타입이 들어가 있지 않다. 때문에 특정 타입을 재지정해 주어야 하는데 `ContextualDeserializer`를 상속받아 createContextual 메서드를 작성해주면 된다.

> **JsonDeserializer와 StdDeserializer의 차이**
> - JsonDeserializer는 ObjectMapper를 이용해 Json로 역질렬화하는 class. Docs에서는 커스텀 Deserializer를 작성한다면 이 클래스가 아닌 StdDeserializer를 상속받아 만들 것을 권장한다.
> - StdDeserializer는 JsonDeserializer를 상속받았으며, 기본적으로 String으로부터 원시타입을 파싱할 수 있는 메서드들을 제공해준다.

<br />  

### Serializer 적용하기
마지막으로 작성한 클래스를 적용이 필요한 필드나 클래스에 추가하면 된다. 특정 필드에 적용 시 다음과 같이 적용하면 되지만 현재 상황에서 이와 같이 작성하는 것은 Legacy Enum을 사용하는 모든 필드에 작성해야하므로 반복적인 작업일 뿐이다.
```kotlin
data class LoginRequest(
	val token: String,
	@JsonDeserialize(using = LegacyDtoEnumCombinedSerializer.LegacyDtoEnumDeserializer::class)  
	@JsonSerialize(using = LegacyDtoEnumCombinedSerializer.LegacyDtoEnumSerializer::class)  
	val type: LoginType,
)
```

필자는 `LegacyDtoEnum`을 상속한 모든 곳에 적용할 것이기 때문에 `LegacyDtoEnum`에 적용하였다. 
```kotlin
@JsonDeserialize(using = LegacyDtoEnumCombinedSerializer.LegacyDtoEnumDeserializer::class)  
@JsonSerialize(using = LegacyDtoEnumCombinedSerializer.LegacyDtoEnumSerializer::class)  
interface LegacyDtoEnum
```

또 작성한 커스텀 클래스에 `@JsonComponent`를 적용하여 전역적으로 자동적용시킬 수도 있다. Jackson 모듈에 직접 등록되어 Jackson 동작시 해당 클래스로 처리할 수 있는 알맞은 타입들은 이를 이용해 역/직렬화 시켜준다.
```kotlin
@JsonComponent
class LegacyDtoEnumCombinedSerializer {
	// ...직렬화, 역직렬화 커스텀 클래스
}
```

`@JsonComponent`를 통해서 적용시키면 좋을까 싶기도 했지만 인터페이스에 직접 적용하는게 보다 선언적이고 가독성이 좋아 해당 방식을 선택했다. 후에 각 Enum에서 `LegacyDtoEnum` 인터페이스를 보는 것만으로 동작흐름을 파악할 수 있을 거라 여겼고 `@JsonComponent` 를 통한 등록은 그 연결고리를 찾기가 쉽지 않을 것 같았다.

위 내용들을 전부 적용한 후 Legacy Enum을 비교해보면 확연히 깔끔하고 가독성이 좋아졌다는게 느껴진다. 이런 Legacy Enum들이 N개, 수십개가 있다면.. 설명하지 않아도 그 효능을 충분히 알 수 있을 것이다.
```kotlin
// 적용 전
enum class LoginType(
	@JsonValue val value: String
) {
	KAKAO("kakao"),
	APPLE("apple");
	
	companion object {
		@JvmStatic
		@JsonCreator
		fun from(lower: String) =
			values().find { it.value == lower }
			?: throw IllegalArgumentException("Unknown LoginType: $lower")
	}
}

// 인터페이스로 분리 & 커스텀 로직 적용 후
enum class LoginType: LegacyDtoEnum {  
    KAKAO,  
    APPLE;  
}
```



<br />  

### 참고
- [StdDeserializer](https://fasterxml.github.io/jackson-databind/javadoc/2.7/com/fasterxml/jackson/databind/deser/std/StdDeserializer.html)
- [JsonDeserializer](https://fasterxml.github.io/jackson-databind/javadoc/2.7/com/fasterxml/jackson/databind/JsonDeserializer.html)
- [JsonComponent](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/jackson/JsonComponent.html)
- [Naver D2 - Jackson의 확장 구조를 파헤쳐 보자](https://d2.naver.com/helloworld/0473330)
