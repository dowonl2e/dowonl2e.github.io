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

### **JPA**

Java Persistence API의 약자로 자바에서 관계형 데이터베이스를 사용에 있어서 필요한 기능을 제공하며 위에서 말한 ORM 기술에 대한 API 표준 명세이다. 라이브러리와 같이 특정 기능을 담당하는 것이 아니라 ORM을 사용하기 위해 만든 **인터페이스의 모음**{: .text-blue }이다. 명세이기 때문에 구현이 없다. 따라서 구현체가 있어야 JPA를 사용할 수 있다.

**영속성 컨텍스트(Persistence Context)**

JPA에는 영속성 컨텍스트가 있다. 영속성 컨텍스트란 **엔티티가 계속해서 같은 상태를 유지하는 환경**을 의미하며, 주로 다음과 같은 기능을 수행한다.

- 엔티티 관리 : 엔티티의 생명주기를 관리한다.
- 엔티티 상태 추적 : 엔티티에 변경이 있을 경우 DB에 변경사항을 반영한다.
- 1차 캐시 : 엔티티 객체를 보관하는 저장소이다. 반복적인 DB 커넥션 횟수를 줄여 성능을 향상시킨다.
- 트랜잭션 지원 : 트랜잭션 내에 발생한 엔티티 변경은 트랜잭션이 커밋이 될 때 자동으로 플러시를 발생시켜 DB에 반영한다.
- 지연 로딩(Lazy Loading) : 엔티티를 실제 필요한 시점에 로딩하여 성능을 최적화 한다.
- 동일성 보장 : 동일한 엔티티 식별자를 가진 객체의 동일성을 보장한다. 식별자를 통한 같은 엔티티를 조회할 때 동일한 객체를 반환한다.

DB에서 데이터를 조회할 때 네트워크 연결 그리고 DB의 리소스 사용 등 시간을 생각했을 때 애플리케이션의 리소스를 사용하는 것보다 시간적인 비용이 많이 소모된다. 개발을 할때 서로 다른 N개의 테이블을 조회할 필요가 있을 때도 최대 N번의 커넥션이 필요할 수 있어 DB 부하도 일으키게 된다. 이러한 부하를 줄이고자 JPA에서는 캐시가 존재하며 이에 대해서 알아보고자 한다.

**1차, 2차 캐시에 대해서**

**(1) 1차 캐시**

영속성 컨텍스트 내부에 존재하는 메모리 공간이며 여기에 엔티티를 보관한다. 엔티티가 영속상태에 있으면 해당 엔티티는 1차 캐시에 저장되어 있는 것이다. 이 후에 해당 데이터를 조회하면 DB 커넥션 없이 1차 캐시에서 가져온다. 반복적인 DB 조회를 방지함으로서, 성능을 향상시킬 수 있다.

그러나, 동일한 트랜잭션 내에서만 유효하기 때문에 해당 트랜잭션이 종료되거나 다른 트랜잭션에서 작업을 해야하는 경우에 DB에서 데이터를 가져온다.

1차 캐시를 통해서 DB 커넥션을 어느정도 줄일 수 있지만, 전체적으로 보면 획기적으로 줄어들지는 않는다.

**(2) 2차 캐시**

2차 캐시는 공유 캐시라고도 하며 영속성 컨텍스트가 아닌 JPA의 구현체(Hibernate, Spring Data JPA 등)가 애플리케이션 범위에서 지원한다. 1차 캐시보다 더 넓은 범위에서 메모리를 사용할 수 있기에 1차 캐시만 사용했을 때 보다 획기적으로 DB 액세스 횟수를 줄일 수 있다. 2차 캐시는 1차 캐시와 다르게 사용여부를 설정할 수 있다.

2차 캐시를 사용하면 엔티티 매니저는 1차 캐시를 먼저 찾고 없으면 2차 캐시에서 찾는다. 둘다 없으면 DB를 조회한다. 찾는 엔티티가 2차 캐시에 있을 경우 원본이 아닌 **복사본**{: .text-blue }을 반환받는다.

이 이유는 원본을 반환 받게되면 여러 곳에서 동일한 객체를 동시에 수정하는 문제가 발생할 수 있기에 복사본을 반환해주는 것이다. 즉, **동시성**{: .text-blue }을 극대화하기 위함이다.

> **영속성 컨텍스트가 다르면 객체 동일성을 보장하지 않는다.**<br/>2차 캐시는 여러 영속성 컨텍스트에 공유될 수 있다. 영속성 컨텍스트는 독립적으로 엔티티를 관리하며, 각각의 영속성 컨텍스트에서 관리하는 엔티티는 서로 다르다. 동일한 엔티티를 서로 다른 영속성 컨텍스트에서 조회할 때 두 엔티티는 서로 다른 객체라는 의미이다.<br/>그렇지만, 서로 다른 영속성 컨텍스이더라도 엔티티의 고유 식별자를 통해서 동일한 엔티티인지 확인할 수 있다.
{: .prompt-info }

**엔티티(Entity)**

엔티티는 데이터베이스의 테이블을 의미한다. 테이블이라는 것을 명시하기위해서 클래스에 @Entity 선언해주면 이 클래스는 엔티티 클래스가 된다. 그리고 엔티티 클래스에 선언된 변수는 테이블의 필드에 해당된다.

엔티티에는 4가지 생명주기가 있다.
![Entity 생명주기]({{site.url}}/assets/img/JPA/Entity Lifecycle.png)

- **비영속(new/transient) 상태** : 영속성 컨텍스트와 관계 없으며, Java단에서 인스턴스화만 된 상태이다.
- **영속(managed) 상태** : 영속성 컨텍스트에 저장되어 관리할 수 있는 상태
- **준영속(detached) 상태** : 영속성 컨텍스트에 저장되었다가 분리된 상태
- **삭제(removed) 상태** : 연속성 컨텍스트 그리고 데이터베이스에서 삭제된 상태

영속상태에 있는 엔티티의 경우 트랜잭션이 종료되면 플러쉬가 발생하여 변경사항이 자동으로 DB에 반영된다.

### **Hibernate**

Hibernate는 JPA의 구현체 중 하나이며, Java를 위한 ORM 프레임워크 중 하나이다. 내부적으로 JDBC API를 사용한다. 앞서 얘기한 MyBatis를 대신해 Hibernate를 사용한다고 보면된다.

> [JPA + Hibernate 사용해보기](/posts/JPA-+-Hibernate-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0/){:target="\_blank"}

### **Spring Data JPA**

JPA와 같은게 아닌가? 라고 생각할 수 있지만, JPA를 쉽게 사용할 수 있도록 Spring에서 제공하고 있는 프레임워크이다.

Hibernate와 Spring Data JPA를 이용했을때에 큰 차이가 없지만 Spring Data의 일관된 프로그래밍 모델로 인한 **구현체와 저장소 교체의 용이성**{: .text-blue }의 이유로 Spring Data JPA를 이용하는게 편하다.

> [JPA + Spring Data JPA 사용해보기(CRUD)](/posts/JPA-+-Spring-Data-JPA-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0_1/){:target="\_blank"}

> [JPA + Spring Data JPA 사용해보기(Dynamic Query, Paging, Sort)](/posts/JPA-+-Spring-Data-JPA-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0_2/){:target="\_blank"}

> [JPA + Spring Data JPA 사용해보기(Join)](/posts/JPA-+-Spring-Data-JPA-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0_3/){:target="\_blank"}