---
title: CS) Priority Queue
excerpt: 우선순위 큐
---
 
# Priority Queue(우선순위 큐)
- 모든 항목에 우선순위가 있음
  - 부모 노드는 항상 자식 노드보다 우선순위가 높음
- 우선순위가 높은 요소 : 우선 순위가 낮은 요소보다 먼저 queue에서 제외
- 우선 순위가 같으면 : queue의 순서에 따라 제공

<br/>

- 시간복잡도 - 내부구조가 힙이기때문에 O(NlogN)

<br/>

## Priority Queue 구현  
우선순위 큐는 힙을 이용하여 구현하는 것이 일반적
- 데이터를 삽입할 때 우선순위를 기준으로 최대힙 혹은 최소 힙을 구성하고 데이터를 꺼낼 때 루트 노드를 얻음
- 루트 노드를 삭제할 때는 빈 루트 노드 위치에 맨 마지막 노드를 삽입한 후 아래로 내려가면서 적절한 자리를 찾아서 옮기는 방식  
=> 값을 추가하거나 빼면 항상 우선순위에 맞게 정렬된 상태

<br/>

## 자바에서 Priority Queue 사용
### 선언

- 낮은 숫자 순으로 우선순위 정렬

```
import java.util.PriorityQueue;

PriorityQueue<Integer> priorityQueue1 = new PriorityQueue<>();
```

- 높은 숫자 순으로 우선순위 정렬

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
pq.add(8);
```

결과 => \[1, 15,8,21,25,18,10\]

- add() : Collection 클래스에서 가져오는 메서드
- offer() : Queue 클래스에서 가져오는 메서드

<br/>

### 값 삭제

```
pq.poll();
pq.remove();
pq.remove(1);
pq.clear();
```

- poll() : 첫번째 값 반환 후 제거(비어있으면 null 반환)
- remove() : 첫번째 값 제거
- clear() : 초기화 

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
<br/>
