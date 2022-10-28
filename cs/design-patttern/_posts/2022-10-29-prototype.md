---
title: CS) Prototype Pattern
excerpt: 디자인패턴 - 생성 패턴 - 프로토타입
--- 

## Prototype Pattern  

Original 객체를 새로운 객체에 복사하여 필요에 따라 수정하는 메커니즘을 제공  

![제목 없음](https://user-images.githubusercontent.com/103614357/198676942-dbae3843-3fef-470c-b0bd-4a80e389af4b.png)  

- `Prototype` : 인스턴스를 복사하여 새로운 인스턴스를 만들기 위한 메소드 결정
- `ConcretePrototype` : 인스턴스 복사해서 새로운 인스턴스 만드는 메소드 구현
- `Client` : 인스턴스 복사 메소드를 사용해서 새로운 인스턴스 생성

<br/> 

- 복사를 위하여 Java에서 제공하는 **clone()** 사용
    - 생성하고자 하는 객체의 클래스에서 clone() 재정의

- 객체를 생성하는 데 비용(시간과 자원)이 많이 들고, 비슷한 객체가 이미 있는 경우에 사용되는 생성 패턴 중 하나
    - ex)
        - 데이터 베이스에서 데이터를 읽어와서 인스턴스를 생성
        - http 요청을 보내서 가져온 데이터로 인스턴스를 만들어야 하는 경우
    - 객체를 복사하는 것이 네트워크 접근이나 DB 접근보다 훨씬 비용이 적음

<br/>
  
### Example

- DB로부터 가져오는 객체    
→ 프로그램에서 수차례 수정을 해야하는 요구사항이 있는 경우    
  - 매번 new 라는 키워드를 통해 객체를 생성하여 DB로부터 항상 모든 데이터를 가져오면?     
    ⇒ DB로 접근해서 데이터를 가져오는 행위는 비용이 큼   

<br/>

⇒ 한 번 DB에 접근하여 데이터를 가져온 객체를 필요에 따라 새로운 객체에 복사하여 데이터 수정 작업을 하는 것이 더 좋은 방법     

<br/>

- 이때 객체의 복사를 얕은 복사(shallow copy)로 할지, 깊은 복사(deep copy)로 할지는 선택적으로  
  - 기본적으로 Object class의 clone()은 얕은복사   
    → 깊은복사 위해서는 재정의하기

<br/>
  
- `Employees class`(직원정보를 DB에서 가져온다고 가정)

```java
public class Employees implements Cloneable{
 
    private List<String> empList;
	
    public Employees(){
        empList = new ArrayList<String>();
    }
	
    public Employees(List<String> list){
        this.empList=list;
    }
    
    public void loadData(){
        empList.add("Pankaj");
        empList.add("Raj");
        empList.add("David");
        empList.add("Lisa");
    }
	
    public List<String> getEmpList() {
        return empList;
    }
 
    @Override
    public Object clone() throws CloneNotSupportedException{
        List<String> temp = new ArrayList<String>();
        for(String s : this.empList){
            temp.add(s);
        }
        return new Employees(temp);
    }
	
}
```

```java
public class PrototypePatternTest {
 
    public static void main(String[] args) throws CloneNotSupportedException {
        Employees emps = new Employees();
        emps.loadData();
		
        Employees empsNew = (Employees) emps.clone();
        Employees empsNew1 = (Employees) emps.clone();
        List<String> list = empsNew.getEmpList();
        list.add("John");
        List<String> list1 = empsNew1.getEmpList();
        list1.remove("Pankaj");
		
        System.out.println("emps List: "+emps.getEmpList());
        System.out.println("empsNew List: "+list);
        System.out.println("empsNew1 List: "+list1);
    }
 
}
```

```
emps List: [Pankaj, Raj, David, Lisa]
empsNew List: [Pankaj, Raj, David, Lisa, John]
empsNew1 List: [Raj, David, Lisa]
```

- `emps.clone()` 을 통해 `empList`에 대하여 copy(deep) 실시
  ⇒ 1회의 DB 접근을 통해 가져온 데이터를 복사하여 사용
  → 객체 생성의 비용을 줄일 수 있음

- 만약 DB로부터 매번 employee 리스트를 직접 가져왔다면?  
  → 그로 인해 DB접근으로 인한 큰 비용(시간, 리소스) 발생

<br/>

참고)   
`Model Mapper` : 객체의 값들을 다른 객체로 복사해주는 편리한 라이브러리

<br/>

참고)   
- shallow copy : 하나의 객체의 주소값을 복사  

```java
public static void main(String[] args) {
  Cat navi = new Cat("navi");

  Cat yo = navi;
  yo.chgName("yo");

  System.out.println(navi.getName());
  System.out.println(yo.getName());
}
```

```
yo
yo
```

<br/>

- deep copy : 하나의 객체의 값들을 복사

```java
public class Cat implements Cloneable{
    ...

     public Cat copy() throws CloneNotSupportedException {
    Cat ret = (Cat)this.clone();
    return ret;
  }
}

public static void main(String[] args) throws CloneNotSupportedException {
  Cat navi = new Cat("navi");

  Cat yo = navi.copy();
  yo.chgName("yo");

  System.out.println(navi.getName());
  System.out.println(yo.getName());
}
```

```
navi
yo
```

<br/><br/>

Reference    
https://readystory.tistory.com/122    
https://m.blog.naver.com/gleblu4422/222675395264    
https://kingchan223.tistory.com/295   
https://lee1535.tistory.com/76   
<br/>
