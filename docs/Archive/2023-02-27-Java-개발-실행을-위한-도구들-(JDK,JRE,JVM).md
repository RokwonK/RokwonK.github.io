---
layout: default
parent: Archive
title: "Java 개발, 실행을 위한 도구들(JDK, JRE, JVM)"
categories: Java
tags:
  - JVM
---

Java 개발을 시작할때 필연적으로 JDK, JRE, JVM과 같은 개념들과 부딪히게 된다. 이 포스팅에서는 이 개념들의 대해 정리해보았다.  

<br />  

### 개발을 위한 도구모음 - JDK 
JDK(Java Development Kit)은 **개발자들이 Java로 개발하기 위해 사용하는 도구모음이다.** 따라서 개발시 필요한 라이브러리와 javac과 같은 개발도구, 자바 실행을 위한 도구들(JRE) 또한 포함하고 있다. 따라서 개발자는 JDK만 설치해도 개발 및 실행에 필요한 모든 도구들을 함께 설치할 수 있다. 

![jdk](https://user-images.githubusercontent.com/52196792/221554941-37cd38f8-9284-4053-8ac3-4822b7a0c6eb.png){: .align-center style="width: 60%;"}  

JDK 파일의 bin 디렉터리를 열어보면 여러가지 개발도구가 보인다. 컴파일러, 실행도구, 디버거 등이 이 디렉터리에 존재한다.  

**javac** : 자바 컴파일러로 자바코드를 바이트 코드로 컴파일  
**java** : 자바 인터프리터로 바이트 코드를 해석하고 실행  
**jar** : 자바 클래스 파일을 압축한 아카이브 파일 생성, 관리해줌  
**jmod** : 자바의 모듈 파일을 만들거나 내용 출력
**jdb** : 디버거  
**jlink** : 응용프로그램에 맞춘 맞춤형 JRE 생성  
**javap** : 컴파일된 클래스 파일을 원래의 소스로 변환  
{: .notice}  

자바코드를 작성하고 JDK에 포함된 `javac`을 이용하여 자바코드를 바이트코드 형태인 `.class`로 변환(컴파일 과정)하였다면 해당 파일을 이용해 실행할 일만 남았다. 이때 실행환경을 구성하는 패키지가 JRE이다.  

<br />  

### 실행을 위한 환경 - JRE  
JRE(Java Runtime Environment)는 **자바 프로그램 실행을 위해 환경(라이브러리 + JVM)이다.** JDK를 설치하면 함께 설치된다.(JDK11 버전 부터는 JRE만 따로 설치 불가능)  
실제로 프로그램을 실행시키는 JVM과 함께 실행을 위한 자바 클래스 라이브러리, JVM과 소통을 하는 class Loader가 존재한다. class Loader는 `.class` 파일들(개발한 코드, core library 등)을 가져와 JVM과 소통한다.

![jre](https://user-images.githubusercontent.com/52196792/221561784-fc9eb011-7495-4d74-b650-a7271ab7a538.png){: .align-center style="width: 60%;"}  


<br />  

### 실행 - JVM  
JVM(Java Virtual Machine)은 **자바를 실행하는 프로그램**이다. `.class`은 JVM만이 해석할 수 있으며 JVM은 해석 후 OS와 소통하며 프로그램을 실행한다. 자바의 큰 장점인 WORA(Write Once, Run Anywhere)은 JVM을 통한 실행 덕분이다. OS환경에 맞는 JVM만 있다면 한 번 작성한 코드를 어느 환경에서든 사용가능하다.  

JVM은 WORA뿐만 아니라 **자바 프로그램의 메모리를 효율적으로 관리하고 최적화**해 준다. 이를 위해 JVM 내부에서는 메모리를 구조화하고 Garbage Collector를 이용하여 메모리 관리, 실행 및 최적화를 위한 Execution Engine을 포함하고 있다.  

![jvm](https://user-images.githubusercontent.com/52196792/221575423-e733dacc-78d8-440b-9dd1-9ea65a7151ca.png){: .align-center style="width: 60%;"}  

<br />  

### Next Step  
자바와 객체지향을 따로 떼어놓고 논할 수 없다. 자바가 어떤 방식으로 객체지향을 구현했는지 알아보자. 또한 JVM 내부 기능들(GC, Execution Engine 등)의 동작원리를 조금 더 깊게 공부해보자.  

<br />

**참고** 
- [Java 프로그램의 실행원리(feat. JVM)](https://ikjo.tistory.com/7)
- [A2 JVM, JRE, JDK의 차이](https://wikidocs.net/257)
- [[JAVA] ☕ JDK / JRE / JVM 개념 & 구성 원리 💯 완벽 총정리](https://inpa.tistory.com/entry/JAVA-%E2%98%95-JDK-JRE-JVM-%EA%B0%9C%EB%85%90-%EA%B5%AC%EC%84%B1-%EC%9B%90%EB%A6%AC-%F0%9F%92%AF-%EC%99%84%EB%B2%BD-%EC%B4%9D%EC%A0%95%EB%A6%AC)
- [[조금 더 깊은 Java] JVM의 Runtime Data Area, Execution Engine, Garbage Collection 에 대해서](https://wonit.tistory.com/591)


