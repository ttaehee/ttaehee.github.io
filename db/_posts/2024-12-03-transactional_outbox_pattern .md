---
title: 분산 시스템에서 이벤트 유실을 방지하는 방법은? Transactional Outbox Pattern 활용하기
excerpt: "Transactional Messaging :: Transaction 내 이벤트 처리와 Outbox table을 활용한 안정적인 비동기 이벤트 처리"
---

<br/>

개발을 해볼수록 느끼는건 분산 시스템에서 동기적인 처리만으로는 사용자 경험과 시스템 성능을 동시에 만족시키기가 어렵다  
이를 해결하기 위해 사내코드에 트랜잭션 내 이벤트 비동기 처리 방식을 도입해보았지만, @TrasactionEventListener의 AFTER_COMMIT 을 사용하는 경우 사실 이벤트 처리의 유실 위험이 있다    
그렇다고 BEFORE_COMMIT 을 동기 방식으로 사용하기에는 결국 전체 로직이 끝나야 해당 api 의 응답이 전송되기 때문에 똑같다 그렇다고 비동기 처리를 하기에는 정합성 문제가 발생할 수 있다    

<br/>

트랜잭션 내 이벤트 처리 시 발생할 수 있는 정합성 문제를 해결하면서도 api 응답 시간을 단축하려면 안정적인 비동기 처리 방식이 필요하다
그래서 여러 대안을 검토한 결과 Transactional Outbox Pattern 을 적용하였다    

<br/>

## 배경 및 문제상황   
현재 우리 서비스에서는 유저가 견적서를 제출할 때 관련 수수료 정보를 생성하도록 구현되어 있다     
그러니까 견적서 제출 로직에 수수료 정보 생성 로직도 있는것이다   
수수료 정보 생성 로직은 비즈니스 핵심 기능 중 하나이지만, 다른 서버에 요청을 해야하는데 기존 구현에서는 동기적으로 처리되었기 때문에 전체 api 응답 시간 지연의 주요 원인이 되었다      
또한 트랜잭션 내부에서 여러 작업을 처리하다 보니 확장성에도 제약이 있었다   

<br/>

따라서, 응답 시간 지연을 해소하고, 트랜잭션 처리와 수수료 정보 생성 로직을 분리하여 확장에도 유연하도록 수정이 필요했다    

<br/><br/>

## 해결방안 1 :: 트랜잭션 내 이벤트 처리 방식
### @TransactionEventListener의 AFTER_COMMIT 의 문제점
@TransactionEventListener의 AFTER_COMMIT은 트랜잭션이 커밋된 후 이벤트를 처리하므로 트랜잭션의 상태가 완전히 반영된 이후에 이벤트를 처리한다   
따라서 이 방식은 트랜잭션 내에서 발생할 수 있는 정합성 문제를 방지할 수 있다    
그러나 트랜잭션 커밋 이후에 발생한 네트워크 장애나 서버 오류 등으로 이벤트가 처리되지 않을 수도 있는데, 그런 경우 트랜잭션 내에 이벤트 처리가 포함되어 있지 않기 때문에 이로 인해 작업이 누락될 수 있다       
수수료 정보 생성 로직은 매출과 직결되는 부분으로 누락되면 안되는 작업이기 때문에 이러한 중요한 로직은 트랜잭션 내에서 처리되는 것이 안전했다   

<br/>

###  @TransactionEventListener의 BEFORE_COMMIT 의 문제점  
그러면 트랜잭션 내에 포함되도록 해서 이벤트를 발행하고 처리하면?    
@TransactionEventListener의 BEFORE_COMMIT 으로 처리하면 트랜잭션 커밋 전에 이벤트를 처리하려는 목적에는 부합하지만, 비동기적으로 이벤트 처리를 할 경우 트랜잭션의 상태와 일치하지 않을 수 있는 위험이 존재한다      
예를 들어, 트랜잭션이 롤백되면 이미 처리된 이벤트는 상태에 반영되지 않아 이벤트 유실이 발생할 수 있다    

<br/>

이 방식은 비즈니스 로직의 정합성을 보장할 수 없기 때문에, 이벤트 처리의 신뢰성을 보장할 수 없는 문제가 있다   
그렇다고 안정적으로 가기위해 동기적으로 처리한다면 응답 지연 문제는 해소되지 못한다    

<br/>

따라서 비동기 처리 방식으로 트랜잭션 내에 포함되지 않은 이벤트 처리는 위험할 수 있으며, 트랜잭션과 별개로 비즈니스 로직의 정합성과 안정성을 보장하면서도 성능을 개선할 방법이 필요했다    

<br/>

## 해결 방안 2 :: Transactional Outbox Pattern
이 문제를 해결하기 위해 Transactional Outbox Pattern을 활용하여 로직을 비동기로 처리했다     
이를 통해 주요 트랜잭션 작업과 부가적인 작업(이벤트 처리)을 분리하고, 시스템의 성능과 확장성을 동시에 확보했다    

<br/><br/>

## Transactional Outbox Pattern 구현

### 1) 트랜잭션 내에서 이벤트 발행
트랜잭션 내에서 이벤트 발행

<br/>

### 2) TransactionalEventListener를 활용한 이벤트 저장
TransactionalEventListener의 BEFORE_COMMIT 단계에서 이벤트가 트랜잭션 범위 내에서 안전하게 저장되므로 데이터 정합성을 보장할 수 있다   
별도 처리 없이 이벤트를 저장만 하므로 응답 지연을 개선할 수 있다    

- outbox 테이블에 이벤트 기록 (TransactionPhase.BEFORE_COMMIT)
  
```java
@TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
public void register(SubmittedEvent event) {
    eventRecorder.save(event.toEventRecordCommand());
}
```

<br/>

### 2) Outbox 테이블과 비동기 이벤트 처리
Outbox 테이블에 저장된 이벤트는 별도의 비동기 처리 서비스에서 주기적으로 읽어들여 비동기로 처리된다       
메시지 큐(RabbitMQ, Kafka 등)나 다른 비동기 로직으로 전달할 수 있는데 나는 크론 작업을 통해 처리하는 방식을 선택했다    
이미 돌아가고 있는 크론에 충분히 접목시킬 수 있었기에 개발 난이도와 시간을 절감할 수 있었다      

<br/><br/>

## 적용 결과
- 응답 시간 개선 : 기존: 2초 → 개선 후: 115ms
- 확장성 향상 : 이벤트 기반 비동기 처리로 인해 시스템 부하 분산 가능 + 새로운 로직 추가 시 기존 트랜잭션 로직에 영향을 주지 않음
- 데이터 정합성 보장 : 트랜잭션 성공 후에만 Outbox 테이블에 이벤트 기록

<br/>

이번 작업은 단순한 성능 향상에 그치지 않고 이벤트 기반 시스템을 겪어본 느낌이다     
이벤트 기반 시스템으로의 전환 가능성에 눈뜬..?    
같은 jvm 내에서의 이벤트 처리는 익숙했는데 분산 환경에서의 이벤트 처리는 처음 해보았다   

<br/>

Transactional Outbox Pattern은 동기적 api 로직에서 발생할 수 있는 성능 저하 문제를 해결하면서도, 데이터 정합성과 확장성을 모두 확보할 수 있는 전략이었다   
시스템이 점점 확장될수록 이러한 설계가 중요한 역할을 할거같다     

<br/><br/>

Reference    
- [트랜잭셔널 아웃박스 패턴의 실제 구현 사례 (29CM)](https://medium.com/@greg.shiny82/%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%94%EB%84%90-%EC%95%84%EC%9B%83%EB%B0%95%EC%8A%A4-%ED%8C%A8%ED%84%B4%EC%9D%98-%EC%8B%A4%EC%A0%9C-%EA%B5%AC%ED%98%84-%EC%82%AC%EB%A1%80-29cm-0f822fc23edb)
- [강남언니 공식 블로그) 분산 시스템에서 메시지 안전하게 다루기](https://blog.gangnamunni.com/post/transactional-outbox/)
- [리디 공식 블로그) Transactional Outbox 패턴으로 메시지 발행 보장하기](https://ridicorp.com/story/transactional-outbox-pattern-ridi/)

<br/>
