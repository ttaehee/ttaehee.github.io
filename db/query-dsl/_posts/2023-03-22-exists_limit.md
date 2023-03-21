---
title: DB) QueryDSL로 limit 사용해서 데이터 존재 여부 쿼리 최적화하기   
excerpt: exists와 count 쿼리 성능 비교 후 exists 구현하기 
---

<br/>

- 회원 API를 구현하면서 이메일과 닉네임 중복체크 기능을 구현하는데    
  현재 이 데이터가 DB에 있는지 없는지를 체크해야했다    
  - 그 과정에서 exists와 count 쿼리의 성능을 비교해보았고   
    exists의 성능이 더 좋음을 알게되었다 
    
<br/>
  
- 관련 내용을 학습하는 중 [\[우아콘2020\] 수십억건에서 QUERYDSL 사용하기](https://www.youtube.com/watch?v=zMAX7g6rO_Y&ab_channel=%EC%9A%B0%EC%95%84%ED%95%9C%ED%85%8C%ED%81%AC)를 보았고 도움이 많이 되었다     
  그러나 현재는 3년후인만큼, querydsl의 수정된 부분을 발견할 수 있었다     
  - QueryDSL에서 기본적으로 지원하는 exists는   
    count 쿼리 방식을 사용해 성능상의 이슈가 있어      
    Querydsl에서 limit을 사용하여 직접 구현한 exists 쿼리를 사용한 설명의 영상인데    
  - `QuerydslJpaPredicateExecutor.java`를 보니   
    exists 메서드에서 fetchCount 함수를 사용한다고 한 부분이    
    fetchFirst 함수를 사용하고 있는것을 알 수 있었다    
    QueryDSL 5.0부터 fetchCount가 deprecated 되면서 변경된듯 하다  
    하지만 여전히 sub query에서만 가능해 limit을 사용하여 쿼리를 최적화해 사용했다   

<br/>

## exists와 count
특정 조건을 만족하는 row가 있는지 체크 = 데이터가 있는지 없는지 체크   

<br/>

### sql에서의 exists와 count

- exists : 조건 만족하는 처음 값 발견하면 -> 쿼리종료

```
select exists (select 1
                from member
                where email = 'email'
)
```

- count : 조건 만족하는 처음 값 발견해도 -> 끝까지 조건 체크   
  -> 전체 데이터를 다 조회하기 때문에 성능이 떨어질 수 밖에 없음 (특히 스캔 대상이 앞에 있을수록 더 심한 성능차이가 있음)    
  
```
select count(*)
from member
where email = 'email'
```

<br/><br/>
 
## 2020 우아콘 영상

### 우아콘 영상에서 나왔던 문제점

**JPQL에서의 exists**     

- JPQL에서 select의 exists(select exists 문법)를 지원하지 않음     
  - where의 exists만 지원하기 때문에        
  -> count 쿼리를 사용해서 데이터 존재 여부를 확인해야함        
  -> 성능 이슈가 따라옴     
  
  ```java
  @Query("SELECT COUNT(m.email) > 0 FROM Member m WHERE m.email =:email")
  boolean existsUsingCount(@Param("email") String email);
  ```
  
<br/><br/> 

**QueryDSL에서의 exists**    

sql의 exists를 사용하지 않고 count 사용해서 체크했었음(fetchCount 함수 사용) (과거)        
= exists 쿼리가 아닌 count query 발생했었음 

- 현재는 fetchFirst 함수를 사용하고 있음을 확인할 수 있었음     
  - `QuerydslJpaPredicateExecutor.java`의 exists()     

  ![querydsl_exists](https://user-images.githubusercontent.com/103614357/221418545-10ca9af4-de11-4c03-badd-80c4ee688069.png)    

<br/>   

- 그럼에도, QueryDSL은 from절 없이 쿼리 생성이 불가하기 때문에      
  = select exists 하고 하위에 select 쿼리를 만드는 방식이 공식적으로 지원 안됨 
  - from 절을 붙여야 하는데, 그렇게 되면 exists로의 성능 개선을 볼 수 없음    
  - 아래의 예시는 컴파일 에러는 나지 않지만 실행 후에는 에러가 떠 사용 불가   

  ```java
  public Boolean exist(String email) {

    return queryFactory.select(queryFactory
            .selectOne()
            .from(member)
            .where(member.email.eq(email))
            .fetchAll().exists())
            .fetchOne();
  }
  ```
  
<br/><br/>   

### 해결 :: limit 1로 개선하기  
exists가 count 보다 성능이 좋은 이유 : 전체를 조회하지 않고 첫번째 결과만 확인하기 때문     
=> 이 내용을 가지고 직접 구현해보자   
=> limit 1로 1개만 조회해보고, 이 결과로 있는지 없는지 판단하기    

<br/>
 
```java
public Boolean existsUsingLimit(String email) {
    Integer result = queryFactory
            .selectOne()
            .from(member)
            .where(member.email.eq(email))
            .fetchFirst();

    return result != null; 
}
```

- `fetchFirst()` = `limit(1).fetchOne()`    
- 조회결과가 없으면 0이 아닌 null 반환함   
  - 1개가 있는지 null인지 판단하기    

[sql의 exists와 동일한 성능효과](https://jojoldu.tistory.com/516)를 볼 수 있었다     

<br/><br/>

- count 쿼리와 limit으로 구현한 쿼리 비교

```java
String email = "test75000@mail.com";

@BeforeEach
void insertData() {
  for (int i = 0; i < 100000; i++) {
    Member member = Member.builder()
        .email("test" + i + "@mail.com")
        .build();
    memberRepository.save(member);
  }
}

@AfterEach
void cleanData() {
  memberRepository.deleteAll();
}

@Test
void sqlCount() {
  long startTime = System.currentTimeMillis();
  long endTime = 0;
  if(memberRepository.existsUsingCount(email)) {
    endTime = System.currentTimeMillis();
  }

  System.out.println("==============================================");
  System.out.printf("sqlCount 수행시간 : %d\n", endTime - startTime);
  System.out.println("==============================================");
}

@Test
void querydslCustomExists() {
  long startTime = System.currentTimeMillis();
  long endTime = 0;
  if(memberRepository.existsUsingLimit(email)) {
    endTime = System.currentTimeMillis();
  }

  System.out.println("==============================================");
  System.out.printf("querydslCustomExists 수행시간 : %d\n", endTime - startTime);
  System.out.println("==============================================");
}
```
  
![KakaoTalk_20230321_215529100](https://user-images.githubusercontent.com/103614357/226653996-ecd6655e-2c44-400a-b57c-905a918b7964.png)   

![KakaoTalk_20230321_215530413](https://user-images.githubusercontent.com/103614357/226654003-4bfe6221-e850-423f-a6de-6e03c7ee891a.png)   

<br/><br/>

## Spring Data JPA 쿼리 메서드의 exists   
쿼리 메서드도 내부에서 limit으로 쿼리 최적화를 하고 있음         

![datajpa](https://user-images.githubusercontent.com/103614357/221423538-707dc335-1213-433e-83da-bde252c5f1b0.png)   

<br/><br/>

## 고려한 방법들

1. 쿼리 메서드
2. QueryDSL에서 limit 1 사용
   
<br/>
  
처음엔 쿼리 메서드로 사용했다 내부에서 limit으로 쿼리 최적화가 되어있으니까    
그런데 현재 나는 Member Entity에서 Email을 embedded type으로 사용하고 있어서         
쿼리 메서드를 사용하려면 Email 객체를 생성해서 조회해야 했다      

- 전 
  - Repository layer

  ```java
  boolean existsByEmail(Email email);
  ```

  - Service layer

  ```java
  private boolean checkEmailDuplicate(Email email) {
    return memberRepository.existsByEmail(email);
  }
  ```

<br/>

불필요한 객체 생성을 피하고자 결론적으로 QueryDSL에서 limit을 사용해 쿼리를 최적화했다     

- 후
  - Repository layer

  ```java
  public boolean existsByEmail(String email) {
    Integer fetchOne = jpaQueryFactory
        .selectOne()
        .from(qMember)
        .where(qMember.email.email.eq(email))
        .fetchFirst();
    return fetchOne != null;
  }
  ```

<br/><br/>

Reference           
[JPA exists 쿼리 성능 개선](https://jojoldu.tistory.com/516)   
[Avoid Using COUNT() in SQL When You Could Use EXISTS()](https://blog.jooq.org/avoid-using-count-in-sql-when-you-could-use-exists/)     
[JPQL, QueryDsl, Spring Data의 exists](https://yainii.tistory.com/36)   

<br/>  
