---
title: Spring) 비동기 태스크에서 context 유지하기 
excerpt: 비동기 작업 데코레이터
---

<br/>

## 상황

아래는 문제가 발생한 상황들
- 비동기 태스크에서 header에 접근해서 토큰 추출하여 처리하려고 하면 에러 발생       
- 슬랙 모니터링 시, 비동기 태스크 수행 중 발생한 오류는 오류 메시지에서 요청 URL이나 request 정보가 노출되지 않음  

<br/><br/>

## 원인 파악

- 발생 에러 메시지

```
No thread-bound request found:  
Are you referring to request attributes outside of an actual web request, 
or processing a request outside of the originally receiving thread? 
If you are actually operating within a web request and still receive this message, 
your code is probably running outside of DispatcherServlet: 
In this case, use RequestContextListener or RequestContextFilter to expose the current request.
```

실질적인 request가 존재하지 않거나 thread-safe 하지 않은 메서드에서 발생하는 에러라고 한다    

<br/>

Spring에서 request가 들어오면 작업을 처리하는 thread에 대한 정보를 ThreadLocal에 저장하기 때문에 해당 작업 중 모든 상황에서 context를 유지할 수 있다       
하지만, 비동기 처리를 위해 TaskExecutor 사용하면 해당 executor는 새로운 thread를 생성한다    
이 때 새로운 thread에 기존 thread의 context가 전달되지 않기 때문에 request가 존재하지 않는다고 판단되는것     

<br/>

### ThreadLocal

ThreadLocal에 대해 조금 알아보고 가자면 각 thread마다 별도의 내부 저장소를 지원해주는 역할을 한다   
일반적으로 지역 변수는 해당 변수를 선언한 코드 블록 내에서만 사용 가능하지만, ThreadLocal를 이용해 저장한 데이터는 데이터를 저장한 thread 내에서라면 어디서든 사용 가능하다   
    
Java logging framework에서 제공하는 기술인 MDC도 ThreadLocal을 이용해 만들었기 때문에 알고보니 나도 ThreadLocal을 간접적으로 사용중이었다    

- MDC (Mapped Diagnostic Context) : 프로그램 실행을 추적할 때 유용한 정보를 저장할 때 사용        
  - [참고) Baeldung : Improved Java Logging with Mapped Diagnostic Context](https://www.baeldung.com/mdc-in-log4j-2-logback)
  - Java logging framework(slf4j 등)에서 지원
  - 현재 실행중인 thread 단위에 meta 정보를 넣고 관리하는 공간
  - Map 형태(Key, Value)로 값 저장
  - 각 thread는 자체 MDC를 가지기 위해 threadLocal 로 구현되어 있음
      - 동일한 logger를 사용하더라도 로깅한 메시지는 thread마다 다른 context(MDC) 정보를 가지게됨   

<br/><br/>

## 해결

### 생각했던 방법들   
그러면 @Async를 사용한 비동기 태스크에서 요청 객체에 접근하려면 어떻게 해야할까?    

- 1 - 일단 간단하게 메인 thread에서 넘겨주려고 해보았다    
  - 1의 문제점 : 그런데 비동기이기 때문에 각각의 thread의 작업이 끝나는 시간이 다른데, 메인 thread의 작업이 끝나면 thread가 비워져서 더이상 접근할 수가 없었다
    
- 2 - Dispatchservlet의 threadContextInheritable 값을 true로 바꾸기    
  - 참고) threadContextInheritable : RequestContextHolder 를 자식 thread에 상속되게 할지 여부를 결정하는 값 (기본값 false)
    - FrameworkServlet.java : HttpServletRequest을 처리
       
      <img width="795" alt="스크린샷 2024-06-16 오후 8 48 10" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/a490a081-9c40-47ae-94de-3832ca1abd90">
      
      
      <img width="711" alt="스크린샷 2024-06-16 오후 8 46 33" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/b6c1ac47-3eef-4f18-a2b4-3dec116a2bb0">
      
      
      <img width="548" alt="스크린샷 2024-06-16 오후 8 44 14" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/9210d073-45e0-4496-9d66-97caa5837492">    
  - 2의 문제점 : 해당 코드의 영향도가 spring 전반적으로 미치게된다 일이 너무 커져        
    
- 3 - 그러면 복사한값을 넘겨줘보자 를 목표로 구현하다가 발견한 TaskDecorator     

<br/>

### TaskDecorator 비동기 작업 데코레이터   

- TaskDecorator : Spring의 TaskExecutor 에 대한 decorator interface

<br/>

비동기 작업 실행동안 작업실행 전, 후에 추가적인 작업 수행 가능할 수 있게해준다   
덕분에 비동기 작업의 작업실행 context 변경이나 작업에 대한 추가적인 로깅, 보안 체크, 성능 모니터링 등이 가능해진다      
요걸 이용해서 비동기 작업실행 전, context를 전달해주었다   

<br/>

- TaskDecorator를 이용해서 만든 Custom TaskDecorator    

```java
public class TaeheeTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        // 현재 요청의 RequestAttributes 를 가져옴
        RequestAttributes requestAttributes = RequestContextHolder.currentRequestAttributes();

        return () -> {
            try{
                // 작업 실행 전, RequestAttributes 설정
                RequestContextHolder.setRequestAttributes(requestAttributes);
                runnable.run();
            } finally {
                // 작업 실행 후, RequestAttributes 제거
                RequestContextHolder.resetRequestAttributes();
            }
        };
    }
}
```

- Custom TaskDecorator를 TaskExecutor
  
```java
@EnableAsync
@Configuration
public class ThreadPoolConfig {
    @Bean(name = "taeheeTaskExecutor")
    public Executor taeheePoolTaskExecutor() {
        ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
        taskExecutor.setCorePoolSize(10); // 기본 Thread 사이즈
        taskExecutor.setMaxPoolSize(100); // 최대 Thread 사이즈
        taskExecutor.setQueueCapacity(1000); // 대기열 크기
        taskExecutor.setThreadNamePrefix("TaeheeExecutor-"); // Thread 접두어

        taskExecutor.setTaskDecorator(new TaeheeTaskDecorator()); // custom TaskDecorator 를 TaskExecutor
        taskExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());

        taskExecutor.initialize();
        return taskExecutor;
    }
}
```

<br/>

### 테스트코드 

```java
@RestController
@RequestMapping("/task")
@RequiredArgsConstructor
public class TaskController {
	private final TaskService taskService;

	@RequestMapping("/test")
	public void test() {
		for(int i = 1; i < 10; i++){
			taskService.asyncMethod(i);
		}
	}

	@RequestMapping("/custom")
	public void custom() {
		for(int i = 1; i < 10; i++){
			taskService.customAsyncMethod(i);
		}
	}
}
```

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class TaskService {
	private static final String TOKEN = "token";

	private final HttpServletRequest httpServletRequest;

	@Async("taskExecutor")
	public void asyncMethod(int s) {
		log.info(s + " : " + httpServletRequest.getHeader(TOKEN));
	}

	@Async("taeheeTaskExecutor")
	public void customAsyncMethod(int s) {
		log.info(s + " : " + httpServletRequest.getHeader(TOKEN));
	}
}
```

<br/>

- taskExecutor (TaskDecorator 미사용)
  
<img width="1314" alt="스크린샷 2024-05-30 오후 11 50 51" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/f9906889-d073-4b24-8158-0d7662ab16ac"> 

<br/><br/>

- taeheeTaskExecutor (TaskDecorator 사용)
    - taeheeTaskExecutor로 진행하니 호출 thread의 context가 전달되어 더이상 에러가 발생하지 않았다  
  
<img width="994" alt="스크린샷 2024-05-30 오후 11 51 48" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/7d58a202-76d6-464a-bea4-e5b7447945a8">

<br/><br/>

추가) RejectedExecutionHandler
- thread pool이 작업처리에 대한 한계에 도달 시, 새로운 요청 거부되는 경우 있음 이런 경우 처리 방식 지정
    - 1) AbortPolicy (Default) 거부된 실행 요청 발생 시, RejectedExecutionException 예외 발생
    - 2) CallerRunsPolicy : 거부된 실행 요청을 해당 요청 호출한 thread에서 직접 실행
    - 3) DiscardPolicy : 거부된 실행 요청 무시
    - 4) DiscardOldestPolicy : 거부된 실행 요청 대신 가장 오래된 요청 제거 후 새로운 요청 수락

<br/>

해당 작업이 반드시 처리되었으면 해서 CallerRunsPolicy를 적용하였다    
단점으로는 응답이 느려질 수 있고 호출 thread는 blocking 될 수 있다는 점이 있으나,    
thread pool의 작업 처리 능력을 초과하는 경우에도 요청을 처리할 수 있어 실행이 보장되기도 하고    
해당 작업에서 동시에 돌아가는 thread가 5개 이내로 예상되었기 때문에 호출 thread 까지 쓰일일은 많이 없을것이라 판단했다       

<br/><br/>

Reference       
[Baeldung : An Introduction to ThreadLocal in Java](https://www.baeldung.com/java-threadlocal)   
[우아한 기술블로그 : 로그 및 SQL 진입점 정보 추가 여정](https://techblog.woowahan.com/13429/)    
inflearn (김영한 스프링 핵심 원리 고급편 - ThreadLocal)        
[Baeldung : Improved Java Logging with Mapped Diagnostic Context](https://www.baeldung.com/mdc-in-log4j-2-logback)

<br/>
