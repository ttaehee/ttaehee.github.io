---
title: Spring) Annotation
excerpt: 어노테이션
---

# Annotaion
- annotation 이 돌아가는 과정
 : 어노테이션 정의 -> 클래스에 어노테이션 배치 -> 코드가 실행되는 중에 Reflection을 이용하여 추가 정보를 획득하여 기능 실시  

## Reflection
  : 프로그램이 실행 중에 자신의 구조와 동작을 검사, 조사, 수정하는 것 <br/><br/>
  
Java와 같은 객체지향 프로그래밍 언어에서 reflection을 사용하면 컴파일 타임에 인터페이스, 필드, 메소드의 이름을 알지 못해도 실행 중에 접근 가능  
  => Reflection을 이용하면 annotation 지정만으로 원하는 클래스 주입가능  

### @ComponentScan
1. <component-scan> 선언에 의해 특정 패키지 안의 클래스들을 스캔  
2. component annotation 이 있는 클래스를 찾아서 Context에 bean 등록, 인스턴스 생성  
  이때 자동으로 등록되는 Bean의 이름은 클래스의 첫 문자가 소문자로 바뀐이름 적용  
  (HomeController -> homeController)  
  * ApplicationContext.xml 에 <bean id="abc" class="ABC"/> 직접등록과 같음  

- @Component를 구체화 : @Controller, @Service, @Repository
	- 해당 클래스가 controller, service, repository 로 사용됨을 spring framework에 알림 (가독성)

### @RequestMapping
: 요청 URL을 어떤 method가 처리할지 mapping해줌 -> controller나 controller의 method 에 적용
- 요청받는 형식 정의 안되어있다면 default GET
- 모든 매핑정보는 spring 에서 제공하는 HandlerMapping Class가 가지고 있다

### @Configuration
: 클래스에 적용하고 @Bean을 해당 class의 method에 적용하면 @Autowired로 Bean을 부를 수 있음

### @Required
: setter method에 적용하면 Bean 생성시 필수 property임을 알림  
-> bean property 구성시 xml 설정 파일에 반드시 property를 채워야 함  
  (그렇지 않으면 BeanInitializationException 예외발생)
	
### @Resource
: @Autowired와 마찬가지로 Bean객체 주입
- 차이점은 Autowired는 타입으로, Resource는 이름으로 연결
- annotation 사용으로 인해 특정 frameword 에 종속적인 어플리케이션을 구성하지 않기위해 사용
- classpath내에 jsr250-api.jar파일 추가해야 사용가능
	

## @Controller와 @RestController의 차이
1. @Controller는 API와 view(화면)를 동시에 사용하는 경우 사용  
  view return이 주목적  
2. @RestController  
  view가 필요없는 API만 지원하는 서비스에 사용  
  data(json, xml) return이 주목적 <br/><br/>
	
  즉=> @RestController  : @Controller + @ResponseBody  
  *@ResponseBody : JSON형태로 반환


## Parameter 받는 방법
### @RequestParam
: HTTP GET 요청에 대해 매칭되는 request parameter 값이 자동으로 들어감  
  =url 뒤에 붙는 parameter 값을 가져올때 사용

### @PathVariable
: HTTP 요청에 대해 매칭되는 request parameter 값이 자동으로 들어감(binding)  
  =uri에서 각 `구분자에 들어오는 값`을 처리해야할 때 사용  
- RestAPI 에서 값 호출할 때 주로 사용 <br/><br/>

ex) @RequestParam와 @PathVariable 동시 사용 예제  
```
@GetMapping("/user/{userId}/invoices")  
public List<Invoice> listUsersInvoices(@PathVariable("userId") int user,
	@RequestParam(value = "date", required = false) Date dateOrNull) {
```
GET /user/{userId}invoices?date=190101  
=> 구분자 {userId}는 @PathVariable("userId")로  
   뒤에 이어붙은 parameter는 @RequestParam("date")로 받아옴  

### @RequestBody
: requestData를 바로 model이나 클래스로 매핑  
- HTTP POST 요청에 대해서만 처리
	- request body에 있는 request message 에서 값얻어와 매칭  
  

### @ModelAttribute
: HTTP parameter, body 내용을 setter함수를 통해 객체에 데이터 연결(binding)  
(RequestBody와 다르게 json을 받아 처리할 수 없음)  


## Spring AOP Annotation
### @Aspect
: 해당 클래스가 부가기능(횡단관심사) 클래스임을 알려줌  
자동으로 bean 등록되는것은 아님 따로 해주어야함 @Component  

### @PointCut ("execution()")
- execution(반환타입 패키지명.클래스명.메서드명(매개변수))  

### advice 기능수행
- @Before : taget method가 호출되기 전에 advice 기능수행
- @After : taget method의 결과(성공,예외)에 관계없이 완료되면 advice 기능수행
- @Around : 핵심관심사의 실패여부와 상관없이 target method 호출 전, 후 수행
- @AfterReturning : target method가 성공적으로 결과값 반환 이후 수행
- @AfterThrowing : target method가 수행중 예외를 던기면 수행
