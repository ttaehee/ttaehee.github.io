---
title: JDBC) Change Oracle Port
excerpt: 오라클 8080 포트 변경
---

# Change Oracle 8080 Port
오라클 설치 시 오라클 서비스가 8080 포트 점유  
톰캣도 기본적으로 8080 포트 사용  
=> 충돌  

### 1. SYSTEM 계정으로 로그인
### 2. 사용중인 XDB 포트 확인
```
SELECT DBMS_XDB.GETHTTPPORT() FROM DUAL;
```
### 3. 포트 변경
```
EXEC DBMS_XDB.SETHTTPPORT(9090);
```
### 4. 다시 확인
```
SELECT DBMS_XDB.GETHTTPPORT() FROM DUAL;
```
<
