---
title: "Body, Query, Entity에 포함된 Legacy Enum 효율적으로 관리하기(2) Query편"
excerpt: "RequestParam, ModelAttribute 내 Legacy Enum들을 깔끔하고 효율적으로 관리해보자"

categories:
  - Project

permalink: /project/legacy-enum-management-2/
---

<br />  


[이전 포스팅](https://rokwonk.github.io/project/legacy-enum-management-1/)에서는 요청, 응답 Body에 포함된 Legacy Enum을 다뤘던 방법을 작성하였다.

아쉽게도, '개발 한 스푼' 서비스에서는 쿼리 파라미터에도 Legacy Enum(소문자 값 사용)들이 사용되고 있다. 사실 소문자 값을 사용하는게 Legacy라 부르는게 맞는가 싶기도 하지만 어찌되었든 어플리케이션에서는 대문자형 Enum 값에 바인딩시켜 사용하고 싶다.

쿼리 문자열은 Jackson을 통해서 역/직렬화가 되는것이 아니기 때문에 이전 포스팅에서 만든 Serializer를 사용할 수 없다. 이번 포스팅에서는 RequestParam 또는 ModelAttribute에 포함된 Legacy Enum들을 객체지향적으로 다룬 방법을 공유해본다.

> 요청/응답 Body, Query, Entity 부분을 각각 포스팅을 나누어 작성하였습니다.
> - [요청/응답 Body에 포함된 Legacy Enum 관리하기](https://rokwonk.github.io/project/legacy-enum-management-1/)
> - **(Now) Query에 포함된 Legacy Enum 관리하기**
> - [Entity에 포함된 Legacy Enum 관리하기]()

<br />

### Query가 객체 혹은 Param 필드에 바인딩 되는 방법
Controller에서 `@RequestParam`로 특정 필드에 `@ModelAttribute`를 통해 객체로 쿼리 정보들을 받을 수 있다. 이때 우리는 각 필드에 String, Int와 같이 타입을 지정하는데 재밌게도 알아서 쿼리값들이 해당 타입으로 변환되어 주입된다.

이러한 동작이 가능한 이유는 WebMvc에서 Converter를 이용해 바인딩(DataBinder 이용) 작업이 이루어지기 때문이다. 내부적으로 가지고 있는 **Converter 중 필드에 할당된 타입으로 전환시킬 수 있는 가장 구체적인 Converter를 골라 사용**한다. 대부분의 `StringTo{기본타입}Converter`들은 기본적으로 등록되어 있다.

만약 다른 값으로 커스텀해서 바인딩 시키고자 한다면 직접 Converter를 작성해서 등록해줘야한다.

<br />

### String을 Enum으로 바꿔주는 기본 Converter
Enum으로 Convert 해주는 기본적인 Converter는 `StringToEnumConverterFactory`이다. 일반적인 Converter와 ConverterFactory의 차이는 ConverterFactory는 추상적인 클래스타입을 받아 구체적인 클래스타입으로 체크하고 Converter를 만들어 준다. 즉, 여러가지 타입간 변환을 처리할 수 있다.

```java
// 이 파일은 java 임
final class StringToEnumConverterFactory implements ConverterFactory<String, Enum> {  
  
   @Override  
   public <T extends Enum> Converter<String, T> getConverter(Class<T> targetType) {  
      return new StringToEnum(ConversionUtils.getEnumType(targetType));  
   }  

   private static class StringToEnum<T extends Enum> implements Converter<String, T> {  
      private final Class<T> enumType;  
      
      StringToEnum(Class<T> enumType) {  
         this.enumType = enumType;  
      }  
      
      @Override  
      @Nullable
      public T convert(String source) {  
         if (source.isEmpty()) {  
            return null;  
         }  
         return (T) Enum.valueOf(this.enumType, source.trim());  
      }  
   }  
  
}
```

필자 또한 다양한 Enum 클래스에 대한 처리가 필요하기에 Converter가 아닌 ConverterFactory를 이용하여 커스텀 Converter를 구현했다.

<br />

### Legacy용 ConverterFactory 작성하기
[Legacy Enum 관리하기 Body편](https://rokwonk.github.io/project/legacy-enum-management-1/)에서 작성한 것과 매커니즘은 거의 똑같다. 쿼리는 요청시에만 사용되므로 소문자 문자열을 대문자 Enum 값으로 바인딩하는 로직만 작성하면 된다.

마찬가지로 `LegacyDtoEnum` **인터페이스 타입을 사용했다**. 모든 Enum 타입에 적용하기에는 이후 Legacy가 아닌 Enum을 만들경우 매우 위험하다. 그렇다고 구체적인 하나의 Enum 타입에만 적용하기에는 같은 로직의 Converter를 여러개 작성해야한다. ConverterFactory와 인터페이스를 이용하면 효율적으로 필요한 부분에만 적용시킬 수 있다.

Converter가 처리할 수 있는 타입을 Enum 클래스 타입이자 LegacyDtoEnum을 상속받은 경우에만 가능하도록 만들었다. Factory에서 Converter를 만들때 구체적인 타입을 넘겨 enum 값들을 불러올 수 있도록 하고, Converter 로직은 소문자 문자열을 대문자로 바꿔 Enum 값과 매칭되는 값으로 선택되도록 만들었다.

```kotlin
class StringToLegacyDtoEnumConverterFactory<F>: ConverterFactory<String, F> where F: Enum<*>, F: LegacyDtoEnum {  
    override fun <T : F> getConverter(targetType: Class<T>): Converter<String, T> {  
        return StringToEnumConverter(targetType)  
    }  
  
    @Suppress("UNCHECKED_CAST")  
    private class StringToEnumConverter<T : Enum<*>?>(private val enumType: Class<T>) : Converter<String, T> {  
        override fun convert(source: String): T {  
            val value = source.uppercase()  
            val enumValue = (enumType as Class<out Enum<*>>)  
                .enumConstants  
                .firstOrNull { it.name == value }  
                ?: throw ApiInvalidEnumException()  
            return enumValue as T  
        }  
    }  
}
```

<br />

### Converter 적용하기
WebMvc 설정에 Converter를 전역적으로 등록할 수 있다. `WebMvcConfigurer` 인터페이스를 사용하면 스프링 부트가 기존에 제공하는 기능에 더해 추가적인 커스텀 기능을 추가할 수 있다.

아래와 같이 ConverterFactory를 등록하면 원하던대로 잘 동작한다.
```kotlin
@Configuration  
class WebMvcConfig: WebMvcConfigurer {  
	// ...
  
    override fun addFormatters(registry: FormatterRegistry) {  
        registry.addConverterFactory(StringToLegacyDtoEnumConverterFactory())  
    }  
}
```

이제 요청/응답 Body, Query에서의 Legacy Enum을 처리할 수 있게 되었다. Enum에 인터페이스를 더해 가독성 좋고 확장성 좋게 Legacy를 다룰 수 있었다. 이제 DB에서 사용중인 Legacy Enum을 다루는 방법만 남았다.

<br />
