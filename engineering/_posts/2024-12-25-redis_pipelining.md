---
title: 왜 내 Redis는 느릴까? 성능 병목의 원인과 해결법  
excerpt: "Pipelining을 활용한 Redis 호출 개선 :: 네트워크 비용 줄이고 응답 속도 높이기"  
---

<br/>

- slow api를 분석할 때, 지금까지 주로 발견했던 병목은 Mysql 쪽에서 발생한 경우가 대부분이었다    
그런데 이번 slow api 는 Datadog 그래프를 활용해 분석하던 중 조금 새로운 원인을 발견했다    
Redis를 다중 호출하고 있는 패턴이 그래프에서 드러난것      

<br/>

## 문제 파악 :: Redis 다중 호출

![스크린샷 2024-12-25 오후 3 03 43](https://github.com/user-attachments/assets/32877f1e-79ce-4893-b9f9-331223c14522)   
![스크린샷 2024-12-25 오후 3 16 04](https://github.com/user-attachments/assets/c99ecfac-9b79-4195-a343-2274cd84ea16)

<br/>

로직을 분석해보니 for 문 내부에서 Redis 를 여러번 호출하며 데이터를 조회해오는 로직이 있었다     
페이징이 30개였는데 관련 데이터를 redis 에서 조회중이라 최대 30번 호출하였다   
이러한 반복 호출로 인해 api 의 전체 응답시간이 지연되었던 것이었다    
이는 시간적인 문제 뿐 아니라, 네트워크 적으로도 비효율적으로 보였다     
 
<br/>

Redis가 단일 스레드 기반으로 동작하는 것으로 유명하지만 호출은 한번에 모아서 할 수 있지않을까   
Mysql 에서 bulk insert, delete 을 사용하는거처럼 말이다   

<br/><br/>

## Redis(Remote dictionary server) 의 동작
방법을 찾아볼겸 Redis 동작에 대해 이 기회에 겸사겸사 알아보았다    

Redis는 TCP 기반의 client-server 모델로 단일 스레드 기반으로 동작하며, client가 요청을 보낸 것을 큐에 넣고 순차적으로 처리하는 방식으로 작동한다       
다른 In-memory 데이터베이스(ex, memcached)와의 가장 큰 차이점은 다양한 자료구조를 지원한다는 것이다   

<br>

Redis는 부분적으로 Single thread와 Multi Thread를 함께 사용한다   
client로 부터 전송된 네트워크를 읽는 부분과 전송하는 부분은 Multi Thread로 구현되어있으며, 우리가 Redis에 요청한 명령을 실행하는 부분은 Single thread로 구현되어 있다   
때문에 single thread의 장점인 atomic한 요청 처리가 가능하고, race condition을 피해 데이터의 정합성을 보장하기 쉽다    

<br/>

이 방식 덕분에 Redis는 자체적으로 빠른 응답 속도를 자랑한다 (초당 100,000개의 요청을 처리할 수 있는 고성능 서버로 알려져있다)  

<br/>

### Redis의 Request/Response 프로토콜과 RTT (Round-trip time)

이 때 client -> server -> client 의 과정을 거치는데, 네트워크 요청을 시작한 후 응답을 받는 데 걸리는 시간을 RTT (Round-trip time, 왕복 시간)라고 한다    
만약 RTT가 100ms라면 Redis가 아무리 고성능 서버로 빠르게 처리한다고 하더라도 application에선 초당 10개의 요청만 처리 가능하다    
이번 slow api 문제는 이 RTT 와 관련이 있었던 것이다   
전체 api 응답 시간에 Redis 의 RTT 까지 포함이 되는데 이걸 한 api 내에서 여러번 겪고있다보니 느릴수밖에

<br/><br/>

## 개선 방법 :: Redis Pipelining을 통한 배치 처리

이를 개선하기 위해 Redis Pipelining을 사용하였다   
[Spring Data Redis > Redis > Pipelining](https://docs.spring.io/spring-data/redis/reference/redis/pipelining.html)    

<br/>

Pipelining은 client가 명령어 마다 요청에 대한 응답을 기다리는 것이 아닌, 여러 명령을 하나로 묶어 서버로 보내고 결과를 한꺼번에 응답 받는 기능이다      
100번 요청을 보내고 응답 받을 것을 한 번으로 해결하므로 RTT로 인한 네트워크 비용을 크게 줄일 수 있어 이번 문제의 해결책으로 딱이었다           

또한, Pipelining은 각 명령어의 실패가 전체 작업에 영향을 미치지 않도록 보장한다       
트랜잭션처럼 원자성을 보장하지 않지만 명령어 단위로 실패를 처리할 수 있어 하나의 명령이 실패하더라도 다른 명령은 정상적으로 처리되므로 보다 안정적인 처리가 가능하다      

<br/>   

### 다른 방법 :: MGET 활용
또 다른 방법으로는 MGET 명령어 사용이 있다      
Redis에서 MGET 명령어를 사용하면 하나의 요청으로 여러 개의 키에 대한 값을 한 번에 가져올 수 있다    
그러나 MGET 자체가 하나의 명령이므로 모든 데이터를 가져올 때까지 다른 명령어는 대기하게 된다

<br/>

또한 우리 서비스는 Redis 클러스터를 사용중인데, Pipelining은 각 명령을 개별적으로 처리하기 때문에 키가 다른 슬롯에 분산되어 있어도 효과적으로 작동한다     
반면, MGET은 모든 키가 동일한 슬롯에 있어야 하며, 그렇지 않으면 여러 노드 간 통신으로 개선의 의미가 없어 제외하였다         

<br/>   

### 병렬처리와 다른점? Pipelining의 작동 원리
Pipelining은 여러 명령을 한 번에 보내고 응답을 한꺼번에 받는 방식이긴 하지만, 여러 작업을 동시에 실행하는 방식은 아니다   

<br/>

![스크린샷 2024-12-25 오후 7 00 15](https://github.com/user-attachments/assets/071b8cb1-fe7c-4446-9153-3546ee36f0b2)

Pipelining은 여러 명령을 순차적으로 보내지만, 각 명령이 응답을 기다리는 동안 다른 명령을 계속해서 보낸다   
수많은 요청을 보내기 위해 Pipeline을 형성한 후, 모든 Reqest를 Pipeline에 실어서 한 번에 보낸다   

<br/>

이를 통해 요청-응답 과정에서 대기 시간을 줄이고 네트워크 오버헤드를 최소화할 수 있다    
즉, 여러 명령을 한 번에 처리하는 것처럼 보이지만 실제로는 각 명령이 Redis 서버에서 순차적으로 처리된다   
이는 비동기적이라고 할 수 있지만, 병렬적인 실행은 아니다    

<br/>

정리해보면     
- 여러 요청을 연속적으로 보내고 각 명령의 응답을 기다리지 않고 다음 명령을 보내는 방식
- 각 명령이 처리되는 순서는 여전히 단일 스레드에서 순차적으로 진행됨

<br/><br/>

## 성능 측정 :: 100개 조회
### loop 돌며 get (기존) 
- for 루프를 돌면서 여러 개의 Redis 키를 하나씩 GET 요청하는 방법
  - 각 GET 요청마다 네트워크 왕복 시간(RTT)이 발생하므로 응답 시간이 많이 소요되고 비효율적    

```java
List<String> values = new ArrayList<>();
for (String key : keys) {
    String value = (String) redisTemplate.opsForValue().get(key);
    values.add(value);
}
```

<br/>

- 결과 :: 365ms
  
![스크린샷 2024-12-25 오후 6 35 26](https://github.com/user-attachments/assets/3abe6ef2-558c-448d-9be1-e0a08714b7ae)

로컬에서 도커 띄워서 네트워크가 가까울거 같음에도 레이턴시가 있나보다      

<br/>

### Mget 활용 
- MGET 명령어로 여러 키에 대한 값을 한 번에 가져오기
  - 모든 데이터를 가져올 때 까지 다른 명령어는 대기하게 됨 

```java
List<Object> values = redisTemplate.opsForValue().multiGet(keys);
```

- 결과 :: 19ms
  
![스크린샷 2024-12-25 오후 6 32 28](https://github.com/user-attachments/assets/14bcbf94-9f96-437e-98c0-dc19e38807ba)

<br/>

### Pipelining 

- Spring data redis의 RedisTemplate이 제공하는 executePipelined 메서드 활용
  - 이 메서드는 반드시 null을 리턴해야함
    - executePipelined는 응답을 바로 리턴하지 않고, 여러 명령을 보낸 뒤 그 결과를 나중에 모아서 반환함
    - 메서드 자체는 null을 리턴하도록 설계되어 있고, 결과는 반환된 목록에서 확인하는 방식

```java
List<Object> results = redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
    keys.forEach(key -> connection.get(key.getBytes()));
    return null;
});
```

- 결과 :: 25ms

![스크린샷 2024-12-25 오후 6 34 15](https://github.com/user-attachments/assets/6f2eef63-1629-45dd-8921-7e1673e9ed72)

<br/><br/>

적용 후 성능 개선 효과를 데이터로 측정한걸 정리해보면   
- loop 돌며 get (기존) :: 365ms
- Mget 활용 :: 19ms
- Pipelining :: 25ms    

<br/>

Mget 사용이 조금 더 빠르긴하지만, 위에 적은 이유로 요 방법은 제외하였다   
나는 지금 단순히 포스트맨으로 요청해서 응답시간만을 지표로 보고있는데, 다른 방법이 또 있을지도 좀 찾아봐야겠다    

<br/><br/>

## Redis 호출 최적화 전과 후의 비교

그래서 최종적으로 Redis Pipelining 을 적용하였고 전과 후를 비교해보면 아래와 같다   

<br/>

- 최적화 전: Redis 호출이 순차적으로 여러번 이루어지고 있었다    

![스크린샷 2024-12-25 오후 3 03 43](https://github.com/user-attachments/assets/32877f1e-79ce-4893-b9f9-331223c14522)   
![스크린샷 2024-12-25 오후 3 16 04](https://github.com/user-attachments/assets/c99ecfac-9b79-4195-a343-2274cd84ea16)

각 호출은 독립적으로 이루어져 응답을 기다리는 동안 시간이 소요되며, 이로 인해 네트워크 지연과 요청-응답 시간의 누적이 발생한다    

![스크린샷 2024-12-25 오후 6 12 46](https://github.com/user-attachments/assets/46f1b1e8-da57-4c73-a92e-450e6129ca9b)

<br/><br/>

- 최적화 후: Pipelining을 적용한 후, 여러 명령어가 한번에 보내지고 응답을 한번에 받게 되었다      

![스크린샷 2024-12-25 오후 5 49 21](https://github.com/user-attachments/assets/8d739fdb-3388-47c7-acb9-7a7086c0d9d0)   
![스크린샷 2024-12-25 오후 3 16 04](https://github.com/user-attachments/assets/c99ecfac-9b79-4195-a343-2274cd84ea16)

이렇게 되면 요청-응답 과정에서 대기 시간을 줄이고, 그래프에서 볼 수 있듯이 호출이 더 효율적으로 처리된다    
덕분에 api 전체 응답속도도 1초대로 개선되었다     

<br/>

이번 Redis 호출 최적화를 통해 네트워크 비용을 줄여 응답 시간을 개선하였고, 그 과정에서 Pipelining의 유용성에 대해서도 알게되었다      
요 탭을 쿼리 개선도 해보고 이번엔 redis 호출도 개선해보고 야금야금 개선해가고 있긴 한데 한계가 좀 보인다 여전히 datadog slow api 최상위,,   
구조를 개선해야할듯한데, 좀 넓은 시야로 개선방향을 찾아봐야겠다    

<br/><br/>

Reference      
- [Redis Docs](https://redis.io/docs/latest/)
- [Spring Docs) Spring Data Redis Pipelining](https://docs.spring.io/spring-data/redis/reference/redis/pipelining.html)   
- [NHN Cloud) 개발자를 위한 레디스 튜토리얼 01](https://meetup.nhncloud.com/posts/224)
- [Beating Round-Trip Latency With Redis Pipelining](https://kn100.me/redis-pipelining/)
- [StackOverFlow) Possible to use pipelining with Redis cluster?](https://stackoverflow.com/questions/50146504/possible-to-use-pipelining-with-redis-cluster)

<br/>
