---
title: Kubernetes는 무엇을 어떻게 자동으로 해주는지
excerpt: 상태(State) 기반 시스템으로서의 Kubernetes, EKS
---

<br/>

- EKS 전환과정에서 서비스 연동을 위해 이 기회에 kubernetes 큰 동작흐름이나 개념을 파악해봄    
  그리고 로그나 모니터링 어찌해야하는지 파악할겸           
  올만에 TIL   

<br/>

## Kubernetes   
가장 널리 사용되고 있는 오픈소스 컨테이너 오케스트레이션 도구       
컨테이너를 **자동으로 정상 상태로 운영**되게 만드는 시스템 (운영 자동화 엔진)    
단순히 컨테이너를 실행하는 도구가 아님     

<br/>

- 엔진 역할
  
```
Desired State ≠ Current State    
→ Kubernetes가 지속 감시
→ 불일치 시 자동 보정
```

<br/>

### Kubernetes를 왜 쓰는지 (운영의 시스템화)   
Kubernetes 도입의 핵심은 **운영 주체의 전환(Human → System)**    
운영을 '사람이 직접 수행하는 일'에서 '시스템이 스스로 유지하는 상태'로 전환하기 위함            

<br/>

**핵심 역할** (시스템이 대신 하는 일)        
- **컨테이너 배포 (Deploy)** : 배포 중 다운타임 발생 => 무중단 배포 
- **확장 (Scale)** : 적정 컨테이너 개수 판단 필요 => 트래픽에 따라 자동 스케일링 
- **복구 (Self-healing)** : 서버 장애 시 서비스 중단 => 죽은 컨테이너 자동 재기동
- **트래픽 연결 (Service)** : Service 기반 트래픽 연결 (서비스 디스커버리)

<br/>

=> **운영 책임** 분리       
Kubernetes는 사람이 운영을 직접 수행하지 않고, 운영을 정책으로 선언하게 만들기 위해 쓰는 시스템     
전통적인 운영이 관리자의 경험과 판단에 의존했다면, Kubernetes는 운영을 **규칙과 자동화로 고정**함     
사람은 원하는 상태(몇 개, 어떻게 연결, 어떤 설정 등)만 말함  
  
<br/>

= 선언적 시스템      
- manifest : Kubernetes 리소스의 Desired State 를 정의한 문서
    - ex) 사람이 원하는 상태 정의 : Pod 2개를 항상 유지 = 선언   

    ```yaml
    replicas: 2
    ```
    
    => Kubernetes의 역할 : 결과적으로 항상 Pod 2개 유지  

<br/><br/>
               
=> 도입 가치      
- Human Error 방지 (사람의 개입을 최소화하여 사람의 실수와 반복 작업을 감소)
- 안정성 확보 (장애·스케일·복구를 항상 동일한 규칙으로 처리)
- 개발 생산성 향상 (인프라 운영 부담 감소)
- 비용 및 리소스 최적화 (서버 자원을 최대한 효율적으로 나누어 사용하여 인프라 비용 절감)

<br/><br/>

### Kubernetes 구조

```
Cluster                         : k8s 가 동작하는 전체 논리적 공간 (모든 리소스의 최상위 묶음)
├ Control Plane                 : 두뇌, Desired State 저장 & 판단 지시
│  ├ API Server                 : 리소스 생성·수정 요청 진입점
│  ├ Scheduler                  : 새로 생성된 Pod 을 Node 자원 보고 어떤 Node 에 배치할지 결정
│  └ Controller Manager         : Desired, Current State 지속 비교 -> 어긋나면 보정 명령
│     ├ Deployment Controller   : Deployment spec 해석 -> ReplicaSet spec 생성/갱신
│     └ ReplicaSet Controller   : ReplicaSet spec 해석 -> Pod 생성/삭제
│
├ Node                          : Control Plane 지시 받아서 Pod 이 실제 실행되는 실행환경 (물리/가상 서버)
│  ├ kubelet
│  ├ container runtime
│  └ Pod
│     ├ 컨테이너 (실행 프로세스)         - 실행
│     ├ 네트워크 (IP / Port)          - 연결
│     ├ Volume                     - 데이터
│     ├ ServiceAccount / Namespace  - 권한
│     └ 생명주기 (생성 / 재시작 / 교체)   - 운영
│
└ Kubernetes Resources (논리 객체)
   ├ Deployment                 : Pod 운영 정책
   ├ ReplicaSet                 : Pod 개수 유지 정책
   └ Service                    : 고정 네트워크 접점, 트래픽 분산, 서비스 디스커버리 담당 -> 무중단 배포 가능   
```

<br/><br/>

**HPA (Horizontal Pod Autoscaler)**    
Deployment의 replicas를 자동 조정하기 위해 추가로 정의하는 운영 정책 리소스

<br/>

```
HPA 리소스 (정책)
→ HPA Controller가 메트릭 기준으로 판단
→ Deployment.spec.replicas 값 변경
→ Deployment / ReplicaSet이 Pod 생성·삭제
```

- HPA 역할 : replicas 숫자 변경 (컨테이너 재시작 x)
- HPA가 보는 기준 : CPU 사용률 (가장 기본) / Memory 사용률 / Custom Metric (QPS, 요청 수 등)

<br/><br/>

### ConfigMap / Secret (설정과 민감정보 관리)
그러면 이제 실행될 때 어떤 설정으로 실행될지는 어디에서 관리하느냐   
Pod는 코드와 이미지로만 실행되지 않고, 실행 시점에 외부 설정을 함께 주입받음   
- ConfigMap / Secret : 실행에 필요한 설정과 민감 정보를 관리하기 위한 리소스
  - Kubernetes에서는 애플리케이션 설정을 코드·이미지에 포함하지 않음 설정은 리소스로 분리해 관리함
  - **ConfigMap** : 일반 설정값 (환경변수, 설정 파일 등)
    - ex) feature flag, endpoint, timeout
  - **Secret** : 민감 정보
    - ex) DB 비밀번호, API Key, 토큰

<br/>

**Pod에서 사용하는 방식**    
Pod는 ConfigMap/Secret을 
- 환경변수(env)로 사용하거나
- 파일(volume)로 마운트해서 사용함

<br/>

**설정 변경 시 특징**   
- Pod 재시작 필요
- 이미지 재빌드 불필요
- 설정만 교체 가능 

<br/>

=> 코드와 설정을 분리해 동일한 이미지로 환경별(dev/qa/prod) 운영이 가능해짐    

<br/><br/>

### Kubernetes 동작 flow   

```
사람이 원하는 상태 Desired State (Deployment, Service, HPA 등)를 manifest로 선언
→ API Server가 리소스 생성·수정 요청을 수신하고 etcd에 Desired State를 영속화
→ Deployment Controller가 Deployment spec을 해석해 필요한 ReplicaSet을 생성 또는 갱신
→ ReplicaSet Controller가 ReplicaSet spec을 기준으로 Pod를 생성·삭제
→ Scheduler가 생성된 Pod를 적절한 Node에 배치
→ Node의 kubelet이 Pod 실행 지시를 받아 container runtime으로 컨테이너를 실행
→ kubelet이 Liveness / Readiness Probe로 컨테이너 상태를 점검
→ Service가 Ready 상태의 Pod에만 트래픽을 연결
→ Controller들이 Desired State ≠ Current State를 지속 감시하며 자동 보정
```

<br/>

요약하면   

<br/>

사람이 원하는 상태 선언    
-> 이 상태 유지하는데 뭐필요한지 kubernetes 가 판단 (Control Plane : 판단·지시)     
-> 판단한대로 실행 (kubelet : 지시 수신 -> container runtime : 실제 컨테이너 실행)    
-> 실행된 결과를 서비스로 묶음    
-> 계속 현재상태와 원하는 상태 비교 & 자동 보정      

<br/><br/>

### 장애 유형별 Kubernetes 복구 동작

- Pod Crash
  
```
Pod Crash
→ Liveness Probe 실패
→ kubelet이 컨테이너 재시작 (restartPolicy=Always)
→ 계속 실패 & 재시작 횟수 증가 (CrashLoopBackOff)
→ kubelet이 Pod 상태 변경을 보고 (CrashLoopBackOff 등) → API Server에 상태 업데이트
→ Control Plane가 Pod 종료 인지
→ ReplicaSet 컨트롤러가 개수 불일치 감지
→ 새 Pod 생성 요청
→ Scheduler가 새 Pod를 Node에 배치
→ kubelet이 새 Pod 실행
```

<br/>

요약하면  

<br/>

컨테이너가 죽으면 재시작     
-> 계속 실패하면 새 Pod 생성해서 자동 보정       

<br/>

+) Pod이 Crash 나면 Endpoint Controller가 해당 Pod의 IP를 Service의 연결 목록(Endpoints)에서 즉시 제거함

<br/>

- Node Down

```
Node Down
→ kubelet / Node 응답 끊김
→ Node Controller가 Node NotReady 감지
→ 일정 시간 대기 (Node Monitor Grace Period)
→ 해당 Node의 Pod들을 Lost/Unknown 처리
→ ReplicaSet이 Pod 개수 부족 인지
→ 새 Pod 생성 요청
→ Scheduler가 다른 Node 선택
→ 새 Node의 kubelet이 Pod 실행
```

<br/>

요약하면   

<br/>

Node가 죽으면 해당 Node의 Pod은 모두 소멸로 판단    
-> ReplicaSet이 다른 Node에 Pod를 재생성해 원하는 개수를 다시 맞춤    

<br/><br/>

### CI/CD   

EC2 + EB 관점에서 CD는 서버에 jar 올림 -> 기존 프로세스 종료 & 새 jar로 프로세스 재시작 = 코드 배포 느낌, 배포 대상이 서버였는데         
Kubernetes 관점에서 CD는 서버에 무언가를 올리는 행위가 아니라 **Git에 정의된 상태로 바꾸는** 행위    
배포 대상이 상태(Desired State = 어떤 코드를 담은 이미지를 실행할지에 대한 상태)임     
= Git 에 정의된 Desired State ↔ Kubernetes 상태를 동기화하는 과정      
(둘다 동일한 개념이긴 한데 좀 헷갈려서 명확히 정리해봄)       
CI 는 둘 다 동일하게 실행가능한 산출물 만드는거     

<br/>

| 구분     | EC2 + EB | Kubernetes          |
| ------ | -------------------- | ---------------------- |
| CI 산출물  | jar                   | 이미지 |
| 배포 대상  | 서버                   | 상태 (Desired State) |
| 코드 위치  | 서버 파일시스템             | 컨테이너 이미지           |
| CD 의미  | jar 업로드 + 프로세스 재시작        | Desired State를 담고 있는 객체(Deployment spec) 변경   |
| 실행 주체  | 사람 / 스크립트            | Controller가 자동 교체  |
| Git 역할 | 코드 저장소               | Desired State 정의 저장소   |

<br/><br/>

그러면 누가 Kubernetes에 apply 하느냐 = 이 상태 동기화를 누가 수행하느냐         
외부에서 적용하는 Push 방식과 클러스터 내부에서 동기화하는 Pull 방식이 있음   

- **Push 방식** : 외부(CI)가 Git 읽고 kubectl apply 실행 = 트리거 느낌
- **Pull 방식** : 클러스터 내부에서 ArgoCD가 Git 감시하면서 상태 비교해서 apply 실행 
   
=> Kubernetes에서 CD는 상태 동기화를 지속적으로 안전하게 수행해야 하므로, 보안·운영 안정성(외부로 클러스터 권한을 노출할 필요 없음) 측면에서 Pull 방식 CD가 가장 적합해 주로 사용

<br/><br/>

**Pull 방식 CD (ArgoCD / Flux)**        

- ArgoCD : Git에 정의된 Kubernetes 리소스(Deployment, Service 등)를 기준으로 Kubernetes의 실제 상태를 **지속적으로 일치시키는 GitOps 기반** CD 도구 (상태 동기화 엔진)

```
Git (Desired State의 원본) 의 manifest 변경

↓

ArgoCD / Flux (동기화 엔진) 가 변경 감지 & Kubernetes 현재 상태와 비교

↓ (다르면 ArgoCD가 Kubernetes API를 통해 apply 요청)

Kubernetes : Git에 정의된 Desired State로 수렴
```      

<br/>

참고) GitOps : 시스템을 명령으로 운영하지 않고, **Git에 정의된 상태를 기준으로 시스템이 변경 스스로 감지**해서 Git과 **일치하게** 하는 운영 방식      
=> Git이 곧 현재 운영 상태임    

= 변경 이력 추적, 롤백 단순화, 보안 강화, 운영 안정성 제공   

<br/><br/>

## 왜 Kubernetes에서는 로그 중앙화가 필수인지

분산 시스템 환경에서 로그는 단순한 텍스트 출력이 아니라, 시스템과 application이 남기는 사건의 기록(event record)        
MSA라서 로그중앙화가 필요하지만 Kubernetes 환경에서는 필수   

<br/>

Kubernetes는 자동 복구(Self-healing)는 해주지만, 왜 그런 일이 발생했는지는 알려주지 않음 원인분석은 기존처럼 로그 기반으로 파악해야 함     
kubectl logs는 특정 Pod 단위의 부분 확인용임   

<br/>

근데 Pod는 휘발성이라 Pod 사라지면 해당 Pod의 로컬 로그 파일도 같이 소멸됨    
+) 장애 원인은 특정 Pod 하나가 아니라 여러 Pod, 여러 Node, 여러 서비스에 걸쳐서 발생함   

=> 원인 분석하려면 Pod 단위 로그를 Cluster 단위로 검색 가능하게 만들어야함       
   서버 들어가서 볼 수 없음 중앙에 모아두지 않으면 장애 분석 자체가 불가능      

= 로그 중앙화가 선택이 아니라 필수    
   
<br/> 

### Kubernetes 로그 수집

```
application log는 stdout/stderr 로만 출력    
-> Kubernetes가 노드의 로그 파일로 자동 기록    
-> 수집기는 노드의 /var/log/containers(컨테이너 로그 파일)를 읽어서 중앙화
```

= application은 출력만 하고, 수집·보관·검색은 플랫폼(Kubernetes + 로그 시스템) 이 담당

<br/> 

장애 추적을 위한 필수 로그 필드는 기존 MSA와 동일하지만, Kubernetes 에서는 원인뿐 아니라 어디에 몰리는지(범위/분포) 도 봐야함    
- ex) 접근방식
  - 특정 Pod/노드에만 몰림 → 배포/노드/리소스 문제 의심
  - 전체 Pod에 고르게 발생 → 코드/DB/외부의존성 문제 의심

<br/> 

**참고) 로그 중앙화(Centralized Logging)**      
여러 곳에서 발생하는 로그를 한 곳에 모아 저장·검색·분석하는 운영 방식

- 대상 : server / Pod / container / application log 전부
=> 각 서버/Pod에 들어가서 로그 보는 방식은 운영 불가

<br/> 

### EFK     
Kubernetes에서 자주 쓰이는 로그 중앙화를 구현하는 대표적인 구현체   

- E (Elasticsearch) : 저장 + 검색
- F (Fluentd / Fluent Bit) : 수집 + 전송
- K (Kibana) : 조회 + 시각화

<br/> 

EFK 는 특정 기술들의 조합(Stack)일 뿐, 중요한 것은 수집 - 저장 - 시각화 라는 로그 중앙화의 3단계 구조(Architecture)      
실무에서는 상황에 따라 각 단계를 Datadog, OpenSearch 등 다른 도구로 대체해 사용     

<br/> 

```
App / Pod : 로그 생성 (stdout/stderr)
→ Fluent Bit : 로그 수집 - Datadog Agent 로 대체하여 사용
→ Elasticsearch : 로그 저장, 검색 - OpenSearch 로 대체 (Datadog logs 안씀)
→ Kibana : 로그 대시보드, 조회, 분석, 알림 - Datadog UI 로 대체
```

- 참고) 수집기인 Datadog Agent는 로그(stdout)뿐만 아니라 시스템의 수치 데이터인 메트릭(CPU, Memory 등)도 함께 수집함

<br/> <br/> 

## EKS (Elastic Kubernetes Service)    
Kubernetes **Control Plane을 AWS가 관리형으로 제공**하여 사용자가 직접 설치·운영하지 않아도 Kubernetes Cluster를 사용할 수 있게 해주는 서비스               

=> 개념은 완전히 동일함 운영 책임 구조는 변하지 않음 (운영 자동화의 주체는 여전히 Kubernetes)      
   
   = EKS가 Pod를 운영해주는 것은 아님 Control Plane 운영 부담만 AWS가 대신 맡아주는 서비스       

<br/> 

| Kubernetes 개념 | EKS에서의 실체 |
|---|---|
| Cluster | EKS Cluster |
| Control Plane | AWS가 관리 |
| Node | EC2 Node Group / Fargate |
| Deployment | 동일 (YAML 그대로) |
| Pod | 동일 |
| Service | Kubernetes와 동일 |

<br/> 

### EKS + IRSA + SA(ServiceAccount) 흐름   

EKS 환경에서 S3 접근하려고 하니 IRSA 설정을 해주어야했음    

<br/> 

- **SA (ServiceAccount)** : Pod의 신분증     
  - EKS에서는 AWS 권한 연결의 기준 단위    
  - Pod는 반드시 하나의 SA로 실행 (지정 안 하면 default SA 사용)   
    - Pod 실행 시 어떤 SA 쓸지 결정됨      


- **IRSA (IAM Roles for Service Accounts)** : ServiceAccount ↔ IAM Role 매핑해주는 규칙          

<br/> 

EKS에서 AWS 권한은 **Pod의 ServiceAccount 신원을 기준**으로 IRSA를 통해 **IAM Role을 Assume하여** 사용함   
이 과정에서 STS(AWS Security Token Service)가 발급해준 임시 자격증명(AccessKey / SecretKey / SessionToken)으로 AWS 리소스 접근   
= 키 관리 없이 **IAM Role 기반**(누구인지 증명하면 그 role 에 붙은 권한으로 행동하게 하는 방식)으로 동작    

<br/><br/> 
