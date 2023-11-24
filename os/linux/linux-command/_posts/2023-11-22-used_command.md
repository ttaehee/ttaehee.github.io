---
title: OS) Linux command
excerpt: 최근에 자주 사용한 명령어들, 헷갈리지 않기 위해 정리해두기
---   

<br/>

- 최근에 리눅스 명령어를 쓸일이 꽤 있었는데
  안쓰다보면 까먹을거 같아서 정리해두려한다
  - 배포 관련해서 쉘스크립트 짤때의 기억이 잘 안나는게 현재 너무 안타까워서리 

<br/>

# Linux command

- `pwd` : print working directory, 현재 위치 경로 출력

<br/>

- `chmod 400 test.pem` : 권한 변경
  - chmod 뒤에 숫자 3개 : 차례대로 `나/그룹/전체`에 대한 권한을 의미
  - `read(4), write(2), execute(1)`
    - `4/0/0` : 나에게만 읽기 권한
      - ex) `744`: 나는 읽기, 쓰기, 실행 권한 / 그룹, 전체는 읽기 권한만
- pem키 too open 이라 인스턴스 못들어가~ 를 너무 많이 겪어서,,

<br/>

- `chown -R ec2-user testfile` : 소유권 변경
  - `R` : 디렉토리일 경우 하위 디렉토리 및 파일들까지 반영
- 보통 하위까지 묶어서 변경할 때가 많았다 

<br/>

- `ps` : process state, 실행중인 프로세스 목록과 상태 확인
  - `ps -ef | grep test` : 실행 중인 모든 프로세스 중 `test`가 포함된 프로세스 확인
    - `e` : 모든 프로세스 출력
    - `f` : 풀 포맷으로 보여줌 (UID, PID 등)
    - `grep` : 특정 문자열 찾는 명령어
    - `|` : 파이프라인 = 처리 결과를 가지고 추가로 이어서 처리 가능
      - terminal은 하나의 process이고
      - 명령어 실행은 해당 terminal의 정보를 기준으로 백그라운드에 자식 process들이 fork되어 명령어를 실행하게 됨
- 개발 서버, qa, stage 서버 모두 nginx로 서버 두개씩 번갈아 운영중인데, proxy pass 설정이 있으니 프로세스 보는법을 알아야, 그니까 어떤게 떠있나 확인해야 껐다켰다 할 수 있었다
  nginx conf.d에서 현재의 service url을 확인해도 되었다    
  
<br/>

- `netstat` : 네트워크 연결 상태 및 포트 정보 확인
  -  `netstat -antp | grep 80`
    - `a` : 모든 소켓 상태 정보
    - `n` (numeric) : 도메인 주소를 숫자로 출력 = 숫자 형태의 IP 주소로 출력
    - `t` (tcp) : tcp 소켓 중 연결된 소켓만 출력
    - `p` (program) : PID와 사용중인 program 출력 -> tcp 소켓을 열고있는 process 확인에 유용    
- 포트가 내가 원하는 애로 열려있는지 확인할 때 주로 썼다

<br/>

- `systemctl status test.service` : 서비스 상태 확인
  - status / stop / start / restart
  - Linux는 OS가 부팅되면서 여러가지 데몬들이 실행됨
    - `데몬` : 사용자가 직접적으로 제어하지 않고, 백그라운드에서 여러 작업을 하는 프로그램    
      이러한 데몬들을 Linux에서는 `service 파일로 설정`하여 실행, systemd라는 프로세스가 관리함
      -  `service` : 시스템 데몬 및 사용자 정의 데몬을 의미 = Linux OS가 부팅되었을 때, 생성되면서 종료될 때까지 실행되는 process및 설정 파일
      -  `systemctl` : service(데몬)들을 관리하는 명령어
- 서버 다운되었을 때 테스트할 일이 있어서 서버 껏다 킬때 썼다    
  /systemd/system 에서 service화 시킨것들을 볼 수 있었고, 이렇게 service로 등록되어 있어야 systemctl 명령어가 먹혔다    

<br/>

- `crontab` : 크론(주기적으로 실행되는) 작업을 예약하기 위해 사용되는 시스템 도구
  - 크론(cron)이라는 데몬 process를 사용하여 작동 
  - `crontab -e`
- 인스턴스 내에서 돌아가는 크론작업을 등록하기 위해 썼다

<br/>
