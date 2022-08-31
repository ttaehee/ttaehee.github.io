---
title: DB) JPA Entity Mapping
excerpt: 엔티티 매핑
---

# JPA Annotation

`객체-테이블` mapping : `@Entity`, `@Table`  
`필드 - 컬럼` mpping : `@Column`  
`기본키` mapping : `@Id`    
`연관관계` mapping : `@ManyToOne`, `@JoinColumn`  
<br/>

```
@Entity
public class User {
	
	@Id //primary key
	@GeneratedValue(strategy=GenerationType.IDENTITY) 
	private int id;
	
	@Column(nullable = false, length = 30, unique = true)
	private String username;
	  
	@Column(nullable = false, length = 100)
	private String password;
	
	@Column(nullable = false, length = 50)
	private String email;

	@Enumerated(EnumType.STRING)
	private RoleType role;
	
	private String oauth;
	
	@CreationTimestamp
	private Timestamp createDate;

}
```

![제목 없음](https://user-images.githubusercontent.com/103614357/187643020-006a0075-ef1a-4851-bfcc-9c186372e78e.png)  


<br/>

## 1. @Entity
table과 mapping 할 **class** 에 꼭 명시하기  

```
@Entity
//@Entity(name = "User")
public class User {
```

- name 
  - JPA에서 사용할 엔티티 이름 지정 (기본값 클래스명 사용)  
    `@Entity`
  - 다른 패키지에 같은 이름이 있다면 직접 지정하여 충돌 방지하여야함  
    `@Entity(name = "테이블명")`
- 접근지정자가 public, protected 인 기본생성자가 필수
- final, enum, interface, inner class에는 사용 불가
- table column에 mapping한 field에 final 사용 불가

<br/>


## 2. @Table
entity와 mapping할 **table** 지정  
지정하지 않으면 entity명을 테이블명으로 사용   
- name : 매핑할 테이블 이름 설정 (기본값 엔티티 이름)
- catalog : 카탈로그 기능이 있는 데이터베이스
- schema : schema 기능이 있는 데이터베이스
- uniqueConstraints(DDL) : 스키마 자동 생성 기능을 사용해서 DDL을 만들 때만 사용 가능
  - DDL 생성 시 유니크 제약조건을 만듬
  - 2개 이상의 복합 유니크 제약조건도 가능

<br/>


## 3. @Id

primary key  

```
@Id
//@Column(name = "id")
private String id;
```

- 키 생성 전략 설정 hibernate.id.new_generator_mappings  
  hibernate5가 AUTO의 전략을 TABLE로 변경 + 스프링 부트 2.x 로 들어오면서 새로운 전략을 따름  
  - true : SequenceStyleGenerator를 사용, 데이터베이스가 Sequence Generator를 지원하지 않는다면 Table Generator
    - 테이블 시퀸스 전략은 모든 엔티티에 대한 id값을 시퀸스 테이블 하나에서 통합적으로 관리
  - false : 기존의 IDENTITY 전략을 AUTO로 사용 + @GeneratedValue에 IDENTITY 전략을 직접 설정
- Java에서 @Id 적용가능 타입 : String, java.util.Date , java.sql.Date , java.math.BigDecimal , java.math.BigIntege 등  

<br/>

### 키 생성 전략

- `자동생성` : 대리키 사용방식 (@GeneratedValue 어노테이션을 추가 명시)  
  - `IDENTITY` 전략 : 선 Entity저장 후 식별자조회  
    - pk 생성을 데이터베이스에게 위임 (DB에 의존, MySQL auto_increment)  
  - `SEQUENCE` 전략 : 선 식별자조회 후 Entity 저장 
    - DB시퀀스(유일한 값을 순서대로 생성하는 DB 기능)를 사용 (DB에 의존, Oracle sequence) 
  - `TABLE` 전략 :  SEQUENCE전략과 내부 동작방식 같음(시퀀스 대신 테이블 사용한다는 것만 다름)
    - 키생성 테이블 만들어서 사용 (DB에 의존x, 모든 데이터베이스에 적용가능)   

<br/>

1)  IDENTITY 전략  

```
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY) 
private int id;
```

=> DDL 쿼리  
```
CREATE TABLE BOARD (  
  ID INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
```

<br/>

 JDBC3에 추가된 Statement.getGeneratedKeys() - 데이터를 저장하면서 동시에 생성된 기본 키 값 가져옴  
 
```
User user = new User();
userRepository.save(user)
System.out.println(user.getId());
```

<br/>
 
참고)  
persist() : reuturn 값이 없는 insert  
save() : return 값이 있는 insert, update  
 
<br/>

 2)  SEQUENCE 전략
 
```
@Entity
@SequenceGenerator(  
 name = "BOARD_SEQ_GENERATOR",     // BOARD_SEQ_GENERATOR라는 시퀀스 생성기 등록
 sequenceName = "BOARD_SEQ",    // DB에서 생성한 "BOARD_SEQ"와 매핑
 initialValue = 1, allocationSize = 1)
 
public class Board {
 @Id
 @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "BOARD_SEQ_GENERATOR") 
 private Long id;
}
```

=> DDL : 시퀀스를 테이블과 별개로 생성

```
CREATE TABLE BOARD (
  ID BIGINT NOT NULL PRIMARY KEY,
  DATA VARCHAR(255)
)
CREATE SEQUENCE BOARD_SEQ START WITH 1 INCREMENT BY 1;
```

boardRepository.save(board) 실행시,  
DB 시퀀스를 사용해서 ID 조회  
-> 조회한 ID를 엔티티에 세팅한 후 DB에 저장    

<br/>

3)  Table전략

```
@Entity
@TableGenerator(

 name = "BOARD_SEQ_GENERATOR",     // "BOARD_SEQ_GENERATOR" 라는 이름의 테이블 키 생성기 등록
 table = "MY_SEQUENCES",      // MY_SEQUENCES 테이블을 키 생성용 테이블로 매핑
 pkColumnValue = "BOARD_SEQ", allocationSize = 1)
 
public class Board {
 @Id
 @GeneratedValue(strategy = GenerationType.TABLE, generator = "BOARD_SEQ_GENERATOR") 
 private Long id
```
 
=> DDL 쿼리
```
create table MY_SEQUENCES (
 sequence_name varchar(255) not null ,
 next_val bigint,
 primary key ( sequence_name ) 
)
```  

sequence_name 컬럼을 시퀀스 이름으로 사용  
next_val 컬럼을 시퀀스 값으로 사용  
 
<br/> <br/>

## 4. @Column

```
@Column(nullable = false, length = 30, unique = true)
private String username;
```

- name : DB 컬럼명 설정 (기본값 객체명)
- `nullable = false` : NOT NULL 제약조건
- `updatable = false` : 컬럼 수정했을 때 DB에 반영 안됨
  - JPA는 ORM이기 때문에 필드명 바꾸면 테이블컬럼명도 바뀜
- `length = 30` : String 길이 설정 (default : varchar(255))
- `unique = true` : UNIQUE 제약조건

<br/>

## 5. @Enumerated

```
@Enumerated(EnumType.STRING)
private RoleType role;
```

Enum 객체 (열거형, 상수, final static)사용 시 사용 (DB에는 Enum Type이 없음)  
- `EnumType.ORDINAL` : Enum 순서를 데이터베이스에 저장 (default, but 비추!)  
  - 요구사항에 따라 새로운 타입 추가가능성 -> 순서 변경 가능성
- `EnumType.String` : Enum 이름을 데이터베이스에 저장 (사용하기!)
  - DB에는 Enum Type이 없으니까 이건 String이야 하고 알려주는것

<br/><br/>

Reference  
https://www.inflearn.com/course/ORM-JPA-Basic#  
자바ORM표준JPA프로그래밍(저자:김영한)  
<br/>
