---
title: Spring Boot + JPA Transaction
# author: dowonl2e
date: 2024-02-21 14:25:00 +0800
categories: [Spring, Transaction]
tags: [Transaction]
pin: true
img_path: "/posts/20240221"
---

## **개발 환경**

- IDE : IntelliJ IDEA (Community Edition)
- JDK : Java 17
- Framework : Spring Boot 3
- DB : MySQL 8.0
- DBCP : HikariCP

### **Spring Project 생성**

Spring Initializr 웹 도구 사이트[(https://spring.io)](https://start.spring.io/)에서 Spring Project를 생성해줍니다.

**Dependency 추가**

- Spring Web
- MySQL Driver
- Spring Data JPA
- Spring Data JDBC
- Lombok

## **애플리케이션 옵션 설정**

### **Datasource & HikariCP 옵션 설정**

**application.yml**

```yaml
#DataSource & HikariCP 설정
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://{DB 호스트}:{DB 포트}/{DB명}?serverTimezone=Asia/Seoul
    username: { DB 계정명 }
    password: { DB 비밀번호 }
    hikari:
      connection-test-query: SELECT NOW() FROM DUAL
      connection-timeout: 30000
      validation-timeout: 30000
      minimum-idle: 5
      max-lifetime: 240000
      maximum-pool-size: 20
  jpa:
    open-in-view: true # JPA의 영속성 컨텍스트가 DB 커넥션 반환 기점을 설정
    show-sql: true # JPA SQL  로그 출력 여부를 설정
    hibernate:
      ddl-auto: none
      naming:
        physical-strategy: com.study.jpatransaction.strategy.CustomCamelCaseToSnakeNamingStrategy
    properties:
      hibernate:
        format_sql: true
        highlight_sql: true # Hibernate에 실행되는 SQL 로그 출력 라인 포맷에 맞게 예쁘게 출력할지를 설정
        use_sql_comments: false # Comment를 출력할지를 설정

logging:
  level:
    org:
      springframework:
        transaction: trace
        orm:
          jpa: debug
```

Spring 2 버전 이후부터는 디폴트 DBCP가 HikariCP 입니다.

- **jpa.open-in-view**
  - **true** : Client에 응답이 완료 된 후 DB 커넥션을 반환
  - **false** : 해당 메서드가 끝날 때 DB 커넥션을 반환
- **jpa.hibernate.naming.physical-strategy** : JPA 테이블의 DDL 네이밍 전략을 설정

**DataSource 설정**

```java
@Configuration
public class DataSourceConfig {

  @Bean
  @Primary
  @ConfigurationProperties("spring.datasource")
  public DataSourceProperties dataSourceProperties() {
    return new DataSourceProperties();
  }

  @Bean
  @ConfigurationProperties("spring.datasource.hikari")
  public HikariDataSource dataSource(DataSourceProperties properties) {
    return properties.initializeDataSourceBuilder().type(HikariDataSource.class).build();
  }

}
```

**HikariCP 설정 결과**
![HikariCP 설정 결과1]({{site.url}}/assets/img/jpatransaction/0-1_HikariCP Test.png)
![HikariCP 설정 결과2]({{site.url}}/assets/img/jpatransaction/0-2_HikariCP Test.png)


### **JPA DDL 네이밍 전략 설정**

**CustomCamelCaseToSnakeNamingStrategy.java**

Hibernate를 통해 신규 테이블을 생성 또는 컬럼 변경이 있을 경우 엔터티에 클래스 또는 변수 명칭을 카멜 케이스 네이밍에서 스네이트 케이스 네이밍 변환을 적용한다.

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

## **JPA 트랜잭션 테스트**

### 1) 트랜잭션 선언

**BoardService.java**

```java
@Transactional
public Long save(BoardDto boardDto){
  return boardRepository.save(boardDto.toEntity()).getId();
}
```

**테스트 결과**
![JPA Transactional 선언 결과]({{site.url}}/assets/img/jpatransaction/1_JPA Transactional Test.png)

### **2) 트랜잭션 미선언**

**BoardService.java**

```java
public Long save(BoardDto boardDto){
  return boardRepository.save(boardDto.toEntity()).getId();
}
```

**테스트 결과**
![JPA Transactional 미선언 결과]({{site.url}}/assets/img/jpatransaction/2_JPA No Transactional Test.png)
결과를 보면 두 케이스는 트랜잭션이 정상적으로 시작되고 끝나는 것을 볼 수 있다. 그런데, Transactional을 사용했을 때 로그에서 **Participating in existing transaction**이라는 문구를 볼 수 있다.

이 의미는 JPARepository 자체적으로 트랜잭션이 사용되고 있으며 자체 트랜잭션을 사용한다는 것으로 보인다.

## **그러면 왜 Service에 트랜잭션을 선언하는 경우가 있는가?**

JPA에 대해 공부하면서 서비스의 클래스 또는 메서드에 트랜잭션을 선언하는 경우가 있었다. JPARepository는 자체적으로 트랜잭션이 있지만 왜 그런지 궁금하여 롤백에 관해서도 테스트 해보았다.

### **1) 트랜잭션 선언 - IOException 예외 발생**

**BoardService.java**

```java
@Transactional(rollbackFor = Exception.class)
public void multiSaveTransactional() throws Exception {
  for(int i = 0 ; i < 10 ; i++){
    BoardDto boardDto = new BoardDto();
    boardDto.setTitle("JPA 트랜잭션 테스트 제목1");
    boardDto.setContents("JPA 트랜잭션 테스트 내용1");
    boardRepository.save(boardDto.toEntity());

    if(i == 5)
      throw new IOException();

  }
}
```

**테스트 결과**
![JPA 다중 데이터 처리 Transactional 선언 결과]({{site.url}}/assets/img/jpatransaction/3_JPA Multi Insert Transactional Result.png)
![JPA 다중 데이터 처리 Transactional 선언 데이터 결과]({{site.url}}/assets/img/jpatransaction/4_JPA Multi Insert Transactional Data Result.png)

예외 발생전에 6개의 데이터 추가는 정상처리 중에 있었지만 오류가 발생하면서 롤백이 되었다.

### **2) 트랜잭션 미선언 - IOException 예외 발생**

**BoardService.java**

```java
public void multiSaveNoTransactional() throws Exception {
  for(int i = 0 ; i < 10 ; i++){
    BoardDto boardDto = new BoardDto();
    boardDto.setTitle("JPA 트랜잭션 테스트 제목1");
    boardDto.setContents("JPA 트랜잭션 테스트 내용1");
    boardRepository.save(boardDto.toEntity());

    if(i == 5)
      throw new IOException();
  }
}
```

**테스트 결과**

![JPA 다중 데이터 처리 Transactional 미선언 결과]({{site.url}}/assets/img/jpatransaction/5_JPA Multi Insert No Transactional Result.png)
![JPA 다중 데이터 처리 Transactional 미선언 결과 데이터]({{site.url}}/assets/img/jpatransaction/6_JPA Multi Insert No Transactional Data Result.png)

트랜잭션 선언시 결과와 다르게 앞의 6개 데이터 처리는 커밋이 되었고 오류 발생 후 롤백이 처리가 되지 않았습니다.

JPARepository 트랜잭션은 싱글 트랜잭션으로 보인다. 그래서 로그에서 앞선 6개의 데이터 처리에 대해 각각의 트랜잭션이 동작하는 것도 보였다. JPA 트랜잭션에 대해서 찾다보면 단일 처리에 대한 메서드에서는 트랜잭션 선언이 없는 경우도 있지만, 다중 처리에 대해서는 트랜잭션이 선언되어있는 것을 볼 수 있었다.

## **읽기 전용 트랜잭션**

JPA에서 트랜잭션을 선언하다보면 특정 메서드 부분에 readOnly 속성을 볼 수 있다. 해당 속성이 무엇이고 왜 사용하는지 알아보자.

정리하기 전 JPA의 영속성 컨텍스트를 먼저 살펴보면

### **영속성 컨텍스트**

영속성 컨텍스트는 엔티티를 영구 저장하는 환경이라는 뜻으로, 애플리케이션과 DB사이에서 관리하는 가상의 DB 역할을 한다고 보면된다. 엔티티 매니저를 통해 엔티티를 저장하거나 조회하면 앤티티 매니저는 영속성 컨텍스트에 보관하고 관리합니다.

영속성 컨텍스트에는 4가지 상태가 있다.

- **비영속 상태** : 영속성 컨텍스트와 관계 없는 상태
- **영속 상태** : 영속성 컨텍스트에 저장된 상태
- **준영속 상태** : 영속성 컨텍스트에 저장되었다가 분리된 상태
- **삭제 상태** : 연속성 컨텍스트에서 삭제된 상태

### 1) **1차 캐시**

영속성 컨텍스트에는 1차 캐시가 있다. 엔티티를 1차 캐시에 저장해둔 상태이며 만약 엔티티를 조회했을 때 1차 캐시에 해당 엔티티가 있다면 DB를 조회하지 않고 1차 캐시에서 데이터를 가져온다.

**영속상태 후 조회 테스트**

```java
@Test
public void testToPersistenceContext() {
  EntityManager entityManager = entityManagerFactory.createEntityManager();
  EntityTransaction transaction = entityManager.getTransaction();
  transaction.begin();

  BoardDto boardDto = new BoardDto();
  boardDto.setId((long)17);
  boardDto.setTitle("JPA 트랜잭션 테스트 제목1");
  boardDto.setContents("JPA 트랜잭션 테스트 내용1");
  Board board1 = boardDto.toEntity();
  entityManager.persist(board1); // => entityManager에 객체를 저장(영속)

  transaction.commit();

  Board board2 = entityManager.find(Board.class, 17); // => 데이터 조회
  assertEquals("board1 == board2", board1, board2);
}
```

**테스트 결과**
![JPA 영속성 컨텍스트 테스트]({{site.url}}/assets/img/jpatransaction/7_JPA EntityManager Persistence Context Result.png)

엔티티 매니저에 객체를 저장해 영속 상태 만들고 데이터를 저장했다. 이후 저장한 데이터를 조회했을 때 영속성 컨텍스트의 1차 캐시에 해당 엔티티가 존재하므로 DB를 조회하지 않는다.

**영속 상태에서 준영속 상태로 변경 후 조회 테스트**

```java
@Test
public void testToPersistenceContext() {
  EntityManager entityManager = entityManagerFactory.createEntityManager();
  EntityTransaction transaction = entityManager.getTransaction();
  transaction.begin();

  BoardDto boardDto = new BoardDto();
  boardDto.setId((long)17);
  boardDto.setTitle("JPA 트랜잭션 테스트 제목1");
  boardDto.setContents("JPA 트랜잭션 테스트 내용1");
  Board board1 = boardDto.toEntity();
  entityManager.persist(board1);

  transaction.commit();

  entityManager.detach(board1); // => 준영속 상태로 변경

  Board board2 = entityManager.find(Board.class, 17);
  assertEquals("board1 == board2", board1, board2);
}
```

**테스트 결과**
![JPA 영속성 컨텍스트 준영속상태 테스트]({{site.url}}/assets/img/jpatransaction/8_JPA EntityManager Persistence Context Detach Result.png)

결과를 보면 우선 영속 상태에서 준영속 상태로 1차 캐시에 없어져 관리에서 벗어나게되고 데이터 조회시 DB에 접근하여 데이터를 조회하는 것을 볼 수 있습니다.

### **트랜잭션 readOnly**

JPA는 영속성 컨텍스트에 Entity를 보관할 때 최초의 상태로 저장하는데, 이것을 **스냅샷**이라고 하며 영속성 컨텍스트가 Flush 되는 시점에 스냅샷과 Entity를 비교하여 달라진 Entity를 찾는다. 만약 변경된 Entity가 있다면 쓰기지연 SQL 저장소에 변경 쿼리문을 보관했다가, 모든 작업이 끝나면 해당 쿼리를 DB에 전달한다.

그러나 readOnly 옵션을 적용하면 스프링은 Hibernate의 세션 플러시 모드를 MANUAL로 설정한다. 이 모드는 강제로 Flush를 호출하지 않으면 Flush가 발생하지 않는다.

따라서 트랜잭션을 커밋하더라도 영속성 컨텍스트가 Flush 되지 않아 엔티티의 등록, 수정, 삭제 등의 동작을 하지 않고, 또한 읽기 전용으로, 영속성 컨텍스트는 변경감지에 대한 **스냅샷**을 보관하지 않기 때문에 메모리가 절약되는 성능의 이점이 존재한다.

> 강제로 Flush를 호출하지 않는 이상 데이터 처리에 대한 동작을 하지 않기 때문에 조회(SELECT) 용도로만 사용해야한다.
{: .prompt-warning }

> Github : [https://github.com/dowonl2e/jpatransaction](https://github.com/dowonl2e/jpatransaction)
