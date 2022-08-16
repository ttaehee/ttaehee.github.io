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
  (실제 Handler의 mapping은 DispatcherServlet에 주입된 HandlerMapping 구현체에 의해 됨)    

5. 가져온 Handler를 실행(invoke) 시킬 수 있는 `HandlerAdapter`객체를 가져옴    
  (Handler 클래스타입마다 적용시킬 Adapter가 다르므로) 

6. (적용할 Interceptor가 존재한다면) `Interceptor의 preHandle` 실행  

7. mapping된 Controller가 존재하면 @RequestMapping을 통해 요청을 처리할 메소드로 이동  
  
8. `Controller-Service-DAO-Service-Controller` 로 데이터 전달  
   
9. Controller -> `DispatcherServlet` : `요청에 맞는 view정보` 전달  
  
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
<br/>
