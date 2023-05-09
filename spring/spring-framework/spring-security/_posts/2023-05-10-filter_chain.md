---
title: Spring) Spring Security에서 Custom Filter를 bean 등록해서 발생했던 문제와 해결    
excerpt: custom filter가 추가되는 과정과 DelegatingFilterProxy, FilterChainProxy 
---

<br/>

- 이번 프로젝트에서 Spring Security를 사용해서 사용자의 **인증, 인가**를 체크하였기 때문에 **filter 단에서 보안처리**가 이루어졌다    
  프로젝트를 진행하면서 **filter chain에 custom filter 추가를 잘못해서**    
  **security filter ignoring 처리에 어려움**이 있었는데 그 이유도 같이 정리해보려 한다   
  - 지난번에 정리한) [filter와 interceptor](https://ttaehee.github.io/spring/spring-framework/spring-basics/filter_interceptor/)

<br/><br/>

## Spring Security에서 Custom Filter를 bean 등록해서 발생했던 문제

filter는 spring 범위 밖에서 동작하지만 빈 등록은 가능하다    
filter chain에 jwt를 추출해 유효성을 체크하는 **custom filter를 추가**했는데     
처음에는 new로 객체를 생성해서 추가했다가 리팩토링을 하면서 **빈등록을 해서 추가**로 바꾸었다     
그래서 발생했던 문제       

<br/>

### 상황 :: security filter chain 생략 필요
토큰 재발급 요청 시에는, 이미 access token이 만료되었을테니 security filter chain 생략이 필요했다    

<br/>

### 이슈 :: web.ignoring()이 동작하지 않음
`web.ignoring().antMatchers(..)`는 파라미터로 전달하는 패턴에 대해 security filter chain을 생략하기 때문에     
이를 사용해 **토큰 재발급 요청을 security filter ignoring 되도록** 설정했음에도    
**직접 만든 custom filter를 제외하지 않아** 계속해서 header의 만료된 access token을 검증하면서 이슈가 발생했었다  

<br/>

### 원인 :: custom filter를 bean으로 등록함
custom filter를 bean으로 등록하게 되면, 해당 filter가 **security filter chain에 포함되는 게 아니라**          
**default filter chain에 포함**되게 되기 때문에 해당 패턴에 접근하게 됐을 때 그대로 filter를 적용하게 된다             
그래서 잘 돌아가던 토큰 갱신까지 문제가 생겼고, filter chain 디버깅과 구글링을 열심히 하면서 찾아내었다   

<br/>

### 해결 :: custom filter를 new로 생성해 filter chain에 추가함
[참고했던) Spring webSecurity.ignoring() doesn’t ignore custom filter](https://stackoverflow.com/questions/39152803/spring-websecurity-ignoring-doesnt-ignore-custom-filter/40969780#40969780)

```java
.addFilterBefore(new JwtAuthenticationFilter(jwtTokenProvider()), UsernamePasswordAuthenticationFilter.class)
```

<br/>

또 다른 해결법으로는 **OncePerRequestFilter의 shouldNotFilter() 메서드를 재정의**해서 custom filter에서 해당 요청을 무시하도록 할 수도 있다     
내가 원하는 방향으로 동작은 하지만 근본적인 해결방안은 아니라고 생각한다    

```java
@Override
protected boolean shouldNotFilter(HttpServletRequest request) {
  return request.getRequestURI().endsWith("reissue") && request.getMethod().equalsIgnoreCase("POST");
}
```

<br/>

이번 이슈를 정리하면서 **custom filter가 추가되는 과정**과 더불어 **DelegatingFilterProxy와 FilterChainProxy**에 대해서도 같이 정리해보려 한다       

<br/><br/>    

## DelegatingFilterProxy   

Spring Security의 기능은 Servlet Filter를 기반으로 구현되어 있다     
Filter는 Java EE 스펙이고 이는 servlet container 영역에서 사용하는 객체이다       

<br/>

따라서 filter는 당연히 spring application context 외부의 객체이므로 **spring bean을 사용할 수 없다**  
이를 해결하기 위해 spring-web module에서는 `DelegatingFilterProxy`라는 Filter 구현체를 제공한다

<br/>

- DelegatingFilterProxy : spring에서 관리하는 filter(servlet filter를 구현한 spring bean)에게 요청을 위임하는 역할을 하는 servlet filter

<br/>

해당 객체가 **servlet container와 spring application context를 연결**해주는 역할을 함으로써 spring bean을 사용하는 filter를 구현할 수 있게 해준다      
servlet filter는 DelegatingFilterProxy 클래스를 사용해서 요청을 위임할 springSecurityFilterChain라는 이름을 가진 spring bean을 찾는데 그게 FilterChainProxy다     

<br/><br/>

## FilterChainProxy      

FilterChainProxy는 위에서도 말했지만, springSecurityFilterChain의 이름으로 생성되는 filter bean이다     
ApplicationContext가 초기화 될 때, springSecurityFilterChain 이름으로,    
설정된 security filter들을 가진 FilterChainProxy를 등록한다     

<br/>

**하는 일**    
- spring security 초기화 시 생성되는 필터들을 관리하고 제어
- DelegatingFilterProxy로부터 요청을 위임 받고 각 필터들을 순서대로 호출하며 실제 보안(인증/인가) 처리   

<br/>

FilterChainProxy는 내부에 SecurityFilterChain을 구현한 필터들을 리스트로 가지고 있는데,      
FilterChainProxy가 doFilter 동작을 위임받으면 다시 내부의 SecurityFilterChain 리스트들에게 필터링 동작을 위임하게 된다     

FilterChainProxy가 spring security에서 제공하는 서블릿 필터링 기능의 시작점인 것이다   

<br/>

**Spring Security 기본 설정 시, FilterChainProxy에 기본으로 추가되는 필터들**    

<img width="345" alt="스크린샷 2023-05-03 오후 5 25 03" src="https://user-images.githubusercontent.com/103614357/235866132-53e96d4a-a3d5-4e53-85a8-4aa70ae6ab9d.png">

**사용자 정의 필터**를 생성해서 **기존의 필터 전후로 추가**가 가능하며, 마지막 filter까지 인증 및 인가 예외가 발생하지 않으면 보안이 통과된 것이다    
아래는 몇몇 필터는 제외시키고 내가 만든 필터들을 추가한 후 디버깅해본 것    

<img width="356" alt="스크린샷 2023-05-03 오후 7 13 45" src="https://user-images.githubusercontent.com/103614357/235889934-b3801d15-49c4-4a09-a9bf-aed3cd19ddc0.png">

<br/><br/>

## Flow     

<img width="779" alt="스크린샷 2023-05-03 오후 5 22 46" src="https://user-images.githubusercontent.com/103614357/235865603-87301193-755c-41df-a3e2-5803a8c7203b.png">

**1.** 사용자가 자원 요청을 하면        
**2.** Servlet Container의 filter들이 처리를 한다     

<img width="546" alt="스크린샷 2023-05-03 오후 5 26 27" src="https://user-images.githubusercontent.com/103614357/235866549-f425b2d3-2fde-4143-afd0-b62688375bb7.png">

**3.** 그 중 **DelegatingFilterProxy가 요청을 받게 될 경우** 자신이 요청받은 요청 객체를    
**요청을 위임할 필터(springSecurityFilterChain)를 찾아**서 해당 요청을 위임(`invokeDelegate()`)한다     

<img width="546" alt="스크린샷 2023-05-03 오후 5 26 12" src="https://user-images.githubusercontent.com/103614357/235866609-d4db3ae2-e37b-4e7f-83c4-2e63152311a0.png">

**4.** invokeDelegate() 함수 호출 시, 내부에서 `FilterChainProxy의 doFilter()` 함수를 호출하고, doFilterInternal() 함수에서는 등록 된 **Filter 목록을 순서대로 수행**하며 인증/인가 처리를 진행한다    
**5.** 보안 처리가 완료되면 최종 자원에 요청을 전달하여 다음 로직이 수행된다    

<br/><br/>

## 교훈

무조건 bean 등록부터 하고보지 말자      

<br/><br/>

Reference    

[docs.spring.io) Spring Security Architecture](https://docs.spring.io/spring-security/reference/servlet/architecture.html)     
[스프링 시큐리티 주요 아키텍처 이해](https://catsbi.oopy.io/f9b0d83c-4775-47da-9c81-2261851fe0d0)    
[스프링 시큐리티 - Filter](https://realrain.net/posts/spring-security-filter)    

<br/>
