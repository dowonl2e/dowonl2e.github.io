---
title: JPA + Spring Data JPA 사용해보기
# author: dowonl2e
date: 2024-02-26 14:00:00 +0800
categories: [Spring, JPA]
tags: [JPA, Spring Data JPA]
pin: true
img_path: "/assets"
image:
  path: /commons/Spring-Data-JPA.png
  alt: Spring Data JPA
---

## **개발환경 및 프로젝트 세팅**

개발 환경 및 프로젝트 세팅은 JPA + Hibernate와 동일하게 이용하겠습니다.

> [JPA + Hibernate 사용해보기]({{site.url}}/posts/JPA-+-Hibernate-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0){:target="\_blank"}

## **신규 테스트 클래스 생성**

**JpaSpringDataJpaApplicationTests.java**

테스트용 설정 파일(application-test.yml)을 따로 생성하여 설정해주는 것이 좋으나 프로젝트 자체가 테스트용이기 때문에 기존에 있는 설정 파일을 사용하도록 하겠습니다.

```java
@DataJpaTest
@TestPropertySource(locations = "classpath:application.yml")
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
public class JpaSpringDataJpaApplicationTests {

}
```
- @DataJpaTest : JPA Components만 테스트하기위한 애너테이션이다.
- @AutoConfigureTestDatabase : @DataJpaTest를 사용하게 되면 자동으로 내장형 DB를 사용하게 되어 기존에 application.yml에 설정한 값들을 사용이 불가능하다.
    - AutoConfigureTestDatabase.Replace.NONE : 자동 구성된 디비를 사용하지 않고 설정한 파일의 속성을 이용한다.

## **Spring Data JPA 인터페이스 생성**

**BoardRepository.java**

```java
@Repository
public interface BoardRepository extends JpaRepository<Board, Long> {

}
```

## **Spring Data JPA 테스트 해보기**
### **BoardRepository 주입**
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
public class JpaSpringDataJpaApplicationTests {

  @Autowired
  private BoardRepository boardRepository;

  //테스트 코드 작성

}
```

### **1) 데이터 추가**

**테스트 코드 작성**
```java
@Test
@DisplayName("JPA Spring Data JPA 추가 테스트")
@Rollback(value = false)
public void testToSpringDataJpaInsert(){
  Board board = Board.builder()
        .id((long)1)
        .title("JPA Hibernate 테스트 제목1")
        .contents("JPA Hibernate 테스트 내용1")
        .build();
  long id = boardRepository.save(board).getId();
  assertEquals(id, board.getId());
}
```
- @Rollback : 테스트시에는 자동 롤백이 되므로 롤백이 되지 않도록 false로 처리해준다.

**테스트 결과**
![Spring Data JPA 데이터 추가 테스트1]({{site.url}}/assets/img/JPA/11_Spring Data JPA Insert Test1.png)
결과를 보면 select 이후에 insert를 하게 된다. 이 이유는 id 필드와 연관이 있다.<br/>아래 SimpleJpaRepository.java 파일에 save 메서드를 보면

```java
@Transactional
@Override
public <S extends T> S save(S entity) {

  Assert.notNull(entity, "Entity must not be null");

  if (entityInformation.isNew(entity)) {
    entityManager.persist(entity);
    return entity;
  } else {
    return entityManager.merge(entity);
  }
  
}
```

**isNew**{: .text-blue } 메서드의 경우 엔티티의 id값 여부에 따라 영속할지 아니면 병합할지를 결정한다. id 값이 null인 경우 조회할 데이터를 찾을 필요가 없기에 영속상태로 만들어 insert 하게되며, id 값이 있을 경우 merge 과정에서 DB에서 데이터 조회 후 영속상태로 만들고 데이터 여부에 따라 insert할지 update할지 결정한다.

테스트 코드에 **id((long)1)**{: .text-blue } 코드를 지우고 테스트할 경우 insert만 실행되는 것을 볼 수 있다.
![Spring Data JPA 데이터 추가 테스트2]({{site.url}}/assets/img/JPA/12_Spring Data JPA Insert Test2.png)

### **2) 데이터 조회**
**테스트 코드 작성**
```java
@Test
@DisplayName("JPA Spring Data JPA 조회 테스트")
public void testToSpringDataJpaSelect(){
  Optional<Board> board = boardRepository.findById((long)1);
  System.out.println(board.toString());
  assertEquals(board.get().getId(), 1);
}
```

**테스트 결과**
![Spring Data JPA 데이터 조회 테스트]({{site.url}}/assets/img/JPA/13_Spring Data JPA Select Test.png)

### **3) 데이터 삭제**
**테스트 코드 작성**
```java
@Test
@DisplayName("JPA Spring Data JPA 삭제 테스트")
@Rollback(value = false)
public void testToSpringDataJpaDelete(){
  boardRepository.deleteById((long)2);
}
```

**테스트 결과**
![Spring Data JPA 데이터 삭제 테스트]({{site.url}}/assets/img/JPA/14_Spring Data JPA Delete Test.png)

### 4)  데이터 수정
JPA는 수정 메서드가 따로 존재하지 않는다. JPQL로 쿼리를 따로 작성할 수 있지만, **변경감지(Dirty Checking)**{: .text-blue }와 **병합(Merge)**{: .text-blue } 2가지 방법으로 데이터 수정이 가능하다.
<br/>수정을 보기전에 **영속 상태, 준영속 상태**{: .text-blue }를 얘기하자면

- **영속상태**{: .text-blue }는 엔티티가 영속성 컨텍스트에 저장되어있고 JPA가 관리를 할 수 있는 상태
- **준영속상태**{: .text-blue }는 영속 상태에 있다가 분리되어 JPA가 관리할 수 없는 상태

트랜잭션이 시작되면 조회한 데이터는 영속상태에 들어가며 트랜잭션이 종료되면 해당 엔티티는 준영속 상태가 된다.

**(1) 변경감지(Dirty Checking)을 이용한 수정 테스트**

**Board.java**

```java
@Entity(name = "tb_board")
@Comment("게시판")
@Getter
@ToString
@NoArgsConstructor
public class Board {
	
	...
	
	//title, contents, modifyDate 필드를 수정하는 메서드 추가
  public void updateTitleContents(String title, String contents){
    this.title = title;
    this.contents = contents;
    this.modifyDate = LocalDateTime.now();
  }
}
```

**테스트 코드 작성**

```java
@Test
@DisplayName("JPA Spring Data JPA 수정 테스트")
@Rollback(value = false)
public void testToSpringDataJpaUpdate(){
  Board board = boardRepository.findById((long)1).get();
  board.updateTitleContents(
      "JPA Hibernate 테스트 제목 수정1",
      "JPA Hibernate 테스트 내용 수정1"
  );
}
```

엔티티 필드만 변경하는 메서드만 호출했다. 이 이유는 엔티티의 **영속상태**와 연관이 있다.

**SimpleJpaRepository.java** 파일에 **findById** 메서드를 찾아가게되면 아래와 같이 코드가 작성되어있다.

```java
@Override
public Optional<T> findById(ID id) {

  Assert.notNull(id, ID_MUST_NOT_BE_NULL);

  Class<T> domainType = getDomainClass();

  if (metadata == null) {
    return Optional.ofNullable(entityManager.find(domainType, id));
  }

  LockModeType type = metadata.getLockModeType();
  Map<String, Object> hints = getHints();

  return Optional.ofNullable(type == null ? entityManager.find(domainType, id, hints) : entityManager.find(domainType, id, type, hints));
}
```

마지막 줄에 엔티티 매니저가 find 메서드 이용해 엔티티를 반환하는 것을 볼 수 있다. 엔티티 매니저를 통해 조회한 엔티티는 **영속상태**{: .text-blue }가 된다.

테스트 코드를 보면 단순히 **findById** 메서드 호출 후 해당 엔티티의 필드 값을 변경하는 메서드만 호출했음에도 update가 되었다. 이유는 엔티티는 영속상태가 되고 무엇이 변경되었는지를 감지하여 자동으로 update 한다. 이것이 **변경감지(Dirty Checking)를 통한 데이터 수정**{: .text-blue }이다.

**테스트 결과**
![Spring Data JPA 데이터 수정 테스트1]({{site.url}}/assets/img/JPA/15_Spring Data JPA Update Test1.png)

**(2) 병합(merge)을 이용한 수정 테스트**

**1) 데이터 추가 테스트**에서 save 메서드는 id 값에 따라 영속 또는 병합 과정이 실행된다. 병합(merge)을 통한 수정은 이와 동일하다.

**테스트 코드**

```java
@Test
@DisplayName("JPA Spring Data JPA 수정(merge) 테스트")
@Rollback(value = false)
public void testToSpringDataJpaMergeUpdate(){
  Board board = Board.builder()
      .id((long)1)
      .title("JPA Hibernate 테스트 제목1 수정")
      .build();
  long id = boardRepository.save(board).getId();
  assertEquals(id, board.getId());
}
```

**테스트 결과**
![Spring Data JPA 데이터 수정 테스트2]({{site.url}}/assets/img/JPA/16_Spring Data JPA Update Test2.png)

![Spring Data JPA 데이터 수정 테스트2 데이터]({{site.url}}/assets/img/JPA/17_Spring Data JPA Update Test2 Data.png)

위 과정이 병합을 이용한 수정 방법이다. 하지만, 테스트 코드를 보면 수정하고자 했던 필드는 title이다. 그런데 실행되는 쿼리를 보면 원치 않는 필드까지 모두 업데이트 되었고, 결과 데이터에 contents, modify_date 필드가 null로 되어있다. 이유는 병합은 모든 필드를 교체하므로 값이 없는 필드의 경우 null로 업데이트 된다.

개발을 하다보면 특정 필드가 null이라도 기존 값을 유지해야하는 경우가 있기에 병합(merge)를 사용하는 것을 권장하지 않는다고 한다.

이 이슈는 엔티티 클래스에 **`@DynamicUpdate`** 애너테이션을 이용해 변경되는 필드만 업데이트할 수 있지만, 반대로 특정 값을 null로 업데이트해야하는 경우도 있기에 애매하다는 생각이 든다.

결론은, 데이터 수정시에는 병합(merge)보다 변경감지(Dirty Checking)을 사용하는 것을 권장한다.