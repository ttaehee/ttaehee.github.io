---
title: JAVA) Conditional Statements
excerpt: 조건문
---

# Control Statement
제어문 : 자바 인터프리터는 위->아래, 오른쪽->왼쪽으로 프로그램 읽는데, 그 흐름을 변경할 수 있음  
1. Conditional Statements 조건문
- if문
- switch문
2. Loops 반복문
- for문
- while문

## Conditional Statements
### if statement
: 조건에 따라 수행됨
```
if( condition ) {
    statement 1;
} else{
    statement 2;
}
```
이 때 조건은 boolean
else문은 필요 시 기술 (생략가능)

### if ~ else if statement
: 다중 선택 가능
```
if( condition ) {
    statement 1;
} else if( condition ){
    statement 2;
} else{
    statement 3;
}
```

### switch statement
: 다중 선택 가능
```
switch(수식) {
    case 수식에대한값1:
        statement 1;
       break;
    case 값2:
         statement 2;
        break;
   defulat:
        default statement;
}
```
- break를 만나거나 마지막 case문 또는 default문에 도달할 때까지 계속 실행  
- break문 생략 가능 -> 바로 아래에 있는 case 문 실행  
=> 다수의 case절을 하나의 시행문으로 지정가능 <br/>
