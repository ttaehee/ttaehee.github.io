---
title: CS) Data Transfer Protocol
excerpt: IP, TCP, UDP, ICMP
---

# 데이터전송 프로토콜

![제목 없음](https://user-images.githubusercontent.com/103614357/183255734-ad7fc183-e5ea-4e8f-ac3d-ab273c9518da.png)  


## 1. IP(Internet Protocol)
- 비연결 지향성 프로토콜
- IP주소를 사용해서 장치를 고유하게 판별하여 데이터 전송
- 로컬환경에서 리모트환경으로 데이터 전송 담당
  - 나와 상대의 주소를 지정, 패킷이 올바른 목적지에 도착하는지 확인하는 데 사용하는 프로토콜 <br/><br/>

-  패킷이 오염되고, 손실되고, 순서없이 도착했을 경우 IP가 해줄 수 있는 일은 없기때문에 IP 위에서 더 높은 수준의 프로토콜을 사용해야 함(IP위에서 돌아가는 TCP, UDP protocol) <br/><br/>


## 2. TCP(Transmission Control Protocol, 전송제어 프로토콜)
- 연결형 서비스 지원(전화와 유사) : 가상 회선 방식, 상대방과 통신수립 연결 실시 후 데이터 요청응답 실시
  - 데이터 송수신 전에 소켓을 통한 연결 필요
  - 소켓/포트로 동시에 여러개의 연결 지원
    - HTTP(80), SMTP(25), POP3(110), FTP(20,21)
- 3-way handshaking 과정을 통해 연결설정 / 4-way handshaking 과정을 통해 연결해제
- 인터넷상에서 데이터를 메세지의 형태로 보내기 위해 IP와 함께 사용 => TCP/IP  
  - IP가 데이터의 배달 처리
  - TCP는 패킷(데이터를 여러 개의 조각들로 나누어 전송할 때의 조각)을 추적,관리 <br/><br/> 

**특징**  
데이터의 흐름제어 가능
높은 신뢰성(패킷의 안정적인 전송 보장)  
UDP보다 속도 느림  
전이중(Full-Duplex), 점대점(Point to Point)방식 <br/><br/>


## 3. UDP(User Datagram Protocol)
- 비연결형(편지배달과 유사) 서비스 : 데이터그램 방식
  - 상대 디바이스가 데이터를 잘 받았는지 확인조차 하지않음
- 실시간 서비스(streaming)에 자주 사용 <br/><br/> 

**특징**  
TCP보다 속도 빠름 
신뢰성낮음
가벼움 <br/><br/>


## 4. ICMP(Internet Control Message Protocol, 인터넷 제어메시지 프로토콜)  
오류메시지 전송받는데 주로 쓰임  
- TCP/IP에서 paket을 처리할 때 발생되는 문제발생  
  (해당 host가 없거나, 해당 port에 대기중인 Server program이 없는 등 에러)      
  -> IP에는 이러한 에러 처리 방법이 명시되어 있지 않음  
  -> ICMP가 IP header에 기록되어 있는 출발지 host로 에러 상황을 보내주는 역할 <br/><br/>

**명령어**
1. Ping : 상대호스트의 작동여부 및 응답시간 측정
2. Tracert : 목적지까지의 라우팅경로 추적


<br/><br/>

Reference  
https://velog.io/@goban/%EA%B8%B0%EB%B3%B8%EC%A0%81%EC%9D%B8-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EA%B0%9C%EB%85%90  
https://mangkyu.tistory.com/15?category=762469  
https://btyy.tistory.com/entry/NETWORK-4%EC%9D%BC%EC%B0%A8-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EC%A0%84%EC%86%A1-%ED%94%84%EB%A1%9C%ED%86%A0%EC%BD%9C-TCP-UDP-IP-ICMP-ARP  
https://matamong.tistory.com/entry/%EC%9D%B8%ED%84%B0%EB%84%B7%EC%9D%98-%EC%9E%AC%EB%A3%8C-2%ED%83%84-IP%EC%9C%84%EC%97%90%EC%84%9C-%EB%8F%8C%EC%95%84%EA%B0%80%EB%8A%94-TCPUDP-%ED%94%84%EB%A1%9C%ED%86%A0%EC%BD%9C?category=965440  
https://run-it.tistory.com/31  
<br/>
