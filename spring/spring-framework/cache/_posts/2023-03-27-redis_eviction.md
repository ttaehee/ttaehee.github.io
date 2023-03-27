---   
title: Spring) Redis cache 사용 시, eviction policy 고려해보기   
excerpt: Redis eviction policy에는 무엇이 있고, 왜 적용하지 않았는가           
---   

<br/>  

- 지난번 프로젝트에서도 RDB의 부하 감소와 조회 성능 개선을 위해 redis cache를 사용했었다    
  in-memory storage이므로 메모리 관리가 중요하다는 걸 알면서도    
  메모리 용량이 가득 찬 경우까지는 고려하지 못했다        
  - ttl을 설정해두긴 했지만, redis 서버에 캐시 데이터로 가득 차게 되면 어떻게 될까? 라는 의문이 들었다          
    어플리케이션의 전체적인 성능까지도 영향이 갈 수 있을 것이라 판단했다            
  - ttl을 더 짧게 가져가거나 메모리를 늘리는 등의 관리도 있지만 eviction policy를 도입하는 편이 가장 효율적이라고 생각했다      
    따라서 이러한 상황을 대비해 Redis에서 제공하는 Eviction 정책을 알아보았다   
    
<br/>

- 결론적으로, 정책들을 알아보았지만 적용은 하지 않았다    
  Eviction policy는 용량이 다 찼을 때 정책에 따라 기존 데이터를 지우게 되는데,    
  현재 프로젝트에서 redis를 데이터를 보관하는 용도로도 사용 중이기 때문이다     
  - 이번에 캐싱하는 데이터는 목록 조회의 첫번째 페이지만 캐싱하기 때문에 캐싱하는 데이터가 많지 않은데, eviction policy를 적용하여 보관해야 하는 데이터가 지워질 수 있는 위험 가능성을 안고갈 이유가 없었다      
  - 이번 프로젝트에 적용은 하지 않았지만, Eviction policy가 캐시서버를 운영하는데 있어 유용한 옵션이라는 점은 알 수 있었다       
  
<br/> 

## Redis Eviction Policy    
- Redis의 메모리 제한 사용량
  - 32bit 시스템 : default 3GB로 설정
  - 64bit 시스템 : 0으로 설정 = eviction policy를 사용하지 않음 = 메모리 사용량에 대해 상한선이 없음    

<br/>   

### 물리적인 메모리 용량 이상을 사용한다면? :: Swap Memory    

<br/>

![redis](https://user-images.githubusercontent.com/103614357/227793833-784b553d-95ef-48be-bc51-050b5d2205d2.png)   

- Swap 영역 : 실제 메모리 Ram 공간이 부족할 때, 디스크 공간을 이용해 부족한 메모리를 대체할 수 있는 공간  
  - 임시 보관하는 느낌   
  - 실제 디스크 공간을 메모리처럼 사용하는 개념이기 때문에 가상 메모리라고 할 수 있음   

<br/>

- 실제 물리 메모리의 크기보다 커져보임   
- but, 성능 저하 요인
  - 저장 장치에 접근하기 때문에 속도 느림     

=> 스왑 영역까지 사용하지 않도록 redis가 사용할 메모리의 용량을 제한해보자   

<br/><br/> 

### Redis의 maxmemory 옵션 지정   

- redis.conf 
  - redis.conf 내 maxmemory 옵션을 지정하여    
    maxmemory에 설정한 용량만큼 사용했다면, Eviction policy에 따라 데이터를 제거하여 설정한 용량 이하로 유지함

```
maxmemory 100mb
```

<br/><br/> 

### Redis Eviction Policy 종류    

새로 추가된 데이터의 용량을 확보하기 위해~       

**1. noeviction**   

- maxmemory에 도달하면 쓰기/삭제 작업 시 오류 반환

<br/>

**2. LRU**    

- 사용한지 가장 오래된 데이터부터 삭제
  - allkeys-lru : 최근에 사용하지 않은 키 제거
    - 키에 TTL 설정하면 -> 메모리를 더 소모함          
      -> allkeys-lru와 같은 정책을 사용할 경우, TTL을 설정하지 않고    
      메모리 부족 상태에서 키가 제거되도록하면 TTL을 설정하지 않아도 되기 때문에 효율성이 좋음
  - volatile-lru : TTL이 설정된 키들 중 최근에 사용하지 않은 키 제거
  
<br/>

**3. Random**    

- 무작위로 키 제거
  - allkeys-random : 무작위로 키 제거
  - volatile-random :  TTL이 설정된 키들 중 무작위로 키 제거

<br/>

- 모든 키들의 사용이 비슷하다고 판단된다면 allkeys-random 사용 권장   

<br/>

**4. TTL**    

- TTL이 짧은 데이터부터 삭제
  - volatile-ttl : TTL이 짧은 키 제거

<br/>

**5. LFU**   

- 사용 빈도수가 가장 적은 데이터부터 삭제
  - LRU와 차이점 : 최근에 사용된 데이터라도 자주 사용되지 않는다면 삭제될 수 있음     
  - allkeys-lfu : 사용빈도수가 가장 적은 키 제거
  - volatile-lfu : TTL이 설정된 키들 중 사용빈도 수가 적은 키 제거 

<br/><br/>

- redis.conf 내 maxmemory-policy 옵션에 지정

```
maxmemory-policy allkeys-lru
```

<br/>

- 참고)    
  volatile ~ 정책의 경우, TTL이 설정된 키 중에서 삭제할 키를 찾기 때문에     
  TTL이 설정된 키가 없는데 maxmemory만큼 용량이 도달했다면 noeviction처럼 오류를 반환함      

<br/><br/>

### LRU 알고리즘과 LFU 알고리즘의 기본 로직

- LRU (Least Recently Used)   

![redis-lru](https://user-images.githubusercontent.com/103614357/227794753-d79ed5b3-afc9-4906-804d-33ab4523a83e.png)

<br/><br/>

- LFU (Least Frequently Used)     

![redis-lfu](https://user-images.githubusercontent.com/103614357/227794730-6c46d5fb-08a0-45ab-abc0-2e62a9c4d169.png)

<br/><br/>

Reference    
[Key eviction | Redis](https://redis.io/docs/reference/eviction/)      
[Redis Eviction 정책을 적용하여 효율적인 캐시 띄우기](https://chagokx2.tistory.com/102)    
[리눅스 : Swap 메모리란?](https://jw910911.tistory.com/122)   
  
<br/>
