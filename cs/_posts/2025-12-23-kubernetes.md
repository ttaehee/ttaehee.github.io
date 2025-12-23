---
title: Kubernetes와 EKS 구조 및 동작 흐름 정리
excerpt: 구조, 동작 흐름, 권한(IRSA), 모니터링
---

<br/>

- eks 전환과정에서 서비스 연동을 위해 이 기회에 kubernetes 큰 동작흐름이나 개념을 파악해봄     

<br/>

## Kubernetes   
컨테이너를 실행하는 도구가 아님 컨테이너를 자동으로 정상 상태로 운영되게 만드는 시스템 (운영 자동화 엔진)     

- 엔진 역할
  
```
Desired State ≠ Current State    
→ Kubernetes가 지속 감시
→ 불일치 시 자동 보정
```

<br/>

### 핵심 역할
- 컨테이너 배포 (Deploy)
  - 배포 중 다운타임 발생 => 무중단 배포 
- 확장 (Scale)
  - 적정 컨테이너 개수 판단 필요 => 트래픽에 따라 자동 스케일링 
- 복구 (Self-healing)
  - 서버 장애 시 서비스 중단 => 죽은 컨테이너 자동 재기동
- 트래픽 연결 (Service)
  - 트래픽 연결 수동 구성 => Service 기반 트래픽 연결 (서비스 디스커버리)

=> 운영 책임 분리 = 운영 판단을 시스템에 위임     
사람은 원하는 상태(몇 개, 어떻게 연결, 어떤 설정 등)만 말함 = 선언적 시스템

<br/>

### 선언적 시스템

ex)   
원하는 상태 정의 : Pod 2개를 항상 유지

```yaml
replicas: 2
```

Kubernetes의 역할 : 항상 Pod 2개 상태 유지   
- Pod 하나 죽으면? => Kubernetes(Deployment/ReplicaSet)가 자동으로 재생성

<br/>

### Kubernetes 핵심 구성요소 (개발자가 다루는 리소스)

전체 구조
```
Cluster
 └ Node
    └ Pod (컨테이너)
       ↑
Deployment (Pod 관리자)
       ↑
Service (고정 접점)
```

<br/>

Cluster : Kubernetes가 동작하는 논리적 공간 (직접 실행은 안 함)
- 모든 리소스(Node, Pod, Service 등)의 최상위 묶음

<br/>

Node : Pod가 실제로 실행되는 실행 환경
- 컨테이너가 여기서 실행됨
- 형태 : VM/EC2
- 역할
  - CPU / Memory 제공
  - 컨테이너 런타임 실행

<br/>

Pod : 컨테이너 실행 최소 단위
- 보통 컨테이너 1개
-	생성·삭제가 매우 빈번
-	IP가 고정되지 않음
-	언제든 종료 후 재생성 가능
- 신뢰 대상이 아님 (Deployment만 믿기) => 로그/파일 등 영속 데이터 저장 금지

<br/>

Deployment : 운영 제어 지점
- 역할
  - Pod 개수 유지 (Replica 관리)
  - 재기동 (장애 시 Pod 재생성)
  - 롤링 배포 / 롤백 관리

```
 Deployment
 └ ReplicaSet
    └ Pod x N
```

<br/>

실무에서는 Pod를 직접 생성하지 않고 Deployment만 생성      

Deployment는 운영 판단을 시스템에 위임하기 위한 장치   
=> Pod만 사용하는 것은 Kubernetes의 자동화 엔진을 끄고 쓰는 것과 같음      
  = 운영 환경은 Deployment + Service 필수


<br/>

Service : Pod 앞단의 고정 네트워크 접점      
- 역할
  - Pod 그룹 추상화 
  - Service Discovery 제공 : Pod IP가 바뀌어도 접근 지점은 유지 가능
    - Client -> Service -> Ready Pod들 
  - 트래픽 분산 (Load Balancing)

<br/>

Namespace : 리소스 논리적 분리 단위 
- 환경/팀/서비스 단위 분리  
  - ex) dev / stage / prod

<br/>

### Control Plane (시스템 내부 판단 영역)
Kubernetes의 두뇌로, 원하는 상태(Desired State)를 저장하고 현재 상태와 비교해 판단·지시를 수행함

- 역할
  - API Server : Kubernetes의 유일한 진입점 (YAML 요청 수신 (kubectl apply))
  - Scheduler : 새로 생성된 Pod를 어떤 Node에 배치할지 판단
  - Controller Manager : 여러 컨트롤러(Deployment Controller, ReplicaSet Controller, Node Controller 등) 묶음 -> Desired State ↔ Current State 비교해서 어긋나면 조치 명령 생성
  - etcd : Kubernetes 상태 저장소 (Desired State 영속 저장)

<br/>

### Kubernetes 동작 흐름
1. 리소스 생성    
   - 사람이 Deployment / Service / HPA 등을 yml로 원하는 상태 선언

<br/>

2. Desired State 저장    
   - Kubernetes Control Plane에 Desired State로 저장됨  

<br/>

3. Deployment 가 운영 판단   
   - 상태를 계속 감시 -> Desired State와 어긋나면 자동 보정
     - Pod 스펙 변경    
     - 버전 전환 관리  
     - 이전 상태 기억     
     => 변경 사항에 따라 적절한 ReplicaSet 생성 or 기존 ReplicaSet 갱신
   - 배포 전략 결정
     - 롤링 업데이트 : 기존 Pod를 조금씩 줄이고, 새 Pod를 조금씩 늘리면서 서비스를 끊지 않고 버전을 교체하는 방식
     - 롤백

<br/>

4. ReplicaSet 이 상태 유지
   - ReplicaSet은 실행자이지 운영자가 아님 ReplicaSet은 판단하지 않고, Deployment의 판단 결과를 실행만 함   
   - Deployment가 지정한 ReplicaSet 기준으로 Pod의 개수와 스펙을 실제로 맞춤
   - 지정된 개수만큼 Pod 생성/삭제

<br/>

5. Node 스케줄링   
   - Scheduler가 생성된 Pod를 적절한 Node에 배치   

<br/>

6. Probe    
    - kubelet (Node 안의 Kubernetes 에이전트) 이 Liveness / Readiness Probe를 주기적으로 체크     

      - Liveness Probe : 살아있는가 = 컨테이너 생존 여부 판단
        - 실패 시 kubelet이 컨테이너 재시작 -> 반복 실패 시 Pod 상태 악화 -> 최종적으로 Pod 교체로 이어짐 
      - Readiness Probe : 트래픽 받아도 되나 = 서비스 가능 상태 판단
        - 실패 시, Service 가 트래픽 제외
          - Pod은 살아있으니 ReplicaSet이 실패 Pod 교체하지는 않음 service 에서만 제외됨
        - Readiness가 롤링 업데이트의 안전장치 = 새 Pod가 Ready 되기 전까지 기존 Pod를 줄이지 않음

<br/>

7. Service 연결  
   - Service가 Pod들을 하나의 서비스로 묶음

<br/>

8. 트래픽 처리     
   - 외부/내부 트래픽 유입 -> Service 가 트래픽 분산 => Pod 교체·재생성과 무관하게 서비스 지속 가능
   - Pod IP는 계속 바뀜, Service가 고정 접점 제공
   - Ready Pod만 대상으로 트래픽 분산 (Client → Service → Ready Pod들)

= 사람이 Pod를 띄우는 게 아님 상태 선언만 함 Kubernetes가 실행·배치·유지를 담당  
- Deployment만이 제어 지점
- Pod은 결과물

<br/>

### 장애 유형별 Kubernetes 복구 동작

- Pod Crash
  
```
Pod Crash
→ Liveness Probe 실패
→ kubelet이 컨테이너 재시작 (restartPolicy=Always)
→ 재시작 횟수 증가 (CrashLoopBackOff)
→ 계속 실패
→ kubelet이 Pod 상태 변경을 보고 (CrashLoopBackOff 등)
→ API Server에 상태 업데이트
→ Control Plane가 Pod 종료 인지
→ ReplicaSet 컨트롤러가 개수 불일치 감지
→ 새 Pod 생성 요청
→ Scheduler가 새 Pod를 Node에 배치
→ kubelet이 새 Pod 실행
```

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

<br/><br/>

## EKS (Elastic Kubernetes Service)    
Kubernetes를 AWS가 대신 설치하고 Control Plane을 관리해주는 서비스   
운영 자동화의 주체는 여전히 Kubernetes(Deployment)  

- 하는 일
  - Kubernetes Cluster 생성 = Kubernetes를 안전하게 실행할 수 있는 환경만 제공 (개념은 완전히 동일함 운영 책임 구조는 변하지 않음, EKS가 Pod를 운영해주는 것은 아님)    
  - Control Plane 생성/운영

<br/>
   
Control Plane : 판단·지시 > Node : 실행 > kubelet : 지시 수신 > 컨테이너 런타임 : 실제 컨테이너 실행      
EKS는 판단 영역을 AWS가 대신 운영해주는 서비스   

<br/> 

### EKS 기준 실제 구조

```
AWS EKS (서비스)
 └ Kubernetes Cluster (관리 범위)
    ├ Control Plane (AWS가 운영)
    │   ├ API Server
    │   ├ Scheduler
    │   └ Controller Manager
    │        └ Deployment / ReplicaSet 컨트롤러
    │
    └ Node (EC2 / Fargate)  ← 실제 실행 환경
         ├ kubelet
         ├ 컨테이너 런타임
         └ Pod / Container
```

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

EKS 환경에서 S3 접근하려고 하니 IRSA 설정 해주어야했음   

<br/> 

**SA(ServiceAccount)**   
Pod의 신분증 (이 Pod가 누구인지를 나타내는 계정)     
- Pod는 반드시 하나의 SA로 실행 (명시 안 하면 default SA 사용)     
- EKS에서는 AWS 권한 연결의 기준점

<br/> 

**IRSA**    
ServiceAccount 신원으로 IAM Role을 사용하게 하는 인증 메커니즘 

- Kubernetes ServiceAccount의 신원이 ↔ IAM Role 과 1:1 매핑됨     
- AccessKey/SecretKey 없이 IAM Role 기반 권한 사용  

<br/> 

1. Pod 실행 시 (1단계: Kubernetes 영역) : 어떤 ServiceAccount를 쓸지 결정됨 = ServiceAccount 변경하려면 Pod 재시작 필요

```
Pod 생성
→ ServiceAccount 지정
→ Kubernetes가 ServiceAccount 토큰(JWT) 발급
→ kubelet이 토큰을 Pod 파일시스템에 마운트
```

<br/> 

2. Pod 실행 후 AWS 호출 시 (2단계: AWS 연동) : IRSA가 실제로 동작하는 시점

```
애플리케이션에서 AWS SDK 호출
→ SDK가 ServiceAccount 토큰 발견
→ STS AssumeRoleWithWebIdentity 호출
→ IAM Role 검증 (OIDC + SA 조건)
→ 임시 자격증명 발급
→ AWS 리소스 접근
```

= Pod 실행 시 ServiceAccount가 결정 + ServiceAccount 토큰 준비, AWS 호출 시 IRSA를 통해 ServiceAccount 신원으로 IAM Role을 Assume 함

<br/> <br/> 

/// 모니터링 추가하기
