---
title: JAVA) java.lang.StringBuffer
excerpt: stringbuffer class
---

# StringBuffer Class
mutable class  
기본적으로 16 버퍼 생성 <br/><br/>

- append() : 문자열로 변환 후, 마지막에 추가
- insert() : 문자열로 변환 후, 지정된 인덱스 위치에 추가

```
StringBuffer str = new StringBuffer("Java");
System.out.println(str.append("수업")); // Java수업
System.out.println(str.insert(4, "Script"));  // JavaScript수업
```

- capacity() : 현재 버퍼크기 반환

```
StringBuffer str01 = new StringBuffer();
StringBuffer str02 = new StringBuffer("Java");

System.out.println(str01.capacity());  // 버퍼크기 16(default)
System.out.println(str02.capacity());  // 버퍼크기 20(default + 문자길이 4)
```

- delete() : 부분 문자열 제거
- deleteCharAt() : 특정위치 문자 한개 제거

```
StringBuffer str = new StringBuffer("Java Oracle");

System.out.println(str.delete(4, 8));  // Javacle
System.out.println(str.deleteCharAt(1));  // Jvacle
```

- reverse() : 인덱스를 역순으로 재배열  <br/>
