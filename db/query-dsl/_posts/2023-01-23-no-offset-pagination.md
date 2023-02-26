---
title: DB) No offset pagination 구현으로 조회 성능 올리기
excerpt: QueryDSL과 MySQL Descending index 사용하기
---

<br/>

QueryDSL과 No offeset, 그리고 MySQL의 Descending index 정리하기!   

<br/>    

## 일기(회고) 

- 지난번 JPA 게시판 과제에서 No offset pagination을 구현하긴 했는데      
  Spring Data JPA로만 구현을 해서 if/else로 짜다보니 코드가 조금 지저분한 느낌이었다     
  그래서 이번에는 QueryDSL을 공부해서 사용해보았다        
  사용해보니 쿼리를 코드로 짜게되니까 훨씬 눈에 잘 보이고 오타걱정이 없어 좋았다     
  현재는 간단한 쿼리라, 최저가 최고가 구현하면서 조금 더 복잡한 쿼리를 만져봐야겠다   
  
<br/>
 
- 이번에 MySQL Descending index를 공부하다가 느낀것은    
  DB 관련 내용을 보다보면 Lock이 빠지지 않는듯하다       
  이번에 Optimistic Lock을 적용할만한 상황이 있는데,      
  조금 더 자세히 공부하고 기회가 된다면 구현까지 해보아야겠다       

<br/><br/>  

## Pagination    

### 기존의 페이징 방식
- 페이지 번호(offset)와 페이지 사이즈(limit) 기반  
- 기존에 사용하던 paging query  

```
SELECT *
FROM items
WHERE 조건문
ORDER BY id DESC
OFFSET 페이지번호
LIMIT 페이지사이즈
```

![제목 없음](https://user-images.githubusercontent.com/103614357/213918299-8bb367fb-b5c7-4257-b683-3b6f355dc605.png)   

=> 앞에서 읽었던 행을 다시 읽게됨 -> 페이지가 뒤로 갈수록 느려짐 
 
- ex) offset 10000, limit 20    
  - offset은 몇번째 row부터 시작할지를 나타냄    
  - 최종적으로 10,020개의 행을 읽어야함(10000부터 20개를 읽어야하니까)
  - 이 중 앞의 10000개 행을 버리게 됨(실제 필요한건 마지막 20개뿐이니까)

  = 뒤로 갈수록 버리지만 읽어야 할 행의 개수가 많아짐 -> 뒤로 갈수록 점점 더 느려짐   

<br/><br/>  
 
- 극단적인 예면서 현실적으로 있을만한 예를 하나 봤는데 확 와닿았다     
  10000번째 게시글까지 누가 보겠어?라는 생각을 할 수 있는데    
  누군가 10000번째 게시글까지 조회를 했고          
  그 글이 마음에 들어서 해당페이지 링크를 단톡방이나 SNS에 공유한다면? 이라는 예시였다            
  - 그 많은 데이터를 읽어오고 버리고 하는 일이 일어나겠지     
    성능도 문제고, 부담도 있고    

<br/><br/>  

### No offset pagination 
- 페이지 번호(offset)가 없는 더보기(More) 방식     
  - 조회 시작 부분을 인덱스로 빠르게 찾아 매번 첫 페이지만 읽도록 하는 방식  
    - cluster index인 PK를 조회 시작 부분 조건문으로 사용했기 때문에 빠르게 조회됨

```
SELECT *
FROM items
WHERE 조건문
AND id < 마지막조회ID
ORDER BY id DESC
LIMIT 페이지사이즈
```

- 마지막 조회 결과의 ID(cluster index)를 조건문에 사용 = 이전에 조회된 결과를 한번에 건너뛸수 있음     

<br/><br/> 
  
## QueryDSL

### QueryDSL dependency 추가 + 설정하기   

**build.gradle**          

```
buildscript {
    ext {
        queryDslVersion = "5.0.0"
    }
}
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.7.7'
    id 'io.spring.dependency-management' version '1.0.15.RELEASE'
    //QueryDSL
    id 'com.ewerk.gradle.plugins.querydsl' version '1.0.10'
}
dependencies {
    ...
    
    //QueryDSL
    implementation "com.querydsl:querydsl-jpa:${queryDslVersion}"
    implementation "com.querydsl:querydsl-apt:${queryDslVersion}"
}
tasks.named('test') {
    useJUnitPlatform()
}
```

- QueryDSL version 전역변수로 정보 추가    
- QueryDSL plugins와 dependencies 추가  

<br/>  

```
//QueryDSL
def querydslDir = "$buildDir/generated/querydsl"
querydsl {
    jpa = true
    querydslSourcesDir = querydslDir
}
```
- QueryDSL에서 JPA 사용 여부와 사용할 경로 설정  

<br/>

```
sourceSets {
    main.java.srcDir querydslDir
}
```

- build 시 사용할 sourceSet 추가  

<br/>   

```
compileQuerydsl {
    options.annotationProcessorPath = configurations.querydsl
}
```

- QueryDSL compile 시 사용할 옵션 설정  

<br/>

```
configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
    querydsl.extendsFrom compileClasspath
}
```
 
- QueryDSL이 compileClassPath를 상속하도록 설정  
 
<br/><br/>     

**JPAQueryFactory bean으로 등록하기**          

```java
@Configuration
public class QueryDslConfig {
	@PersistenceContext
	EntityManager entityManager;
	@Bean
	public JPAQueryFactory jpaQueryFactory() {
		return new JPAQueryFactory(entityManager);
	}
}
```

<br/>

- 이렇게 설정 후, 

<img width="253" alt="KakaoTalk_20230123_003354451" src="https://user-images.githubusercontent.com/103614357/213924604-c88fabee-9231-4665-964f-5798ba6e9711.png">

Gradle Tasks에서 compileQuerydsl을 실행하면    

<img width="296" alt="KakaoTalk_20230123_003356382" src="https://user-images.githubusercontent.com/103614357/213924650-bb45d341-7b5a-446d-ad87-c50b66db71b2.png">

`build/generated/querydsl` 경로에 Project Entity들의 QClass가 생성됨    

![KakaoTalk_20230123_003352477](https://user-images.githubusercontent.com/103614357/213924692-1021e173-14fc-426d-ae75-2ba83e8bc1c4.png)

<br/><br/>

###  QueryDSL로 No offset 구현하기       

- 첫 페이지 조회와 두번째 페이지부터 조회 시 사용되는 쿼리 다름 -> 동적 쿼리 필요     
  - 첫 페이지를 조회할때는 기준이 되는 id 값이 없으니까     
  
=> QueryDSL의 where에 조건문을 쓰되 첫 페이지 조회처럼 parameter(id)가 비어있다면, 조건절에서 생략 되게끔 구현하기       
=> Querydsl의 BooleanExpression을 사용하자     

<br/>

```
CREATE INDEX index_product_name ON product (`name`);
```
  
```
@Repository
@RequiredArgsConstructor
public class ProductCustomRepositoryImpl implements ProductCustomRepository {
	private final JPAQueryFactory jpaQueryFactory;
	QProduct qProduct = QProduct.product;
	@Override
	public List<Product> findAllByCursor(Long cursorId, int pageSize) {
		return jpaQueryFactory
				.selectFrom(qProduct)
				.where(
					ltProductId(cursorId),
					qProduct.name.like(searchWord + "%")
				)
				.orderBy(qProduct.id.desc())
				.limit(pageSize)
				.fetch();
	}
	private BooleanExpression ltProductId(Long cursorId) {
		if (cursorId == null) {
			return null;
		}
		return qProduct.id.lt(cursorId);
	}
}
```

- BooleanExpression : where에서 사용할 수 있는 값  
- QueryDSL의 where는 null이 parameter로 올 경우 조건문에서 제외함     
  - ltProductId(Long cursorId)에서 cursorId가 null일 경우 where 조건문에 null이 들어가 제외됨   

<br/>

- 요청에서 받은 cursorId보다 작은 아이디의 상품을 pageSize만큼 조회한다   
- 그리고 Service layer에서 조회한 데이터 중 마지막 데이터의 id를 lastId로 해서 같이 응답해주었다  
  - 다음번 페이지 요청시 lastId를 보내게끔   
  - 처음엔 Slice를 이용해서 boolean hasNext를 보냈는데, lastId 보내는 방법이 더 깔끔한듯해 변경했다   

<br/>  

- 추가(23/01/30)
- 상품 목록 조회 시, 검색 기능을 추가했다    
  검색어와 같이 요청 시, 검색어가 포함된 상품만 조회하도록 하였다     
  - 처음에 일단 기능이 돌아가는것에 초점을 맞추어, Like와 %를 사용해 구현하였고    
    대신, full scan은 막고자 뒤에만 %를 붙였다
  - 시간이 된다면 중간값 검색 기능을 위해, 리팩토링 시 Full-Text search를 공부해 적용해 볼 예정   

<br/>

- QueryDSL 표현식 
  - lt : < 
  - loe : <= (Less or Equal)
  - gt : >
  - goe : >=

<br/><br/>

## MySQL의 Descending index   

- 현재 구현할 때, 최신순으로 상품을 보여주려고 id순으로 desc정렬을 하는데,        
  `Descending index` 기능이 MySQL 8.0부터 도입되었다는 글을 읽고 적용해보았다           
  = 역순으로 정렬되는 인덱스(Descending index)를 생성할 수 있게됨     
  
```
CREATE INDEX index_product ON product(id DESC);
```
  
- 전엔 mysql에서 desc 문법만 존재하고, 실제 Descending index가 지원되는 것은 아니었다고 한다        
  - 기존 index 생성 시에 desc로 생성하여도 문법 상 오류가 발생하진 않지만, 실제로는 asc로 생성되었음    
    = Ascending index를 생성하고 `ORDER BY id DESC` query로 index를 Backward scan으로 읽는 실행 계획 사용  

<br/>
 
### Forward index scan(Forward scan)과 Backward index scan(Backward scan)

![제목 없음](https://user-images.githubusercontent.com/103614357/213920717-3b63a40a-ad33-4bc8-9346-fc553df7a983.png)

- Descending index를 사용하려고 결정한 이유는, Backward index scan이 느리기 때문이다               
  [Ascending index를 Forward scan하는 경우와 Backward scan하는 경우의 성능 비교](https://tech.kakao.com/2018/06/19/mysql-ascending-index-vs-descending-index/)  

<br/>    

- 왜 느린가?   
  - 1)페이지 잠금이 Forward index scan에 적합한 구조 
    - InnoDB 스토리지 엔진에서는 페이지 잠금 과정에서 데드락을 방지하기 위해 B-Tree의 왼쪽에서 오른쪽 순서(Forward)로만 잠금 획득하도록 하고 있음   
      - 그래서 Forward index scan에서는 다음 페이지 잠금 획득이 매우 간단하지만   
      - Backward index scan에서 이전 페이지 잠금을 획득하는 과정은 상당히 복잡한 과정       
        -> 많은 페이지를 스캔해야 하는 Index scan에서는 잠금 획득으로 인한 쿼리 처리 지연 발생   
  - 2)페이지 내에서 인덱스 레코드는 단방향으로만 연결된 구조(Forwarded single linked link)    

<br/>

- 그렇지만 사실, 첨부한 글에서 보듯이 1천2백여만건을 스캔하는데, 1.2초 정도의 차이가 난다     
  = 현재 내 데이터의 크기에서는 성능의 차이를 못느꼈다 ㅋㅋ(포스트맨으로 열심히 해봤지만 이렇다할 정도의 차이를 느끼지 못함 ㅎ)        
  그리고, 상품 목록 가져오는 부분에서 요 정도의 성능은 크게 중요하지 않을듯 하지만ㅋㅋ    
  아무튼 공부한걸 적용해보았고, 내 데이터 크기에서 느껴지지 않을뿐이지 위의 결과를 보면 역순 정렬 쿼리가 정순 정렬 쿼리보다 28.9% 더 시간이 걸리는 걸 알 수 있었다       

<br/><br/>

Reference    
https://jojoldu.tistory.com/528     
https://jojoldu.tistory.com/394      
https://tech.kakao.com/2018/06/19/mysql-ascending-index-vs-descending-index/      
https://wooseok-uzi.tistory.com/5      
https://medium.com/naver-cloud-platform/%EC%9D%B4%EB%A0%87%EA%B2%8C-%EC%82%AC%EC%9A%A9%ED%95%98%EC%84%B8%EC%9A%94-mysql-8-0-%EA%B0%9C%EB%B0%9C%EC%9E%90%EB%A5%BC-%EC%9C%84%ED%95%9C-%EC%8B%A0%EA%B7%9C-%EA%B8%B0%EB%8A%A5-%EC%82%B4%ED%8E%B4%EB%B3%B4%EA%B8%B0-3-indexes-e32249e2dae5    
https://data-make.tistory.com/728   

<br/>   
