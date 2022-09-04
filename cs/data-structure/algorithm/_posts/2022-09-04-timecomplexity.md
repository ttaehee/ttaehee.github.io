---
title: CS) Time Complexity
excerpt: 시간복잡도
---

## 시간복잡도  
시간복잡도를 고민한다 = 효율적인 알고리즘 구현을 고민한다  
= `입력값이 커짐`에 따라 `증가하는 시간의 비율`을 `최소화`한 알고리즘을 구성한다   

<br/>  

### 시간복잡도 표기하는 방법    
1. 최상의 경우 : 오메가 표기법 (Big-Omega(Ω) Notation) => 하한 점근  
2. 평균의 경우 : 세타 표기법 (Theta(θ) Notation) => 둘의 평균  
3. 최악의 경우 : 빅오 표기법 (Big-O Notation) => 상한 점근   

<br/>

## Big-O 표기법
- 주로 사용하는 시간복잡도 표기하는 방법 (최악의 경우 고려)   
  - '이 정도 시간까지 걸릴 수 있다' : 최악의 경우를 고려하므로 프로그램이 실행되는 과정에서 소요되는 최악의 시간까지 고려할 수 있음
  - 최악의 경우를 사용하면 '아무리 나빠도 다른 알고리즘 보다는 같거나 좋다' 라는 비교분석에 따라 평균에 가까운 성능을 예측하기 쉽기 때문  
- 알고리즘 `실행 시간의 상한선`을 나타낸 표기법  
- `O(f(n))` 과 같이 표기(O는 order 라고 읽는다.)  

<br/>

[Excellent] &nbsp; `O(1)` < `O(logn)` < `O(n)` < `O(n log n)` < `O(n2)` < `O(2n)` &nbsp; [Horrible]    

<br/>

![ff](https://user-images.githubusercontent.com/103614357/188302933-be5fa720-c95b-4739-8360-2c9576b2e781.png)  

<br/><br/>

**`n` : 입력데이터 크기**   

<br/>

### O(1)

![ff](https://user-images.githubusercontent.com/103614357/188299507-6c690c9c-1ac9-48f1-96f4-f9a71a712598.png)   

- 일정한 복잡도 `constant complexity`  
  - n이 증가하더라도 처리시간이 늘어나지 않음
  - n 상관없이 즉시 출력값 얻어낼 수 있음 = 문제 해결을 위해 오직 `한 단계`만 거침
    - ex) 배열의 크기가 아무리 커도 해당 index로 접근해 바로 값 반환  

<br/>

### O(n)

![ff](https://user-images.githubusercontent.com/103614357/188299578-1d1c05f7-4ab7-479d-9908-d8ee05c3514b.png)  

- 선형 복잡도 `linear complexity`
  - n이 증가함에 따라 처리시간도 `같은 비율로 증가`
    - ex) 1차원 for loop  
      입력데이터 크기 1일 때 1초의 시간이 걸리고, 입력데이터 크기 100배로 증가시켰을 때 100초가 걸리는 알고리즘

<br/>

### O(log₂ n)  

![ff](https://user-images.githubusercontent.com/103614357/188299691-0bb500f2-7569-47bb-985a-ddabb4bb46eb.png)  

- 로그 복잡도 `logarithmic complexity`  
  - n이 커질수록 처리시간이 `로그(log) 만큼 짧아지는` 알고리즘
    - 문제를 해결하는데 필요한 단계들이 연산마다 특정요인에 의해 줄어듬
    - ex) 이진탐색  
      입력데이터 크기가 10배로 증가시켰을 때 처리시간은 2배

<br/>

### O(n²)  

![ff](https://user-images.githubusercontent.com/103614357/188300619-897c744b-4377-476b-a694-ed00cf6e1556.png)   

- 2차 복잡도 `quadratic complexity`
  - n이 증가함에 따라 처리시간이` n의 제곱수의 비율`로 증가
    - ex) 2중 for loop(버블정렬, 삽입정렬 알고리즘)  
      입력값이 1일 경우 1초가 걸리던 알고리즘에 5라는 값을 주었더니 25초

<br/>

### O(2ⁿ)  

![ff](https://user-images.githubusercontent.com/103614357/188300810-817c7fb0-e7ec-45c7-be8e-6e9a5a653872.png)  

- 기하급수적 복잡도 `exponential complexity`  
  - n에 따라 걸리는 시간은 `2의 n 제곱만큼` 비례  
  - 보통 문제를 풀기위한 모든 조합과 방법을 시도할 때 사용됨  
    - ex) 피보나치 수열, 재귀가 역기능하는 경우    

<br/><br/>


## 시간복잡도 구하는 요령 (간단하게)      

- 하나의 루프를 사용 + 단일 요소 집합 반복 : O (n)

  ```
  for(int i=0; i<n; i++){
      System.out.println("hello");
  }
  ```

  반복문이 n번만큼 반복 => O (n)  

<br/>

- 컬렉션의 절반 이상 반복 : O (n / 2) -> O (n)

<br/>

- 두 개의 다른 루프 사용 + 두 개의 개별 콜렉션 반복 : O (n + m) -> O (n)

  ```
  for(int i=0; i<n; i++){
      System.out.println("hello");
  }

  for(int i=0; i<n; i++){
      System.out.println("hello");
  }
  ```

  n번만큼 반복하는 반복문 2개  
  가장 큰 영향을 미치는 알고리즘 하나만 시간복잡도 계산 => O(n)

<br/>

- 두 개의 중첩 루프 사용 + 단일 컬렉션 반복 : O(n²)

  ```
  for(int i=0; i<n; i++){
      for(int j=0; j<n; j++){
          System.out.println("hello");
      }
  }
  ```

  n번만큼 반복하는 이중 for문 => O(n²)

<br/>

- 두 개의 중첩 루프 사용 + 두 개의 다른 콜렉션 반복 : O (n * m) -> O (n²)  

<br/>

- 컬렉션 정렬 : O(n*log(n)) 

<br/><br/>


## 시간복잡도 줄이는 법 & 참고자료   

알고리즘에서 시간복잡도에 가장 큰 영향을 끼치는 것은 반복문  
=> 적절한 알고리즘 설계  
알고리즘마다 핸들링 가능한 적절한 문제해결 케이스 외워두기 or 참고자료 참고  
알고리즘 형태에 맞는 효율적인 자료구조 이용하기   

<br/>  

![ff](https://user-images.githubusercontent.com/103614357/188301965-f40c197d-c0af-4108-9c37-8e37624080f2.png)  

<br/>

![ff](https://user-images.githubusercontent.com/103614357/188301984-8e74dea1-4e01-4311-b155-5a638156362a.png)  

<br/>

![ff](https://user-images.githubusercontent.com/103614357/188302013-a1cfafbb-04a4-4ca8-b8fc-3e515e1cedaf.png)  

<br/>

![ff](https://user-images.githubusercontent.com/103614357/188302098-6e7870a2-2aae-40c7-984b-5bcfb96ada0a.png)  

<br/>
 
프로그래머스 lv2 풀기 시작했는데 힘겹게 풀어놔도 자꾸 효율성에서 실패가 떠서 찾아본 시간복잡도ㅠ 이중루프 아웃ㅎ     

<br/><br/>

Reference  
https://hanamon.kr/%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-time-complexity-%EC%8B%9C%EA%B0%84-%EB%B3%B5%EC%9E%A1%EB%8F%84/  
https://coding-factory.tistory.com/608   
https://blog.chulgil.me/algorithm/    
https://joontae-kim.github.io/2021/04/15/algorithm-big-O/  
https://callmedevmomo.medium.com/%EC%9B%B9-%EA%B0%9C%EB%B0%9C%EC%9E%90%EB%A5%BC-%EC%9C%84%ED%95%9C-%EC%9E%90%EB%A3%8C%EA%B5%AC%EC%A1%B0%EC%99%80-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-01-%EB%B9%85%EC%98%A4-%ED%91%9C%EA%B8%B0%EB%B2%95-ff369f0efc1d  
<br/>
