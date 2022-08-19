---
title: Spring) Spring Security Setting
excerpt: 스프링시큐리티 기본세팅
---

SpringBoot, JPA, Spring Security 를 사용해서 Blog 만드는중!  
지난번엔 [SpringBoot Project setting](https://ttaehee.github.io/spring/spring-framework/spring-boot/setting/)
을 하고, 오늘은 Spring Security setting 후 회원가입, 로그인을 구현했다  

<br/>

## Spring Security Setting
### builde.gradle
지난번과 동일  

```
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
  implementation 'org.springframework.boot:spring-boot-starter-security'
  implementation 'org.springframework.boot:spring-boot-starter-web'
  compileOnly 'org.projectlombok:lombok'
  developmentOnly 'org.springframework.boot:spring-boot-devtools'
  runtimeOnly 'mysql:mysql-connector-java'
  annotationProcessor 'org.projectlombok:lombok'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
  testImplementation 'org.springframework.security:spring-security-test'

  // JSTL
  implementation 'javax.servlet:jstl'	
  // JSP 탬플릿 엔진							
  implementation 'org.apache.tomcat.embed:tomcat-embed-jasper'
  // Security 태그 라이브러리		
  implementation 'org.springframework.security:spring-security-taglibs'
}
```

### SecurityConfig.java (extends WebSecurityConfigurerAdapter)

WebSecurityConfigurerAdapter를 상속받은 커스텀 설정을 빈으로 등록하면  
스프링부트의 기본 시큐리티 설정은 더이상 제공되지 않음(커스터마이징 활성화)  

```
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true) 
public class SecurityConfig extends WebSecurityConfigurerAdapter{
```

- @Configuration : 해당 클래스를 Configuration으로 등록, 빈등록(IoC)
- @EnableWebSecurity : Spring Security는 활성화 되어있고, 시큐리티 필터(추가 설정)가 등록
- @EnableGlobalMethodSecurity(prePostEnabled = true) : Controller에서 특정 권한이 있는 유저만 접근을 허용하려면 @PreAuthorize 어노테이션을 사용하는데, 해당 어노테이션을 활성화 시킴

<br/>

```	
  @Autowired
  private PrincipalDetailService principalDetailService;

  @Bean
  public BCryptPasswordEncoder encodePWD() {
    return new BCryptPasswordEncoder();
  }
```

- PrincipalDetailService : 로그인 요청 시, 입력된 유저 정보와 DB의 회원정보를 비교해 인증된 사용자인지 체크하는 로직이 정의
- BCryptPasswordEncoder : 비밀번호 암호화/복호화 로직 담긴 객체
  
<br/>
  
```
  @Bean
  @Override
  public AuthenticationManager authenticationManagerBean() throws Exception {
    return super.authenticationManagerBean();
  }
 
  @Override
  protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.userDetailsService(principalDetailService)
      .passwordEncoder(encodePWD());
  }
```
AuthenticationManager를 생성하기 위해 authenticationManagerBean()을 상속받아 사용  

AuthenticationManager는 사용자 인증을 담당  
`auth.userDetailsService(service)`에 `org.springframework.security.core.userdetails.UserDetailsService` 인터페이스를 구현한 Service(나는 principalDetailService)를 넘겨야함  
시큐리티가 대신 로그인해줄 때 password를 가로채기를 하는데 해당 password가 뭘로 해쉬가 되어 회원가입이 되었는지 알아야 같은 해쉬로 암호화해서 DB에 있는 해쉬랑 비교할 수 있음

<br/>  

```
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http
      .csrf().disable()
      .authorizeRequests()
         .antMatchers("/", "/auth/**", "/css/**", "/image/**", "/dummy/**") 
         .permitAll()
         .anyRequest()
         .authenticated()
      .and()
         .formLogin()
         .loginPage("/auth/loginForm")
         .loginProcessingUrl("/auth/loginProc")
         .defaultSuccessUrl("/");
  }
}
```
configure(HttpSecurity http) : filter chain 안으로 스프링 시큐리티는 허용 + HTTP로 거르기

<br/>

http()  
- csrf().disable() : csrf 토큰 비활성화 (현재 테스트 중이기 때문에 disable() 걸어두는 게 좋음)
  - 시큐리티는 기본적으로 요청 시 CSRF Token이 있어야 응답해줌
- authorizeRequests() : HttpServletRequest 요청 URL에 따라 접근 권한을 설정
- antMatchers("pathPattern") : 요청 URL 경로 패턴을 지정
- permitAll() : 모든 유저 접근 허용
- authenticated() : 인증된 유저만 접근 허용

and()  
- formLogin() : form Login 설정을 진행
- loginPage("path") : 커스텀 로그인 페이지 경로와 로그인 인증 경로를 등록
- loginProcessingUrl("path") : path url로 들어오는 로그인요청을 가로챔
- defaultSuccessUrl("path") : 로그인 인증을 성공하면 이동하는 페이지를 등록

<br/>

### PrincipalDetail (implements UserDetails)

```
public class PrincipalDetail implements UserDetails{

	private User user; // 콤포지션(객체를 품고있음)
	
	public PrincipalDetail(User user) {
		this.user=user;
	}
	
	@Override
	public String getPassword() {
		return user.getPassword();
	}
	
	@Override
	public String getUsername() {
		return user.getUsername();
	}
	
	//계정이 갖고있는 권한목록을 리턴
	@Override
	public Collection<? extends GrantedAuthority> getAuthorities() {
		
		Collection<GrantedAuthority> collectors = new ArrayList<>();	
		collectors.add(()->{return "ROLE_"+user.getRole();});
		
		return collectors;
	}
	
	//계정이 만료되었는지 리턴(true : 만료안됨)
	@Override
	public boolean isAccountNonExpired() {
		return true;
	}
	
	//계정이 잠겨있는지 리턴(true : 잠기지않음)
	@Override
	public boolean isAccountNonLocked() {
		return true;
	}
	
	//비밀번호가 만료되었는지 리턴(true : 만료안됨)
	@Override
	public boolean isCredentialsNonExpired() {
		return true;
	}
	
	//계정이 활성화(사용가능)인지 리턴(true: 활성화)
	@Override
	public boolean isEnabled() {
		return true;
	}
}
```

<br/>

### PrincipalDetailService (implements UserDetailsService)

```
@Service
public class PrincipalDetailService implements UserDetailsService{
	
  @Autowired
  UserRepository userRepository;
  
  @Override
  public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
    User principal = userRepository.findByUsername(username)
        .orElseThrow(()->{
          return new UsernameNotFoundException("해당 사용자를 찾을 수 없습니다.: "+username);
        });
    return new PrincipalDetail(principal);
  }
}
```  

`return new PrincipalDetail(principal)` : 입력된 username과 동일한 사용자가 있으면 PrincipalDetail(UserDetails) 타입으로 시큐리티의 세션에 유저정보가 저장이 됨









<br/><br/>  

Reference  
https://bamdule.tistory.com/53  
https://kimchanjung.github.io/programming/2020/07/02/spring-security-02/    
https://knoc-story.tistory.com/78
<br/>
