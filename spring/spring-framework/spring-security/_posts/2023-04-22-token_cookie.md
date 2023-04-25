---
title: Spring) JWT 토큰 보안 관리를 위해 프론트와 협업 시 고려한 점
excerpt: Cookie 옵션 설정(httpOnly, samsite, secure, maxAge 등)
---

<br/>

- 현재 프로젝트에서 서버 확장을 고려해 JWT 토큰 인증 방식으로 구현하였기 때문에        
  클라이언트에서는 요청을 보낼때마다 토큰을 보내려면 토큰을 보관하고 있어야 했다    
  보관에는 브라우저 메모리, local storage, cookie를 이용하는 방법들이 있는데    
  - 브라우저 메모리에 저장하면 한번 렌더링될 때마다 다시 로그인 필요        
  - local storage에 보관하는 것은 JS로 접근할 수 있어 보안에 취약    
  - cookie는 일단 모든 HTTP 요청에 포함되어 보내지며 httpOnly나 secure 옵션을 사용하면 JS로 접근 불가
         
  => 따라서 우리는 위의 이유들과 보안 강화를 위해 refresh token 저장에 httpOnly cookie를 사용하기로 했다    
  XSS와 CSRF 공격 모두 막을 수 있도록 서버에서 토큰 발급 시, body에 access token을, httpOnly cookie에 refresh token을 담아 응답하고   
  프론트에서 access token을 private variable로 관리하기로 했다    

<br/>

- 쿠키를 사용해서도 인증을 하는 것이기 때문에 cors 설정 시 withCredentials 설정이 필요하다      
  withCredentials 옵션 이해가 애매했는데 이럴 때 필요하구나    
  
- 이번 기회를 통해 프론트 관련 지식과 XSS 공격, CSRF 등을 접하고 습득할 수 있었다

<br/>

## LocalStorage와 Cookie

### LocalStorage의 장단점   

**장점**    
- CSRF 공격에 안전

<br/>

**단점**     

- XSS에 취약
  - 공격자가 localStorage에 접근하는 JS 코드 한줄만 주입하면 접근 가능하게 됨      

<br/>

### Cookie의 장단점  

cookie : 클라이언트(브라우저) 로컬에 저장되는 키와 값이 들어있는 작은 데이터 파일

<br/>

**장점**     
- httpOnly 적용하면 -> JavaScript로 접근 불가 -> local storage에 비해 XSS 공격에 안전
- 자동으로 모든 HTTP 요청에 포함되어 보내짐

<br/>

**단점**    
- size limit : 4KB
  - 나는 cookie에 refresh token만을 넣을거고 이는 4KB가 넘는 데이터가 아니므로 상관 없었다   
- CSRF 공격에 취약
  - 자동으로 HTTP 요청에 담기기 때문에 -> 공격자가 request url만 알면 사용자 관련 링크를 클릭하도록 유도하여 요청을 위조하기 쉬움  
  - 쿠키가 없으면 CSRF 공격도 없음, 브라우저에 저장되는 쿠키가 CSRF 공격의 매개체     

<br/>

### 참고) XSS와 CSRF   
**XSS(Cross Site Scripting)**   
- 공격자가 의도하는 악의적인 JS 코드를 피해자 web browser에서 실행시키는 것
- 이를 통해 피해자 브라우저에 저장된 중요 정보들 탈취 가능
  - javascrip를 통해 site의 글로벌 변수값을 가져가 사이트인척 할 수 있음     
    = 공격자의 코드가 내 사이트의 로직인척 행동할 수 있음   

<br/>

**CSRF(Cross Site Request Forgery) 사이트간 요청 위조**    
- 유저가 의도하지 않은 요청(공격자가 의도한 행위)을 특정 웹사이트에 하도록 만드는 공격

<br/>

**XSS를 막으면 CSRF도 막히는것 아닌가? :: NO**    
- img 태그 or link로도 사용자 브라우저에서 악의적인 요청을 보낼 수 있음      

<br/><br/>

### 선택한 최종 방법 :: httpOnly Cookie    
 
- 일단, httpOnly 쿠키 사용으로 정했다 따라서 XSS를 막을 수 있다              
- 서버에서 http body에 담아 발급한 access token을 프론트 쪽에서 JS private variable에 저장해두고,     
  httpOnly cookie에는 refresh token만 담고        
  url이 새로고침 될 때마다 refresh token을 통해 access token 다시 발급받기로 했다       
- 이를 통해 refresh token이 CSRF에 사용된다고 하더라도 공격자가 access token은 알 수 없기 때문에 CSRF까지 막을 수 있다    

<br/><br/>

## Cookie 옵션   

- 서버는 Set-Cookie라는 응답 헤더에 브라우저가 수신해야 할 쿠키 정보를 명시함
  - Set-Cookie Header로 쿠키 설정 시, 옵션 설정 가능    
    - 옵션 사이는 세미콜론(;) 사용해 구분, 데이터 사이는 콤마(,)로 사용해 구분
    - ex) `Set-Cookie: <cookie-name>=<cookie-value>; Max-Age=<number>; Secure; HttpOnly`
<br/>

### httpOnly
- javascript로 쿠키를 조회하는 것을 막는 옵션 = 브라우저에서 쿠키를 조회할 수 없음    
  => XSS 공격 차단 가능    
 
<br/>

### secure
- web browser와 web server가 HTTPS로 통신하는 경우에만 web browser가 server로 쿠키를 전송하는 옵션
- httpOnly로 브라우저에서 접근하는 것은 막았지만, 통신과정에서 감청한다면?    
  - 이 경우까지 막기 위해 https protocol 사용해 데이터 암호화

<br/>

### maxAge
- 따로 설정하지 않는다면 `쿠키의 생존주기 = 브라우저의 생존주기`이기 때문에     
  브라우저를 닫으면 저장되었던 쿠키도 사라짐    
  => maxAge 옵션을 통해 브라우저와 별개로 외부파일로 저장하자

<br/>

### samsite
- 쿠키가 같은 도메인에서만 접근할 수 있어야 하는지 여부를 결정하는 옵션
- cookie에서도 cors처럼 origin이 다를경우 각 브라우저마다 정해놓은 보안정책이 존재
  - same-site 정책 : third party cookie에 대한 보안 정책을 어떻게 할 것인지를 결정하는 것   
    - None : cross-site 요청의 경우에도 항상 전송
    - Strict : cross-site 요청의 경우 항상 전송 불가 = first party cookie만 전송
    - Lax : 대체적으로 third party cookie는 전송되지 않지만 예외적인 요청에는 전송(form의 get 메소드 or a 태그의 href 속성)

<br/>

#### 참고) Third-party cookie와 First-party cookie 
- third party cookie : 사용자가 접속한 페이지와 다른 도메인으로 전송하는 쿠키
- first party cookie : 사용자가 접속한 페이지와 같은 도메인으로 전송되는 쿠키
  
<br/>

- 20년 크롬 80 버전이 배포되면서 SameSite의 기본값이 Lax로 변경           
  -> 이에 따라 예외사항에 포함되지 않으면 다른 도메인 간의 요청에서 쿠키를 담아주지 않음   
- 또한, 기본 SameSite 속성으로 None을 사용하려면 반드시 해당 쿠키는 secure 쿠키여야함    

<br/><br/>

## 구현하기    

- 서버에서 cookie 옵션 설정을 하고 데이터를 담아 보내기로 하였다 
- access token 재발급 요청 시, HttpServletRequest에서 cookie를 추출해야하는데 스프링은 역시 이것도 편하게 제공해준다    
  - `@CookieValue`

<br/>

### cookie의 옵션을 설정하고 데이터를 담는 역할    
- CookieProvider.java
  - controller에서 하게 되면 역할이 커지니까, cookie의 옵션을 설정하고 데이터를 담는 역할을 하는 아이를 만들었다   
  - httpOnly 설정을 하고,     
  - 크롬 외 대부분의 브라우저들이 사용하고있는 정책을 준수하기 위해 samsite 전략을 none으로 설정하고 secure 설정을 true로 설정했다  
    - 프론트와 우리 서버는 다른 origin이니까
  - refresh token을 담은 쿠키와 로그아웃 시 유효시간을 0으로 셋팅해 소멸의 역할을 하는 쿠키 만드는 메서드를 각각 만들었다  

```java
@Component
public class CookieProvider {

    private static final String REFRESH_TOKEN = "refreshToken";
    private static final int RESET = 0;
    private static final int EXPIRED = 604800000;

    public ResponseCookie generateTokenCookie(String refreshToken) {
        return generateTokenCookieBuilder(refreshToken)
                .maxAge(Duration.ofMinutes(EXPIRED))
                .build();
    }

    public ResponseCookie generateResetTokenCookie() {
        return generateTokenCookieBuilder("")
                .maxAge(RESET)
                .build();
    }

    private ResponseCookie.ResponseCookieBuilder generateTokenCookieBuilder(String value) {
        return ResponseCookie.from(REFRESH_TOKEN, value)
                .httpOnly(true)
                .secure(true)
                .path("/")
                .sameSite(Cookie.SameSite.NONE.attributeValue());
    }
}
```

<br/>

- Controller
  - 로그인과 리이슈 시, body에 access token을 담고 cookie에 refresh token을 담아 응답했고
  - 로그아웃 시에는 동일한 키값에 빈 value로 유효시간도 0으로 셋팅한 쿠키를 담아 응답하도록 했다  
  - 리이슈 시에는 cookie에 담긴 refresh token을 추출해 검증했다      

```java
@PostMapping("/login")
public ResponseEntity<MemberLoginResponse> loginMember(
    @RequestBody @Valid MemberLoginRequest memberLoginRequest) {

  MemberTokenResponse memberTokenResponse = memberService.loginMember(memberLoginRequest);
  ResponseCookie responseCookie = cookieProvider.generateTokenCookie(memberTokenResponse.refreshToken());
  return ResponseEntity.ok()
      .header(HttpHeaders.SET_COOKIE, responseCookie.toString())
      .body(toMemberLoginResponse(memberTokenResponse));
}

@DeleteMapping("/logout")
public ResponseEntity<Void> logoutMember() {

  memberService.logoutMember();
  ResponseCookie responseCookie = cookieProvider.generateResetTokenCookie();
  return ResponseEntity.ok()
      .header(HttpHeaders.SET_COOKIE, responseCookie.toString())
      .build();
}

@PostMapping("/reissue")
public ResponseEntity<MemberLoginResponse> reissueMember(
    @CookieValue String refreshToken,
    @RequestBody @Valid MemberReissueRequest memberReissueRequest) {

  MemberTokenResponse memberTokenResponse = memberService.reissueMember(memberReissueRequest, refreshToken);
  ResponseCookie responseCookie = cookieProvider.generateTokenCookie(memberTokenResponse.refreshToken());
  return ResponseEntity.ok()
      .header(HttpHeaders.SET_COOKIE, responseCookie.toString())
      .body(toMemberLoginResponse(memberTokenResponse));
}
```

<br/>

### 코드 작성에서 고민했던 점 :: Cookie를 어떻게 만들것인가      

- 처음에는 cookie 객체를 통해 정보를 넣은 후 addCookie 메서드로 넣으려 했으나 samsite 옵션 설정을 할 수 없어 header에 String으로 넣었다   

```java
httpServletResponse.setHeader("Set-Cookie",
                            "refreshToken=" + refreshToken + "; " +
                            "Max-Age=604800000; +
                            ...
                            "HttpOnly; " +
                            "Secure; "
                            );
```

<br/>

- 코드가 지저분한게 너무 마음에 안들어서 돌아가는 것을 확인하고 [ResponseCookie 객체](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/ResponseCookie.html)를 사용하여 리팩토링을 진행했다 
  - 옵션 설정도 내부에서 빌더패턴으로 구현이 되어있다     
- `"Set-Cookie"`도 HttpHeaders에서 정의되어있으니 사용하고 SameSite도 Cookie에 정의되어 있는걸로 사용하고        

```java
return ResponseCookie.from(REFRESH_TOKEN, value)
                .httpOnly(true)
                .secure(true)
                .path("/")
                .sameSite(Cookie.SameSite.NONE.attributeValue());
```

- 음 훨씬 마음에 든다    
  refresh token을 cookie가 아닌 다른곳에 담게 되는 경우도 고려해 코드를 짜고싶었는데    
  이는, header에 cookie를 담는 코드 자체가 변할수 밖에 없게 될것 같아 인터페이스로 분리하지는 않았다   
  이러한 변화에도 유연한 구조를 만들수 있을까?   

<br/><br/>

Reference     

[LocalStorage vs. Cookies: JWT 토큰을 안전하게 저장하기 위해 알아야할 모든것](https://hshine1226.medium.com/localstorage-vs-cookies-jwt-%ED%86%A0%ED%81%B0%EC%9D%84-%EC%95%88%EC%A0%84%ED%95%98%EA%B2%8C-%EC%A0%80%EC%9E%A5%ED%95%98%EA%B8%B0-%EC%9C%84%ED%95%B4-%EC%95%8C%EC%95%84%EC%95%BC%ED%95%A0-%EB%AA%A8%EB%93%A0%EA%B2%83-4fb7fb41327c)       
[Cookie의 옵션](https://pomo0703.tistory.com/207)    
[SameSite=Lax가 Default로? SameSite Cookie에 대해 정확하게 알아보기](https://www.hahwul.com/2020/01/18/samesite-lax/)

<br/>
