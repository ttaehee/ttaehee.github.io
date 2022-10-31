---
title: CS) Composite Pattern
excerpt: 디자인패턴 - 구조 패턴 - 컴포지트
--- 

## Composite Pattern

단일 객체(Single Instance)든 객체들의 집합(Group of Instance)이든 같은 방법으로 취급하는 것   
⇒ 개별적인 객체들과 객체들의 집합간의 처리 방법의 차이가 없을 경우 사용      
여기서 Composite의 의미는 일부 또는 그룹을 표현하는 객체들을 트리 구조로 구성한다는 것   
⇒ 컴포지트의 의도는 트리 구조로 작성하여, 전체-부분(whole-part) 관계를 표현하는 것  

![111](https://user-images.githubusercontent.com/103614357/199099463-a05962c5-ccc6-4d09-84d8-df846a1fab23.png)   

- Component
    - Leaf와 Composite 가 구현해야하는 Interface
    - Leaf 와 Composite 는 모두 Component 라는 같은 타입으로 다뤄짐
    - 예제코드에서 Node
- Leaf
    - Component 인터페이스를 구현한 것
    - 단일 객체
    - Composite 의 부분(자식) 객체로 들어감( Component 의 형태로)
    - 예제코드에서 File
- Composite
    - Component 인터페이스를 구현한 것
    - 집합객체
    - Component 요소(Leaf or Composite)를 자식으로 가짐   
        → Component 요소를 관리하기 위한 메소드를 추가적으로 구현해야함          
    - 예제코드에서 Directory  

<br/><br/>

### Example - File System

트리 구조를 표현할 수 있는 파일 시스템

- File

```java
class File {
    private String name;
}
```

- Directory 

```java
class Directory {
    private String name;
    private List<File> children;
    
    public void add(File file){
        //파일추가
    }
}
```

<br/>
   
- 디렉토리 안에 디렉토리가 있는 것을 어떻게 표현?
    
![222](https://user-images.githubusercontent.com/103614357/199099503-e093ab0b-fb3d-4009-8508-33f48c66935d.png)  
    
```java
interface Node {
    public String getName();
}

class File implements Node {
    private String name;
   
    @Override
    public String getName(){ 
        return name; 
    }
}

class Directory implements Node {
    private String name;
    private List<Node> children;
  
    @Override
    public String getName(){ 
        return name; 
    }
    public void add(Node node) {
        children.add(node);
    }
}
```
  
- Node 클래스는 기본적인 파일 및 디렉토리의 근간이라고 가정  
  - 모든 파일과 디렉토리는 이름을 가지고 있을테니 이름을 반환할 getName() 가짐  
- File 클래스는 Node 인터페이스를 구현하면 끝   
  (자신은 자식 요소를 가질 필요가 없기 때문)  
- Directory 클래스는 Node 인터페이스를 구현하는 것 + 자식 요소를 담아둘 List 필요

<br/>
   
- 사용

```java
Directory dir = new Directory();
dir.add(new File()); // 디렉토리에 파일 하나 삽입
dir.add(new Directory()); // 디렉토리에 디렉토리 삽입
Directory secondDir = new Directory();
secondDir.add(dir); // 기존 루트 디렉토리를 새로 만든 디렉토리에 삽입
```

⇒ Component 인터페이스 정의 + Leaf와 Composite가 이를 구현    
→ Leaf와 Composite 클래스가 동등하게 다뤄지는 핵심적인 이유    

<br/>

### 2가지 형태의 방식

Composite class는 자식들을 관리하기 위해 추가적인 메소드 필요   
→ 이러한 메소드들이 어떻게 작성되느냐에 따라, Composite Pattern은 다른 목적을 추구할 수 있음  

![333](https://user-images.githubusercontent.com/103614357/199099544-61fac32f-fb7d-408a-8bf2-c3e692968b76.png)   
 
- 일관성을 추구하는 방식
    
    ```java
    Node file = new File();
    Node dir = new Directory();
    ```
     
    - 자식을 다루는 메소드들을 Composite가 아닌 Component에 정의   
    → Client는 Leaf와 Composite를 일관되게 취급가능  
    하지만 Client는 Leaf객체가 자식을 다루는 메소드를 호출할 수 있기 때문에, 타입의 안정성을 잃게 됨   

<br/>
   
- 타입의 안정성을 추구하는 방식
    
    ```java
    File file = new File();
    Directory dir = new Directory();
    ```
    
    - 자식을 다루는 메소드들은 오직 Composite만 정의되었음  
    → Client는 Leaf와 Composite를 다르게 취급   
    하지만 Client에서 Leaf객체가 자식을 다루는 메소드를 호출할 수 없기 때문에, 타입에 대한 안정성을 얻게 됨   

<br/>

어떤 방식이 더 좋냐를 따지기에는 너무 많은 것이 고려됨  
위키에서의 이론은 컴포지트 패턴은 타입의 안정성보다는 일관성을 더 강조한다고 함

<br/><br/>

Reference   
https://mygumi.tistory.com/343      
https://jdm.kr/blog/228  
