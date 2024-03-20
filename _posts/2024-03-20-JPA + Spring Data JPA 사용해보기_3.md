---
title: JPA + Spring Data JPA 사용해보기(Join)
# author: dowonl2e
date: 2024-03-20 12:55:00 +0800
categories: [Spring, JPA]
tags: [JPA, Spring Data JPA]
pin: true
img_path: "/assets"
image:
  path: /commons/Spring-Data-JPA.png
  alt: Spring Data JPA
---

**JPA + Spring Data JPA 사용해보기(Dynamic Query, Paging, Sort)**에서 Spring Data JPA에서의 동적 쿼리, 페이징, 정렬에 대한 내용을 포스팅했습니다. 이번에는 Spring Data JPA의 Join에 대해 적어보겠습니다.

> [JPA + Spring Data JPA 사용해보기(Dynamic Query, Paging, Sort)](/posts/JPA-+-Spring-Data-JPA-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0_2/){:target="\_blank"}

# **Join**

Join 이란 DB에서 두 개 이상의 테이블을 연결하여 하나의 결과 테이블을 만드는 것을 뜻하며, 테이블을 분리하여 중복 데이터를 최소화하고 데이터의 일관성을 유지하기 위해 사용합니다. 

## **Inner Join**

N개의 테이블에서 공통된 값을 찾기 위해 사용합니다.

- **Self Inner Join** : 하나의 테이블을 이용해 필요한 다른 값을 찾기 위해 자기 자신과 조인하는 방식입니다.

## **Outer(Left/Right/Full) Join**

기준이 되는 테이블에서 공통된 값을 가지지 않는 레코드들도 반환합니다.

- **Left Outer Join** : 왼쪽 테이블을 기준으로 오른쪽 테이블에 공통된 값이 없더라도 해당 레코드를 반환합니다.
- **Right Outer Join** : 오른쪽 테이블을 기준으로 왼쪽 테이블에 공통된 값이 없더라도 해당 레코드를 반환합니다.
- **Full Outer Join** : 기준이 되는 테이블 구분 없이 모든 레코드들을 반환합니다. 공통된 값이 없는 테이블의 경우 해당 레코드는 Null로 반환됩니다.

## **Cross Inner Join**

N개 이상의 테이블에서 모든 가능한 조합을 만들어 결과를 반환합니다. 일반적으로 테이블 간의 관계가 없는 경우에 사용하며, 가능한 모든 조합을 만들어내기에 결과가 매우 클 수 있습니다.

## **Fetch Join**

보편적인 SQL에서 얘기하는 Join은 아니며, JPQL에서 성능 최적화를 위해 제공하는 Join의 종류입니다. `**JOIN FETCH**` 명령어를 통해 사용할 수 있습니다.

# **Spring Data JPA에서 Join 사용**

Spring Data JPA에서 Query Method를 이용해 테이블간 Join을 할 수 있습니다. Join을 하기 위해서는 먼저 Entity 설계 과정에서 관계를 설정해주어야 합니다.

## **OneToMany, ManyToOne 단방향 연관관계**

단방향 연관관계는 한 엔티티에서만 다른 엔티티의 연관관계를 맺는 것 입니다.

### **OneToMany (Member : Board) 단방향**

**Member.java**

```java
@Getter
@Entity(name = "tb_member")
@NoArgsConstructor
public class Member {

  ...

  @OneToMany(fetch = FetchType.EAGER)
  @JoinColumn(
    name = "writerId", referencedColumnName = "memberId",
    insertable = false, updatable = false,
  )
  private List<Board> boards = new ArrayList<>();

  ...

}
```

- @OneToMany : Join 확인을 위해 즉시로딩(EAGER)로 지정해두었습니다. (default : FetchType.LAZY)
- @JoinColumn
    - name : 타겟 엔티티(Board)의 필드를 지정해줍니다.
    - referencedColumnName : Board 엔티티와 연관관계를 맺을 Member 엔티티의 필드를 지정해줍니다. 필드를 별도로 지정하지 않는 경우 @Id로 지정된 필드로 조인됩니다.

> Board.java 파일에서는 ManyToOne 관계를 설정할 필요 없습니다.
{: .prompt-info}

**테스트 결과**

![Spring Data JPA OneToMany 단방향 결과]({{site.url}}/assets/img/JPA3/7_DIRECTION_ONE_ONETOMANY_JOIN.png)

### **ManyToOne (Board : Member) 단방향**

**Board.java**

```java
@Entity(name = "tb_board")
@Comment("게시판")
@Getter
@ToString
@NoArgsConstructor
public class Board {

  ...

  @ManyToOne
  @JoinColumn(
      name = "writerId", referencedColumnName = "memberId",
      insertable = false, updatable = false
  )
  private Member member;

  ...

}
```

- @ManyToOne : Join 확인을 위해 즉시로딩(EAGER)로 지정해두었습니다. (default : FetchType.EAGER)
- @JoinColumn
    - name : Member 엔티티와 연관관계를 맺을 Board 엔티티의 필드를 지정해줍니다.
    - referencedColumnName : 타겟 엔티티(Member)의 필드를 지정해줍니다. 필드를 별도로 지정하지 않는 경우 @Id로 지정된 필드로 조인됩니다.

> Member.java 파일에서는 OneToMany 관계를 설정할 필요 없습니다.
{: .prompt-info }

**테스트 결과**

![Spring Data JPA ManyToOne 단방향 결과]({{site.url}}/assets/img/JPA3/8_DIRECTION_ONE_MANYTOONE_JOIN.png)

## **OneToMany과 ManyToOne 양방향 연관관계**

OneToMany는 하나의 레코드가 서로 다른 여러 레코드와 연결된 관계를 의미합니다. 그 예로 사용자(게시판 작성자)와 게시판을 예로 관계 설정 및 Join 테스트를 해보겠습니다.

### **사용자와 게시판 - OneToMany**

아래의 사용자와 게시판 엔티티의 경우 게시판의 작성자를 사용자 아이디로 보겠습니다. 

**Member.java - 사용자**

```java
@Getter
@Entity(name = "tb_member")
@NoArgsConstructor
public class Member {

  @Id
  @Column(length = 30)
  @Comment("PK")
  private String memberId;

  @Column(length = 300)
  @Comment("사용자명")
  private String memberName;

  @OneToMany(mappedBy = "member", cascade = {CascadeType.PERSIST})
  private List<Board> boards = new ArrayList<>();

  ...

}
```

- @OneToMany : 1대 N 관계를 지정합니다.
    - mappedBy : 게시판 엔티티(Board)에서 **`Member 엔티티 선언 변수명`**을 지정해줍니다.
    - cascade : 데이터 추가, 삭제 등이 발생하면 boards에 있는 객체도 같이 추가 또는 삭제됩니다. **CascadeType.java** 파일에서 세부 정보를 확인할 수 있습니다.
- List<Boards> boards : 1대 N 관계이므로 게시판의 경우 리스트 타입으로 선언해줍니다.
    - 제네릭 없이 `**List boards**`로 사용 가능하며 대신 @OneToMany 애너테이션에 `**targetEntity = Board.class**` 속성을 통해 대상 엔티티를 명시해 주어야 합니다.

**Board.java - 게시판**

```java
@Entity(name = "tb_board")
@Comment("게시판")
@Getter
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

  @Comment("작성자ID")
  @Column(length = 30)
  private String writerId;

  ...

  @ManyToOne
  @OnDelete(action = OnDeleteAction.CASCADE)
  @JoinColumn(
      name = "writerId", referencedColumnName = "memberId",
      insertable = false, updatable = false,
      foreignKey = @ForeignKey(name = "board_member_id_fk", value = ConstraintMode.CONSTRAINT)
  )
  private Member member;

  ...

}
```

- @ManyToOne : 게시판의 기준으로는 사용자 엔티티와의 관계는 N대 1이므로 ManyToOne으로 설정해줍니다.
- @JoinColumn : 게시판, 사용자 엔티티에서 Join의 대상 컬럼을 지정합니다.
  - name : 해당 엔티티에서 조인 대상 필드명
  - referencedColumnName : 타겟 엔티티의 조인 대상 필드명
  - insertable : 엔티티 저장시 해당 필드를 저장할지에 대한 여부를 설정합니다. false로 설정하면 데이터베이스에 저장하지 않으며, 읽기 전용일때 사용한다. (default : true)
  - updatable : 엔티티 수정시 해당 필드를 저장할지에 대한 여부를 설정합니다. false로 설정하면 데이터베이스에 저장하지 않으며, 읽기 전용일때 사용한다. (default : true)
- Member member : 게시판 기준에서는 N대 1 관계이므로 Member 엔티티를 변수로 선언해줍니다.
- @OnDelete : 테이블에 삭제에 대한 Cascade를 설정할 수 있습니다. **OnDeleteAction.java** 파일에서 설정 값 별 정보를 확인 할 수 있습니다.
- foreignKey : 외래키 정보를 설정합니다.
  - @ForeignKey name : 제약조건 명을 설정할 수 있습니다.
  - @ForeignKey value : 제약조건을 설정할 수 있으며 **CONSTRAINT**은 물리적 제약조건을 설정해주며, **NO_CONSTRAINT**는 논리적으로만 설정하고 물리적으로 설정하지 않습니다. 즉, 외래키는 생성되지 않습니다. (default : ConstraintMode.CONSTRAINT)

> 위와 같이 양방향 연관관계를 설정하기 위해서 각 엔티티에 서로에 대한 관계를 설정해주어야합니다.
<br/>사용자(@OneToMany) / 게시판(@ManyToOne)
{: .prompt-info }

### **게시판과 사용자 - ManyToOne**

**테스트 코드**

```java
@Test
@DisplayName("Spring Data JPA ManyToOne 관계 확인")
@Rollback(value = false)
public void testToSpringDataJPABoardJoin(){
  Optional<Board> board = boardRepository.findById((long)1);
  assertEquals(board.get().getId(), 1);
}
```

**테스트 결과**

![Spring Data JPA 즉시로딩 Inner Join 단건 결과]({{site.url}}/assets/img/JPA3/1_BOARD_SINGLE_SELECT_EAGER_TRUE.png)

조인은 정상적으로 되었지만, 결과에서 left join이 되는 것을 볼 수 있습니다. 요구사항에 따라 다르겠지만 아래와 같은 경우도 있을 것 입니다.

1. 게시물 작성자가 실제 사용자인 게시물만 출력(작성자가 실제 사용자 테이블 있을 경우) → **Member와 Inner Join 필요**
2. 게시판 화면에 작성자 정보 불필요 → **Member와 Join이 필요가 없음**

### **1) 게시판과 사용자의 Inner Join**

**Board.java**

```java
...
public class Board {

  ...
  
  @ManyToOne(optional = false) //optional = false 추가
  @OnDelete(action = OnDeleteAction.CASCADE)
  @JoinColumn(
      name = "writerId", referencedColumnName = "memberId",
      insertable = false, updatable = false,
      foreignKey = @ForeignKey(name = "board_member_id_fk", value = ConstraintMode.CONSTRAINT)
  )
  private Member member;

  ...

}
```

**테스트 결과**

![Spring Data JPA 즉시로딩 Outer Join 단건 결과]({{site.url}}/assets/img/JPA3/2_BOARD_SINGLE_SELECT_EAGER_FALSE.png)

- optional : **false**로 지정하면 Non-Null 관계로 설정하는 것이며 Inner Join가 실행되게 됩니다. (default : true)

### **2) 게시판과 사용자의 Join이 필요 없는 경우**

이 경우에는 단방향 연관관계로 해결할 수 있지만, 양방향 연관관계를 유지해야할 때 fetch 옵션을 변경하여 Join을 피할 수 있습니다.

**Board.java**

```java
...
public class Board {

  ...
  
  @ManyToOne(fetch = FetchType.LAZY) //FetchType.LAZY 추가
  @OnDelete(action = OnDeleteAction.CASCADE)
  @JoinColumn(
      name = "writerId", referencedColumnName = "memberId",
      insertable = false, updatable = false,
      foreignKey = @ForeignKey(name = "board_member_id_fk", value = ConstraintMode.CONSTRAINT)
  )
  private Member member;

  ...

}
```

**테스트 결과**

![Spring Data JPA 지연로딩 Outer Join 단건 결과]({{site.url}}/assets/img/JPA3/3_BOARD_SINGLE_SELECT_LAZY.png)

## **즉시로딩과 지연로딩**

FetchType에서 EAGER는 즉시로딩을 의미합니다. 반대로, LAZY는 지연로딩을 의미합니다. 두 의미와 테스트 결과를 보면 알 수 있듯이 EAGER는 연관관계가 있는 엔티티를 즉시로딩 하고, LAZY는 연관관계가 있더라도 **필요에 따라**{: .text-blue} 로딩 할지를 결정합니다.

여기서, **'필요에 따라’**{: .text-blue}의 의미는 위 예제로 Member가 필요한 순간(Member에 있는 필드가 사용되었을 때)에 로딩합니다.

```java
@Test
@DisplayName("Spring Data JPA 게시판 Member Join 1건 조회 테스트")
@Rollback(value = false)
public void testToSpringDataJPABoardJoin(){
  Optional<Board> board = boardRepository.findByIdAndMemberMemberName((long)1, "멤버1");
  assertEquals(board.get().getId(), 1);
}
```

![Spring Data JPA 지연로딩 Outer Join 리스트 결과]({{site.url}}/assets/img/JPA3/4_BOARD_SINGLE_LAZY_JOIN.png)

이 경우 findByIdAndMemberMemberName 메서드에서 Board의 연관관계의 Member에 MemberName을 조건절에 추가하겠다는 의미로 Member를 사용하므로서 Join 되는 것을 확인하실 수 있습니다.

Query Method 뿐만 아니라 Specification을 이용해도 동일한 결과를 얻을 수 있습니다.

> Left Join이 아니라 Inner Join이 필요한 경우에 **optional = false**{: .text-blue } 속성을 추가하면 됩니다.
{: .prompt-info }

### **즉시로딩에서의 이슈**

즉시로딩과 지연로딩을 생각해보면 대부분 Join 필요한 경우 즉시로딩, 그렇지 않으면 지연로딩을 사용하면 되겠다는 생각이 들 수 있습니다. 직접 실무에서 사용해보지는 않았지만, 이번 Join에 대해 찾아보고 공부했던 바로는 즉시로딩의 경우 특정 비즈니스 로직에서 필요한 연관 객체가 한번에 로딩되어 편리하다는 이점이 있지만, 아래의 이슈로 인해 사용하지 않는 것을 권장합니다.

위의 게시판(Board), 작성자(Member)를 예로

1. Board 데이터만 필요할 때 연관된 Member를 로딩하여 불필요한 데이터 조회
2. OneToMany의 경우 Member 1건의 데이터를 조회할 경우 연관된 Board도 로딩하여 N건의 데이터가 조회되는 현상으로 예상치 못한 쿼리 및 결과 발생
3. Board 데이터 N개를 조회할 경우 N+1의 DB 접근의 문제 발생
    ![Spring Data JPA 즉시로딩 리스트 결과]({{site.url}}/assets/img/JPA3/5_BOARD_LIST_EAGER1.png)
    ![Spring Data JPA 즉시로딩 리스트 결과]({{site.url}}/assets/img/JPA3/6_BOARD_LIST_EAGER2.png)
    게시판 데이터 조회 DB 접근 1회 + 게시판 레코드 별 사용자(작성자) 데이터 조회 DB 접근 N회 발생

# **정리**

- OneToMany, ManyToOne, ManyToMany를 통해 엔티티의 연관관계를 설정할 수 있습니다.
- 설정된 연관관계로 생성되는 쿼리의 결과는, **`optional = false`**{: .text-blue }을 통해  Inner Join으로 사용 가능합니다.
- 즉시로딩은 연관 관계의 모든 객체가 로딩되고, 지연로딩은 필요에 따라 연관 관계의 객체를 로딩합니다.
- 즉시로딩의 경우 비즈니스 로직에서 항상 같이 조회되어야하는 경우에는 고려해보는 것도 좋지만, 불필요한 객체 로딩과 DB 엑세스 등의 문제가 있을 수 있어 지연로딩을 권장합니다.