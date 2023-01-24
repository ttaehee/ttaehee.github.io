---   
title: Spring) Image upload
excerpt: SpringBoot project에서 이미지 파일 업로드하기(Local)       
---   

<br/>   
     
이번주는 팀 프로젝트 구현 기간이었다!           

<br/>    

## 일기(회고) 
- 이번 우리팀의 스프린트 목표는 `유저스토리에 초점을 맞추어 핵심 비즈니스 로직에 대한 mvp모델을 제작한다`로 정했다     
  따라서 상품 파트에서 기본 CRUD 기능에 초점을 맞추어 구현하였다    
  - C에서는 이미지 업로드를 처음 다뤄보았기 때문에 공부 + 구현에서 조금 시간이 걸렸고,     
    R에서는 목록 조회 기능에 QueryDSL을 처음 사용해 No offset pagination을 공부 + 구현하느라 시간이 조금 걸렸고    
    TestContainers에서도 시간을 꽤 잡아먹었고(로컬 DB로 들어가버림, 조금 허무했지만 해결 완)          
    팀원들이랑 코드 리뷰하고, 컨벤션이나 플로우도 계속 상의했다             
    U,D는 아직 구현 전이지만, 두가지는 금방 할듯하다   
    
<br/>

- 구현을 위해서 공부한 것들, 알아본 것들 등이   
  지금은 잘 기억이 나지만 영원히 기억할리 없으니   
  설 기간동안은 구현을 멈추고 정리할 예정     

<br/>

- 이번주에 영상을 보지 않았고, 알고리즘을 풀지 않았고,,    
  오로지 플젝 구현뿐이었다(코딩만 하고싶,,)    
  내가 생각한 기능들이 내가 짠 코드들로 인해 결과물로 나오니까    
  다른것도 더 결과물을 보고싶고 그러다보니 공부하고 구현하고      
  다른건 계속 미루게되고(변명)     
  - 사실 구현을 위한 공부면 정말 나에게 도움이 되는 공부가 맞지만   
    나는 취준도 해야하기 때문에!  
    나의 할일에는 코테준비 + 면접준비도 있다는 사실..! 잊지말자      
  - 설 기간동안은 정리 + 모던자바인액션 책 + 알고리즘 (+ 구현은 쉬기)     
    지난번 플젝처럼 너무 기능구현에만 집중하며 달리는 느낌이 들어 지난주 내용을 정리할겸 쉬려한다   

<br/><br/>

## Image(File) Upload     

[김영한님 강의](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-mvc-2#curriculum)를 참고해서 구현했다   

<br/> 

상품을 등록하려면 상품의 정보와 이미지를 같이 전송하고 받아야함     
-> 문자열 데이터와 파일을 동시에 보내기 적합한 Content-Type은 무엇일까?     
=> Multipart/form-data!           
  
<br/>

### Multipart/form-data     

HTTP는 multipart/form-data라는 전송방식 제공   
- 문자, 바이너리 동시에 전송가능 
  - 다른 종류의 여러파일과 폼의 내용 함께 전송 가능     
  
=> part들로 나누어져있음    

<br/>  

```
POST /save HTTP/1.1
Host: localhost:8080
Content-Type: multipart/form-data; boundary=-----%%%
Content-Length: 12309

-----%%%
Content-Disposition: form-data; name="username"

steve
-----%%%
Content-Disposition: form-data; name="age"

25
-----%%%
Content-Disposition: form-data; name="file"; filename="profile.png"
Content-Type: image/png

2q432#@$%#%#@REWFfsf3e3f3@4ewR$f4r4wF4gs4t4obvy734or84...
-----%%%
```   

- 폼 항목마다 Content-Disposition header가 붙음 + 부가정보가 붙어있음   
  - 모든 항목은 임의로 생성되는 boundary(`-----%%%`)를 통해 구분됨
    - 이 boundary는 UUID로 매번 임의로 생성     
  - Part : boundary로 구분된 각 항목   
- file의 경우 : Content-Type 추가 + 바이너리 데이터가 전송됨   


<br/><br/>   

#### Servlet이 제공하는 Part         
- HttpServletRequest에 여러 part들로 옴          
  - HttpServletRequest에서 parts 빼서 part 하나씩 분리해서       
    하나의 part에 담긴 filename과 data읽어서 스트림으로 저장 ... (복잡)     

<br/>
    
- `spring.servlet.multipart.enabled = true`   
  -> DispatcherServlet에서 MultipartResolver 실행      
  -> multipart 요청인 경우,    
    Servlet Container가 반환하는 일반적인 요청인 `HttpServletRequest`를    
    `MultipartHttpServletRequest` (HttpServletRequest 상속, multipart 관련 추가기능 제공)로 변환해서 반환함    
    (MultipartHttpServletRequest의 구현체 : `StandardMultipartHttpServletRequest`)   
    -> Controller에서 MultipartHttpServletRequest 주입받을 수 있음     
    = multipart 관련처리 편하게 할 수 있음      
  - but Spring이 제공하는 MultipartFile이 더 편함!    

<br/><br/>     

- 결론   
  - Servlet이 제공하는 Part
    - HttpServletRequest 사용해야함
    - 여러 part에서 파일 부분만 구분하려면 여러 코드를 넣어야함       
      
  => Spring이 편리하게 제공하는 MultipartFile를 사용해보자   

<br/><br/>

#### Spring이 제공하는 MultipartFile    
Spring은 MultipartFile이라는 인터페이스로 multipart 파일을 매우 편리하게 지원    
- ex) `@RequestParam MultipartFile multipartFile`        
- filename(client가 업로드한 파일명)만 빼내는거 : `multipartFile.getOriginalFilename()`        
- 저장하는거 : `multipartFile.transferTo(new File(경로))`   

<br/><br/>

## MultipartFile 사용해서 Image Upload 구현하기   

### application.yml    

```
spring:
  servlet:
    multipart:
      maxFileSize: 5MB
      maxRequestSize: 20MB

image.path: /Users/taehee/IdeaProjects/shoe-kream/src/main/resources/static/images/   
```

- `spring.servlet.multipart.max-file-size` : 최대 총 파일 사이즈  
- `spring.servlet.multipart.max-request-size` : multipart/form-data의 최대 request size   
- 경로 끝에 필수로 `/` 붙게하기    
  
<br/><br/>  

### Controller layer   

- Request Dto

```java
public record ProductRegisterRequest(
  @NotBlank(message = "상품 이름은 필수 입력사항입니다.")
  @Size(max = 30, message = "상품 이름은 {max}글자 이하로 입력할 수 있습니다.")
  String name,

  @NotNull(message = "출시 가격은 필수 입력사항입니다.")
  @Positive(message = "출시 가격은 0 또는 음수일 수 없습니다.")
  int releasePrice,

  @NotBlank(message = "상품 설명을 입력해주세요.")
  String description,

  List<@ShoeSize(message = "잘못된 형식의 신발 사이즈가 입력되었습니다.") Integer> sizes,

  List<MultipartFile> images) {
}
```

<br/>   

```java
@PostMapping
@ResponseStatus(code = HttpStatus.CREATED)
public ApiResponse<ProductRegisterResponse> register(
    @ModelAttribute @Valid ProductRegisterRequest productRegisterRequest) {

  return ApiResponse.of(productFacade.register(productRegisterRequest));
}
```

- 처음에는 `@RequestPart`를 사용해서 JSON 데이터와 이미지 파일을 따로 받도록 작성했었다    
  그렇지만, 그러기에는 JSON 데이터부분만 content-type을 application/json으로 설정해주어야 했고, 그렇게 되면 multipart/formdata가 아니라는 생각이 들었다   
  찾아보니 front 단에서도 content type 2개를 보내주는 경우는 없다고 하기도 하고       
  그래서 multipart가 name-value 형식으로 오니까   
  문자열 데이터와 이미지 파일을 하나의 DTO로 묶어 `@ModelAttribute`로 받아보았더니 잘 작동이 되었다      
   
<br/>

- @ModelAttribute와 @RequestBody의 차이를 가볍게만 알고있어, 이번에 조금 자세히 알아보았다   
- 둘 다 client 측에서 보낸 data를 Java code에서 사용할 수 있는 object로 만들어준다, 둘의 차이는?   
  - 세부 수행 동작에서 큰 차이

<br/><br/>

### @ModelAttribute와 @RequestBody   

**@ModelAttribute**   
- HTTP parameter data를 Java 객체에 mapping   
- Form 형식의 HTTP 요청 본문 data 인식해 mapping
  - JSON 형태의 data는 binding 되지 않음 -> @RequestBody   
- 따라서 객체의 field에 접근해 data를 binding할 수 있는 생성자 or setter() 필요  
  - 적절한 생성자를 먼저 찾고, 그 뒤에 바인딩되지 않은 값을 setter() 통해 binding해주는 순서로 동작함   

<br/>   

=> 정리 : Query String or Form 형식 데이터 처리  

<br/><br/>

**@RequestBody**       
요 아이는 정리먼저 해보자면,   

- 요청 본문의 JSON, XML, Text 등의 data가 적합한 HttpMessageConverter를 통해 parsing되어 Java 객체로 변환됨     
- @RequestBody를 사용할 객체는 field를 binding할 생성자 or setter() 필요 없음   
  - 다만 `직렬화`를 위해 `기본 생성자`는 필수
  - 또한 data binding을 위한 field명을 알아내기 위해 getter() or setter() 중 1가지는 필수 정의      

<br/>

- 어떻게 기본 생성자만으로 JSON 값을 Java 객체로 재구성할 수 있는거지?      
  - Spring에 등록된 여러 MessageConverter 중 MappingJackson2HttpMessageConverter를 사용함
  - 이 때, 내부적으로 ObjectMapper를 통해 JSON 값을 Java 객체로 역직렬화함   
    - 역직렬화란 생성자를 거치지 않고 reflection을 통해 객체를 구성하는 메커니즘   
  - 직렬화 가능한 클래스들은 기본 생성자가 항상 필수
    - 따라서 @RequestBody에 사용하려는 RequestBodyDto에 기본 생성자가 없다면 데이터 바인딩 실패   

<br/>

- 그치만 기본생성자에는 field명이 없는데?       
  ObjectMapper는 어떻게 JSON의 Key를 Java 객체의 field명과 mapping시켜 값을 대입하는걸까     
  
	![제목 없음](https://user-images.githubusercontent.com/103614357/213904116-49765ea9-d34e-44db-b78b-28f024b9cf7e.png)      

- [공식 문서](https://jenkov.com/tutorials/java-json/jackson-objectmapper.html#how-jackson-objectmapper-matches-json-fields-to-java-fields)
  - Jackson ObjectMapper : JSON object의 field를 Java object의 field에 mapping할 때 getter() or setter() 사용
    - getter나 setter 메서드 명의 접두사(get, set)를 지우고, 나머지 문자의 첫 문자를 소문자로 변환한 문자열을 참조하여 필드명을 알아냄    
  - RequestBodyDto에 getter(), setter() 둘 다 없으면, HttpMessageNotWritableException 예외 발생  

<br/><br/>

- 참고) Reflection
  - run-time에 동적으로 class들의 정보를 알아내고 실행할 수 있는 것  
  - 일반적인 객체 생성은   
    - compile-time에 class와 method가 compile되어 JVM의 메모리 영역에 올라옴 (relfection은 run-time에 되는 것)      
  - Java는 compile 시, Class Loader가 동적로딩으로 byte code를 JVM의 method영역에 class의 정보 로드함         
    -> 이미 메모리에 로드되어 있으니까 run-time 시점에 class 정보 가져오고, class의 instsance를 생성하고, 접근 제어자와 관계 없이 field와 method에 접근하여 필요한 작업 수행 가능        
    = run-time(프로그램이 실행 중)에 객체의 type을 몰라도 특정 class의 field 또는 method를 호출할 수 있음 (그래서 binding이 가능하구낭)        
    - 단점 : 캡슐화 저해  

<br/><br/>

- [Java record class와 Jackson](https://dev.to/brunooliveira/practical-java-16-using-jackson-to-serialize-records-4og4)       
  - Jackson 2.12부터 record class(Immutable Object) 직렬화 및 역직렬화 가능    
  - 현재 내가 record class를 사용중인데, binding이 잘 된 이유  

<br/><br/>  

### Service layer  

```java
@Service
public class ImageLocalService implements ImageService {

	private final String uploadPath;

	private final ImageRepository imageRepository;

	public ImageLocalService(@Value("${image.path}") String uploadPath, ImageRepository imageRepository) {
		this.uploadPath = uploadPath;
		this.imageRepository = imageRepository;
	}
```

- @Value 는 application.yml 의 값을 가져와서 주입
- 생성자 안에서 @Value 한 이유   
  - 처음에 field로 `@Value("${image.path}") String uploadPath`해서 property값을 변수에 주입하려 했는데 실행해보면 NPE     
  - 확인해보니 순서가 injection을 모두 마친후에 @Value를 불러오기 때문 

<br/>

### @Value와 Spring Bean의 생명주기   
 
- 왜일까? 스프링 빈의 생명주기와 관련해서 생각해보자        
  - 스프링 컨테이너 생성 -> 스프링 빈 생성 -> 의존관계 주입 -> 초기화 콜백 -> 사용 -> 소멸전 콜백 -> 스프링 종료
  - `생성자 주입` : 생성 시점에 의존성 모듈을 찾지 못하면 빈 생성을 하지 못함   
    - 스프링 빈 생성시점에 동작   
  - `field 주입` : field에 @Autowired를 붙이면 주입
    - 의존성 관계 주입 시점에 동작(런타임)
  - `Setter 주입(method를 통한 주입)` : method에 @Autowired를 붙이면 주입
    - 의존성 관계 주입 시점에 동작(런타임)
   
<br/>  
   
=> @Value는 field 주입과 동일하게 의존관계 주입 시점에 동작   
= 생성자 호출 시점 이후   

=> @Value를 갖는 변수를 생성자의 인자로 받는 방향으로 해결  

<br/><br/>    
  
```java
@Override
public void register(List<MultipartFile> multipartFiles, Long referenceId, DomainType domainType) {
  if (multipartFiles != null && !multipartFiles.isEmpty()) {
    List<Image> images = uploadImages(multipartFiles, referenceId, domainType);
    imageRepository.saveAll(images);
  }
}
```
- List<MultipartFile>가 null이 아니고 empty 아닐때만 ImageService를 타게할까 하다가    
  null이거나 empty면 안된다는것 조차 밖에서 알게 하는것보다 ImageService만 아는게 좋겠다 싶어 이렇게 구현했다      

<br/><br/>  
  
```java
private List<Image> uploadImages(List<MultipartFile> multipartFiles, Long referenceId, DomainType domainType) {
  return multipartFiles.stream()
      .map(multipartFile -> uploadImage(multipartFile, referenceId, domainType))
      .collect(Collectors.toList());
}

private Image uploadImage(MultipartFile multipartFile, Long referenceId, DomainType domainType) {
  String originalName = multipartFile.getOriginalFilename();
  String uniqueName = createUniqueName(originalName);

  storeImage(multipartFile, uniqueName);

  return Image.builder()
      .originalName(originalName)
      .fullPath(getFullPath(uniqueName))
      .referenceId(referenceId)
      .domainType(domainType)
      .build();
}

private String createUniqueName(String originalName) {
  String ext = extractExtension(originalName);
  String uuid = UUID.randomUUID().toString();

  return uuid + "." + ext;
} 

private String extractExtension(String originalName) {
  int pos = originalName.lastIndexOf(".");

  return originalName.substring(pos + 1);
}
```
  
- client에게서 받은 이름이 중복될 수 있으니, 저장시에는 uuid로 바꾸어 저장한다   
- client에게서 받은 이름에서 확장자만 추출해서, uuid값과 같이 저장한다     
  
<br/><br/>    
  
```java
private void storeImage(MultipartFile multipartFile, String uniqueName) {
  try {
    File file = new File(uploadPath);
    if (!file.exists()) {
      file.mkdirs(); 
    }
    multipartFile.transferTo(new File(getFullPath(uniqueName)));
  } catch (IOException e) {
    throw new RuntimeException(e);
  }
}

private String getFullPath(String uniqueName) {
  return uploadPath + uniqueName;
}

```

- 실제 저장하는 로직   
  - 이미지를 저장하기로 지정한 폴더가 없으면 상위폴더를 생성한다  
  
<br/><br/>   
  
## 그 외 고려했던 or 중인 사항들  
  
- 이미지 정보를 MultipartFile로 받지 않고, base64로 인코딩된 값을 JSON 포맷으로 받는 방식도 있었는데,      
  인코딩하는 과정도 필요하고 파일크기가 약 1.3배정도 늘어난다고 한다고 해서 패스     
- 또 다른 방식은 두 요청을 아예 분리해서 처리하는 것       
  - 이미지만 AJAX등을 통해서 별도 요청으로 먼저 서버에 전송
  - 서버는 이미지 저장하고, 저장된 이미지의 id(또는 파일명)를 client에 전송   
  - client는 이미지의 id와 전송할 데이터를 application/json으로 전송  
  - 이건, 내가 중간테이블 없이, 이미지 테이블이 상품 아이디를 가지고 있게 설계했기 때문에 적합하지 않은 방법이었다      

<br/>   
  
- 현재 내가 구현한 방법은 상품의 문자열 정보와 이미지 정보를 같이 받아서 처리한다        
  이렇게 되면, 이미지가 저장되는 부분에서 병목이 생겨 전체 응답 속도에 영향이 생길 수도 있을 듯하다     
  그래서 스레드를 분리하는 방법도 다음주에 고민해봐야겠다      

- 그리고 이미지파일을 현재는 로컬에 저장하고 있는데, S3를 사용해서도 구현해볼 예정이다       
  ImageService를 interface로 두고 ImageLocalService를 구현했으니, ImageS3Service 구현 예정     
 
<br/><br/>    

Reference     
https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-mvc-2#curriculum      
https://www.inflearn.com/course/http-%EC%9B%B9-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC      
https://tecoble.techcourse.co.kr/post/2021-05-11-requestbody-modelattribute/      
https://mhwan.tistory.com/72    
https://joont92.github.io/spring/MessageConverter/    
https://www.inflearn.com/questions/307133/image-%EC%A0%84%EC%86%A1%EA%B3%BC-%ED%95%A8%EA%BB%98-%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%8A%94-json%EC%9C%BC%EB%A1%9C-%EB%B3%B4%EB%82%B4%EA%B3%A0-%EC%8B%B6%EC%9D%80-%EA%B2%BD%EC%9A%B0    
https://hello-bryan.tistory.com/343          
https://chb2005.tistory.com/102    
https://velog.io/@jyyoun1022/SPRING%ED%8C%8C%EC%9D%BC-%EC%97%85%EB%A1%9C%EB%93%9C-%EC%B2%98%EB%A6%AC       
https://way-be-developer.tistory.com/m/292   

<br/>   
