---
title: Variables & Memory Structure
excerpt: learn Java
taxonomy: variables
---

# Variables
: 값을 저장할 수 있는 메모리공간

## Local variable
: method 안에서 선언된 변수 - method 내에서만 사용가능  
(method 실행될 때 memory 할당 받으며, 끝나면 소멸되어 사용할 수 없음) <br/><br/>
**default 초기화가 안됨 -> String name = null; 해줘야함** <br/><br/>

## Member variable
: class 안에서 선언된 변수 - instance가 생성될 때 생성됨  
초기값 설정하지 않으면 자동 default 초기화됨 -> String name; 가능 (String name = null; 과 같음) <br/><br/>

## Static variable
instance variable에 static만 붙이면 됨  
=> 클래스가 로딩될 때 생성(=메모리에 딱 한번 올라감) -> static memory에 올려서 모두가 공유(언제나 모두 같은상태)  
- new 등을 써서 heap 영역에 할당하지 않고 공유로 사용 가능  
- static 끼리는 인스턴스 변수의 접근법과 다르게 인스턴스를 생성하지 않고 클래스이름을  접근가능  
- this. 불가 <br/><br/>

# Memory Structure
: 프로그램이 OS로부터 할당받는 대표적인 메모리공간  

## Code (코드 영역)
: 실행할 프로그램의 코드가 저장되는 영역 (=텍스트영역)
- 프로그램이 시작되고 종료될 때까지 메모리에 남아있음 <br/><br/>

## Data (데이터 영역)
: 전역변수와 static 변수가 저장되는 영역
- 프로그램의 시작과 함께 할당되며 프로그램이 종료되면 소멸 <br/>
- `static` : 메모리에 한번 할당되어 프로그램이 종료될 때 해제되는 것 (main method는 무조건 static)     
&nbsp; static final => 상수 (final : 고정 = 수정불가,확장불가)   

static 키워드를 통해 static 영역에 할당된 메모리는 Garbage Collector 관리영역 밖에 존재  
->프로그램 종료시까지 메모리가 할당된 채로 존재 -> 자주 사용하게되면 시스템의 퍼포먼스에 악영향 <br/><br/>

## Heap (힙 영역)
: 프로그래머가 직접 공간을 할당, 해제하는 메모리공간

- 참조형의 데이터객체의 실제 데이터들이 담기는 공간 = **new 키워드**를 통해 생성된 객체가 힙에 존재  
&nbsp; : 실제 데이터를 가지고 있는 heap 영역의 참조 값을 stack영역의 객체가 가지고 있음
- 선입선출(FIFO, First-In-First-Out) : 가장 먼저 들어온 데이터가 가장 먼저 인출됨  
&nbsp; (힙 영역이 메모리의 낮은 주소에서 높은 주소의 방향으로 할당되기 때문)   
- heap에 저장된 데이터가 더이상 불필요할 경우, 메모리관리를 위해 Garbage Collector를 통해 관리되어 자동으로 소멸됨<br/><br/>

## Stack (스택 영역)
: 프로그램이 자동으로 사용하는 임시 메모리공간

- 함수 호출 시 생성되는 지역변수, 매개변수가 저장되는 영역 / 참조형 변수는 참조값만 저장됨
- 함수 호출이 완료되면 사라짐
- 후입선출(LIFO, Last-In-First-Out) : 가장 나중에 들어온 데이터가 가장 먼저 인출됨  
&nbsp; (스택영역이 메모리의 높은 주소에서 낮은 주소의 방향으로 할당되기 때문) <br/>

==> new 를 통해 인스턴스 객체를 생성했을 때,  
&nbsp; heap 영역에는 생성된 객체가 올라가고, stack 영역에는 해당 객체에 대한 주소값이 저장 <br/><br/>

## Over Flow
: 한정된 메모리 공간이 부족하여 메모리 안에 있는 데이터가 넘쳐흐름   

힙은 메모리 위쪽주소부터 할당, 스택은 메모리 아래쪽 주소부터 할당 => 서로 상대 영역을 침범하는 경우   
- 힙 오버플로우 : 힙이 스택 침범  
- 스택 오버플로우 : 스택이 힙 침범 <br/>
