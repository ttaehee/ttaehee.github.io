---
title: Java Basic
excerpt: ""
---

## Java의 특징
- `Simple` : C++보다 간단 but 느림
- `Object Oriented Programming` : 객체지향 프로그래밍 언어
- `Garbage Collection` : GC가 메모리를 자동관리
- `Platform Independent` : JVM(Java Virtual Machine)에서 동작하기 때문에 운영체제에 상관없이 작동 가능
- `Multi Thread` : 자바는 thread 생성 및 제어와 관련된 library API를 제공하기 때문에 구현이 쉬움  
   여러 thread에 대한 scheduling을 Java Interpreter가 담당
- `Dynamic Loading` : 실행 시 모든 객체가 생성되지 않고 필요한 시점에 class를 동적로딩해서 객체생성  
   일부 class 변경되어도 전체 application을 다시 compile 하지 않아도됨 - 유지보수가 빠르고 간편
- `Open Source Library` : 오픈소스 언어이기 때문에 다양한 라이브러리 활용가능 <br/><br/>

## Java Code 실행과정
1. `.java` : Java로 작성된 소스파일 생성
2. `.class` : Compiler를 통해 byte code로 compile
3. `Java program` : Interpreter를 통해 JVM에서 실행 <br/>

## JVM (Java Virtual Machine) 자바 가상 머신
- Java에서 각각의 OS (Application=JVM's platform)에 맞는 JVM(Java's platform) 제공  
 : 운영체제에 독립적, JDK만 있으면 실행가능  
JDK(Java Development Kit) : 자바개발도구, JVM 포함  
(platform이란 상대적, OS's platform은 hardware pc)  
- 일반 application code는 os만 거치고 하드웨어로 전달  
but Java Application은 JVM을 한번 더 거침, 실행 시 해석(interpret)되기 때문에 속도가 느리다는 단점  
그러나 요즘엔 바이트코드 (compile 된 자바 코드)를 하드웨어의 기계어로 바로 변환해 주는 JIT 컴파일러와 향상된 최적환 기술이 적용되어서 속도 격차를 많이 줄였다 <br/><br/>

## Java's platform
1. J2SE : 일반 데스크탑pc
2. J2EE : 서버용pc
3. J2ME : 소형제품 <br/><br/>

## API (Application Programming Interface)
: 응용프로그램에서 사용할 수 있도록 운영체제나 프로그래밍 언어가 제공하는 기능을 제어할 수 있게 만든 인터페이스   
Java에 기본으로 내장되어 있는 API + 개발자가 추가적으로 만든 API <br/><br/>

## Java 실행환경 / 개발환경
- 실행환경 : JDK (JRE + tool)
- 개발환경 : JAVA_HOME, path, classpath 
1. `JAVA_HOME` : JDK(JRE)있는 폴더를 자바홈으로 환경변수 잡아주기 - JDK 없는 폴더에서도 사용가능  
&nbsp;(ex) cmd창에서 javac 명령어가 제대로 실행되려면 잡아줘야함)  
3. `path` : 실행파일을 찾아오는 경로 - 실행프로그램의 위치만을 나타냄
4. `classpath` : 클래스를 찾아오는 경로 - 실행프로그램에서 사용하는 라이브러리의 위치를 나타냄 (default : `.;` 현재폴더) <br/><br/>

## Source code & Class code
Java는 따로 관리함 but Eclipse는 자동으로 컴파일해서 bin에 컴파일된 byte code 저장해줌, src에 source code <br/><br/>

## Access Modifier
- public : 모든 접근허용
- protected : 동일패키지, 하위패키지 접근허용
- default : 동일패키지 접근허용
- private : 현재 객체내에서만 접근허용

