---
title: JAVA) Exception
excerpt: 예외처리
---

# Exception
: 프로그램을 사용하는 사용자의 잘못된 조작 or 개발자의 잘못된 코딩에 따라 발생되는 오류  
 자바는 예외를 클래스를 통해 관리함
- `FileNotFoundException` : 존재하지 않는 파일 열려고 할 때
- `ArithmeticException` : 0으로 숫자를 나눌 때
- `ArrayIndexOutOfBoundsException` : 배열의 크기보다 클 때
- `NullPointerException` : null 값으로 설정된 객체 사용하려 할 때
- `NumberFormatException` : 문자열을 int형으로 변환할 때 변환할 수 없는 값이 포함되었을 때
- etc <br/><br/>  

예외는 에러와 다름, 에러는 메모리부족 or 컴퓨터 고장 등 심각한 오류

## java.lang.Exception Class
모든 예외 클래스는 java.lang.Exception Class의 상속을 받음 = exception의 최상위  

### checked exception
- 컴파일과정에서 예외검사 -> 예외처리하지 않으면 `컴파일 오류` 
- Exception class 상속
- RuntimeException class는 상속받지 않음

### runtime exception = unchecked exception
- 컴파일과정에서 예외검사 안함 -> 예외처리 안해도 컴파일 가능
- 개발자의 경험에 따라서 예외발생 가능코드에 대해서 예외처리 알아서 해줘야함
- Exception class, `RuntimeException class` 상속 <br/><br/>


## 예외처리
: 예외가 발생 가능한 코드를 식별하여 예외가 발생되면 실행의 흐름을 바꾸고 프로그램의 종료를 막는 것  
&nbsp; 좋은 프로그램일수록 오류에 대해서 적절하게 반응해야 함

### 1. try - catch - finally
```
try{
    예외발생 가능 로직;
}catch(Exception e){
    예외 처리할 로직;
}finally{ //생략가능
    예외발생여부와 상관없이 반드시 실행할 로직
}
```
finally : return 있어도 실행 ( System.exit(0)은 프로세스종료라 finally도 실행안함)  

- 다중 catch문
```
} catch (NumberFormatException ne) {
    System.out.println("문자는 숫자로 변환할수 없습니다.");
} catch (ArrayIndexOutOfBoundsException ae) {
    System.out.println("배열방을 초과하였습니다.");
}
```
여러개의 예외처리 가능  
동시에 두개의 예외를 처리하지는 않음 - 먼저 발생된 예외만 처리하고 빠져나감  
=> 위에 더 상위 Exception 두면 다 잡음, 자세히 알고싶으면 하위 먼저 쓰기! <br/><br/>

### 2. throws
예외 던지기, 떠넘기기
- 예외를 일으키는 메서드를 `호출한 메서드로` 예외를 넘기고 책임을 전가
- 예외를 넘겨받은 메소드는 try - catch 문을 이용하여 예외를 처리  
- main에서도 던지면 main 호출한 JVM으로 -> JVM 에서 다시 class로 (재귀)  
==> 여러 발생 가능한 예외를 특정 메서드에서 한 번에 처리해 주어 관리를 용이  

```
try{
    메소드(); // 메소드호출 - throws에 의해 호출한 곳에서 try-catch로 예외처리
}catch(Exception e){
    예외처리코드
}
```

## 인위적 예외
- Exception 을 확장해서 만듬
- 쓸 때는 내가 직접 예외발생될만한 곳에 예외발생시켜주기  
ex) `throw new` myException <br/>
