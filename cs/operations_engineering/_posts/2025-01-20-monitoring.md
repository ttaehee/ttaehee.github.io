---
title: 서버 안정성 향상과 성능 최적화를 위한 데이터 기반 모니터링 접근법
excerpt: Grafana와 Prometheus로 성능을 분석하고, 문제를 사전에 예방하는 모니터링 시스템 구축하기
---

<br/>  

## 도입 배경

이번에 private VPC에 서버를 구축하면서, 시스템의 안정성과 효율성을 높이기 위해 모니터링 시스템도 함께 구축했다  
기존에는 일부 서버에서만 Datadog을 사용했으나, 모든 서버에서 일관된 모니터링이 필요하다고 판단했다    

<br/>

단순히 서버를 운영하는 것을 넘어, 시스템 전반의 지표와 분석을 통해 성능을 관리하는 것은 중요하다    
예를 들면, 슬로우 쿼리를 개선한 경우 스스로의 테스트 결과에만 의존하지 않고 실제 운영 환경의 메트릭을 기반으로 평가하고 싶었다     
이를 위해 서버의 CPU 사용률, 메모리 사용량, 디스크 사용률, 네트워크 대역폭 등을 실시간으로 모니터링하고, 문제 발생 시 알림을 받을 수 있도록 시스템을 설계해보았다    

<br/><br/>

## 모니터링 도구  
모니터링 도구로는 Grafana와 Prometheus를 사용했다    
Datadog, 뉴렐릭 등도 있지만, 무료 오픈소스로도 내가 필요한 데이터를 충분히 수집하고 시각화할 수 있었기에 선택하였다     

|구분|Prometheus|Datadog|New Relic|InfluxDB|
|------|---|---|---|---|
|가격|무료|유료|유료|유/무료|
|형태|설치형|서비스형|서비스형|서비스형/설치형|
|참고 자료|많음|많음|많음|많음|
|기능 확장성|좋음|좋음|좋음|좋음|

<br/>

- Prometheus와 Grafana는 비용 면에서 유리하며, 커스터마이징 기능을 제공하지만 설정과 유지보수에 시간과 노력이 필요하다  
- Datadog은 통합성, 자동화, 예측 분석, 확장성 등 다양한 고급 기능을 제공하여, 대규모 환경이나 복잡한 시스템의 모니터링에 적합하다       

<br/>

시간을 절약하고 세밀한 기능을 제공한다는 점에서 Datadog의 장점은 분명했다       
하지만 먼저 Prometheus와 Grafana를 가볍게 설정해본 결과, 약간의 노력이 필요했지만 감당할 수 있는 수준이었고 필요한 지표를 수집하고 시각화하는 데 충분히 만족스러운 성과를 얻을 수 있었다   

<br/>

Datadog은 다양한 지표를 통합적으로 확인할 수 있는 강점이 있지만, 유료 서비스라는 점에서 우선 오픈소스 도구를 도입한 뒤 필요에 따라 도입하는 방향이 더 적합하다고 판단했다   

<br/><br/>

## Prometheus와 Grafana를 사용한 Monitoring Architecture

<img width="806" alt="image" src="https://github.com/user-attachments/assets/8bec0cf0-3a31-4c6b-abe4-9be35783e5d2" />

<br/> 

크게 내가 이해한 바로는   
- prometheus는 데이터를 저장하는 데이터베이스 및 서버 (architecture에서 backend 역할)     
- grafana는 prometheus에 query하여 데이터 조회 및 노출 (frontend 역할)

<br/>

### Prometheus

- Prometheus Docs  
> Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud.
Prometheus collects and stores its metrics as time series data, i.e. metrics information is stored with the timestamp at which it was recorded, alongside optional key-value pairs called labels.

<br/>

<img width="796" alt="image" src="https://github.com/user-attachments/assets/b85e63a5-3ac5-4378-80bc-37df573b150f" />


prometheus는 오픈 소스 시스템 모니터링 도구로, targets 로부터 metric data를 시계열 데이터로 수집하고 저장한다    
알람을 전달할 필요가 있다면 전달을 하고, grafana와 같은 데이터 시각화 도구가 PromQL이라는 쿼리를 통해 데이터를 질의하면 데이터를 전달해준다     
prometheus는 각 서비스에서 metrics를 수집할 때 pull 방식과 push 방식 둘 다 지원한다       

- pull 방식 : prometheus에서 각 서비스에 직접 metrics를 가져오는 방식
- Push 방식 : 각 서비스가 prometheus에 metrics를 업로드하는 방식

<br/><br/>

아키텍처에서 내가 prometheus를 백엔드, grafana를 프론트 역할로 나누었는데 사실 prometheus 도 그래프를 그릴 수 있는 기능이 있다    
하지만 서비스형으로 사용하는 데이터톡과 뉴 렐릭보다 시각화 부분이 다소 약하다   
이를 보완하기 위해 grafana를 이용하였다 대시보드를 커스텀할 수 있고, 다른 사람이 만든 대시보드를 이용할 수 있다는 장점이 있다   

- [ex) Grafana 대시보드 템플릿) Spring Boot 2.1 System Monitor](https://grafana.com/grafana/dashboards/11378-justai-system-monitor/)    
<img width="925" alt="image" src="https://github.com/user-attachments/assets/58bc9f67-ebfc-41fc-aec3-d661b87549c0" />   

<br/><br/>

요게 grafana 에서 사용할 수 있는 대시보드인데, 그에 반해 prometheus 그래프는 투박한편이다    
가시성도 중요하므로 grafana 를 선택하였다   

<br/><br/>

## Grafana  

- Grafana github
>Grafana allows you to query, visualize, alert on and understand your metrics no matter where they are stored.
>Create, explore, and share dashboards with your team and foster a data-driven culture

grafana는 prometheus와 연동하여 metric data를 시각적으로 표현할 수 있는 데이터를 시각화하는 대시보드를 제공한다  

<br/>

그러면 Prometheus 는 어떻게 metric data 를 얻을까?   

<br/><br/>

## Spring Actuactor
Prometheus는 Spring Actuator가 제공하는 메트릭 데이터를 HTTP endpoint(`/actuator/prometheus`)를 통해 수집한다    
이 endpoint는 Spring Actuator에서 제공하는 데이터를 Prometheus가 이해할 수 있는 형식으로 변환하여 제공하고, Prometheus는 이 데이터를 주기적으로 스크랩하고 시계열 데이터로 저장한다     

<br/>

Spring Actuator는 다양한 metric를 제공하며, MicroMeter라는 라이브러리를 통해 데이터를 Prometheus와 같은 모니터링 툴에 맞는 형식으로 변환하여 제공할 수 있다       
MicroMeter에서 변환이 가능한 모니터링 툴들에 대한 정보는 [Actuator Docs](https://docs.spring.io/spring-boot/reference/actuator/metrics.html#actuator.metrics)에서 볼 수 있다   

<br/>

### Actuator에서 제공하는 지표  

Actuator에서 제공하는 지표가 많은데, 위의 Spring boot 2.1 System Monitor에서 사용한 지표 위주로 살펴보았다   

- Basic Statistics : 시스템이 언제 시작했고, 얼마나 가동되었는지, CPU, Memory의 사용율에 대한 지표  
- JVM Statistics
  - Heaps : Heap과 Non-Heap 영역에서의 메모리를 얼마나 이용했는지에 대한 지표
  - Threads/Buffers : 작동하는 Thread와 Buffere에 관련된 지표  
  - GC : GC가 동작한 횟수, Stop the world 가 작동한 기간에 관한 지표  
- HikariCP Statistics : 전체 커넥션의 수, 커넥션이 타임아웃된 횟수, 작동, 대기, 휴식중인 커넥션의 수 등 커넥션과 관련된 지표 
- Tomcat Statistics : 작동중인 쓰레드, 초당 Http 요청의 수, 요청을 처리하는데 걸린 시간, 발생한 응답 코드 등 Web 관련된 지표 
- Logback Statistics : 각 레벨(info, error, warn, debug, trace)에 해당하는 로그가 언제 몇개가 발생했는지에 대한 지표    

<br/><br/>

## 알람 설정
Prometheus Alert Rule 을 설정해두면 그에 따라 metric data 를 평가하고, 알람 조건이 충족되면 Alertmanager에 알림을 보낸다     
Alertmanager는 받은 알림을 처리하여 이메일, Slack, PagerDuty 등의 알림 채널로 전달한다     

<br/>

Prometheus에서 설정하는 방법도 있지만, 나는 Grafana 에서 설정했다    
Prometheus는 yml 에 적어야 하는데, Grafana 는 GUI 에서 설정하길래 더 편해서리   
여기저기 안보내고 우리팀 슬랙으로만 보낼거기도 하고, 조건도 많이 복잡한 알림은 넣지 않아서 편한 방법을 택했다   

<br/>

설정한 알림   
- CPU 사용량 알림 : CPU 사용률이 80% 이상 5분 이상 지속될 경우  
- 메모리 사용량 알림 : 메모리 사용량이 80% 이상 10분 이상 지속될 경우
  - ex)
   
   ```
   alert: HighMemoryUsage
   expr: (jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}) > 0.8
   for: 10m
   ```
- 디스크 사용량 알림 : 디스크 사용량이 80% 이상 도달했을 경우  
- 응답 시간(latency) 알림 : 응답 시간이 300ms 이상 2분 이상 지속될 경우 => 알림이 너무 자주 와서 4분 이상으로 변경하였다

<br/>

알람이 너무 자주 발생하면 오히려 무시하게 되는 경향이 생겨서, 실제 서비스 상황에 맞게 계속해서 적절한 임계값을 찾아가는 것이 중요하다는 생각이 든다   
실제로 경험해보면서 적절한 수치를 찾아가야할듯하다   

<br/><br/>

## 추후 고려해야할 점  
이번에 구축한 Prometheus와 Grafana 기반의 모니터링 시스템은 효율적이지만, 몇 가지 한계점과 개선 가능성을 염두에 두고 있다   

<br/>

- 확장성 문제  

Prometheus는 단일 서버에 데이터를 저장하는 구조(single server node)이므로, 수집해야 할 메트릭 데이터의 양이 급격히 증가하거나 대규모 클러스터 환경을 모니터링할 때 성능 문제가 발생할 수 있다    
스케일링이 필요한 경우, Prometheus의 페더레이션(Federation) 또는 Thanos와 같은 도구를 도입해 스케일링 전략을 검토해야 한다   

<br/>

- 데이터 보존 기간   

Prometheus는 장기 데이터 저장에 한계가 있다   
기본적으로 데이터를 로컬 디스크에 저장하며, 디스크 용량에 따라 보존 기간이 제한된다    
장기적인 데이터를 기반으로 분석이 필요하게 된다면 데이터 보존을 위한 외부 스토리지 솔루션(ex) AWS S3와 Thanos, VictoriaMetrics) 도입을 검토해보아야 한다    

<br/>

- 로그 수집 및 분석  

현재는 메트릭 기반의 모니터링에 초점을 맞추고 있지만, 로그 수집 및 분석을 한다면 더욱 심층적인 원인 분석과 문제 해결이 가능해질 것이다    
이를 위해 ELK Stack (Elasticsearch, Logstash, Kibana) 또는 Loki와 같은 로그 수집 도구를 통합하여 로그와 메트릭 데이터를 함께 분석하는 환경을 구축하려고 한다   

<br/>

- Datadog 도입 검토  

Prometheus와 Grafana는 높은 커스터마이징 기능을 제공하지만, 유지보수에 시간이 소요된다   
필요에 따라 서비스 규모가 커지거나 분산된 마이크로서비스 환경으로 더 확장된다면 더 높은 자동화와 예측 분석 기능을 제공하는 Datadog 도입을 검토해볼 수 있다   

<br/><br/>

## 참고) Metric

Metric 은 간단히 말해 현재 시스템의 상태를 알 수 있는 측정값    
이를 통해 시스템이 잘 작동하고 있는지, 문제가 발생했는지를 파악할 수 있다     

<br/>

- 시스템 레벨 메트릭 (System-Level Metrics) : 서버나 인프라의 상태를 측정하는 값   
  - ex)
    - CPU 사용률: 현재 서버의 CPU가 얼마나 사용되고 있는지  
    - 메모리 사용량: 사용 중인 메모리와 가용 메모리
    - 디스크 I/O: 디스크 읽기/쓰기 속도
    - 네트워크 트래픽: 들어오고 나가는 데이터 전송량
    - 서버 가용성: 서버가 정상적으로 작동 중인지

- 애플리케이션 레벨 메트릭 (Application-Level Metrics) : 애플리케이션의 상태나 성능을 측정하는 값   
  - ex)
    - HTTP 요청 속도: 초당 처리되는 HTTP 요청 수
    - 응답 시간(latency): 요청이 처리되어 응답을 받기까지 걸리는 시간
    - 오류율: 실패한 요청의 비율
    - DB 쿼리 시간: 데이터베이스 쿼리가 실행되는 데 걸리는 시간
    - 큐 대기 시간: 메시지 대기열에서 처리되지 않은 항목의 시간

<br/>

### Metric의 중요성
- 문제 탐지 : 시스템이 정상적으로 작동하지 않을 경우(ex) CPU 과부하, 디스크 부족) 이를 빠르게 감지
- 성능 최적화 : 메트릭을 분석해 병목 구간(ex) slow query, 과도한 네트워크 트래픽)을 파악하고 해결 가능
- 지속적인 모니터링 : 실시간 데이터 수집을 통해 장기적인 트렌드를 파악하고, 예측 가능한 문제를 방지  
- 알림(Alerts) 설정 : 특정 임계치를 초과하면 알림을 보내 문제에 즉각적으로 대응할 수 있음  

<br/><br/>

Reference   
- [Prometheus Docs](https://prometheus.io/docs/introduction/overview/)  
- [DevOps school) What is Prometheus and How it works?](https://www.devopsschool.com/blog/what-is-prometheus-and-how-it-works/)
- [Grafana github](https://github.com/grafana/grafana)

<br/>
