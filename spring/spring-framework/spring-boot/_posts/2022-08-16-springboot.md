---
title: Spring) SpringBoot
excerpt: 스프링과 스프링부트
---

## SpringBoot (Spring과 차이)
- Embed Tomcat 사용 : 스프링부트 내부에 `내장탐캣` 포함
- starter를 통한 `dependency 자동화`
- application.yml(application.properties) 사용
- jar file로 배포 가능

### application.yml
web.mxl  
+  
root-context.xml : 한번만 new 하면 되는 애들 ex) 데이터베이스  
+  
servlet-context.xml : 지속적으로 new 해서 써야하는 애들  

- 참고) 설정파일
  - xml : `<name>value</name>`
  - JSON : `{"name":"value"}`
  - yml : `name: value` <- 가벼움
 
<br/>
