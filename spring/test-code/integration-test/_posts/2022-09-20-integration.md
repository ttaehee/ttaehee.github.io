---
title: Spring) Integration Test
excerpt: Book project, Junit5 통합테스트
---

## Integration Test 통합테스트  
기술스택 : SpringBoot, Gradle, JPA, Junit5  
DB : h2  

<br/>

## 통합테스트 이유  
- 단위테스트처럼 기능검증을 위한 것이 아니라 전체적인 flow가 제대로 동작하는지 검증하기 위해  
- 모든 Bean을 Container에 올리고 테스트  
  - 운영환경과 유사한 환경에서 전체적인 테스트 가능 -> Code Coverage가 높아짐  
  - but 시간 오래걸림, 특정계층 or 특정빈에서 발생하는 오류 디버깅 어려움  
- 단위테스트로는 RequestMapping, Data Binding, Type Conversion, Validation 등 커버할 수 없음  

<br/>

참고)    
[Code Coverage](https://tecoble.techcourse.co.kr/post/2020-10-24-code-coverage/) : 테스트케이스가 얼마나 충족되었는지 나타내는 지표     

<br/>

### @SpringBootTest
- Spring(SpringBoot아닌)에서 test할 때 사용하던 @ContextConfiguration 대용  
- test를 위한 Application Context를 로딩하며 여러가지 속성 제공  
- Spring Boot에서 제공하는 "spring-boot-starter-test"에 포함된 라이브러리  

<br/>

- JUnit4와 사용 시 @RunWith(SpringRunner.class) 추가해주어야 함

<br/>

참고)  
- spring-boot-starter-test 에 포함된 라이브러리    
  - JUnit 5 
  - Spring Test & Spring Boot Test 
  - AssertJ
  - Hamcrest
  - Mockito
  - JSONassert
  - JsonPath

<br/>

## BookApiControllerTest.java

### @SpringBootTest

통합테스트 작성준비  

```java  
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class BookApiControllerTest {
```

- webEnvironment : 웹 테스트 환경구성 가능
  - RANDOM_PORT 
    - EmbeddedWebApplicationContext를 로드하며 실제 서블릿환경 구성    
    - 실제 내장톰캣 사용  
    - MockMvc 대신 RestTemplate 사용할 수 있음  
    - 실제 가용한 포트로 내장톰캣을 띄우고 응답을 받아서 테스트를 수행
    - 임의의 port listen
  - DEFINED_PORT
    - application property 에서 지정한 port listen
  - Mock
    - WebApplicationContext를 로드하며 내장된 서블릿 컨테이너가 아닌 Mock 서블릿 제공

<br/>

```	
@Autowired
private BookService bookService;

@Autowired
private TestRestTemplate rt;

private static ObjectMapper om;
private static HttpHeaders headers;

@BeforeAll
public static void init() {
  om = new ObjectMapper();
  headers = new HttpHeaders();
  headers.setContentType(MediaType.APPLICATION_JSON);
}
```

- TestRestTemplate : REST 방식으로 개발한 API의 Test를 최적화 하기 위해 만들어진 클래스
  - RestTemplate과 거의 유사한 형태 but RestTemplate을 확장하지 않으며 몇가지 다른 기능을 제공
- ObjectMapper : Java 객체 <-> JSON 객체
  - 생성비용이 비쌈 -> bean/static 으로 처리하는 것이 좋음
  - writeValue() : Java객체 -> JSON 직렬화
  - Jackson 라이브러리는 Getter와 Setter를 이용하여 prefix를 잘라내고 맨 앞을 소문자로 만드는 것으로 필드를 식별   
    -> 내가 만드는 메서드나 생성자 이름에 get이라는 단어가 들어가지 않도록 주의  

<br/>

```	
@Test
public void saveBook_test() throws JsonProcessingException {

  //given
  BookSaveReqDto bookSaveReqDto = new BookSaveReqDto();
  bookSaveReqDto.setTitle("스프링1강");
  bookSaveReqDto.setAuthor("태희");

  String body = om.writeValueAsString(bookSaveReqDto);

  //when
  HttpEntity<String> request = new HttpEntity<>(body, headers);

  ResponseEntity<String> response = rt.exchange("/api/v1/book", HttpMethod.POST, request, String.class);

  //then
  DocumentContext dc = JsonPath.parse(response.getBody());
  String title = dc.read("$.body.title");
  String author = dc.read("$.body.author");

  assertThat(title).isEqualTo("스프링1강");
  assertThat(author).isEqualTo("태희");

}
```

- DocumentContext : interface
- JsonPath : JSON 객체를 탐색하기 위한 표준화된 방법
  - 경로 표현식을 사용하여 JSON 문서에서 요소, 중첩된 요소 및 배열을 탐색, 액세스   
  - Dot표기법
    - `$` : 모든 path표현식의 시작, root node로부터 시작하는 기호
    - `@` : 현재노드
    - `.` : 자식노드

<br/><br/>

Reference  
https://galid1.tistory.com/735  
https://goddaehee.tistory.com/211   
https://easybrother0103.tistory.com/64   
https://velog.io/@zooneon/Java-ObjectMapper%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-JSON-%ED%8C%8C%EC%8B%B1%ED%95%98%EA%B8%B0    
https://seongjin.me/how-to-use-jsonpath-in-kubernetes/
https://tecoble.techcourse.co.kr/post/2020-10-24-code-coverage/  
<br/>
