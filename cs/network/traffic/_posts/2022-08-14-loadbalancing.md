---
title: CS) Load Balancing
excerpt: 로드밸런싱
---
 
## Load Balancing (로드밸런싱)
서버가 처리해야할 요청(load)을 `여러대의 서버로` 트래픽을 균등하게 분산해주는(balancing) 처리 <br/><br/>

## Load Balancer (로드밸런서)
로드밸런싱 기술을 제공하는 서비스 또는 장치  

![제목 없음](https://user-images.githubusercontent.com/103614357/184502768-9d7d24f0-74f9-4d0d-9a15-2725f9cc2526.png)  

- 클라이언트와 서버(트래픽이 집중된) or 네트워크 허브 사이에 위치 <br/><br/>

### 로드밸런서의 기본동작 방식
1. `클라이언트의 브라우저`에서 ttaehee.net 입력
2. `메인 DNS서버`로 ttaehee.net IP주소 문의  
  -> ttaehee.net 주소를 관리하는 별도의 DNS서버에 IP주소 문의  
  ->`별도관리 DNS서버`가 메인 DNS서버에게 `로드밸런서의 IP(Virtual IP)` 주소 알려줌  
3. 메인 DNS서버가 Virtual IP주소를 클라이언트에게 전송
4. 클라이언트에서 Virtual IP주소로 Http 요청
5. 로드밸런서는 별도 로드밸런싱 방법을 통하여 서버에게 요청전송  
  ->로드밸런서는 서버의 작업 후 결과를 클라이언트에게 전송 <br/><br/>


### L4 로드밸런서

![제목 없음](https://user-images.githubusercontent.com/103614357/184503020-8e17e84d-bf86-4322-96bf-0668c736826c.png)  

- 네트워크계층(IP)이나 전송계층(TCP, UDP)에서 로드 분산  
  = `IP주소`나 `포트번호`, `MAC주소` 등에 따라 트래픽 나누고 분산처리
- CLB(Connection Load Balancer) or SLB(Session Load Balancer)라고 부르기도 함 

- 장점  
  속도가 빠름/ 효율이 높음/ 데이터내용 복호화할 필요 없어 안전/ 가격 저렴
- 단점  
  - 섬세한 라우팅 불가(패킷 내용 확인하지 않으니까)
  - 사용자의 IP가 수시로 바뀌는 경우라면 연속적인 서비스 제공 어려움
 
<br/><br/>

### L7 로드밸런서  

![제목 없음](https://user-images.githubusercontent.com/103614357/184503104-5bf8f4f4-067b-4f0e-8f62-b185c9caa790.png)  

- L4 로드밸런서의 기능 + `애플리케이션 계층(HTTPS, SMTP)`을 바탕으로도 분산처리 가능
  - HTTP header, Cookie 등과 같은 사용자요청을 기준으로 특정서버에 트래픽 분산이 가능
    - URL Switching method : `특정 하위URL`들은 특정 서버로 처리   
      (ex) /image : 이미지처리 서버, /video : 동영상처리 서버)
    - Context Switching method : 클라이언트가 요청한 `특정 리소스`(확장자 참조)에 대해 특정서버로 연결
    - Persistence with Cookies : `쿠키정보 바탕`으로 클라이언트가 연결했었던 동일한 서버에 계속 할당

- 장점
  - 상위계층에서 로드분산하기 때문에 섬세한 라우팅 가능
  - 캐싱기능 제공
  - 비정상적인 트래픽(Docs/DDos)을 사전에 필터링할 수 있어 서비스안정성 높음
  - 특정한 패턴을 지닌 바이러스를 감지해 네트워크 보호가능
- 단점
  - 비쌈
  - 패킷내용 복호화해야함(비용)
  - 클라이언트가 로드밸런서와 인증서 공유해야하기 때문에 보안상의 위험성 존재

<br/>

## Load Balancing Algorithm

**1. Round Robin method (라운드로빈 방식)**   
요청을 `순서대로` 돌아가며 분배  

- 서버들이 동일한 스펙 갖고 있고, 세션(서버와의 연결)이 오래 지속되지 않는 경우 적합   <br/><br/>

**2. Weighted Round Robin method (가중 라운드로빈 방식)**    
`서버마다 가중치` 매김 -> 가중치 높은 서버에 요청 우선 배분  

- 서버의 트래픽 처리능력이 상이한 경우
  - ex) 서버A 가중치 : 5 / 서버B 가중치 : 2 => A서버에 5개 요청, B서버에 2개 요청 할당 <br/><br/>

**3. IP Hash method(IP해시 방식)**    
`클라이언트의 IP주소` 를 특정서버에 매핑(해싱)   
(hashing : 임의의 길이를 지닌 데이터를 고정된 길이의 데이터로 매핑)  

- 경로 보장(사용자가 항상 동일한 서버로 연결)
- 접속자 수 많을수록 분산 및 효율 뛰어남 <br/><br/>

**4. Least Connection method (최소연결 방식)**    
요청 들어온 시점에 `가장 적은 연결(세션)` 상태의 서버에 우선 배분  

- 트래픽이 일정하지 않은 경우 적합 <br/><br/>

**5. Least Response Time method (최소 응답시간 방식)**    
가장 적은 연결상태와 `가장 짧은 응답시간`을 보내는 서버에 우선 배분  

- 서버의 성능, 처리중인 데이터양 등이 상이할 경우 적합 <br/><br/>

**6. Bandwidth method(대역폭 방식)**   
서버들과의 `대역폭` 고려해 배분 <br/><br/>

<br/><br/>

Reference  
https://co-no.tistory.com/22?category=1069105  
https://www.stevenjlee.net/2020/06/30/%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC%EC%9D%98-%EB%B6%80%ED%95%98%EB%B6%84%EC%82%B0-%EB%A1%9C%EB%93%9C%EB%B0%B8%EB%9F%B0%EC%8B%B1-load-balancing-%EA%B7%B8/  
https://velog.io/@yanghl98/OS%EC%9A%B4%EC%98%81%EC%B2%B4%EC%A0%9C-%EB%A1%9C%EB%93%9C%EB%B0%B8%EB%9F%B0%EC%8B%B1-Load-Balancing-%EC%A0%95%EC%9D%98-%EC%A2%85%EB%A5%98-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98  
<br/>
