---
title: Spring) Filter와 Interceptor 중 무엇을 사용해야할까   
excerpt: filter와 interceptor 특징과 차이점
---

<br/>

- 이번 프로젝트에서 Spring Security를 사용해서 사용자의 **인증, 인가**를 체크하였기 때문에 **filter 단에서 보안처리**가 이루어졌다    
  공통 관심사로 보고 분리되어 있기에 효율적으로 인증, 인가 처리가 가능했다       
  interceptor도 이러한 역할을 하기 때문에 **filter와 interceptor의 차이**를 알아보았다    

<br/><br/>

## 사용자의 모든 요청 전, 공통된 처리하기

로그인이 필요한 서비스를 이용 시, 사용자가 이 기능을 요청할 때마다 **수행 전에 요청한 사용자를 체크**해야 한다      
로그인이 되어 있다면 기능을 수행하고, 안되어 있거나 올바르지 않다면 사용자를 로그인 페이지로 이동시켜야 한다   
이러한 사용자 체크 로직, 즉 인증 로직을 어디에서 구현하면 좋을까?   
  
<br/>

### controller에서 처리하면?

Spring MVC를 사용하면 클라이언트의 요청이 controller로 들어오니, **각 controller마다 기능 수행 전 인증 체크**를 하면 된다    
그러나 기능의 수마다 인증 체크 로직이 생길 것이고 이는 **중복된 코드**이다     

<br/>

그렇다면 유틸로 인증체크 로직을 관리하고, 각 컨트롤러에서 이를 호출하면 되지 않을까?     
응집도는 높아지더라도 여전히 기능의 수마다 유틸을 호출하는 코드가 중복된다    

<br/>

### controller를 가기 전에 처리하면?    
controller에서 처리할 것이 아니라, 요청들이 controller로 가기 전에 한번에 처리를 한다면 중복코드를 제거할 수 있을 것이다         
이러한 기능을 하는, 그러니까 요청들을 controller가 처리하기 전에 가로채서 로직을 끼워 넣는 기능을 하는 것이 filter와 interceptor이다   

<br/>

둘 다 공통된 기능을 Controller Layer에 요청이 들어가기 전에 처리한다는 공통점이 있는데, 차이점은 무엇일까?

<br/><br/>

## Filter와 Interceptor

<img width="477" alt="스크린샷 2023-04-28 오후 5 34 22" src="https://user-images.githubusercontent.com/103614357/235097961-5afdab83-21c2-4b38-b902-8700d72b98d9.png">

[사진출처](https://justforchangesake.wordpress.com/2014/05/07/spring-mvc-request-life-cycle/)    

<br/>

### Filter
Filter는 J2EE의 표준 스펙 기능이며 Servlet 2.3 부터 지원되는 기능으로 Spring Framework에서 지원을 하고는 있지만 Spring framework만의 기능은 아니다     
spring container가 아닌 tomcat과 같은 **web container에 의해 관리**된다  

<br/>

따라서 Filter는 Spring 범위 밖에서 처리 되며, **Request/Response 객체 조작**이 가능하다       
여기서 조작이란 Request와 Response의 내부 상태를 변경한다는 것이 아닌 **다른 객체로 바꿔친다는 의미**이다       
Spring 범위 밖에서 처리되지만, Filter도 spring bean으로 등록하는 것이 가능하다          
[참고) Filter가 스프링 빈 등록과 주입이 가능한 이유(DelegatingFilterProxy의 등장)](https://mangkyu.tistory.com/221)

<br/>

filter는 javax.sevlet의 Filter 인터페이스를 구현해야하며 3가지 method를 가지고 있다      

```java
public interface Filter {

    public default void init(FilterConfig filterConfig) throws ServletException {}
    
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException;
            
    public default void destroy() {}

}
```

- Filter 실행 메소드
  - init() : 필터 인스턴스 초기화
  - doFilter() : 실제 처리 로직
  - destroy() : 필터 인스턴스 종료

<br/>

#### Filter Chain     
Filter는 filter chain 안에 있을 때 동작하게 된다             
여러개의 filter들이 사슬처럼 연결되어 있고 서로 연결되어 연쇄적으로 동작한다    
WAS 구동 시, Application Context에 등록된 filter들이 Context Layer에 설정된 순서대로 filter chain을 구성한다  

<br/>

구성된 chain은 인스턴스 초기화(`init()`)를 거친 후 각 필터에서 `doFilter()`를 통해 필터링 작업을 처리하고 `destroy` 된다     
doFilter()의 경우 필터 처리를 구현할 수 있으며, 매개변수로 request와 response를 받아 `chain.doFilter`를 통해 다음 filter로 값을 변경하여 넘길 수 있다      

```java
chain.doFilter(request, response);
```

이 때 환경 설정은 tomcat을 사용할 경우 web.xml 또는 Java Configuration을 이용해 구현한다    

<br/><br/>

### Interceptor

<img width="741" alt="스크린샷 2023-04-30 오후 10 57 18" src="https://user-images.githubusercontent.com/103614357/235356863-bda9b8fa-fda8-4cce-a5ad-b445d52d4637.png">

(실제로는 Interceptor가 Controller로 요청을 위임하지는 않음, 그림은 단순히 처리 순서를 도식화한 것)

<br/>

Interceptor는 **Spring MVC에서 제공**하는 기술이다            
**Spring Context 내에서** HTTPRequest/Response 처리 기능을 제공한다  

<br/>

Java Servlet level에서 동작하는 filter와 다르게 Spring Context level에서 동작하므로 Dispatcher Servlet이 Request 및 Response를 처리하는 시점에 Interceptor Handler가 동작한다     

Interceptor는 filter와 다르게 HttpServletRequest/Response 등과 같은 객체를 제공받으므로 객체 자체를 조작할 수는 없다    
하지만 내부적으로 가지는 값은 조작할 수 있으므로 controller로 넘겨주기 위한 정보를 가공하기에는 용이하다    

<br/>

interceptor의 경우 HandlerInterceptor 인터페이스를 구현해야하며 3가지 method를 가지고 있다        

```java
public interface HandlerInterceptor 

    default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
	throws Exception {
        return true;
    }
    
    default void postHandle(HttpServletRequest request, HttpServletResponse response,
	Object handler, @Nullable ModelAndView modelAndView) throws Exception {
    }
        
    default void afterCompletion(HttpServletRequest request, HttpServletResponse response,
	Object handler, @Nullable Exception ex) throws Exception {
    }
}
```

- Interceptor 실행 메소드
  - preHandler() : controller 실행 전, 요청 데이터 전처리나 가공에 사용 
  - postHandler() : controller 실행 후, view rendering 실행 전
  - afterCompletion() : view rendering을 포함한 모든 작업이 완료된 후

<br/>

dispatcher servlet 후, 요청받은 URL에 대해 등록된 interceptor가 호출되며 interceptor는 `prehandle - controller 실행 - postHandle - afterCompletion` 순서로 작업을 처리한다    
이 때 환경 설정은 주로 servletContext.xml 또는 Java Configuration을 이용해 구현한다    

<br/><br/>

- 참고) AOP  
  - filter와 interceptor를 정리하다보니 AOP도 비슷하다고 생각했다    
  - filter와 interceptor는 요청과 응답 시 DisplatcherServlet이나 Handler 주변에서 동작하지만, AOP는 비즈니스 단에서의 동작이 가능하다   
  - AOP의 Advice는 logging, transaction, error 처리 등 비즈니스 단에서의 메서드를 더 세밀하게 조정하고 싶을 때 사용한다     

<br/><br/>

### 차이점 정리    

||Filter|Interceptor|
|------|---|---|
|호출되는 시점|dispatcher servlet으로 요청 가기 전|controller로 요청 가기 전|
|관리되는 컨테이너|servlet container|spring container|
|동작|J2EE의 표준 스펙 기능으로 web context에서 동작|Spring이 제공하는 기술로 spring context에서 동작|
|Request/Response 객체 조작 가능 여부|o|x|
|환경설정|web.xml 또는 Java Configuration을 이용해 구현|servletContext.xml 또는 Java Configuration을 이용해 구현|

<br/><br/>

## Filter와 Interceptor, 무엇을 써야할까?

### filter의 용도(예시)
spring과 무관하게 전역적으로 처리해야 하는 작업들을 처리할 수 있다        
- 공통된 보안 및 인증,인가 관련 작업
  - 전역적으로 해야하는 보안 검사(XSS 방어 등)를 통해 올바른 요청이 아닐 경우 차단 가능
- 모든 요청에 대한 로깅 또는 감사
- web application에 전반적으로 사용되는 기능
  - 이미지나 데이터 압축
  - 문자열 인코딩

<br/>

### interceptor의 용도(예시)
클라이언트의 요청과 관련되어 전역적으로 처리해야하는 작업들을 처리할 수 있다    
- 세부적인 보안 및 인증/인가 공통 작업
  - 특정 그룹의 사용자는 어떤 기능을 사용하지 못하게 해야할 경우
- API 호출에 대한 로깅 또는 감사
- controller 로 넘겨주는 정보의 가공

<br/>

### 정리 :: interceptor에 유저 정보 추출 로직 구현하기  

filter와 interceptor의 특징과 차이에 대해 알아보았다    
내 생각에 인증 체크는 클라이언트에서 들어오는 디테일한 처리라고 생각되어 **interceptor에서 하는 것이 나을듯** 하지만, spring security를 사용하고 있기 때문에 보안관련 체크는 filter단에서 이루어지고 있다     

<br/>

**비즈니스 로직에 집중**할 수 있도록, 제공해주는 기능을 사용하는 것이 좋다고 판단했기 때문에 spring security 사용은 계속해서 유지할 것이다    
다만 현재 인증 후 유저의 정보가 필요한 경우, **jwt로부터 유저 정보를 추출하는 util을 호출**하고 있다   
호출하는 코드가 중복되므로 **interceptor에서 유저 정보를 추출해 controller에게 사용자의 정보를 제공**할 수 있도록 처리 해봐야겠다  

<br/><br/>

Reference      

[필터(Filter) vs 인터셉터(Interceptor) 차이 및 용도](https://mangkyu.tistory.com/173)     
[Filter와 Interceptor](https://junghyungil.tistory.com/123)

<br/>
