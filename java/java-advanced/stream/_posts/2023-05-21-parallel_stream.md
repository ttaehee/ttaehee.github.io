---
title: JAVA) Stream에서 항상 parallelStream이 stream보다 빠를까
excerpt: 모든 사용자에게 메일을 전송하는 작업 처리 시, parallelStream 사용이 유리할까
---

<br/>

chat gpt를 활용한 면접 대비 서비스를 구현하면서, **모든 사용자에게 메일을 전송**하는 작업이 필요해 stream을 사용하였다      
이 부분에서, 각각의 전송하는 로직이 **이전 작업에 종속적이지 않으므로 병렬적으로 처리**해도 되지 않을까 생각이 들었다     
그래서 이 기회에 parallelStream에 대해 더 알아보고, 현재 내가 원하는 작업에 **parallelStream의 사용이 더 효율적일지** 테스트해보았다     

<br/>

결론부터 말해보자면 

> Stream에서 항상 parallelStream이 stream보다 빠를까?    
> => No! 상황에 따라 다르다      

병렬 처리라는 것이 단순히 **작업을 쪼개서 수행**하는 것이라고 생각할 수 있지만, 쪼개면서 발생하는 **오버헤드와 컨텍스트 스위칭** 등을 고려해야 하고 쪼갠 것을 다시 **취합하는 과정**도 필요하기 때문이다  

<br/>

그렇다면, 어떤 상황에 병렬처리를 하는 것이 효율적인 것이며 현재 내가 원하는 작업은 병렬처리가 효율적인 것인지 알아보자  

<br/>

## ParallelStream

Java 8부터 `parallelStream(), parallel()` 메서드로 병렬 처리를 지원해준다    
복잡하던 스레드 관리 방식(= ForkJoinPool 관리 방식)을 Fork와 Join을 통해서 작업들을 분할 정복(Divide and Conquer) 기법으로 처리한다       

<br/>

**ParallelStream의 특징**    

- 순서 보장 안됨 (병렬처리니까)
- 별도의 설정이 없으면 해당 application이 구동되는 스펙에 따라 thread 수 결정됨
- thread가 N개 생성되었을 때, 하나는 main thread로 스트림을 처리하는 기본 스레드이고 나머지 N-1개의 thread가 ForkJoinPool thread
- 여기서 사용하는 ForkJoinPool을 Custom Pool을 별도로 생성해서 활용하지 않으면, JVM instance에서 공통적으로 사용하는 Pool을 사용함

<br/>

### forEach vs stream vs parallelStream 속도 비교 테스트    

```java
@Test
void sendQuestionUsingForEach() {
    long startTime = System.currentTimeMillis();

    members.forEach(
            member -> generateAndSend(member.getCategory().name(), member.getEmail())
    );

    long endTime = System.currentTimeMillis();
    System.out.println("==============================================");
    System.out.printf("send question using for each 수행시간 : %d\n", endTime - startTime);
    System.out.println("==============================================");
}

@Test
void sendQuestionUsingStream() {
    long startTime = System.currentTimeMillis();

    members.stream().forEach(
            member -> generateAndSend(member.getCategory().name(), member.getEmail())
    );

    long endTime = System.currentTimeMillis();
    System.out.println("==============================================");
    System.out.printf("send question using stream 수행시간 : %d\n", endTime - startTime);
    System.out.println("==============================================");
}

@Test
void sendQuestionUsingParallelStream() {
    long startTime = System.currentTimeMillis();

    members.parallelStream().forEach(
            member -> generateAndSend(member.getCategory().name(), member.getEmail())
    );

    long endTime = System.currentTimeMillis();
    System.out.println("==============================================");
    System.out.printf("send question using parallel stream 수행시간 : %d\n", endTime - startTime);
    System.out.println("==============================================");
}
```

<br/>

#### 10,000건 데이터인 경우   

<img width="406" alt="스크린샷 2023-05-21 오후 8 05 46" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/6b55466b-952a-46db-813f-27b83d8a433e">     

<br/>

<img width="410" alt="스크린샷 2023-05-21 오후 8 06 13" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/48004d56-da35-46d2-b8ad-a256cd1a3549">  

<br/>

<img width="424" alt="스크린샷 2023-05-21 오후 8 06 38" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/328a1588-ae6a-4242-8323-abb55dacdd43">  

<br/>

=> 속도 빠른 순 : `forEach - stream.forEach - parallelstream.forEach`          

예상과 다르게 parallelstream이 제일 느려서 데이터 크기를 늘려보았다   

<br/>

#### 100,000건 데이터인 경우    

<img width="417" alt="스크린샷 2023-05-21 오후 8 08 12" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/1191802b-0092-44e0-8b1e-3a4d07a92b46">     

<br/>

<img width="418" alt="스크린샷 2023-05-21 오후 8 09 07" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/1adacfa9-93c8-4ce6-ad88-9a3b93aea278">    

<br/>

<img width="430" alt="스크린샷 2023-05-21 오후 8 10 07" src="https://github.com/ttaehee/ttaehee.github.io/assets/103614357/2ec0964c-0f21-4b21-902f-e07d5e0100cb">  

<br/>

=> 속도 빠른 순 : `parallelstream.forEach - forEach - stream.forEach`     

<br/><br/>

## 결론 :: 소량의 데이터에서는 병렬 처리가 도움이 되지 않음    
만건의 데이터에서는 parallelstream의 속도가 **가장 느렸고** 십만건의 데이터에서는 parallelstream이 **가장 빨랐다**    
**오버헤드 발생**으로 인해 일어난 결과라고 생각한다    

<br/> 

parallelStream은 병렬 처리라 무조건 더 빠를거라는 생각은 하면 안되겠다     
**특정 상황에서 잘 써야** 빠르다           
일단 이번 테스트를 통해 **소량의 데이터에서는 stream의 효율이 낫다**라는 결론을 얻었다      

<br/>

이 서비스를 이용하는 **모든 유저에게** 메일을 전송하는 작업이므로 **데이터의 수가 100,000만큼 클 것**이라고 판단하였다(꿈이 야무짐 ㅎㅎ)   
또한 외부 api를 호출하는 로직이 섞여있어 작업이 무겁기 때문에 병렬처리가 효율적이라고 생각했다   
따라서 병렬처리를 적용 하였다   
소량의 데이터라는 기준이 애매하기도 하고 사용하는 메서드에 따라 다를 것으로 예상되므로, 앞으로도 잘 모르겠는 경우 **속도를 측정해보고 사용할지 여부를 판단**해야겠다        

<br/><br/>

Reference       
Effective Java

<br/>

