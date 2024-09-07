---
title: Spring) 분산환경에서 경쟁상태 해소하기
excerpt: 동시성 이슈 해결을 위한 Redis 분산락 AOP 적용
---

<br/>

- 회사에서 최근에 들어온 문의가 있었는데, 파고들다 보니 동시성 이슈였다    
  소문으로만 듣던 동시성 이슈를 실제로 접해보니 너무 신기했다   
  - 상태 변경이 안맞는터라 db를 통해 데이터를 직접 건들게 되었는데, 로그도 남지 않는 이런 수정은 지양하고 싶다      
    그래서 동시성 이슈를 해결하고자 적용해본 분산락      
  - (그나저나 카테고리가 마땅치않아 일단 spring code로 넣었다 점점 애매해지는 카테고리,,)   

<br/>

## 이슈  
간단하게 말하자면 우리 서비스에는 신청서와 견적서 개념이 있는데, 하나의 신청서에는 다른 사람들이 여러 견적서를 낼 수 있다        
신청서를 낸 사람이 취소한다면 당연히 더이상 견적서를 낼 수 없다      
그런데 취소한 신청서에 대해 견적서가 제출된 것     

<br/><br/> 

## 파악   
로그를 보니 신청서를 취소한 시각과 견적서를 제출한 시각이 같았다    
서로 다른 transaction이 locking mechanism 없이 같은 신청서에 접근하다보니 발생한 race condition 이었다     

<br/><br/>

## 해결방안 
동시성 이슈 해결은 Application(Java 등)에서도 해결이 가능하고, Database에서도 해결이 가능하다   

<br/>

### Java에서?
Java에서 해결하는 경우는 application 운영 입장에서 큰 문제가 있다      
여러 instance와 하나의 database server로 구성되어 있는 경우, 서로 다른 Instance server에서 실행되다보니 코드 내에선 문제가 없더라도 database의 입장에선 동시성 문제 발생이 가능하다     
같은 JVM 내라면 코드 내에서 synchronized keyword 등을 통해 제어할 수 있겠지만, 우리는 현재 분산환경이고 실제로도 신청서 취소와 견적서 제출 요청은 같은 시각에 각각 다른 서버를 통해 들어왔었다     

<br/>

### Database에서?
Database 단에서의 처리도 문제가 있다    
요청서는 우리 서비스의 핵심 개념 중 하나로, 조회와 수정이 빈번하게 이루어진다   
그렇기 때문에 성능상의 이유로 데이터베이스 잠금(DB lock) 또한 선택지에서 제외하였다    

<br/>

### Distributed Lock
그래서 선택한 것이 분산락     
분산락은 multi instance에서 동일한 resource에 접근할 때도 공통된 lock을 걸어 동시성 문제를 해결할 수 있다      
이러한 특성으로 이번 이슈에서 분산락은 적절한 해결 방안이라 판단했다       

<br/>

분산락을 통해 신청서와 견적서 간의 동시성 문제를 해결하면서 여러 instance에서도 일관된 처리를 해보자       

<br/><br/>

## Redis 분산락 AOP 적용
분산락 구현방법에는 Zookeeper, MySQL의 Named Lock 등 여러가지가 있는데 Redis 를 선택하였다    
현재 우리 서비스에서 이미 쓰고 있어 Zookeeper처럼 추가적인 인프라 구성이 불필요하기도 하고, Spring boot에서 Redis는 Lettuce,  Redisson 등 여러 가지 library로 지원되기 때문이다    
MySQL의 Named Lock의 경우에는 lock으로 인한 connection 대기를 피하기 위해 별도의 connection pool을 관리해야 하기도 하고, lock에 관련된 부하를 RDS에서 받는다는 점에서 제외하였다     

<br/>

### Lettuce vs Redisson

||분산락 구현|lock 획득 방식|
|------|---|---|
|Lettuce|공식적으로 분산락 기능을 제공하지 않아 직접 구현해서 사용해야 함|spin lock (lock을 획득하지 못한 경우 lock을 획득하기 위해 Redis에 계속해서 요청을 보냄) -> redis에 부하가 생길 수 있다는 단점|
|Redisson|RLock이라는 lock을 위한 interface를 제공|pub/sub (lock이 해제될 때마다 subscribe 중인 client에게 알림을 보냄 = lock을 획득하지 못한 경우에도 redis에 지속적으로 요청을 하지않음) -> 부하가 발생하지 않음|

 <br/>
 
개발의 편리함, 부하 감소를 위해 Redisson으로 선택했고, 덕분에 분산락을 구현하는 로직 자체는 간단했다       
Redis,, Pub/Sub messaging도 지원된다니 아주 쓸모가 많구만       

<br/>

**RLock**   

Redisson에서 제공하는 lock을 위한 interface    

```java
boolean tryLock(long waitTime, long leaseTime, TimeUnit timeUnit) throws InterruptedException;
```

- waitTime: lock 획득을 위해 기다리는 시간
- leaseTime: lock을 임대하는 시간
- timeUnit: 시간 단위
  
=> lock 획득 요청 시, lock을 획득할 수 없다면 waitTime 만큼 기다리고 / lock을 획득했다면 최대 leaseTime만큼 lock을 점유 가능    

<br/>

### lock 획득과 반납 코드 & 키 생성

```java
RLock rLock = redissonClient.getLock(key);

try {
  boolean available = rLock.tryLock(waitTime, leaseTime, timeUnit);
  
  if(!available){
    // 락 획득 실패 시 실행할 로직
  }
  // 락 획득 시 실행할 로직
}catch (InterruptedException e){
  //락을 얻으려고 시도하다가 인터럽트를 받았을 때
}finally{
	try{
      rLock.unlock();
    }catch (IllegalMonitorStateException e){
      //이미 종료된 락일 때
    }
}
```

분산락은 key 단위로 lock 을 관리하기 때문에 신청서 prefix와 id값을 통해서 고유 key를 만들도록 했다     

```java
String key = REDISSON_LOCK_PREFIX + CustomSpringELParser.getDynamicValue(signature.getParameterNames(), joinPoint.getArgs(), distributedLock.key()).toString();
```

```java
public class CustomSpringELParser {
    private CustomSpringELParser() {
    }

    public static Object getDynamicValue(String[] parameterNames, Object[] args, String key) {
        ExpressionParser parser = new SpelExpressionParser();
        StandardEvaluationContext context = new StandardEvaluationContext();

        for (int i = 0; i < parameterNames.length; i++) {
            context.setVariable(parameterNames[i], args[i]);
        }

        return parser.parseExpression(key).getValue(context, Object.class);
    }
}
```

<br/>

이렇게 구현한 분산락 코드를 비즈니스 로직에 추가하면 된다         

그런데 현재는 신청서 취소와 견적서 제출 로직에만 분산락 로직을 추가하면 되지만, 서비스가 확장되어 분산락 적용을 추가한다면 중복 로직이 늘어날것이다    
그리고 lock 획득은 비즈니스 로직이 아닌 부가 기능이다    

=> AOP 적용을 하자    

<br/>

### AOP 적용

```java   
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedLock {

    TimeUnit timeUnit() default TimeUnit.SECONDS;

    long waitTime() default 10L;

    long leaseTime() default 1L;
}
```

<br/>

### 주의 :: 놓쳤던점
당연해보이지만 lock 획득은 transaction이 시작된 후가 아닌 전        
즉, lock을 획득한 다음 transaction이 시작되어야 하고, lock은 transaction commit 이후 lock이 해제되게끔 설정해야 한다       
당연해보이는걸 놓친 나는 상위 transaction이 있는데 하위 transaction에만 lock 을 적용해두어서 다른 transaction에서 commit 이전 데이터를 읽어 테스트가 실패했었다

<br/><br/>

## 테스트 코드    
테스트 코드 작성에서 중요했던 점은 비동기로 진행하면서 동시에 실행되도록 하기    
이를 위해 여러 방법을 시도하다가 최종적으로 ExecutorService와 CountDownLatch를 사용하여 문제를 해결했다   
시도해본 방법과 최종 선택한 방법은 아래와 같다   

<br/>

### 시도 1 :: Runnable과 Thread
Runnable을 통해 task를 만들고 Thread를 통해 thread를 생성하였다     
테스트 코드를 비동기로 실행하려 했으나 메인 thread가 종료되면 테스트 결과를 확인하지 못하는 문제가 있어 Thread의 join()을 통해 main thread가 모든 비동기 작업이 완료될 때까지 기다리도록 하였다    

<br/>

### 시도 2 :: Completablefuture의 allof()      
두번째는 비동기 작업을 간결하게 처리하고, 여러 작업이 완료될 때까지 기다렸다가 후처리를 chaining 하기 위해 Completablefuture의 allof()를 사용했다     
allOf()는 모든 비동기 작업을 병렬로 실행하고 완료될 때까지 기다리지만, 작업이 동시에 시작되지 않는다는 점에서 테스트 코드의 정확성이 부족하다고 느껴졌다    

<br/>

정리해본 내가 테스트 코드를 작성하는데에 필요한 조건들    
- 여러 thread가 필요하다   
- 여러 비동기 작업을 동시에 실행한다  
- 여러 비동기 작업을 병렬로 실행한다  
- 메인 thread가 모든 작업이 완료될 때까지 기다린다
    
그래서 위의 사항들을 다 적용한 최종적으로 작성한 코드    

<br/>

### 최종 :: ExecutorService와 CountDownLatch   
그래서 위의 조건들을 만족시키기 위해 최종적으로 ExecutorService와 CountDownLatch 를 사용하였다   

**ExecutorService 사용**    
thread 생성을 직접 제어하지 않고 thread pool을 관리해주는 ExecutorService를 통해 작업을 관리했다 이를 통해 thread 관리를 효율적으로 수행할 수 있었다    

**CountDownLatch 적용**    
CountDownLatch를 사용하여 여러 thread가 특정 시점까지 기다리도록 조정했다     
각 thread가 countDown()을 호출하면서 설정된 count가 0이 되면, await()이 해제되었다     

<br/>   

```java
int numberOfThreads = 2;
ExecutorService executorService = Executors.newFixedThreadPool(numberOfThreads);
CountDownLatch latch = new CountDownLatch(numberOfThreads);

Runnable submitTask = () -> {
    try {
	견적서 제출 로직
    } catch (Exception e) {
	e.printStackTrace();
    } finally {
	latch.countDown();
    }
};

Runnable cancelTask = () -> {
    try {
	신청서 취소 로직
    } catch (Exception e) {
	e.printStackTrace();
    } finally {
	latch.countDown();
    }
};

executorService.submit(submitTask);
executorService.submit(cancelTask);

latch.await();
executorService.shutdown();

if (신청서가 취소 상태라면) {
    해당 신청서의 견적서 목록 중 내가 제출한 견적서는 없다  
} else {
    해당 신청서의 견적서 목록 중 내가 제출한 견적서가 있다   
}
```

위의 작업들을 적용한 테스트 코드로 비동기 작업이 동시 실행되었고, 원하는 결과를 얻을 수 있었다     
이 테스트 코드로 분산락 적용 전의 코드들을 테스트 해보았는데 실제로 동시성 이슈가 발생함을 확인할 수 있었다   
이를 통해 분산락 적용의 필요성을 다시 한 번 실감했다     

<br/>

## 끝   
이번 동시성 문제를 해결하는 과정에서 분산락의 중요성을 느낄 수 있었고, 이러한 기술적 해결 방법을 통해 시스템 안정성이 한층 올라간것 같다    
그리고 테스트 코드를 작성하면서 비동기, 병렬 작업에 대해 파고들다보니 관심이 커졌는데 이러한 기술이 효율적인 application 구현에는 필수적이라는 생각이 들었다    
그래서 읽게된 기차책~ 표지에 기차가 있어서 기차책 ㅎ   
[멀티 코어를 100% 활용하는 자바 병렬 프로그래밍],, 정말 전공서적처럼 생겨서 힘들지만 꿋꿋하게 읽어보겠어   

<br/><br/>

Reference     
- [Baeldung::A Guide to Redis with Redisson](https://www.baeldung.com/redis-redisson)
- [Github::Redisson](https://github.com/redisson/redisson)
- [JavaDoc::Redisson](https://javadoc.io/doc/org.redisson/redisson/latest/index.html)
- [Kurly Tech Blog::풀필먼트 입고 서비스팀에서 분산락을 사용하는 방법](https://helloworld.kurly.com/blog/distributed-redisson-lock/#%EB%9D%BD-%ED%9A%8D%EB%93%9D-%EB%B0%A9%EC%8B%9D)

<br/>
