---
title: JPA + Hibernate 사용해보기
# author: dowonl2e
date: 2024-02-24 14:00:00 +0800
categories: [Spring, JPA]
tags: [JPA, Hibernate]
pin: true
img_path: "/assets"
image:
  path: /commons/Hibernate.jpeg
  alt: Hibernate
---

## **개발 환경**

- IDE : IntelliJ IDEA (Community Edition)
- JDK : Java 17
- Framework : Spring Boot 3
- DB : MySQL 8.0
- DBCP : HikariCP

## **Spring Project 생성**

Spring Initializr 웹 도구 사이트[(https://spring.io)](https://start.spring.io/)에서 Spring Project를 생성해줍니다.

### **Dependency 추가**

- Spring Web
- MySQL Driver
- Spring Data JPA

## **애플리케이션 옵션 설정**

```yaml
#DataSource & HikariCP 설정
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://{DB 호스트}:{DB 포트}/{DB명}?serverTimezone=Asia/Seoul
    username: { DB 계정명 }
    password: { DB 비밀번호 }
    hikari:
      connection-test-query: SELECT NOW() FROM dual #  DB Connection 테스트 쿼리
      connection-timeout: 30000 # DB Connection 테스트 쿼리 실행 주기
  jpa:
    open-in-view: true
    show-sql: true
    hibernate:
      ddl-auto: create
      naming:
        physical-strategy: com.study.jpahibernate.strategy.CustomCamelCaseToSnakeNamingStrategy
    properties:
      hibernate:
        format_sql: true
        highlight_sql: true
        use_sql_comments: false

logging:
  level:
    org:
      springframework:
        orm:
          jpa: debug
```

### **ddl-auto 속성**

중요 속성은 **jpa.hibernate.ddl-auto**이며 **create, create-drop, update, validate, none**{: .text-blue } 값 중 하나를 설정할 수 있다.

- **create** : 엔티티로 등록된 클래스와 매핑되는 테이블을 자동으로 생성해준다. 이미 생성된 클래스가 있다면 기존 테이블을 삭제(drop) 후 다시 생성(create)한다.
  ![JPA Hibernate Create]({{site.url}}/assets/img/JPA/2_Hibernate Create.png)

- **create-drop** : create와 동일한 기능을 하되, 추가로 애플리케이션이 종료될때 테이블을 삭제(drop) 한다.
  ![JPA Hibernate Create-Drop]({{site.url}}/assets/img/JPA/3_Hibernate Create-drop.png)

- **update** : 엔티티로 등록된 클래스와 매핑되는 테이블이 없으면 자동으로 생성해주며, 매핑되는 테이블이 있으면 테이블 세부 구조(컬럼, 타입 등)이 변경된 내용이 있으면 변경한다.
  ![JPA Hibernate Update]({{site.url}}/assets/img/JPA/4_Hibernate Update.png)

- **validate** : 테이블을 추가 또는 삭제하지 않고 엔티티로 등록된 클래스와 테이블이 정상적으로 매핑되어있는지 유효성 검사를 한다. 만약, 매핑이 비정상적이면 오류 발생 후 애플리케이션을 종료된다.
  ![JPA Hibernate Validate]({{site.url}}/assets/img/JPA/5_Hibernate Validate.png)

- **none(default)** : 아무 일도 일어나지 않는다. ddl-auto 옵션을 추가하지 않거나 선언시에는 none으로 설정해주면된다.

### **주의사항**

ddl-auto 속성의 경우 테이블을 자동으로 생성해주고 테이블 구조를 변경할 수 있어 편리해보인다. 프로젝트 개발 시작 단계에서는 create 또는 create-drop을 사용할 수 있지만, 개발 도중에 테이블이 드랍되면 테스트 데이터도 날라가는 경우가 생기게 된다.

update도 문제가 있다. 특정 컬럼이 null 허용에서 not null로 변경하거나 unique 옵션을 추가해야하는 순간 사이드 이펙트도 고려해야한다.

특히, 운영 DB에서 위 이슈가 발생하지 않도록 해야한다. 결국, 개발 시작부터 운영까지 과정에서 사용해야하는 부분을 정리한다면 아래와 같이 사용하는 것을 권장한다.

> - **개발 초기 로컬서버 : create, update**
> - **테스트서버 : update, validate**
> - **운영서버 : validate, none**

> 위 기능에 대해서는 정말로 편해보이고 시간적인 비용을 줄일 수 있지만, 운영과정에서 사용하기에는 위험부담이 크다. 결국, 개발자가 직접 관리해야하는 작업은 필요해 보인다.
{: .prompt-warning }

## **Board Entity 클래스 생성**

**Board.java**

```java
@Entity(name = "tb_board")
@Getter
@Setter
@ToString
@NoArgsConstructor
public class Board {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String title;
  private String contents;
  private LocalDateTime writeDate = LocalDateTime.now();
  private LocalDateTime modifyDate;

  @Builder
  public Board(Long id, String title, String contents){
    this.id = id;
    this.title = title;
    this.contents = contents;
  }
}
```

- @Entity : 클래스가 엔티티임을 지정하는 애너테이션이다. (테이블명을 tb\_로 시작되도록 작성했을 경우 name 속성에 따로 테이블명을 지정 가능)
- @Id : 필드가 PK임을 뜻하는 애너테이션
- @GeneratedValue : PK에 대한 전략을 생성하는 애너테이션이다.
  - GenerationType.IDENTITY : 식별자 역할인 PK 생성을 JPA가 아닌 DB에 위임하는 것을 의미하며, DB의 스키마 생성 시 지정했던 PK 생성 전략에 따라 만들게된다. MySQL을 예로 PK가 숫자형 타입의 경우 AUTO_INCREMENT로 지정된다.

![JPA 엔티티 클래스]({{site.url}}/assets/img/JPA/10_Entity Class And Table.png)

hibernate를 통해 엔티티와 매핑되는 테이블은 위와 같이 생성된다. String 타입의 경우 기본 크기는 255로 되는 것을 볼 수 있다. 컬럼의 크기를 다르게 하려면 Column 애너테이션을 이용하면 된다.

```java
@Entity(name = "tb_board")
@Comment("게시판")
@Getter
@Setter
@ToString
@NoArgsConstructor
public class Board {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  @Comment("PK")
  private Long id;

  @Column(length = 300)
  @Comment("제목")
  private String title;

  @Column(length = 1000)
  @Comment("내용")
  private String contents;

  @Comment("작성일")
  private LocalDateTime writeDate = LocalDateTime.now();

  @Comment("수정일")
  private LocalDateTime modifyDate;

  @Builder
  public Board(Long id, String title, String contents){
    this.id = id;
    this.title = title;
    this.contents = contents;
  }
}
```

- @Column : 컬럼을 세부속성(컬럼명, 디폴트, 유니크, 크기 등)을 지정
- @Comment : 테이블 또는 필드에 코멘트를 지정

> 엔티티 클래스에 필드 네이밍 전략은 CamelCase이지만 DB의 경우 SnakeCase이다. 프로그램 네이밍 전략은 CamelCase이고 DB는 SnakeCase로 사용하는 경우가 있다. 이것을 맞추기위해 application.yml에 spring.jpa.hibernate.naming.physical-strategy 속성에 네이밍 전략을 지정해준다.
{: .prompt-info }

```java
public class CustomCamelCaseToSnakeNamingStrategy extends CamelCaseToUnderscoresNamingStrategy {
  @Override
  protected Identifier getIdentifier(String name, final boolean quoted, final JdbcEnvironment jdbcEnvironment) {
    if ( isCaseInsensitive( jdbcEnvironment ) ) {
      name = name.toUpperCase( Locale.ROOT );
    }
    return new Identifier( name, quoted );
  }
}
```

> org.hibernate.boot.model.naming.CamelCaseToUnderscoresNamingStrategy를 사용해서 지정해줄 수 있지만, 테이블 필드가 모두 소문자로 생성되기에 대문자로 생성하고 싶을 경우 커스텀 네이밍 전략을 이용하면 된다.
{: .prompt-info }

## **EntityManager 테스트해보기**

### **테스트 클래스에 EntityManagerFactory 주입**

```java
@Autowired
private EntityManagerFactory factory;
```

### **1) 데이터 추가**

**테스트 코드 작성**

```java
@Test
@DisplayName("JPA Hibernate 추가 테스트")
public void testToSave(){
  EntityManager manager = factory.createEntityManager();
  EntityTransaction transaction = manager.getTransaction();
  transaction.begin();

  BoardDto boardDto = new BoardDto();
  boardDto.setId((long)1);
  boardDto.setTitle("JPA Hibernate 테스트 제목1");
  boardDto.setContents("JPA Hibernate 테스트 내용1");
  Board board1 = boardDto.toEntity();
  manager.persist(board1);

  transaction.commit();

  manager.close();
}
```

**테스트 결과**

![JPA Insert]({{site.url}}/assets/img/JPA/6_JPA Insert.png)

### **2) 데이터 조회**

**테스트 코드 작성**

```java
@Test
@DisplayName("JPA Hibernate 조회 테스트")
public void testToFind(){
  EntityManager manager = factory.createEntityManager();
  Board board = manager.find(Board.class, (long)1);
  System.out.println(board.toString());
}
```

**테스트 결과**

![JPA Select]({{site.url}}/assets/img/JPA/7_JPA Select.png)

### **3) 데이터 수정**

**테스트 코드 작성**

```java
@Test
@DisplayName("JPA Hibernate 수정 테스트")
public void testToUpdate(){
  EntityManager manager = factory.createEntityManager();
  EntityTransaction transaction = manager.getTransaction();
  transaction.begin();

  Board board = manager.find(Board.class, (long)1);
  board.setTitle("JPA Hibernate 테스트 제목 수정1");
  board.setContents("JPA Hibernate 테스트 내용 수정1");
  board.setModifyDate(LocalDateTime.now());

  transaction.commit();

  manager.close();
}
```

데이터 수정의 경우 수정하고자 하는 데이터를 먼저 가져와 해당 엔티티의 필드 값을 변경해주면 데이터가 수정된다.

**테스트 결과**

![JPA Update]({{site.url}}/assets/img/JPA/8_JPA Update.png)

### **4) 데이터 삭제**

**테스트 코드 작성**

```java
@Test
@DisplayName("JPA Hibernate 삭제 테스트")
public void testToRemove(){
  EntityManager manager = factory.createEntityManager();
  EntityTransaction transaction = manager.getTransaction();
  transaction.begin();

  Board board = manager.find(Board.class, (long)1);
  manager.remove(board);

  transaction.commit();

  manager.close();
}
```

**테스트 결과**
![JPA Delete]({{site.url}}/assets/img/JPA/9_JPA Delete.png)
