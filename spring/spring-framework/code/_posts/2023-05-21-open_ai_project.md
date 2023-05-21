---
title: Spring) Open AI(Chat GPT)를 활용한 면접 대비 서비스 구현하기
excerpt: 설계부터 구현 과정과 고려사항들   
---

<br/>

## 개요   
요즘 면접 준비로 매일 질문들을 찾아보는 중인데 이 **과정이 자동화**되면 좋겠다고 생각했다    
그래서 Open AI(Chat GPT)를 활용해 면접 질문을 받아와서 이를 사용자의 메일로 보내는 서비스를 만들어 보았다         
면접 질문들은 **처음에 크롤링**을 생각하였는데, 조금 더 재미있게 새롭게 해보고자 요즘 말 많은 **Chat GPT를 사용**하기로 하였다    

<br/>

## 서비스 요구사항

- 사용자는 카테고리(**DB, OS, 네트워크, 자료구조**)를 선택할 수 있다
- 사용자는 **입력한 메일**로 **매일 일정 개수**만큼 질문을 받을 수 있다 
  - 일단 10개로 고정
- 매일 **새벽 5시**에 모든 사용자에게 일괄적으로 이메일을 전송한다 
  - 해당 시간에 사용자들이 서비스를 가장 적게 사용할 것이라고 예상하였다 현재 단계에서는 예상일 뿐이기 때문에 실제 서비스를 운영 시 수정 반영이 필요하다
- 이메일 전송 시, **답은 없이** 질문만 전송한다

<br/>

### 추가 예정인 요구사항    
- 사용자는 이메일 **수신 시간**을 정할 수 있다
- 사용자는 받을 수 있는 **질문 개수를 선택**할 수 있다    
- **중복**되는 내용은 제거한다(고도화)

<br/>

## Flow
1. 사용자는 본인의 **메일 주소와, 원하는 수신 시간, 질문 카테고리**를 입력한다  
2. 면접 질문을 **Open AI(Chat GPT)** 를 통해 가져온다  
3. 사용자에게 **메일로** 질문을 보낸다   

<br/>

## 설계 시 고려한 내용
### 1. 속도    
외부 api 호출 시 **데이터의 로딩 속도 저하**가 있어 비동기적으로 처리하였다    
이는 chat gpt와 mail 전송 시 모두 **비동기로 처리**하였다    
thread pool 관련하여 추가로 공부해서 커스터마이징 하려한다    

<br/>

### 2. 단순한 네이밍과 호출 구조  
**QuestionGenerator에서 질문 생성**, **QuestionSender에서 메일 전송의 기능**을 맡기고 QuestionService에서 이들을 호출하도록 하였다     

<br/>

### 3 유지 보수     
어떤 시스템이 계속 돌아가는 한 **기능을 추가하거나 수정**해야 하는 일은 언젠가 올테니, 최대한 **변할 수 있는 것들을 인터페이스로** 분리하여 구현하였다    
현재는 chat gpt와 gmail smtp를 사용하고 있으나 언제든 바뀔 수 있으니, 위에서의 **QuestionGenerator와 QuestionSender를 인터페이스로 분리**하였다   

<br/>

## 클래스 다이어그램    

<img width="725" alt="스크린샷 2023-05-21 오후 5 32 52" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/651bed2e-9cc9-4ce5-9d29-1deec310e1fa">

<br/>

## 개선이 필요한 점 :: 고도화
### 중복 제거
현재 서비스에는 사용자가 **전에 받았던 같은 질문을 받는 것**이 고려되어 있지 않다    
사실, 면접을 준비하는 입장에서 **리마인드 느낌**이라 크게 상관이 없다고 생각해 **중요도를 낮추었다**    

<br/>

하지만 사용자 입장에서 그 부분을 선택할 수 있다면 훨씬 좋을거라고 생각하기 때문에 개선사항으로 두었다   
현재 생각해둔 플로우는 질문을 저장할 때 **키워드를 바탕으로 중복을 가려내서** DB에 저장해두고 메일로 보낼까 하는데 키워드 분석기가 제대로 먹히질 않아 고민 중이다    

<br/>

## 구현   
### Open AI API 연동    
면접 질문을 Open AI(Chat GPT)로부터 가져오기 위해 Open AI API를 사용했다      
[OpenAI](https://platform.openai.com/overview)에서 키를 발급받고, **api 호출을 위해 OpenFeignClient**를 사용했다     
([지난번에 정리한 OpenFeignClient](https://ttaehee.github.io/spring/spring-http/open_feign/))  

<br/>

### JavaMailSender 사용
Spring Boot의 메일 전송 라이브러리 중 `spring-boot-starter-mail`의 JavaMailSender 인터페이스를 사용했다    
JavaMailSender는 **Spring에서 제공하는 API**이기 때문에 굳이 서드 파티 라이브러리를 사용하지 않아도 빠르게 개발에 전념할 수 있다고 판단해 사용했다        
서드 파티 라이브러리들과의 큰 차이를 못느끼기도 했고     
([지난번에 정리한 JavaMailSender](https://ttaehee.github.io/spring/spring-framework/spring-boot/smtp/))

<br/>

**MailMessage의 setTo()** 메서드의 경우, 한 번에 여러 사용자에게 전송할 수 있도록 varargs를 지원해서 사용하려했는데 같이 보낸 모든 사람의 이메일이 같이 보인다       
**개인정보 유출 가능성**이 있기 때문에 사용하지 않고 **loop를 통해 각각 메일이 발송**되도록 했다      

```java
@Service
@RequiredArgsConstructor
public class QuestionService {

    private final QuestionGenerator questionGenerator;
    private final QuestionSender questionSender;
    private final MemberRepository memberRepository;

    @Transactional
    public void sendQuestion() {
        List<Member> members = memberRepository.findAll();
        members.parallelStream().forEach(
                member -> generateAndSend(member.getCategory().name(), member.getEmail())
        );
    }

    private void generateAndSend(String categoryName, String toAddress){
        SendQuestionResponse generatedQuestions = questionGenerator.generate(categoryName);
        questionSender.send(toAddress, generatedQuestions.combineToSting());
    }
}
```

또한, 각각의 전송하는 로직이 이전 작업에 **종속적이지 않으므로** 병렬적으로 처리해도 된다고 판단하여 **parallelStream을 사용하여 병렬로** 전송했다     

<br/>

현재는 메일로 보낼 질문들을 한줄씩 보내고 있다     
질문들을 한줄씩 문자열로 정리하는 부분을 dto에서(`generatedQuestions.combineToSting()`) 하고 있는데, 지금 생각해보니 이 **기능을 하는 아이**를 두면 좋을 것 같다        
그래야 **질문 정렬 방법이 달라졌을 때도** 유연하게 대응할 수 있을거라 생각된다    
리팩토링에 반영해야지    

<br/>

### 스케줄링     

매일 새벽 5시에 모든 사용자에게 일괄적으로 질문이 담긴 메일이 전송될 수 있도록 스케줄링 기능을 사용하였다    
스케줄링에는 `spring-boot-starter`에 기본으로 지원되는 spring scheduler를 사용하였다   

<br/>
