---
title: 클래스 다이어그램(Class Diagram) 정리
# author: dowonl2e
date: 2024-04-25 14:30:00 +0800
categories: [Diagram, Class Diagram]
tags: [Diagram, Class Diagram, UML]
pin: true
---

## **클래스 다이어그램 접근제어자 표기법**

- **\-** : private
- **\+** : public
- **\#** : protected

## **클래스 간의 관계**

- 일반화 관계 (Generalization)
- 실체화 관계 (Realization)
- 의존 관계 (Dependency)
- 연관 관계 & 직접 연관 관계 (Association & Direct Association)
- 집합 관계 & 합성 관계 (Aggregation & Composition)

## **일반화 관계(Generalization)**

일반화 관계는 일반적인 상속을 의미하며 **`상속 관계(IS-A 관계)`{: .text-blue}**로 표현한다.

### **UML 표기**

![일반화 UML 표기]({{site.url}}/assets/img/ClassDiagram/1_Generation_UML.png)

사람과 개발자 클래스를 예로 클래스 다이어그램과 코드로 일반화를 표현하면 아래와 같습니다.

![일반화 클래스 다이어그램 UML 표기]({{site.url}}/assets/img/ClassDiagram/2_Generation_UML2.png){: width="972" height="589" .w-25 .right}

```java
public class Person {
  private String name;
  private int age;
  private String phoneNumber;

  public void eat(){
    ...
  }
  public void study(){
    ...
  }
  public void sleep(){
    ...
  }
}

public class Developer extends Person {
  private String[] skills;
  private Double career;

  public void test(){
    ...
  }
  public void writeCode(){
    ...
  }
}
```

## **실체화 관계(Realization)**

실체화 관계는 인터페이스에 선언한 메서드를 오버라이딩하여 실제 기능을 구현하는 것을 뜻한다.

### **UML 표기**

![실체화 UML 표기]({{site.url}}/assets/img/ClassDiagram/3_Realization_UML.png)

개발에 대한 인터페이스와 사람을 예로 클래스 다이어그램과 코드로 실체화를 표현하면 아래와 같습니다.

![실체화 클래스 다이어그램 UML 표기]({{site.url}}/assets/img/ClassDiagram/4_Realization_UML2.png){: width="972" height="589" .w-25 .right}

```java
public interface Developable {
  void test();
  void writeCode();
}

public class Person implements Developable {
  private String[] skills;

  public void eat(){
    ...
  }
  public void study(){
    ...
  }
  public void sleep(){
    ...
  }
  @Override
  public void test(){
    ...
  }
  @Override
  public void writeCode(){
    ...
  }
}
```

## **의존 관계(Dependency)**

의존 관계는 한 클래스가 다른 클래스를 참조하는 것으로 대상 클래스의 객체 생성, 사용, 메서드 호출, 객체 반환, 매개변수로 해당 객체를 받는 것을 의미한다. 

### **UML 표기**

![의존관계 UML 표기]({{site.url}}/assets/img/ClassDiagram/5_Dependency_UML.png)

개발도구와 사람을 예로 클래스 다이어그램과 코드를 의존 관계를 표현하면 아래와 같습니다.

![의존관계 클래스 다이어그램 UML 표기]({{site.url}}/assets/img/ClassDiagram/6_Dependency_UML2.png)

스테레오 타입 어떠한 목적의 의존 관계인지 의미를 명시할 수 있는데 위의 경우 **parameter** 목적을 명시했습니다.

```java
public class DevelopmentTools {
  private String tool;

  public void writeCode(){
    ...
  }
}

public class Person {
  
  //create
  public DevelopmentTools(){
    return new DevelopmentTools();
  }
  
  //parameter
  public void develop(DevelopmentTools tools){
    tools.writeCode();
  }
}
```

develop 메서드가 종료되면 대상 클래스(DevelopmentTools)와의 관계는 더이상 유지되지 않습니다.

## **연관 관계 & 직접 연관 관계(Association & Direct Association)**

연관관계는 **`HAS-A`{: .text-blue}** 관계를 나타내며 한 클래스가 다른 클래스를 필드로 가지는 것을 의미한다. 연관 관계와 직접 연관 관계는 방향성의 유무이다.

## **연관관계(Association)**

두 객체가 서로를 인지하고 있는 것을 의미합니다.

### **UML 표기**

![연관관계 UML 표기]({{site.url}}/assets/img/ClassDiagram/7_Assosiation_UML.png)

고객과 주문을 예로 클래스 다이어그램과 코드를 연관관계로 표현하면 아래와 같습니다.

![연관관계 클래스 다이어그램 UML 표기]({{site.url}}/assets/img/ClassDiagram/8_Assosiation_UML2.png)

```java
public class Customer {
  private Long customerId;
  private String customerName;
  private List<Order> orders;

  public Customer(int customerId, String customerName) {
    this.customerId = customerId;
    this.customerName = customerName;
    this.orders = new ArrayList<>();
  }

  public List<Order> getOrders(){
    return orders;
  }
  public void addOrder(Order order){
    orders.add(order);
  }
}

public class Order {
  private Long orderId;
  private Customer customer;
  
  public Order(Long orderId, Customer customer){
    this.orderId = orderId;
    this.customer = customer;
  }
}
```

고객이 주문을 한다고 보면 고객의 경우 주문을 인지하고, 주문의 경우 특정 고객을 인지해야하는 관계에 있다는 것에서 두 객체가 서로를 인지하고 있습니다.

## **직접 연관 관계(Direct Association)**

방향이 있는 연관관계로 단방향을 의미하며 A에서는 B를 알지만 B에서는 A에 대한 존재를 모르는 것을 의미합니다.

### **UML 표기**

![직접연관관계 UML 표기]({{site.url}}/assets/img/ClassDiagram/9_DirectAssosiation_UML.png)

주문과 상품을 예로 클래스 다이어그램과 코드로 직접 연관 관계를 표현하면 아래와 같습니다.

![직접연관관계 클래스 다이어그램 UML 표기]({{site.url}}/assets/img/ClassDiagram/10_DirectAssosiation_UML2.png)

```java
public class Order {
  private Long orderId;
  private List<Product> products;

  public Order(Long orderId){
    this.orderId = orderId;
    this.products = new ArrayList<>();
  }
  public List<Product> getProducts(){
    return products;
  }
  public void addProduct(Product product){
    products.add(product);
  }
}

public class Product {
  private Long productId;
  private String productName;

  public Product(Long productId, String productName) {
    this.productId = productId;
    this.productName = productName;
  }
}
```

주문과 상품을 예로 주문은 상품에 대한 정보를 지니고 있습니다. 하지만 상품의 경우 주문에 대한 정보는 인지 할 필요가 없기에 직접 연관 관계로 나타낼 수 있습니다.

## **집합 관계 & 합성 관계(Aggregation & Composition)**

집합 관계는 합성관계와 함께 연관관계를 특수하게 나타낸 것이며, 합성 관계는 집합 관계와 비슷하나 집합 관계보다 조금 더 강한 집합을 의미하며, 두 관계는 **`전체와 부분의 관계`{: .text-blue}**를 나타냅니다.

- **전체(Whole)** : 다른 객체의 기능을 사용하는 객체
- **부분(Part)** : 전체 객체에 사용되는 객체

## **집합 관계(Aggregation)**

**`HAS-A`{: .text-blue}** 관계이며 전체 객체가 부분 객체를 가지고 있음을 나타냅니다. 이 관계에서 전체 객체와 부분 객체 간의 생명주기가 독립적이며, 전체 객체가 소멸되어도 부분 객체는 있을 수 있습니다.

### **UML 표기**

![집합관계 UML 표기]({{site.url}}/assets/img/ClassDiagram/11_Aggregation_UML.png)

자동차와 엔진을 예로 클래스 다이어그램과 코드로 집약 관계를 표현하면 아래와 같습니다.

![집합관계 클래스 다이어그램 UML 표기]({{site.url}}/assets/img/ClassDiagram/12_Aggregation_UML2.png)

```java
public class Car {
  private Engine engine;

  public Car(Engine engine){
    this.engine = engine;
  }
}

public class Engine {
  ...
}
```

자동차 객체(전체 객체)는 엔진 객체(부분 객체)을 포함하고 있으며, 엔진 객체(부분 객체)의 경우 자동차 객체(전체 객체)와 관계 없이 생성되므로 두 객체는 서로 독립적입니다. 

이로 인해 자동차 객체(전체 객체)가 소멸되어도 엔진 객체(부분 객체)는 소멸되지 않으며, 엔진 객체(부분 객체)는 다른 객체와 공유할 수 있습니다.

## **합성 관계(Composition)**

**`HAS-A`{: .text-blue}** 관계이며 전체와 부분의 강한 관계에 있습니다. 이 관계에서 전체 객체가 부분 객체를 포함하나 집약 관계와 반대로 전체 객체가 부분 객체의 생명주기를 관리합니다. 즉, 부분 객체는 전체 객체가 소멸될 때 같이 소멸합니다.

### **UML 표기**

![합성관계 UML 표기]({{site.url}}/assets/img/ClassDiagram/13_Composition_UML.png)

자동차와 엔진을 예로 클래스 다이어그램과 코드를 합성 관계로 표현하면 아래와 같습니다.

![합성관계 클래스 다이어그램 UML 표기]({{site.url}}/assets/img/ClassDiagram/14_Composition_UML2.png)

```java
public class Engine {
  ...
}

public class Car {
  private Engine engine;

  public Car(){
    this.engine = new Engine();
  }
}
```

자동차 객체(전체 객체)는 엔진 객체(부분 객체)을 포함하고 있으며, 엔진 객체(부분 객체)은 자동차 객체(전체 객체)에서 생성되어, 자동차 객체(전체 객체)가 소멸되면 엔진 객체(부분 객체)도 같이 소멸되게 됩니다.