---
title: Spring Boot Redis Caching 해보기
# author: dowonl2e
date: 2024-03-25 13:20:00 +0800
categories: [Spring, Redis]
tags: [Spring Boot, JPA, Spring Data JPA, Querydsl, Redis, Lettuce]
pin: true
img_path: "/assets"
image:
  path: /commons/Spring-Boot-Redis.png
  alt: Spring Boot + Redis
---

이전 게시물에서 **Docker를 이용한 Redis Master/Replica(Slave), Sentinel 환경 구축 및 AOF 매커니즘을 이용한 백업**에 대해 보았습니다. 구축한 환경을 기반으로 이번 게시물에서는 Spring Boot에서 Redis 캐싱 설정 및 처리를 해보겠습니다.

> [Redis Master / Replica(Slave), Sentinel, AOF With Docker](/posts/Redis-Master-Replica(Slave)_Sentinel-%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%B6%95-%EB%B0%8F-AOF-%EB%B0%B1%EC%97%85-%ED%95%B4%EB%B3%B4%EA%B8%B0/){:target="\_blank"}

## **Redis 클라이언트 라이브러리**

Spring에서 Redis 캐싱을 위해 클라이언트 라이브러리가 필요하며, Java 언어용 클라이언트 라이브러리 종류는 Jedis, Lettuce가 있습니다. 

### **Jedis**

Jedis는 Redis의 모든 주요 기능을 지원하며 Redis를 쉽게 사용할 수 있지만, 클러스터와 동기적이다 보니 동시에 여러 요청이 들어오는 경우 스레드를 blocking 이슈가 발생할 수 있다는 단점이 있습니다.

### **Lettuce**

동기, 비동기 방식을 모두 지원하며 non-blocking 처리로 높은 성능을 제공합니다. 또한, Connection Pooling및 Clustering을 지원하여 높은 확장성을 가집니다. 반면에, Jedis에 비해 사용성이 더 어렵다는 점이 있습니다.

이번 Redis Caching은 Lettuce 클라이언트를 이용해보겠습니다.

## **테스트 환경**

- Java 17
- Spring Boot 3.2
- JPA 3.1
- Hibernate 6.4
- Spring Data JPA 3.2, Spring Data Redis, Spring Data Cache
- Querydsl 5.0
- Docker / docker-compose

## **Spring Boot Redis 설정**

### **Dependency 추가**

```gradle
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
implementation 'org.springframework.boot:spring-boot-starter-cache'
```

### **application.yml**

```yaml
spring:
  data:
    redis:
      master:
        host: {Master IP}
        port: {Master Port}
      replicas:
        - host: {Replica1 IP}
          port: {Replica1 Port}
        - host: {Replica2 IP}
          port: {Replica2 Port}
```

- Redis의 Master, Replica의 IP, Port를 지정해줍니다.

### **RedisInfo.java**

```java
@Getter
@Setter
@Configuration
@ConfigurationProperties(prefix = "spring.data.redis")
public class RedisInfo {

    private String host;
    private int port;
    private RedisInfo master;
    private List<RedisInfo> replicas;
    
}
```

- ConfigurationProperties : application.yml에서 설정한 Redis 옵션들을 가져올 수 있도록 합니다.

### **RedisConfig.java**

```java
@Configuration
@EnableCaching
@RequiredArgsConstructor
public class RedisConfig {

  private final RedisInfo redisInfo;

  private final ObjectMapper objectMapper;

  /**
   * Lettuce 클라이언트 설정
   * @return LettuceClientConfiguration
   */
  @Bean
  public LettuceClientConfiguration lettuceClientConfiguration(){
    return LettuceClientConfiguration.builder()
        .readFrom(ReadFrom.REPLICA_PREFERRED) //Replica(Slave)를 먼저 읽고 불가능하면 master를 읽는다.
        .commandTimeout(Duration.ofSeconds(3000)) //Redis 요청에 대한 응답 시간
        .build();
  }

  /**
    * LettuceRedisConnectionFactory 설정
    * @param lettuceClientConfiguration
    * @return RedisConnectionFactory
    */
  @Bean
  public LettuceConnectionFactory redisConnectionFactory(LettuceClientConfiguration lettuceClientConfiguration) {
    RedisStaticMasterReplicaConfiguration replicaConfiguration =
        new RedisStaticMasterReplicaConfiguration(redisInfo.getMaster().getHost(), redisInfo.getMaster().getPort());

    redisInfo.getReplicas().forEach(
        replica -> replicaConfiguration.addNode(replica.getHost(), replica.getPort())
    );
    return new LettuceConnectionFactory(replicaConfiguration, lettuceClientConfiguration);
  }

  /**
    * RedisCacheManagerBuilderCustomizer 설정
    * @return RedisCacheManagerBuilderCustomizer
    */
  @Bean
  public RedisCacheManagerBuilderCustomizer redisCacheManagerBuilderCustomizer() {
    return (builder) -> builder
        .withCacheConfiguration("BOARD", //캐시명 지정
            RedisCacheConfiguration.defaultCacheConfig()
                .computePrefixWith(cacheName -> cacheName + "::") //Redis의 키 Prefix 지정
                .entryTtl(Duration.ofHours(1))  //데이터 생명주기 설정 지정된 시간이 지나면 삭제된다.(Duration.ZERO = 무한대)
                .disableCachingNullValues() //Null값을 허용하지 않는다.
                .serializeKeysWith(
                    RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer())
                )  //Key의 타입을 지정(String 값을 정상적으로 읽어서 저장한다. 그러나 엔티티나 VO같은 타입은 cast 할 수 없다.)
                .serializeValuesWith(
                    RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())
                ));  //Value의 타입을 지정(모든 classType을 json 형태로 저장할 수 있도록 한다.)
  }
}
```

**@EnableCaching**

Spring에서 @Cacheable 애너테이션 기반으로 캐시 기능을 사용하기 위해 추가한다. Application 클래스에 선언해도 상관없다.

**LettuceClientConfiguration**

- ReadFrom : Master / Replica(Slave) 구조에서 어떤 것 부터 조회할지 설정한다.
  - MASTER(= UPSTREAM) : 마스터 노드만 조회한다.
  - MASTER_PREFERRED(= UPSTREAM_PREFERRED) : 마스터노드 위주로 조회한다.
  - REPLICA : 복제 노드만 조회한다. 복제 노드 중 하나만 집중하는 경향이 있어 마스터가 살아있어도 복제노드가 다운되면 오류 발생한다.
  - ANY : 마스터, 복제 노드를 고르게 분배하여 조회한다.
  - ANY_REPLICA : 복제 노드만 조회한다. 여러 복제 노드를 고르게 분배하지만, 마스터가 살아있어도 복제 노드가 다운되면 오류 발생한다.
  - REPLICA_PREFERRED : 복제 노드를 우선으로 조회하고 복제 노드가 다운되면 마스터 노드를 조회한다. 복제 노드가 한 곳에 집중되는 경향이 있다.
- commandTimeout : Redis 요청에 대한 응답 시간을 지정한다. (default : 60(초))

**LettuceConnectionFactory**

- 커스텀이 아닌 경우 기본은 RedisStandaloneConfiguration을 통해 구성하게 되며, host, port를 통해 1개 마스터 노드를 구성할 수 있다.
- Master/Replica(Slave) 구성시에는 LettuceConnectionFactory를 이용해 커스텀화하여 Master, Replica 노드의 host, port를 설정한다.

**RedisCacheManagerBuilderCustomizer**

- 여러 곳에 Redis Cache 설정을 해야하는 경우 RedisCacheManagerBuilderCustomizer를 구현할 수 있습니다.

**Key-Value Serializer**

- JdkSerializationRedisSerializer : 값이 엔티티 혹은 VO와 같은 classType의 경우 java.io.Serializable를 implements하지 않으면 아래와 같이 오류가 발생한다. (default)
  ![JdkSerializationRedisSerializer Serialize Exception]({{site.url}}/assets/img/SpringBootRedis/1_SERIALIZER_EXCEPTION.png)
- StringRedisSerializer : String 값을 읽어서 저장한다. (엔티티나 DTO같은 타입은 cast 불가능)
- Jackson2JsonRedisSerializer(classType.class) : classType 값을 json 형태로 저장한다. 특정 클래스에만 직속되어있다는 단점이 있다.
- GenericJackson2JsonRedisSerializer : 모든 classType을 json 형태로 저장할 수 있어 범용으로 사용할 수 있다.
- Jackson2JsonRedisSerializer이다. 캐싱에 클래스 타입도 저장된다는 단점이 있지만 RedisTemplate을 이용해 다양한 타입 객체를 캐싱할 때 사용하기에 좋다.

### **BoardService.java**

조회, 수정, 삭제 캐시 처리를 위해 각 메서드에 @Cacheable, @CachePut, @CacheEvict를 선언합니다.

```java
@Service
@RequiredArgsConstructor
public class BoardService {

  private final BoardRepository boardRepository;

  @Transactional(readOnly = true)
  @Cacheable(value = "BOARD", key = "'Board_'+#id", unless = "#result == null")
  public Board getBoard(final Long id){
    return boardRepository.findById(id).get();
  }

  @Transactional
  @CachePut(value = "BOARD", key = "'Board_'+#id", unless = "#result == null")
  public Board updateBoard(final Long id, final String title, final String contents){
    Board board = getBoard(id);
    board.updateTitleContents(title, contents);
    return board;
  }

  @Transactional
  @CacheEvict(value = "BOARD", key = "'Board_'+#id")
  public void deleteBoard(final Long id){
    boardRepository.deleteById(id);
  }
}
```

### **Cacheable, CachePut, CacheEvict 애너테이션**

**Cacheable**

메서드 호출시 캐시명(value), 키(key)를 이용해 Redis에 저장된 데이터가 있으면 해당 데이터를 반환하며, 없을 경우 메서드 내부의 로직을 실행 후 반환할 결과를 Redis에 저장합니다.

- value : 캐시명
- key : Redis 데이터 키 (Spring Expression Language)
- unless : Null 값을 캐싱하지 않기 위해 사용한다. RedisConfig 설정시에 disableCachingNullValues를 통해 Null 값에 대해 처리했다고 생각했지만, Null 값 방지를 위한 정책이며 실제 캐싱 처리의 경우 **`unless = "#result == null"`{: .text-blue}** 옵션으로 캐싱이 되지 않도록 한다. 그렇지 않은 경우 아래와 같이 오류가 발생한다.
  ![Null Value With Not Use Unless Option]({{site.url}}/assets/img/SpringBootRedis/2_CACHEABLE_NOUNLESS_RESULT.png)
- keyGenerator : 중복되는 캐시의 key가 발생하는 경우 커스텀 KeyGenerator를 Bean으로 등록해 사용할 수 있다.
- cacheManager : 캐시 설정시 각 기능 별로 cacheManager를 구성할 경우 Bean으로 등록해 사용할 수 있다.

**CachePut**

Cacheable과 같이 데이터를 저장하는 역할이다. 차이점은 Cacheable의 경우 데이터가 있으면 메서드내 로직을 실행하지 않지만, CachePut의 경우 항상 메서드내 로직을 수행 후 데이터를 저장한다. 데이터 갱신 시 많이 사용한다.

**CacheEvict**

캐시에 있는 데이터를 삭제하는 역할이다.

**CacheConfig**

메서드가 아닌 클래스에 붙여서 공통 캐시 기능으로 지정할 수 있다.

**Caching**

Cacheable, CachePut, CacheEvict를 여러 개 사용할 때 이용할 수 있다.

## **Redis Caching 테스트**

### **조회(Redis 데이터 추가) 테스트**

**테스트 코드**

```java
@SpringBootTest
@Import({RedisConfig.class})
@DisplayName("Redis 캐시 테스트")
public class SpringDataJPARedisTests {

  @Autowired
  private BoardService boardService;

  @Test
  @DisplayName("Spring Data JPA Redis Caching 조회 테스트")
  @Rollback(value = false)
  public void testToSpringDataJPARedisSelect1() {
    Board board = boardService.getBoard((long) 1);
    Assertions.assertEquals(board.getId(), 1);
  }

}
```

**테스트 결과**

![Select Query Result]({{site.url}}/assets/img/SpringBootRedis/3_DATA_SELECT.png)

![Select Redis Data Result]({{site.url}}/assets/img/SpringBootRedis/4_REDIS_DATA.png)

![Select Redis Data Detail Result]({{site.url}}/assets/img/SpringBootRedis/5_REDIS_DATA_DETAIL.png)

위와 같이 Redis에 데이터가 추가되는 것을 볼 수 있으며, 다시 한번 더 실행할 경우 Redis에 데이터가 있기 때문에 SQL이 출력되지 않는 것을 확인할 수 있다.

### **수정(Redis 데이터 갱신) 테스트**

**테스트 코드**

```java
@Test
@DisplayName("Spring Data JPA Redis Caching 수정 테스트")
@Rollback(value = false)
public void testToSpringDataJPARedisUpdate() {
  Board board = boardService.updateBoard((long)1, "영화 게시판 제목1 수정", "영화 게시판 내용1 수정");
  Assertions.assertEquals(board.getId(), 1);
}
```

**테스트 결과**

![Update Query Result]({{site.url}}/assets/img/SpringBootRedis/6_REDIS_DATA_UPDATE.png)

![Update Query Redis Result]({{site.url}}/assets/img/SpringBootRedis/7_REDIS_DATA_UPDATE_RESULT.png)

매서드 내 로직에서 UPDATE SQL이 실행되고 데이터 확인시 변경된 것을 볼 수 있습니다

### **수정(Redis 데이터 수정)시 Content 반환이 아닌 경우**

**테스트 코드**

Spring Data JPA 데이터 수정시 Dirty Checking으로 인해 Querydsl을 이용

```java
@Transactional
@CachePut(value = "BOARD", key = "'Board_'+#id", unless = "#result == null")
public HttpStatus updateBoardOnly(final Long id, final String title, final String contents){
  BoardDto boardDto = new BoardDto(id, title, contents);
  long result = boardRepository.updateBoard(boardDto);
  return result > 0 ? HttpStatus.NO_CONTENT : HttpStatus.INTERNAL_SERVER_ERROR;
}
```

**테스트 결과**

![Update Query No Content Result]({{site.url}}/assets/img/SpringBootRedis/8_REDIS_DATA_NOCONTENT.png)

반환 결과를 HttpStatus로 할 경우 위와 같이 데이터가 HttpStatus의 값으로 변경됩니다.

### **삭제(Redis 데이터 삭제) 테스트**

**테스트 코드**

```java
@Test
@DisplayName("Spring Data JPA Redis Caching 삭제 테스트")
@Rollback(value = false)
public void testToSpringDataJPARedisDelete() {
  boardService.deleteBoard((long)1);
}
```

**테스트 결과**

![Delete Query Result]({{site.url}}/assets/img/SpringBootRedis/9_REDIS_DATA_DELETE.png)

![Delete Query Redis Result]({{site.url}}/assets/img/SpringBootRedis/10_REDIS_DATA_DELETE_RESULT.png)

DELETE SQL이 실행되고 레디스에 데이터 조회시 삭제된 것을 볼 수 있습니다.

### **페이징(Redis 데이터 조회) 테스트**

현재 약 2900만건 데이터가 있으며 Redis 캐시 처리 전과 후 실행 시간을 비교해보겠습니다.

**Page 인터페이스 캐시 조회 오류 발생**

페이징 처리에 대한 캐시 처리해야할 때 반환이 **`Page`**의 경우 **`new PageImpl<>(boards.get().toList(), pageable, boards.getTotalElements());`{: .text-blue }** 와 같이 사용하면 Redis에 데이터는 저장될 것이다.

하지만, 하지만, 캐시 데이터를 조회할 경우 PageImpl에 기본 생성자 혹은 매핑될 수 있는 생성자가 없어 역직렬화가 불가능해 **`SerializationException`{: .text-red }** 오류가 발생한다.

![PAGE Interface SerializationException]({{site.url}}/assets/img/SpringBootRedis/11_PAGE_SerializationException.png)

그래서 아래와 같이 PageImpl을 커스텀화 하여 해당 클래스를 반환하는 방식으로 진행해야 한다.

**CustomPageImpl.java**

```java
public class CustomPageImpl<T> extends PageImpl<T> {

  @Serial
  private static final long serialVersionUID = -9150242189787752130L;

  @JsonCreator(mode = JsonCreator.Mode.PROPERTIES)
  public CustomPageImpl(@JsonProperty("content") List<T> content, @JsonProperty("number") int page,
                        @JsonProperty("size") int size, @JsonProperty("totalElements") Long totalElements,
                        @JsonProperty("last") boolean last, @JsonProperty("totalPages") int totalPages,
                        @JsonProperty("sort") JsonNode sort, @JsonProperty("numberOfElements") int numberOfElements,
                        @JsonProperty("first") boolean first, @JsonProperty("empty") boolean empty) {
    super(content, PageRequest.of(page, size), 30);
  }

  public CustomPageImpl(List<T> content, Pageable pageable, long total) {
    super(content, pageable, total);
  }

}
```

- @JsonCreator : Json 데이터를 Class로 바꾸고자 할 때 사용하며, 기본 생성자가 아닌 다른 생성자나 팩토리 함수를 통해서 Class를 만들 수 있다.
  - JsonCreator.Mode.PROPERTIES : 들어오는 JSON 데이터를 파라미터에 매핑시키기 위해 설정한다. (매핑을 위해 JsonProperty 애너테이션 사용)
- @JsonProperty : Redis의 데이터를 조회하면 Page에 관련된 Key-Value가 있습니다. 이 Key를 모두 지정해주면 된다.

**BoardService.java**

```java
@Transactional(readOnly = true)
@Cacheable(value = "BOARD", key = "'Boards_'+#offset+'_'+#pageSize", unless = "#result == null")
public CustomPageImpl<Board> getBoards(int offset, int pageSize) {
  Pageable pageable = PageRequest.of(offset, pageSize, Sort.Direction.DESC, "id");
  Page<Board> boards = boardRepository.findAll(pageable);
  return new CustomPageImpl<>(boards.get().toList(), pageable, boards.getTotalElements());
}
```

**테스트 코드**

```java
@Test
@DisplayName("Spring Data JPA Redis Caching 게시물 페이징 테스트")
@Rollback(value = false)
public void testToSpringDataJPARedisSelectList() {
  Instant stime = Instant.now();

  CustomPageImpl<Board> boards = boardService.getBoards(0, 30);

  Instant etime = Instant.now();
  System.out.println("Execution Time: "+ Duration.between(stime, etime).toMillis()+"ms");

  Assertions.assertEquals(boards.getSize(), 30);
}
```

**1) SQL 테스트 결과**

![Paging Caching SQL Execution Time]({{site.url}}/assets/img/SpringBootRedis/12_SELECT_LIST_TIME.png)

**2) 캐시 테스트 결과**

![Paging Caching Redis Execution Time]({{site.url}}/assets/img/SpringBootRedis/13_REDIS_LIST_TIME.png)

10번 정도 테스트하여 응답속도를 비교했을 때 최소 8배 차이가 나는 것을 볼 수 있었습니다.

### **페이징 데이터 캐시처리 해도 될까?**

위 결과를 보면, 당연히 페이징에서도 캐시 처리가 필요할 것으로 보인다.

주관적인 생각이지만, 추가/수정/삭제에 대한 정보가 실시간으로 사용자에게 제공되어야 할 경우에 캐시 데이터를 계속해서 갱신해야 하기에 문제가 있을 것으로 보인다. 데이터가 수정될 경우에는 해당 데이터가 어디에 포함되어 있는지 정확히 알 수 없기에 페이징에 대한 캐시 데이터를 모두 찾아야 할 수 있다.

실시간성 데이터가 아닌 주기적으로 갱신해도 되는 경우 성능 이슈를 발생시키지 않는 선에서는 “좋은 선택이지 않을까?”라고 생각한다.