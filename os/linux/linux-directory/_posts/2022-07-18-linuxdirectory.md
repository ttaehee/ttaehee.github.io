---
title: OS) Linux Directory Structure
excerpt: linux 디렉토리 구조
---

# Linux Directory Structure
Linux 는 `Unix 계열`의 OS -> Unix 의 많은 부분을 그대로 사용함
- **System**과 관련된 정보 /  **Hardware** 같은 장치 모두 파일로 관리
- 파일을 효율적으로 관리하기위해 **directory**를 사용
  - 디렉토리는 계층구조 (`Tree 구조`)
  - 모든 디렉토리의 **최상위** 디렉토리 = root directory = `/ `  
  - 명령어의 성격, 내용, 사용권한 등에 따라 디렉토리로 구분
- 대부분의 리눅스는 `FHF` (Filesystem Hierarchy Standard) 표준파일시스템 계층 사용
  - 같은 목적의 파일들은 같은 장소에 모아 관리 -> 시스템 자원이나 프로그램 쉽게 찾음
  - 즉, 명령어, 파일, 문서들 독립된 장소에서 관리됨 <br/><br/>

## Linux File
###  일반파일
텍스트파일, 실행파일, 이미지파일

### 디렉토리
해당 디렉토리에 저장된 파일 or 하위 디렉토리에 대한 정보 저장  
(리눅스에서는 디렉토리도 파일로 취급)

### 심벌릭 링크
원본파일을 다른 파일명으로 지정한 것  
(윈도우의 바로가기 개념과 비슷)

### 장치파일
하드디스크, 마우스 등의 장치를 관리하기 위한 파일  
장치관리 위해 시스템관리자는 해당 장치 파일에 접근해야함  
(장치파일 : /dev 디렉토리 아래 위치) <br/><br/>

## Directory Structure
![linux_directory](https://user-images.githubusercontent.com/103614357/179536403-ba64d7e3-f4f0-4e4f-b5ad-ad7e6df3362e.png)

### `/` : The Rood Directroy
- **최상위** 디렉토리
- 파티션 설정 시 반드시 존재해야함
- **절대경로**의 기준

### `/bin` : Essential User Binaries
- linux의 **기본명령어(binary)** 존재
  - 시스템 운영하는데 필요한 기본적인 명령어
  - 부팅에 필요한 명령어
  - 부팅 후 시스템사용자들이 사용할 수 있는 일반적 명령어도 위치

### `/lib` : Essential Shared Libraries
- 프로그램들이 의존하고 있는 **라이브러리 파일**들 존재
  - `/usr/bin` (응용프로그램의 실행파일)의 binary 를 실행하기 위한 library 는 `/usr/lib`에 위치
- 대부분의 libraries는 링크로 연결되어 있음
- `/lib/modules` : 커널 모듈 파일 존재

### `/usr` : User Binaries & Read-Only Data
- 기본 실행파일
- 라이브러리 파일
- 헤더 파일 등의 파일 저장
- `/usr/local` : 새로운 프로그램들이 설치되는 곳 (windows 의 Program Files 와 유사)

### `/home`
- 일반사용자의 home directoroy가 만들어지는 곳
  - 사용자계정 만들면 계정과 같은이름으로 새로운 사용자 디렉토리가 /home 하위디렉토리로 생성됨

### `/boot`
- 부팅 시 매우 중요한 directory
- 부팅에 필요한 정보 가진 파일 존재
  - system 부팅 시 필요한 커널이미지 (`/etc/lilo.conf`에서 지정한 커널부팅 이미지파일)
  - 부팅정보 파일

### `/root` : Root Home Directory
- root 사용자의 홈 디렉토리  
- `/` directoroy 와 `/root` 모두 root 라고 부르지만 서로 다름

### `/mnt` : Temporary Mount Points
- 파일시스템을 임시로 연결하는 directory
- 다른 장치들을 mount 할 때 일반적으로 사용 (다른 디렉토리 사용가능)

### `/media` : Removable Media
- USB 같은 외부장치 연결하는 directory

### `/dev` : Device Files
- 장치파일들(가상의) 저장된 directory
- 실제 파일이 아닌 **가상의** 파일 시스템! = 물리적인 용량 차지하지 않음

### `/proc`
- **커널관련** 정보 저장되는 directoroy
  - system의 각종 processor
  - 프로그램정보
  - 하드웨어정보들이 저장됨
- 현재 **시스템의 설정** 보여줌
- cat 명령어 이용해서 보면 system 정보 확인가능  
  ex) Interrupt 정보확인 : `cat/proc/interrupts`
- `/dev` directory처럼 **가상의** 파일 시스템 (물리적 용량을 갖지않음)
  - 하드디스크에 저장되지않음
  - 커널에 의해 memory에 저장됨

### `/etc` : Configuration Files
- 시스템 환경설정파일이 있는 directory
  - **네트워크** 관련 설정파일
  - **사용자**정보 / **암호**정보
  - **보안**파일
  - 파일시스템 정보
  - 시스템**초기화** 파일 등

### `/var` : Variable Data Files
- system 에서 사용되는 동적 파일들 저장 = 가변자료 저장 directory
  - system 운영 중 발생한 데이터
  - 로그(작동기록)

### `/tmp` : Temporary Files
- system 사용 중 발생한 임시데이터 저장
- 부팅 시 초기화

### `/opt` : Optional Packages
- 추가패키지가 설치되는 directory <br/><br/>

Reference  
https://coding-factory.tistory.com/499  
https://realforce111.tistory.com/entry/%EB%A6%AC%EB%88%85%EC%8A%A4-%EB%94%94%EB%A0%89%ED%86%A0%EB%A6%AC-%EA%B5%AC%EC%A1%B0-%EB%B0%8F-%EA%B8%B0%EB%8A%A5  
<br/>
