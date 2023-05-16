---
title: Spring) 외부 API 호출을 위해 RestTemplate 대신 Open Feign 사용하기
excerpt: spring 내부에서 http client 도구로 외부 API 호출 쉽게 호출하기
---

## Spring 내부에서 외부 API 호출하기 :: OpenFeignClient 사용

`지역별 미세먼지 농도를 확인할 수 있는 서비스`를 구현하기 위해 외부 api를 호출해야 했다    
Spring 내부에서 외부 api를 호출하는 경우 **RestTemplate을 사용**해왔는데 **이번에는 FeignClient를 사용**해보았다    
RestTemplate과 달리 **annotation과 interface 기반**이라 코드작성도 줄고 훨씬 편할거라 예상했고 실제로도 그랬다        

<br/>

## OpenFeignClient 설정하기    

- dependency

```
implementation 'org.springframework.cloud:spring-cloud-starter-openfeign:3.1.6'
```

<br/>

- config    

```java
@Configuration
@EnableFeignClients("com.taehee.dust.common.feign")
public class OpenFeignConfig {

}
```   

SpringBoot test 시 해당 설정이 활성화되지 않도록 `@EnableFeignClients`을 main class에 붙이지 않고 configuration 파일로 분리했다         
이처럼 따로 파일로 설정 시에는 **feign 인터페이스의 위치를 명시**해주어야 한다    

<br/>

### api 호출을 수행할 client

OpenFeign으로 보낼 api는 [공공 api](https://www.data.go.kr/data/15073861/openapi.do)를 사용하였다      

- application.yml과 properties class

```
api:
  serviceKey: {servicekey}
  url: {url}
  returnType: json
  numOfRows: 100
  pageNo: 1
  ver: 1.3
```

```java
@Getter
@Component
public class OpenApiProperties {
    @Value("${api.serviceKey}")
    private String serviceKey;

    @Value("${api.returnType}")
    private String returnType;

    @Value("${api.numOfRows}")
    private String numOfRows;

    @Value("${api.pageNo}")
    private String pageNo;

    @Value("${api.ver}")
    private String ver;
}
```

공공 api는 받을 데이터를 **xml과 json 중 선택**할 수 있어 json으로 명시해주었다  
호출에 필요한 정보들은 yml에 적어두고 properties class로 만들어 관리하였다   

<br/>

- api 호출을 수행할 client 구현

```java
@FeignClient(name = "ParticulateMatterClient", url = "${api.url}", configuration = OpenFeignConfig.class)
public interface ParticulateMatterClient {

    @GetMapping
    ParticulateMatterResponse call(@RequestParam(value = "serviceKey") String serviceKey,
                                   @RequestParam(value = "returnType") String returnType,
                                   @RequestParam(value = "numOfRows") String numOfRows,
                                   @RequestParam(value = "pageNo") String pageNo,
                                   @RequestParam(value = "sidoName") String sidoName,
                                   @RequestParam(value = "ver") String ver);
}
```

api 호출을 수행할 client는 interface에 `@FeignClient` annotation을 붙여주면 된다     
annotation 기반이라 정말 편했다 그치만 제공하는 기본 설정이 적은 느낌?    
테스트 관련해서 제공하는게 없어서 이 부분은 직접 구현했다     
그렇다하더라도 난 RestTemplate보다 OpenFeignClient가 훨씬 좋았다 일단 굉장히 편하다         

<br/><br/>

### Feign logging 설정     

Feign은 남길 로그에 따라 [4가지 수준](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/#feign-logging)을 제공한다    

- `NONE` : default, 로깅하지 않음
- `BASIC` : 요청 메소드와 uri & 응답 상태와 실행시간 로깅
- `HEADERS` : 요청과 응답 header, 기본 정보들 로깅
- `FULL` : 요청과 응답 header, body, meta data 로깅     

<br/>

OpenFeign 설정 클래스에 `Logger.Level`을 빈으로 등록해주어도 되지만 나는 yml에서 해주었다    

```
feign:
  client:
    config:
      default:
        loggerLevel: full

logging:
  level:
    com.taehee.dust.common.feign: DEBUG
```

Feign은 **DEBUG level로만 log를 남길 수** 있어 log level은 DEBUG로 설정했다         

<br/><br/>

### OpenFeign 부분만 테스트 해보기 :: @OpenFeignTest 구현  

현재 프로젝트에서 외부 api 호출 시 다른 로직들과 결합되어 호출하고 있다          
이 부분에서 **OpenFeign 부분만 테스트**할 수 있다면 효율적일텐데 OpenFeign은 테스팅 도구를 지원하지 않으므로, `@OpenFeignTest`를 직접 만들어보았다    

<br/>

- feign 테스트에 필요한 클래스들을 포함하는 `FeignTextContext` class

```java
@ImportAutoConfiguration({
        OpenFeignConfig.class
        FeignAutoConfiguration.class,
        HttpMessageConvertersAutoConfiguration.class
})
public class OpenFeignTestContext {
}
```

<br/>

- 위의 설정들을 메타 정보로 사용하는 `@OpenFeignTest` annotation

```java
@SpringBootTest(
        classes = {OpenFeignTestContext.class},
        properties = {
                "spring.profiles.active=local"
        })
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface OpenFeignTest {
}
```

<br/>

- `@OpenFeignTest`를 적용한 실제 테스트 코드

```java
@OpenFeignTest
@ImportAutoConfiguration(OpenFeignConfig.class)
public class ParticulateMatterClientTest {

    @Test
    @DisplayName("open api를 호출하여 미세먼지 정보를 가져올 수 있다.")
    void call() {
        ...
    }
}
```

<br/>

테스트 뿐 아니라 실제 로직에서도 호출이 잘 되었다    

<img width="458" alt="스크린샷 2023-05-16 오후 6 51 06" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/f9981645-abcf-48f1-934a-0daa3a70210f">

<br/><br/>

Reference      

[Spring Cloud OpenFeign 공식 문서](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/)         
[OpenFeign 타임아웃(Timeout), 재시도(Retry), 로깅(Logging) 등 설정하기](https://mangkyu.tistory.com/279)     

<br/>
