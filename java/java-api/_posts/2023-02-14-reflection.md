---
title: JAVA) java.lang.Reflection
excerpt: reflection api
---

<br/>

데브코스에서 한가지 주제로 글을 쓸 기회가 있었고, 나는 주제를 reflection으로 골랐다    
관련하여 작성했던 글!   

<br/>

❓ Spring에서 어떻게 실행시점에 Bean을 주입할 수 있는걸까?

❓JPA Entity에 기본생성자가 꼭 필요한 이유는 무엇일까?

이 글은 위의 두가지 궁금증에서 시작되었습니다.

<br/>   

리플렉션 덕분이지! 라고만 끄덕이고 넘어갔던 한달 전의 저의 모습을 반성하며, 
이번 기회에 리플렉션에 대해 알아보았습니다 📚✏ 

<br/><br/>

# Reflection

![asdf](https://user-images.githubusercontent.com/103614357/218317488-1d584150-cb52-4434-b3a6-59145f7cfd05.png)   

Reflection을 번역하면 거울에 비친 상(모습)을 뜻합니다.  

<br/><br/>

## Java Reflection

Java에서의 Reflection은 실체가 아닌 반사된 이미지를 통해서 어떤 행위를 하는것을 의미합니다.   
여기에서 실체는 Class, 거울은 JVM 메모리 영역이라고 볼 수 있습니다.   
말을 정리해보자면, 리플렉션이란 실제 Class를 사용하는 것이 아닌 JVM 메모리 영역에 올라와있는 데이터를 사용하는 기술입니다.   

<br/>

![제목 없음](https://user-images.githubusercontent.com/103614357/218630289-a507ac2c-be09-4ed7-ba34-0a2a1c0f6a50.png)   

자바에서는 JVM이 실행되면 

1. 소스코드가 컴파일러를 거쳐 바이트 코드로 변환되고, 
2. 이 바이트 코드를 클래스로드가 읽어 JVM 내 메모리 영역에 저장합니다. 
3. 리플렉션은 이 저장된 클래스의 정보를 꺼내와 필요한 정보들(생성자, 필드, 메서드)을 가져와 사용합니다. 

<br/> 

따라서 구체적인 클래스 타입을 알지 못해도 클래스 이름만 알고 있다면 언제든 메모리 영역에서 그 클래스의 정보를 찾아 접근할 수 있습니다.    
코드를 작성할 시점에는 어떤 타입의 클래스를 사용할지 모르지만, 런타임 시점에 지금 실행되고 있는 클래스를 가져와서 실행해야 하는 경우가 있을 때 리플렉션을 사용하면 좋겠죠.     
 
<br/>

감은 잡히나, 완벽하게 와닿지 않으니 예시와 함께 리플렉션의 기능들을 하나씩 살펴보겠습니다.   

<br/><br/>

## 리플렉션의 기능

```java
public class 백둥이 {

	private String name;
	public String team;

	public 백둥이() {
		this.name = "백둥이 이름";
		this.team = "백둥이 팀이름";
	}

	public 백둥이(String name, String team) {
		this.name = name;
		this.team = team;
	}

	private void study(String subject) {
		System.out.println(name + "는 " + subject + "을 공부한다.");
	}
}
```

기능을 살펴보기 위해 만든 예시 클래스입니다.   
필드인 name과, study 메서드의 접근제어자가 private인 점을 한번 언급하고 가겠습니다.   

<br/><br/>

### 1. 생성자를 통해 객체를 생성하는 기능

```java
//백둥이 클래스를 Class 타입 객체로 가져오기
Class<?> exClass = Class.forName("org.example.백둥이");

//백둥이 클래스의 생성자를 Constructor 타입의 객체로 가져오기
Constructor<?> exConstructor1 = exClass.getDeclaredConstructor();
Constructor<?> exConstructor2 = exClass.getDeclaredConstructor(String.class, String.class);

//생성자를 통해 객체 생성하기
Object 백둥이1 = exConstructor1.newInstance();
Object 백둥이2 = exConstructor2.newInstance("김태희", "훈");
```

![KakaoTalk_20230212_172645265](https://user-images.githubusercontent.com/103614357/218317842-178b47a6-2fb8-4288-a146-01e0932b3351.png)   

첫번째는 생성자를 통해 객체를 생성하는 기능입니다.  
먼저 java.lang.reflect 패키지의 Class 클래스를 사용하여 클래스의 정보를 가져옵니다.   
Class 클래스를 불러오는 방법은 여러방법이 있는데, 저는 그 중 forName을 통해 가져왔습니다.  

<br/>

Class 클래스로 선택한 클래스의 생성자 정보(백둥이 클래스의 생성자)를 가져옵니다.   
`getDeclaredConstructor` 메서드를 사용하면 생성자 정보를 Constructor 타입의 객체로 받습니다.  
인자를 넣으면 그 타입과 일치하는 생성자를 찾습니다.   

<br/>

결과를 보니 백둥이1과 2 인스턴스가 잘 만들어졌네요.

<br/><br/>

참고) 클래스의 이름을 문자열로 사용하는 이유

 :: 해당 클래스가 컴파일할 때 없기 때문

```java
Class c = Class.forName("클래스이름");
```

컴파일 타임에 클래스의 이름 자체를 사용할 수 없기 때문에 문자열 형태로 클래스의 이름을 사용합니다.      
`Class.forName("클래스이름")`이 호출되는 순간에 클래스를 로딩하겠다는 의미로, 동적 바인딩기법으로 클래스를 로딩하게 됩니다.    

<br/><br/>  

### 2. 필드 정보 조회 기능

```java
Field[] fields = exClass.getDeclaredFields();

for(Field field : fields) {
	field.setAccessible(true);  //접근제어자가 private인 경우에도 접근가능하도록
	System.out.println(field);
	System.out.println("value : " + field.get(백둥이2));
	System.out.println("==============================");
}
```

![KakaoTalk_20230212_172553319](https://user-images.githubusercontent.com/103614357/218317790-33f3722f-473f-48ff-8580-ede9d8d5d5af.png)  

두번째는 필드의 정보를 조회하는 기능입니다.    
백둥이 클래스에 정의된 필드들에 대한 정보를 Field라는 타입의 객체로 받은 후   
위와 같이 필드의 접근제어자와 타입, 이름, 값을 알 수 있습니다.   

<br/><br/>

그런데 잠깐, 백둥이 클래스에서 name의 접근제어자가 private이었던것 기억하시나요?   
private인 필드의 정보까지 가져왔네요    
위의 코드에서 보는것처럼 `setAccessible` 메서드를 사용하면 필드의 접근제어자가 private인 경우에도 접근가능해집니다.   
신기하면서도 오개월내내 객체지향과 함께한 백둥이로서, 무언가 불편함이 느껴지네요 저만 그런가요..! 🧐    

<br/><br/>

### 3. 필드의 값 변경 기능

```java
Field field = exClass.getDeclaredField("team");

field.setAccessible(true);
System.out.println("기존 : " + field.get(백둥이2));

field.set(백둥이2, "앨런");
System.out.println("변경 : " + field.get(백둥이2));
```

![KakaoTalk_20230212_172555227](https://user-images.githubusercontent.com/103614357/218317826-05694970-e396-40a3-a52c-a6268822cdc9.png)   

세번째는 필드의 값을 변경해주는 기능입니다.    
위의 코드처럼 변경하기 원하는 인스턴스와 값을 넣어주면 필드의 값도 변경 가능합니다.   
심지어 필드의 접근제어자가 private이어도 말이죠   

<br/>

이러한 방법으로 리플렉션이 값을 넣거나 변경해주었던 것이군요!   
(어디까지나 예시입니다 저는 영원한 훈훈한 훈팀이죠 🥰 )   
 
<br/><br/>

### 4. 메서드 정보 조회 기능

```java
Method method = exClass.getDeclaredMethod("study", String.class);
method.setAccessible(true);
System.out.println(method);
method.invoke(백둥이2, "Reflection");
```

![KakaoTalk_20230212_172556808](https://user-images.githubusercontent.com/103614357/218317885-16318f41-03fa-4c05-9bd7-b5c4da610a39.png)     

네번째는 메서드 정보 조회 기능입니다.    
필드와 마찬가지로 `setAccessible` 메서드를 사용해 private메서드도 접근가능합니다.   
따라서 private 메서드의 호출도 가능합니다.   

<br/><br/>

리플렉션의 기능은 클래스의 이름만으로 해당 클래스의 정보를 가져올 수 있다 한마디로 정리할 수 있겠네요

<br/><br/>

## 중간 정리   

지금까지 알아본것을 정리해보고 가자면,    
리플렉션은 JVM 내 메모리 영역에 저장된 클래스의 정보를 꺼내서 사용하기 때문에     
클래스의 이름만 가지고도 생성자, 필드, 메서드 등 해당 클래스에 대한 거의 모든 정보를 가져올 수 있다! 겠군요 정말 신기합니다!      

<br/>

그렇지만 신기한 기능이라고 무조건 사용하면 안되겠죠     
무엇이든 단점까지 고려 해본 후에 판단해야 하니까요!    

<br/><br/>

## 리플렉션의 단점

**1. 성능 오버헤드, 느리다**    
- 컴파일 타임이 아닌 런타임에 동적으로 타입을 분석하고 정보를 가져오므로, JVM을 최적화 할 수 없기 때문에 일반 메서드 호출보다 훨씬 느립니다.
- 이는 reflection의 invoke 메서드 실행 시간을 측정한 테스트들의 결과로, reflection만을 테스트 하는것이 아니라 동적으로 class를 load하고, heap에 인스턴스를 저장하는 절차까지 포함하기 때문일 수 있습니다. 초기 호출 이후로 캐싱을 통해 최적화가 된다면 다른 결과가 있을 수도 있을 것 같습니다. 추후 테스트를 해봐야겠네요.

<br/>

**2. 캡슐화를 깨트린다**    
- 직접 접근할 수 없는 private 필드, 메서드에도 접근이 가능해 내부가 노출됩니다.

<br/>

**3. 컴파일 타임에 type, exception 등의 검증을 할 수 없다**    
- 컴파일 타임 타입검사의 이점을 누릴 수 없겠네요.

<br/>

**4. 런타임에 인스턴스가 선택되기 때문에, 해당 로직의 구체적인 동작 흐름을 파악하는 것이 어렵다**    

<br/>

**5. 코드가 복잡해진다**    
- 일반적인 객체 생성, 메서드 호출 코드와 비교하면 복잡합니다.

<br/>

단점까지 알아본 후에 낸 결론은, 리플렉션은 사용하지 않을 수 있다면 사용하지 않는게 좋겠다입니다.      
우리가 코드를 작성하면서 구체적인 클래스를 모를일이 거의 없기도 하고,     
위의 단점들까지 고려했을 때 장점보다는 단점이 더 크다고 생각이 드네요.    

<br/><br/>

그러면 신기하지만 그만큼 단점이 명확한 리플렉션은 어디서 사용되는걸까요?   
바로 저의 궁금증이 시작된 곳이죠, Spring과 JPA!   
Framework와 Library에서 사용됩니다.    

<br/><br/> 

## 리플렉션이 사용되는 곳

Framework나 Library에서는 사용자가 어떤 클래스를 만들지 예측할 수 없기 때문에 동적으로 해결하기 위해 사용합니다.    
리플렉션의 장점이 필요하고 잘 반영된 부분이라고 생각합니다.   

- Spring
- Annotation
- JPA
- Jackson
- Mockito
- JUnit
- IntelliJ의 자동완성 기능

<br/>

리플렉션을 사용해 테스트 프레임워크를 직접 만들 수도 있습니다.
관심있을 분들을 위해 첨부합니다.

[xUnit 테스팅 프레임워크를 TDD로 만들어보자](https://www.youtube.com/live/tdKFZcZSJmg?feature=share)

<br/><br/>

## 정리

이렇게 리플렉션에 대해 알아보았습니다.    
첫번째 궁금증이었던 Spring에서 어떻게 실행시점에 Bean을 주입할 수 있는걸까?에 대해서는 지금까지 정리한 내용만으로도 해소가 되었어요    

- Bean은 application이 실행된 후, 런타임에(= 객체가 호출될 때) 동적으로 객체의 인스턴스를 생성하는데, 이때 Spring Container의 BeanFactory에서 리플렉션을 사용하기 때문에 가능했던거죠.

<br/>

두번째 궁금증이었던 JPA Entity에 기본생성자가 꼭 필요한 이유는 무엇일까?에 대해서는 일부만 해소가 되었습니다.     
왜 꼭 기본생성자여야 하고, 기본생성자만으로 어떻게 알맞은 필드에 값을 넣어주는걸까요?   

- 여러 블로그에서 Reflection API로 가져올 수 없는 정보 중 하나가생성자의 인자 정보이기 때문에 기본생성자가 필요하다고들 했습니다.   
- 그렇지만 아래와 같이 생성자의 파라미터 정보를 가져오는 것 또한 가능했습니다. 찾아보니 Java 8부터 추가되었다고 합니다.      
  그렇다면 이 이유는 아닐듯합니다.      

  ```java
  System.out.println("파라미터 개수 : " + exConstructor2.getParameterCount());
  Parameter[] parameters = exConstructor2.getParameters();

  for (Parameter parameter : parameters) {
    System.out.println("파라미터 이름 : " + parameter.getName());
    System.out.println("파라미터 타입 : " + parameter.getType());
  }
  ```

  ![KakaoTalk_20230212_202118187](https://user-images.githubusercontent.com/103614357/218318989-91c68f74-f268-40b3-a647-2d050f21bc0d.png)   

- 찾다보니 기본 생성자로 객체를 생성하고 필드의 이름에 맞추어 값을 넣어주는 것이 가장 간단한 방법이기 때문이라는 의견을 보았습니다.      
  설득이 된 이유는, 실제로 여러 생성자가 있을 때 Framework나 Library가 어떤 생성자를 사용해야할지 고르기 어렵기도 할 것이고, parameter의 type이 같은 경우 필드와 이름이 다르다면 값을 알맞게 넣어주기가 힘들 것이기 때문입니다.    
  기본 생성자를 사용한다면, 위와 같은 난감한 경우들을 고려할 필요가 없어 간단하겠죠.   
- 아래 예시처럼 파라미터의 타입이 겹치고, 필드와 이름까지 다르면 어디에 어떤값을 넣어줘야 할지 Framework나 Library가 판단할 수 있을까요?
    
    ```java
    public 백둥이(String personName, String teamName) {
    		this.name = personName;
    		this.team = teamName;
    	}
    ```
    

- 그렇지만 이 또한 어디까지나 하나의 의견일 뿐이고, 정확한 이유는 찾지 못했습니다.   
  아시는 분이 있다면 공유해주시면 정말정말 감사하겠습니다 🥰   

<br/><br/>

Reference       
[Guide to Java Reflection](https://www.baeldung.com/java-reflection)     
[Reflection API 간단히 알아보자](https://tecoble.techcourse.co.kr/post/2020-07-16-reflection-api/)          
[java.lang.Reflection](https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html)           
[Using Java Reflection](https://www.oracle.com/technical-resources/articles/java/javareflection.html)         
[기본 생성자가 필요한 '진짜' 이유 (리플렉션 오해 바로 잡기!!!)](https://colour-my-memories-blue.tistory.com/16)        
[파랑, 아키의 리플렉션](https://www.youtube.com/watch?v=67YdHbPZJn4&feature=youtu.be)         

<br/>
