---
title: Micrometer-metrics 오픈소스 기여 과정
excerpt: netty.eventexecutor.workers Gauge metric 추가
---

<br/>

- [Micrometer Netty eventexecutor.workers metric 에 기여](https://github.com/micrometer-metrics/micrometer/pull/6612)를 했다     
  벌써 4개월전이지만 더 가물가물해지기 전에 기록용으로 남기기     
  - 이번에 좀 인상적이었던 점은 코드리뷰가 상당히 활발했다는것         
    특히 메인테이너가 아닌 이슈 원 작성자와의 핑퐁이 새로웠음 (메인테이너보다 더 적극적임ㅎ)   

<br/>

## 문제   
[이슈](https://github.com/micrometer-metrics/micrometer/issues/6375)    
- `Netty EventExecutor`의 worker 수를 바로 확인할 수 있는 metric이 없음   
- 운영 중 thread pool 상태를 빠르게 파악하기 어려움

<br/>

당시 서비스에서 힙 메모리 누수 이슈를 추적하면서 worker 수 바로 확인할 수 있으면 분석에 도움되겠다 싶었는데, 마침 관련 이슈가 이미 올라와있어 진행하였다 (그렇지만 Micrometer 릴리즈 이전에 이직을 해버린,, 아숩)     

<br/><br/>

## 해결
- `netty.eventexecutor.workers` Gauge metric 추가    

  => `NettyEventExecutorMetrics` 를 통해 worker 수가 실시간으로 노출 가능해짐   

<br/><br/>

## 코드리뷰들

초기 구현은 worker 수 노출에 초점을 두었지만, 리뷰 과정에서 설계 관점의 검증이 추가로 이루어졌다   

<br/><br/>

**EventLoopGroup 구현체에 따른 worker 수 수집 전략**     

초기 구현에서는 단일 스레드 이벤트 루프의 경우 worker 수가 항상 1이므로 모니터링 관점에서 의미가 제한적이라고 판단해 멀티 스레드 구현의 기반이 되는 `MultithreadEventLoopGroup`을 중심으로 처리했다      
또한 worker 수를 얻기 위해 `MultithreadEventLoopGroup`이 제공하는 `executorCount()` 메서드를 사용하는 것이 가장 효율적인 접근이라고 판단했다 (EventLoopGroup 인터페이스 자체는 worker 수를 직접 제공하지 않음)   


<br/>

그러나 리뷰 과정에서 메트릭 사용자는 이벤트 루프의 구체적인 구현을 알지 못할 수 있고, 구현체에 따라 메트릭 유무가 달라지지 않는 것이 바람직하다는 의견이 제시되었다    
이에 따라 최종 구현에서는 `MultithreadEventLoopGroup`에 대해서는 `executorCount()`를 사용하는 fast-path를 유지하고,    
그 외 경우에는 모든 `EventExecutor`를 순회하는 fallback 경로를 추가하여 싱글/멀티 스레드 이벤트 루프 모두에서 worker 수가 일관되게 노출되도록 변경하였다    

<br/><br/>

**Gauge 등록 시 GC 안전성**      
리뷰 중 Gauge가 내부 객체를 강한 참조로 유지할 가능성이 지적되었다       
이는 과거 Micrometer에서 실제로 GC 이슈를 유발했던 사례와도 연결되는 부분이었다    

<br/>

최종 구현에서 Micrometer의 기본 Gauge 등록 방식(weak reference)을 활용하여 이벤트 루프 객체가 정상적으로 GC 대상이 되도록 보장하였다     
이를 통해 기존 GC 관련 테스트의 의미를 유지하면서 메모리 안정성을 확보할 수 있었다    


<br/><br/>

**서브클래스 및 fallback 경로에 대한 테스트**      
구현의 범용성을 검증하기 위해 다음 테스트들을 추가하였다   

- `MultiThreadIoEventLoopGroup` 서브클래스에서도 worker 수가 정상적으로 계산되는지 검증
- `Iterable<EventExecutor>` 기반 fallback 경로가 실제로 동작하는지에 대한 명시적인 테스트

이를 통해 Netty 구현체에 종속되지 않는 동작을 테스트 레벨에서 보장할 수 있었다   

<br/><br/>

결과적으로 운영 환경에서 바로 사용 가능한 수준의 메트릭으로 정리되었고, 메인테이너 반응도 긍정적이었다   

<img width="877" height="247" alt="image" src="https://github.com/user-attachments/assets/4d6125bb-95dc-48ca-89c0-651f587c250a" />


리뷰 과정에서 타입 추상화, GC 안전성, 테스트 의미까지 체크되며 Micrometer 라이브러리 기준에 맞는 구현으로 다듬어진 PR이었다 (다들 아주 꼼꼼쓰 구웃)        
리뷰를 통해 PR 범위와 기준을 맞춰가는 과정 자체가 인상적이었고, 최종적으로 merge 까지 이루어져서 뿌듯했다     

<br/>

<img width="792" height="98" alt="image" src="https://github.com/user-attachments/assets/a9daec46-5845-484b-a093-149582cecb24" />     

<br/><br/>

contributor 라벨 생겼다 ㅎ   

<br/>
