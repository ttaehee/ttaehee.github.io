---
title: Spring) Spring Repository Layer Unit Test
excerpt: Book project, JUnit5
---

## JUnit5를 이용한 Spring Repository Layer 단위테스트

**기술스택** : SpringBoot, Gradle, JPA, Junit5   
**DB** : h2  

### build.gradle

```
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	implementation 'org.springframework.boot:spring-boot-starter-web'
	compileOnly 'org.projectlombok:lombok'
	developmentOnly 'org.springframework.boot:spring-boot-devtools'
	runtimeOnly 'com.h2database:h2'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testImplementation("org.assertj:assertj-core:3.23.1")
}
```

- spring-boot-starter-web를 선택했다면 자동으로 spring-boot-starter-test 의존성이 추가
  - spring-boot-starter-test에 Junit5가 기본적으로 있음

<br/>

### applcation.yml

- application-dev.yml 별도 생성

```
spring:
  profiles:
    active:
    - dev
```

dev 버전 사용하겠다는 의미 

<br/>

### test를 위한 폴더구조 생성

![aaa](https://user-images.githubusercontent.com/103614357/186900152-97542dcc-fe99-4862-b8b7-350197789226.png)   

<br/>

### BookRepositoryTest.java

```
@DataJpaTest
public class BookRepositoryTest {
	
	@Autowired
	private BookRepository bookRepository;
	
    @BeforeEach
    public void getData() {
        String title = "junit";
        String author = "태희";
        Book book = Book.builder()
                .title(title)
                .author(author)
                .build();
        bookRepository.save(book);
    }
```

- `@DataJpaTest` : DB와 관련된 component만 메모리에 로딩
  - @Entity 가 붙은 클래스, Jpa Repository 스캔
  - auto configure 해주는 옵션이 많이 붙어있음
    - `@Transactional` 도 포함되어 있음
- `@BeforeEach` : 각 테스트 시작전 한번씩 실행
  - 메서드마다 데이터준비가 필요해서 만들었음
 
<br/> 
 
```
    @DisplayName("책 등록")
    @Test
    public void insert_test() {
		
        //given (데이터 준비)
        String title = "junit5";
        String author = "김태희";
        Book book = Book.builder()
                .title(title)
                .author(author)
                .build();

        //when(테스트 실행)
        Book bookPS = bookRepository.save(book);

        //then(검증)
        assertEquals(title, bookPS.getTitle());
        assertEquals(author, bookPS.getAuthor());
        
    }
```

- `@DisplayName` : 가독성 위해 사용
- `@Test` : Test 메서드로 인식
  - rollback default - true
- 하나의 메서드가 끝나면 트랜잭션 종료 = 저장된 데이터를 초기화함
  - 하나의 메서드는 `@BeforeEach부터 시작해서 insert_test()까지`

<br/>

```    
    @DisplayName("책 삭제")
    @Sql("classpath:db/tableInit.sql")
    @Test
    public void delete_test() {
    	
    	//given
    	Long id = 1L;
    	
    	//when
    	bookRepository.deleteById(id);
    	
    	//then
    	assertFalse(bookRepository.findById(id).isPresent());
    	
    }
```

하나씩 테스트를 할 때는 괜찮았는데 여러 테스트메서드 같이 테스트 돌리니까 에러  
=> 이유 : 테스트메서드는 `순서가 보장되지 않음`,  
하나의 메서드가 끝나면 트랜잭션 종료(데이터 초기화) 된다고 했는데,  
이 때 primary key를 auto_increment로 해놓으면 `pk는 초기화가 되지 않음`   

<br/> 
 
ex) insert_test()시 들어간 데이터(id가 1L)의 트랜잭션이 종료되어 데이터는 롤백되더라도, 메모리에는 pk(1L)가 남아있음  
delete_test()시의 들어간 데이터(@BeforeEach로 넣는)는 id가 2L로 들어가게 됨!    
나는 id가 1L인 데이터(롤백되어 비어있음)를 삭제하려고 하니까 에러  
순서가 보장되지 않기 때문에(보장 원하면 order 쓰기) id가 뭐가 될지 모름  
=> @Sql 사용  

<br/>

- `@Sql("파일")` : 명시한 파일은 테스트를 실행하는데 적합한 상태로 DB를 초기화하기 위해 DELETE, INSERT, CREATE와 같은 쿼리 포함
  - delete_test()에서 pk인 id를 사용해야 해서 table drop 하고 create하는 sql문 담긴 파일(tableInit.sql) 실행   
    => `아이디를 찾는 메서드`(findById(id))에는 사용해줘야함
- classpath:db/tableInit.sql
  - classpath(resource 영역, src/main/resources)에 있는 db폴더에 있는 tableInit.sql 파일

<br/>

```
    @DisplayName("책 수정")
    @Sql("classpath:db/tableInit.sql")
    @Test
    public void update_test() {
    	
    	//given
    	Long id = 1L;
    	String title = "transaction";
    	String author = "Kimtaehee";
    	Book book = new Book(id, title, author);
    	
    	//when
    	Book bookPS = bookRepository.save(book);
    	
    	//then
    	assertEquals(id, bookPS.getId());
    	assertEquals(title, bookPS.getTitle());
    	assertEquals(author, bookPS.getAuthor());
    	
    }

}
```

- 마찬가지로 아이디가 필요한 메서드라 @Sql 사용
- 메서드가 끝나자마자 롤백 -> dirty checking이 안되기 때문에 save()로 수정 진행  

<br/><br/>

Reference   
https://www.youtube.com/playlist?list=PL93mKxaRDidEZfpXoyWZ-2ZLsYrQByDMP  
<br/>
