---
title: Spring) Spring Service Layer Unit Test
excerpt: JUnit & Mockito
---

## JUnit5 & Mockito를 이용한 Spring Service Layer 단위테스트 예시  
### 회원가입 / 유저목록 조회 비즈니스 로직 레이어(Service Layer)  

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserServiceImpl {

    private final UserRepository userRepository;
    private final BCryptPasswordEncoder passwordEncoder;

    @Transactional
    public UserResponse signUp(final SignUpRequest request) {
        final User user = User.builder()
                .email(request.getEmail())
                .pw(passwordEncoder.encode(request.getPw()))
                .role(UserRole.ROLE_USER)
                .build();

        return UserResponse.of(userRepository.save(user));
    }

    public List<User> findAll() {
        return userRepository.findAll().stream()
            .map(UserResponse::of)
            .collect(Collectors.toList()));
    }
}
```

<br/>

### 단위테스트 작성준비

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @InjectMocks
    private UserService userService;

    @Mock
    private UserRepository userRepository;

    @Spy  // 실제로 사용자 비밀번호 암호화해야하므로
    private BCryptPasswordEncoder passwordEncoder;

}
```

<br/>

### 회원가입 단위테스트 메소드

```java
@DisplayName("회원 가입")
@Test
void signUp() {
    // given
    BCryptPasswordEncoder encoder = new BCryptPasswordEncoder();
    SignUpRequest request = signUpRequest();
    String encryptedPw = encoder.encode(request.getPw());

    doReturn(new User(request.getEmail(), encryptedPw, UserRole.ROLE_USER)).when(userRepository)
        .save(any(User.class));
        
    // when
    UserResponse user = userService.signUp(request);

    // then
    assertThat(user.getEmail()).isEqualTo(request.getEmail());
    assertThat(encoder.matches(signUpDTO.getPw(), user.getPw())).isTrue();

    // verify : userRepository의 save(), encode() 1번만 호출되었는지 검증
    verify(userRepository, times(1)).save(any(User.class));
    verify(passwordEncoder, times(1)).encode(any(String.class));
}
```

<br/>

### 유저목록 조회 단위테스트 메소드

```java
@DisplayName("사용자 목록 조회")
@Test
void findAll() {
    // given
    doReturn(userList()).when(userRepository)
        .findAll();

    // when
    final List<UserResponse> userList = userService.findAll();

    // then
    assertThat(userList.size()).isEqualTo(5);
}

private List<User> userList() {
    List<User> userList = new ArrayList<>();
    for (int i = 0; i < 5; i++) {
        userList.add(new User("test@test.test", "test", UserRole.ROLE_USER));
    }
    return userList;
}
```

<br/>
