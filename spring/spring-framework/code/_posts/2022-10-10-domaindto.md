---
title: Spring) Separate Domain and DTO    
excerpt: domain(entity) 과 dto를 분리해야하는 이유   
---

## Domain(Entity)과 DTO를 분리해야하는 이유

### 1. 관심사의 분리    
Entity와 DTO를 분리해야 하는 근본적인 이유는 관심사가 다르기 때문     
- DTO의 핵심 관심사는 이름 그대로 데이터의 전달    
  - 데이터를 담고, 다른 계층 또는 다른 컴포넌트들로 데이터를 넘겨주기 위한 자료구조    
    -> 어떠한 기능 및 동작도 없어야 함  
- Entity는 핵심 Business Logic을 담는 비지니스 도메인의 영역의 일부    
  -> Business Logic이 추가될 수 있음  
  
=> Entity와 DTO는 엄연히 서로 다른 관심사를 가지고 있음  
&nbsp; -> 분리하는 것이 합리적!    

<br/>

참고)  
관심사의 분리(separation of concerns, SoC)       
- 소프트웨어 분야의 오랜 원칙 중 하나  
- 서로 다른 관심사들을 분리  
  -> 변경 가능성 최소화하고, 유연하며 확장가능한 시스템을 만드는 것  

<br/>

### 2. Validation 로직 및 불필요한 코드 등과의 분리   
- @Valid 처리를 위해서는 @NotNull, @NotEmpty, @Size 등과 같은 annotation들을 필드에 붙여주어야 함    
  (Spring에서 요청 데이터 검증 위한 @Valid annotation 지원)  
- JPA도 변수에 @Id, @Column 등과 같은 annotation들을 활용해 객체-관계형 데이터베이스 매핑      

=> DTO와 Entity를 분리하지 않는다면 Entity의 코드가 상당히 복잡해짐     

<br/>

- ex) 분리하지 않았을 때    
  annotation들이 중구난방으로 작성되어 있는 Entity  
  -> 유지보수가 힘들어짐  

```java
@Entity
@Table
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Membership {

  @NotNull
  @Size(min = 0)
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  @Column(nullable = false)
  private Long id;

  @NotBlank
  @Column(nullable = false)
  private String userId;

  @NotNull
  @Size(min = 0)
  @Setter
  @Column(nullable = false)
  @ColumnDefault("0")
  private Integer point;

  @CreationTimestamp
  @Column(nullable = false, length = 20, updatable = false)
  private LocalDateTime createdAt;

```

또한, createdAt(생성일자) 변수는 API 요청 및 응답에서 필요 없을 수 있음  
-> 응답에서 해당 변수를 제거위해 @JsonIgnore 등과 같은 또 다른 annotation 붙여주어야 함  

=> Entity에 핵심 Business Domain code들이 아닌 요청/응답을 위한 값, 유효성 검증을 위한 코드 등이 추가됨     
-> Entity class가 지나치게 비대해짐, 확장 및 유지보수 어려움  

<br/> 

### 3. API 스펙의 유지    

- 아래 Entity class를 API의 응답으로 활용한다고 가정

```java
public class Membership {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  @Column(nullable = false)
  private Long id;

  @Column(nullable = false)
  private String userId;

}
```

- 응답 : JSON 메세지  

```
{
  "id" : "28",
  "userId" : "ttaehee"
}

```
 
- 내부 정책 변경으로 userId를 memberId로 변경해야 하는 상황이면?     
  -> DTO를 사용하지 않는다면 API의 스펙이 변경됨      
  -> API 사용하던 사용자들은 모두 장애를 겪게 됨   
  (물론 @JsonProperty로 반환되는 값의 이름 변경가능 but Entity를 무겁게 만들어 근본적인 해결책이 될 수 없음)  

- 스펙이 변경되어 table에 column이 추가되는 경우라면?      
  - table에 새로운 컬럼 추가 -> Entity에 새로운 변수 추가   
    -> 별도로 처리를 하지 않는 이상 API 응답이 추가되어 스펙이 변경됨  
  - 응답을 위한 DTO 클래스를 활용한다면   
    -> Entity class의 변수가 변경되어도 API 스펙이 변경되지 않으므로 안정성 확보

<br/>

=> DTO 이용해 분리    
-> 독립성을 높이고, 변경이 전파되는 것을 방지해야함     

<br/>

### 4. API 스펙의 파악이 용이   
 
```java
public class UserRequestDto {

	@Email
	@Size(max = 100)
	private final String email;

	@NotBlank
	private final String pw;

}
```
 
- 해당 스펙 완벽히 파악은 불가능, 그래도 꽤 많은 API 스펙 파악가능       
  - email의 값은 반드시 email 포맷, 최대 글자수 100, pw는 비어있음 안됨 등   
=> 어느 정도 API 문서의 요약본을 작성하는 것과 유사한 효과    
&nbsp; 특히 MSA 아키텍처에서 다른 사람이 작성한 코드 파악 시 요청/응답 스펙을 비교적 손쉽게 파악할 수 있음  

<br/>

### 5. DB Layer(Persistence Tier)와 View Layer(Presentation Tier) 사이의 역할을 분리
- Entity의 값 변경되면?  
  -> Repository 클래스의 Entity Manager의 flush가 호출될 때 DB에 값이 반영  
  -> 다른 로직들에도 영향    
  - DTO는 View와 통신하면서 필연적으로 데이터 변경이 많음   
    -> Entity의 값이 변경되지 않도록 따로 분리하기    

<br/>

- Domain Model을 아무리 잘 설계했다고 해도,   
  각 View 내에서 Domain Model의 getter만으로 원하는 정보를 표시하기가 어려운 경우가 있음    
  - 그래서 Domain Model 내에 Presentation을 위한 필드나 로직을 추가하면?  
    -> 모델링의 순수성을 깨고 Domain Model 객체를 망가뜨리게 됨  

<br/>

=> DTO는 Domain Model 객체(Entity)를 그대로 두고 복사하여, 다양한 Presentation Logic을 추가한 정도로 사용       
Domain Model 객체(Entity)는 Persistent만을 위해서 사용해야함!  

<br/><br/>

Reference    
https://mangkyu.tistory.com/192?category=761302     
https://velog.io/@ohzzi/Entity-DAO-DTO%EA%B0%80-%EB%AC%B4%EC%97%87%EC%9D%B4%EB%A9%B0-%EC%99%9C-%EC%82%AC%EC%9A%A9%ED%95%A0%EA%B9%8C   
https://umbum.dev/1206      
<br/>
