---
title: Spring) Spring Controller Unit Test
excerpt: JUnit5 & Mockito
---

## Spring Controller 단위테스트 예시

### 회원가입/ 유저목록 조회 API  

```java
@RestController
@RequiredArgsConstructor
public class UserController {

  private final UserService userService;

  @PostMapping("/users/signUp")
  public ResponseEntity<UserResponse> signUp(@RequestBody SignUpRequest request) {
      return ResponseEntity.status(HttpStatus.CREATED)
          .body(userService.signUp(request));
  }

  @GetMapping("/users")
  public ResponseEntity<List<UserResponse>> findAll() {
      return ResponseEntity.ok(userService.findAll());
  }
}
```  

<br/>

### 단위테스트 준비  

```java
@ExtendWith(MockitoExtension.class)  // JUniit5와 Mockito를 연동
class UserControllerTest {

  @InjectMocks  // 가짜 객체 주입
  private UserController userController;

  @Mock  // 가짜 Mock 객체 생성
  private UserService userService;

  private MockMvc mockMvc;  //  Spring에서 제공하는 MockMVC를 이용하여 HTTP 호출

  @BeforeEach
  public void init() {
      mockMvc = MockMvcBuilders.standaloneSetup(userController).build();
  }

}

```

<br/>

### 회원가입 단위테스트 메소드  

```java
@DisplayName("회원 가입 성공")
@Test
void signUpSuccess() throws Exception {
  // given
  SignUpRequest request = signUpRequest();
  UserResponse response = userResponse();

  doReturn(response).when(userService)
      .signUp(any(SignUpRequest.class));  // any(특정클래스타입) : 특정클래스타입이라면 어떠한 객체도 처리할 수 있음

  // when
  ResultActions resultActions = mockMvc.perform(  // 요청정보 작성
      MockMvcRequestBuilders.post("/users/signUp")  // 요청정보에는 MockMvcRequestBuilders 사용 (요청메소드종류, 내용, 파라미터 등 설정가능)
              .contentType(MediaType.APPLICATION_JSON) 
              .content(new Gson().toJson(request)) 
              // 보내는 데이터는 객체가 아닌 문자열이여야 하므로 별도의 변환 필요하여 Gson 사용해 변환
  );

  // then (검증) : 회원가입 API 호출 결과로 200 response와 응답결과 검증해야함
  MvcResult mvcResult = resultActions.andExpect(status().isOk())
      .andExpect(jsonPath("email", response.getEmail()).exists())  // jsonPath를 이용해 해당 json값이 존재하는지 확인
      .andExpect(jsonPath("pw", response.getPw()).exists())
      .andExpect(jsonPath("role", response.getRole()).exists())
}

//SignUp에 대한 stub
private SignUpRequest signUpRequest() {
    return SignUpRequest.builder()
        .email("test@test.test")
        .pw("test")
        .build();
}

private UserResponse userResponse() {
    return UserResponse.builder()
        .email("test@test.test")
        .pw("test")
        .role(UserRole.ROLE_USER)
        .build();
}
```

<br/>

### 유저목록 조회 단위테스트 메소드 

```java
@DisplayName("사용자 목록 조회")
@Test
void getUserList() throws Exception {
  // given
  doReturn(userList()).when(userService)
      .findAll();

  // when
  ResultActions resultActions = mockMvc.perform(
      MockMvcRequestBuilders.get("/user/list")
  );

  // then
  MvcResult mvcResult = resultActions.andExpect(status().isOk()).andReturn();
  
  UserListResponseDTO response = new Gson().fromJson(mvcResult.getResponse()
      .getContentAsString(), UserListResponseDTO.class);  // json 응답을 객체로 변환하여 확인해봄
  assertThat(response.getUserList().size()).isEqualTo(5);
}

//UserService의 findAll에 대한 Stub
private List<UserResponse> userList() {
  List<UserResponse> userList = new ArrayList<>();
  for (int i = 0; i < 5; i++) {
      userList.add(new UserResponse("test@test.test", "test", UserRole.ROLE_USER));
  }
  return userList;
}
```

<br/>

### @WebMvcTest  
SpringBoot가 제공하는 Controller Test annotation   
  - MockMvc 객체 자동생성
  - 웹계층 테스트에 필요한 요소들을 모두 빈으로 등록해 스프링컨텍스트 환경 구성
    - ControllerAdvice
    - Filter
    - Interceptor 등
  - @Mock 대신 @MockBean
  - @Spy 대신 @SpyBean <br/><br/><br/>


Reference  
https://mangkyu.tistory.com/145
