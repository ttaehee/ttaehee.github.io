---
title: JAVA) String, StringBuffer, StringBuilder
excerpt: 차이점
---

## String vs StringBuffer, StringBuilder

### 1) immutable(String)  
인스턴스가 한번 생성되면 값 **변경 불가**  
= 할당된 공간이 변하지 않음

```
String str = "hello";
str = str + " world";  // hello world
```

=> str("hello" 값을 가지고 있는 참조변수)이 가리키는 곳에 저장된 "hello"에 "world"를 더해 "hello world"로 변경된 것이 아님!  
- str(기존에 "hello" 값이 들어가있던 참조변수)이 "hello world"라는 값을 가지는 `새로운 메모리영역을 가리키게` 변경  
- 처음 선언 "hello"로 값이 할당되어 있던 메모리 영역은 `Garbage`로 남아있다가 GC(garbage collection)에 의해 사라짐 

![제목 없음](https://user-images.githubusercontent.com/103614357/184467727-a7fa83ad-e147-4044-bf90-1d344df23321.png)  

- String 클래스는 불변하기 때문에 문자열을 `수정`하는 시점에 `새로운 String 인스턴스`가 생성
  -  `+` 연산자를 많이 사용하면 할수록 String 객체의 수가 늘어나기 때문에 프로그램 성능이 느려짐

![제목 없음](https://user-images.githubusercontent.com/103614357/184468575-cfe10091-90d4-4834-bd2e-1911104d076a.png)

<br/>

### 2) mutable(StringBuffer, StringBuilder)
인스턴스 값을 자유롭게 **변경 가능**     
= 객체의 공간이 부족해지면 `버퍼의 크기`를 유연하게 늘려줌  
=> `동일 객체내에서` 문자열 변경 가능 : 내부 버퍼에 문자열 저장해두고 추가,수정,삭제 <br/><br/>


## StringBuffer vs StringBuilder
동기화 유무!  
(String 도 불변성을 가지기 때문에 멀티쓰레드 환경에서의 안정성 가짐)  

### StringBuffer : thread-safe
각 메소드별로 synchronized keyword가 존재  
-> 멀티스레드 상태에서 동기화 지원

### StringBuilder
단일 스레드 환경에서만 사용하도록 설계  
속도는 더 빠름 <br/><br/>


## 사용
- `String` : 문자열 연산이 적고, 멀티쓰레드 환경 / 단순하게읽는 조회연산
- `StringBuffer` : 문자열 연산이 많고, 멀티쓰레드 환경
  - web or socket환경 등의 비동기로 동작하는 경우
- `StringBuilder` : 문자열 연산이 많고, 단일쓰레드 or 동기화 고려 안해도 되는경우

<br/><br/>

Reference   
https://ifuwanna.tistory.com/221  
https://coding-factory.tistory.com/546  
https://novemberde.github.io/post/2017/04/15/String_0/  
