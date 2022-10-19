---
title: CS) Design Pattern  
excerpt: SOLID원칙 & 디자인패턴과 종류  
---

## 객체지향 프로그래밍의 5가지 설계원칙, SOLID  
### 1.SRP(Single Responsibility Principle) : 단일 책임 원칙  
- 하나의 모듈은 `한 가지 책임`을 가져야 한다는 것(해당 모듈이 여러 대상 또는 액터들에 대해 책임을 가져서는 안됨) = 모듈이 변경되는 이유가 한가지여야 함을 의미    
   
### 2. OCP(Open-Closed Priciple) : 개방 폐쇄 원칙  
- 확장에 대해 열려있고 수정에 대해서는 닫혀있어야
  - 확장에 대해 열려 있다: 요구사항이 변경될 때 `새로운 동작을 추가`하여 애플리케이션의 기능확장 가능    
  - 수정에 대해 닫혀 있다: 기존의 `코드를 수정하지 않고` 애플리케이션의 동작을 추가하거나 변경가능    

### 3. LSP(Liskov Substitution Priciple) : 리스코프 치환 원칙  
 - 하위 타입은 상위 타입을 대체할 수 있어야  
 - 추상객체로 사용되는 부분에 구상객체가 들어가도 상관없어야

### 4. ISP(Interface Segregation Principle) : 인터페이스 분리 원칙  
- 목적과 관심이 각기 다른 클라이언트가 있다면 인터페이스를 통해 적절하게 분리  

### 5. DIP(Dependency Inversion Principle) : 의존 역전 원칙   
- 고수준 모듈은 저수준 모듈의 구현에 의존해서는 안됨, 저수준 모듈이 고수준 모듈에서 정의한 추상 타입에 의존해야 한다는 것  
  - 고수준 모듈: 변경이 없는 추상화된 클래스(또는 인터페이스)
  - 저수준 모듈: 변하기 쉬운 구체 클래스  
- 의존 역전 원칙 - 개방 폐쇄 원칙 밀접한 관련    
  의존 역전 원칙이 위배되면 개방 폐쇄 원칙 역시 위배될 가능성 높음  

<br/>

=> SOLID 핵심은 결국 `추상화`      
구체 클래스에 의존하지 않고 추상 클래스(또는 인터페이스)에 의존함으로써 우리는 유연하고 확장가능한 애플리케이션을 만들 수 있는 것!  

<br/>  

=> 원칙에 따라 설계를 해봄 => 공통점이 있네 => `디자인패턴`    

<br/><br/>

## Design Pattern
[refactoring.guru사이트](https://refactoring.guru/design-patterns/catalog)  

- 개발을 할 때 비슷한 유형의 문제에 대한 재사용가능한 해결방안  
- 특정 구현방식이 아니며 문제를 해결하는데에 쓰이는 템플릿 혹은 아이디어  
- 패턴은 문제 혹은 해결책에서의 유사점  

<br/> 

### Design Pattern 구조  
디자인 패턴은 context, problem, solution의 구조  
- `context` : 문제가 발생하는 상황 기술 = 패턴이 적용될 수 있는 상황을 나타냄    
  (경우에 따라서는 패턴이 유용하지 못한 상황을 나타내기도)  
- `problem` : 패턴이 적용되어 해결해야 할 여러 디자인 이슈 기술, 여러 제약사항과 영향력도 고려해야함  
- `solution` : 문제를 해결하도록 설계를 구성하는 요소, 그 요소들 사이의 관계, 책임, 협력관계를 기술  

<br/>

### GoF 디자인패턴 종류 요약  

- GoF(Gang of Fout) 디자인패턴  
  - 에리히 감마(Erich Gamma), 리차드 헬름(Richard Helm), 랄프 존슨(Ralph Johnson), 존 블리시디스(John Vissides)
소프트웨어 개발 영역에서 디자인 패턴을 구체화하고 체계화한 사람들
  - 23가지의 디자인 패턴을 정리

![zzz](https://user-images.githubusercontent.com/103614357/196756012-bcd6fe72-d981-4c52-bbe9-98ce334920f7.png)   

**생성**  
- `Singleton(싱글턴)` : 유일한 하나의 인스턴스 보장  
- `Builder(빌더)` : 생산 단계를 캡슐화 -> 구축 공정을 동일하게 이용  
- `Prototype(프로토타입)` : 복사하여 새 개체 생성  
- `Factory Method(팩토리메서드)` : 객체를 생성하기 위한 인터페이스 정의 -> 어떤 클래스가 인스턴스화 될 것인지는 서브 클래스가 결정  
- `Abstract Factoroy(추상팩토리)` : 생성군들을 하나에 모아놓고 팩토리 중에서 선택

**구조**  
- `Bridge(브릿지)` : 추상-구현을 분리하여 결합도를 낮춤  
- `Decorator(데커레이터)` : 소스를 변경하지 않고 기능을 확장
- `Facade(퍼사드)` : 하나의 인터페이스를 통해 느슨한 결합 제공
- `Flyweight(플라이웨이트)` : 대량의 작은 객체들을 공유
- `Proxy(프록시)` : 대리인이 대신 그 일을 처리
- `Composite(컴퍼지트)` : 개별 객체와 복합 객체를 클라이언트에서 동일하게 사용
- `Adapter(어댑터)` : 인터페이스로 인해 함께 사용하지 못하는 클래스를 함께 사용

**행위**  
- `Interpreter(인터프리터)` : 언어 규칙 클래스 이용
- `Template Method(템플릿메서드)` : 알고리즘 골격의 구조 정의
- `Chain of Responsibility(책임연쇄)` : 객체들끼리 연결 고리를 만들어 내부적으로 전달
- `Command(커맨드)` : 요청 자체를 캡슐화 -> 파라미터로 넘기는 패턴
- `Iterator(이터레이터)` : 내부 표현은 보여주지 않고 순회  
- `Mediator(미디에이터)` : 객체 간 상호작용을 캡슐화  
- `Memento(메멘토)` : 상태 값을 미리 저장해 두었다가 복구  
- `Observer(옵서버)` : 상태가 변할 때 의존자들에게 알리고, 자동 업데이트  
- `State(스테이트)` : 객체 내부 상태에 따라서 행위를 변경  
- `Strategy(스트래티지)` : 다양한 알고리즘 캡슐화 -> 알고리즘 대체가 가능
- `Visitor(비지터)` : 오퍼레이션을 별도의 클래스에 새롭게 정의  

<br/><br/>  

Reference    
https://mangkyu.tistory.com/194  
https://gmlwjd9405.github.io/2018/07/06/design-pattern.html   
https://velog.io/@sicksong/CS%EB%94%94%EC%9E%90%EC%9D%B8-%ED%8C%A8%ED%84%B4  
https://effortguy.tistory.com/m/182   
https://velog.io/@poiuyy0420/%EB%94%94%EC%9E%90%EC%9D%B8-%ED%8C%A8%ED%84%B4-%EA%B0%9C%EB%85%90%EA%B3%BC-%EC%A2%85%EB%A5%98   
https://www.hanbit.co.kr/channel/category/category_view.html?cms_code=CMS8616098823  
<br/>
