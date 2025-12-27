---
title: Kubernetes는 무엇을 어떻게 자동으로 해주는지
excerpt: 상태(State) 기반 시스템으로서의 Kubernetes와 EKS
---

<br/>

- EKS 전환과정에서 서비스 연동을 위해 이 기회에 kubernetes 큰 동작흐름이나 개념을 파악해봄      
  올만에 TIL   

<br/>

## Kubernetes   
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

### 핵심 역할
- **컨테이너 배포 (Deploy)** : 배포 중 다운타임 발생 => 무중단 배포 
- **확장 (Scale)** : 적정 컨테이너 개수 판단 필요 => 트래픽에 따라 자동 스케일링 
- **복구 (Self-healing)** : 서버 장애 시 서비스 중단 => 죽은 컨테이너 자동 재기동
- **트래픽 연결 (Service)** : Service 기반 트래픽 연결 (서비스 디스커버리)

<br/>

=> **운영 책임** 분리 = 운영 판단을 시스템에 위임        
사람은 원하는 상태(몇 개, 어떻게 연결, 어떤 설정 등)만 말함 

<br/>

= 선언적 시스템      
- manifest : Kubernetes 리소스의 Desired State 를 정의한 문서
    - ex) 사람이 원하는 상태 정의 : Pod 2개를 항상 유지 = 선언   

    ```yaml
    replicas: 2
    ```
    
    => Kubernetes의 역할 : 결과적으로 항상 Pod 2개 유지  

<br/>

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

<br/>

**HPA (Horizontal Pod Autoscaler)**    
Deployment의 replicas를 자동 조정하기 위해 추가로 정의하는 운영 정책 리소스

<br/>

```
HPA 리소스 (정책)
→ HPA Controller가 읽음
→ replicas 값 변경
→ Deployment의 replicas 값 변경 → ReplicaSet이 Pod 생성/삭제
```

- HPA 역할 : replicas 숫자 변경 (컨테이너 재시작 x)
- HPA가 보는 기준 : CPU 사용률 (가장 기본) / Memory 사용률 / Custom Metric (QPS, 요청 수 등)

<br/>

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
-> (kubelet : 지시 수신) 판단한대로 실행 (container runtime : 실제 컨테이너 실행)    
-> 실행된 결과를 서비스로 묶음    
-> 계속 현재상태와 원하는 상태 비교 & 자동 보정      

<br/>

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

컨테이너가 죽으면 kubelet이 먼저 재시작을 시도   
-> 계속 실패하면 Control Plane이 Pod 종료를 인지해    
-> ReplicaSet이 새 Pod를 생성하도록 자동 보정      

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

<br/>

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

<br/> <br/> 
