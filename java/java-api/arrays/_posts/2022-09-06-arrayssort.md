---
title: JAVA) Arrays.sort() & Comparable, Comparator
excerpt: 배열 정렬하기
---

## Arrays.sort()  
기본 정렬조건 : `오름차순`  
=> Class 내에 기본적으로 구현되어있는 `Comparable` Interface의 `compareTo()` 를 기준으로 하기 때문  
=> 내부적으로 객체를 Comparable로 형변환 -> compareTo() 호출해서 비교, 정렬

<br/>

참고)  
Java에서 `인스턴스를 서로 비교하는 클래스`들은 모두 Comparable 인터페이스가 구현 되어 있음  

<br/>

### reverseOrder()  
Comparator(익명인터페이스), Collections 클래스의 내림차순 메서드   
=> 이미 내림차순으로 구현되어 있는 Comparator 반환  

```
Arrays.sort(arr, Comparator.reverseOrder());
Arrays.sort(arr, Collections.reverseOrder()); 
```

<br/>

### 정렬조건 바꾸기   
Arrays 클래스뿐 아니라 Collection을 다루는 정렬 메서드 대부분 적용 가능  
- 정렬할 Class내에 구현되어 있는 `compareTo` 메서드(Comparable 인터페이스의) override  

```
class Grade implements Comparable<Grade>{
    int kor;
    int eng;
    
    @Override
    public int compareTo(Grade other) {
        //return this.kor - other.kor;   //오름차순
        return other.kor - this.kor;  //내림차순
    }
}
```

return값이 양수인 경우, 두 값의 자리가 변경되면서 정렬이 이루어짐    

```
Grade[] arr = new Grade[5];
Arrays.sort(arr);
```

<br/>

- `Comparator를 구현한 Class`(Custom)에서 `compare()` override  
    \+ sort() 호출시 구현한 클래스 명시(new Custom())  

```
class Custom implements Comparator<String> {

    @Override
    public int compare(String o1, String o2) {
       return o2.compareTo(o1);  
    }
    
} 

Arrays.sort(arr, new Custom());   
```

<br/>  

- Comparator 인스턴스 생성과 동시에 compare() override

```
Arrays.sort(arr, new Comparator<String>() {

       @Override
       public int compare(String o1, String o2) {
              return o2.compareTo(o1);  
       }
       
});  
```

<br/>

참고)  

![제목 없음](https://user-images.githubusercontent.com/103614357/188464375-03cd57a6-45e5-4d89-9ede-8368cf82ac0f.png)   

Arrays.sort()는 단순히 배열만 파라미터로 받는 것이 아니라 Comparator 또한 파라미터로 받기도 함      
Comparator 파라미터로 넘어온 c의 비교 기준으로 파라미터로 넘어온 객체배열 a 정렬  

<br/> 

- 람다식  

```
Comparator<String> comp = (o1, o2) -> o2.compareTo(o1); 
Arrays.sort(arr, comp);
```

```
Arrays.sort(arr, (o1, o2) -> o2.compareTo(o1));  
```

<br/> 

### 2차원 배열 정렬 
2차원 평면 위의 점 n개  
x좌표 오름차순, x좌표 같으면 y좌표 오름차순 정렬 코드   

```
int[][] xy = new int[n][2];

Arrays.sort(xy, new Comparator<int[]>(){

  public int compare(int[] s1, int[] s2){
  
    if(s1[0] == s2[0]){
      return s1[1]-s2[1];
    }else{
      return s1[0]-s2[0];
    }
  }
  
}
```

<br/><br/>

## Comparable & Comparator  

타입이 객체거나 2차원 배열일 경우 sorting or 정렬조건 변경 시 사용   

- Comparable `compareTo()` : `파라미터 1개` 받아서 this 키워드 사용해서 `자기 자신`과 `매개변수 객체`를 비교  
- Comparator `compare()` : `파라미터 2개` 받아서 `두 매개변수 객체`를 서로 비교

<br/>

### java.lang.Comparable(Interface)

```
interface Comparable<T> {
    int compareTo(T obj)
}
```

return값이 양수인 경우, 두 값의 자리가 변경되면서 정렬  
- 오름차순 : `return 자기자신-비교값;` or `if( 자기자신 > 비교값 ) { return 1; }`    
- 내림차순 : `return 비교값-자기자신;` or `if( 자기자신 < 비교값 ) { return 1; }`    

<br/>

### java.util.Comparator(FunctionalInterface)

Comparator 인터페이스는 `추가정렬을 정의`(비교 기준이 여러개 일때) 시 사용하기 때문에
정렬할 클래스에 구현하는 것이 아니라 따로 클래스를 구현해서 만들어 주는 것이 좋음  

<br/>

프로그래머스 가장큰수 풀다가 여기까지 왔다ㅎ  
아무리 풀어도 안되어서 검색했더니 다들 처음보는 Comparator 사용  
새로운거 알았다 ㅎ   


<br/><br/>

Reference   
https://ifuwanna.tistory.com/232  
https://lotuus.tistory.com/35  
https://sinsucoding.tistory.com/14   
https://st-lab.tistory.com/243  
https://codevang.tistory.com/294  
<br/>
