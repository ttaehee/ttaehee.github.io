---
title: CS) Authentication Method (Cookie & Session vs JWT)
excerpt: 인증방식 (쿠키 & 세션 vs JWT)   
---

<br/>  

- 토큰기반, 세션기반 인증 차이
  - 인증 확인 증거를 어디에 저장하는가의 차이  
    - 세션 기반 인증은 Server memory or DB에
    - 토큰 기반 인증은 Client 측에 저장  
  
<br/>

## HTTP 특성   

- 비연결성
- 무상태성 

=> 요청에 대한 응답을 처리하게 되면 `연결을 끊어버림`    
=> Client 상태 정보, 통신의 상태가 `남지 않음`     
=> Client를 식별해야 하는 경우, Server가 Client가 누구인지 `매번 확인`해야함      

<br/>

=> `Cookie와 Session` : HTTP의 비연결성 및 무상태성 특징 보완한 기술    

<br/><br/>

## Cookie   
client가 web site 방문 시,      
그 ste가 사용하고 있는 server를 통해 client의 `browser에 설치`되는 작은 `기록 정보 파일`    

<br/>

### 동작과정  

**1. Client -> Server : Http 첫 요청**     
   
**2. Server : cookie 생성해서 발급**     
- cookie 생성    

``` 
Cookie taeheeCookie = new Cookie("cookieName", cookieValue);     
taeheeCookie.setMaxAge(600);
```

- cookie 발급 : response header의 Set-Cookie 속성에 클라이언트측에 저장하고 싶은 정보를 key-value 형태로 담음 
     
```
Headers = [Set-Cookie:"userName=taehee", "password=taehee1234"]
```  

**3. Client : cookie가 client의 browser에 저장됨**    

**4. Client -> Server : 요청 시 cookie 같이 보냄**     
- 요청 시 browser가 자동으로 request header에 cookie 넣어서 전송  

<br/>

- 참고) cookie 제거

```
Cookie taeheeCookie = new Cookie("cookieName", null);     
taeheeCookie.setMaxAge(0);
```

<br/>

### 사용
- 쇼핑몰 장바구니
- 팝업(오늘 더이상 이 창을 보지 않음 버튼)  

<br/><br/>  

### 장단점   

**장점**   
- 대부분의 브라우저가 지원

**단점**  
- 암호화 안되어있음, `보안에 취약`
  - 요청 시 cookie의 값을 그대로 보냄 -> 유출, 조작 당할 위험   
- 용량제한(4kb)   
- web browser마다 cookie에 대한 지원 형태가 다름 -> 브라우저간 공유 불가능

<br/>
  
=> cookie의 `개인정보가 유출`될 수 있다는 것은 큰문제!             
(개인정보를 HTTP로 주고 받는 것은 위험)    
 
=> Session과 같이 쓰자    

<br/><br/>

## Cookie & Session 기반 인증   

- Session : 비밀번호 등 client의 `인증 정보`를 cookie가 아닌 `서버 측에` 저장하고 관리   
  - 서버에 저장되는 쿠키 느낌  
  - cookie와 큰 차이점은 `인증정보의 저장위치`        

<br/>

### 동작과정

**1. Client -> Server : Http 첫 요청**       
**2. Server : sessionId 생성 후 저장**   
- response의 cookie 속성에 sessionId 같이 전송 

```
HTTP/1.1 200
Set-Cookie: JSESSIONID=FDB5E30BF20045E8A9AAFC788383680C;
```

**3. Client : cookie가 client의 browser에 저장됨**  

**4. Client -> Server : 요청 시 cookie에 sessionId 같이 보냄**   

**5. Server : request header에 cookie(JSESSIONID) 있는지 확인**   
-  있다면 저장된 sessionId와 비교 후, 일치한다면 수행 후 응답   

<br/><br/>

### Cookie와 Session의 차이  

||쿠키(Cookie)|세션(Session)|
|:---:|---|---|
|저장 위치|클라이언트(=접속자 PC)|웹 서버|
|저장 형식|text|Object|
|만료 시점|쿠키 저장시 설정<br/>(브라우저가 종료되도, 만료시점이 지나지 않으면 자동삭제되지 않음)|브라우저 종료시 삭제<br/>(기간 지정 가능)|
|사용하는 자원(리소스)|클라이언트 리소스|웹 서버 리소스|
|용량 제한|총 300개<br/>하나의 도메인 당 20개<br/>하나의 쿠키 당 4KB(=4096byte)|서버가 허용하는 한 용량제한 없음|
|속도|세션보다 빠름|쿠키보다 느림|
|보안|세션보다 안좋음|쿠키보다 좋음|

<br/><br/>   

### Session의 장단점   
**장점**    
- cookie 보다 보안 좋음
  - cookie에 sessionId만 저장 -> 외부에 노출되더라도 유의미한 개인정보 안담겨있음  
- borwser 종료 시 삭제됨(만료시간과 상관없이)  
  
<br/>  

**단점**   
- 보안은 좋아졌지만 여전히, 탈취 후 `클라이언트인척 위장` 할 수 있음   
- 속도는 쿠키보다 `느림`   
- 서버에서 session 저장 -> `서버 RAM에 부하`   
  - session을 대부분 메모리 or DB에 저장  
- `서버확장` 어려움(세션클러스터라는 별도의 component 필요)
  - 사용자수 증가(많은 트래픽 처리필요)     
    -> 서버 확장 필요(여러 프로세스를 돌리거나 컴퓨터 추가하는 등)    
    -> 확장하게 되면, 처음 로그인 했었던 서버에만 요청을 받도록 설정 or 세션 분산시키는 시스템 설계필요    
    but 매우 어렵고 복잡     
    
<br/>

- 그러면 서버 수평 확장을 대비해서 session을 안쓰면?      
  - Http Session을 사용하지 않는다 = 서비스가 stateless 하다   
  - 서버가 아무런 상태도 안가짐 => 수평확장이 쉬움 
  - 그렇지만, 우리가 만드는 서비스는 사용자식별 필요해!   
   
=> 해결 : JWT 사용!

<br/><br/>  

## JWT  
Json 포맷 이용   
사용자에 대한 속성을 저장하는 Claim 기반의 Web Token       
JSON 데이터(인증에 필요한 정보들)를 Base64URL을 통해 인코딩하여 직렬화한 것   

<br/>
   
```
Authorization: [type][access token]
```

- 일반적으로 토큰은 request header의 Authorization 필드에 담아 보냄  
- 토큰에는 많은 종류가 있고, 서버는 다양한 종류의 토큰을 처리하기 위해 전송받은 type에 따라 토큰을 다르게 처리   
  - `ex) Authorization: "Bearer {생성된 토큰값}"`  
  - Basic : 사용자 아이디와 암호를 Base64로 인코딩한 값을 토큰
  - Bearer : JWT 혹은 OAuth에 대한 토큰 사용

<br/><br/>

### JWT를 이용한 클라이언트 인증  
1. `client -> server` : 인증처리요청(로그인)
2. `server -> client` : 처리결과에 JWT값 포함해서 응답
3. `client` : 받은 JWT를 static 변수 & 로컬 스토리지에 저장해두고 모든 api 요청을 할 때마다 Http Header에 JWT 포함 (URL에대해 안전한 문자열로 구성되어 있기 때문에 어디든 포함 가능하지만, 주로 헤더에 함)
4. `server` : JWT 추출해서 해당 JWT 통해 어떤 클라이언트인지 식별 후 api 기능 수행
   1) 서버는 허가된 JWT인지만 검사 (DB 조회 불필요!)

<br/>

- 참고) static 변수에 저장 이유
  - HTTP 통신 할 때마다 HTTP 헤더에 JWT 담아 보내는데, 이를 로컬 스토리지에서 계속 불러오면 오버헤드가 발생하기 때문

<br/><br/>   

### SessionId와 차이점   
- sessionId : 서버에서 단지 세션식별위해서만 만든 아무 의미없는 값(랜덤 생성)
- JWT : Json포맷 사용해서 어떤 데이터를 표현하는 값

<br/><br/>   

### JWT 구조  
JWT는 `Header, Payload, Signature` 3 부분으로 이루어짐       

**Header**     

```
{
  "alg": "HS256",
  "type": "JWT"
}
```

- `alg` : 헤더를 암호화 하는 것이 아니고, `Signature를 해싱하는 알고리즘` 정보  
  - Signature 및 토큰 검증에 사용   
  - ex) HS256(SHA256) 또는 RSA
- `typ` : 토큰의 타입을 지정   
  - ex) JWT 
    - type이 JWT이다 라는것

<br/>

**Payload**   
- `Claim`이 담겨있음  
  - payload에 담겨진 `정보의 한 조각("name": "value")`을 claim이라고 함   
  - JWT통해서 실질적으로 전달하고자 하는 데이터    
  - Json(Key/Value) 형태로 다수의 정보 넣을 수 있음   
- JWT 자체가 암호화되는게 아니기 때문에(단지 `Base64로 인코딩` 되었을 뿐)  
  연락처, 비밀번호 등의 민감정보 포함되면 안됨   
  
```
{
  "sub": "ACCESS_TOKEN",
  "USER_EMAIL": "taehee@test.com",
  "AUTHORITIES": "ROLE_USER",
  "iat": "1644573704",
  "exp": "1644577304"
}
```

- sub : 토큰 제목
- iss : 토큰 발급자
- iat : 발급 시각
- exp : 만료 시각
- 사용자의 정보(권한, 이메일 등)도 포함 가능 -> custom claims  

<br/>

**Signature**   
- 토큰의 `위변조 검증`을 위한 데이터   
- pseudo-code 슈도코드로 나타냄  

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

- 서명생성
  - 헤더와 페이로드의 값을 각각 `BASE64Url로 인코딩`
  - 인코딩한 값을 서버만 알고있는 `비밀 키(맨마지막 줄 secret)`를 이용해 헤더에 정의한 알고리즘으로 `서명값 생성(해싱)`     
  - 이 값을 `다시 BASE64Url로 인코딩`하여 생성    

<br/>

- 참고)
  - Header와 Payload는 단순히 인코딩된 값 -> 제 3자가 복호화 및 조작가능
  - but Signature는 비밀키(서버측에서 관리하는)가 유출되지 않는 이상 복호화할 수 없음     
    = 비밀키 없으면 서명데이터 올바르게 생성할 수 없음   

<br/><br/>

### JWT의 장단점
**장점**    
- 스토리지 필요없음
- 수평확장 쉬움
- active user 많으면 유리(session 이용 시 user수만큼 세션저장 - 스토리지 관리 어려움)

<br/>

**단점**   
- 모든 Http 요청에 JWT 포함해야하니까 JWT 크기 `최대한 작게`해야함   
- 한번 발급된 토큰은 `임의삭제 불가능` -> 보안 면에서 분리  
  - 외부에 유출된 경우여도, 만료기간 남은 JWT의 `만료처리 어려움`      
    (토큰이 해커에게 탈취되었다면, 해커는 토큰이 만료될 때까지 계속 공격가능)   
  -> 토큰 만료기간 꼭 넣어주어야 함 + 짧게 설정해야함    
  
- Payload는 암호화가 아닌 인코딩
  - Payload 자체는 암호화 된 것이 아니라, BASE64Url로 인코딩 된 것    
    -> 중간에 Payload를 탈취하여 디코딩하면 데이터를 볼 수 있음    
    -> `중요 데이터를 넣지 말 것`    
  - 참고) 토큰의 진짜 목적은 정보보호가 아닌, `위조방지!`   

- 토큰 길이 -> 요청이 많아질수록 서버 자원의 낭비 많아짐  

<br/><br/>

Reference   
https://hahahoho5915.tistory.com/32     
https://tecoble.techcourse.co.kr/post/2021-05-22-cookie-session-jwt/      
https://mangkyu.tistory.com/56      
https://inpa.tistory.com/entry/WEB-%F0%9F%93%9A-JWTjson-web-token-%EB%9E%80-%F0%9F%92%AF-%EC%A0%95%EB%A6%AC        
https://www.youtube.com/watch?v=XXseiON9CV0&ab_channel=%EC%BD%94%EB%94%A9%EC%95%A0%ED%94%8C      
https://velog.io/@cada/%ED%86%A0%EA%B7%BC-%EA%B8%B0%EB%B0%98-%EC%9D%B8%EC%A6%9D%EC%97%90%EC%84%9C-bearer%EB%8A%94-%EB%AC%B4%EC%97%87%EC%9D%BC%EA%B9%8C   

<br/>   
