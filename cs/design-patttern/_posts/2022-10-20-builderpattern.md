---
title: CS) Builder Pattern 
excerpt: 디자인패턴 - 생성 패턴 - 빌더패턴
---

## 생성 패턴(Creational Pattern)      
객체 인스턴스를 생성하는 패턴  
클라이언트와 그 클라이언트가 생성해야 하는 객체 인스턴스 사이의 연결을 끊어주는 패턴  

<br/>

## Builder
인스턴스를 생성할 때 생성자(Constructor)만을 통해서 생성하는데는 어려움이 있어서 고안된 패턴

- 생성과 표기를 분리해서 복잡한 객체 생성
- `복잡한 객체를 생성하는 클래스`와 `표현하는 클래스`를 분리
-> `동일한 절차`에서도 `서로 다른 표현 결과`를 만들 수 있는 디자인 패턴
- 생성자에 들어갈 매개 변수가 많든 적든 차례차례 매개 변수를 받아들이고, 모든 매개 변수를 받은 뒤에 이 변수들을 통합해서 한번에 사용

- ex) Person class

```java
class Person {
    String a;
    String b;
    String c;
    String d;
}
```

<br/>  

### 점층적 생성자패턴   
Overloading을 통해 내가 사용할 인자를 받는 생성자를 모두 만드는 것
- 코드가 길어지고, 지져분해지는 단점
- 순서를 바꿔서 넣을수도 있고, 생성자 인자가 많을 때 각 인자가 어떤 의미인지 어려움

```java
class Person {
    String a;
    String b;
    String c;
    String d;

    public Person(String a) {
        this.a = a;
    }

    public Person(String a, String b, String c, String d) {
        this.a = a;
        this.b = b;
        this.c = c;
        this.d = d;
    }
    
    public Person(String a, String c) {
        this.a = a;
        this.c = c;
    }
} 
```

<br/> 

### 자바 빈 패턴
생성자패턴의 단점 보완, setter 메소드를 이용한 패턴
- 가독성은 생성자를 사용한 것보다 좋아지고 객체를 생성하기도 쉬워짐
- but setter 이용하여 객체 생성
    - 함수 호출 1회로 객체를 생성할 수 없음(여러번 호출해야함)
    - 객체 일관성(consistency)가 일시적으로 깨질 수 있음 (Getter, Setter 존재)
    - 변경 불가 클래스를 만들 수 없음 (객체가 변할 여지가 존재한다 => 쓰레드간의 공유 가능한 상태가 존재하기 때문)

```java
public static Person create() {
    Person person = new Person();
    person.setA("a");
    person.setB("b");
    person.setC("c"); 
    person.setD("d");
    return person;
}
```
  
<br/>

### 빌더 패턴
각 인자를 받고나서 반환으로 빌더 객체를 반환하여 체이닝(.a().b().c().d())형식으로 사용
- 필요한 객체를 직접생성하는 대신 먼저 필수 인자들을 생성자에 전부 전달하여 빌더 객체를 만든다.
- 그리고 선택인자는 가독성이 좋은 코드로 인자를 넘길 수 있다.
- 객체 일관성을 깨지 않을 수 있다.

```java
public class PersonBuilder {
    private String a;
    private String b;
    private String c;
    private String d;

    public PersonBuilder setA(String a) {
        this.userIdx= userIdx;
        return this;
    }

    public PersonBuilder setB(String b) {
        this.name = name;
        return this;
    }

    public PersonBuilder setC(String c) {
        this.part = part;
        return this;
    }

    public PersonBuilder setD(String d) {
        this.age = age;
        return this;
    }

    public PersonBuilder build(){
        PersonBuilder personBuilder = new PersonBuilder(a, b, c, d);
        return personBuilder;
    }
}
```

⇒ 위의 빌더 클래스를 사용하면 `PersonBuilder`  객체를 가능한 헷갈리지 않고 생성가능    

```java
public class BuilderPattern {
    public static void main(String[] args) {
        // 빌더 객체
        PersonBuilder personBuilder = new PersonBuilder ();
        // 빌더 객체에 원하는 데이터를 입력 (순서는 노상관)
        Person person = personBuilder 
                .setA("a")
                .setB("b")
                .setC("c")
                .setD("d")
                .setA("A")
                .build();
    }
}
```

- 다시 같은 메소드를 호출하면 나중에 호출한 값 들어감
- 마지막에 .build() 호출해서 최종적인 결과물을 만들어 반환

<br/>

- 빌더 클래스를 꼭 객체를 만들어낼 클래스와 분리할 필요는 없음      
  객체를 만들어낼 클래스 내부에 빌더 클래스 포함가능    

```java
public class Main {
    public static void main(String[] args) {
        Person person = Person.builder()
                .a("a")
                .b("b")
                .c("c")
                .d("d")
                .build();
        System.out.println(
                person.getA() +
                person.getB() +
                person.getC() + person.getD());
    }
}

class Person {
    private String a;
    private String b;
    private String c;
    private String d;
  
    public static PersonBuilder builder() {
        return new PersonBuilder();
    }

    static class PersonBuilder {
        String a;
        String b;
        String c;
        String d;

        PersonBuilder a(String a) {
            this.a = a;
            return this;
        }

        PersonBuilder b(String b) {
            this.b = b;
            return this;
        }

        PersonBuilder c(String c) {
            this.c = c;
            return this;
        }

        PersonBuilder d(String d) {
            this.d = d;
            return this;
        }
         
        Person build() {
            Person person = new Person();
            person.a = this.a;
            person.b = this.b;
            person.c = this.c;
            person.d = this.d;
            return person;
        }
    }

```
<br/>

### 빌더패턴 단점
1. 객체를 생성하려면 우선 빌더객체를 생성해야 함
2. 다른 패턴들보다 많은 코드를 요구하기 때문에 인자가 충분히 많은 상황에서 이용할 필요

<br/><br/>  

Reference   
https://velog.io/@poiuyy0420/%EB%94%94%EC%9E%90%EC%9D%B8-%ED%8C%A8%ED%84%B4-%EA%B0%9C%EB%85%90%EA%B3%BC-%EC%A2%85%EB%A5%98   
https://www.hanbit.co.kr/channel/category/category_view.html?cms_code=CMS8616098823    
https://devlog-wjdrbs96.tistory.com/207  
https://jdm.kr/blog/217  
https://devlog-wjdrbs96.tistory.com/207  
https://ktko.tistory.com/338    
https://cjw-awdsd.tistory.com/43
<br/>
