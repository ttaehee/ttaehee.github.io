---
title: 성능 테스트 중 발견한 IN절 바인딩 수의 요상함, 얘는 쿼리 바인딩이 왜 이렇게 여러번 되는거지?  
excerpt: hibernate 의 in_clause_parameter_padding 속성
---

<br/>

- 이번엔 쿼리 성능 테스트 중 새로 알게된 hibernate 의 in_clause_parameter_padding 속성 에 대해 기록해보겠다    

<br/>

## 상황
사내 쿼리 성능 테스트를 위해 쿼리 로그를 활성화한 상태에서 IN절의 바인딩 수가 비정상적으로 증가하는 현상을 발견했다   

코드와 쿼리를 대강 비슷하게 적어보자면   

<br/>

```java
jpaQueryFactory.select(e.id)
        .from(e)
        .where(
            e.status.in(List.of(a, b, c, d, e),
            ...
        ).fetch();
```

```sql
select e1_0.`id`
from `e` e1_0
where e1_0.`status` in (?, ?, ?, ?, ?, ?, ?, ?)
  ...
```

<img width="620" alt="스크린샷 2024-11-02 오전 12 02 33" src="https://github.com/user-attachments/assets/e2a9a55b-e0f6-4059-9752-ff1ec897d51b">    

<br/><br/>  

IN절에 5개의 항목을 사용했음에도 쿼리 로그에는 8개의 바인딩 값이 기록되었으며, 그중 3개는 예상하지 못한 값으로 채워져 있었다    
뭐지? 

그래서 in절에 항목을 1개부터 순서대로 쭉 넣어보았다

<br/>

## 테스트 과정 및 결과
문제를 파악하기 위해 IN절 항목 개수를 1개부터 점진적으로 증가시키며 로그를 확인해 보았다 (노 가다 ㅎ)      
그 결과 바인딩 수가 아래처럼 2의 제곱으로 증가하는 패턴이 관찰되었다    

- 항목 1개 → 바인딩 1개
- 항목 2개 → 바인딩 2개
- 항목 3개 → 바인딩 4개
- 항목 4개 → 바인딩 4개
- 항목 5개 → 바인딩 8개
- 항목 6개 → 바인딩 8개   
...   
- 항목 9개 → 바인딩 16개

<br/>

힌트는 로그의 org.hibernate.orm.jdbc.bind 에 있었다    
Hibernate의 공식 문서와 관련 자료에서 in_clause_parameter_padding 속성이 바인드 변수를 2의 제곱수 단위로 패딩하여 SQL 캐시 효율을 높이는 역할을 한다는 것을 알게되었다      
이 속성의 작동 방식이 IN 절 항목과 바인드 변수 개수의 증감 패턴과 일치했다    

<br/>

사내 application.yml 을 살펴보니 in_clause_parameter_padding 속성이 이미 활성화 되어있었다   

- hibernate.in_clause_parameter_padding=true    

  <img width="418" alt="스크린샷 2024-11-02 오전 12 11 29" src="https://github.com/user-attachments/assets/aca18c7e-de1e-4e46-a819-6403e15b52e9">   

처음보는 느낌 반성해 환경별 yml만 보다보니까,, (변명)     


<br/>

실제로 in_clause_parameter_padding 속성을 비활성화한 후 동일한 테스트를 했을 때, 항목 개수와 바인드 변수 개수가 1:1로 대응되며 더이상 불필요한 바인드 변수가 추가되지 않는 것을 확인했다   
신기하군   


<br/>

## hibernate 의 in_clause_parameter_padding 속성    
hibernate에서 DB 접근 및 요청은 결국 내부적으로 jdbc API 이용한다   
이 때, 매번 새로운 쿼리를 조립하지 않기 위해 hibernate에서는 실행되는 쿼리를 따로 memory에 캐시하여 성능을 최적화한다     
쿼리의 캐시 히트율을 높이기 위해 prepareStatement 기준으로 쿼리를 메모리에 저장해두며, IN 절에서는 캐싱을 효율적으로 하기 위해 in_clause_parameter_padding 옵션을 제공한다    

- in_clause_parameter_padding 옵션을 활성화하면   
    - IN 절에 들어가는 파라미터 수를 2의 거듭제곱으로 맞추고, 부족한 부분을 null로 채워 일정한 쿼리 구조를 유지하기 때문에      

      ⇒ 이로써 동일한 쿼리가 캐시에서 재사용될 수 있게 하여 메모리를 절약하고 쿼리 성능을 높일 수 있다       

- IN 절에서 parameter의 갯수만 달라져도 쿼리 결과에는 영향이 없으므로, 패딩된 쿼리가 기존과 같은 결과를 반환한다        

이 설정을 통해 IN 절에서의 캐싱 효율을 높여 메모리 사용량을 줄이고 쿼리 성능을 개선되는 것이다        

<br/>

↔ ex) 이러한 옵션이 없으면?    
IN 절에 전달되는 파라미터 개수가 매번 달라지면, 쿼리가 각각 다른 것으로 인식되어 캐싱되지 않기 때문에 캐시 효율이 떨어진다   
- 예를 들어, `IN (a, b, c)`과 `IN (a, b, c, d)`는 별도의 쿼리로 취급되어 캐싱의 이점이 줄어든다    

<br/>

결론은 IN절 항목 수와 바인딩 수의 증감이 단순히 항목 수와 1:1 대응되지 않는 이유는 Hibernate의 내부 최적화 전략이 작용하였기 때문이었다       

<br/>

## 참고) ArrayList 크기 확장 방식

ArrayList는 초기 용량이 설정되지 않으면 기본 크기 10으로 시작한다   
이후, 추가 공간이 필요할 때마다 기존 크기의 약 1.5배로 확장하는 방식으로 크기가 증가한다    

- 정확히는 ArrayList가 꽉 차면 \[newCapacity = (oldCapacity * 3) / 2 + 1] 계산식을 통해 약 1.5배 정도 크기가 증가된다    

1.5배씩 점진적으로 증가하기 때문에 메모리 낭비를 줄이고 필요한 만큼만 공간을 확보하는 효과가 있는데, 요건 2의 제곱으로 증가되는건 아니지만 용량이 점진적으로 증가한다는 점이 비슷하다고 생각하였다    

<br/><br/>

Reference    
- [nhncloud :: in-clause-padding](https://meetup.nhncloud.com/posts/211)
- [Baeldung :: in-clause-padding](https://www.baeldung.com/java-hibernate-in-clause-padding#parameter-padding-for-in-clause)

<br/>
