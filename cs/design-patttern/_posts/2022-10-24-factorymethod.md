---
title: CS) Factory Method Pattern
excerpt: 디자인패턴 - 생성 패턴 - 팩토리메서드
---

## Factory Pattern  

![제목 없음](https://user-images.githubusercontent.com/103614357/197549177-73f74484-620b-46ff-9e0d-b59b8385b399.png)  

<br/>
  
## Factory Method Pattern

객체를 만들어내는 공장(Factory 객체)을 만드는 패턴     
구체적인 `객체의 생성 과정`을 `Factory로 모듈화`하여 구체적인 부분이 아닌 추상적인 부분에 의존할 수 있도록  `캡슐화`하는 패턴   
= 객체의 생성코드를 별도의 `클래스/메서드로 분리`함으로써 객체 생성의 변화에 대비하는 데 유용   
(기존 코드의 변경 없이 확장하기 위한)

<br/>

- 객체를 생성하기 위해 인터페이스 정의   
- 객체를 생성할 때 어떤 클래스의 인스턴스를 만들지는 `서브 클래스에서 결정` 
   
⇒ 클래스간의 결합도 낮아짐

<br/>

### 사용  

- 생성할 객체 타입을 예측할 수 없을 때
- 생성할 객체를 기술하는 책임을 서브클래스에게 정의하고자 할 때
- 객체 생성의 책임을 서브클래스에 위임시키고 서브클래스에 대한 정보를 은닉하고자 할 때

<br/>

### Factory Method Pattern의 적용 방법

1. 객체 생성을 전담하는 별도의 Factory 클래스 이용
    - 스트래티지 패턴과 싱글턴 패턴을 이용
2. 상속 이용 :  하위 클래스에서 적합한 클래스의 객체를 생성
    - 스트래티지 패턴, 싱글턴 패턴과 템플릿 메서드 패턴 이용

<br/>

### Factory Method의 장단점

- 장점
    - 기존코드(인스턴스를 만드는 과정) 수정 불필요
    새로운 인스턴스를 다른방법으로 생성하도록 확장
    - 수정에 닫혀있고 확장에는 열려있는 OCP 원칙
- 단점
    - 간단한 기능을 사용할 때보다 많은 클래스를 정의해야 하기 때문에 클래스가 많아짐
    (바뀔 때마다 새로운 서브클래스를 생성해야하니까)

<br/>

### 실제 적용예시

```jsx
Robot(abstract class)
	┗ SuperRobot
	┗ PowerRobot

RobotFactory(abstract class)
	┗ SuperRobotFactory
	┗ ModifiedSuperRobotFactory
```

- `package pattern.factory;`
  - 두종류의 로봇(SuperRobot, PowerRobot)
  - 두종류의 로봇공장(SuperRobotFactory, ModifiedSuperRobotFactory)

<br/>

- Robot

```java
public abstract class Robot {
	public abstract String getName();
}
```

```java
public class SuperRobot extends Robot {
	@Override
	public String getName() {
		return "SuperRobot";
	}
}
```

```java
public class PowerRobot extends Robot {
	@Override
	public String getName() {
		return "PowerRobot";
	}
}
```

<br/>

- RobotFactory

```java
public abstract class RobotFactory {
	abstract Robot createRobot(String name);
}
```

```java
public class SuperRobotFactory extends RobotFactory {
	@Override
	Robot createRobot(String name) {
		switch( name ){
			case "super": return new SuperRobot();
			case "power": return new PowerRobot();
		}
		return null;
	}
}
```

```java
public class ModifiedSuperRobotFactory extends RobotFactory {
	@Override
	Robot createRobot(String name) {
		try {
			Class<?> cls = Class.forName(name);
			Object obj = cls.newInstance();
			return (Robot)obj;
		} catch (Exception e) {
			return null;
		}
	}
}
```

`ModifiedSuperRobotFactory` : 로봇 클래스의 이름을 String 인자로 받아서 직접 인스턴스를 만들어냄

<br/>
  
- Main Program

```java
public class FactoryMain {
	public static void main(String[] args) {

		RobotFactory rf = new SuperRobotFactory();
		Robot r = rf.createRobot("super");
		Robot r2 = rf.createRobot("power");

		System.out.println(r.getName());
		System.out.println(r2.getName());

		RobotFactory mrf = new ModifiedSuperRobotFactory();
		Robot r3 =  mrf.createRobot("pattern.factory.SuperRobot");
		Robot r4 =  mrf.createRobot("pattern.factory.PowerRobot");

		System.out.println(r3.getName());
		System.out.println(r4.getName());
	}
}
```

```java
SuperRobot
PowerRobot
SuperRobot
PowerRobot
```

- 객체 생성을 RobotFactory 클래스에 위임 → Main Program에서 new 키워드가 없음
  - Main Program은 어떤 객체가 생성 되었는지 신경 쓰지 않고 단지 반환된 객체를 사용만 하면 됨
  - 새로운 로봇이 추가 되고 새로운 팩토리가 추가 된다 하더라도 Main Program에서 변경할 코드는 최소화

<br/><br/>

### Spring의 **BeanFactory**

xml 설정과 java 설정으로 읽어오는 방식

```java
BeanFactory factory = new XmlBeanFactory(
            new InputStreamResource(
            new FileInputStream("oraclejavacommunity.xml")));
OracleJavaComm ojc = (OracleJavaComm)factory.getBean("oracleJavaBean");
```

<br/><br/>

Reference     
https://ko.wikipedia.org/wiki/%ED%8C%A9%ED%86%A0%EB%A6%AC_%EB%A9%94%EC%84%9C%EB%93%9C_%ED%8C%A8%ED%84%B4   
https://dev-youngjun.tistory.com/195   
https://jdm.kr/blog/180   
<br/>
