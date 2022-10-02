---
title: Spring) Transaction
excerpt: 스프링이 제공하는 트랜잭션
---

## Transaction
데이터베이스의 상태를 변화시키기 위해 수행(CRUD)하는 작업의 단위  
데이터베이스의 데이터를 작업하다가 문제가 생겼을 경우,  
이전 상태로 롤백하기 위해 사용
- transaction 처리 명령어
  -  commit : 모든 부분작업이 정상적으로 완료 -> 변경사항 한꺼번에 DB에 반영하여 마무리 됨
  - rollback : 부분작업 실패 -> 작업을 취소하고 실행 전으로 돌림 

<br/>

## Spring이 제공하는 Transaction

### AOP를 이용한 트랜잭션 분리  

- business logic code 와 transaction code 가 같이 있음  

```java
public void addUsers(List<User> userList) {
	TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
	
	try {
		for (User user: userList) {
			if(isEmailNotDuplicated(user.getEmail())){
				userRepository.save(user);
			}
		}

		this.transactionManager.commit(status);
	} catch (Exception e) {
		this.transactionManager.rollback(status);
		throw e
	}
}
```

<br/>

- @Transactional 이용해서 코드 분리 (선언적 트랜잭션처리) -> 핵심 비지니스 로직만 남김 
  - 트랜잭션코드(부가기능코드)를 클래스 밖으로 빼내서 `별도의 모듈로` 만듬(AOP)   
    - @Transactional은 Spring의 Aop이용  
    - AOP는 Dynamic Proxy 이용  
    - Dynamic Proxy는 인터페이스 기반으로 동작   
    	=> 인터페이스가 있어야만 동작함  
  - `DB와 관련된` 트랜잭션이 필요한 `Service` class or method에 @Transactional 붙여주기    
  - 트랜잭션 기능이 포함된 프록시 객체가 생성되어 자동으로 commit 혹은 rollback을 진행
    - @Transactional이 붙은 메서드는 메서드가 포함하고 있는 작업 중에 `하나라도 실패할 경우` 전체 작업을 취소



```java
@Service
@RequiredArgsConstructor
@Transactional
public class UserService {

    private final UserRepository userRepository;

    public void addUsers(List<User> userList) {
        for (User user : userList) {
            if (isEmailNotDuplicated(user.getEmail())) {
                userRepository.save(user);
            }
        }
    }
}
```

<br/>
 
## @Transactional  
@Transactional 붙은 클래스, 메서드 앞뒤에 프록시 생성 <br/><br/>
모든 예외상황에서 롤백 되는게 아님  
기본적으로 error, unchecked exception 만 롤백  
= checked exception에 대해서 롤백 안되게 설계되어 있음 <br/><br/> 
=> 모든 예외에 대해서 전부 트랜잭션 롤백하고싶으면 rollbackFor={Exception.class}  

<br/>

- 자바에서 예외 :  error, exception  
  - Error :  시스템레벨에서 발생하는 오류 -> 미리 예측불가 개발 시 신경안써도됨
  - Exception : 로직 상에서 발생하는 오류 -> 예측가능 상황에 맞게 처리  
  
- Checked exception, Unchecked exception - runtimeException 상속여부에 따라  
  - checked exception : 컴파일부터 안됨, 예외처리코드 필수  
  (IOException, SQLException)
  - unchecked exception : 런타임중 예외발생, 예외처리코드 필수아님, runtimeException 상속  
  (NullPointerException, IndexOutOfBoundException)

<br/>

## @Transactional 옵션
### 1. `isolation` : 격리레벨 

```java
@Transactional(isolation=Isolation.DEFAULT)
public void addUser(User user) throws Exception {
```

- DEFAULT : 기본
- READ_UNCOMMITED (level 0) :  커밋되지 않는 데이터에 대한 읽기를 허용
- READ_COMMITED (level 1) : 커밋된 데이터에 대해 읽기 허용
- REPEATEABLE_READ (level 2) : 동일 필드에 대해 다중 접근 시 모두 동일한 결과를 보장
- SERIALIZABLE (level 3) : 가장 높은 격리, 성능 저하의 우려가 있음

<br/>

### 2. `propagation` : 전파속성  

```java
@Transactional(propagation=Propagation.REQUIRED)
public void addUser(User user) throws Exception {
```  

<br/>

![제목 없음](https://user-images.githubusercontent.com/103614357/193441154-11ac6005-3301-45a8-9e72-5a060410a06f.png)       

- REQUIRED : default
  - 진행중인 트랜잭션이 있다면 해당 트랜잭션 속성 따름
  - 진행중이 아니라면 새로운 트랜잭션 생성   

<br/>

![제목 없음](https://user-images.githubusercontent.com/103614357/193441266-b477721e-22c9-4f53-b943-3bd67f733393.png)  

- REQUIRES_NEW : 항상 새로운 트랜잭션 생성
  - 진행중인 트랜잭션이 있다면 잠깐 보류
  - 해당 트랜잭션 작업 먼저 진행

<br/>

![제목 없음](https://user-images.githubusercontent.com/103614357/193441308-4c556277-38ae-4141-847c-870672e53424.png)  

- MANDATORY
  - 진행중인 트랜잭션이 있어야만 작업수행
  - 없으면 Exception 발생

<br/>

![제목 없음](https://user-images.githubusercontent.com/103614357/193441361-0bd6c774-45d2-45bc-b2de-63d1d8dec406.png)  

- NEVER
  - 진행중인 트랜잭션 있으면 Exception
  - 진행중이지 않을 때만 작업수행

<br/>

![제목 없음](https://user-images.githubusercontent.com/103614357/193441338-72f5cce9-b87f-4de4-be47-19ade2cf807e.png)  

- NESTED
  - 진행중인 트랜잭션 있으면 중첩된 트랜잭션 실행
  - 없으면 REQUIRED와 동일하게 실행

- SUPPORT
  - 진행중인 트랜잭션이 있다면 해당 트랜잭션 속성 따름
  - 없다면 트랜잭션 없이 작업수행

- NOT_SUPPORT
  - 진행중인 트랜잭션 있다면 보류, 트랜잭션 없이 작업수행

<br/>

### 3. `noRollbackFor` : 예외무시  

특정예외 발생 시 rollback처리 하지 않음

```java
@Transactional(noRollbackFor=Exception.class)
public void addUser(User user) throws Exception {
```

<br/>

### 4. `rollbackFor` : 예외추가  

특정예외 발생 시 강제로 rollback  

```java
@Transactional(rollbackFor=Exception.class)
public void addUser(User user) throws Exception {
```

<br/>

### 5. `timeout` : 시간지정  

지정한 시간 내에 해당 메소드수행이 완료되지 않을경우 rollback  
default : -1 (no timeout)  

```java
@Transactional(timeout=10)
public void addUser(User user) throws Exception {
```

<br/>

### 6. `readOnly` : 읽기전용

default : false  
true 시 insert, update, delete 실행 시 예외발생 = select만 가능    

```java
@Transactional(readonly = true)
public void addUser(User user) throws Exception {
```

<br/>

## Test method + @Transactional
메서드 종료될 때 자동으로 롤백됨  

- Auto Increment 옵션으로 인해 증가한 id는 rollback되어도 감소되지 않음
  - Auto Increment 옵션은 트랜잭션 범위 밖에서 동작

<br/><br/>

Reference  
https://mangkyu.tistory.com/154  
https://velog.io/@kdhyo/JavaTransactional-Annotation-%EC%95%8C%EA%B3%A0-%EC%93%B0%EC%9E%90-26her30h  
토비의스프링(저자:이일민)    
https://deveric.tistory.com/86  
<br/>
