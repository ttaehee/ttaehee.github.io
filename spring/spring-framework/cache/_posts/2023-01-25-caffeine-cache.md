---   
title: Spring) Caffeine cache
excerpt: SpringBoot project에서 Local cache 사용하기        
---   

<br/>  

- SpringBoot project에서 상품 조회 시 반복되는 DB 조회를 줄이기 위해      
  Local Cache인 Caffeine cache를 사용해서 상품 정보를 caching 했다      
  
<br/> 

**상품 정보가 caching하기 좋은 데이터인가?**   
- 상품정보는 수정이 많지 않은 데이터이면서  
  - 사용자 요청과 무관하게 서비스 관리자에 의해 변경됨 = 컴퓨터의 시간을 봤을 때 잦지 않음   
- 조회는 매우 많은 데이터라 캐싱하기 적합하다고 생각한다  

<br/>  

**Local cache를 사용한 이유**     

- 상품 정보를 caching 하기위해서 local cache와 global cache에 대해 알아보았고, local cache를 사용하기로 결정하였다        
  서버를 하나만 사용할 예정이고, 상품에 대한 수정이 자주 일어날것 같지 않아, 일관성유지에 크게 상관이 없을거라 판단하였기 때문이다    
  다만, 입찰 방식이라 최저가와 최고가는 자주 바뀌기 때문에 따로 정보를 빼서 관리했고, 아직 구현전이지만 @TransactonalEventListener를 사용해서 caching 예정이다(아직 공부 더 필요)   

<br/>

**그 중에 Caffeine cache를 사용한 이유**      
- 그래서 일단 local cache 중에 찾아보았고, 많이 사용하는 라이브러리인 Caffeine cache와 Ehcache 중에 고민하였다       
  결론을 먼저 말하면 Caffeine cache로 정했다       
  - 아래에도 내용을 정리하겠지만, Ehcache가 지원하는 기능은 많으나       
    나는 캐시의 기본기능만 사용할 예정이라 많이 기능이 필요하지 않았고      
    기본 기능만 사용할 때, Caffeine Cache가 EhCache보다 성능이 좋아 결정했다(벤치마크 자료 참고함) 

<br/><br/>     

## Spring의 Cache 추상화   
- AOP 통해 적용

- Spring Framework는 application level에서 cache 기능의 추상화를 지원해줌
  - Cache의 추상화 : 흔히 캐시를 사용할 때 필요한 작업에 대한 인터페이스를 제공 
    - Spring : `CacheManager`라는 Interface 제공 -> cache를 구현하도록 하고 있음   
  - method 통해 기능 지원   
    - method가 실행되는 시점에 parameter에 대한 cache 존재여부 판단
      - 없으면 cache에 등록
      - 있으면 method를 실행시키지 않고 caching 된 데이터를 Return  

<br/>
  
```
implementation 'org.springframework.boot:spring-boot-starter-cache'   
```
- Spring Boot에서는 `spring-boot-starter-cache` Artifact를 추가 -> CacheManager 구성 할 수 있음  
  - 기본적으로 별도의 추가적인 third-party module이 없는 경우    
    -> Local Memory에 저장이 가능한 ConcurrentMap기반인 `ConcurrentMapCacheManager`가 Bean으로 등록     

<br/>

- Spring이 cache 추상화 지원 -> 개발자는 별도의 cache logic 작성 안해도됨   
  - 하지만 cache 저장하는 저장소는 직접 설정해주어야 함
  
<br/><br/>

## Caffeine Cache
Java 8을 기반으로 하는 High Performance 캐싱 라이브러리       

- 특징
  - max size 설정해두고, 해당 사이즈가 넘어갈 경우 eviction(내보내기)
  - 마지막 접근(read based) 혹은 최초 쓰기(write based)에 따라 만료 시간 설정 가능
  - refreshAfterWrite로 자동 refresh 가능
  - key, values 가 자동적으로 weak reference로 wrap되기 때문에 GC를 통해 삭제 가능
  - cache access에 대한 statistics 제공하므로 모니터링 가능   

<br/>

```
implementation 'org.springframework.boot:spring-boot-starter-cache'  
implementation 'com.github.ben-manes.caffeine:caffeine' 
```

- Caffeine third-party module 추가 -> CaffeineCacheManager를 Bean으로 등록 가능        

<br/><br/>

### EhCache   
EhCache 또한 Java 진영에서 유명한 Local Cache 라이브러리 종류 중 하나 

- EhCache는 Caffeine Cache 보다 더 많은 기능 제공  
  - 분산 처리, Cache Listener, Off Heap에 캐싱된 데이터를 저장하기 등      
    [EhCache 공식문서](https://www.ehcache.org/documentation/3.1/clustered-cache.html)  
    
<br/><br/>
 
### Caffeine Cache vs EhCache    

[벤치마크 자료 참고](https://github.com/ben-manes/caffeine/tree/master/wiki/throughput)

<br/>

**읽기 100% 성능 측정 테스트**      

![1](https://user-images.githubusercontent.com/103614357/214115052-6d8fe247-5e56-49ba-9fb2-5663a229d09e.png)   

- Caffeine Cache > ConcurrentLinkedHashMap > EhCache    
 
<br/><br/>

**읽기 75% 쓰기 25% 성능 측정 테스트**     

![2](https://user-images.githubusercontent.com/103614357/214115864-2f03dd0c-5945-49e7-8c2b-9b7c549305c5.png)   

- Caffeine Cache > ConcurrentLinkedHashMap > EhCache    
  - Caffeine Cache 와 ConcurrentLinkedHashMap 성능 2배 정도 차이   

<br/><br/>

**쓰기 100% 성능 측정 테스트**     

![3](https://user-images.githubusercontent.com/103614357/214115123-4eae4b4d-2d5a-404e-88aa-60ab03bc7e78.png)   

- Caffeine Cache > ConcurrentLinkedHashMap > EhCache    
  - 마찬가지로, Caffeine Cache 와 ConcurrentLinkedHashMap 성능 2배 정도 차이 

<br/>

==> Caffeine Cache는 EhCache처럼 다양한 기능은 제공하지는 않지만,        
단순히 memory에 data caching하고 불러오는 작업만 한다면 성능이 가장 뛰어남!          
나는 상품 정보를 저장하고 불러오는 기능만 사용할거라 Caffeine Cache를 사용하기로 결정    

<br/><br/>

## Spring Cache Annotation

- `@EnableCaching` : Spring Application이 cache 사용할 수 있게 cache 기능 활성화
  - 내부적으로 Spring AOP 이용   

- `@Cacheable` : 캐싱할 수 있는 method 지정

- `@CacheEvict` : method 실행 시, 해당 cache 삭제

- `@CachePut` : method 실행에 영향을 주지 않고 cache를 갱신(update)해야 하는 경우 사용
  - 캐시의 값 유무에 상관없이 무조건 해당 로직을 실행시켜 나온 결과를 캐시에 저장
  - 보통 @Cacheable과 @CachePut은 같이 사용안함   
    (둘은 다른 동작을 하기 때문에, 실행순서에 따라 다른 결과가 나올 수 있음)
  - @CachePut은 캐시 생성용으로만 사용함  

<br/>

**속성**   
- key : 같은 캐시명을 사용 할 때, 구분되는 구분 값
  - cache는 근본적으로 key-value 구조를 가진 저장소
  - cache가 적용된 method들은 각각 cache에 접근하기 위한 적절한 키 생성 or 지정할 수 있음  
    - parameter 값이 default (캐싱 추상화는 simple KeyGenerator 사용해서 키 생성)  

```java
@Cacheable(cacheNames = "exampleStore", key = "#id")  
public A getA(String id) {
```

- parameter 값이 default이므로, 이 경우 key를 명시해주지 않아도됨  

<br/>
 
- `key = "#key"`인 이유  
  - SpEL 문법 사용
  - `#변수명`
- 객체안의 멤버변수를 비교해야 하는 경우
  `#객체명.멤버변수명`       

```java
@Cacheable(cacheNames = "exampleStore", key = "#b.id")
public A getA(B b) {
```

<br/>

- condition : 조건부여   

```java
@Cacheable(cacheNames = "exampleStore", key = "#b.id", condition = "#b.id.length() < 3")  
public A getA(B b) {
```

<br/><br/>

### SpringBoot에서 Caffeine Cache 사용하기

**1. dependency 추가**    

```
implementation 'org.springframework.boot:spring-boot-starter-cache'  
implementation 'com.github.ben-manes.caffeine:caffeine' 
```

<br/><br/>
    
**2. Cache 설정을 가진 Enum 생성**    
- Enum을 사용하여 캐시 이름, 만료 시간, 저장 가능한 최대 갯수 정의

```java
@Getter
@RequiredArgsConstructor
public enum CacheType {

	PRODUCT("product", ConstantConfig.DEFAULT_TTL_SEC, ConstantConfig.DEFAULT_MAX_SIZE);	
	PRODUCTS("products", ConstantConfig.DEFAULT_TTL_SEC, ConstantConfig.DEFAULT_MAX_SIZE);

	private final String cacheName;
	private final int expiredAfterWrite;
	private final int maximumSize;

	class ConstantConfig {
		static final int DEFAULT_TTL_SEC = 600;
		static final int DEFAULT_MAX_SIZE = 10000;
	}
}
```  

- 만료시간은 일단 10분으로 설정해두었는데, 조금 더 고민이 필요할듯,,    
  - 조금 더 길게 두자니, 잘 안쓰이는 데이터도 계속 쌓여있어 메모리를 차지할까봐 고민이 된다      

<br/><br/>

**3. Config 클래스**           
- @EnableCaching : cache 기능 활성화   
- CacheType에 등록한 cache들(PRODUCT, PRODUCTS)을 Caffeine cache 객체로 생성 후, SimpleCacheManager 객체에 등록     

```java
@EnableCaching
@Configuration
public class CacheConfig {

  @Bean
  public CacheManager cacheManager() {
    List<CaffeineCache> caffeineCaches = Arrays.stream(CacheType.values())
        .map(cache -> new CaffeineCache(
            cache.getCacheName(),
            Caffeine.newBuilder().recordStats()
                .expireAfterWrite(cache.getExpiredAfterWrite(), TimeUnit.SECONDS)
                .maximumSize(cache.getMaximumSize())
                .build()))
        .collect(Collectors.toList());

    SimpleCacheManager simpleCacheManager = new SimpleCacheManager();
    simpleCacheManager.setCaches(caffeineCaches);

    return simpleCacheManager;
  }
} 
```

<br/><br/>  

**4. 캐시 적용**     

- 사용할 method 위에 `@Cacheable(cacheNames="cacheName")`으로 캐시 이름 지정 -> 캐시 적용 가능      

```java
//ProductFacade.java   
@Cacheable(cacheNames = "product", key = "#productId")
@Transactional(readOnly = true)
public ProductGetResponse get(Long productId) {
  ProductGetFacadeResponse productGetFacadeResponse = productService.get(productId);
  List<String> imagePaths = imageService.getAll(productId, DomainType.PRODUCT);
  return toProductGetResponse(productGetFacadeResponse, imagePaths);
}
```

- key의 default가 parameter 값이기 때문에 key값은 productId지만 명시해주었다(아래의 getAll()과 통일성을 위해)           
  -> productId 값이 cache에 있는지 확인      
  -> 해당 값이 cache에 없으면 로직을 실행 후 반환값을 cache에 저장 / 있다면 cache에서 값 반환      
  
<br/>

```java
public record ProductGetAllRequest(
  Long cursorId,

  @Positive(message = "page size는 0 또는 음수일 수 없습니다.")
  int pageSize) {
}
```

```java
//ProductFacade.java  
@Cacheable(cacheNames = "products", key = "#productGetAllRequest")
@Transactional(readOnly = true)
public ProductGetAllResponses getAll(ProductGetAllRequest productGetAllRequest) {
  return productService.getAll(productGetAllRequest);
}
```

- cache의 key는 parameter가 default라고 해서, 처음엔 getAll()도 key를 지정해주지 않았는데 원하는대로 동작하지 않았다      
  - 같은 cursorId와 pageSize가 연달아 들어오는 경우만 caching이 이용되고, 중간에 다른 요청이 들어온다면 cache의 전체 내용이 그 요청에 대한 응답으로 변경되어, 첫번째 요청의 응답은 cache에 저장되어 있지 않았다       
  - 그래서 `key = "#productGetAllRequest"`로 key를 지정해주었더니 잘 동작했다      
    cursorId와 pageSize가 같은 요청이라면 select 쿼리가 나가지 않았다      
  - 참고) parameter 객체인 ProductGetAllRequest가 record class라 hashCode(), equals() 구현되어 있음         

<br/><br/> 

![KakaoTalk_20230124_132553857](https://user-images.githubusercontent.com/103614357/214212220-be2acecb-4649-46ad-b5c4-42f329769919.png)    

![KakaoTalk_20230124_132555170](https://user-images.githubusercontent.com/103614357/214212231-1cb5b90b-c270-483c-803c-59735b6126a0.png)   
   
- 한번 조회해두면 설정한 만료시간 동안 조회 쿼리가 나가지 않는 것을 확인함        
  - 당연하지만, 그동안 변경사항 적용도 안됨      
  - 변경사항이 적용이 되지 않는다면 정합성이 보장이 안될텐데?!   

  => 변화가 있을시, @CacheEvict로 직접 삭제해보자     

<br/>

#### Caffeine Cache에서의 Eventual Consistency   

```java
@CacheEvict(value = "products", allEntries = true)
@Transactional
public ProductRegisterResponse register(ProductRegisterRequest productRegisterRequest) {
  ProductRegisterResponse productRegisterResponse
      = productService.register(toProductRegisterFacadeRequest(productRegisterRequest));
  imageService.register(productRegisterRequest.images(), productRegisterResponse.id(), DomainType.PRODUCT);
  return productRegisterResponse;
}
```

- 상품에 변화가 생길시, 작업을 처리하는 server의 cache를 @CacheEvict 통해 직접 삭제함         

<br/><br/>

Reference    
https://jaehun2841.github.io/2018/11/07/2018-10-03-spring-ehcache/#Local-Cache-vs-Global-Cache   
https://github.com/ben-manes/caffeine/tree/master/wiki/throughput   
https://gosunaina.medium.com/cache-redis-ehcache-or-caffeine-45b383ae85ee    
https://velog.io/@_koiil/Caffeine     
https://wave1994.tistory.com/182      
https://sunghs.tistory.com/132    
https://blog.yevgnenll.me/posts/spring-boot-with-caffeine-cache     
https://velog.io/@soongjamm/Caffeine-Cache-%EB%A5%BC-%EC%A0%81%EC%9A%A9%ED%95%B4%EB%B3%B8-%EA%B2%BD%ED%97%98    
https://ykh6242.tistory.com/entry/Spring-Cache-Abstraction-%EC%A0%95%EB%A6%AC   

<br/>   
