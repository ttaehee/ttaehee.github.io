---
title: JAVA) ArrayList size는 어떻게 동적으로 늘어나는가
excerpt: arraylist add() 동작 방식
---

<br/>

- ArrayList와 LinkedList에 데이터 삽입 시, ArrayList의 속도가 훨씬 느리다 왜일까?    
  ArrayList에서 add()는 왜 속도가 느린지, size를 어떻게 동적으로 늘리는것인지 코드를 살펴보았다     
  
<br/>

- 결론을 적어보자면, LinkedList는 데이터 삽입 시, 삽입 앞 뒤 노드의 참조값만 변경하면 되어서 빠른데             
  ArrayList는 내부에서 복사가 일어나기 때문에 느릴 수 밖에 없는거였다      
  동적으로 늘어나는것 또한, 용량을 키운 배열에 복사를 하는 것이었다    

<br/>

## ArrayList
- ArrayList는 내부에서 element 배열을 기반으로 구성되어 있음    

  <img width="657" alt="스크린샷 2023-04-22 오후 6 33 30" src="https://user-images.githubusercontent.com/103614357/233776964-df79548c-8f44-47e3-b910-f26079ccbb20.png">

- 기본 capacity는 10
  
  <img width="403" alt="스크린샷 2023-04-22 오후 6 54 05" src="https://user-images.githubusercontent.com/103614357/233777012-2bcda210-8f9a-40ba-8c26-5610fab8ad99.png">

- 생성자를 통해 직접 element 배열의 capacity 설정 가능     
  기본 생성자를 사용할 경우 elementData에 EMPTY_ELEMENTDATA (빈 Object 배열)을 할당   

  <img width="606" alt="스크린샷 2023-04-22 오후 6 55 41" src="https://user-images.githubusercontent.com/103614357/233777085-e284ff9e-1f22-4b2d-81a8-147e439a6c58.png">

 <br/><br/> 

### add() 내부 구조

<img width="361" alt="스크린샷 2023-04-22 오후 6 36 46" src="https://user-images.githubusercontent.com/103614357/233776133-02f421f1-8996-4d5f-9c48-95d820007403.png">

- `modCount++`로 변경된 횟수를 카운트함
- `add(e, elementData, size);`   

  <img width="533" alt="스크린샷 2023-04-22 오후 6 39 24" src="https://user-images.githubusercontent.com/103614357/233776203-3c0bda04-7ba9-4c0a-8f5e-367825d7d9d0.png">

  - elementData가 overflow 될 상황인지 체크 후    
  - `elementData = grow();` : 배열의 크기를 동적으로 늘어나게 해주는 부분
    
    <img width="665" alt="스크린샷 2023-04-22 오후 6 43 28" src="https://user-images.githubusercontent.com/103614357/233776431-b1486e07-35c5-407a-a129-d850d2eb364f.png">
  
    - 현재 배열의 길이에 해당 하는 oldCapacity로 두고    
      oldCapacity + (oldCapacity >> 1) 를 새로운 capacity로 설정
      - `oldCapacity >> 1`은 1 right shift이므로 2로 나눈 것과 같음

    => 이후, 기존의 elementdata 값들을 포함하고 새로운 capacity 크기를 갖는 배열을 copyof() 메소드를 통해 복사하여 사용

<br/>

#### ex) resize 되는 과정   
- ArrayList 기본 생성자로 생성 -> element 배열 크기 0     
  -> element add 시, 배열의 resize 발생 -> 배열 크기 10으로 설정     
  -> 10개의 데이터가 채워진 후 데이터 추가 시 -> resize 발생 -> 배열 크기 15     

=> element 배열의 capacity가 조정 된 후 배열에 element가 추가되고, ArrayList의 size 1 증가     
=> 이렇게 임의의 capacity 를 설정하지 않는 일반적인 상황에서는 10 -> 15 -> 22 -> 33 -> 49 ...로 배열 사이즈가 조정됨      

<br/>

### 정리

- 정리를 해보면 element를 add하려고 할 때, capacity가 elementData의 길이와 같아지면     
  `기존용량 + 기존용량/2` 만큼 크기가 늘어난 배열에     
  기존 elementData를 copy함    

=> grow() 메서드로 용량을 늘린 뒤, 추가하려는 element를 할당하면      
우리 입장에서는 자동으로 사이즈를 늘려주면서 element가 추가된 것으로 보이지만,     
실제로는 가지고 있던 용량이 꽉 찼을 때, 용량이 기존의 1.5배를 늘린 새로운 배열에 기존 배열을 copy하는 것   

<br/><br/>

Reference      
[Class ArrayList<E>](https://docs.oracle.com/javase/8/docs/api/java/util/ArrayList.html)     
[[Collection]ArrayList](https://m.blog.naver.com/PostView.nhn?isHttpsRedirect=true&blogId=pistolcaffe&logNo=221019498243&proxyReferer=https:%2F%2Fwww.google.com%2F)     
[array(배열)과 arrayList(리스트)의 차이](https://zorba91.tistory.com/287)
  
<br/>
