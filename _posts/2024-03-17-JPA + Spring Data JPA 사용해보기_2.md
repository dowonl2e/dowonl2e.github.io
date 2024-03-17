---
title: JPA + Spring Data JPA 사용해보기(2)
# author: dowonl2e
date: 2024-03-17 15:00:00 +0800
categories: [Spring, JPA]
tags: [JPA, Spring Data JPA]
pin: true
img_path: "/assets"
image:
  path: /commons/Spring-Data-JPA.png
  alt: Spring Data JPA
---

**JPA + Spring Data JPA 사용해보기(1)**에서 기본적인 CRUD에 대한 내용을 포스팅했습니다. 이번에는 Spring Data JPA Query Method의 조건절 및 동적 쿼리, 페이징 및 정렬을 대해 적어보겠습니다.

> [JPA + Spring Data JPA 사용해보기(1)](/posts/JPA-+-Spring-Data-JPA-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0_1/){:target="\_blank"}

# **Query Method**

Spring Data JPA은 SQL을 작성하지 않고 메소드만으로 기능을 구현할 수 있습니다. 이 메서드 명으로 SQL을 생성하는 것을 Query Method라고 합니다. Query Method의 이름은 Subject Part, Predicate Part로 나뉩니다.

- Subject Part : 메서드 명에서 첫 번째 By까지의 부분
    - find…By, exists…By, delete…By, count..By 등
- Predicate Part : 첫 번째 By 이후에 나오는 부분
    - And, Or, Equals, GreaterThan, IsNull 등

두 부분에 대한 전체 키워드는 아래 Spring Data JPA 공식 문서를 통해서 확인하실 수 있습니다.

> [Spring Data JPA Repository query keywords](https://docs.spring.io/spring-data/jpa/reference/repositories/query-keywords-reference.html){:target="\_blank"}

# **조건절**

## **비교 연산자 ( = )**

**BoardRepository.java**

```java
@Repository
public interface BoardRepository extends JpaRepository<Board, Long> {

  List<Board> findByTitle(String title);

}
```

**테스트 코드**

```java
@Test
@DisplayName("Spring Data JPA 조건절 테스트")
public void testToSpringDataJpaBoards(){
  List<Board> boards = boardRepository.findByTitle("영화 게시판 제목1");
  boards.stream().forEach(board -> System.out.println(board.getId() + ", " + board.getTitle() + ", " + board.getContents()));
}
```

**테스트 결과**

![Spring Data JPA 조건절 비교 연산자]({{site.url}}/assets/img/JPA2/1_Spring Data JPA Where Equals.png)

단순히 Repository에 findByTitle로 메소드를 만들기만 했는데 아래와 같이 title 컬럼에 대한 조건절이 추가됩니다. 그렇다면, BoardRepository 파일에 findByTitle를 findByText로 변경하면 어떻게 될까?

> No property 'text' found for type 'Board’

위 오류가 확인됩니다. Board 엔티티에 text 필드를 찾지못했다는 의미로 메서드명에서 findBy(Subject Part) 다음에 설정된 Text(Predicate Part)는 엔티티의 프로퍼티여야 하며 이 프로퍼티로 Equals 검색을 하게됩니다.

## **AND, OR을 이용한 조건절 추가**

**BoardRepository.java 메소드 추가**

```java
List<Board> findByTitleOrContents(String title, String contents);

List<Board> findByTitleAndContents(String title, String contents);
```

추가적인 조건 적용 방법은 findBy 이후에 Predicate Part에 표기할 프로퍼티를 And 또는 Or로 분기하면 사용이 가능합니다. 다만, And, Or를 분기해서 추가한 프로퍼티 만큼 파라미터를 추가해줘야합니다.

## **그렇지만?**

동일한 화면을 제공하는 경우 메소드 재사용으로 코드는 줄일 수 있습니다. 그렇지만, 검색 대상이 많을 경우의 그 만큼 조회 메소드를 따로 만들어줘야하며 메소드명이 길어지면 가독성이 떨어지고 메소드를 찾는데 어려움이 있어 **유지보수**가 어려워집니다.

### **동적 쿼리를 위해 메소드를 계속 만들어야하는 상황**

```java
//1. 날짜 검색
if(date != null){
  return repository.findByWriteDateAfter(date);
}
//2. 날짜 + 제목 검색
else if(date != null && title != null){
  return repository.findByWriteDateAfterAndTitleLike(date, title);
}
//3. 날짜 + 제목 + 내용 검색
else if(date != null && title != null && contents != null){
  return repository.findByWriteDateAfterAndTitleLikeAndContentsLike(date, title, contents);
}
...
```

# **JPA Specification를 이용한 동적 쿼리**

Spring Data JPA에서 동적 쿼리는 JPA Specification을 이용해 처리할 수 있습니다. JPA Specification은 Criteria API를 기반으로 만들어졌으며 JPA Specification을 보기 전에 Criteria API를 먼저 보겠습니다.

## **Criteria API**

Criteria API는 JPQL 작성을 도와주는 빌더이며, 동적 쿼리를 사용하기 위한 JPA 라이브러리입니다. 컴파일 시점에 쿼리 문법을 확인할 수 있으며,  **`type-safety`**{: .text-blue }하게 쿼리를 작성할 수 있습니다.

```java
EntityManager entityManager = entityManagerFactory.createEntityManager();
CriteriaBuilder criteriaBuilder = entityManager.getCriteriaBuilder();

CriteriaQuery<Board> criteriaQuery = criteriaBuilder.createQuery(Board.class);

Root<Board> root = criteriaQuery.from(Board.class);

String title = "영화 게시판 제목1";
LocalDateTime now = null;
Predicate predicates = null, predicates2 = null;
if(title != null) {
  predicates = criteriaBuilder.like(root.get("title"), '%'+title+'%');
}
if(now != null){
  predicates2 = criteriaBuilder.lessThan(root.get("writeDate"), now);
}
criteriaQuery.where(predicates, predicates2);

TypedQuery<Board> boardsQuery = entityManager.createQuery(criteriaQuery);

List<Board> boards = boardsQuery.getResultList();
boards.stream().forEach(board -> System.out.println(board.getId() + ", " + board.getTitle() + ", " + board.getContents()));
```

확실히 메소드만으로 동적 쿼리 구현이 어려운 문제를 해결할 수 있습니다. 하지만, Criteria API를 사용했을 때에도 조건절이 많을 경우 Predicate 객체를 계속 생성해야한다는 점에서 코드가 복잡해지고 가독성이 떨어지는 단점이 있습니다.

## **JPA Specification**

JPA Specification은 Criteria API를 기반으로 만들어졌으며, Criteria API에서의 복잡한 코드 단점을 보완할 수 있습니다.

### **장점**

JPA Specification의 장점으로 동적 쿼리 생성 코드 캡슐화 및 중복방지를 통해 **`코드 재사용성`**이 높아집니다. 이러한 이점으로 자연스럽게 **`유지보수가 용이`**해집니다. 그리고, SQL 인젝션 같은 보안 문제로 부터 보호할 수 있는 **`안전한 SQL을 생성`**해줍니다.

**BoardRepository.java**

```java
@Repository
public interface BoardRepository extends JpaRepository<Board, Long>, JpaSpecificationExecutor<Board> {

}
```

- JpaSpecificationExecutor : Specification을 허용하기 위한 인터페이스이며 Specification 사용을 위해 Repository에 상속해줍니다.

### **동적 쿼리 Specification 생성**

**BoardSpecification.java**

```java
public class BoardSpecification {

  public static Specification<Board> alwaysTrue(){
    return (root, query, criteriaBuilder) -> criteriaBuilder.equal(criteriaBuilder.literal(1), 1);
  }

  public static Specification<Board> likeBoardTitle(String keyword){
    return (root, query, criteriaBuilder) -> criteriaBuilder.like(root.get("title"), '%' + keyword + '%');
  }

  public static Specification<Board> lessThanBoardWriteDate(LocalDateTime date){
    return (root, query, criteriaBuilder) -> criteriaBuilder.lessThan(root.get("writeDate"), date);
  }
}
```

### **Specification 테스트**

**테스트 코드**

```java
@Test
@DisplayName("JPA Specification 동적 쿼리 테스트")
public void testToSpecificationBoards(){
  String title = "영화 게시판 제목1";
  LocalDateTime now = null;

  Specification<Board> spec = Specification.where(BoardSpecification.alwaysTrue());
  spec = title == null ? spec : spec.and(BoardSpecification.likeBoardTitle(title));
  spec = now == null ? spec : spec.and(BoardSpecification.lessThanBoardWriteDate(now));
  List<Board> boards = boardRepository.findAll(spec);
  boards.stream().forEach(board -> System.out.println(board.getId() + ", " + board.getTitle() + ", " + board.getContents()));
}
```

**테스트 결과**

![Spring Data JPA Specification 동적 쿼리]({{site.url}}/assets/img/JPA2/2_JPA Specification DQ1.png)

위와 같이 JPA Specification을 이용하면 메소드명을 이용한 조건절 처리의 문제, Criteria API를 사용할 때 생기는 단점을 보완할 수 있습니다.

# **페이징 및 정렬 처리**

### **Pageable을 이용한 페이징 및 정렬**

리스트 페이징을 위해서 Pageable 인터페이스를 이용할 수 있으며 Pageable의 구현체인 AbstractPageRequest의 하위 클래스 PageRequest를 통해서 페이징 처리에 페이지 번호, 페이지 사이즈, 정렬 등을 설정해 이용할 수 있습니다.

**테스트 코드**

```java
@Test
@DisplayName("Spring Data JPA 리스트 페이징")
@Rollback(value = false)
public void testToSpringDataJPAPaging(){
  Pageable pageable = PageRequest.of(0, 5, Sort.Direction.DESC, "id");
  Page<Board> boards = boardRepository.findAll(pageable);
  boards.stream().forEach(board -> System.out.println(board.getId() + ", " + board.getTitle() + ", " + board.getContents()));
}
```

- PageRequest.of(…)
  - 0 : 페이지 번호
  - 5 : 페이지 사이즈(레코드 수)
  - Sort.Direction.DESC : 내림차순 정렬
  - “id” : 정렬 컬럼
- 정렬 대상 컬럼이 여러개이며 각 컬럼별 정렬 방식이 다른 경우 아래와 같이 Sort를 따로 생성하여 사용하실 수 있습니다.
  ```java
  Sort sort = Sort.by(Sort.Order.desc("title"), Sort.Order.asc("id"));
  Pageable pageable = PageRequest.of(0, 5, sort);
  ```

**테스트 결과**

![Spring Data JPA Paging 결과]({{site.url}}/assets/img/JPA2/3_Paging Result1.png)

## **동적 쿼리 + 페이징 + 정렬 처리**

검색 환경에서 페이징 및 정렬처리는 동적 쿼리 처리에 사용한 JpaSpecificationExecutor 인터페이스를 통해 처리할 수 있으며, 아래 메서드에서 Specification 및 Pageable을 파라미터로 넘기면 됩니다.

```java
Page<T> findAll(Specification<T> spec, Pageable pageable);
```

**테스트 코드**

```java
@Test
@DisplayName("Spring Data JPA 리스트 페이징")
@Rollback(value = false)
public void testToSpringDataJPAPaging(){
  Sort sort = Sort.by(Sort.Order.desc("title"), Sort.Order.asc("id"));
  Pageable pageable = PageRequest.of(0, 5, sort);

  String title = "제목";
  LocalDateTime now = null;

  Specification<Board> spec = Specification.where(BoardSpecification.alwaysTrue());
  spec = title == null ? spec : spec.and(BoardSpecification.likeBoardTitle(title));
  spec = now == null ? spec : spec.and(BoardSpecification.lessThanBoardWriteDate(now));

  Page<Board> boards = boardRepository.findAll(spec, pageable);
  boards.stream().forEach(board -> System.out.println(board.getId() + ", " + board.getTitle() + ", " + board.getContents()));
}
```

**테스트 결과**

![Spring Data JPA Dynamic Query + Paging 결과1]({{site.url}}/assets/img/JPA2/4_DQ Paging Result1.png)

이 때 반환받는 타입이 Page이며  인터페이스에서 페이지 화면 구성에 필요한 값들을 가져올 수 있다.

![Spring Data JPA Dynamic Query + Paging 결과2]({{site.url}}/assets/img/JPA2/5_DQ Paging Result2.png)

- getTotalElements : 페이징과 관계없이 모든 데이터 수
- getTotalPages : 페이지 레코드 사이즈 기준 전체 페이지 수
- getNumber : 현재 페이지 번호
- getSize : 페이지 레코드 수
- hasPrevious : 이전 페이지 여부
- hasNext : 다음 페이지 여부