---
title: Spring) refresh token을 저장하기에 Redis가 적절하다고 생각한 이유
excerpt: Spring Security & JWT & Redis를 활용한 토큰 기반 인증 구현하기
---

<br/> 

- [참고) 토큰 기반 로그인 및 access token, refresh token 발급 구현 정리](https://ttaehee.github.io/spring/spring-framework/spring-security/jwt_login/)

- 사실 JWT와 같은 Claim 기반 토큰을 사용하면 refresh token을 서버쪽에서 꼭 저장할 필요는 없지만,     
  토큰 탈취시 대응하겠다는 취지로 강제 로그아웃, 유저 차단이 가능하도록 서버쪽에서 refresh token을 저장하도록 구현했다    
  처음에는 데이터베이스에 저장했는데 스케줄러 등을 사용해 주기적으로 refresh token을 만료처리 해주어야 하는 과정이 서버의 자원을 사용하는 데에 있어 비효율적이라고 생각했고 redis를 사용했다   

- 내가 이렇게 시큐리티, 토큰에 진심이 될줄은 몰랐다   
  파도파도 끝이 없는 관련 정보들    

<br/>

## Refresh Token이 필요한 이유          

- access token은 한번 발급되면 만료 전까지 삭제가 불가능     
  발급 후, 서버에 저장해두지 않고 토큰 자체로 검증하며 사용자 권한을 인증함       
  - 이런 역할을 하는 access toke이 탈취되면?     
    토큰 만료 전까지, 토큰을 획득한 사람은 누구나 권한 접근이 가능해짐     
    
=> 따라서 access token의 유효기간을 짧게 설정하고 refresh token으로 이 문제를 해결해보자     

- refresh token은 접근에 대한 권한을 주는 것이 아니고 access token 재발급에만 관여    
  - 로그인 시 발급되고 저장소에 저장하여 관리함    
  - 로그아웃 시 저장소에서 삭제하여 사용이 불가능하도록 함   

<br/><br/>

### refresh token 저장을 Redis에 한 이유와 고민했던 점        
- 일단 redis는 데이터 저장의 기간을 정하기 매우 편하다   
  refresh token은 일정 시간 이후 만료되어야 한다는 점에서 redis의 데이터 유효기간(time to live)을 지정할 수 있다는 점이 좋았다     
- 두번째, access token의 만료 기간을 짧게 잡는 만큼 refresh token에 접근할 일이 꽤 많을텐데    
  redis는 in-memory로 데이터를 관리하기 때문에 
  빠른 접근이 가능해 refresh token 발급 과정에 병목이 되지 않을 것이라 판단했다   
  
<br/>

- 다만 in-memory 기반이다보니 데이터의 저장에 관해서는 안전하지 않을 수 있다         
  redis의 문제로 인해 저장한 refresh token 정보들이 휘발될 경우 최악의 경우 모든 회원들이 로그아웃된다      
  - 치명적인 큰 문제인가? 생각했을 때, 서비스를 제공하는 입장에서 쉽게 아니다!라고 말하긴 어렵다         
  	일단 매 요청마다 redis에 접근해야하는 경우라면 큰 문제라고 생각이 든다         
	그렇지만 지금 경우에는 access token이 만료된 경우에만 접근하기 때문에 위의 장점들의 메리트가 더 크다고 생각했다           

<br/><br/>

## access token과 refresh token 플로우   

refresh token은 access token의 보안을 위해 나온 개념이기 때문에 명확하게 정해진 정보는 없다       

<br/>

- [Oauth 2.0 RFC 문서](https://www.ietf.org/rfc/rfc6749.txt)       

  ![rfc_token](https://user-images.githubusercontent.com/103614357/223489071-c6b44878-e9a0-4710-8a03-63a9f28f4849.png)


  ![rfc_token](https://user-images.githubusercontent.com/103614357/223489484-0184736c-fcc1-45e5-9dbf-f31e7559dd88.png)

해당 문서를 참고해보면     
재발급 시, access token만 재발급 하는 경우와, access token과 refresh token 둘 다 재발급하는 경우가 있을 수 있다고 한다    
나는 현재 access token과 refresh token 둘 다 재발급하도록 구현했다    
계속해서 보안성을 가져가면서 무한 로그인이 유지되도록    

<br/><br/>

### Access Token 재발급 플로우 

이것도 두가지 방법이 있다     
1. 요청할 때마다 access token과 refresh token을 같이 넘기기
2. 서버에서 Access Token이 만료되었다고 응답하면 재발급 API 호출해서 Refresh Token으로 요청하여 재발급 받기   

<br/>

- 나는 두번째 방법으로 구현했다   
  - 요청 시마다 access token과 refresh token을 같이 넘기면 refresh token도 마찬가지로 탈취 위험이 높아진다고 생각했다   
    물론 refresh token은 탈취 시 해결방법이 있지만, 가능성은 줄이는게 좋다고 생각했다   
  - API 호출이 더 많이 일어나겠지만 안전함을 고려했을 때 충분히 감안할 수 있는 정도라고 판단했다 

<br/><br/>

- 만료된 access token으로 요청 시 서버쪽에서 만료되었다고(http status code 401 + 만료된 토큰이라는 에러 메시지) 알려줌
  - access token이 검증되지 않는다는건 인증되지 않은 사용자와 같다고 생각하여 401 code 사용    
- client에서 가지고 있던 access token과 refresh token을 같이 보내며 재발급 요청     
  - access token에 담긴 회원 정보로 redis에서 refresh token 정보 가져와서 client에서 보낸 refresh token과 비교     
    - 검증되었다면 새로운 access token과 refresh token 발급해서 응답    
    - 없다면 만료되었다는 의미일테니, 마찬가지로 인증되지 않은 사용자와 같다고 생각하여 401 code를 사용했다  

<br/><br/>

## redis의 BGSAVE 관련 에러 해결

프론트에서 갑자기 로그인이 안된다며 서버가 돌아가고있는지 확인해달라고 했다    
오잉..!    
확인해본 바 서버는 잘 돌아가고 있는데 로그를 확인해보니 레디스 쪽이 문제인듯 했다    

```
io.lettuce.core.RedisCommandExecutionException: MISCONF Redis is configured to save RDB snapshots
```

<br/>   

[해결에 참고한 글](https://charsyam.wordpress.com/2013/01/28/%EC%9E%85-%EA%B0%9C%EB%B0%9C-redis-%EC%84%9C%EB%B2%84%EA%B0%80-misconf-redis-is-configured-to-save-rdb-snapshots-%EC%97%90%EB%9F%AC%EB%A5%BC-%EB%82%B4%EB%A9%B0-%EB%8F%99%EC%9E%91%ED%95%98%EC%A7%80/)   

- Redis는 Persistent를 위해 BGSAVE로 RDB를 만들어냄   
  BGSAVE 실패하면 write 명령어를 전부 거부함    
  (어떤 경우에 실패하는거지? 다양한 이유가 있다고 함 vm_overcommit_memory 정책문제 or 디스크 이슈 등)         
  - redis를 데이터 저장용으로 사용 중이기 때문에 rdb를 끌 순 없었다
  - 따라서 해당 옵션을 비활성화했다  `config set stop-writes-on-bgsave-error no`   

  ```
  [root@localhost ~]# redis-cli
  127.0.0.1:6379> config set stop-writes-on-bgsave-error no
  OK
  127.0.0.1:6379> quit
  ```
  
<br/>

- stop-writes-on-bgsave-error
	- 백그라운드에서 RDB로 데이터를 저장할 때 오류 발생에 대한 처리를 설정
	- yes : RDB에 데이터를 저장하다 실패하면 모든 쓰기 요청을 거부함  
	- no : 쓰기 요청은 처리하지만 RDB에 데이터가 저장되지 않음   

<br/>

**참고**   

- Redis는 백업을 위해 RDB 방식과 AOF 방식 지원
	- RDB 저장 방법
		- SAVE or BGSAVE 명령어로 RDB 저장함  
		- save 명령 : redis에서 클라이언트 요청 못받음    
		- bgsave 명령 : 자식 프로세스를 생성하여 백그라운드에서 메모리에 있는 데이터를 RDB로 저장함
			- 백그라운드로 수행하기 때문에 클라이언트 요청 처리 가능   
			  -> save보다 bgsave 명령어 통한 저장 권고

<br/><br/>

## 구현하기   

- KeyValueRepository.java

```java
public interface KeyValueRepository {
	void save(String key, String value, long expireSeconds);

	String get(String key);

	void delete(String key);
}
```

- RedisRepository.java

```java
@Component
@RequiredArgsConstructor
public class RedisRepository implements KeyValueRepository {

	private final StringRedisTemplate stringRedisTemplate;

	@Override
	public void save(String key, String value, long expireSeconds) {
		ValueOperations<String, String> valueOperations = stringRedisTemplate.opsForValue();
		valueOperations.set(key, value, Duration.ofSeconds(expireSeconds));
	}

	@Override
	public String get(String key) {
		ValueOperations<String, String> valueOperations = stringRedisTemplate.opsForValue();
		return valueOperations.get(key);
	}

	@Override
	public void delete(String key) {
		stringRedisTemplate.delete(key);
	}
}
```   

- service layer   

```java
@Transactional
public MemberLoginResponse reissueMember(MemberReissueRequest memberReissueRequest) {
    String accessToken = memberReissueRequest.accessToken();
    String refreshToken = memberReissueRequest.refreshToken();
    jwtTokenProvider.validateToken(refreshToken);
    String memberId = jwtTokenProvider.extractMember(accessToken);

    if (!redisRepository.get(memberId).equals(refreshToken)) {
      throw new InvalidRefreshTokenException("인증 정보가 만료되었습니다.");
    }

    redisRepository.delete(memberId);
    WumoJwt wumoJwt = getWumoJwt(memberId);

    return toMemberLoginResponse(wumoJwt);
}

private WumoJwt getWumoJwt(String memberId) {
    WumoJwt wumoJwt = jwtTokenProvider.generateToken(memberId);
    redisRepository.save(
      memberId,
      wumoJwt.getRefreshToken(),
      jwtTokenProvider.getRefreshTokenExpireSeconds()
    );
    return wumoJwt;
}
```

- 현재는 redis를 사용하지만, 추후에는 변경될 수 있으니(변경 가능성은 적지만) 부모 클래스를 KeyValueRepository로 지어보았다    
  변수 네이밍은 어려워   
  
	![KakaoTalk_20230306_230214406](https://user-images.githubusercontent.com/103614357/224497169-0b9f1730-175a-4f6c-9d3a-3b21e8f82b21.png)	

<br/>

- RedisTemplate 그냥 썻다가   

	![KakaoTalk_20230306_230215780](https://user-images.githubusercontent.com/103614357/224497618-c8573b80-3534-470e-b188-8faf3b38b2cb.png)
	
	- 이렇게 나왔다 직렬화 부분만 String으로 바꾸어줘도 되지만 제공해주는 StringRedisTemplate으로 변경했다      

<br/>

### 패키지 고민   
redis에 접근하는게 추후에는 Member domain뿐이진 아닐것 같기도 하고   
도메인 정보를 가지고 있느냐 생각했을때 아니기 때문에              
redis 관련 클래스들은 global 패키지 안에 두었다      

<br/><br/>

## 글로벌 회사들의 토큰 정책   

- 내가 구현한 것처럼 access token 발급할 때, refresh token도 새로 발급하는 정책   
- refresh token을 아예 사용하지 않는 정책(애플의 토큰 정책)
- refresh token을 사용하지 않으면서 access token을 short-term, long-term 2가지로 분리하는 정책(페이스북의 토큰 정책)
- 서비스에서 관리하는 고유한 키값으로 token을 재발급 받는 정책 [구글의 토큰 정책](https://developers.google.com/identity/protocols/oauth2/web-server?hl=ko#httprest_2)

<br/><br/>

## 나중에 참고하기 위해 정리해두기) EC2에 Redis 설치하기     

[참고한 글](https://small-stap.tistory.com/109?category=989595)      

<br/>
  
**1. linux update 하고 gcc make 설치**      

```
sudo yum update -y
sudo yum install gcc make -y
```

<br/>
  
**2. Redis 설치파일 다운로드 및 압축풀기 + gcc make로 컴파일**    

```
wget http://download.redis.io/releases/redis-6.2.5.tar.gz
tar xzf redis-6.2.5.tar.gz
cd redis-6.2.5
make
```

<br/>

**3. 디렉토리 생성 및 Redis 설정 파일 복사**    

```
sudo mkdir /etc/redis
sudo mkdir /var/lib/redis
sudo cp src/redis-server src/redis-cli /usr/local/bin/
sudo cp redis.conf /etc/redis/
```

<br/>

**4. redis.conf 설정파일 수정**   

```
sudo vi /etc/redis/redis.conf
```

- bind 속성 : 내가 bind 하고자 하는 IP로 변경
- daemonize 속성 : no에서 yes로 변경 (Redis를 백그라운드에서 실행하기 위해)
- logfile : redis log 파일 저장할 위치 설정
- working directory

<br/>

**5. Redis 실행**   

```
redis-server
```
  
다른 작업을 위해 exit하면 Redis 서버 종료됨 = Redis 실행하면서 다른 작업을 수행할 수가 없음        

=> 백그라운드에서 Redis 실행가능하게 하도록 하기   

<br/><br/>

### Redis 백그라운드에서 실행하기   

**1. Redis 서버 초기화 스크립트 다운로드**     

[saxenap github : install-redis-amazon-linux-centos](https://github.com/saxenap/install-redis-amazon-linux-centos/blob/master/redis-server)

```
sudo wget https://raw.github.com/saxenap/install-redis-amazon-linux-centos/master/redis-server
```

<br/>

**2. 다운 받은 파일 /etc/init.d로 옮기고 권한 설정**   

```
sudo mv redis-server /etc/init.d
sudo chmod 755 /etc/init.d/redis-server
```

<br/>

**3. redis-server 파일의 redis 변수값 체크**   

```
sudo vim /etc/init.d/redis-server
```

<br/>

**4. redis-server Auto-Enable 설정**    

```
sudo chkconfig --add redis-server
sudo chkconfig --level 345 redis-server on
```

<br/>

**5. redis 실행**    

```
sudo service redis-server star
```

<br/>

**6. redis 접속**   

```
redis-cli
127.0.0.1:6379>
```

<br/>
