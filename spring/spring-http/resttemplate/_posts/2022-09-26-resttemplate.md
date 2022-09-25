---
title: Spring) RestTemplate
excerpt: spring에서 http 통신
---

## Spring에서 REST 서비스 호출방법
- `RestTemplate` : REST API 호출 이후 응답을 받을 때까지 기다리는 동기 방식 (Spring 3부터 지원)  
- `AsyncRestTemplate` : 비동기 RestTemplate (Spring 4에 추가)  
- `WebClient` : 리액티브 웹 클라이언트로 동기, 비동기 방식을 지원 (Spring 5에 추가)  

=> Spring Framework 5부터 WebClient 라는 새로운 HTTP 클라이언트 도입    
WebClient 는 RestTemplate에 대한 최신 대체 HTTP 클라이언트    
기존의 동기식 API를 제공 할뿐 아니라 효율적인 비차단 및 비동기 접근 방식도 지원   

즉, 새 애플리케이션을 개발하거나 이전 애플리케이션을 마이그레이션하는 경우 WebClient를 사용하는 것이 좋음  

<br/>

## RestTemplate  
Spring에서 제공하는 http 통신에 사용할 수 있는 템플릿  
- HTTP 서버와의 통신을 단순화하고 RESTful 원칙을 지킴(json, xml을 응답받을 수 있음)  
- HttpClient(HTTP를 사용하여 통신하는 범용 라이브러리)를 추상화(HttpEntity의 json, xml 등)해서 제공

<br/>

### RestTemplate 메서드  
**Get요청 보내기**  
- getForObject() : 결과를 객체로 반환받음  
- getForEntity() : 결과를 ReponseEntity로 반환받음  

**Post요청 보내기**  
- postForLocation() :새로 생성 된 리소스의 위치만 반환받음
- postForObject() : 결과를 객체로(새로 생성 된 리소스자체를) 반환받음
- postForEntity() : 결과를 ResponseEntity로 반환받음 

**Delete요청 보내기**   
- delete() : 주어진 URL 주소로 HTTP DELETE 메서드 실행(특정 URI의 리소스를 삭제)    

**Update요청 보내기**  
- put() : 주어진 URL로 HTTP PUT 메서드 실행
- patchForObject() : 주어진 URL로 HTTP PATCH 메서드 실행

**Any**  
- `exchange()` : HTTP 헤더를 새로 만들 수 있고 어떤 HTTP 메서드도 사용가능 
  - HTTP 메서드, URL, 헤더 및 본문을 포함한 RequestEntity를 입력받아 ResponseEntity를 리턴
- `execute()` : 요청을 수행하는 가장 일반화된 방법
  - Request/Response 콜백 수정가능  

<br/>

### RestTemplate 동작원리  

![제목 없음](https://user-images.githubusercontent.com/103614357/192153179-e5a090fc-4367-4445-9884-1c0fcc2d3f53.png)  

1. application이 내부에서 REST API 요청을 위해 RestTemplate을 생성하고, URI, HTTP 메소드 등의 헤더를 담아 요청 
RestTemplate의 메서드 호출.

2. RestTemplate은 HttpMessageConverter를 이용하여 requestEntity(Java Object)를 요청 메세지(Request Body에 담을 Message(JSON 등))로 변환  

3. RestTemplate은 ClientHttpRequestFactory에서 ClientHttpRequest를 받아와서 요청 전달    
  참고)    
  &nbsp; RestTemplate은 통신과정을 ClientHttpRequestFactory(ClientHttpRequest, ClientHttpResponse)에 위임    
  &nbsp; (ClientHttpRequestFactory의 실체는 HttpURLConnection, Apache HttpComponents, HttpClient와 같은 HTTP Client)

4. ClientHttpRequest는 요청메세지를 만들어 HTTP 프로토콜을 통해 서버와 통신   
(= 실질적으론 ClientHttpRequest가 HTTP 통신으로 요청 수행)  

5. RestTemplate은 ResponseErrorHandler로 에러 핸들링 수행, 오류 있다면 처리로직을 태움   
  ResponseErrorHandler는 오류가 있다면 ClientHttpResponse에서 응답데이터를 가져와서 처리   

6. RestTemplate은 HttpMessageConverter를 이용해서 응답메세지(Response Body의 Messag)를 Java Object(Class responseType)로 변환  

7. Application에 결과 전달  

<br/>

### RestTemplate 사용  
**1. URI 설정**    
- String 변수 사용방법

```java
String url = "요청 URL";

//쿼리스트링 추가
url = url + "?파라미터명=" + 값;
url = url + "&파라미터명=" + 값;
```

- StringBuffer 객체 사용방법

```java
StringBuffer urlBuffer = new StringBuffer();
urlBuffer.append("요청 URL")
         .append("?파라미터명=")
         .append(값)
         .append("&파라미터명=")
         .append(값);

//urlBuffer.toString()로 사용하기
```

- URI 객체 사용방법  

```java
//URL파라미터 만들기
URI uri = UriComponentsBuilder.fromHttpUrl("요청 URL")
                .queryParams("파라미터명", 값)
                .queryParams("파라미터명", 값)
                .queryParams("파라미터명", URLEncoder.encode(값, "UTF-8"))
                .build();

//uri.toString()로 사용하기
```

```java
//URL파라미터 MultiValueMap 사용해서 만들기
MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
params.add("파라미터명", 값);
params.add("파라미터명", 값);

URI uri = UriComponentsBuilder.fromHttpUrl("요청 URL")
                .queryParams(params)
                .build().encode()
                .toUri();
```

- UriComponents 객체 사용방법

```java
UriComponents uriComponents 
    = UriComponentsBuilder.newInstance()
                          .path("요청 URL")
                          .queryParam("파라미터명", 값)
                          .queryParam("파라미터명", 값)
                          .build();

//uriComponenets.toUriString()로 사용하기
```

<br/><br/>

**2. Header 만들기**  
- HttpHeaders 사용
- 만든 header는 HttpEntity 클래스에 추가하여 사용

```java
HttpHeaders headers = new HttpHeaders();

//ContentType 설정
headers.add("Content-Type", "application/json");

//ContentType, Accept 설정
//headers.setContentType(new MediaType("application","json",Charset.forName("UTF-8")));
//headers.setAccept(Arrays.asList(new MediaType[] { MediaType.APPLICATION_JSON }));
```

<br/><br/>

**3. Body 만들기**  
- body는 보통 key, value의 쌍으로 이루어지기 때문에 -> MultiValueMap 사용  
- 만들어진 body는 HttpEntity 클래스에 추가하여 사용  

```java
//MultiValueMap 사용해서 만들기
MultiValueMap<String, String> body = new LinkedMultiValueMap<>();

body.add("키", "값");
body.add("키", "값");
```

```java
//JSONObject 사용해서 만들기
JSONObject jsonObject = new JSONObject();

jsonObject.put("memberId", memberId);
jsonObject.put("sessionId", request.getRequestedSessionId());
jsonObject.put("publicDataId", publicDataId);
```

```java
//Dto 사용해서 만들기  
SubmitData dto = SubmitData.builder()
                .SubNum(SubNum)
                .Pnum(Pnum)
                .Pcode(Pcode)
                .build();
```
Body로 데이터를 전달하는 HTTP.POST의 경우, Dto 인스턴스 그대로 데이터를 전달할 수 있음  


<br/><br/>

**4. HttpEntity(RequestEntity) 생성**  
- `HttpEntity<T>` : HTTP요청또는 응답에 해당하는 HttpHeader와 HttpBody를 포함하는 클래스
  - RequestEntity, ResponseEntity : HttpEntity 클래스를 상속받아 구현한 클래스  

```java
HttpEntity<MultiValueMap<String, String>> entity 
    = new HttpEntity<>(body, headers);
```

```java
HttpEntity<SubmitData> entity 
    = new HttpEntity<>(dto, headers);
```


```java
HttpEntity<String> entity 
    = new HttpEntity<>(jsonObject.toString(), headers);
```
  
<br/>  
   
참고) HttpEntity<T> 클래스  

```java
public class HttpEntity<T> {
    public static final HttpEntity<?> EMPTY = new HttpEntity();
    private final HttpHeaders headers;
    private final T body;
    ...
}
```

<br/><br/>

**5. HTTP 요청해서 ResponseEntity 객체로 응답받기**  
- `ResponseEntity<T>` : 사용자의 HttpRequest에 대한 응답 데이터를 포함하는 클래스
  - HttpStatus, HttpHeaders, HttpBody 포함

```java
RestTemplate restTemplate = new RestTemplate();

ResponseEntity<String> response 
    = restTemplate.exchange("요청 URL", HttpMethod.POST, entity, String.class);
```

- HTTP POST 요청에 대한 응답 확인  
  
```java    
System.out.println("status : " + response.getStatusCode());
System.out.println("body : " + response.getBody());
```

<br/><br/>

Reference  
https://velog.io/@soosungp33/%EC%8A%A4%ED%94%84%EB%A7%81-RestTemplate-%EC%A0%95%EB%A6%AC%EC%9A%94%EC%B2%AD-%ED%95%A8  
https://frogand.tistory.com/91  
https://hongdosan.tistory.com/entry/Spring-RestTemplate-%EB%A5%BC-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90  
https://hoonmaro.tistory.com/46  
https://blog.naver.com/hj_kim97/222295259904    
https://jojoldu.tistory.com/478   
https://bimmm.tistory.com/34  
<br/>
