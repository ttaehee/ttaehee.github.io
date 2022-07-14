---
title: Spring) Mac command - sudo, yum
excerpt: mac 터미널 명령어
---
# Mac 터미널 명령어2
## sudo  
: root권한(관리자권한)  
- `chgrp [options] group file_list` : 파일 또는 디렉토리의 소유그룹을 변경  
- `chown [options] group file_list` : 파일 또는 디렉토리의 소유계정을 변경 <br/><br/>

옵션 
- `c` : 변경된 내용 출력
- `R` : 디렉토리일 경우 하위 디렉토리 및 파일들까지 반영
- `v` : 변경 또는 변경되지 않은 내용 출력

## YUM (Yellowdog Updater Modified) 
: RPM기반의 시스템을 위한 자동 업데이터, 패키지 설치/삭제도구  
- `yum -h`, `yum help` : yum에대한 사용법 도움말확인 (help)
- `yum install 패키지` : (yum을 통한) 시스템으로 패키지 설치 실시, 여러개 가능
- `yum reinstall 패키지` : 패키지 재설치
- `yum localinstall` : 로컬에 설치
- `yum erase 패키지` : 시스템에서 패키지삭제
- `yum list` : 서버에 있는 패키지 리스트  
	ex) yum list all : 설치 가능한 모든 패키지 목록  
	ex) yum list installed 패키지명 : 패키지 설치여부 확인  
	ex) yum list updates : 업데이트목록 <br/><br/>

- `yum update 패키지` : 패키지 업데이트
- `yum check-update` : 현재 설치된 프로그램 중 업데이트 된 것 체크해줌
- `yum upgrade 패키지` : 패키지 업그레이드
- `yum downgrade 패키지` : 패키지 다운그레이드
- `yum search 키워드` : 키워드로 시작하는 패키지 검색 <br/><br/>

- `yum makecache` : 캐쉬 다시 올림
- `yum clean all` : 캐시 되어 있는 것 모두 삭제
- `yum info 패키지` : 패키지의 정보 <br/><br/>

- `yum groupinstall` : 그룹패키지 설치
- `yum groupremove 그룹` : 그룹리스트 삭제
- `yum groupinfo 그룹` : 그룹패키지의 정보
- `yum grouplist 그룹` : 그룹리스트의 정보 <br/>
