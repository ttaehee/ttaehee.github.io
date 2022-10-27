---
title: CS) Abstract Factory Pattern
excerpt: 디자인패턴 - 생성 패턴 - 추상팩토리
---

## Abstract Factory Pattern

많은 수의 연관된 서브 클래스를 `특정 그룹으로 묶어(팩토리화)` 한번에 교체할 수 있도록 만든 디자인 패턴

<br/>  

### Example

- `PizzaStore`
- 그걸 상속받는 서브 클래스 : `NYPizzaStore`, `ChicagoPizzaStore`

![55](https://user-images.githubusercontent.com/103614357/198224238-3d573fd6-de92-4223-a941-11a0ee88c186.png)   

- 뉴욕가게와 시카고가게에서 Pizza(객체)를 만들 때 사용되는 재료들은 중복됨  
    - 두 가게 모두 도우, 소스, 토핑, 치즈가 필요   
    이러한 부분을 묶어서 팩토리화하는 것이 추상팩토리패턴  
    
<br/>  

⇒ 팩토리를 이용하여 피자에서 쓰이는 재료를 만드는 것  
만들어지는 구체적인재료들은 어떤 팩토리를 쓰는지에 따라 달라짐  
(ex) 도우 → 두꺼운 도우일지, 씬일지는 팩토리에 의해 결정됨)   

<br/>  

- `PizzaIngredientFactory`

```java
public interface PizzaIngredientFactory {
		public Dough createDough();
    public Sauce createSauce();
    public Cheese createCheese();
    public Pepperoni createPepperoni();
}
```

```java
public class NYPizzaIngredientFactory implements PizzaIngredientFactory {
		public Dough createDough() {
    	return new ThinCrustDough();
    }
    public Sauce createSauce() {
    	return new MarinaraSauce();
    }
    public Cheese createCheese() {
    	return new ReggianoCheese();
    }
    public Pepperoni createPepperoni() {
    	return new SlicePepperoni();
    }
    public Clams createClam() {
    	return new FreshClams();
    }
}
```

<br/><br/>   

- `Pizza`

```java
public class CheesePizza extends Pizza {
    PizzaIngredientFactory ingredientFactory;
    
    public CheesePizza(PizzaIngredientFactory ingredientFactory) {
    	this.ingredientFactory = ingredientFactory;
    }
    
    voide prepare() {
        System.out.println("Preparing " + name);
        doug = ingredientFactory.createDough();
        sauce = ingredientFactory.createSauce();
        cheese = ingredientFactory.createCheese();
    }
}
```

<br/><br/>

- Client단 구현 : `PizzaStore`

```java
public abstract class PizzaStore {
  public Pizza orderPizza(String type) {
        Pizza pizza;
        
        pizza = createPizza(type);
        
        pizza.prepare();
        pizza.bake();
        pizza.cut();
        pizza.box();
        
        return pizza;
  }
    
  abstract Pizza createPizza(String type);
}
```

```java
public class NYPizzaStore extends PizzaStore {
	protected Pizza createPizza(String item) {
        Pizza pizza = null;
        PizzaIngredientFactory ingredientFactory = new NYPizzaIngredientFactory();
        
        if( item.equals("cheese") ) {
        	pizza = new CheesePizza(ingredientFactory);
            pizza.setName("New York Style Cheese Pizza");
        } else if( item.equals("clam") ) {
        	pizza = new ClamPizza(ingredientFactory);
            pizza.setName("New York Style Clam Pizza");
        } else if( item.equals("pepperoni") ) {
        	pizza = new PepperoniPizza(ingredientFactory);
            pizza.setName("New York Style Pepperoni Pizza");
        }

        return pizza;
    }
}
```  

=>구상 클래스에 직접 의존하지 않고도 서로 관련된 객체들로 이루어진 제품군을 만들 수 있음  

<br/><br/>

### 장단점

- 장점 : 구상 형식에 대한 의존을피하고 추상화 지향 가능
- 단점 : 새로운 ConcreteFactory를 추가할 때 많은 작업이 필요

<br/><br/>  

### 정리

- Factory Method Pattern과 유사하면서도 사용 용도가 다름
- Abstract Factory Pattern에 Factory Method Pattern을 차용할 수도 있음
- 모든 팩토리의 공통점 : 객체 생성을 캡슐화해서 애플리케이션의 결합을 느슨하게 만들고, 특정 구현에 덜 의존이게 만든다

<br/>
  
**Factory Method Pattern과 차이점**   

- 결합도를 낮추는 대상
    - 팩토리 메서드 패턴
        - ConcreteProduct와 Client 간의 결합도를 낮출때 사용
    - 추상 팩토리 패턴
        - ConcreteFactory와 Client간의 결합도를 낮출 때 사용
- Inheritance(상속), Composition(구성)
    - 팩토리 메서드 패턴
        - 상속을 사용하여 객체의 인스턴스 생성에 대해서는 서브클래스에 의존
        - 지역 레퍼런스 없이 바로 하위 클래스의 함수를 호출하여 객체를 만듦
    - 추상 팩토리 패턴
        - 지역 레퍼런스를 두어 , 외부로부터 Factory 객체를 DI 받아서 위임

<br/><br/>
  
Reference     
https://jdm.kr/blog/192  
https://flower0.tistory.com/416   
https://beomseok95.tistory.com/246   
<br/>   
