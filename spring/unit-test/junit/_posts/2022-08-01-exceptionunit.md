---
title: Spring) Junit5 Exception Test
excerpt: junit5로 예외테스트하기
---

## JUnit5로 예외테스트하기

<br/> 

### 예외테스트 해야하는 이유

실패 테스트를 하지 않고 unit test를 구성할 경우 통합 테스트에서 시간을 많이 소요  
예외에 대한 단위 테스트는 애플리케이션에 대해 빠르게 견고하게 만들어줌!

<br/>

### RuntimeException 발생하는 메소드

```java
public class Hello {
    public static void func() {
        throw new RuntimeException("some exception message...");
    }
}
```  

<br/>
 
func()가 예외를 잘 발생하는지 테스트코드 작성해보기 <br/><br/>


### 1. assertThrows

```java
import static org.junit.jupiter.api.Assertions.assertThrows;
 
@Test
public void exceptionTest1() {
    Assertions.assertThrows(RuntimeException.class, () -> {
        Hello.func();
    });
}
```

- Assertions.assertThrows() : 첫번째 인자인 RuntimeException.class(부모 Exception도 가능)와 두번째 인자인 Hello.func()의 실행결과가 같은지 검사

<br/>

### 2. assertThrows 반환값 사용 - 예외메시지 테스트

```java
@Test
public void exceptionTest2() {
    Throwable exception = assertThrows(RuntimeException.class, () -> {
        Hello.func();
    });
    assertEquals("some exception message...", exception.getMessage());
}
```

assertThrows는 발생하는 예외를 Throwable(모든 예외클래스의 선조클래스) 타입으로 반환함  
발생한 예외객체의 메시지가 같은지 테스트  

<br/>

### 3. assertEquals (try ~ catch) - 예외메시지 테스트

```java
import static org.junit.jupiter.api.Assertions.assertThrows;
 
@Test
public void exceptionTest3() {
    try {
        Hello.func();
    } catch (RuntimeException e) {
        Assertions.assertEquals("some exception message...", e.getMessage());
    }
}
```

테스트코드가 무엇을 테스트하는지 명확하게 보이지 않을 수 있어 비추  

<br/>

### 4. assertj의 assertThatThrownBy     

```java
testImplementation 'org.assertj:assertj-core:3.21.0' 
```

테스트코드 가독성을 높여주기 위해 Junit5와 assertj를 같이 사용함

```java
import static org.assertj.core.api.Assertions.assertThatThrownBy;
 
@Test
public void exceptionTest4() {
    assertThatThrownBy(() -> Hello.func())
            .isInstanceOf(RuntimeException.class);
}
```

 isInstanceOf에 발생하는 예외 타입을 작성하여 테스트
 
 <br/> 

Reference  
https://www.baeldung.com/junit-assert-exception  
https://covenant.tistory.com/256  
