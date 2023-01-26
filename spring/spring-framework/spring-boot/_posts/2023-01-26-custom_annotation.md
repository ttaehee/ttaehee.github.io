---   
title: Spring) Custom Annotation
excerpt: SpringBoot project에서 커스텀 어노테이션 만들기     
---   

<br/>    

- 이번 프로젝트 때, client에서 들어오는 `신발 사이즈 값의 유효성 검증`을 위해 custom annotation을 만들어 코드와 분리하고 가독성을 높여보았다 그 과정 기록하기      
- custom annotation의 `간결하다는 장점`이 있지만, 그만큼 `로직 흐름이 응축되어` 있기 때문에 그 흐름을 파악하기 어렵다는 양면성을 가진다는 생각이 들었다   
  - 그렇지만, `관심사의 분리`를 통해 비지니스 로직에 집중할 수 있게 해주는 장점에 초점을 맞추어 구현하였다
  - 굳이 코드 상으로 검증 로직을 넣고, 에러 처리를 해줄 필요가 없다고 생각했다     
  - 의도와 목적을 명확히하여 팀원들과 공감대를 이룬 후 추가하였다   
  - 다만, `무분별한 custom annotation 생성`은 지양해야 한다고 생각한다    
- 추가로, annotation을 알아보면서 
  평소에 궁금했던 `어떻게 @Component를 class에 붙이기만 하면 자동으로 Bean으로 등록될까?`에 대해서도 알 수 있었다   

<br/>

## Annotation      

- annotation의 본질적인 목적은 source code에 추가적인 정보인 meta data를 표현하는 것       
- 단순히 부가적인 표현 + reflection을 이용해 annotation 지정만으로 원하는 클래스 주입하기 등 가능  

<br/>

### Built-in annotation  
- 자바에서는 제공하는 기본 annotation
  - @Override, @FunctionalInterface, @Deprecated 등

<br/>

### Meta annotation        
- 기본 annotation 외에 meta annotation    
- meta annotation을 이용해 custom annotation 만들 수 있음   

<br/>

**1. @Retention**    
annotation의 지속시간 (어떤 시점까지 어노테이션이 영향을 미치는지 결정)     
= anntation이 어떻게 저장될지 결정 (코드 or 클래스로 컴파일 or 런타임) 

- RetentionPolicy enum에서 참조 가능   
  - `RetentionPolicy.SOURCE` : 컴파일 후에 정보들이 사라짐 (byte code에 기록되지 않음)
    - ex) @Override, @SuppressWarnings
  - `RetentionPolicy.CLASS` : 컴파일 타임 때만 .class 파일에 존재하고 런타임 때는 없어잠 (byte code에 기록됨)  
    - default 값  
    - byte code level에서 어떤 작업을 해야할 때 유용     
    - Reflection 사용 불가능
  - `RetentionPlicy.RUNTIME` : 런타임시에도 .class 파일에 존재
    - custom annotation 만들 때 주로 사용   
    - Reflection 사용 가능

  ![제목 없음](https://user-images.githubusercontent.com/103614357/214767074-cb5dd9d8-16e3-495b-9698-52f63d532802.png)     

<br/>

**2. @Documented**     
문서에 annotation 정보 포함하겠다는 의미 (public contract에 나옴)   

<br/>

**3. @Target**      
annotation 적용 가능한 위치 지정 (ElementType enum 사용)    
default 값은 모든 대상    

- ElementType.FIELD: enum 상수를 포함한 field 정의에 사용      
  = field에만 annotation 달 수 있음(다른부분에 어노테이션 사용하면 컴파일 에러)
- ElementType.TYPE : class, interface, enum 정의에 사용 
- ElementType.METHOD: method 정의에 사용   
- ElementType.PARAMETER: 파라미터 정의에 사용
- ElementType.CONSTRUCTOR: 생성자 정의에 사용
- ElementType.TYPE_USE: 자바 8이상부터, 타입에 사용

<br/>

**4. @Inherited**       
하위 클래스가 이 annotation을 상속 받을 수 있게 하겠다는 의미   

<br/>

**5. @Repeatable**      
이 annotion을 같은 곳에서 중복해 사용 가능하다는 의미 (value에 여러 값을 넣고 싶을 때)    

<br/><br/> 
   
## Custom Annotation    
annotation을 내가 만들어서도 쓸 수 있다   

- annotation 정보를 얻고 싶으면, reflection만 이용해서 얻을 수 있음   

<br/>  

### custom annotation 구현하기

**ShoeSize annotation**           

```java
@Documented   //문서에 표시하겠다
@Retention(RetentionPolicy.RUNTIME)   //런타임에서도 적용되게 하겠다    
@Target({ElementType.FIELD, ElementType.TYPE_USE})   //필드와 타입에만 적용하겠다    
@Constraint(validatedBy = {ShoeSizeValidator.class})   //유효성 검증을 위해 넣었다  
public @interface ShoeSize {

	String message() default "";

	Class<?>[] groups() default {};

	Class<? extends Payload>[] payload() default {};
}
```

- annotation type은 @interface로 정의해야함     
  - 자동적으로 java.lang.Annotation interface 상속받음       
    -> 다른 class 상속 받을 수 없음    
 
 <br/>   

- JSR-303 표준 어노테이션들이 갖는 3가지 공통 속성들
  - message : 유효하지 않을 경우 반환할 메세지 정의
  - groups : 유효성 검증이 진행될 그룹 정의 = 상황별 validation 제어가능   
    - ex) insert 할 때와 update할 때, validation을 구분해서 실행하고 싶을 때 사용   
  - payload : 유효성 검증 시에 전달할 메타 정보 및 제약 사항 정의   
    - ex) 해당 유효성 검증의 중요도    

<br/><br/>  

**ShoeSizeValidator.java**       
- ConstraintValidator 상속 + isValid() override해서 값이 유효한지 판단하는 로직 작성   
  - 신발 사이즈에 대한 유효성 검증이기 때문에 입력값의 크기가 0초과 400이하인지와 5단위인지 체크했다    

```java
public class ShoeSizeValidator implements ConstraintValidator<ShoeSize, Integer> {

	@Override
	public boolean isValid(Integer value, ConstraintValidatorContext context) {
		return isValidSizeRange(value) && isValidSizeUnits(value);
	}

	private boolean isValidSizeRange(int value) {
		return value > 0 && value <= 400;
	}

	private boolean isValidSizeUnits(int value) {
		return value % 5 == 0;
	}
}
```

<br/><br/>

**사용 : ProductRegisterRequest.java**      

```java
public record ProductRegisterRequest(
  ...

  List<@ShoeSize(message = "잘못된 형식의 신발 사이즈가 입력되었습니다.") Integer> sizes) {
}
```

<br/><br/>

## Spring의 Annotation   
Spring에서 application 실행 시, @Component가 붙은 클래스들을 스캔해서 IoC 컨테이너에 등록해줌    
어떻게? 어떤 과정으로 되는걸까?      

<br/>

### @Component  

**Component annotation**      

```java
@Target({ElementType.TYPE})    //TYPE : Class나 Interface에 적용하겠다 = 타겟으로 삼겠다 
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Indexed
public @interface Component {
    String value() default "";
}
```

<br/>    

**Service annotation**      
- @Service를 붙여도 스캔됨   
  - Component annotation 사용중이기 때문에       

```java
@Target({ElementType.TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented
@Component  
public @interface Service {
    @AliasFor(
        annotation = Component.class
    )
    String value() default "";
}
```

<br/>

**ClassPathBeanDefinitionScanner.java의 doScan()**        
- 스캔해서 bean으로 등록해주는 Spring의 코드   
  -  ClassPath에 있는 package의 모든 class를 읽어, annotation이 붙은 class를 처리(ex) IoC 컨테이너에 클래스 등록)해줌 

  ![doscan](https://user-images.githubusercontent.com/103614357/214771658-57bc9485-2002-483c-8ecd-054dd77946de.png)    

<br/>
  
- doScan method      

  ```java
  //doScan()
  Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();

  for (String basePackage : basePackages) {
    //basePackage에 속한 모든 리소스 파일 읽어옴   
    Set<BeanDefinition> candidates = findCandidateComponents(basePackage);  //아래 코드블럭 참고   
  ```

  - findCandidateComponents method  
  
    ```java
    public class ClassPathScanningCandidateComponentProvider implements EnvironmentCapable, ResourceLoaderAware {
        @Nullable   //어떤 메타 데이터가 있을 때 채워지는 변수  
        private CandidateComponentsIndex componentsIndex;
        ...

        public Set<BeanDefinition> findCandidateComponents(String basePackage) {
        if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
          return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);  //아래 코드블럭 참고 
        }
        else {
          return scanCandidateComponents(basePackage);  //아래 코드블럭 참고   
        }
      }
        ...
    }
    ```
  
  - addCandidateComponentsFromIndex, scanCandidateComponents 메서드에 존재하는 isCandidateComponent 메서드   
  
    ```java
    //Component annotation 인지 판별 - 맞으면 filter 통과해서 bean으로 등록   
    protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
      for (TypeFilter tf : this.excludeFilters) {
        if (tf.match(metadataReader, getMetadataReaderFactory())) {
          return false;
        }
      }
      for (TypeFilter tf : this.includeFilters) {
        if (tf.match(metadataReader, getMetadataReaderFactory())) {
          return isConditionMatch(metadataReader);
        }
      }
      return false;
    }
    ```

<br/> 

- doScan method 이어서  

  ```java    
    for (BeanDefinition candidate : candidates) {   
      //bean의 Scope이 싱글톤인지 프로토타입인지 판별  
      ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);     
      candidate.setScope(scopeMetadata.getScopeName());
      //bean 이름 결정(beanNameGenerator 사용 - 아래 코드블럭 참고)
      String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
      if (candidate instanceof AbstractBeanDefinition) {
        postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
      }
      if (candidate instanceof AnnotatedBeanDefinition) {
        AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
      }
  ```  

  - BeanNameGenerator

    ```java
    private BeanNameGenerator beanNameGenerator = AnnotationBeanNameGenerator.INSTANCE;   
    ```
    
<br/>

- doScan method 이어서  

  ```java  
  if (checkCandidate(beanName, candidate)) {
      BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);   
      definitionHolder =
          AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);  
      //BeanDefinitionHolder로 만든 후 BeanDefinition로 만듬   
      beanDefinitions.add(definitionHolder);  
      registerBeanDefinition(definitionHolder, this.registry); // 만든걸 빈팩토리에 bean 등록(registry에 등록)  
  }
  ```   

<br/><br/>

Reference    
https://jdm.kr/blog/216    
https://en.wikipedia.org/wiki/Java_annotation        
https://donghyeon.dev/spring/2020/08/18/Spring-Annotation%EC%9D%98-%EC%9B%90%EB%A6%AC%EC%99%80-Custom-Annotation-%EB%A7%8C%EB%93%A4%EC%96%B4%EB%B3%B4%EA%B8%B0/        
https://mangkyu.tistory.com/206     
https://ittrue.tistory.com/158    
https://tecoble.techcourse.co.kr/post/2021-06-21-custom-annotation/    
https://techblog.woowahan.com/2684/    
https://pamyferret.tistory.com/65     
https://minkukjo.github.io/framework/2020/07/09/Spring-133/    

<br/>
