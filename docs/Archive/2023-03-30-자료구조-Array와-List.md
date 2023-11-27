---
layout: default
parent: Archive
title: "[자료구조] 연속되는 데이터 Array와 List (with Kotlin)"
categories: CS
tags:
  - data structure
---  

Array와 List는 다수의 **데이터를 저장하고 조작하는 기본적인 자료구조**이자 **다른 자료구조들을 구현**하기 위해 중요한 자료구조이다.  

## Array
Array 또는 배열이라고 불린다.  
> **특징**  
- 데이터들이 **메모리에 연속적으로 저장**됨
- **고정 길이**로 요소 삽입/삭제가 불가능
- **인덱스로 요소에 접근**할 수 있으며 이때 시간복잡도는 O(1)이다.  
- **따라서 요소의 개수가 고정되어있을때 사용하기 좋다**

코틀린에서 Array의 사용은 다음과 같다.  
```kotlin
fun main(args: Array<String>) {
    var arr = arrayOf(1,2,3,4,5,6,7)
    println(arr[0]) // 1
}
```

<br />  <br />  

## List
List는 인터페이스로 **크기를 동적으로 조절가능하며 순서가 존재하는 자료구조**{: .font-highlight}를 일컫는다. 이러한 List 인터페이스를 구현한 자료구조로 대표적으로 ArrayList, LinkedList가 있다.  


### ArrayList
List 인터페이스를 구현한 **배열 기반의 List**{: .font-highlight}, 배열을 보다 유연하게 만든 자료구조로 여러가지 장점을 가지고 있지만 단점 또한 명확한 자료구조이다.  

> **특징**  
- 배열이기 때문에 **데이터들이 메모리에 연속적으로 저장**됨  
- 요소 삽입/삭제
  - 마지막 인덱스에 요소 삽입,삭제의 경우 `O(1)`, 삽입시 내부 배열의 크기가 부족하면 더 큰 배열 생성 및 복사함으로 최악의 경우 `O(n)`
  - 그 외 `O(n)`
- 요소 접근 - 인덱스를 이용하여 `O(1)`
- **크기가 가변적이며 빠른 데이터접근, 요소 추가/삭제가 빈번한 경우 사용하기 좋음**


> **단점**  
- 삽입/삭제의 경우 성능이 좋지 않음
- **내부적으로 동기화를 보장하지 않음**
  - 멀티스레드 환경에서 안전하지 않음
  - 값 접근,수정만이라면 괜찮지만 시간이 걸리는 삽입,삭제의 경우 문제발생
  - 동기화 자료구조(Vector 등) or 명시적인 동기화 메커니즘을 사용해야함  

```kotlin
fun main(args: Array<String>) {
    var arrList = arrayListOf(1,2,3,4,5,6,7)

    arrList.add(8) // 가장 뒤에 추가 -> 보통 O(1)
    arrList.add(index = 0, element = 0) // index 0 에 값 0 추가 -> O(n)
    arrList.removeLast() // 가장 마지막 삭제 -> O(1)
    println(arrList[0]) // 0 -> O(1)
    println(arrList) // [0, 1, 2, 3, 4, 5, 6, 7]
}
```

<br />  

### LinkedList  
List 인터페이스를 구현한 자료구조로 연결리스트라고 부른다. LinkedList는 배열처럼 메모리에 연속적으로 담기는 것이 아닌 **요소가 체인처럼 이어져 있는 형식**{: .font-highlight}이다. 

> **특징**  
- 요소들이 메모리에 연속적으로 담겨있지 않음
- 요소 삽입/삭제
  - 첫요소 혹은 마지막요소 에서의 연산은 `O(1)`
  - 그 외에는 데이터를 찾는데 `O(n)`, 삭제연산에 `O(1)`이 걸린다.
  - 결국 ArrayList와 비슷한 시간복잡도를 가진다.
  - 하지만 전체를 새로 생성하는 것이 아닌 해당 요소만 생성/삭제하므로 효율이 더 좋다.
- 요소 접근
  - 인덱스 접근이 어려워 요소를 찾기 위새 순회해야한다. 따라서 `O(n)`의 시간이 걸린다.
  - 여러 언어에서 인덱스 접근을 허용하지만 연산은 `O(n)`이 걸린다.  
- 동기화를 보장하지 않는다.(멀리스레드 환경에서 안전하지 않음)
- 메모리에 연속적으로 담겨있지 않으므로 **정렬에 비효율적**  
- **따라서 삽입/삭제가 빈번한 경우에 사용하기 좋다.**  

```kotlin
import java.util.LinkedList  

fun main(args: Array<String>) {
    var linked = LinkedList<Int>(listOf(0,1,2,3,4))
    linked.add(5) // O(1)
    linked.add(6) // O(1)
    linked[4] // 순회해야하므로 O(n)
    linked.removeLast() // O(1)
    println(linked)
}
```  

<br />  

**💡 코틀린에서 List**  
코틀린에서 `listOf()` 를 이용해 List를 구현한 객체를 리턴해준다. 이때 내부적으로 ArrayList 또는 배열기반의 불변 리스트를 생성해서 리턴해준다.  
{: .notice--info}  




