---
title: DB) JPA Association Mapping
excerpt: Book project, 연관관계 매핑
---

# JPA 연관관계 매핑
`테이블 간의 연관 관계`가 있을때 `객체지향`스럽게 사용하는 방법을 제공  
- `방향` : 단방향, 양방향 존재
  - 방향은 객체관계에만 존재하고, 테이블은 항상 양방향
  - 회원, 팀 관계가 있을 때
    - 게시글 → 댓글 or 댓글 → 게시글 둘 중 `한 쪽만 참조`한다면 `단방향`
    - 게시글 → 댓글, 댓글→ 게시글 `양쪽 모두 서로 참조`하는 것은 `양방향` 관계
- 객체를 양방향 연관관계로 만들면 `연관관계의 주인`을 정해야함

<br/>

## Board, Reply 예시

하나의 Board(게시글)는 여러개의 Reply(댓글) 가질 수 있음  
`Board 엔티티`가 `Reply 엔티티를 여러개` 가질 수 있다는 연관관계 

<br/>

### 1. Board 입장 : @OneToMany

Board Entity 

```
@OneToMany(mappedBy = "board", fetch = FetchType.EAGER, cascade = CascadeType.REMOVE)
@JsonIgnoreProperties({"board"})
@OrderBy("id desc")
private List<Reply> replys;
``` 

- `mappedBy = "board"` : Reply 엔티티의 board 필드와 매칭  
  - 실질적으로 생성되는 Board 테이블을 보면 Reply 에 대한 정보가 아무것도 없음  
    -> mappedBy를 사용해 단순히 참조  
  - `주인이 아니면 mappedBy를 사용`하여 주인을 지정해야 함

- `fetch` : 하나의 Entity 를 조회할 때, `연관관계에 있는 객체들을 어떻게 가져올 것이냐`를 나타내는 설정값
  - `FetchType.EAGER` : 연관 관계에 있는 Entity 들 같이 가져옴 (Eager 전략)  
    - 게시글 조회하면 연관 관계에 있는 댓글도 무조건 따라오게
  - `FetchType.LAZY` : 연관 관계에 있는 Entity 가져오지 않고, getter 로 접근할 때 가져옴 (Lazy 전략, default)
    - 게시글 조회하더라도 원할때만 댓글 따라오게

- `cascade` : 특정 엔티티를 영속 상태로 만들때 `연관된 엔티티도 함께 영속 상태로` 만들고 싶을때 사용
  - ex) 부모 엔티티 저장(삭제) 시 자식 엔티티도 저장(삭제)하게끔  
  - `CascadeType.REMOVE` : 부모 엔티티를 삭제할때 연관되어 있는 자식 엔티티도 `같이 삭제`하는 옵션(게시글 삭제하면 댓글도 삭제되게)

<br/>

- `@JsonIgnoreProperties` : 해당하는 json 데이터(reply의 board필드) null -> 순환참조 해결  
  - 안그러면 현재 양방향 매핑이라 Board 호출 시 -> Reply 참조 -> Board -> Reply -> ... 순환참조 발생  

- `@OrderBy` : 리스트 순서 지정  

<br/>

### 2. Reply 입장 : @ManyToOne

Reply Entity  

```
@ManyToOne
@JoinColumn(name="boardId")
private Board board;
```

- `@JoinColumn(name="boardId")` : Reply 테이블에서 Board pk 저장 컬럼명이 boardId
  - 연관관계의 주인임을 나타냄
  - 물리 테이블에 있는 boardId 컬럼을 통해 board 필드를 채워줌

<br/>

### 정리  
- 사실 객체에는 양방향 연관관계라는 것이 없음  
  => `서로 다른 단방향 연관관계 2개를 묶어서` 양방향인 것처럼 보이게 하는 것  
- 데이터베이스는 `외래 키 하나`로 `양쪽이 서로 조인`할 수 있음  
  => 외래 키 하나만으로 양방향 연관관계

<br/>

- 객체 연관관계  
  - 게시글 → 댓글 연관관계 1개(단방향)  
  - 댓글 → 게시글 연관관계 1개(단방향)  
- 테이블 연관관계  
  - 게시글 <-> 댓글 연관관계 1개(양방향)  

<br/>

엔티티를 양방향 연관관계로 설정하면 `객체의 참조는 둘`(서로 참조하니까)인데 `외래 키는 하나`    
-> 따라서 둘 사이에 차이가 발생 -> 외래키 관리하는 쪽(연관관계의 주인) 필요  
=> `연관관계의 주인` : `Foreign Key(외래키)` 가지고 있는 Reply   

<br/>

참고)  
@ManyToOne에는 mappedBy 옵션이 없기 때문에 @OneToMany가 관계의 주인일 경우 양방향 매핑이 불가능 = @ManyToOne은 항상 주인  

<br/>

### BoardEntity

```
@Entity
public class Board {
	
	@Id
	@GeneratedValue(strategy=GenerationType.IDENTITY) 
	private int id;
	
	@Column(nullable = false, length = 100)
	private String title;
	
	@Lob
	private String content;
	
	private int count;
	
	@ManyToOne
	@JoinColumn(name="userId")
	private User user;
	
	@OneToMany(mappedBy = "board", fetch = FetchType.EAGER, cascade = CascadeType.REMOVE)
	@JsonIgnoreProperties({"board"})
	@OrderBy("id desc")
	private List<Reply> replys;
	
	@CreationTimestamp
	private LocalDateTime createDate;

}
```

![제목 없음2](https://user-images.githubusercontent.com/103614357/187645387-ecaf21e2-ff00-4566-a6a9-feccb55d12c8.png)  

Board 테이블에는 reply 컬럼이 없음  
Reply 테이블에 Board pk키를 가진 컬럼이 있음  

<br/>

참고)  
- `@Lob` : 데이터베이스에서 VARCHAR보다 큰 데이터를 담고 싶을 때 사용    
- `@CreationTimestamp` : insert 시 현재 시간으로 쿼리 생성  

<br/><br/>

Reference  
https://velog.io/@devsh/JPA-%EC%97%B0%EA%B4%80-%EA%B4%80%EA%B3%84-%EB%A7%A4%ED%95%91-OneToMany-ManyToOne-OneToOne-ManyToMany  
https://soojong.tistory.com/entry/JPA-ManyToOne-OneToMany-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0  
https://cornswrold.tistory.com/350  
https://cornswrold.tistory.com/350   
<br/>
