---
title: DB) JPA의 deleteAll로 인한 N+1 해결하기
excerpt: bulk delete 구현하기
---

테스트

- 예전에 [bulk insert, bulk delete 관련해서 정리](https://ttaehee.github.io/spring/spring-framework/spring-boot/batch/)한적이 한번 있었는데     
  이번에 회사코드에 적용하려다가 bulk delete 방법들이 그 때와 조금 다르게 와닿아서 다시 정리해보려한다     

<br/>

## 상황

Cron 돌려야 하는 작업을 관리하는 테이블에서 처리 완료된 row를 한번에 삭제하려고함
(테이블 이름을 CronBatchTask 라고 임의로 지칭)

<br/>

## 문제    

### deleteAll(Iterable<? extends T> entities) → N+1   

일단 저번에 알아본 기억으로 JPA의 deleteAll도 saveAll 처럼 for문을 돌기 때문에 쿼리가 N번 나갈거라 예상했고 실제로 테스트 결과에서도 그러했다
그래서 요 문제를 풀고싶었다   

- ex)
  
```java
List<CronBatchTask> cronBatchTaskList = cronBatchTaskRepository.findByTaskType(CronTaskType.TEST);
... (관련 처리)
cronBatchTaskRepository.deleteAll(cronBatchTaskList);
```

<img src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/bc01b91a-0da7-424d-890d-e465b6e34c3d" width="600"/>

=> N+1

<br/>

- 참고)
  
<img src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/f1588779-ddc9-4f37-b96f-0a66095475ec" width="600"/>

- deleteAll : N개의 엔티티들을 null이 아닐시 반복문을 통해 delete() 중

<br/>

## 해결법

### 1) @Query    

지난번에 사용했던 방법이다 

```java
@Modifying(clearAutomatically = true) 
@Query("DELETE FROM CronBatchTask cbt WHERE cbt.id in :cronBatchTaskIdList")
void deleteAllBy(List<Long> cronBatchTaskIdList);
```

- 단점
    - 단순 문자열 → 가독성 떨어짐, 문법상 오류 파악 어려움
    - 오류를 컴파일 시점에 알 수 없음
    - 쿼리 작동시 오류 파악 가능
    - DB에 의존적   

<br/>

지난번엔 크게 와닿지 않았던 단점들인데 회사코드에 적용하려니 추후 유지보수에 있어 단점들이 크게 와닿았다   
아마도 예전엔 유지보수 경험이 없고 초기 개발만 경험해봐서 그렇지 않았을까 싶다     
그래서 이번에 이 방법은 패쓰   

<br/>

### 2) QueryDSL

```java
public void deleteAllBy(List<Long> cronBatchTaskIdList) {
    QCronBatchTask cronBatchTask = QCronBatchTask.cronBatchTask;
    jpaQueryFactory.delete(cronBatchTask)
        .where(cronBatchTask.id.in(cronBatchTaskIdList))
        .execute();
}
```

- 장점
    - 복잡한 쿼리도 가능
- 단점
    - 1차 캐시 미사용 (커스텀 가능)

복잡한 쿼리에 적용하면 좋을듯하다     
1차 캐시를 사용하지 않기 때문에 필요에 따라 커스텀하면 좋을듯    

<br/>

### 3) deleteAllInBatch(Iterable<T> entities)

내가 알아본 방법중 마지막 방법, 요 아이도 JPA에서 제공해주는 아이다         

```java
List<CronBatchTask> cronBatchTaskList = cronBatchTaskRepository.findByTaskType(CronTaskType.TEST);
... (관련 처리)
cronBatchTaskRepository.deleteAllInBatch(cronBatchTaskList)
```

<img src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/37ef1feb-1abe-4fae-b6ed-22f60beb2e79" width="600"/>

<img src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/77e58ec7-1216-44b4-8365-0bedb1394682" width="600"/>

- 단점 + 그렇지만 현재 상황에서는 ~
    - 외래키 관련 삭제 안됨    
        => CronBatchTask 테이블에는 외래키가 없어서 괜찮을것으로 보임
    - 삭제하기 전에 전체적인 데이터에 대한 select()를 한번 하고 1차 캐시에 저장함    
        => 삭제 전에 조회해서 관련 처리 완료 후 삭제해야해서 어차피 1차 캐시에 저장되어 있음    
        테스트 했을 때도 삭제를 위한 별도 조회 일어나지 않는것 파악하여 괜찮을것으로 보임

<br/>

<img width="582" alt="스크린샷 2024-05-23 오후 11 44 09" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/66fb2968-a7cd-47c8-8faf-cf4a94e2ce7f">

좋은 방법이지만 삭제하고자 하는 Entity들을 메모리상에 가져와서 호출해야 하기 때문에 지난번에는 선택하지 않았다     
하지만 이번에는 위에 작성했듯이 삭제 전, 해당 entity 관련해서 필요한 작업이 있었어서 어차피 1차캐시에 저장되어 있을 상황이었기 때문에 단점이라고 받아들여지지 않았다    
테스트 결과로 쿼리가 한번만 나가는(bulk delete) 것과 삭제를 위한 별도 조회 쿼리가 나가지 않는 것을 확인했다      

<br/>

## 결론

조건 등이 복잡한 쿼리가 아니기 때문에 개발 효율성과 가독성 측면에서 더 좋다고 생각하여 JPA에서 제공해주는 deleteAllInBatch()를 사용했다     
만약 추후 조건이 복잡한 쿼리의 경우를 마주한다면 querydsl을 조금 커스텀해서 사용할듯하다    

<br/>
