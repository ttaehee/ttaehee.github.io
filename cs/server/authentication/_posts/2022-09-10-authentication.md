---
title: CS) Token vs Session Authentication
excerpt: 토큰기반, 세션기반 인증 차이
---

<br/>  

- 인증 확인 증거를 어디에 저장하는가의 차이  
  - 세션 기반 인증은 Server memory or DB에
  - 토큰 기반 인증은 Client 측에 저장  
  
<br/>

## 서버(세션) 기반 인증  

![제목 없음](https://user-images.githubusercontent.com/103614357/189394019-4fa078aa-f439-4b00-b628-4f7cc470817e.png)  

기존의 인증 시스템     
- `서버에서` 사용자 정보 `기억`   
  - 세션 식별 위한 Session Id를 기준으로 세션에 사용자 정보를 저장해두고 서비스 제공할 때 사용 = session 유지 필요(Stateful)  
- 브라우저 쿠키에 Session Id 저장

<br/>

### 세션기반의 문제점  
- **서버 RAM에 부하**
  - session을 대부분 메모리에 저장 & Stateful -> 서버 RAM에 부하   
  - 데이터베이스에 저장을 하기도 함 -> 데이터베이스에 무리  

- **서버확장 어려움**
  - 사용자수 증가(많은 트래픽 처리필요) -> 서버 확장필요(여러 프로세스를 돌리거나 컴퓨터 추가하는 등)       
    -> 처음 로그인 했었던 서버에만 요청을 받도록 설정 or 세션 분산시키는 시스템 설계필요 but 매우 어렵고 복잡  

- **CORS(Cross-Origin Resource Sharing)** 
  - 쿠키는 단일 도메인 및 서브 도메인에서만 작동 -> 여러 도메인에서 쿠키 관리하는 것은 번거로움  

<br/>

=> Token 기반 인증시스템 사용

<br/><br/>

## 토큰 기반 인증  

![제목 없음](https://user-images.githubusercontent.com/103614357/189396142-6a36def0-0746-42d4-867b-d0ed6f7fdbed.png)   

1. `인증받은 사용자`들에게 `토큰 발급`  
2. `클라이언트` 측에서 전달받은 `토큰 저장`  
3. 서버에 요청 할 때마다 `헤더에 해당 토큰`을 함께 전달    
4. 서버에서 토큰 유효성검사 & `인가(허락)`      

<br/>

### 토큰기반의 이점  
- **세선유지 불필요**  
  - 클라이언트 측에서 들어오는 요청만으로 작업처리 (Stateless)   
- 사용자가 로그인이 되어있는지 안되어있는지 신경쓰지 않고 **시스템 확장가능** (Scalability)  
- **CORS 해결**  
  - assests 파일(Image, html, css, js 등)은 모두 CDN에서 제공하고, 서버 측에서는 API만 다루도록 설계가능  

<br/>

Json 포맷을 이용하는 JWT(Json Web Token) 주로 사용  

<br/><br/>
  
## JWT(Json Web Token)  
Json 포맷 이용, 사용자에 대한 속성을 저장하는 Claim 기반의 Web Token      
JSON 데이터(인증에 필요한 정보들)를 Base64URL을 통해 인코딩하여 직렬화한 것   
 
- Self-Contained 방식 : `토큰 자체를 정보로` 사용, 정보를 안전하게 전달   
- 토큰 내부에는 위변조 방지를 위한 개인키를 통한 전자서명도 들어있음 
- 클라이언트 측에서 JWT를 저장하는 공간은 대표적으로 localStorage, Cookie  

<br/>

![제목 없음](https://user-images.githubusercontent.com/103614357/189476499-e143779b-6701-481f-8eb1-8a2127feccf6.png)  


1) application 실행 -> `JWT`가 `static 변수` & `로컬 스토리지`에 저장    
  참고) static 변수에 저장되는 이유   
  HTTP 통신 할 때마다 HTTP 헤더에 JWT 담아 보내는데, 이를 로컬 스토리지에서 계속 불러오면 `오버헤드`가 발생하기 때문   

2) 클라이언트에서 요청(JWT 포함) -> 서버는 `허가된 JWT인지` 검사 (DB 조회 불필요!)   

3) 로그아웃-> 로컬 스토리지에 저장된 `JWT 데이터 제거`   
  (실제 서비스의 경우에는 로그아웃 시, 사용했던 토큰을 blacklist라는 DB 테이블에 넣어 `해당 토큰의 접근을 막는 작업`을 해주어야 함)  

<br/>

### JWT 구조  

![제목 없음](https://user-images.githubusercontent.com/103614357/189477632-bbb4db21-6af0-4781-a710-a39b86f3e2f3.png)    

생성된 토큰  

![제목 없음](https://user-images.githubusercontent.com/103614357/189478134-4379d470-9c55-44fd-aec2-c2b808643aee.png)    

![제목 없음](https://user-images.githubusercontent.com/103614357/189477472-b8ffbce3-81e2-45aa-a93e-d036c90f3996.png)  

- 생성된 토큰은 HTTP 통신을 할 때 `Authorization이라는 key`의 `value`로 사용  
- 일반적으로 value에는 Bearer이 앞에 붙여짐

<br/>

JWT는 Header, Payload, Signature 3 부분으로 이루어짐  
- 각 부분은 `Base64Url로 인코딩` 되어 표현  
  - Base64Url는 암호화된 문자열이 아니고, 같은 문자열에 대해 항상 같은 인코딩 문자열을 반환함
- 각각의 부분을 이어 주기 위해 `.` 구분자를 사용하여 구분 

<br/><br/>

**1. Header(헤더)**   
alg과 typ 두 가지 정보로 구성   

![제목 없음](https://user-images.githubusercontent.com/103614357/189481312-623328ac-d016-4f7e-9600-1e27adb8ab61.png)  

- `alg`: 헤더를 암호화 하는 것이 아니고, `Signature를 해싱`하기 위한 `알고리즘 방식`을 지정  
  Signature 및 토큰 검증에 사용 ex) HS256(SHA256) 또는 RSA
- `typ`: 토큰의 타입을 지정 ex) JWT

<br/>

**2. PayLoad(페이로드)**  
클레임(Claim)이 담겨 있음    

![제목 없음](https://user-images.githubusercontent.com/103614357/189481337-d33c500f-1845-4556-bf33-72ba88cf6f74.png)   

- `클레임`
  - 토큰에서 사용할 정보의 조각들(실제로 사용될 정보에 대한 내용)
  - Json(Key/Value) 형태로 다수의 정보를 넣을 수 있음
  - 총 3가지
    - `Registered Claim(등록된 클레임)` : 토큰 정보를 표현하기 위해 이미 정해진 종류의 데이터들  
      ![제목 없음](https://user-images.githubusercontent.com/103614357/189478471-ee018aa3-b906-4ed0-8fc7-70c521ad29f5.png)     
      (subject로는 unique한 값 사용(주로 사용자 이메일 사용))  
    - `Public Claim(공개 클레임)` : 사용자 정의 클레임, 공개용 정보를 위해 사용 / 충돌 방지를 위해 URI 포맷 이용  
    - `Private Claim(비공개 클레임)` : 사용자 정의 클레임, 서버와 클라이언트 사이에 임의로 지정한 정보를 저장    

<br/>

**3. Signature(서명)**  
토큰을 인코딩하거나 유효성 검증을 할 때 사용하는 `고유한 암호화 코드` -> 토큰의 `위변조 여부 확인`하는데 사용 

![제목 없음](https://user-images.githubusercontent.com/103614357/189481385-12e2fc5e-fd9d-404d-83bc-292d56502496.png)  

- 서명생성  
  - 1. `헤더`와 `페이로드`의 값을 각각 `BASE64Url로 인코딩`  
  - 2. 인코딩한 값을 `비밀 키`를 이용해 헤더에서 정의한 알고리즘으로 `해싱`  
  - 3. 이 값을 `다시 BASE64Url로 인코딩`하여 생성 

<br/>

참고) Header와 Payload는 단순히 인코딩된 값 -> 제 3자가 복호화 및 조작가능  
but Signature는 비밀키(서버측에서 관리하는)가 유출되지 않는 이상 복호화할 수 없음 
  
<br/><br/>

### JWT 단점 및 고려사항  
- Self-contained  
  - 토큰 자체에 정보를 담고 있으므로 양날의 검이 될 수 있음  

- 토큰 길이
  - 토큰의 페이로드에 3종류의 클레임 저장 -> 정보가 많아질수록 토큰의 길이증가 -> `네트워크에 부하`  

- Payload 인코딩  
  - 페이로드 자체는 암호화 된 것이 아니라, BASE64Url로 인코딩 된 것  
  -> 중간에 Payload를 탈취하여 디코딩하면 데이터를 볼 수 있음 -> JWE로 암호화하거나 Payload에 중요 데이터를 넣지 않아야함  
  - 참고) 토큰의 진짜 목적은 정보보호가 아닌, `위조방지!`

- Stateless  
  - 한번 발급된 토큰은 임의삭제 불가능 -> `토큰 만료 시간`을 꼭 넣어주어야 함  
  (토큰이 해커에게 탈취되었다면, 해커는 토큰이 만료될 때까지 계속 공격가능)

- Store Token  
  - 토큰은 클라이언트 측에서 관리해야 하기 때문에, 토큰 저장 필수  

<br/><br/>

Reference  
https://velog.io/@arthur/%EC%84%B8%EC%85%98-%EC%9D%B8%EC%A6%9D-%EB%B0%A9%EC%8B%9D-vs-%ED%86%A0%ED%81%B0-%EC%9D%B8%EC%A6%9D-%EB%B0%A9%EC%8B%9D   
https://mangkyu.tistory.com/55?category=925341   
https://mangkyu.tistory.com/56   
https://inpa.tistory.com/entry/WEB-%F0%9F%93%9A-JWTjson-web-token-%EB%9E%80-%F0%9F%92%AF-%EC%A0%95%EB%A6%AC  
<br/>
