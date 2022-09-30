---
title: Spring) Spring MVC
excerpt: Model, View, Controller
---

# Spring MVC
DispatcherServlet, View Resolver, Interceptor, Handler, View 등으로 구성

## 동작과정

![qqq](https://user-images.githubusercontent.com/103614357/184902462-9df22f0f-3ffd-4060-868f-3e407f5f8677.png)  

1. DispatcherServlet 등록 -> 웹 컨테이너가 구동될 때 DispatcherServlet의 빈들 생성  

2. 실제 사용자의 Request -> 웹 컨테이너는 DispatcherServlet 객체를 생성하고 초기화 (첫 호출시에만)  
 
3. `DispatcherServlet이 Request` 가로챔
  (모든 요청은 아니고, web.xml의 <url-pattern>에 등록된 내용만)

4. DispatcherServlet은 `HandlerMapping`에게 요청정보 보냄  
  HandlerMapping은 request uri에 알맞은 `Handler객체`를 가져옴  
  (실제 Handler의 mapping은 DispatcherServlet에 주입된 HandlerMapping 구현체(RequestMappingHandlerMapping)에 의해 됨)    

5. 가져온 Handler를 실행(invoke) 시킬 수 있는 `HandlerAdapter`객체를 가져옴     
  (Handler 클래스타입마다 적용시킬 Adapter가 다르므로   
  = 스프링은 HandlerAdapter라는 어댑터 인터페이스를 통해 어댑터 패턴을 적용함으로써 컨트롤러의 구현 방식에 상관없이 요청을 위임)   
  (DispatcherServlet은 Controller로 요청을 직접 위임하는 것이 아니라 HandlerAdapter를 통해 요청 위임)   

 ![제목 없음](https://user-images.githubusercontent.com/103614357/193285977-23f9ee05-ddfc-4510-8e67-a29aa046d510.png)  
 
6. mapping된 Controller가 존재하면 @RequestMapping을 통해 매칭  
 (@RequestMapping은 RequestMappingHandlerAdapter를 어노테이션화 한 것)  

7. HandlerAdapter가 Controller로 요청을 넘기기 전에 공통적인 전/후처리 과정  
- (적용할 Interceptor가 존재한다면) `Interceptor의 preHandle` 실행  
- (HandlerMethod)Argument Resolver : 컨트롤러의 메서드의 인자로 사용자가 임의의 값을 전달하는 방법을 제공하고자 할 때 사용 (파라미터를 유연하게 처리할 수 있게 해줌)    
  - 요청 시에 @RequestParam, @RequestBody 등을 처리  
  - 파라미터 값을 확인하여 정보를 전달받으면 Argument Resolver가 해당 parameter의 객체를 생성하여 Handler를 호출함과 동시에 객체 또한 같이 전달     
 
  (ReturnValueHandler : 응답 시에 ResponseEntity의 Body를 Json으로 직렬화하는 등의 처리함)  
 
  참고)  
  Argument Resolver와 Return Value Handler만 있다고 객체를 생성하여 전달하고, 반환 값을 반환할 수는 없음  
  값들이 어떠한 Class, Media Type인지 확인하는 절차 필요 => MessageConverter가 수행   
  ex) @Controller라면 String 값이 ViewResolver를 호출하지만 @ResponseBody가 붙어 있다면 MessageConverter 호출   
  
8. @RequestMapping을 통해 요청을 처리할 메소드로 이동   
 `Controller-Service-DAO-Service-Controller` 로 데이터 전달   
   
9. HandlerAdapter가 반환값 처리  
 Controller ->HandlerAdapter가 반환값을 처리 -> `DispatcherServlet` : `요청에 맞는 view정보` 전달     
 (RestController -> `HandlerAdapter` : `ResponseEntity` 반환     
 HandlerAdapter -> `DispatcherServlet` : HttpEntityMethodProcessor가 MessageConverter를 사용해 응답 객체를 직렬화하고 응답 상태(HttpStatus)를 설정)   
  
10. (적용할 Interceptor가 존재한다면) `Interceptor의 postHandle` 실행  
  
11. DispatcherServlet -> `ViewResolver` : `view 정보` 전달  
   
12. ViewResolver -> `DispatcherServlet` : 응답할 view에 대한 `JSP` 전달  
  
13. DispatcherServlet은 `rendering된 뷰`를 응답   
  
  => 요청 마침 <br/><br/>

## HandlerMapping & HandlerAdapter
- `HandlerMapping` : 요청을 처리할 Controller 탐색
- `HandlerAdapter` : 매핑된 Controller의 실행 요청 <br/><br/>
  
## Interceptor
Controller의 `Handler를 호출하기 전과 후`에 요청과 응답을 참조하거나 가공할수 있는 일종의 `필터`  
  (Handle : MVC의 Controller 안에서 실제 요청을 처리하는 메소드)
  
![qqq](https://user-images.githubusercontent.com/103614357/184910856-56902f70-98f0-49b2-bd2a-c0039539bc5b.png)  
  
- 핸들러 인터셉터를 등록하지 않았다면, 곧바로 컨트롤러가 실행 
- 반대로 하나이상의 인터셉터가 지정되어 있다면 지정된 순서에 따라서 인터셉터를 거쳐서 컨트롤러를 실행
  
### Interceptor 사용이유  
특정 Controller의 Handler가 실행되기 전이나 후에 추가적인 작업을 원할때  
ex) `로그인체크`, `권한체크` 등
  
- 코드양 줄음, 메모리 낭비 줄임
  - 핸들러 수 만큼 작성했던 세션 체크 코드를 인터셉터 클래스에 한번만 작성
- 코드누락 위험 줄음
  - 인터셉터 적용의 유무 기준이 되는 url을 servlet-context.xml에 설정해주게 되면 스프링에서 적용해줌 <br/><br/>

  
## xml
xml파일은 모두 객체(bean)에 대한 정의를 내림

![111](https://user-images.githubusercontent.com/103614357/184899888-e931bf29-3e64-4325-b381-99560fba3a93.png) 

### web.xml
설정을 위한 설정파일
- 처음 WAS가 구동될 때 `각종 설정 정의`해줌
- 여러 xml파일을 인식하도록 각 파일 가리켜줌 <br/><br/>

### root-context.xml 
- 한번만 new 하면 되는 애들  
- `view와 관련되지 않은 객체`를 정의  
ex) Service, Repository(DAO), DB등 비즈니스 로직과 관련된 설정 <br/><br/>

### servlet-context.xml 
- 지속적으로 new 해서 써야하는 애들 
- `요청과 관련된 객체`를 정의
- web.xml에서 DispatcherServlet 등록 시 설정한 파일 
  - 설정파일을 이용해서 스프링컨테이너 초기화시킴 
- url과 관련된 Controller, ViewResolver, (Handler)Interceptor, MultipartResolver 등의 설정 <br/><br/>
  

Reference  
https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc  
https://velog.io/@vector9999/servlet-context.xml%EA%B3%BC-root-context.xml%EC%9D%98-%EC%B0%A8%EC%9D%B4%EC%A0%90   
https://popo015.tistory.com/115 
https://ss-o.tistory.com/160   
https://nankisu.tistory.com/46   
https://maenco.tistory.com/entry/Spring-MVC-Argument-Resolver%EC%99%80-ReturnValue-Handler   
<br/>
