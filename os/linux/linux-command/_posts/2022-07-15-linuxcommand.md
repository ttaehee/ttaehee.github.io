---
title: OS) Linux Basic Command
excerpt: linux 기본명령어
---

# Linux Basic Command
command + space + terminal 입력

- `ls` : 현재 경로에 존재하는 파일/폴더 확인하기 (list of directory)
- `pwd` : 현재 위치한 경로 출력 (print working directory)
- `cd` : 디렉토리 이동 (change directory)  
	ex) cd Desktop  
	ex) cd . . (상위폴더)  
	ex) cd ~ (가장상위폴더)  

- `touch` : 파일 생성하기   
	ex) touch test.txt  
- `cat` : 파일 내용 확인하기 (concatenate)  
	ex) cat test.txt
- `rm` : 파일 삭제하기 (remove)  
	ex) rm text.txt  
- `head` : 파일 첫째줄  
	ex) head test.txt  
- `tail` : 파일 마지막줄
	ex) tail test.txt <br/><br/>

- `mkdir` : 폴더 생성하기 (make directory)  
	es) mkdir test
- `rmdir` : 폴더 삭제하기, 내부에 파일이 없을 때만 가능 (remove directory)  
	ex) rmdir test
- `rm-r` : 폴더 삭제하기, 내부에 파일 있어도 가능, 내부 파일까지 삭제 <br/><br/>

- `cp` : 파일/폴더 복사 (copy)  
	ex) cp test.txt  test2.txt 테스트텍스트를 테스트2텍스트로 복사
- `mv` : 파일/폴더 이동 (move), 이름변경  
	ex) mv test.txt test 테스트텍스트를 테스트폴더로 이동  
	ex) mv test.txt test2.txt 테스트텍스트를 테스트2텍스트로 이름변경 <br/><br/>

- `clear` : 터미널 정리  
- `history` : 이전에 사용한 명령어들 확인  
-> 번호와 명령어 나옴 -> ! + 번호 쓰면 해당 명령어 다시 사용가능  
- `man` : 원하는 명령어의 매뉴얼 확인 (manual)  
	ex) man ls

- `open -a` : 앱 열기  
	ex) open -a Test.app
- `kill` : 프로세서 강제종료  
	ex) kill ichat
- `exit`, `logout` : 터미널 안전하게 종료 <br/><br/>

- `ipconfig getifaddr en0` : 내 로컬 ip주소확인
- `du -sh*` : 파일용량 확인 <br/>
