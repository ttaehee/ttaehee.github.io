---   
title: Spring) Redis cache
excerpt: SpringBoot project에서 Global cache 사용하기        
---   

<br/>  

- 지난번 정리했던, `상품 정보가 caching하기 좋은 데이터인지 & Local cache를 사용했던 이유`에 대한 정리글        
  [Caffeine cache 사용하기](https://ttaehee.github.io/spring/spring-framework/cache/caffeine-cache/)     

<br/> 

**왜 Caffeine cache에서 Redis cache로 변경했는가?**       
- 상품정보만 따졌을 때는 로컬캐시로 충분할거라 생각했다     
  상품 정보의 수정은 잦지않고, 동시성 이슈에 크게 영향을 받을 데이터가 아니라 생각했다    
  그러나 최고가, 최저가는 매번 바뀌기 때문에 가격면에서는 동시성 이슈의 타격이 컸다        
  동시성 이슈 문제를 보완하고자 global cache인 redis로 변경하였다          
  spring은 cache를 추상화해 제공하기 때문에 변경은 어렵지 않았다          
  
<br/><br/> 

## Redis (REmote DIctionary System)    
Key-Value 형태를 띄고 있는 In Memory 기반의 NoSQL DBMS    
- 저장소로 사용될 때도 있지만, DB의 부하를 줄이기 위해, 혹은 select 를 빠르게 하기 위해 캐시 서버로서도 사용됨   
- List, Hash, Set, Sorted Set 등 여러 형식의 자료구조 지원 <- 캐시로서 redis의 큰 장점!   
- IO 작업이 필요한 DB와 다르게 캐시 서버인 Redis는 in-memory 저장소이기 때문에 빠름   

![redis](https://user-images.githubusercontent.com/103614357/216903519-3173e003-85dd-4577-ab0a-26be8a48ab51.png)   



<br/><br/>

## Mac에 Redis 설치하기

- Homebrew로 설치하였다   
  [redis.io/docs](https://redis.io/docs/getting-started/installation/install-redis-on-mac-os/)

```
$ brew install redis
```

<br/><br/> 

- 설정파일 redis.conf 확인하기
  - 기본 포트(6379) 확인   
  - 실제 서비스라면 Maxclient 값이 충분해야할 듯 하다 나는 default 값을 사용했다  

  ```
  /opt/homebrew/etc/redis.conf
  ```

  <img width="537" alt="KakaoTalk_20230206_152021651" src="https://user-images.githubusercontent.com/103614357/216898344-86adee6f-446c-4707-8989-892121ee133c.png">

<br/><br/>

- redis server 실행

  ```
  $ redis-server
  ```

  ![KakaoTalk_20230206_150628063](https://user-images.githubusercontent.com/103614357/216898482-d528c916-696a-4022-8392-91f6906f4e43.png)

- redis server 종료  

  ```
  control + c
  ```

<br/><br/>

- redis server 실행중인지 확인
  - 새로운 터미널 창에서 확인
  - 응답 pong = 서버 실행중임을 의미  

  ```
  $ redis-cli ping
  ```
  
- redis-cli : command line interface로 데이터를 저장, 조회, 삭제    
  - 테스트로 keys 명령어를 사용해보았는데, O(N)으로 전체 장애의 대부분이 KEYS, SAVE 설정 사용으로 발생할 정도라고 해서 비활성화했다    

  <img width="422" alt="KakaoTalk_20230206_150638776" src="https://user-images.githubusercontent.com/103614357/216898523-c2d23358-7745-4f79-b999-c172f467287f.png">

<br/>

- 참고) O(N) command List
  - KEYS. FLUSHALL, FLUSHDB, CONFIG

<br/><br/>

## Redis Cache

### 설정하기   

- dependency 추가

```
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

<br/>

- application-redis.yml

```
spring:
  redis:
    host: localhost
    port: 6379
    password:
```

<br/><br/>

- RedisProperties.java

```java
@Getter
@ConstructorBinding
@RequiredArgsConstructor
@ConfigurationProperties(prefix = "spring.redis")
public class RedisProperties {
	private final String host;
	private final int port;
}
```

<br/><br/>

- RedisConfig.java
  - RedisTemplate 방식 사용  
    - 참고) Spring Data Redis는 Redis에 RedisTemplate, RedisRepository 방식의 접근 방식 제공 

```java
@Configuration
@RequiredArgsConstructor
public class RedisConfig {

	private final RedisProperties redisProperties;

	@Bean
	public RedisConnectionFactory redisConnectionFactory() {
		return new LettuceConnectionFactory(redisProperties.getHost(), redisProperties.getPort());
	}

	@Bean
	public RedisTemplate<?, ?> redisTemplate() {
		RedisTemplate<?, ?> redisTemplate = new RedisTemplate<>();
		redisTemplate.setConnectionFactory(redisConnectionFactory());
		return redisTemplate;
	}

}
```

- RedisConnectionFactory interface 구현체  
  - LettuceConnectionFactory : [성능이 조금 더 좋아 선택했음](https://jojoldu.tistory.com/418)   
    - 비동기 방식으로 요청 -> TPS/CPU/Connection 개수와 응답속도 등 전 분야에서 Jedis 보다 뛰어남    
  - JedisConnectionFactory
- RedisTemplate  
  - Redis data access code를 간소화 하기 위해 제공되는 클래스 
  - Connection management 수행  
  - 주어진 객체들을 자동으로 직렬화/역직렬화
  - 트랜잭션 기능 제공
    - 롤백 지원은 안함    

<br/><br/>

- CacheType.java
  - 로컬캐시 때 사용했던 것 그대로! 
    - 관리 편의성을 위해 cache setting 시 사용할 key와 expire 등의 정적 정보를 하나의 class에서 정의했다   
    - 그랬더니 이번에 캐시매니저만 레디스로 바꾸어 편했다   
  - default ttl만 늘렸다    

```java
@Getter
@RequiredArgsConstructor
public enum CacheType {
	PRODUCT("product", ConstantConfig.DEFAULT_TTL_SEC, ConstantConfig.DEFAULT_MAX_SIZE),
	PRODUCTS("products", ConstantConfig.DEFAULT_TTL_SEC, ConstantConfig.DEFAULT_MAX_SIZE)

	private final String cacheName;
	private final int expiredAfterWrite;
	private final int maximumSize;

	static class ConstantConfig {
		static final int DEFAULT_TTL_SEC = 3600;
		static final int DEFAULT_MAX_SIZE = 10000;
	}
}
```

<br/><br/>

- RedisCacheConfig.java
  - CaffeineCacheConfig에는 profile 설정을 해주었다    

```java
@EnableCaching
@Configuration
public class RedisCacheConfig {

	@Bean
	public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {

		RedisCacheManager.RedisCacheManagerBuilder redisCacheManagerBuilder
			= RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(redisConnectionFactory);

		RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
			.serializeKeysWith(
				RedisSerializationContext.SerzializationPair.fromSerializer(new StringRedisSerializer()))
			.serializeValuesWith(
				RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

		return redisCacheManagerBuilder.cacheDefaults(redisCacheConfiguration)
			.withInitialCacheConfigurations(getCacheConfiguration(redisCacheConfiguration))
			.build();
	}

	private Map<String, RedisCacheConfiguration> getCacheConfiguration(RedisCacheConfiguration redisCacheConfiguration) {

		return Arrays.stream(CacheType.values())
		  .collect(Collectors.toMap(
			  cacheType -> cacheType.getCacheName(),
			  cacheType -> redisCacheConfiguration.entryTtl(Duration.ofSeconds(cacheType.getExpiredAfterWrite())))
		  );
	}
}
```

- Serializer
  - JdkSerializationRedisSerializer: default   
  - StringRedisSerializer: String 값을 정상적으로 읽어서 저장 but Entity나 VO같은 type은 cast 할 수 없음      
  - Jackson2JsonRedisSerializer(classType.class): classType 값을 json 형태로 저장    
    - 특정 classType에만 적용되게 되는 단점      
  - GenericJackson2JsonRedisSerializer: 모든 classType을 json 형태로 저장할 수 있는 범용적인 Jackson2JsonRedisSerializer
    - caching에 class type도 저장된다는 단점
    - but RedisTemplate을 이용해 다양한 타입 객체를 캐싱할 때 사용하기에 좋음

- 이번에도 지난번과 마찬가지로 cache type에 정리해둔 캐시 종류들을 사용해 세팅         

<br/>

### Spring의 Cache 추상화 

- 지난번 로컬캐시 때 알아본 내용 복습하기
  - AOP 통해 적용 & CacheManager라는 Interface 제공하여 cache를 구현하도록 하고 있음   
  - method가 실행되는 시점에 parameter에 대한 cache 존재여부 판단

<br/>
   
=> Spring의 추상화 덕분에 구현부분은 바꿀게 전혀 없었다!     
캐시를 저장할 저장소를 알려주고, 어떠한 cache manager를 사용할건지만 알려주면 변경됨      
깔끔해서 너무 좋다!    

<br/><br/>

### 결과(성능 개선)   

- caching 전    

![KakaoTalk_20230206_161423051](https://user-images.githubusercontent.com/103614357/216955233-eba0727e-2812-45e5-83a2-81eeab67c358.png)        

- caching 후  

![KakaoTalk_20230213_180006324](https://user-images.githubusercontent.com/103614357/218416905-bf35450d-bac2-4127-814c-2aafe9ea763f.png)   

<br/>
  
=> 데이터 10건 조회 기준, 26초에서 3초로 개선되었다        
다이나믹하다..!   
  
<br/><br/>

Reference     
https://redis.io/docs/getting-started/installation/install-redis-on-mac-os/     
https://www.baeldung.com/spring-boot-redis-cache     
https://www.youtube.com/watch?v=92NizoBL4uA&ab_channel=NHNCloud    
https://m-falcon.tistory.com/499        
https://velog.io/@zzarbttoo/CacheMysqlJPA%EA%B3%BC-Redis%EB%A5%BC-%ED%95%A8%EA%BB%98-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EC%9E%90     


<br/>
