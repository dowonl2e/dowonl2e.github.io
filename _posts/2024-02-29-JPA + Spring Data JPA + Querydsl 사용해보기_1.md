---
title: JPA + Spring Data JPA + Querydsl 사용해보기(1)
# author: dowonl2e
date: 2024-03-01 10:00:00 +0800
categories: [Spring, JPA]
tags: [JPA, Spring Data JPA, QueryDSL]
pin: true
img_path: "/assets"
image:
  path: /commons/QueryDSL.png
  alt: QueryDSL
---

## **Querydsl란?**

Querydsl는 정적 타입을 이용해서 SQL과 같은 쿼리를 생성할 수 있도록 해 주는 **프레임워크**다. SQL을 직접 작성하거나 XML 파일에 쿼리를 작성하는 대신, Querydsl이 제공하는 **플루언트(Fluent) API**를 이용해서 쿼리를 생성할 수 있다.

단순 문자열과 비교해서 Fluent API를 사용할 때의 장점은 다음과 같다.

1. IDE의 코드 자동 완성 기능 사용
2. 문법적으로 잘못된 쿼리를 허용하지 않음
3. 도메인 타입과 프로퍼티를 안전하게 참조할 수 있음
4. 도메인 타입의 리팩토링을 더 잘 할 수 있음

Querydsl의 핵심 원칙은 **타입 안정성(Type safety)**{: .text-blue }이다. 도메인 타입의 프로퍼티를 반영해서 생성한 쿼리 타입을 이용해서 쿼리를 작성하게 된다. 또한, 완전히 타입에 안전한 방법으로 함수/메서드 호출이 이루어진다.

또 다른 중요한 원칙은 **일관성(consistency)**{: .text-blue }이다. 기반 기술에 상관없이 쿼리 경로와 오퍼레이션은 모두 동일하며, Query 인터페이스는 공통의 상위 인터페이스를 갖는다.

모든 쿼리 인스턴스는 여러 차례 재사용 가능하다. 쿼리 실행 이후 페이징 데이터와 프로젝션 정의는 제거된다.

> **Fluent API**<br/>메소드 체이닝에 기반한 객체지향 API 설계 메소드
{: .prompt-info }

## **개발환경 및 프로젝트 세팅**

개발 환경 및 프로젝트 설정은 JPA + Spring Data JPA에서 프로젝트를 이용하겠습니다.

> [JPA + Spring Data JPA 사용해보기](/posts/JPA-+-Spring-Data-JPA-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0/){:target="\_blank"}

## **Querydsl 설정**

### **build.gradle**

dependencies 안에 querydsl 및 QClass 저장 디렉토리 설정 코드를 추가해준다.

```java
dependencies {
  ...

  implementation "com.querydsl:querydsl-jpa:5.0.0:jakarta"  //추가
  annotationProcessor "com.querydsl:querydsl-apt:5.0.0:jakarta"  //추가
  annotationProcessor "jakarta.persistence:jakarta.persistence-api"  //추가

  ...
}

def querydslSrcDir = "src/main/generated" // Q파일 경로

clean {
  delete file(querydslSrcDir)
}

tasks.withType(JavaCompile) {
  options.generatedSourceOutputDirectory = file(querydslSrcDir)
}
```

### **QuerydslTestConfig.java 생성**

```java
@TestConfiguration
public class QuerydslTestConfig {

  @PersistenceContext
  private EntityManager entityManager;

  @Bean
  public JPAQueryFactory jpaQueryFactory(){
    return new JPAQueryFactory(entityManager);
  }  
}
```

- Querydsl를 통해 쿼리를 작성하기 위해서 JPAQueryFactory가 필요하다. EntityManager를 통해 쿼리 결과를 반환할 수 있도록 빈을 추가한다.
- 테스트를 위해 설정파일을 테스트 클래스 경로에 추가해줍니다.

### **BoardResponseDto.java**
```java
@Getter
@Setter
@RequiredArgsConstructor
public class BoardResponseDto {

  private Long id;
  private String title;
  private String contents;
  private LocalDateTime writeDate;
  private LocalDateTime modifyDate;

  @QueryProjection
  public BoardResponseDto(
      Long id, String title, String contents,
      LocalDateTime writeDate, LocalDateTime modifyDate
  ){
    this.id = id;
    this.title = title;
    this.contents = contents;
    this.writeDate = writeDate;
    this.modifyDate = modifyDate;
  }
}
```

- Querydsl를 통해 쿼리 작성시에 특정 컬럼만 필요할 경우에 생성자를 따로 추가하여 `@QueryProjection`을 추가해서 사용할 수 있다.

Gradle 설정 및 BoardResponseDto 클래스 파일을 생성한 후 gradle에서 컴파일을 한다.
![Gradle Compile Java]({{site.url}}/assets/img/Querydsl/1_compileJava.png)
**compileJava**를 실행하면 프로젝트 구조의 generated 디렉토리 안에 Q파일이 생성되어있는 것을 볼 수 있다. Q파일이 생성되는 대상은 엔티티 클래스, 생성자에 `@QueryPojection` 선언된 클래스이며, QBoardResponseDto.java 파일을 보면 com.querydsl.core.types.Expression로 각 필드의 타입을 표현한다.

## **Querydsl 코드 작성**

### **BoardRepositoryCustom.java 인터페이스 생성**
```java
public interface BoardRepositoryCustom {

  List<BoardResponseDto> getSearchBoards();

  BoardResponseDto getBoard(final Long id);

  long insertBoard(final BoardDto boardDto);

  long updateBoard(final BoardDto boardDto);

  long deleteBoard(final Long id);
}
```

### **BoardRepository.java 인터페이스 수정**

BoardRepositoryCustom 클래스를 상속받도록 코드를 수정한다.
```java
@Repository
public interface BoardRepository extends JpaRepository<Board, Long>, BoardRepositoryCustom {

}
```

### **BoardRepositoryImpl.java 클래스 생성**
```java
@RequiredArgsConstructor
public class BoardRepositoryImpl implements BoardRepositoryCustom {

  private final JPAQueryFactory queryFactory;

  ...

}
```

## **Querydsl 테스트**

먼저 테스트 클래스에 `@Import(QuerydslTestConfig.class)` 로 Querydsl 설정 파일을 추가해줍니다.
<br/>리스트, 조회, 추가, 수정, 삭제 쿼리를 보면 SQL을 직접입력했던 형식과 크게 다른 부분은 없다. 다만, QClass로 컬럼 및 SELECT 결과를 반환한다.

### **1) 데이터 리스트 조회 테스트**

BoardRepositoryImpl.java 게시판 리스트 코드 작성
```java
@Override
public List<BoardResponseDto> getBoards() {
  QBoard board = new QBoard("board");
  return queryFactory
      .select(
          new QBoardResponseDto(
              board.id, board.title, board.contents,
              board.writeDate, board.modifyDate
          )
      )
      .from(board)
      .orderBy(board.id.asc())
      .fetch();
}
```

**테스트 코드**
```java
@Test
@DisplayName("Spring Data JPA + Querydsl 리스트 테스트")
public void testToQuerydslList(){
  List<BoardResponseDto> boards = boardRepository.getBoards();
  boards.stream().forEach(o -> System.out.println(o.toString()));
}
```

**테스트 결과**
![Querydsl 리스트 테스트]({{site.url}}/assets/img/Querydsl/2_Querydsl List Test.png)

### **2) 데이터 조회 테스트**

**BoardRepositoryImpl.java 게시판 조회 코드 작성**

```java
@Override
public BoardResponseDto getSchedule(Long id) {
  QBoard board = new QBoard("board");
  return queryFactory
      .select(
          new QBoardResponseDto(
              board.id, board.title, board.contents,
              board.writeDate, board.modifyDate
          )
      )
      .from(board)
      .where(board.id.eq(id))
      .fetchOne();
}
```

**테스트 코드**
```java
@Test
@DisplayName("Spring Data JPA + Querydsl 단건 조회 테스트")
public void testToQuerydslBoard(){
  BoardResponseDto board = boardRepository.getBoard((long)2);
  System.out.println(board.toString());
  assertEquals(board.getId(), 2);
}
```

**테스트 결과**
![Querydsl 조회 테스트]({{site.url}}/assets/img/Querydsl/3_Querydsl Select Test.png)

### **3) 데이터 추가 테스트**

**BoardRepositoryImpl.java 게시판 추가 코드 작성**
```java
@Override
public long insertBoard(BoardDto boardDto) {
  QBoard board = new QBoard("board");
  return queryFactory
      .insert(board)
      .columns(board.title, board.contents, board.writeDate)
      .values(boardDto.getTitle(), boardDto.getContents(), LocalDateTime.now())
      .execute();
}
```

**테스트 코드**
```java
@Test
@DisplayName("Spring Data JPA + Querydsl 추가 테스트")
@Transactional
@Rollback(value = false)
public void testToQuerydslBoardInsert(){
  BoardDto boardDto = new BoardDto();
  boardDto.setTitle("Querydsl 추가 테스트 제목");
  boardDto.setContents("Querydsl 추가 테스트 내용");
  long result = boardRepository.insertBoard(boardDto);
  assertEquals(result, 1);

  BoardResponseDto board = boardRepository.getBoard((long)2);
}
```

**테스트 결과**
![Querydsl 추가 테스트]({{site.url}}/assets/img/Querydsl/4_Querydsl Insert Test.png)
데이터 추가 테스트 코드에서 영속성 컨텍스트 1차 캐시를 이용한 동작에 대해 테스트를 위해 insert이후 select까지 해보았다. 하지만 동작하지 않는 것으로 보인다.
<br />Querydsl은 JPQL 빌더이지 JPA가 아니다. **JPAQueryFactory** 클래스를 보면 **JPQLQueryFactory** 인터페이스를 구현하고 있다. 그래서 Querydsl은 조회할 때 1차 캐시가 아닌 DB를 먼저 조회하게 된다. 이 후 조회한 결과를 바로 반환하는 것이 아니라, 영속성 컨텍스트에 넣으려고 하는데, 만약 이 결과가 영속성 컨텍스트에 있을 경우에는 DB로 부터 가져온 데이터를 버린다.
이 이유는 영속성 컨텍스트에 값과 DB에서 가져온 결과의 충돌로 인해 **DIRTY READ**{: .text-blue }가 발생할 수 있기 때문이며, 영속성 컨텍스트에 있는 값이 변경될 수 있어 DB의 결과를 버림으로써 **NON-REPEATABLE READ**{: .text-blue } 발생을 막기위함이다.

### **4) 데이터 수정 테스트**

**BoardRepositoryImpl.java 게시판 수정 코드 작성**
```java
@Override
public long updateBoard(BoardDto boardDto) {
  QBoard board = new QBoard("board");
  return queryFactory
      .update(board)
      .set(board.modifyDate, LocalDateTime.now())
      .set(board.title, boardDto.getTitle())
      .set(board.contents, boardDto.getContents())
      .where(board.id.eq(boardDto.getId()))
      .execute();
}
```

**테스트 코드**
```java
@Test
@DisplayName("Spring Data JPA + Querydsl 수정 테스트")
@Rollback(value = false)
public void testToQuerydslBoardUpdate(){
  BoardDto boardDto = new BoardDto();
  boardDto.setId((long)14);
  boardDto.setTitle("Querydsl 추가 테스트 제목 수정");
  boardDto.setContents("Querydsl 추가 테스트 내용 수정");
  long result = boardRepository.updateBoard(boardDto);
  assertEquals(result, 1);
}
```

**테스트 결과**
![Querydsl 수정 테스트]({{site.url}}/assets/img/Querydsl/5_Querydsl Update Test.png)

### **5) 데이터 삭제 테스트**

**BoardRepositoryImpl.java 게시판 삭제 코드 작성**
```java
@Override
public long deleteBoard(Long id) {
  QBoard board = new QBoard("board");
  return queryFactory
      .delete(board)
      .where(board.id.eq(id))
      .execute();
}
```

**테스트 코드**
```java
@Test
@DisplayName("Spring Data JPA + Querydsl 삭제 테스트")
@Rollback(value = false)
public void testToQuerydslBoardDelete(){
  long result = boardRepository.deleteBoard((long)14);
  assertEquals(result, 1);
}
```

**테스트 결과**
![Querydsl 삭제 테스트]({{site.url}}/assets/img/Querydsl/6_Querydsl Delete Test.png)

## **JPQL와 Querydsl를 사용하는 이유?**

Spring Data JPA를 사용하면 쿼리 작성 없이 엔티티 설계만 잘하면 메서드만으로 CRUD가 가능합니다. 편리한 장점이 있지만 필요한 컬럼만을 호출할 수 없으며, 요구하는 기능에 따라 한계가 있어 보였습니다. 개발을 하다보면 UNION, Subquery 등을 이용해야 하는 상황이 생깁니다. 하지만, Spring Data JPA만으로 이 복잡한 SQL을 생성하기에 많은 생각과 시간이 필요할 수 있습니다.

이러한 이슈를 JPQL, Querydsl를 사용해서 해결할 수 있습니다.

### **그러면 JPQL, Querydsl만 사용하면 안되나?**

앞에서 말한 것 처럼 JPQL과 Querydsl의 경우 복잡한 SQL을 처리할 수 있다. 하지만, 단순히 1건의 데이터를 추가하는 기능을 만들때, 애너테이션 선언, 쿼리 작성, 커스텀 리포지토리 생성 등 반복적인 작업이 생기며 코드는 늘어나게 됩니다.

이러한 경우에는 Spring Data JPA을 사용하여 메소드 호출을 하면 반복적인 작업과 코드를 줄일 수 있습니다.

이러한 차이점으로 구현해야하는 기능에 따라 적절한 선택이 필요해 보입니다.

### **JPQL과 Querydsl의 차이**

JPQL은 @Query 애너테이션을 이용해 Querydsl 보다 편리하게 SQL을 작성할 수 있습니다. 하지만, 실무에서는 JPQL보다 Querydsl을 더 많이 선호합니다. 그 이유는 타입 체크, 실행 전에 쿼리 오류를 바로 확인할 수 있다는 것 그리고 동적 쿼리 때문입니다.

만약, 리스트 페이지에서 검색 조건만 10개가 넘는 경우에 이 모든 조건을 JPQL로 동적쿼리를 작성하게 되면 너무 복잡해보이고 가독성이 떨어질 것 입니다. 이러한 이유로 Querydsl를 선호하는 것 같습니다. 게다가, 조건절을 분리해두면 필요에 따라 편리하게 사용할 수 있습니다.