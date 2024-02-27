---
title: JPA, Hibernate, Spring Data JPA 정리
# author: dowonl2e
date: 2024-02-24 12:30:00 +0800
categories: [Spring, JPA]
tags: [JPA, Hibernate, Spring Data JPA]
pin: true
img_path: "/posts/20240224"
---

## **JPA를 보기전에**

### **ORM**

Object Relational Mapping의 약자로 말 그대로 객체와 데이터베이스의 관계를 매핑하는 것이다.

비교하자면 SQL Mapper 예로 MyBatis를 사용할 때 SQL을 직접 작성하여 데이터를 조작하면 단순히 필드에만 매핑된다. 반면에, ORM은 RDB 관계를 Object에 매핑을 목적으로 객체와 테이블을 자동으로 매핑하며, 쿼리가 아닌 메서드로 데이터를 조작할 수 있다.

즉, SQL을 직접 작성하는 방식에서 ORM을 통해 객체와 관계를 일치시켜 SQL을 자동으로 생성하는 것이다.

## **JPA, Hibernate, Spring Data JPA**

JPA, Hibernate, Spring Data JPA, RDB의 구조를 보면 아래와 같다.

![JPA Hibernate Spring Data JPA RDB]({{site.url}}/assets/img/JPA/1_JPA Hibernate.png)

[<center>https://medium.com/@dafikabukcu/what-is-hibernate-and-jpa-ef77ba1dac15</center>](https://medium.com/@dafikabukcu/what-is-hibernate-and-jpa-ef77ba1dac15){:target="\_blank"}

### **JPA**

Java Persistence API의 약자로 자바에서 관계형 데이터베이스를 사용에 있어서 필요한 기능을 제공하며 위에서 말한 ORM 기술에 대한 API 표준 명세이다. 라이브러리와 같이 특정 기능을 담당하는 것이 아니라 ORM을 사용하기 위해 만든 **인터페이스의 모음**{: .text-blue }이다. 그래서 명세이기 때문에 구현이 없다. 따라서 구현체가 있어야 JPA를 사용할 수 있다.

### **Hibernate**

Hibernate는 JPA의 구현체 중 하나이며, Java를 위한 ORM 프레임워크 중 하나이다. 내부적으로 JDBC API를 사용한다. 앞서 얘기한 MyBatis를 대신해 Hibernate를 사용한다고 보면된다.

### **Spring Data JPA**

JPA와 같은게 아닌가? 라고 생각할 수 있지만, JPA를 쉽게 사용할 수 있도록 Spring에서 제공하고 있는 프레임워크이다.

Hibernate와 Spring Data JPA를 이용했을때에 큰 차이가 없지만 Spring Data의 일관된 프로그래밍 모델로 인한 **구현체와 저장소 교체의 용이성**{: .text-blue }의 이유로 Spring Data JPA를 이용하는게 편하다.
