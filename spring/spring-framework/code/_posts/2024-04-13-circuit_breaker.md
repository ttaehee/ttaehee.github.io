---
title: Spring) Cricuit Breaker 적용하여 장애에 강한 서비스 구현하기   
excerpt: Resilience4j 라이브러리 사용
---

<br/>

## Cricuit Breaker 도입기

### 상황    
결제 모듈 분리 (완벽한 MSA는 아니지만 유사하게 일단 모듈과 서버만 분리) -> 빈번하게 결제 서비스 API 호출 필요

### 이로 인해 예상되는 문제점들
만약 결제 서비스에 장애가 발생한다면?

- 예상되는 장애 상황
  - 호출한 결제 서비스로 요청이 아예 실패하는 경우
  - 호출한 결제 서비스에 요청은 전달되지만, 응답이 지연되는 경우

  => 다른 상황이지만 서비스에 미치는 악영향은 비슷

<br/>

- 예상되는 악영향
  - 현재 동기 방식으로 상호 통신 중
    
    -> 연쇄 장애 발생 여지가 있음     
      => 장애 전파 & 장애가 발생한 서버에 계속 요청을 보낸다면 장애 복구까지 힘들어짐
      
      - 장애가 발생한 서비스를 호출한다면 요청이 타임아웃만큼 대기하게 됨  
      - 일정시간동안 thread와 memory 및 CPU 등의 자원 점유       
        = system resource를 부족하게 만들어 장애 유발 

<br/>        

=> 장애 발생한 서비스 탐지 + 장애 발생 시, 요청 차단 필요성 파악 

- 하고 싶은거
  - 호출할 서비스가 이미 장애가 발생한 상태라면 요청을 보내고 싶지 않음
  - 호출한 서비스가 장애가 발생한 경우, 사용자에게 에러가 아닌 정상적인 응답을 보여주고 싶음
  - 호출할 서비스가 장애가 발생하진 않았지만 응답 지연이 길어 일정 자원을 점유한다면, 일정 자원은 격리해서 점유하지 못하도록 한 후 다른 요청은 성공하게 하고 싶음

<br/>

### 해결 방법 생각해보기   

- cron으로 health check를 주기적으로 진행하여서 요청 차단하기     
  - 현재 채팅 서버도 해당 문제를 해결하고자 cron으로 주기적인 헬스체크 진행중       
    그러나, 채팅 서버가 다시 동작하더라도 수동으로 변경사항 반영 필요 => 번거로움 비효율적    

=> 시스템 환경 체크를 탄력적으로 만들어 보자 = Circuit Breaker를 추가하자  

- Circuit Breaker
  - 서비스에 문제가 감지되면 시간초과(timeout)를 무시하고 바로 실패하도록 circuit 열기
  - 요청이 차단되면 해당 서비스가 정상 동작하는지 확인하고자 주기적으로 요청 보내서 검사
  - 장애 복구용 프로브(반열림 서킷(half-open circuit)) 사용하기     
  - 프로브가 서비스의 정상 동작을 감지하면 circuit 닫기
 
=> 시스템 환경 자가치유 가능      
  Circuit Breaker의 상태 및 히스토리 관리 등을 위한 추가 비용은 발생     
  
<br/><br/>

## Design Pattern) Circuit Breaker Pattern

서킷 브레이커 패턴 : 외부 서비스에 의한 문제를 방지하기 위해 등장    

- 문제가 발생한 지점을 감지 -> 실패하는 요청을 계속하지 않도록 방지함       
  => 시스템의 장애 확산 방어 + 장애 복구를 도와줌 => 사용자의 불필요한 대기 막아줌

<br/>

### Circuit Breaker의 3가지 상태

- Callee : 외부 API    
- Caller : 클라이언트(다른 서버를 호출하는 서버)   

|   |Closed|Open|Half Open|
|------|---|---|---|
|상황|모든 것이 정상|외부(Callee)에 장애가 발생한 상황|Open 상태가 되고 일정 시간이 지난 상황|
|요청|Open 상태가 되고 일정시간이 지난 상황|외부(Callee)로의 요청을 차단하고 바로 에러를 받음|외부(Callee)로의 요청을 차단하고 바로 에러를 받음|
|상태 전이|외부(Callee)로의 요청을 차단하고 바로 에러를 받음|특정 시간이 지나면 Half Open 상태가 됨|일부 허용된 요청들이 성공한 경우 Closed 상태로, 실패인 경우 Open 상태로 변경|    

<br/>

- 외부에 장애가 발생했는지 판단하는 기준 (각각의 정해진 임계치가 넘어갈 경우 요청 차단)
  - slow call: 설정한 시간보다 오래 걸린 요청 = 지연 요청
  - failure call: 요청이 실패하거나 에러를 응답받은 요청 = 실패 요청

=> 이러한 기준을 적용해 규칙 만들 수 있음   
ex) 현재 채팅서버에 적용한 규칙 : 헬스체크가 연속 2번 실패할 경우 요청을 차단한다   

<br/>

### 동작방식   

<img width="426" alt="스크린샷 2024-04-12 오후 11 40 32" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/80a55f49-4b98-4b6a-acb3-44c2f227f02b">

1. 외부 서버 정상 실행중 : circuit이 닫혀있고 요청이 정상적으로 전달됨      
2. 외부 서버에 장애 발생 -> 요청 계속해서 실패 -> 회로가 Open 상태가 됨
3. 이후의 요청들은 더 이상 전달되지 않고 차단됨 + 빠르게 에러 또는 실패 응답 반환   
4. 이후에 외부 서버가 정상적으로 복구됨
5. 회로가 Open 상태가 된지 특정 시간이 지나고, Half Open 상태로 변경됨 (만약 Half Open 상태가 되었는데도 외부 서버가 복구되지 않았다면, 요청들은 실패해서 다시 Open 상태로 변경될 수도)       
  = 중요! 이러한 상태 변경이 자동으로 수행됨   
6. 일부 요청들이 외부 서버로 전달되고, 응답에 성공하여 Closed 상태됨
7. 모든 요청들이 정상적으로 전달됨

<br/>

- ex)
  - 요청이 2번 연속 실패하면 요청을 차단한다
    - failure call 기준으로 Closed -> Open 상태로 전환    
  - 요청이 차단한 상태에서 10초 이후의 2번 연속의 요청이 성공하면 정상적으로 요청을 처리한다
    - Open -> Half_Open 상태로 전환 시간 : 10초로 설정
    - Half_Open -> Closed/Open 판단 요청 개수 : 3개로 설정
      - Open 상태 10초 이후 요청이 2번 연속 성공하면 Closed 상태로 전환, 실패하면 Open 상태 유지

<br/>
  
### 장점

- 장애 감지 및 격리
- 자동 시스템 복구
- 빠른 실패 및 고객 응답
- 장애 서비스로의 부하 감소    
  - 외부 서비스가 완전히 죽지는 않았지만 slow query 등의 이유로 사용 가능한 thread가 더 남아있지 않은 경우에도 더 이상의 요청이 유입되지 않아 장애 복구 기회를 얻음   
- 장애 대안 커스터마이징    

<br/><br/>

## Java 진영의 Circuit Breaker 라이브러리
자바 진영의 서킷 브레이커 라이브러리로는 크게 Hystrix와 Reslience4J가 존재    
- Hystrix는 netflix에서 만든 open source인데, deprecated 되었으므로 [Resilience4j](https://resilience4j.readme.io/docs/getting-started) 사용하기 (Hystrix에서도 resilience4j 사용 권장)    

<img width="608" alt="스크린샷 2024-04-12 오후 5 02 09" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/9372d3a1-d0ba-4881-a7c1-02bdedd557b5">

<br/>

### Resilience4j의 Circuit Breaker 구현 원리
Resilience4j에서는 어떻게 Circuit Breaker Pattern을 구현할까?   

-> 각 호출 결과를 저장하고 집계하는 과정에서 `슬라이딩 윈도우`를 사용    

<br/>

- Resilience4j의 슬라이딩 윈도우
  - Count-based sliding window : 요청 `개수` 단위로 요청을 저장 및 집계하는 슬라이딩 윈도우  
  - Time-based sliding window : 요청 `시간` 단위로 요청을 저장 및 집계하는 슬라이딩 윈도우  

=> 이러한 슬라이딩 윈도우를 기반으로 요청을 집계 및 저장하고, 이를 기반으로 Circuit Breaker의 상태를 업데이트함   
(더 자세한 구현은 공식문서를 참고)

<br/>

### RestTemplate에 Resilience4J 적용하기     
[SpringBoot에서 Resilience4J 사용하기](https://resilience4j.readme.io/docs/getting-started-3)   
[Resilience4J의 Circuit Breaker 모듈](https://resilience4j.readme.io/docs/circuitbreaker)   
<br/>

**1. 의존성 추가**     

```
// resilience4j (actuator, aop는 필수로 같이 선언되어야함)
implementation 'org.springframework.boot:spring-boot-starter-actuator'
implementation 'org.springframework.boot:spring-boot-starter-aop'

implementation 'io.github.resilience4j:resilience4j-spring-boot2'
// implementation 'io.github.resilience4j:resilience4j-spring-boot3'
//or
implementation 'org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j'
```

<br/>

**2. 설정 파일 추가**      

[Circuit Breaker 모듈에서 제공하는 설정들](https://resilience4j.readme.io/docs/circuitbreaker#create-and-configure-a-circuitbreaker)

- yaml 파일을 이용하면 설정값을 바탕으로 자동 설정(AutoConfig)
  - 공통으로 사용할 값들은 configs에 정의하고 개별 instance 설정은 instances에 작성   
    
```yml
resilience4j:
  circuitbreaker:
    configs:
      default:
        failure-rate-threshold: 50   # failure call의 임계치 설정 : 실패 비율이 이 값을 넘어가면 회로를 차단(Open)함
        slow-call-rate-threshold: 80   # slow call 임계치 설정 : slow call 발생 시 server thread 점유로 인해 장애가 생길 수 있으므로 기본값(100)보다 조금 작게 설정
        slow-call-duration-threshold: 5s  # slow call이라고 판단할 시간 / 해당 값은 TimeLimiter의 timeoutDuration보다 작게 설정해야함
        permitted-number-of-calls-in-half-open-state: 3  # Half_Open 상태일 때, 받아들일 요청(Open/Closed 상태 전환을 판단할 요청)의 개수 (3개 요청이 전부 성공 -> Closed 상태로 전환)
        max-wait-duration-in-half-open-state: 0  # Half_Open 상태를 유지할 시간 (위의 permitted calls 개수의 요청이 올 때까지 무한정으로 상태 유지 / 해당 시간만큼 대기했는데 permitted calls 개수까지 도달하지 않았다면 Open 상태로 전환)
        sliding-window-type: COUNT_BASED   # 슬라이딩 윈도우를 요청의 개수로 지정 (시간으로도 지정 가능)
        sliding-window-size: 10  # 10개의 요청 단위로 circuit breaker 상태 판단 (기본은 Closed 상태)
        minimum-number-of-calls: 10   # 실패율과 느린 응답 비율을 계산할 최소 요청 수 (= 이 값이 10이니까 9개의 요청이 모두 실패하더라도 회로는 닫히지 않음 10개가 되지 않아 실패율과 느린 응답 비율을 계산하지 않았기 때문) / 기본값은 슬라이딩 윈도우 크기와 같은 값으로 설정하기 때문에 동일하게 설정함
        wait-duration-in-open-state: 30s   # Open -> Half_Open으로 최소 대기 시간 / Half_Open 상태로 빨리 전환되어 장애가 복구 될 수 있도록 기본값(60s)보다 작게 설정
        registerHealthIndicator: true   # actuator 정보 노출을 위한 설정
        recordFailurePredicate: com.taehee.circuit.resttemplate.RestTemplateCircuitRecordFailurePredicate
    instances:
      default:
        baseConfig: default
  timelimiter:
    configs:
      default:
        time-out-duration: 6s # slow-call-rate-threshold 보다는 크게 설정되어야 함
        cancel-running-future: true

# actuator
# circuit breaker status를 Spring Actuator에서 보기위해서 설정
management:
  endpoints:
    web:
      exposure:
        include:
          - "*"  # 테스트를 위해 actuator 전체 노출
  endpoint:
    health:
      show-details: always
  health:
    circuitbreakers:
      enabled: true


exchange:
  currency:
    api:
      #      uri: https://api.apilayer.com/currency_data/live
      uri: http://localhost:8090/currency_data/live
      key: eipMczgiQB0BDmiHx2dX2dv9pPjGxXnh

logging:
  level:
    ROOT: INFO
```

- Resilience4J는 Thread-safe와 원자성 보장을 제공하는 ConcurrentHashMap 기반의 in-memory CircuitBreakerRegistry를 제공
- 해당 객체에서 설정 내용이 관리되며, CircuitBreaker 객체를 얻어올 수 있음   

<br/>

**3. recordFailurePredicate 작성**    
recordFailurePredicate : 어떤 예외를 Fail로 기록할 것인지를 결정하기 위한 Predicate 설정     
- 해당 클래스에서 true를 반환하면 요청 실패로 기록됨 -> 실패가 쌓이면 서킷이 OPEN 상태로 변경        


- ex) RestTemplate을 사용하여 400번대 client 외의 에러가 발생한 경우에는 모두 fail로 기록하도록 작성          

  ```java
  public class RestTemplateCircuitRecordFailurePredicate implements Predicate<Throwable> {
      //true 를 리턴하면 Fail 로 기록됨.
  	
      @Override
      public boolean test(Throwable throwable) {
          // 4XX 클라이언트 에러는 fail로 기록하지 않음
          if (throwable instanceof HttpClientErrorException) {
              return false;
          }
  
          // 그 외에 에러는 모두 failure로 기록함(HttpServerErrorException, connection, timeout, IOException 등)
          return true;
      }
  }
  ```

- 예시처럼 Predicate 클래스 적용하려면 yml에 recordFailurePredicate 내용 추가

  ```
  resilience4j:
  circuitbreaker:
    configs:
      default:
        ... 
        recordFailurePredicate: com.taehee.app.resttemplate.circuit.RestTemplateCircuitRecordFailurePredicate
  ```

<br/>

**4. CircuitBreakerNameResolver 작성**      
서킷브레이커 적용하는 방법   
- code 방식 : Circuit Breaker를 직접 주입받고 적용해주는 것      
  - 기본적으로 executeSupplier 사용   
  - fallback 처리가 필요하다면 decorateSupplier 사용
 
  ```java
  @Component
  @RequiredArgsConstructor
  public class GetExchangeRateTemplate {
  
      private final RestTemplate restTemplate;
      private final CircuitBreakerRegistry registry;
  
      public void call(String circuitName) {
          CircuitBreaker circuitBreaker = registry.find(name)
              .orElseThrow(() -> new IllegalArgumentException("invalid circuitBreaker name - name:" + name));
  
          return circuitBreaker.executeSupplier(() -> {
              return restTemplate.getForEntity(RestTemplateExchangeRateResponse.class);
          });
      }
  
  }
  ```

  => 작업 번거롭고 중복 많음   

<br/>

- annotation 방식 : Circuit Breake instance를 지정하여 적용하는 것
  - 필요하다면 fallback 속성도 지정 가능
 
  ```java
  @Component
  @RequiredArgsConstructor
  public class GetExchangeRateTemplate {
  
      private final RestTemplate restTemplate;
  
      @CircuitBreaker(name="exchange")
      public void call() {
          return restTemplate.getForEntity(RestTemplateExchangeRateResponse.class);
      }
  
  }
  ```

  => 작업이 상당히 간결해지지만   
  매번 annotation 붙여주어야함 + Circuit Breake instanc(name값) 관리 필요
  새롭게 연동해야 하는 서버가 생긴다면 번거로움, 해당 값을 잘못 지정했을 경우에 문제 발생 가능

  => AOP 적용

  ```java
  @Component
  @RequiredArgsConstructor
  public class RestTemplateCircuitBreakerAspect {
  
  	private final CircuitBreakerRegistry registry;
  
  	@Around("execution(* org.springframework.web.client.RestTemplate.*(..)) && args(url,..)")
  	public Object aspect(ProceedingJoinPoint pjp, String url) throws Throwable {
  		return aspect(pjp, new URI(url));
  	}
  
  	@Around("execution(* org.springframework.web.client.RestTemplate.*(..)) && args(uri,..)")
  	public Object aspect(ProceedingJoinPoint pjp, URI uri) throws Throwable {
  		return registry.circuitBreaker(findHost(uri))
  			.executeCheckedSupplier(pjp::proceed);
  	}
  
  	private String findHost(URI uri) {
  		return Optional.ofNullable(uri)
  				.map(URI::getHost)
  				.orElse("default");
  	}
  }
  ```

  => 중복 코드 제거 + 자동으로 Circuit Breaker도 적용 + Circuit Breaker instance도 host 기반으로 자동 식별 가능     

<br/>

**5. CallNotPermittedException 예외 처리**   

서킷이 OPEN 상태로 바뀌면 요청을 차단하고 바로 CallNotPermittedException 예외를 발생시킴     
-> 그러므로 각각의 예외처리 방법에 맞게 CallNotPermittedException 예외 처리 필요

- ControllerAdvice 클래스에 구현하기   

```java
@ExceptionHandler(CallNotPermittedException.class)
public ResponseEntity<?> handleCallNotPermittedException(CallNotPermittedException e) {
    return ResponseEntity.internalServerError()
        .body(Collections.singletonMap("code", "InternalServerError"));
}
```

<br/>

### 테스트해보기   

#### CircuitBreakerRegistry 주입받기   

```java
@Slf4j
@RestController
@RequiredArgsConstructor
@RequestMapping("/circuit")
public class CircuitBreakerTestController {
	private final CircuitBreakerRegistry circuitBreakerRegistry;

	@GetMapping("/close")
	public ResponseEntity<Void> close(@RequestParam String name) {
		circuitBreakerRegistry.circuitBreaker(name)
				.transitionToClosedState();
		return ResponseEntity.ok().build();
	}

	@GetMapping("/open")
	public ResponseEntity<Void> open(@RequestParam String name) {
		circuitBreakerRegistry.circuitBreaker(name)
				.transitionToOpenState();
		return ResponseEntity.ok().build();
	}

	@GetMapping("/status")
	public ResponseEntity<CircuitBreaker.State> status(@RequestParam String name) {
		CircuitBreaker.State state = circuitBreakerRegistry.circuitBreaker(name)
				.getState();
		return ResponseEntity.ok(state);
	}

	@GetMapping("/all")
	public ResponseEntity<Void> all() {
		Set<CircuitBreaker> circuitBreakers = circuitBreakerRegistry.getAllCircuitBreakers();
		for (CircuitBreaker circuitBreaker : circuitBreakers) {
			log.error("circuitName={}, state={}", circuitBreaker.getName(), circuitBreaker.getState());
		}
		return ResponseEntity.ok().build();
	}
}
```

<br/>

#### actuator 모니터링

`{호출하는 서버 URL}/actuator/circuitbreakers`

- Closed 상태
  
<img width="591" alt="스크린샷 2024-04-28 오후 10 59 12" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/bdef77a9-d00f-4340-9d5e-e73db4ba6f9d">

- Open 상태

<img width="457" alt="스크린샷 2024-04-28 오후 11 00 37" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/17b4eb50-1d9a-4e6d-9969-c3c9d474caeb">

<br/><br/>

Reference    
- [Resilience4j 문서](https://resilience4j.readme.io/docs/getting-started)   
- [Resilience4j github](https://github.com/resilience4j/resilience4j)
- Release It   
- [Circuit Breaker Pattern](https://topswagcode.com/2016/02/07/Circuit-Breaker-Pattern/)
- [[디자인패턴] 서킷 브레이커 패턴(Circuit Breaker Pattern)의 필요성 및 동작 원리](https://mangkyu.tistory.com/261)

<br/>
