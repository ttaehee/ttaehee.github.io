---
title: Spring) filter에서 발생하는 예외가 @ControllerAdvice로 잡히지 않은 이유
excerpt: Spring Security Filter 내에서 발생하는 예외 처리하기
---

<br/> 

- filter단에서 토큰의 유효성을 검증하면서 예외를 던져야했다        
  평소처럼 @ControllerAdvice와 @ExceptionHandler로 잡았다가 아차 싶었다     
- Spring Security는 filter 기반으로 동작하므로           
  Spring MVC와 분리되어 있기 때문에 핸들링되지 않는다     
- filter에서 핸들링 되도록 custom filter를 만들어 처리했다   

<br/>  

## @ControllerAdvice와 예외 처리

### @ControllerAdvice의 적용범위  
기본적으로 모든 Controller에게 적용되는 Advice     

- 요청이 DispatcherServlet에 의해 처리되는 경우 작동    
- 그 전에 발생하는 Filter단에서 발생한 예외 핸들링은 못함

![security](https://user-images.githubusercontent.com/103614357/222224318-6fec39bb-308d-4e74-b837-4621c2f578bf.png)

- 초록색 동그라미가 @ControllerAdvice의 적용범위
  = filter에서 발생한 예외는 @ControllerAdvice의 적용범위 밖   
  
=> 따라서 filter내에서 발생한 인증, 인가 및 토큰 관련 예외가 핸들링 되지 않았다   
 
<br/>   

=> filter에서 발생하는 예외를 filter단에서 핸들링하기 위해       
예외 발생이 예상되는 filter 상위에 예외 핸들링하는 filter 만들어 filter chain에 추가

<br/><br/>

## Spring Security Filter에서 발생하는 예외 및 처리      


### 인증 관련 예외 처리 (AuthenticationEntryPoint)  

![security](https://user-images.githubusercontent.com/103614357/222238461-888ea5b8-434b-4b8b-9acb-d19c841296af.png)   

- 인증 안된 익명 사용자가 인증이 필요한 엔드포인트로 접근하면?  
  - Spring Security의 기본 설정 : HttpStatus 401 + spring 의 기본 오류페이지 보여줌   

<br/>  

- AuthenticationException :인증 과정과 관련한 예외
  - 인증이 실패한 상황(401)
  - 예외 처리 위해 AuthenticationEntryPoint 인터페이스 구현

```java
@Slf4j
@RequiredArgsConstructor
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {

	private final ObjectMapper objectMapper;

	@Override
	public void commence(
		HttpServletRequest request, HttpServletResponse response, AuthenticationException authException)
		throws IOException {

		log.info("Not Authenticated Request", authException);
		ExceptionResponse exceptionResponse = new ExceptionResponse("로그인 후 사용가능합니다.");

		String responseBody = objectMapper.writeValueAsString(exceptionResponse);
		response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
		response.setContentType(MediaType.APPLICATION_JSON_VALUE);
		response.setCharacterEncoding("UTF-8");
		response.getWriter().write(responseBody);
	}
}
```

- 직접 Response 생성해서 클라이언트에게 응답  
- objectMapper 사용해 ExceptionResponse 객체를 바디 값으로 파싱

<br/><br/>

## 인가 관련 예외 처리 (AccessDeniedHandler)   

- AccessDeniedException : 액세스 권한이 없는 리소스에 접근할 경우 발생하는 예외
  - 인증은 되었지만 인가는 실패한 상황(403) 
  - 예외 처리 위해 AccessDeniedHandler 인터페이스 구현   

```java
@Slf4j
@RequiredArgsConstructor
public class CustomAccessDeniedHandler implements AccessDeniedHandler {

	private final ObjectMapper objectMapper;

	@Override
	public void handle(
		HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException)
		throws IOException {

		log.info("No Authorities", accessDeniedException);
		ExceptionResponse exceptionResponse = new ExceptionResponse("없는 리소스입니다.");

		String responseBody = objectMapper.writeValueAsString(exceptionResponse);
		response.setStatus(HttpServletResponse.SC_FORBIDDEN);
		response.setContentType(MediaType.APPLICATION_JSON_VALUE);
		response.setCharacterEncoding("UTF-8");
		response.getWriter().write(responseBody);
	}
}
```

<br/><br/>

### Http Status Code 401 & 403

- 401:  인증되지 않은 요청 의미
- 403 : 인증은 되었지만 요청에 대한 권한이 부족하다는 것 의미

<br/><br/>
 

## 토큰 관련 예외 처리    

- 위의 두 가지는 AccessDeniedException과 AuthenticationException만 처리함     
- 이를 상속하지 않는 예외(내가 만든 토큰 관련 예외들) 처리 위해서 만든 필터     

<br/>

### 토큰 관련 예외를 왜 만들었는가
일반적인 401 Unauthorized 예외와 JWT 만료 예외 발생을 구분하기 위해서 구현했다   
- 일반적인 401 예외 발생 -> 인증 권한을 얻기 위해서 재로그인 시킴 
- access token 만료 예외 발생 -> 클라이언트에게 refresh token 검증이 필요하다고 알려주며 access token 재발급하는 API를 호출하도록 해야함

=> JWT Exception만 담당으로 처리할 수 있는 필터를 JWT 인증 필터 앞에 붙이자

<br/><br/>   

### 구현하기

- JwtTokenProvider에서 토큰관련 예외를 잡아 TokenException을 던지도록 했다
  - custom exception은 세부처리(예외 메시지)는 다르게 하지만 custom filter에서 같이 처리해주고자 계층구조로 구현했다
  - TokenException (extends RuntimeException)
    - ExpiredTokenException (extends TokenException)
    - InvalidTokenException (extends TokenException)

- 지난번과 JwtTokenProvider 구현을 조금 바꾸었다  
  boolean 반환이었는데, 처리해야하는 예외 발생시에 예외를 던지도록 하면서 불필요한 반환값이라고 생각했다  

```java
public void validateToken(String accessToken) {
    try {
      Jwts.parserBuilder()
        .setSigningKey(secretKey)
        .build()
        .parseClaimsJws(accessToken);
    } catch (ExpiredJwtException exception) {
        log.info("Expired JWT Token", exception);
        throw new ExpiredTokenException("만료된 토큰입니다.");
    } catch (JwtException | IllegalArgumentException exception) {
        log.info("Invalid JWT Token.", exception);
        throw new InvalidTokenException("올바르지 않은 토큰입니다.");
    }
	}
```

<br/>

- JwtTokenProvider에서 발생한 예외 처리하는 custom filter  

```java
@Slf4j
@RequiredArgsConstructor
public class JwtExceptionFilter extends OncePerRequestFilter {

  private final ObjectMapper objectMapper;

  @Override
  protected void doFilterInternal(
    HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
    throws ServletException, IOException {

    try {
        filterChain.doFilter(request, response);
    } catch (TokenException tokenException) {

        ExceptionResponse exceptionResponse = new ExceptionResponse(tokenException.getMessage());

        String responseBody = objectMapper.writeValueAsString(exceptionResponse);
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write(responseBody);
    }
  }
}
```

<br/><br/>

### 토큰 만료 상태 코드 고민

토큰 만료 상태 코드는 무엇을 사용할까 고민하다가 401 code를 선택했다       
이유는 결국 `토큰이 만료되었다 = 인증되지 않은 요청이다`와 같다고 생각했기 때문이다    

<br/><br/>

### filter chain에 custom jwt exception filter 추가하기   

- 만든 필터를 토큰 관련 예외가 발생되는 JwtAuthenticationFilter 상위에 추가했다  

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        ...

        .addFilterBefore(new JwtAuthenticationFilter(jwtTokenProvider()),UsernamePasswordAuthenticationFilter.class)
        .addFilterBefore(new JwtExceptionFilter(objectMapper), JwtAuthenticationFilter.class);

    return http.build();
}
```  

![jwt_exception](https://user-images.githubusercontent.com/103614357/222977128-9b23b814-643d-4548-8795-8d8ad8ebeab5.png)   

<br/><br/>

### Filter와 OncePerRequestFilter    
**Filter**    
- Dispatcher Servlet에 요청이 전달되기 전, 후에 서블릿을 거쳐서 필터링함  

<br/>

**GenericFilterBean**    
- 스프링에서 제공하는 필터
- Filter 확장
  - Spring의 설정 정보도 가져올 수 있음(Filter에서 얻어올 수 없는 정보)

<br/>

- 둘 다 매 서블릿 마다 호출됨   
  - 참고) 서블릿은 클라이언트 요청 받으면 서블릿 생성해서 메모리에 저장
    - 같은 클라이언트의 요청 받으면 생성해둔 서블릿 객체 재활용해서 요청 처리 
  - 서블릿이 다른 서블릿으로 dispatch되는 경우
    - ex) Spring Security에서 인증과 접근 제어 기능이 Filter로 구현되어, 인증과 접근 제어는 RequestDispatcher에 의해 다른 서블릿으로 dispatch 됨       

<br/>

위의 예시의 경우       
-> 이 때 이동할 서블릿에 도착 전, 또 filter chain을 거치게 됨 (Filter나 GenericFilterBean으로 구현된 filter 모두)       
= 필터가 매번 실행하게 됨

<br/>

**OncePerRequestFilter** 

```
Filter base class that aims to guarantee a single execution per request dispatch, on any servlet container.   
It provides a doFilterInternal method with HttpServletRequest and HttpServletResponse arguments.  
```

- 어느 서블릿 컨테이너에서나 요청 당 한 번의 실행을 보장함    
  => 이를 구현한 필터는 사용자 요청 당 한번만 실행되는 필터를 만들 수 있음     
  - 동일한 request안에서 한번만 필터링을 할 수 있게 해줌
  - 인증, 인가와 같이 한번만 거쳐도 되는 로직에 적합   
    - ex) 인증 or 인가 후 특정 url로 포워딩하면 인증 및 인가필터를 다시 실행시켜야 하지만   
      OncePerRequestFilter를 사용함으로써 인증 or 인가 다시 안거치고 다음 로직 진행 가능    

<br/><br/>

Reference    
[Spring Security Architecture](https://spring.io/guides/topicals/spring-security-architecture/)    
[Exception Handling In Spring Security](https://www.devglan.com/spring-security/exception-handling-in-spring-security)     
[spring security 파헤치기](https://sjh836.tistory.com/165)
[What is OncePerRequestFilter?](https://stackoverflow.com/questions/13152946/what-is-onceperrequestfilter)

<br/>
