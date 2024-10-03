---
title: 오픈소스 기여 시도와 놓쳤던 점
excerpt: Resilience4j 의 이슈 해결 시도
---

- 지난번에 Resilience4j를 통해 서킷브레이커를 구성해두었는데, 이번에 spring boot 3.x 로 업그레이드 하면서 Fallback method와 관련된 문제가 발생했다    
  이미 Resilience4j github 에도 동일한 이슈가 등록이 되어있는걸 찾을 수 있었다       
  - 관련 PR도 없어 보였고, 해결 전인거 같아서 직접 해결해볼 생각으로 혼자 이것저것 시도해보았다  
    결과는 좀 허무하지만 일단 첫 시도이고 처음 해본 경험이었기에 다음을 기약하며 기록해두려고 한다    
  - [github에 등록된 관련 이슈](https://github.com/resilience4j/resilience4j/issues/1993)

<br/>

## 이슈 내용
fallback method는 해당 클래스에서만 사용하므로 private으로 구성해두었는데 Spring Boot 3.x으로 업그레이드 후, 관련 bean 주입과 관련하여 NullPointerException이 발생하였다     
이 문제는 Spring의 reflection utility의 update로 인해 발생한것으로 파악되었다 

<br/>

문제를 정리해보면 Spring Boot 3.x에서는 fallback method가 private인 경우, Spring Proxy에 의해 감싸지지 않아 자동으로 주입된 bean에 null 값이 발생한다는 것이다             
이를 해결하기 위해 fallback method를 public으로 만들어 적절한 proxying을 보장하였고 실제로 문제는 해결되었다    

<br/> 

찾아보니 Resilience4j 이슈로 이미 등록되어 있던 사항이었다    
해결방법은 위와 같이 있긴 하지만 이전 버전과 호환문제도 있고 fallback method가 많다면 모든 메서드를 수동으로 수정하는 것은 비효율적이기 때문에 다른 해결방법을 모색해보았다

한창 proxy 강의(김영한님 고급편)를 듣던 중이었기에 해결방향이 이미 머리에 좀 떠올라서 의외로 쉽게 풀릴수도 있겠다 싶었기에,,    
프록시 객체 말고 실제 객체를 반환하면 되지 않을까 막연하게 생각하면서 시도해보았다      

<br/>

## 문제의 코드 예시
코드를 그대로 적을 수는 없어 비슷하게 예시로 만들어보면

```java
@Service
@RequiredArgsConstructor
public class CircuitBreakerService {
	private final MyFallBack myFallBack;

	@CircuitBreaker(name = "test", fallbackMethod = "privateFallback")
	public void callPrivateService() {
		throw new RuntimeException("Service failure!");
	}

	private void privateFallback(Throwable throwable) {
		myFallBack.handleException(throwable.getMessage());
	}
}
```

```java
@Component
public class MyFallBack {
	public void handleException(String e) {
		System.out.println("MyFallBack: " + e);
	}
}
```

<br/>

fallback method를 타도록 테스트 코드를 만들어서 돌려보면

```java
@SpringBootTest
public class CircuitBreakerServiceTest {
	@Autowired
	private CircuitBreakerService service;

	@Test
	void testCircuitBreakerFallback() {
		for (int i = 0; i < 5; i++) {
			assertThatThrownBy(service::callPrivateService).isInstanceOf(NullPointerException.class);
		}
	}
}
```

<img width="1376" alt="스크린샷 2024-10-03 오후 4 10 31" src="https://github.com/user-attachments/assets/29c74a00-c99a-4396-a8cb-4493ef2d90f9">   

<br/><br/>

혹시나 하고 fallback method를 public으로 변경해서 돌려보니 아무런 예외가 발생하지 않았다   

```java
@Test
void testCircuitBreakerFallback() {
  for (int i = 0; i < 5; i++) {
    //assertThatThrownBy(service::callPrivateService).isInstanceOf(NullPointerException.class);
    assertThatCode(service::callPublicService).doesNotThrowAnyException();
  }
}
```

<img width="989" alt="스크린샷 2024-10-03 오후 4 12 16" src="https://github.com/user-attachments/assets/cf0599c2-1293-46a2-95a9-4166e8322017">

<br/><br/>

## 디버깅

디버깅을 통해서 관련 코드를 찾고, 프록시 객체 말고 실제 객체를 반환하게끔 변경할 수 있는 코드도 찾았다    

- io.github.resilience4j.spring6.fallback.FallbackExecutor.java

<img width="1192" alt="스크린샷 2024-10-03 오후 4 17 29" src="https://github.com/user-attachments/assets/017ca28e-a814-467f-8f47-a8f78fd39521">

proceedingJoinPoint.getThis()를 통해 proxy 객체를 반환하고 있으니, proceedingJoinPoint.getTarget() 으로 변경해서 proxy 객체가 아니라 실제 대상 객체(원래 class의 instance)를 반환하도록 변경하면 되겠다! 하면서 뿌듯해했다        

<br/><br/>

### 결과 & 놓쳤던 부분
그런데 결과는 너무 허무하다    
코드를 수정해서 테스트해보기 위해 Resilience4j 코드를 클론해서 변경할 부분을 찾았는데, 이미 코드가 프록시 객체가 아닌 실제 대상 객체를 반환하도록 되어 있었다    

<img width="1377" alt="스크린샷 2024-10-03 오후 4 38 01" src="https://github.com/user-attachments/assets/062cdaa5-7bc3-408d-a61b-0838389f0c51">

<br/>

나는 `implementation 'io.github.resilience4j:resilience4j-spring-boot2:2.1.0'`에서 `implementation 'io.github.resilience4j:resilience4j-spring-boot3:2.1.0'`로 변경했는데 이 때 버전도 같이 확인하고 바꾸었어야 했다    
2.2.0 버전에서 해당 코드가 이미 반영되었던것 ㅠ
알고보니 다른 이슈 관련 PR에서 해결되었다    
- [해결 PR](https://github.com/resilience4j/resilience4j/pull/2058) 

<br/>

- 내가 놓쳤던 부분
  - 외부 라이브러리의 버전을 늘 확인하자
  - 해당 이슈는 열려있더라도 해결 여부도 미리 확인하자 (쉽게 확인할 수 있는 방법이 있는건가?ㅎ)   

<br/>

어쨌든! 혹시 또 오픈소스에 기여할만한 일이 생긴다면 이번과 같은 실수를 하지 말아야지    
그래도 덕분에 한바퀴 플로우를 겪으면서 관심이 생겼는데 오픈소스 기여에 대해서는 정보가 많지 않아서 쪼~끔 허들이 있어보인다만, 여러 사람들이 이슈를 제기하고 의견을 나누며 해결책을 모색하는 모습이 인상깊었기 때문에 한번쯤은 꼭 기여해보고싶다     

<br/>

그동안 라이브러리를 있는 그대로 받아들이기만 했었는데, 이제는 라이브러리 코드까지 개선할게 있는지 이슈는 없는지 등을 확인하면서 더 넓은 시야로 바라볼 수 있을 것 같다     
아 그리고 요즘 프로젝트 구조, 설계에 대해 점점 관심이 생기는데 오픈소스 보면서 여러가지 구조를 파악해보는것도 좋겠다는 생각이 들었다    

<br/>

비록 이번 결과는 기대와 다르게 허무했지만, 기술적인 고민을 하고 직접 수정해서 테스트해보려는 도전 자체가 의미가 있었다     
앞으로 더 적극적으로 참여할 수 있을 것 같고, 그 과정에서 또 나에게 도움이 되는 성취감과 배움을 겪을거라는 기대가 되었다         

<br/><br/>
