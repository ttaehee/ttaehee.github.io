---
title: Spring) Service Layer Unit Test
excerpt: Book project, JUnit5 & Mockito
---

## JUnit5 & Mockito를 이용한 Spring Service Layer 단위테스트  
기술스택 : SpringBoot, Gradle, JPA, Junit5  
DB : h2  

<br/>

### build.gradle  
- assertJ의 `assertThat` 사용위한

```
testImplementation("org.assertj:assertj-core:3.23.1")
```

참고) JUnit 의 assertThat & AssertJ의 assertThat  
- JUnit 의 assertThat
  - 메소드들을 import 해놓아야함, 안그럼 자동완성 안됨  
  - 필요한 메소드 중 데이터타입에 맞는 알맞은 항목 찾아야함  
  - 추가 조건 검증 시 : allOf이라는 메소드로 기존 조건과 묶어줘야 함  
    ```
    assertThat(result, allOf(
        greaterThan(0),
        lessThan(10)
    ));
    ```
- AssertJ의 assertThat
  - assertThat에서 반환되는 Assert클래스 사용 -> 자동완성 지원  
  - 인자의 데이터타입에 맞는 Assert 클래스를 반환 -> 필요한 메소드만 분류되어 있음  
  - Method Chaining 패턴(메소드 리턴 객체를 받고 이 리턴 객체의 메소드를 호출하는 방법을 반복) -> 가독성 좋음  
    ex) A.method1().method2()...
    ```
    assertThat(result)
      .isGreaterThan(0)
      .isLessThan(10);
    ```

<br/>

### BookServiceTest.java

1. 단위테스트 작성준비  

```java
@ExtendWith(MockitoExtension.class)
public class BookServiceTest {
	
    @InjectMocks
    private BookService bookService;

    @Mock
    private BookRepository bookRepository;

    @Mock
    private MailSender mailSender;

}
```

- `@ExtendWith(MockitoExtension.class)` : Mockito 의 Mock 객체 사용위한 annotation
  - `Mockito` : 가짜객체 보관환경
  - @SpringBootTest :  application을 띄우기때문에 통합테스트에 용이(시간도 오래걸리고 무겁기때문에 단위테스트하는데는 알맞지 않음)
- `@InjectMocks` : `가짜환경에 있는 의존성`(@Mock 붙은)까지 주입해줌
- `@Mock` : Mock 객체로(가짜객체) -> Mockito(가짜환경)에 뜸   

![qwe](https://user-images.githubusercontent.com/103614357/187077034-e50f843a-4514-4b11-98a0-d564df011a75.png)   

- `Mockito 사용 이유`  
  : Repository 테스트는 이미 완료했기 때문에 서비스-레파지토리-DB 까지 테스트할 필요가 없음(무겁기만함)      
  -> `Repository는 메모리에 로드할 필요 없음` => 가짜객체 Mock으로!  

<br/>

2. 책 등록 단위테스트 메소드

```java
    @DisplayName("책 등록")
    @Test
    public void insertBook_test() {
    	
        //given
        BookSaveReqDto dto = new BookSaveReqDto();
        dto.setTitle("junit5");
        dto.setAuthor("김태희");

        //stub(가설)
        when(bookRepository.save(any())).thenReturn(dto.toEntity());
        when(mailSender.send()).thenReturn(true);

        //when
        BookRespDto bookRespDto = bookService.insertBook(dto);

        //then
        assertThat(bookRespDto.getTitle()).isEqualTo(dto.getTitle());
        assertThat(bookRespDto.getAuthor()).isEqualTo(dto.getAuthor());
    }
```

<br/>

3. 책 수정 단위테스트 메소드

```java
    @DisplayName("책 수정")
    @Test
    public void updateBook_test() {
    	
    	//given
        Long id = 1L;
        BookSaveReqDto dto = new BookSaveReqDto();
        dto.setTitle("spring");
        dto.setAuthor("태희");

        //stub
        Book book = new Book(1L, "junit", "김태희");
        Optional<Book> bookOP = Optional.of(book);
        when(bookRepository.findById(id)).thenReturn(bookOP);

        //when
        BookRespDto bookRespDto = bookService.updateBook(id, dto);

        //then
        assertThat(bookRespDto.getTitle()).isEqualTo(dto.getTitle());
        assertThat(bookRespDto.getAuthor()).isEqualTo(dto.getAuthor());	
    }  
```

<br/>

### Dto 생성이유

- Domain : `Book.java`  
- Dto : `BookRespDto.java`, `BookSaveReqDto.java` 생성   

Dto 생성한 이유!

```
jpa:
  open-in-view: true
```

true일 경우 영속성 컨텍스트가 트랜잭션 범위를 넘어선 레이어(Controller)까지 살아있음   
= `Controller단까지 영속화된 객체(Persistent Context)`를 넘겨주게 되면 `lazy loading` 이라는 변수 발생!  

<br/>

참고) lazy loading : 연관관계 있는 애들까지 다 가져옴  
사용자가 보지 않는 것들을 당장 로딩하지 않고  
나중에 사용자가 필요로 하는 시점에 로딩하는 것  

<br/>

1. Controller : Dto를 받아서 Service로  
2. Service : Dto를 받아서 Object(book)로 바꿔서 Repository로  

```
//BookSaveReqDto.java
public Book toEntity() {
    return Book.builder()
            .title(title)
            .author(author)
            .build();
}
```

```
//BookService.java  
Book bookPS = bookRepository.save(dto.toEntity());  
```

(Controller에서 받은 Dto(`dto`)를 Entity(`dto.toEntity()`)로 바꿔서 Repository로)   

4. Repository : Object(book) 받아서 CRUD ex) save()  
5. Persistent Context check : 없으면 DB(메모리에)에 book 저장 -> `영속화된 Book 객체(bookPS)`생성   
6. `Service에서 transaction 종료` + commit(메모리에서 HDD로 보내서 저장) or rollback(안보내고 메모리에서 지움) 
    - 값 변경은 끝    
7. `Controller에서 DB세션 종료`  
    - select는 가능  
    - 근데 만약, bookPS를 Service에서 넘겨줘서 `Controller에서 bookPS.getId()`를 해버리면?  
      => DB에서 끌고오니까 lazy loading  
      => 클라이언트로 보낼 때 message converter가 bookPS와 연관관계 있는 애들까지 JSON으로 다 보냄 굳이 요청하지 않은 정보인데도!  

==> 결론 : `Service에서 Controller로 Dto로 보내주기!`  
      
```
//Book.java
public BookRespDto toDto() {
    return BookRespDto.builder()
            .id(id)
            .title(title)
            .author(author)
            .build();
}
```

```
//BookService.java 
return bookPS.toDto();
```

(Repository에서 받은 Entity(`bookPS`)를 Dto(`bookPS.toDto()`)로 바꿔서 Controller로)  

<br/><br/>

Reference  
https://www.youtube.com/playlist?list=PL93mKxaRDidEZfpXoyWZ-2ZLsYrQByDMP  
https://jwkim96.tistory.com/168  
<br/>
