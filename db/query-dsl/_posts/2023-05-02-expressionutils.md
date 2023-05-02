---
title: DB) 동적으로 여러 조건의 정렬 기능 구현하기  
excerpt: Querydsl에서의 subquery 위해 ExpressionUtils 사용하기
---

<br/>

- ORM 사용 시, 객체지향적으로 entity가 구성되어있으면 subquery가 필요한 일은 거의 없겠지만 동적으로 정렬을 구현하려다보니 필요했다       
  현재 [커서기반으로 페이지네이션을 구현](https://ttaehee.github.io/db/query-dsl/no-offset-pagination/)했는데 이 과정에서 커서를 식별자인 pk로만 해두니까 **좋아요순 정렬 시** 문제가 생겼다   
  - 이 과정에서 **ExpressionUtils**을 사용하여 where 절의 subquery를 생성했고 이를 기록해두려 한다    

<br/>

- 참고) 문제가 생긴 이유 : 좋아요순 정렬이면 현재 커서 기준으로 **식별자** 뿐 아니라     
  현재 커서의 좋아요 개수도 따져서 이보다 **적은 좋아요 개수를 가진 데이터**를 페이지사이즈만큼 보내줘야하는데     
  식별자만을 기준으로 페이지네이션을 해서 좋아요순 정렬이 제대로 작동하지 않았다    

<br/><br/>

## 정렬 기준 동적으로 적용하기

우리 서비스가 기획한 목록 조회의 정렬 기준은 최신순과 좋아요순 2가지이다    
따라서 클라이언트에서 온 **sort type에 따른 동적 정렬**이 필요하다    
sort type은 확장 가능성이 있어 enum으로 관리했다(NEWEST, LIKES)   
우선, 정렬 기준을 동적으로 적용하기 위해 orderBy가 필요로 하는 인자인 [OrderSpecifier](http://querydsl.com/static/querydsl/4.0.7/apidocs/com/querydsl/core/types/OrderSpecifier.html)를 사용했다       

<img width="553" alt="스크린샷 2023-04-28 오후 2 54 30" src="https://user-images.githubusercontent.com/103614357/235065841-dd206e5b-2265-4759-94a8-be5fbd75785c.png">

```java
private OrderSpecifier<Long> getSortType(SortType sortType) {
  if (sortType.equals(SortType.NEWEST)) {
    return qRoute.id.desc();
  }
  return qRoute.likeCount.desc();
}
```

위와 같이 관심사를 분리해서 **해당 로직에 맞는 동적 함수**를 만드니 가독성 뿐 아니라 디버깅이 쉬워 좋았다   
pk를 AUTO_INCREMENT로 설정했기에 최신순은 단순히 pk를 역순으로 정렬하였다   

<br/>

sort type 확장성을 고려해 SortType안에서 반환값을 만들고 switch문을 사용할까 했는데         
Q객체를 사용해야해서 SortType 안에서 처리가 어려웠고, 타입이 늘어난다면 코드 변화가 불가피할 것이라고 판단해    
조건이 2가지일 때 더 빠른 if문으로 구현하였다    

<br/><br/>

## ExpressionUtils

그리고, 좋아요 정렬 시에는 **현재 커서의 좋아요 개수보다 적은 데이터**들만 조회하도록 조건을 추가했다      
querydsl로 구현하는게 바로 머리 속에서 그려지지 않아서 쿼리문을 먼저 짜보았다(성능 개선의 여지가 있다)        
그래서 그런지 **where절의 subquery**가 필요했고 [ExpressionUtils](http://querydsl.com/static/querydsl/4.1.4/apidocs/com/querydsl/core/types/ExpressionUtils.html)를 사용했다    
ExpressionUtils는 Querydsl 내부에서 새로운 Expression을 사용할 수 있도록 지원한다   

<br/>

### cursor data가 유니크하지 않다면?     

좋아요 순으로 정렬하다보니 **커서가 좋아요 개수**가 되고 이는 **중복될 수 있는 유니크하지 않은 값**이다   
그렇다면 문제가 생길 것이다    
따라서 pk값인 id도 넣어 구현했다    

```java
private Predicate cursorFilter(Route cursorRoute, SortType sortType) {
  if (cursorRoute == null) {
    return null;
  }

  if (sortType == SortType.LIKES) {
    return ExpressionUtils.or(qRoute.likeCount.ne(cursorRoute.getLikeCount()), qRoute.id.lt(cursorRoute.getId()));
  }
  return null;
}
```
 
=> 정리 : 좋아요수가 같은 데이터가 있을 수 있으니까 **pk값을 기준으로 중복 데이터를 제거**하려는 시도를 해보았다   
(pk가 고유값이기 때문에 **조회의 정확성**을 높이고자)   

<br/><br/>

## 최종 쿼리   

- [github code](https://github.com/ttaehee/Team-5YES-WuMo-BE/blob/develop/src/main/java/org/prgrms/wumo/domain/route/repository/RouteCustomRepositoryImpl.java)

```java
@Override
public List<Route> findAllByCursorAndSearchWord(Route route, int pageSize, SortType sortType, String searchWord) {

  return jpaQueryFactory
      .selectFrom(qRoute)
      .where(inRouteAndHasSearchWord(searchWord),
          cursor(route, sortType),
          cursorFilter(route, sortType),
          isPublic())
      .orderBy(getSortType(sortType))
      .limit(pageSize)
      .fetch();
}
```

<br/><br/>

## 현재 구현에서 아쉬운 점

- 쿼리가 복잡
- or절 사용 -> 좋아요 개수는 변동사항이 많아 index를 걸지 않았으나, pk도 index를 타지 않게됨
- 성능의 우려가 있음

<br/>

### subquery 외에 현재 고려 예정인 방법들

- join으로 해결할 수 없을까?
- application 단에서 처리할 수 없을까?
- query를 나누어 실행할 수 없을까?    
- or 연산자를 없애고 custom cursor를 만들 수 없을까?

=> 현재 검색기능까지 동적으로 처리하고 있어 join이나 application단에서는 개선이 힘들어 보인다         
그러나 쿼리가 복잡하고 성능 우려가 있기 때문에 개선이 필요하다           
따라서 **좋아요개수와 식별자를 가지고 custom cursor를 만들어** 구현해 볼 예정이다    

<br/><br/>

Reference     

[Querydsl 서브쿼리 사용하기](https://jojoldu.tistory.com/379)    
[Querydsl로 무한스크롤 구현하기](https://velog.io/@znftm97/%EC%BB%A4%EC%84%9C-%EA%B8%B0%EB%B0%98-%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98Cursor-based-Pagination%EC%9D%B4%EB%9E%80-Querydsl%EB%A1%9C-%EA%B5%AC%ED%98%84%EA%B9%8C%EC%A7%80-so3v8mi2)     
[QueryDSL을 활용하여 SNS 피드 만들기](https://netal.tistory.com/93)  

<br/>
