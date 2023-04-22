---
title: Spring) OAuth2.0 flow와 Authorization code grant 방식으로 구현한 이유
excerpt: Spring Security와 OAuth2.0으로 소셜 로그인(구글) 구현하기
---

<br/>

- oauth flow를 익힐겸 처음에는 spring security 없이     
  직접 구글 로그인 페이지로 redirect 시키는 api와 authorization code를 요청하는 api, 코드 받아서 토큰 요청하는 api 등을 직접 구현했다     
  - 익힌 후에는 코드양이 훨씬 줄어들기 때문에 spring security oauth2를 사용해 구현했다 

<br/>

- OAuth 2.0의 4가지 방식 중 Authorization code를 통해 토큰을 받아 권한을 부여받는 Authorization code grant 방식 기반으로 구현했다 
  - 사용한 이유는 access token을 얻기 위한 인증 코드를 교환하는 단계가 있어 보안에 더 효과적이라고 생각했다          
  - 그 외의 방식 : Implicit Grant, Password Credentials Grant, Client Credentails Grant  

<br/>

## OAuth 2.0    
Open Authorization    

- 인터넷 사용자들이 비밀번호를 제공하지 않고    
  구글, 네이버, 카카오와 같은 다른 웹사의트 상의 자신들의 정보에 대해      
  제3자 클라이언트(우리의 서비스)에게 접근 권한을 부여할 수 있는(접근 위임) 표준 프로토콜   
  = 우리의 서비스가 우리 서비스를 이용하는 유저의 타사 플랫폼 정보에 접근하기 위해서 권한을 타사 플랫폼으로부터 위임 받는 것    

<img width="386" alt="스크린샷 2023-03-31 오후 5 55 41" src="https://user-images.githubusercontent.com/103614357/229075142-8473b5d3-ef3a-422b-8caa-4cd3547254c2.png">

<br/>

### OAuth 2.0 주체

**Resource Owner**     
- 리소스 소유자
- 우리의 서비스를 이용하면서, 구글, 네이버, 카카오 등의 플랫폼에서 리소스를 소유하고 있는 사용자

<br/>

**Authorization & Resource Server**     
- Authorization Server : Resource Owner를 인증하고, Client에게 액세스 토큰을 발급해주는 서버
- Resource Server : 구글, 네이버, 카카오 같이 리소스를 가지고 있는 서버

<br/>

**Client**     
- Resource Server의 자원을 이용하고자 하는 서비스
  - 보통 우리가 개발하려는 서비스
  - Resource Server와 Authorization Server의 입장에서는 우리 서비스가 클라이언트

<br/>

2.0의 특징인 Authorization Server와 Resource Server를 나눈 그림으로 보자면    

<img width="480" alt="스크린샷 2023-03-31 오후 6 03 56" src="https://user-images.githubusercontent.com/103614357/229146326-c318feff-40a7-4765-a0ec-cc5d0b4cc67d.png">

<br/><br/>

### Authorization Code가 필요한 이유

- redirect URI를 통해 Authorization Code 발급받는 과정이 생략된다면?    
  - Authorization Server가 redirect URI를 통해 Access Token을 Client에 전달해야 함       
    - redirect URI를 통해 데이터를 전달할 방법은 -> URL 자체에 데이터를 실어 전달하는 방법뿐     
      = 브라우저를 통해 데이터가 곧바로 노출됨       
- Access Token이 노출되지 않도록 보안을 위해 Authorization Code를 사용하는 것     

<br/>

**Flow**    
1. redirect URI를 프론트엔드 주소로 설정하여, authorization code를 프론트엔드로 전달   
2. 프론트에서 authorization code를 백엔드로 전달    
3. 코드를 전달받은 백엔드는 Authorization Server의 token 엔드포인트로 요청하여 access token을 발급받음

=> Access Token이 항상 우리의 application과 OAuth service의 백채널을 통해서만 전송됨 = 공격자가 access token을 가로챌 수 없음      
(처음에 생각했던 플로우는 이거였는데, 프론트쪽에서 시간상 구현이 어려워 일단 백엔드 쪽에서 code를 받는걸로 구현을 하였다)     

<br/><br/>

## Google OAuth 서비스 등록 & Client를 Resource Server에 등록

[구글 클라우드 플랫폼](https://console.cloud.google.com/home/dashboard)에서 새 프로젝트 만들기    

<img width="645" alt="스크린샷 2023-03-09 오후 4 22 38" src="https://user-images.githubusercontent.com/103614357/229043893-7723e05c-b8ca-4c34-b5c7-9c5dd87d7266.png">

- 웹 애플리케이션에서 승인된 redirection URI 설정 해주기
  - 승인된 리디렉션 URI : 서비스에서 파라미터로 인증 정보를 주었을 때 인증이 성공하면 구글에서 리다이렉트 할 URL 
  - `/login/oauth2/code/google`
    - SpringBoot 2.x버전의 Spring Security에서 default로 지원하는 redirect URL : `{도메인}/login/oauth2/code/{소셜서비스코드}`     
      -> 별도로 리다이렉트 URL을 지원하는 Controller를 만들 필요가 없음

<br/><br/>

## 구현

- security, oauth2-client dependency 추가

  ```
  //Spring Security
  implementation 'org.springframework.boot:spring-boot-starter-security'
  testImplementation 'org.springframework.security:spring-security-test'

  //OAuth2 Client
  implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
  ```

<br/>

### Google의 OAuth Client ID 설정

- 구글 서비스 적용을 위한 인증정보 application.yml에 설정
  - SpringBoot 2.0부터 CommonOAuth2 Provider라는 enum이 추가되었음    
    -> Google, Github, Facebook, Okta의 기본 설정 값들이 모두 제공됨    
    -> client에 관련된 정보들만 입력해 줘도 됨    
  - (Naver, Kakao는 지원해주지 않기 때문에 입력해줘야 하는 입력값들이 구글보다 많음)

  ```
  spring:
    security:
      oauth2:
        client:
          registration:
            google:
              client-id: {client-id}
              client-secret: {client-secret}
              redirect-uri: "http://localhost:8080/login/oauth2/code/google"
              scope: profile, email
  ```
  
  - scope 설정의 기본값 : openid, email, profile     
    - openid 제외한 이유
      - openid라는 scope가 존재하면 OpenId Provider로 인식됨 -> OpenId Provider인 서비스(구글)와 그렇지 않은 서비스(네이버, 카카오 등)로 나누어 OAuth2 Service를 각각 만들어야 함     
      - 따라서 하나의 OAuth2 Service로 사용하기 위해 openid scope을 빼고 등록함    

<br/>

### Social Login 설정

- Oauth library를 이용하여 security 설정 자바 클래스에 social login 설정 코드들 작성

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

	...

	@Bean
	public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
		http
		    ...

		    .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)  //세션을 생성하지 않고, 요청마다 새로운 인증을 수행하도록 구성하는 옵션
		    .and()

		    .oauth2Login() //OAuth 2.0 기반 인증을 처리하기위해 Provider와의 연동 지원
		    .authorizationEndpoint().baseUri("/oauth2/authorization")
		    .and()
		    .userInfoEndpoint()  //OAuth 2.0 provider로부터 사용자 정보를 가져오는 엔드포인트를 지정하는 메서드
		    .userService(customOauthService)  //provider로부터 받은 회원 정보를 가공하는 서비스 지정
		    .and()
		    .successHandler(oauthAuthenticationSuccessHandler)  //OAuth 로그인(인증) 성공 시 호출할 handler 지정(redirect 시킬 목적) 
		    .and()
        
		    ...

		return http.build();
	}
```

<br/>

### provider로부터 받은 회원 정보를 가공할 객체

```java
@Service
public class CustomOauthService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {

	@Override
	public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
		OAuth2UserService<OAuth2UserRequest, OAuth2User> oAuth2UserService = new DefaultOAuth2UserService();
		OAuth2User oAuth2User = oAuth2UserService.loadUser(userRequest);

		String registrationId = userRequest.getClientRegistration().getRegistrationId();
		SocialOauthType socialOauthType = SocialOauthType.valueOf(registrationId.toUpperCase());

		String userNameAttributeName = userRequest
				.getClientRegistration()
				.getProviderDetails()
				.getUserInfoEndpoint()
				.getUserNameAttributeName();

		Oauth2Attributes oauth2Attributes = Oauth2Attributes.of(socialOauthType, oAuth2User.getAttributes());

		return new DefaultOAuth2User(
				Collections.EMPTY_LIST,
				oauth2Attributes.convertToMap(),
				userNameAttributeName
		);
	}
}
```

- registrationId : 현재 로그인 진행 중인 서비스를 구분
  - ex) 구글, 네이버 등

<br/>

### provider에서 제공받는 정보들 관리    
- OAuth2UserService를 통해 가져온 OAuth2User의 attribute를 담을 클래스
- 비즈니스 로직에 따라 필요한 정보들만 추출해서 제공하기 위해 customAttribute를 만들어서 반환
  - 현재는 구글만 구현했지만, 추후 네이버, 카카오도 추가할 예정인데      
    google, naver, kakao 등 provider에서 제공하는 JSON 형식들이 모두 다르기 때문에 따로 관리했다    
    - ex) kakao는 kakao_account에 유저정보가 있고, kakao_account 안에 profile이라는 JSON객체가 있음(nickname, profile_image)

```java
public enum SocialOauthType {
	GOOGLE
}
```

```java
public record Oauth2Attributes(
		String sub,
		SocialOauthType provider,
		String email,
		Map<String, Object> attributes
) {

	public static Oauth2Attributes of(SocialOauthType provider, Map<String, Object> attributes) {
		switch (provider) {
			case GOOGLE -> {
				return ofGoogle(provider, attributes);
			}
			default -> throw new RuntimeException("알 수 없는 소셜 로그인 형식입니다.");
		}
	}

	public Map<String, Object> convertToMap() {
		return Map.of(
				"sub", this.sub,
				"provider", this.provider,
				"email", this.email,
				"id", this.attributes
		);
	}

	private static Oauth2Attributes ofGoogle(SocialOauthType provider, Map<String, Object> attributes) {
		String sub = (String)attributes.get("sub");
		String email = (String)attributes.get("email");
		return new Oauth2Attributes(sub, provider, email, attributes);
	}
}
```

- sub : 구글의 기본 코드, 네이버와 카카오는 기본 지원X
  - OAuth2 로그인 진행 시 키가 되는 필드값(Primary Key와 같은 의미)    

<br/>

### OAuth 로그인(인증) 성공 시 호출되는 handler (redirect 시킬 목적)      

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class OauthAuthenticationSuccessHandler extends SimpleUrlAuthenticationSuccessHandler {

	@Value("${front.server}")
	private String TARGET_URL;

	private final MemberService memberService;

	@Override
	public void onAuthenticationSuccess(
			HttpServletRequest request, HttpServletResponse response, Authentication authentication)
			throws IOException {

		if (authentication.getPrincipal() instanceof OAuth2User oAuth2User) {
			log.info("oAuth2User -> {}", oAuth2User);
			String email = (String)oAuth2User.getAttributes().get("email");
			OauthLoginResponse oauthLoginResponse = memberService.registerOrGet(email);
			String targetUrl = determineTargetUrl(oauthLoginResponse);

			getRedirectStrategy().sendRedirect(request, response, targetUrl);
		}
	}

	private String determineTargetUrl(OauthLoginResponse oauthLoginResponse) {
		return UriComponentsBuilder.fromOriginHeader(TARGET_URL)
				.queryParam("accessToken", oauthLoginResponse.accessToken())
				.queryParam("refreshToken", oauthLoginResponse.refreshToken())
				.build()
				.toUriString();
	}
}
```

<br/><br/>

### 고민했던 점    
- 구글 로그인을 통해 우리 서비스를 이용하려는 유저를 관리하기 위해 memberService에 `registerOrGet(email)` 메서드를 만들었다     
  - 처음에는 OauthAuthenticationSuccessHandler class 안에서 만들었는데 db와 연결까지 한다면 책임이 과하다는 생각이 들었다    
- registerOrGet 메서드는 사용자의 이메일로 유저 정보를 찾아 없다면 유저 정보를 등록하고, 있으면 정보를 조회해오는 메서드인데     
  OAuth2UserService에 넣을지 OauthAuthenticationSuccessHandler에 넣을지 고민을 많이 했고 실제로도 여러번 왔다갔다 했다      
  - 스스로의 결론은 각각의 책임을 분리하고자 OAuth2UserService에서 정보만 가공하고 인증이 성공된 후에 호출되는 handler에서 memberService를 호출해 처리하도록 했다    

<br/>

- 나름의 이슈?    
  처음에 spring security 없이 api를 직접 구현할 때, Authorization Server의 로그인 페이지로 이동하기 위한 인증 URL을 생성하는 것은 프론트엔드, 백엔드 모두 가능했다        
  client-secret이나 scope와 같은 정보들의 응집도를 위해 백엔드에서 생성했다    
  프론트에서 백엔드가 생성한 인증 URL을 가져온 후 사용자를 인증 URL로 리디렉션시키는 플로우로     
  - 이후부터 생각했던 플로우는 사용자 로그인 후의 redirect URI를 프론트로 설정해 프론트에서 authorization code를 받아 백엔드 api를 통해 전달하는 플로우를 생각했는데,      
    프로젝트 시간 상 프론트에서 구현이 어렵다고 하여 백엔드쪽에서 code를 받는걸로 구현을 했다     
    - 현재 프론트와 협업 중이므로 백엔드는 사용자가 브라우저로 직접 접속하기 위해 존재하는 것이 아니니까 좋은 설계가 아니라고 생각한다    
      이로 인해 인증이 완료되면, 백엔드에서 사용자를 다시 프론트엔드 URI로 리다이렉션하는 작업이 필요하다는 점에서 효율적이지 못하다는 생각도 들고, 추후에 리팩토링 해야지(프론트,, 함께 해주시죠ㅎㅎ)        
  - authorization code, client-id, client-secret으로 Authorization Server로부터 access token을 발급받는다     

<br/><br/>

## 참고) OAuth 1.0과 2.0의 차이     
둘 다 보안을 강화하기 위한 프로토콜인데 다른 방식으로 동작하고 호환도 되지 않는다  
OAuth 2.0이 api서버에서 인증서버와 리소스 서버가 분리되고 인증 방식도 다양해서    
더 유연하고 보안성 높은 프로토콜이라는 생각이 들었다    

| | OAuth 1.0 | OAuth 2.0 |
|------|---|---|
| 참여자 | 고객, <br/> 고객이 이용하려는 애플리케이션, <br/> 고객 정보를 가지고 있는 애플리케이션 | 자원 소유자, <br/> 클라이언트, <br/> 권한 서버, <br/> 자원 서버 |
| 토큰 | request token, <br/> access token | access token, <br/> refresh token |
| 유효기간 | access token의 유효기간이 없음 <br/> (토큰 갱신이 어려움, <br/> 토큰을 만료하려면 제공자 애플리케이션의 비밀번호를 바꿔야 함) | access token의 유효기간 부여, <br/> 만료 시 refresh token 이용 |
| 승인 절차 | 두 단계 승인 절차 사용 | 보다 단순한 승인 절차 사용 |
| 클라이언트 | 웹 | 웹, 모바일 기기 등 다양한 플랫폼에서 사용 가능 |

<br/><br/>

Reference     
[Oath Spring](https://docs.spring.io/spring-security/site/docs/5.2.12.RELEASE/reference/html/oauth2.html)    
[Setting OAuth Client properties & Property Mappings](https://docs.spring.io/spring-security/reference/servlet/oauth2/login/core.html)    
[OAuth2ClientProperties](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/autoconfigure/security/oauth2/client/OAuth2ClientProperties.html)    
[Using OAuth 2.0 to Access Google APIs](https://developers.google.com/identity/protocols/oauth2)    
[OAuth 2.0 개념과 동작원리](https://hudi.blog/oauth-2.0/)     

<br/>
