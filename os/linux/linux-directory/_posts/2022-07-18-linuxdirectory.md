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
  - 즉, 명령어, 파일, 문서들 독립된 장소에서 관리됨

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
(장치파일 : /dev 디렉토리 아래 위치)

## Directory Structure
![linux_directory](https://user-images.githubusercontent.com/103614357/179536403-ba64d7e3-f4f0-4e4f-b5ad-ad7e6df3362e.png)

### `/` : The Rood Directroy
- **최상위** 디렉토리
- 파티션 설정 시 반드시 존재해야함
- **절대경로**의 기준

### `/bin` : Essential User Binaries
- linux의 **기본명령어(binary)** 들어있는 디렉토리
- 시스템 운영하는데 필요한 기본적인 명령어
- 부팅에 필요한 명령어
  - 부팅 후 시스템사용자들이 사용할 수 있는 일반적 명령어도 위치

### `/lib' : Essectial Shared Libraries
- 프로그램들이 의존하고 있는 라이브러리 파일들 존재
  - `/usr/bin` (응용프로그램의 실행파일)의 binary 를 실행하기 위한 library 는 `/usr/lib`에 위치
- 대부분의 libraries은 링크로 연결되어 있음
- `/lib/modules` : 커널 모듈 파일 존재

### `/usr` : User Binaries & Read-Only Data
- 기본 실행파일
- 라이브러리 파일
- 헤더 파일 등의 파일 저장
- `/usr/local` : 새로운 프로그램들이 설치되는 곳 (windows 의 Program Files 와 유사)

### `/home`
- 일반사용자의 home directoroy가 만들어지는 곳

Reference  
https://coding-factory.tistory.com/499  
https://realforce111.tistory.com/entry/%EB%A6%AC%EB%88%85%EC%8A%A4-%EB%94%94%EB%A0%89%ED%86%A0%EB%A6%AC-%EA%B5%AC%EC%A1%B0-%EB%B0%8F-%EA%B8%B0%EB%8A%A5  
<br/>



