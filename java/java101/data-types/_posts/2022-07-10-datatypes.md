---
title: Data Types
excerpt: ''
---

# Data Types
- `Primitive Data Type` : (정수, 실수, 문자, 논리 리터럴 등의) 실제 데이터값을 저장하는 타입
- `Reference Data Type` : 메모리 번지 값을 통해 객체를 참조하는 타입

Java에서 실제객체는 heap 영역에 저장되며, 참조타입변수는 stack 영역에 실제 객체들의 주소를 저장  
객체를 사용할 때마다 참조변수에 저장된 객체의 주소를 불러와 사용함  
기억공간에 주소가 들어가고, 주소를 찾아 들어가야지만 실질적인 데이터가 있음 <br/><br/>

## Primitive Data Type (8개)
>논리형   
- `boolean`  
  1. true/false  
  2. 1bit
>문자형
- `char`
  1. 'a' 문자하나  
  2. 2bit   
  3. 유니코드 정수형태로 저장 => char ch='A', ch변수에는 'A'의 정수 값인 65가 들어감  
  ** 유니코드 : 사용중인 운영체제, 프로그램, 언어에 관계없이 문자마다 고유한 코드 값을 제공하는 개념의 코드
  4. single quotation(')사용
  5. Java에서 유일하게 제공되는 unsigned 형태 (음수 없이 0부터 시작하여 양수값만 가짐)  
>정수형
- `byte` : 8bit (-128 ~ 127)
- `short` : 16bit = 2byte
- `int` : **정수형의 기본** / 32bit
- `long` : 64 bit = 8byte
>실수형
- `float` : 32bit
- `double` : **실수형의 기본** / 64bit <br/><br/>

## Reference Type (Primitive type 제외한 타입들)
- Primitive type과 차이
1. null 담을 수 있음
2. Generic type 에서 사용가능 <br/><br/>

**String** : immutable (불변, 정적)  
**StringBuffer** : mutable (가변, 동적), 내부적으로 buffer 라고하는 독립적인 공간을 가짐  
인스턴스의 값을 변경할 수 있음
**Integer**  
**null 값** : 참조형에서 주소가 없는 것을 명시적으로 알려주는 값 (참조하는게 없다) <br/><br/>

## Call by value (값에 의한 호출)
함수 호출 시 전달되는 변수의 값을 복사하여 함수의 인자로 전달  
=> 함수 안에서 인자의 값이 변경되어도, 외부의 변수 변경 안됨(local value)  
- 장점 : 복사하여 처리하기 때문에 안전, 원래의 값이 보존됨
- 단점 : 복사를 하기 때문에 메모리가 사용량이 늘어남 <br/><br/>

## Call by reference (참조에 의한 호출)
함수 호출 시 인자로 전달되는 변수의 레퍼런스를 전달 (해당 변수를 가리킴)  
=> 함수 안에서 인자의 값이 변경되면, 인자로 전달된 객체도 값이 바뀜  
**해당 객체의 주소값을 직접 넘기는 게 아닌 객체를 보는 또 다른 주소값을 만들어서 넘김**
(Googling : swap 함수 검색) <br/><br/>

## Casting
1. Down casting : 좁은 범위로의 형변환 - 데이터 손실 있어 직접(explicit) 선언해줘야함  
```java
double a = 1.24;
int b = (int) a; // 1.24 -> 1 (소수점 이하 버려짐)  

int c = 65;
char d =(char) c ; // char 2byte < int 4byte
```
2. Up casting : 넓은 범위로의 형변환 - 자동(implicit)으로 변환됨 (Promotion) <br/>
3. 
