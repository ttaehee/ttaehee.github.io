---
title: CS) Priority Queue & Heap
excerpt: 우선순위 큐와 힙
---

## Heap(힙)  
최솟값 또는 최댓값을 빠르게 찾아내기 위해 `완전이진트리 형태`로 만들어진 자료구조  
- `루트 노드`에 `우선순위가 높은 데이터`를 위치시키는 자료구조
- 모든 노드에 저장된 값(우선순위)들은 자식 노드들의 것보다 우선순위가 크거나 같음  

= 모든 요소들을 고려하여 우선순위를 정할 필요 없이 `부모 노드는 자식노드보다 항상 우선순위가 앞선다는 조건만` 만족시키며 완전이진트리 형태로 채워나가는 것

- 엄청 빠름 : `O(logN)`, 즉 트리의 높이만큼만 비교
	- 단순한 순차 탐색: 최댓값, 최솟값을 찾을 경우 O(N)의 탐색 시간  

![44](https://user-images.githubusercontent.com/103614357/186442281-0cdd72da-1c5d-42cd-9af0-c5cbe9e54c02.png)  

### 최대 힙   
부모 노드의 값(key 값) ≥ 자식 노드의 값(key 값)  
루트 노드로 올라갈수록 저장된 값이 커지는 구조  
우선순위는 값이 큰 순서대로  

### 최소 힙  
부모 노드의 값(key 값) ≤ 자식 노드의 값(key 값)  
루트 노드로 올라갈수록 값이 작아지는 구조  
우선순위는 값이 작은 순서대로    

=> 최대 힙이던 최소 힙이던 루트 노드에는 우선순위가 높은 것이 자리잡음
 
## Priority Queue(우선순위 큐)
`우선 순위가 높은 요소`가 `우선 순위가 낮은 요소`보다 먼저 제공되는 자료구조   
- 모든 항목에 우선순위가 있음
- 우선순위가 높은 요소 : 우선 순위가 낮은 요소보다 먼저 queue에서 제외
- 우선 순위가 같으면 : queue의 순서에 따라 제공

<br/>

부모노드와 자식노드간의 관계만 신경(형제 간 우선순위는 고려안함)  
'반 정렬 상태' 혹은 '느슨한 정렬 상태' , '약한 힙(weak heap)'이라고도 불림

<br/>

- 시간복잡도 - 내부구조가 힙이기때문에 O(NlogN)

<br/>

### Priority Queue 구현 (힙 자료구조를 이용한 우선순위 큐 구현)
우선순위 큐는 `힙을 이용하여 구현`하는 것이 일반적 (힙과 유사한 구조를 갖고 있으나, 엄연히 따지자면 개념 자체는 다름)  
- 데이터를 삽입할 때 우선순위를 기준으로 `최대힙` 혹은 `최소 힙`을 구성하고 `데이터를 꺼낼 때 루트 노드를 얻음`  
- 루트 노드를 삭제할 때는 빈 루트 노드 위치에 맨 마지막 노드를 삽입한 후 아래로 내려가면서 적절한 자리를 찾아서 옮기는 방식  
=> 값을 추가하거나 빼면 `항상 우선순위에 맞게 정렬`된 상태   

<br/>

## 자바에서 Priority Queue 사용
### 선언

- 낮은 숫자 순으로 우선순위 정렬(최소힙)

```
import java.util.PriorityQueue;

PriorityQueue<Integer> priorityQueue1 = new PriorityQueue<>();
```

- 높은 숫자 순으로 우선순위 정렬(최대힙)

```
import java.util.PriorityQueue;
import  java.util.Collections;

PriorityQueue<Integer> priorityQueue2 = new PriorityQueue<>(Collections.reverseOrder());
```

<br/>

### 값 추가

```
pq.add(1);		
pq.add(15);		
pq.offer(10);
pq.add(21);		
pq.add(25);		
pq.offer(18);
```

결과 => \[1,15,21,25,18,10\]

- add() : Collection 클래스에서 가져오는 메서드
- offer() : Queue 클래스에서 가져오는 메서드  

```
pq.offer(8);
```

![11](https://user-images.githubusercontent.com/103614357/186437980-edc5d1cf-e29f-451e-93f2-9b848e155871.png)   

8 추가  

![22](https://user-images.githubusercontent.com/103614357/186438050-88ab178f-0ba4-41b3-9a63-9cd64860d0e2.png)  

부모 노드와 자식 노드만 우선순위 비교 후 자리 변경  

<br/>

### 값 삭제

```
pq.poll();
pq.remove();
pq.clear();
```

- poll() : 첫번째 값 반환 후 제거(비어있으면 null 반환)
- remove() : 첫번째 값 제거
- clear() : 초기화 

<br/> 

```
pq.remove(1);
```

- remove(idx) : idx 순위에 해당하는 값 제거
	- 첫 번째 우선순위를 삭제하면 하나씩 밀리는 것이 아닌 우선순위를 재정렬 

![33](https://user-images.githubusercontent.com/103614357/186439158-a707305e-c75a-4680-8bc2-d1c30a0f8388.png)  

루트노드(1) 과 마지막노드(10) 스왑  

![44](https://user-images.githubusercontent.com/103614357/186439177-19b683a5-f910-4246-8512-b3232a527fa9.png)  

루트노드(1) 제거  
부모노드(10) & 자식노드(15,8) 과 비교해서 더 작은값인 8이 부모노드로 올라옴  

<br/>

### 크기 구하기

```
pq.size();
```

<br/>

### 값 출력

```
pq.peek();
```

- peek() : 우선순위 가장 높은 값 출력

```
Iterator iterator = pq.iterator();
		while(iterator.hasNext()){
			System.out.print(iterator.next() + " ");
    }
```

- iterator() : 전체 Queue 값 출력

<br/><br/>

Reference  
https://www.geeksforgeeks.org/priority-queue-class-in-java/  
https://www.youtube.com/watch?v=AjFlp951nz0&ab_channel=%EB%8F%99%EB%B9%88%EB%82%98   
https://crazykim2.tistory.com/575   
https://st-lab.tistory.com/219  
<br/>
