---
title: Lock, Transaction Isolation Level, JPA 낙관적/비관적 락
# author: dowonl2e
date: 2024-05-02 14:30:00 +0800
categories: [Transaction, Lock]
tags: [Transaction, Lock, JPA]
pin: true
---

> [[Spring Boot + JPA] JPA 트랜잭션](/posts/Spring-Boot-+-JPA-Transaction/){:target="\_blank"}

위 글에서 JPA 트랜잭션에 대해 간단하게 정리했었다. 이번에는 세부적인 내용을 이해하기 위해 **`락(Lock)`{: .text-blue}**, **`트랜잭션 격리 수준(Transaction Isolatin Level)`{: .text-blue}**, JPA에서의 **`낙관적 락(Optimistic Lock), 비관적 락(Pessimistic Lock)`{: .text-blue}**에 대한 내용을 적어봅니다.

## **Lock**

Lock은 DB에서 여러 트랜잭션 간에 데이터의 일관성을 유지하기 위해 필요한 매커니즘이다. 동시에 같은 데이터를 읽거나 수정할 때 발생할 수 있는 문제를 방지하기 위해 사용한다. 

### **공유 락(Shared Lock)**

- 여러 트랜잭션이 데이터를 읽을 때 사용된다.
- 공유 락을 획득한 트랜잭션은 데이터를 읽을 수 있지만 수정은 불가능하다.
- 배타적 락(Exclusive Lock)이 없는 경우 사용 가능하다.

### **배타적 락(Exclusive Lock)**

- 특정 데이터를 수정할 때 사용된다.
- 배타적 락을 획득한 트랜잭션은 해당 데이터에 읽기/쓰기 권한을 가진다.
- 다른 트랜잭션이 동시에 해당 데이터에 배타적 락을 요청할 경우, 락이 풀릴 때까지 트랜잭션은 대기 상태에 들어간다.

### **행 락(Row Lock)**

- TABLE의 특정 행에 대한 락을 나타낸다.
- 여러 트랜잭션이 동시에 같은 테이블에 접근할 때 발생하는 경합 조건을 방지하기 위해 사용된다.

### **테이블 락(Table Lock)**

- 특정 테이블에 대한 전체 락을 나타낸다.
- 특정 테이블에 대한 모든 작업을 차단하고, 다른 트랜잭션이 해당 테이블에 접근하지 못하도록 한다.
- 일반적으로는 배타적 락(Exclusive Lock)을 사용하며, 동시에 여러 트랜잭션이 작업할 수 없다.

### **데드락(Deadlock)**

- 두 개 이상의 트랜잭션이 각자의 락을 획득하고, 서로 상대방의 락을 기다릴 때 발생한다.
- 서로의 락을 기다리는 상황에서 각 트랜잭션이 상대방이 소유한 락을 대기하며, 둘 다 락을 획득하지 못한 채 영원히 대기할 수 있다.

## **트랜잭션 격리 수준(Transaction Isolation Level)**

트랜잭션 격리 수준은 여러 트랜잭션이 동시에 실행될 때 서로의 간섭을 최소화 하기 위해 제공하는 격리 수준이다. 트랜잭션 특성이 ACID(Atomicity, Consistency, Isolation, Durability) 중 Isolation에 해당되며, 각 트랜잭션이 다른 트랜잭션에 영향을 미치지 않고 독립적으로 실행되도록 보장하는 것을 말한다.

### **격리 수준에 따른 일관성 문제**

#### **Dirty Read**

특정 트랜잭션의 작업이 커밋되지 않은 상태에서 다른 트랜잭션이 작업한 데이터를 읽는 현상을 말한다. 

특정 데이터를 수정했으나 커밋이 되지 않은 상황에서 다른 트랜잭션이 해당 데이터를 읽을 때 수정된 데이터를 읽는 것이다.

![Dirty Read]({{site.url}}/assets/img/Transaction-JPA-Lock/1-Dirty_Read.png)

트랜잭션이 데이터를 수정한 후 롤백되어 변경 내용이 취소되었지만, 다른 트랜잭션은 변경된 내용을 계속 가지게되어 데이터의 일관성을 깨뜨리는 문제를 발생시킬 수 있다.

#### **Non-repeatable Read**

한 트랜잭션에서 동일한 데이터를 연속적으로 읽을 때, 첫 번째 데이터를 읽고 두 번째 데이터를 읽기 전 다른 트랜잭션에 의해 해당 데이터의 변경 작업이 발생해 두 데이터는 서로 다른 상황을 말한다.

![Non-repeatable_Read]({{site.url}}/assets/img/Transaction-JPA-Lock/2-Non-repeatable_Read.png)

이 경우 서로 다른 값이기에 데이터 일관성이 깨지게 된다.

#### **Phantom Read**

한 트랜잭션에서 N개의 데이터를 조회하는 쿼리를 여러번 실행할 때, 첫 번째 쿼리를 실행하고 두 번째 쿼리를 실행하기 전 다른 트랜잭션에 의해 데이터가 추가/삭제가 발생할 수 있다. 이 때 첫 번째 쿼리 결과의 집합과 두 번째 쿼리 결과의 집합이 다른 경우를 의미한다.

![Phantom Read]({{site.url}}/assets/img/Transaction-JPA-Lock/3-Phantom_Read.png)

이는 트랜잭션이 같은 범위를 조회하는 동안에도 데이터의 일관성이 보장되지 않는 것을 의미한다.

### **격리 수준 종류**

#### **SERIALIZABLE**

가장 높은 격리 수준으로 모든 트랜잭션을 순차적으로 실행한다. 순차적인 실행으로 대상 레코드에 대해 여러 트랜잭션이 동시에 접근할 수 없어 가장 안전하지만 동시 처리 성능이 매우 떨어진다.

단순 SELECT 작업에서도 대상 레코드에 넥스트 키 락을 공유 락(Shared Lock)으로 걸기에 **추가/수정/삭제**가 불가능하다.

**`Dirty Read`{: .text-blue}**, **`Non-repeatable Read`{: .text-blue}**, **`Phantom Read`{: .text-blue}** 등의 문제가 발생하지 않지만, 동시성이 가장 낮다.

#### **REPEATABLE READ**

한 번 읽은 데이터를 트랜잭션이 종료될 때 까지 계속해서 동일한 값을 읽을 수 있다.

RDBMS에서 변경 전 레코드를 백업 하기에 **변경 전/후의 데이터**가 존재한다. 동일한 레코드에 여러 버전의 데이터가 존재하여 **`MVCC(Multi-Version Concurrency Control, 다중 버전 동시성 제어)`{: .text-blue}**라고 한다.

이를 통해 서로 다른 트랜잭션 간에 접근할 수 있는 데이터 제어가 가능하다. 각각의 트랜잭션은 순차적으로 증가하는 **고유 번호가 있어 백업 레코드에는 어느 트랜잭션에 의해 백업되었는지 고유 번호를 함께 저장된다.**

#### **REPEATABLE READ와 Phantom Read**

MySQL의 경우 기본 격리 수준은 **REPEATABLE READ**이다. **REPEATABLE READ**는 레코드 추가에 대해서는 막지 않기에 **`Phantom Read`{: .text-blue}** 문제가 발생할 수 있지만, **`MVCC`{: .text-blue}**로 인해서 발생하지 않는다.

A 트랜잭션(5), B 트랜잭션(6)에서 5번이 먼저 실행되면 **REPEATABLE READ**는 트랜잭션 아이디를 참고하여 5번보다 먼저 실행되는 트랜잭션의 데이터만 조회한다. 만약, 테이블에 자신보다 먼저 실행되는 트랜잭션의 데이터가 존재하면 Undo 로그를 참고해 데이터를 조회한다.

**`Phantom Read`{: .text-blue}**의 예시를 통해 **REPEATABLE READ**를 보면 아래와 같다.

![Phantom Read Solve1]({{site.url}}/assets/img/Transaction-JPA-Lock/4-Phantom_Read_Solve1.png)

그렇다면, 어떠한 상황에서 Phantom Read가 발생할까? 락(Lock)이 사용되는 경우이다. 

- **베타적 락(Exclusive Lock) 구문** : SELECT FOR UPDATE
- **공유 락(Shared Lock) 구문** : SELECT FOR SHARE

일반적인 RDBMS를 먼저 보자면 **`갭 락(Gap Lock)`{: .text-blue}**이 존재하지 않아 **`Phantom Read`{: .text-blue}**는 발생하게 된다고 한다.

![Phantom Read Solve2]({{site.url}}/assets/img/Transaction-JPA-Lock/5-Phantom_Read_Solve2.png)

**MVCC**를 통해 **Phantom Read**가 발생하지 않을 것 같지만 두 번째 실행되는 **SELECT FOR UPDATE**로 인해서 발생하게 된다. 그 이유는 **배타적 락**을 통한 데이터 조회의 경우 Undo 로그가 아닌 테이블을 읽기 때문이다. Lock을 통한 읽기는 테이블에 변경이 일어나지 않도록 **테이블 락**을 걸어 조회한다.

따라서 **SELECT FOR UPDATE**와 **SELECT FOR SHARE**의 경우 테이블의 레코드를 가져오게 되므로 COMMIT된 레코드도 모두 가져오게 되고, 결국 **`Phantom Read`{: .text-blue}**가 발생한다.

#### **갭 락(Gap Lock)**

위에서 **`갭 락(Gap Lock)`{: .text-blue}**이 언급되었는데, 갭 락은 MySQL에서 사용되는 특정 유형의 매커니즘이다. 트랜잭션 동안 쿼리간 범위에 존재하는 간격을 잠그는데 사용된다.

![REPEATABLE READ GAP LOCK]({{site.url}}/assets/img/Transaction-JPA-Lock/6-REPEATABLE_READ_GAP-LOCK.png)

트랜잭션 A에서 배타적 락을 걸어 ID가 1 이상인 값을 조회하게 되면 MySQL에서는 ID가 1보다 큰 범위에는 갭 락으로 넥스트 키 락을 걸게된다. 다음 트랜잭션 B에서 데이터 추가가 발생하면 트랜잭션 A가 종료될 때까지 대기하게 된다.

따라서, MySQL은 **REPEATABLE READ** 격리 수준에서 **`갭 락(Gap Lock)`{: .text-blue}**으로 인해 **Phantom Read**가 발생하지 않는다. 이 경우는 다른 트랜잭션에서 데이터 추가 전에 **배타적 락(SELECT FOR UPDATE)** 혹은 **공유 락(SELECT FOR SHARE)**을 이용해야 가능하기에 일반적인 SELECT 구문을 이용한 경우에는 **Phantom Read**는 발생하게 된다.

#### **REPEATABLE READ에서 Phantom Read가 발생하는 케이스**

<table>
  <colgroup>
    <col width="50%" />
    <col width="50%" />
  </colgroup>
  <thead>
    <tr class="header">
      <th>구분</th>
      <th>Phantom Read 발생 여부</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td markdown="span">SELECT FOR UPDATE → SELECT</td>
      <td markdown="span">갭 락(Gap Lock)으로 Phantom Read 발생하지 않음</td>
    </tr>
    <tr>
      <td markdown="span">SELECT FOR UPDATE → SELECT FOR UPDATE</td>
      <td markdown="span">갭 락(Gap Lock)으로 Phantom Read 발생하지 않음</td>
    </tr>
    <tr>
      <td markdown="span">SELECT → SELECT</td>
      <td markdown="span">MVCC로 Phantom Read 발생하지 않음</td>
    </tr>
    <tr>
      <td markdown="span">SELECT → SELECT FOR UPDATE</td>
      <td markdown="span">Phantom Read 발생</td>
    </tr>
  </tbody>
</table>

#### **READ COMMITTED**

READ COMMITTED는 반복 읽기를 수행하면 다른 트랜잭션에 의해 커밋된 내용을 조회하게 된다. **`Dirty Read`{: .text-blue}**는 방지되지만, 다른 트랜잭션의 커밋 여부에 따라 결과가 달라져 데이터의 부정합이 발생할 수 있는 **`Non-repeatable Read`{: .text-blue}** 문제가 발생할 수 있다. 

**`Non-repeatable Read`{: .text-blue}**의 경우 일반적인 경우에는 문제가 되지 않지만, 은행 시스템에서의 돈과 관련된 처리에서는 큰 문제가 발생할 수 있다. 

예를 들어, 당일 전체 결제 비용 합산과 사용처별 결제 비용 합산을 처리하는 기능이 있다고 하자. 전체 결제 비용 합산이 완료되고 사용처별 결제 합산 비용을 처리하기 전 다른 트랜잭션에 의해 결제 내역이 계속해서 커밋되면, 전체 결제 비용과 사용처별 결제 비용은 맞지 않게 된다.

따라서, 격리 수준을 명확히 알고 결과를 예측할 수 있어야 한다.

#### **READ UNCOMMITTED**

READ UNCOMMITTED는 가장 낮은 격리 수준이다. 다른 트랜잭션에 의해 커밋되지 않은 작업 내용도 읽을 수 있다. 이러한 점에서 **`Dirty Read`{: .text-blue}**, **`Non-repeatable Read`{: .text-blue}** 등의 문제가 발생할 수 있다.

해당 격리수준에서 발생하는 **`Dirty Read`{: .text-blue}**를 보자면, 다른 트랜잭션에서 커밋되지 않는 데이터 읽어 처리하다가 갑작스런 오류로 인해 해당 데이터가 갑자기 롤백이 된 상황에서 다시 해당 데이터를 읽을 때 결과가 없는 것을 보게 된다. 정리하면, 데이터를 가져와서 열심히 처리하고 있는데 추가적인 작업으로 해당 데이터를 다시 읽었을 때 없어진 경우 앞에서 처리한 로직은 불필요한 작업이 된다.

그래서 **READ UNCOMMITTED**는 RDBMS에는 적합성에 문제가 맞은 격리 수준이며 최소한 **READ UNCOMMITTED**을 사용해야하는 것으로 보인다.

### **트랜잭션 격리 수준 정리**

<table>
  <colgroup>
    <col width="25%" />
    <col width="25%" />
    <col width="25%" />
    <col width="25%" />
  </colgroup>
  <thead>
    <tr class="header">
      <th style="text-align:center">격리수준</th>
      <th style="text-align:center">Dirty Read</th>
      <th style="text-align:center">Non-repeatable Read</th>
      <th style="text-align:center">Phantom Read</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td markdown="span">SERIALIZABLE</td>
      <td markdown="span" style="text-align:center">X</td>
      <td markdown="span" style="text-align:center">X</td>
      <td markdown="span" style="text-align:center">X</td>
    </tr>
    <tr>
      <td markdown="span">REPEATABLE READ</td>
      <td markdown="span" style="text-align:center">X</td>
      <td markdown="span" style="text-align:center">X</td>
      <td markdown="span" style="text-align:center">O<br/>(RDBMS에 따라 다름)</td>
    </tr>
    <tr>
      <td markdown="span">READ COMMITTED</td>
      <td markdown="span" style="text-align:center">X</td>
      <td markdown="span" style="text-align:center">O</td>
      <td markdown="span" style="text-align:center">O</td>
    </tr>
    <tr>
      <td markdown="span">READ UNCOMMITED</td>
      <td markdown="span" style="text-align:center">O</td>
      <td markdown="span" style="text-align:center">O</td>
      <td markdown="span" style="text-align:center">O</td>
    </tr>
  </tbody>
</table>

## **JPA 락(Lock)**

JPA는 DB에 대한 동시 접근으로부터 엔티티에 대한 무결성을 유지할 수 있게 해주는 동시성 제어 매커니즘을 지원해준다. 이 매커니즘은 **`낙관적 락(Optimistic Lock)`{: .text-blue}**과 **`비관적 락(Pessimistic Lock)`{: .text-blue}**이다.

### **낙관적 락(Optimistic Lock)**

특정 자원에 대한 경쟁을 낙관적으로 바라보는 방식으로 **충돌이 일어나지 않을 것이라 가정하에** 여러 트랜잭션의 수정에 대해 충돌을 방지하는 기법이다. 데이터베이스의 락 매커니즘에 의존하지 않으며 애플리케이션 레벨에서 구현 가능하다.

여러 트랜잭션에 동시에 데이터에 접근할 수 있도록 허용한다는 점에서 성능상 이점이 있고 충돌이 일어날 경우 예외를 반환하는 방식으로 동시성 이슈를 해결한다. 하지만, 동시성 충돌이 자주 발생하는 경우에는 성능 저하가 발생한다.

#### **낙관적 락 구현**

JPA에는 **`@Version`{: .text-blue}** 어노테이션이 있다. 이를 이용해 엔티티의 **낙관적 락(Optismistic Lock)**을 구현할 수 있다.

**Board.java 엔티티**

```java
@Entity(name = "tb_board")
@Getter
@NoArgsConstructor
@DynamicUpdate
public class Board {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String title;
  private String contents;
  private LocalDateTime writeDate = LocalDateTime.now();
  private LocalDateTime modifyDate;

  //낙관적 락을 이용하기 위한 속성
  @Version
  private Long version;
 
  ...
}
```

**BoardService.java**

```java
@Service
@RequiredArgsConstructor
public class BoardService {

  private final BoardRepository boardRepository;
  
  public void modifyTitle(Long id, String title){
    Board board = boardRepository.findById(id).orElse(null);
    if(board != null)
      board.updateTitle(title);
  }
}
```

#### **게시물 데이터 1건 수정 테스트**

**테스트 코드**

```java
@SpringBootTest
@Import(DataSourceConfig.class)
class JpatransactionApplicationTests {

  @Autowired
  private BoardService boardService;

  @Test
  public void 낙관적락_수정_테스트(){
    boardService.modifyTitle((long)29737444, "낙관적 락 제목 수정");
  }
}
```

**테스트 데이터**

![JPA Optimistic Lock Data]({{site.url}}/assets/img/Transaction-JPA-Lock/7_JPA_OPTIMISTIC_LOCK_DATA.png)

**테스트 결과**

![JPA Optimistic Lock Log]({{site.url}}/assets/img/Transaction-JPA-Lock/8_JPA_OPTIMISTIC_LOCK_LOG.png)

일반적으로 실행되는 JPA에서의 수정 쿼리와 달리 **version** 컬럼에 대한 별도 수정코드를 작성하지 않아도 수정 대상 및 조건절에 추가 되어있으며 결과 데이터를 확인하면 아래와 같이 **version** 컬럼 값이 1 증가 한 것을 볼 수 있다.

![JPA Optimistic Lock Result Data]({{site.url}}/assets/img/Transaction-JPA-Lock/9_JPA_OPTIMISTIC_LOCK_RESULT_DATA.png)

#### **게시물 데이터 5건 수정 동시성 테스트**

**테스트 코드**

```java
@SpringBootTest
@Import(DataSourceConfig.class)
class JpatransactionApplicationTests {

  @Autowired
  private BoardService boardService;

  @Test
  public void 낙관적락_수정_동시성_테스트() throws Exception {
    int threadCount = 5;

    //ExecutorService : 비동기를 처리할 수 있도록 하는 Java API
    ExecutorService executorService = Executors.newFixedThreadPool(5);

    //다른 스레드에서 수행이 완료될 때 까지 대기할 수 있도록 하는 API이며 요청이 완료될 때까지 대기한다
    CountDownLatch latch = new CountDownLatch(threadCount);

    for (int i = 0; i < threadCount; i++) {
      int finalI = i;
      executorService.submit(() -> {
        try {
          boardService.modifyTitle((long)29737444, "낙관적 락 제목 수정" + (finalI +1));
        }
        finally {
          latch.countDown();
        }
      });
    }

    latch.await(); //다른 쓰레드에서 수행중인 작업이 완료될때까지 대기한다.
  }
}
```

**테스트 결과**

![JPA Multithread Concurrency Test Log1]({{site.url}}/assets/img/Transaction-JPA-Lock/10_JPA_MULTITHREAD_CONCURRENCY_TEST_LOG1.png)

![JPA Multithread Concurrency Test Log2]({{site.url}}/assets/img/Transaction-JPA-Lock/11_JPA_MULTITHREAD_CONCURRENCY_TEST_LOG2.png)

![JPA Multithread Concurrency Test Data]({{site.url}}/assets/img/Transaction-JPA-Lock/12_JPA_MULTITHREAD_CONCURRENCY_TEST_DATA.png)

로그를 보면 2개 이상의 트랜잭션이 동일한 엔티티를 동시에 수정하는 상황에서 **`ObjectOptimisticLockingFailureException`{: .text-red}** 오류가 발생했다.

특정 트랜잭션에서 엔티티가 수정되어 **version**은 1에서 2가 되었다. 이후 다른 트랜잭션에서의 엔티티 **version** 값은 1인 상황에서 엔티티를 수정하려 할 때 이미 특정 트랜잭션에 의해 엔티티가 변경되었기에 오류를 반환하게 된다.

#### **낙관적 락(Optimistic Lock) - LockModeType**

JpaRepository를 사용한다면 @Lock 어노테이션을 이용해 LockModeType를 설정할 수 있다.

```java
@Repository
public interface BoardRepository extends JpaRepository<Board, Long> {
  @Lock(LockModeType.OPTIMISTIC) //NONE, OPTIMISTIC, OPTIMISTIC_FORCE_INCREMENT
  Optional<Board> findById(Long id);
}
```

**NONE**

엔티티에 @Version 어노테이션이 적용된 필드가 있다면 낙관적 락이 적용된다.

**OPTIMISTIC(Read)**

낙관적 락이 엔티티 수정시에만 발생하지만 읽기 시에도 발생하도록 한다. 트랜잭션이 시작과 종료에서 버전검사를 수행한다. 버전 값을 체크하면서 다른 트랜잭션에서 변경하지 않음을 보장하며, **`Dirty Read`{: .text-blue}**와 **`Non-repeatable Read`{: .text-blue}**를 방지한다.

![JpaRepository LockMode Optimistic]({{site.url}}/assets/img/Transaction-JPA-Lock/13_JPAREPOSITORY_LOCKMODE_OPTIMISTIC.png)

위와 같이 테이블의 버전을 체크해 검사한다.

**OPTIMISTIC_FORCE_INCREMENT(Write)**

낙관적 락을 사용하면 버전 정보를 강제로 증가시키는 옵션이다. 읽기에서 사용시 강제로 버전 정보를 증가시킨다.

```java
@Transactional
public Board findById(Long id){
  return boardRepository.findById(id).get();
}
```

- 버전을 강제로 증가시키기에 트랜잭션 선언은 필수적이다. 선언을 하지 않을 경우 아래 오류가 발생한다.

  ![JpaRepository LockMode Optimistic Transaction Error]({{site.url}}/assets/img/Transaction-JPA-Lock/14_JPAREPOSITORY_LOCKMODE_OPTIMISTIC_TRANSACTION_ERROR.png)

![JpaRepository LockMode Optimistic force Increment]({{site.url}}/assets/img/Transaction-JPA-Lock/15_JPAREPOSITORY_LOCKMODE_OPTIMISTIC_FORCE_INCREMENT.png)

단순 조회만 했을 뿐인데 버전을 증가시키는 것을 확인할 수 있다.

### **비관적 락(Pessimistic Lock)**

비관적 락은 낙관적 락과 반대로 **여러 트랜잭션이 데이터를 동시에 수정할 것이라고 가정하에** 특정 자원 경쟁을 비관적으로 바라보는 매커니즘이다. 하나의 트랜잭션이 데이터를 읽는 시점에서 락(Lock)을 걸고 조회 혹은 업데이트 처리가 완료될 때 다른 트랜잭션은 대기 상태가 된다.

정리하면, 먼저 락(Lock)을 거는 방식이다. 데이터베이스 트랜잭션 락 매커니즘에 의존하는 방법이며 대표적인 구문으로 **배타적 락(Exclusive Lock)의 SELECT FOR UPDATE**가 있다.

#### **두 번의 갱신 분실 문제(Second Lost Updates Problem)**

비관적 락으로 발생가능한 문제로 **`두 번의 갱신 분실 문제`{: .text-blue}**가 있다. 예를 들어 사용자 A와 B가 게시물의 제목을 동시에 수정한 상황을 보면, 두 사용자가 동시에 수정했을 때 A가 요청한 처리가 우선적으로 완료되고 바로 다음에 B의 요청이 처리되었다. 이 경우 A의 수정 내용은 손실되고 B 수정 내용만 남게 된다.

- **최초 커밋만 인정** : A 사용자의 수정내용만 인정하고 B 사용자의 수정내용은 오류를 발생시킨다.
- **마지막 커밋만 인정** : A 사용자 수정내용은 무시하고 B 사용자의 수정내용만 처리한다.
- **갱신 내용 병합** : 사용자 A,B의 수정사항을 병합한다.

갱신 분실 문제에 대해서 3가지 선택 방법이 있는데, 기본은 **마지막 커밋 인정**이다.

#### **데드 락(Dead Lock) 문제**

비관적 락의 경우 락(Lock)을 먼저 건다는 것에서 데드락(Dead Lock)의 가능성이 있다. 배타적 락(Exclusive Lock)을 사용할 때의 단점과 같다.

#### **비관적 락(Pessimistic Lock) - LockModeType**

```java
@Repository
public interface BoardRepository extends JpaRepository<Board, Long> {
  @Lock(LockModeType.PESSIMISTIC_WRITE) //PESSIMISTIC_WRITE, PESSIMISTIC_READ, PESSIMISTIC_FORCE_INCREMENT
  Optional<Board> findById(Long id);
}
```

**PESSIMISTIC_WRITE**

비관적 락에서의 일반적인 옵션이며, 베타적 락(Exclusive Lock)을 걸 때 사용한다.

**PESSIMISTIC_WRITE - 동시성 테스트**

```java
@Test
public void 비관적락_수정_동시성_테스트() throws Exception {
  int threadCount = 5;

  //ExecutorService : 비동기를 처리할 수 있도록 하는 Java API
  ExecutorService executorService = Executors.newFixedThreadPool(5);

  //다른 스레드에서 수행이 완료될 때 까지 대기할 수 있도록 하는 API이며 요청이 완료될 때까지 대기한다
  CountDownLatch latch = new CountDownLatch(threadCount);

  for (int i = 0; i < threadCount; i++) {
    int finalI = i;
    executorService.submit(() -> {
      try {
        boardService.modifyTitle((long)29737444, "낙관적 락 제목 수정" + (finalI +1));
      }
      finally {
        latch.countDown();
      }
    });
  }

  latch.await(); //다른 쓰레드에서 수행중인 작업이 완료될때까지 대기한다.
}
```

![Pessmistic Write Log]({{site.url}}/assets/img/Transaction-JPA-Lock/16_PESSIMISTIC_WRITE_LOG.png)

![Pessmistic Write Data]({{site.url}}/assets/img/Transaction-JPA-Lock/17_PESSIMISTIC_WRITE_DATA.png)

낙관적 락과 다르게 충돌에 대한 오류가 발생하지 않았다. 그리고 다섯번의 수정 중 마지막 수정내역이 적용된 것을 볼 수 있다.

![Optimictic Lock Time]({{site.url}}/assets/img/Transaction-JPA-Lock/18_OPTIMISTIC-LOCK_TIME.png)

![Pessmistic Lock Time]({{site.url}}/assets/img/Transaction-JPA-Lock/19_PESSIMISTIC-LOCK_TIME.png)

추가적으로, 동시에 100건 요청으로 테스트했을 때 같은 로직에서 낙관적 락과 비관적 락을 비교하면 비관적 락이 수행시간이 더 짧은 것을 볼 수 있다. 충돌이 많은 경우 락을 획득할 때까지 대기하는 비관적 락이 성능이 좋은 것을 볼 수 있다.

**PESSIMISTIC_READ**

데이터를 반복 읽기만 하고 수정하지 않는 용도로 락을 걸 때 사용한다.

**PESSIMISTIC_FORCE_INCREMENT**

비관적 락 중 유일하게 버전 정보를 사용한다. 낙관적 락에서와 같이 번정보를 강제로 증가시킨다. Hibernate는 nowait를 지원하는 데이터베이스에 대해서 for update nowait 옵션을 사용한다.

## **정리**

### **트랜잭션 격리 수준(Transaction Isolation Level)**

- 트랜잭션에는 격리 수준이 있으며 DBMS에 따라 기본 격리 수준이 다를 수 있다.
**트랜잭션 격리 수준에서 일관성 문제로 `Dirty Read`{: .text-blue}**, **`Non-repeatable Read`{: .text-blue}**, **`Phantom Read`{: .text-blue}**가 있다.
- 가장 낮은 수준의 **READ UNCOMMITED**의 경우 **`Dirty Read`{: .text-blue}**, **`Non-repeatable Read`{: .text-blue}**, **`Phantom Read`{: .text-blue}**의 일관성 문제가 모두 발생할 수 있으니 최소한 **READ COMMITED** 격리 수준을 적용해야할 필요가 있다.
- **SERIALIZE**의 경우 가장 높은 격리 수준으로 일관성 문제는 발생하지 않지만, 그 만큼 성능에 영향이 있다.
- **REPEATABLE READ**의 경우 DBMS에 따라 **`Phantom Read`{: .text-blue}**가 발생할 수 있지만, MySQL에서는 **`MVCC(Multi-Version Concurrency Control, 다중 버전 동시성 제어)`{: .text-blue}**와 **`배타적 락(Exclusive Lock)`{: .text-blue}**에서의 **`갭 락(Gap Lock)`{: .text-blue}** 으로 **`Phantom Read`{: .text-blue}**가 발생하지 않도록 가능하다.

### **낙관적 락(Optimistic Lock)**

- DB의 트랜잭션 락 매커니즘에 의존하지 않고 애플리케이션 레벨에서 구현 가능하다.
- 데이터를 변경할 때만 충돌을 확인하기에 여러 사용자가 동시에 데이터를 읽을 수 있는 점에서 비관적 락과 비교해 성능이 좋다.
- 동시성 이슈가 많이 발생하는 경우 성능이 떨어질 수 있기에 충돌이 발생할 가능성이 낮은 경우에 유용하다.

### **비관적 락(Pessimistic Lock)**

- DB의 트랜잭션 락 매커니즘에 의존한다.
- 데이터를 읽을 때부터 락을 걸어 트랜잭션이 완료되기 전까지 다른 트랜잭션이 데이터를 변경할 수 없기에 데이터의 일관성을 보장한다.
- 락이 걸린 데이터에 대해서 다른 트랜잭션이 접근할 수 없기에 성능 저하와 데드락(Dead Lock) 발생 가능성이 높다.
  - 트랜잭션 대기시간을 설정하여 데드락 문제를 해결할 수 있다.
- 갱신 분실 문제 발생의 경우 낙관적 락을 통해 해결 가능하다.
- 충돌이 많이 발생할 수 있는 경우에는 낙관적 락보다 성능면에서 좋다.