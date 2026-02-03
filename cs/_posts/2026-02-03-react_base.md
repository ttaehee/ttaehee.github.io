---
title: React 개발 중 여기는 어디인가 화면 플로우 이해하기
excerpt: Route/Service/Page/Component 계층과 상태 관리
---

<br/>

- 시간이 빠른듯 느린듯 빠른듯한 요즘, 이번해에는 흘러가는 대로 지내보지않고 체력과 시간관리를 해보아야할듯 하다 (과연)     
  점점 체력은 떨어지고 여전히 하고싶은건 많고 시간은 없어 🫠

<br/>

- 이번에도 TIL    
  모르는게 여전히 많음 🐣 이번엔 react                  
  admin tool 을 맡았는데 화면쪽을 마지막에 하길 잘했다 react 궁금했는데 막상 준비없이 맞닥뜨려버리니 당황스럽긴 함       
  화면 외의 프로젝트들 개발, 연동을 생각보다 빨리 끝내서 뿌듯했는데 화면쪽에서 아주우 헤맴     
  - 어지러워 여기는 어딜까 아까 온곳 같은데 다 거기서 거기같아여 ㅎ 다른 분들도 어렵다고 하셔서 좀 위안이 되었달까   
    일정이 있어서 일단 어찌저찌 돌아가게 만들어두고, 담엔 길 안잃을라고 프론트 아키텍처 파악해보기    

<br/><br/>

## 프론트 화면 구조 4개 layer
프론트는 Route → Route Service → Page → Component 구조이고, Props와 Redux는 레이어를 잇는 데이터 전달 수단     

```text
User Event (버튼 클릭 등) / Route             // 진입 트리거 = 어떤 페이지를 열지 결정 (URL)

→ Route Service (routes/*.service.ts)      // 어떤 API를 호출할지 & 호출 순서 제어 & 여러 API 응답 조합하고 화면 기준 데이터로 변환 & 조건부 분기 / redirect 판단 => 최종 결과물은 순수 데이터 (아직 Page Props 아님) 화면 진입 시 필요한 초기 데이터를 생성함 = 초기화까지만 담당
  (백엔드의 UseCase / Application Service 와 유사)    

→ Page (pages/*.tsx - register.tsx : write / display.tsx : read)   // Route 진입점 + Presentation Adapter = Route Service 호출해 반환 데이터를 Page Props로 받아서 Redux 초기 상태 세팅(Orchestration) => 이 시점부터 Props (페이지 진입 데이터 컨테이너) 라는 개념이 생김 
  (백엔드의 Controller 와 유사)

→ Component (client/components/*.tsx)      // 표현 전용(보여주고, 입력받고, 이벤트만 발생시키는 역할 = 결정은 안 하고, 결과만 표현) = JSX로 화면 그리기 & props 값 그대로 렌더 & 클릭 / 입력 이벤트 발생 & 부모로 콜백 호출
  (백엔드로의 View / Template 와 유사)

(+ Props는, State / Redux 연결 수단)
```

<br/><br/>

## Props / 작업상태 (State, Redux)

**Props**       
컴포넌트 간 데이터 전달 수단 = 백엔드의 Request DTO / Response DTO 와 유사
- 전달 경로가 명확함 : 부모 → 자식 (다시 위로 못 올라감)
  - 상위 → 하위로 내려주는 읽기 전용 입력 데이터 = 이 컴포넌트는 이 데이터로 그려진다
    - 화면 구성값 / 초기값 / 조회 결과 전달
- 변경 불가 (immutable)
- 생명주기 : 렌더링 단위 = 부모가 다시 렌더되면 새로 내려옴

<br/><br/>

**작업상태**     
유저의 행동으로 변경되고, 다음 액션 판단에 사용되며, 해당 작업 범위 안에서는 사라지면 안 되는 상태

- Page Props로 받은 데이터 중 유저가 수정·선택·결정의 대상으로 삼는 순간(= 다음 액션 판단의 기준이 되는 순간) 그때부터 그 데이터는 Props가 아니라 작업상태가 되며, 이는 사용 컴퍼넌트 범위에 따라 State/Redux의 영역이 됨       
  = Page Props는 컨테이너지 상태가 아님 진입 순간의 결정된 데이터, Redux는 그 데이터를 기반으로 시작된 작업의 현재 상태 그래서 **Props → Redux로 승격** 이라고 표현    

<br/>

- **State** : 컴포넌트 안에서만 쓰는 UI 임시 상태 = 백엔드의 Method Local Variable 와 유사
  - UI 반응용 (열림/닫힘, 입력 중 값, 포커스 등)
  - 생명주기 : 컴포넌트 생명주기

<br/>

- **Redux** : 여러 컴퍼넌트가 공유하는 화면의 작업 상태 저장소 = 백엔드의 UseCase 실행 중 유지되는 Context / Aggregate State 와 유사
  - 작업 상태 중에서도 여러 컴포넌트가 공유하거나 화면 이동 후에도 유지되어야 하는 상태를 담음
  - 생명주기 : Store 가 살아있는 동안
    - 보통 앱 생명주기 또는 명시적 reset까지 

<br/>

정리
- 읽기만 한다 → Props
- 유저가 바꾸기 시작한다 → 작업 상태
  - 한 컴포넌트만 쓰면 → State
  - 여러 컴포넌트 / 이동 후 유지 → Redux

<br/>

내가 이해한 바로는
백엔드는 요청 단위로 처리되므로 요청 완료 후 상태를 유지할 필요가 없지만,    
프론트는 페이지가 열려 있는 동안 입력 → 이동 → 다시 돌아오는 흐름이 계속되므로, 유저 작업 상태를 화면이 살아 있는 동안 어디선가 관리해야함 Redux 가 그 저장소 역할을 함    

<br/><br/>

## 화면 구조 layer 와 상태 Flow
그럼 데이터가 화면에 쓰일때 어떤 플로우로 데이터가 사용되는건지?   

```
Redux 에 저장된 작업 상태를 = 유저가 수정·선택·결정한 값이 여기에 있음 (저장)
 → Page 가 Redux에서 이 화면에 필요한 것만 골라서 (선택)
   → Page 가 Component 에 전달할 props 를 조립해서 내려주고 (조립)
     → Component 는 받은 props 그대로 그대로 렌더 (표현)
```

이 흐름을 지키면   

- Component는 Redux를 몰라도 됨
- Page는 화면 조립만 하며
- Redux는 작업 상태 저장소의 역할만 하면 됨

= 책임 분리가 됨    

<br/><br/>

### 내가 이번에 했던 플로우 예시로 정리

다음 작업할때 참고하려고 우리 프로젝트 패키지 기준으로 정리하기   
내가 해야했던건 화면에 select 박스 하나 추가해야했고, 그걸 구성하기 위한 데이터를 추가로 api 에서 응답하였다  

- 화면 진입 + 초기 데이터 준비

  ```
  그래서 amdmin 에서 관리하고 있는 api 응답 데이터 (graphql/*.graphql) 와 graphql 스키마(graphql.schema.ts) 와 프론트 데이터 모델에 필드 추가  
  ↓
  그 api 응답을 graphql service (graphql/*.service.ts - 서버 응답을 그대로 받음 화면 기준 아님) 쪽에서  
  서버 응답을 프론트용 데이터 모델(Page Props 후보인데 객체 이름이 ~Props여서 헷갈렸음)로 컨버팅하는 로직에도 해당 필드 매핑하는 로직 추가 
  ↓
  그 변환된 데이터 받아서 route service (routes/*.service.ts) 쪽에서 화면 진입 시 사용할 Redux 초기 상태 구성  
  ↓
  Page (pages/*.tsx)에서 이 Page Props 후보 데이터와 Redux 상태를 기반으로 Component에 전달할 Page Props 조립  
  ↓
  Component에서 해당 props 그대로 사용해 select 박스 구성하여 화면 작업 완료
  ```

<br/>

- 사용자가 선택 (렌더링 루프) = 트리거가 UI 이벤트 (입력, 클릭)

  ```
  Component (입력 이벤트)
   → onChange / onClick (props로 전달된 핸들러)
   → action dispatch
   → reducer 상태 갱신
   → Redux 상태 변경
   → Component 재렌더
  ```

<br/>

- 저장, 수정 버튼 클릭 (사용자 이벤트 + 서버 I/O 케이스)

  ```
  Component
   → action
   → GraphQL Mutation Service
   → 응답 처리
   → 성공 시 Redux 상태 갱신
   ```

<br/>

우리 프로젝트에서는 죄다 props 라 좀 헷갈렸다 내가 이해한바로는    

- Route Service에서 props를 만들고
- Page에서 이 props와 redux 로 다시 화면용 props 객체를 만들고
- Component에서 이 component props를 쓰는듯

컨테이너라기보다 단순 dto 처럼 (아직 작업 한번밖에 안해봐서 아닐수도 ㅎ 다음 작업할때 더 봐야지)   
이게 react 랑 graphql 같이 섞여있으니 더 헷갈렸던듯   

<br/>

초반 무지상태에서 무언갈 배울때는 백지이다 보니 빠르게 흡수하는 느낌이었다면, 요즘은 새로운거 마주하면 기존에 알던거랑 비교하면서 파악하는 느낌 (긍 정 ㅎ)      

<br/>
