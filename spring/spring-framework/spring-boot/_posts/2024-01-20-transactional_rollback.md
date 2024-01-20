---   
title: Spring) 선언적 트랜잭션 내에서 예외처리를 했는데도 롤백된 이유
excerpt: Transaction silently rolled back because it has been marked as rollback-only     
---   

<br/>   

- 넘 오랜만의 기록이다 24년이 되어버렸어    
  시간 여유 되면 23년 회고도 써봐야쥐   
  - 요즘 회사 코드 통합작업이랑 리팩토링하면서 기존코드 파악하고 머리쓰느라 정신없당   
    서비스가 커지면서 슬슬 순환참조 문제도 생기고,,   
    순환참조 없애볼겸 리팩토링하면서 아키텍쳐 바꿔보자구 스터디하면서 소소하게 ddd 적용해보는중인데 생각보다 쉽지 않다 서로의 의견도 다 달라서 논의하다보면 가끔 회의실이 핫하다   
  - 아무튼 그러던 와중에 맞이한 에러 `Transaction silently rolled back because it has been marked as rollback-only`   
    분명 예외처리를 해주었는데도 롤백되길래 관련사항들을 파악해보았다   

<br/>

## 이슈 내용

<img width="934" alt="스크린샷 2024-01-20 오후 1 11 00" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/0fd3b530-b535-4f19-85e3-b43691d16a49">

에러를 살펴보니 채팅 서버에서 응답을 못받은 경우 던지는 예외와 관련이 있었다     
그런데 이 예외는 메서드를 호출한 메서드에서 잡아서(try-catch) 기본값을 대신 응답하도록 되어있었다   
Spring의 선언적 트랜잭션(`@Transactional`) 안에서 예외를 잡았기 때문에 당연히 롤백 없이 커밋될거라 예상했는데 에러가 발생했다      
회사 코드를 쓸 수 없으니 비슷한 상황을 만들어보자면   

<br/>

1. A 클래스 내에 있는, 메서드를 호출한 메서드

```java
@Transactional
public Test test(String a) {
    ...

    try {
        b.inner();

        return new Test(a);
    } catch (TestException e) {
        // 예외 발생한 경우 기본값 응답하도록
        return new Test();
    }
}
```

2. B 클래스 내에 있는, 예외를 던진 메서드

```java
@Transactional
public Inner inner() {
    ...

    throw new TestException();
}
```

<br/>

## 기대했던 동작 & 이와 다른 부분

1. B.java의 inner()에서 `throw new TestException();`를 통해 예외를 던짐

2. 해당 메서드를 호출한 A.java의 test()에서 try catch를 통해 예외 처리함 
    
  - 💡 예상했던 방향     
    현재 선언적 트랜잭션(@Transactional)에서 별도로 전파 속성을 주지 않았기 때문에 default인 `PROPAGATION_REQUIRED`니까   
    → `B.java의 inner()`는 호출한 쪽인 `A.java의 test()`의 이미 만들어진 트랜잭션에 참여하게 될 것이다   
    → 따라서 호출한 쪽에서 try catch를 통해 예외처리를 할 수 있을 것이다     
 
    ⇒ 디버깅을 통해 원하는대로 catch되어 요기까지 오는걸 확인함   

3.그러나 끝까지 처리되지 않고(=return 하지 않고) `UnexpectedRollbackException` 발생

![image](https://github.com/ttaehee/ttaehee.github.io/assets/103614357/c6c5e0fb-bfaf-4ee2-99b5-740723f696b5)

<br/>

## Why?  

(도대체 내가 어디에서 rollback-only를 마킹했다는거야..!)   

디버깅으로 따라가다 만난 코드를 보면

![image](https://github.com/ttaehee/ttaehee.github.io/assets/103614357/5edfbe4b-47d2-4574-8613-2c86b40880ac)

<br/>

- `txInfo.transactionAttribute.rollbackOn(ex)` 따라가보면   

```java
@Override
public boolean rollbackOn(Throwable ex) {
    return (ex instanceof RuntimeException || ex instanceof Error);
}
```

<br/>
 
- `txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());` 따라가다 보면

```java
private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
      // Participating in larger transaction
      if (status.hasTransaction()) {
        if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
          if (status.isDebug()) {
            logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
          }
          doSetRollbackOnly(status);  //요기서 롤백이 마킹되었음
        }
    
        ...
        
        // Raise UnexpectedRollbackException if we had a global rollback-only marker
        if (unexpectedRollback) {
          throw new UnexpectedRollbackException(
              //익숙한 예외메시지~,,
              "Transaction rolled back because it has been marked as rollback-only");
        }
      }
      ...
}
```

<br/>

- 던진 예외인 TestException은 RuntimeException이기 때문에 rollback-only 마킹을 하게 됨
- 참여 중인 트랜잭션이 실패하면 기본정책이 전역롤백(= globalRollbackOnParticipationFailure default 값이 true) 이기 때문에 마지막 순간에 UnexpectedRollbackException을 던짐

> **globalRollbackOnParticipationFailure** 속성 (주석)   
Set whether to globally mark an existing transaction as rollback-only after a participating transaction failed.   
Default is "true": If a participating transaction (e.g. with PROPAGATION_REQUIRED or PROPAGATION_SUPPORTS encountering an existing transaction) fails, the transaction will be globally marked as rollback-only. The only possible outcome of such a transaction is a rollback: The transaction originator cannot make the transaction commit anymore.   
참여 중인 트랜잭션이 실패한 후에 기존 트랜잭션을 전역적으로 rollback-only로 마킹할 것인지 설정   
디폴트는 true   
PROPAGATION_REQUIRED 또는 PROPAGATION_SUPPORTS 인 참여 중인 트랜잭션이 실패하면, 그 트랜잭션은 전역적으로 rollback-only로 마킹됨   
이런 트랜잭션은 결과적으로 롤백되고 최초의 트랜잭션관리자도 그 트랜잭션을 커밋시킬 수 없음  
> 

<br>

한마디로 정리해보면,     
전파속성때문에 실제 트랜잭션이 재사용되어서 **커밋이나 롤백같은 최종완료처리는 최초 트랜잭션이 반환될 때** 일어나더라도, **트랜잭션의 완료처리(completion)는 트랜잭션 메서드의 반환시점마다** 하기 때문에 **마킹만 해두고** 최초의 트랜잭션이 완료처리되는 마지막 순간에 **UnexpectedRollbackException**을 던진것!     
따라서 @Transactional 에서 예외가 터지면 롤백 마크를 하기 때문에 해당 트랜잭션을 재사용할 수 없다    

<br/>

## 해결

1. globalRollbackOnParticipationFailure 속성을 false로 바꾼다    
⇒ 전체 트랜잭션에서 다 바뀌므로 굳이!
2. 현재 **채팅 서버가 응답을 주지 않을 경우** 별도의 정책이 있기 때문에     
채팅서버와의 통신에서 실패 시, 별도의 예외를 던지지 않고 정책에 맞는 값을 return 한다 ✔️   

<br/><br/>

Reference   
[우아한 기술블로그 : 응? 이게 왜 롤백되는거지?](https://techblog.woowahan.com/2606/)    
[Transaction silently rolled back because it has been marked as rollback-only](https://keencho.github.io/posts/transaction-rollback/)     
[당신은 트랜잭션에 대해 얼마나 알고 있는가](https://dkswnkk.tistory.com/700)

<br/>
