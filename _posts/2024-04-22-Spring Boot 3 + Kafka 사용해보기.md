---
title: Spring Boot 3에서 Kafka를 이용해 데이터 처리해보기
# author: dowonl2e
date: 2024-04-22 13:30:00 +0800
categories: [Spring, Kafka]
tags: [Spring Boot, Kafka, Docker]
pin: true
img_path: "/assets"
image:
  path: /commons/Spring-Boot-Kafka.png
  alt: Spring Boot 3 + Kafka
---

최근 **MSA**에서의 주요 기술 중 **메시지 지향 미들웨어(Message-Oriendted Middleware, MOM)**에 대해 공부하면서 Spring Boot에서 **Kafka**를 이용해 데이터를 처리한 내용을 적어봅니다.

## **메시지 지향 미들웨어(Message-Oriented Middleware, MOM)**

MOM이란 분산 시스템에서 메시지를 안전하고 신뢰성 있게 교환하는 데 사용되는 소프트웨어 프레임워크 또는 서비스이며, 제공하는 기능은 아래와 같습니다.

1. **Message Queue** : MOM은 메시지를 안전하게 저장하고 관리하는 **메시지 큐**를 제공합니다. 이를 통해 송신자는 메시지를 큐에 넣고, 수신자는 해당 큐에서 메시지를 가져와 처리할 수 있습니다.
2. **비동기 통신** : MOM은 비동기적인 메시지 통신을 지원하여 송신자가 메시지를 보낸 후 즉시 다른 작업을 수행할 수 있도록 합니다. 수신자는 메시지를 처리할 준비가 되면 메시지를 가져와 처리합니다.
3. **신뢰성과 내구성** : MOM은 메시지를 안전하게 전달하고, 메시지의 손실을 방지하기 위해 내구성 있는 메시지 전달 기능을 제공합니다. 이를 통해 시스템이 안정적으로 동작하고 데이터의 무결성을 보장할 수 있습니다.
4. **메시지 라우팅** : MOM은 메시지의 수신자를 동적으로 선택하거나 메시지를 특정 대상으로 라우팅하는 기능을 제공합니다. 이를 통해 유연하고 효율적인 메시지 전달이 가능합니다.
5. **확장성** : MOM은 대규모의 메시지 처리를 지원하기 위해 수평 확장이 가능하며, 새로운 노드를 추가하여 처리량을 쉽게 확장할 수 있습니다.
6. **다양한 프로토콜 지원** : MOM은 다양한 통신 프로토콜을 지원하여 서로 다른 시스템 간의 통신을 쉽게 구현할 수 있습니다. 일반적으로는 TCP/IP, HTTP, AMQP 등의 프로토콜이 사용됩니다.

## **메시지 브로커(Message Broker)와 이벤트 브로커(Event broker)**

### **메시지 브로커(Message Broker)**

- 메시지 브로커는 발신자와 수신자의 메시지를 중개하고 관리하는 중앙 집중형 시스템입니다.
- 발신자(Sender)는 메시지를 메시지 브로커에게 보내고, 메시지 브로커는 수신자(Receiver)에게 메시지를 전달합니다.
- 메시지 브로커는 메시지를 안전하게 저장하고 관리하여 송신자와 수신자 간의 통신을 조정합니다. 이를 통해 메시지 전달의 신뢰성과 내구성을 보장합니다.
- 대표적인 메시지 브로커는 Apache Kafka, RabbitMQ, ActiveMQ 등이 있습니다.

### **이벤트 브로커(Event Broker)**

- 이벤트 브로커는 이벤트 중심 아키텍처(Event-Driven Architecture)에서 이벤트를 중개하고 관리하는 시스템입니다.
- 이벤트 브로커는 이벤트 발생자(Event Producer)와 이벤트 소비자(Event Consumer) 간의 통신을 담당합니다. 이벤트 발생자는 이벤트를 이벤트 브로커에 보내고, 이벤트 소비자는 이벤트 브로커에서 이벤트를 구독하여 처리합니다.
- 이벤트 브로커는 이벤트를 안전하게 저장하고 관리하여 이벤트 기반 아키텍처의 확장성과 유연성을 보장합니다.
- 대표적인 이벤트 브로커로는 Apache Kafka가 있으며, Kafka는 메시지 브로커와 이벤트 브로커의 역할을 모두 수행할 수 있습니다.

### **메시지 브로커와 이벤트 브로커의 차이**

이벤트 브로커는 기본적으로 메시지 큐의 기능을 갖고 있기에 메시지 브로커의 역할을 할 수 있습니다. 이벤트 브로커는 Producer가 발생한 이벤트를 DB에 데이터를 저장하듯이 데이터를 관리하는데, Consumer는 저장된 이벤트를 전체 혹은 특정 시점부터 읽을 수 있습니다. 

하지만, 메시지 브로커의 경우 이러한 역할을 할 수 없습니다. Receiver 시스템이 갑작스런 장애로 서비스가 다운 상태가 되면 정상 운영 전까지 Sender에서 전달한 메시지 처리가 불가능할 수 있습니다.

반대로, 이벤트 브로커의 경우 Consumer 시스템이 정상 운영 상태가 되면 특정 시점을 체크해 다운 타임 동안 처리하지 못한 이벤트들을 읽어 처리할 수 있는 차이가 있습니다.

## **Apache Kafka**

Apache Kafka는 실시간 이벤트 기반 애플리케이션 개발을 지원하는 것을 비롯하여 대용량의 데이터 스트림을 안정적으로 처리하는 데 사용되는 오픈 소스 분산형 스트리밍 플랫폼입니다.

대규모의 데이터 스트림 처리와 실시간 데이터 분석을 위한 강력한 플랫폼으로 평가받고 있으며, 웹 서비스 로그 처리, 실시간 대시보드, 이벤트 기반 마이크로서비스 아키텍처 등 다양한 분야에서 활용되고 있습니다.

### **Kafka의 구조**

1. **토픽(Topic)**: 카프카에서 데이터는 토픽에 저장됩니다. 토픽은 유사한 유형이나 주제를 가진 메시지들의 모음입니다. 예를 들어, "주문", "로그", "이벤트" 등이 토픽이 될 수 있습니다.
2. **프로듀서(Producer)**: 데이터를 생성하고 토픽에 전송하는 역할을 합니다. Producer는 카프카에 메시지를 보내는 주체입니다.
3. **브로커(Broker)**: 카프카 클러스터를 구성하는 개별 서버입니다. 브로커는 데이터를 저장하고 관리하며, Producer로부터 메시지를 받아 Consumer에게 전달합니다.
4. **컨슈머(Consumer)**: 토픽에서 데이터를 소비하는 역할을 합니다. 컨슈머는 특정 토픽을 구독하여 메시지를 소비하고 처리합니다.
5. **컨슈머 그룹(Consumer Group)**: 여러 컨슈머가 특정 토픽을 동시에 소비할 수 있도록 그룹화하는 개념입니다. 컨슈머 그룹은 각 컨슈머에게 메시지를 분배하는 역할을 합니다.
6. **주키퍼(ZooKeeper)**: 카프카 클러스터의 구성 정보와 상태를 관리하는 분산 코디네이터 서비스입니다. 주키퍼는 브로커, 토픽, 컨슈머 그룹 등의 메타데이터를 저장하고 관리합니다.

## **Spring Kafka**

Spring Kafka는 Apache Kafka를 이용하여 Spring 애플리케이션을 개발할 수 있도록 하는 라이브러리입니다. Kafka의 기능 통합을 단순화하고 Spring의 기능을 함께 사용하여 보다 Kafka를 편리하게 사용할 수 있습니다.

## **Kafka를 이용한 주문 및 상품 처리 프로세스**

Kafka를 사용해보려고하는 주제는 상품 주문에 대한 것이며 아래 이미지에서 처리 과정을 확인하실 수 있습니다.

분산 시스템과 비슷한 구조로 확인해보기 위해 주문(Producer) 및 상품(Consumer) 서비스를 8080, 8081 포트로 나누었습니다.

![주문 재고 프로세스]({{site.url}}/assets/img/Spring-Boot-Kafka/1_ORDER_STOCK_PROCESS.png)

## **Docker 환경에서 Kafka 서버 실행**

**docker-compose.yml**

```yaml
version: "2"

services:
  zookeeper:
    image: bitnami/zookeeper:latest
    container_name: zookeeper
    ports:
      - 2181:2181
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes

  kafka:
    image: bitnami/kafka:latest
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - 9092:9092
      - 9094:9094
    environment:
      - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=true
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181 
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    depends_on:
      - kafka
    ports:
      - 9090:8080
    environment:
      - DYNAMIC_CONFIG_ENABLED=true
      - KAFKA_CLUSTERS_0_NAME=study_kafka
      - KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:9092
```

### **Zookeeper Container**

ZooKeeper는 Kafka 클러스터의 구성 및 조정을 관리하는 데 사용됩니다. Kafka 브로커는 주키퍼를 사용하여 클러스터의 메타데이터 및 브로커 상태를 저장하고 관리합니다. 따라서 Kafka 브로커가 실행될 때 ZooKeeper와의 연결이 필요합니다.

- **ALLOW_ANONYMOUS_LOGIN** : 인증되지 않는 사용자 허용하며, **`no`{: .text-blue}**로 설정하면 Zookeeper에 **`SSL/TLS`{: .text-blue}**설정이 필요합니다.

### **Kafka Container**

- **KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE** : 일반적으로 Kafka 브로커는 클라이언트가 토픽에 메시지를 게시하거나 구독할 때 토픽이 미리 만들어져 있어야 합니다. 그러나 **`true`{: .text-blue}**로 설정하면 클라이언트가 존재하지 않는 토픽에 메시지를 게시하거나 구독하려고 할 때 Kafka 브로커가 자동으로 해당 토픽을 생성
- **KAFKA_CFG_ZOOKEEPER_CONNECT** : Kafka 브로커가 Zookeeper와 통신하는데 필요한 주키퍼의 **`호스트:포트`{: .text-blue}**를 지정
- **KAFKA_CFG_LISTENERS**
  - **PLAINTEXT://:9092** : 기본적인 텍스트 기반의 통신을 제공합니다. **`PLAINTEXT`{: .text-blue}**는 보안이 적용되지 않은 통신을 의미하며, **`9092`{: .text-blue}** 포트를 통해 클라이언트 요청을 수신합니다. **`:`{: .text-blue}**는 모든 네트워크 인터페이스를 나타냅니다.
  - **CONTROLLER://:9093** : Kafka 컨트롤러가 브로커와 통신할 수 있는 설정입니다. 이 포트를 통해 컨트롤러 간 통신 및 브로커 관리 작업이 이루어집니다.
  - **EXTERNAL://:9094** : 외부에서 Kafka 브로커와 통신할 수 있는 설정을 제공합니다. 외부 클라이언트 또는 애플리케이션이 이 포트를 통해 Kafka에 연결할 수 있습니다.
- **KAFKA_CFG_ADVERTISED_LISTENERS**
  - **PLAINTEXT://kafka:9092**: 외부 클라이언트가 Kafka 브로커에 접속할 때 사용됩니다. 브로커의 **`호스트(kafka):포트(9092)`{: .text-blue}**로 설정합니다. 보통 내부 네트워크에서 브로커에 접속하는 클라이언트들을 위한 것입니다.
  - **EXTERNAL://localhost:9094**: 외부 클라이언트가 브로커에 접속할 때 사용됩니다. **`호스트명(localhost):포트(9094)`{: .text-blue}**로 설정합니다. 보통 외부 네트워크에서 브로커에 접속하는 클라이언트들을 위한 것입니다.
- **KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP** : 사용할 리스너(listener)와 해당 보안 프로토콜을 매핑(mapping)하는 역할을 합니다.
  - **PLAINTEXT** : 일반적인 클라이언트와 브로커 간의 통신에 사용됩니다. 보안이 적용되지 않은 텍스트 통신이 사용됩니다.
  - **CONTROLLER** : 컨트롤러 간 통신에 사용됩니다. 이 설정은 컨트롤러가 서로 통신할 때 사용할 보안 프로토콜을 정의합니다.
  - **EXTERNAL** : 외부 클라이언트가 브로커에 접속하는 데 사용됩니다.

### **Kafka UI Container**

- **DYNAMIC_CONFIG_ENABLED** : 브로커 재시작없이 설정을 동적으로 변경하기 위해 사용합니다. 일반적으로 클러스터 설정은 정적으로 관리되어 설정을 변경하면 브로커를 재시작해야합니다.
- **KAFKA_CLUSTERS_0_NAME** : 첫 번째 클러스터의 이름을 지정합니다. 여러개의 클러스터 관리시 KAFKA_CLUSTERS_1_NAME, KAFKA_CLUSTERS_2_NAME과 같은 방식으로 추가할 수 있습니다.
- **KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS** : 첫 번째 클러스터의 부트스트랩 서버를 설정합니다. 호스트 이름과 포트 번호의 쉼표로 구분하여 추가 설정할 수 있습니다.(kafka-1:9092,kafka-2:9092,kafka-3:9092)

```bash
$ docker-compose up -d
```

docker-compos.yml이 있는 경로로 이동 후 위 명령어를 실행합니다. **`http://127.0.0.1:9090`{: .text-blue}**에 접속하면 아래와 같이 Kafka UI 확인할 수 있습니다.

![Kafka UI]({{site.url}}/assets/img/Spring-Boot-Kafka/2_KAFKA_UI.png)

## **Spring Boot에서 Kafka를 이용한 주문 및 재고 확인 요청**

### **주문 서비스(Producer)**

**Spring Kafka 의존성 추가**

```gradle
implementation 'org.springframework.kafka:spring-kafka'
```

**application.yml 설정**

```yaml
...
spring:
  kafka:
    bootstrap-servers: kafka:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      transactional:
        id: kafka-stock-id
```

**KafkaProducerInterceptor.java**

메시지 발송 전 데이터를 확인할 수 있도록 interceptor를 추가해줍니다.

```java
@Slf4j
@Component
public class KafkaProducerInterceptor implements ProducerInterceptor<String, Map<String, Object>>  {

  /**
    * 메시지 발송 전 호출됩니다.
    * @param producerRecord
    * @return
    */
  @Override
  public ProducerRecord<String, Map<String, Object>> onSend(ProducerRecord<String, Map<String, Object>> producerRecord) {
    log.info("ProducerInterceptor.onSend()");
    log.info("Message Header -> " + producerRecord.headers());
    log.info("Message Body -> " + producerRecord.value());
    return producerRecord;
  }
  
  /**
    * 메시지 발송 후 호출됩니다.
    * exception이 Null이면 정상동작이며, Null이 아닌 경우 오류가 발생했다는 의미입니다.
    * @param recordMetadata
    * @param exception
    */
  @Override
  public void onAcknowledgement(RecordMetadata recordMetadata, Exception exception) {
    log.info("ProducerInterceptor.onAcknowledgement()");
    log.info("Send Result Topic -> " + recordMetadata.topic());
    log.info("Send Result Partition -> " + recordMetadata.partition());
    log.info("Send Result -> " + (exception == null ? "Success" : "Fail"));
    if(e != null) log.info("Send Result Exception -> " + exception);
  }

  @Override
  public void close() {
    log.info("ProducerInterceptor.close()");
  }

  @Override
  public void configure(Map<String, ?> map) {
    log.info("ProducerInterceptor.configure()");
    log.info("Configuration -> " + map);
  }
}
```

**KafkaProducerListener.java**

메시지 발송 이후에 정상처리 혹은 오류가 발생하면 로그를 출력할 수 있도록 Listener를 설정해줍니다.

```java
@Slf4j
@Component
public class KafkaProducerListener implements ProducerListener {

  /**
   * Message 전송된 이후 정상 처리된 경우 호출된다.
   * @param producerRecord
   * @param recordMetadata
   */
  @Override
  public void onSuccess(ProducerRecord producerRecord, RecordMetadata recordMetadata) {
    ProducerListener.super.onSuccess(producerRecord, recordMetadata);
    log.info("Message onSuccess Header -> " + producerRecord.headers());
    log.info("Message onSuccess Topic -> " + recordMetadata.topic());
    log.info("Message onSuccess Offset -> " + recordMetadata.offset());
    log.info("Message onSuccess Body -> " + producerRecord.value());
  }

  /**
   * Message 전송에 실패한 경우 호출된다.
   * @param producerRecord
   * @param recordMetadata
   * @param exception
   */
  @Override
  public void onError(ProducerRecord producerRecord, RecordMetadata recordMetadata, Exception exception) {
    ProducerListener.super.onError(producerRecord, recordMetadata, exception);
    log.info("Message onError Header -> " + producerRecord.headers());
    log.info("Message onError Offset -> " + recordMetadata.offset());
    log.info("Message onError Topic -> " + recordMetadata.topic());
    log.info("Message onError Body -> " + producerRecord.value());
    log.error("Message onError exception -> " + exception);
  }
}
```

**KafkaProducerConfig.java**

```java
@Configuration
@RequiredArgsConstructor
public class KafkaProducerConfig {

  @Value("${spring.kafka.bootstrap-servers}")
  private String bootstrapServers;

  @Value("${spring.kafka.producer.key-serializer}")
  private String keySerializer;

  @Value("${spring.kafka.producer.value-serializer}")
  private String valueSerializer;

  private final KafkaProducerInterceptor producerInterceptor;
  private final KafkaProducerListener producerListener;
  
  /**
   * BOOTSTRAP_SERVERS_CONFIG
   *   - Kafka의 기본 서버는 Bootstrap 서버로 Docker 환경에서 설정한 Kafka 서버 정보를 지정합니다. 
   * KEY_SERIALIZER_CLASS_CONFIG
   *   - 토픽의 메시지 Key 타입을 지정해줍니다.
   * VALUE_SERIALIZER_CLASS_CONFIG
   *   - 토픽의 메시지 Value 타입을 지정해줍니다.
   */
  @Bean
  public ProducerFactory<String, Map<String, Object>> producerFactory() {
    Map<String, Object> configs = new HashMap<>();
    configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
    configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, keySerializer);
    configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, valueSerializer);

    DefaultKafkaProducerFactory<String, Map<String, Object>> producerFactory = new DefaultKafkaProducerFactory<>(configs);
    return producerFactory;
  }
  
  /**
   * setProducerInterceptor
   *   - 메시지 발송 전/후에 대한 확인을 위해 Interceptor를 등록합니다. 
   * setProducerListener
   *   - 메시지 발송 결과 확인을 위해 Listener를 등록합니다. 
   */
  @Bean
  public KafkaTemplate<String, Map<String, Object>> kafkaTemplate() {
    KafkaTemplate<String, Map<String, Object>> kafkaTemplate = new KafkaTemplate<>(producerFactory());
    kafkaTemplate.setProducerInterceptor(producerInterceptor);
    kafkaTemplate.setProducerListener(producerListener);
    return kafkaTemplate;
  }
}
```

- **Key-Value Serializer** : Kafka 키-값 형식으로 **StringSerializer**, **IntegerSerializer**, **LongSerializer** 등 Java Wrapper 클래스 타입이 있으며 객체를 Json 형식으로 발송할 때 **JsonSerializer**를 이용할 수 있습니다.

**설정 후 Kafka 연동 오류 처리**

의존성 추가 및 설정을 완료 후 실행하게 되면 아래와 같은 오류가 발생합니다. 해당 오류의 경우 호스트명을 해석할 수 없을 때 발생하게 됩니다.

![Spring Boot Kafka 서버 연동 오류]({{site.url}}/assets/img/Spring-Boot-Kafka/3_SERVE_ERROR.png)

Docker의 Kafka설정에서 bootstrap-servers: kafka:9092로 지정했습니다. kafka라는 호스트명에 대한 별도 설정이 없어 발생하는 것으로, Mac 기준 /etc/hosts에 127.0.0.1 kafka 추가해 로컬에 대한 호스트명을 지정해줍니다.

![Spring Boot Kafka 서버 연동 Host 설정]({{site.url}}/assets/img/Spring-Boot-Kafka/4_SERVE_ERROR_SOLVE.png)

**StockProducer.java**

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class StockProducer {

  private static final String REQUEST_TOPIC = "stock-valid-request";

  private final KafkaTemplate<String, Map<String, Object>> kafkaTemplate;
  
  private final ObjectMapper objectMapper;

  public void sendMessage(final String key, final ProductCountDto productCountDto) {
    kafkaTemplate.send(
        new ProducerRecord<>(
            REQUEST_TOPIC, //Topic name
            0, //Partition
            LocalDateTime.now().atZone(ZoneId.of("Asia/Seoul")).toInstant().toEpochMilli(), //Timestamp
            key, //Message Key
            objectMapper.convertValue(productCountDto, Map.class) //Message Value 
        )
    );
  }
}
```

- ProducerRecord에 토픽명, 파티션 번호, 타임스탬프, 메시지 키, 메시지 값을 추가해 발송합니다.

**OrderService.java**

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class OrderService {

  private final OrderRepository orderRepository;
  
  private final StockProducer stockProducer;
  
  /**
   * 1. 주문 이력 데이터 베이스 저장
   * 2. Kafka 재고 확인 메세지 발송
   * @param orderDto
   * @return HttpStatus
   */
  @Transactional
  public HttpStatus order(final OrderDto orderDto) {
    String orderId = orderRepository.save(orderDto.toEntity()).getOrderId();
    stockProducer.sendMessage(
        "ORDER_" + orderDto.getProductId(),
        new ProductCountDto(
            orderId,
            orderDto.getProductId(),
            orderDto.getOrderCount(),
            LocalDateTime.now()
        )
    );

    return orderId != null ? HttpStatus.OK : HttpStatus.INTERNAL_SERVER_ERROR;
  }
}
```

### **제품 서비스(Consumer)**

**application.yml**

```yaml
...
spring:
  kafka:
    bootstrap-servers: kafka:9092
    consumer:
      group-id: order-product
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
```

**KafkaConsumerConfig.java**

```java
@Configuration
@RequiredArgsConstructor
public class KafkaConsumerConfig {

  @Value("${spring.kafka.bootstrap-servers}")
  private String bootstrapServers;
  
  @Value("${spring.kafka.consumer.auto-offset-reset}")
  private String autoOffsetReset;

  @Value("${spring.kafka.consumer.key-deserializer}")
  private String keyDeserializer;

  @Value("${spring.kafka.consumer.value-deserializer}")
  private String valueDeserializer;

  /**
   * Kafka Listener Container를 생성하고 구성하는 인터페이스로, Kafka Consumer의 동작을 제어하기 위해 사용됩니다.
   * setConsumerFactory
   *   - ConsumerFactory를 등록해줍니다.
   * setConcurrency
   *   - 여러 스레드에서 Kafka Consumer를 실행할 때 사용되는 스레드 수를 설정합니다.
   * setAckMode
   *   - Consumer가 메시지를 처리한 후에 Broker에게 해당 메시지의 처리를 어떻게 알리는지 설정합니다.
   
   * @return
   */
  @Bean
  public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, Map<String, Object>>> kafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, Map<String, Object>> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory()); //ConsumerFactory 지정
    factory.setConcurrency(1); //여러 스레드에서 Kafka Consumer를 실행할 때 사용되는 스레드 수를 설정
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
    return factory;
  }

  /**
   * Consumer가 받는 메시지 형식을 지정합니다. (역직렬화)
   * Key : StringDeserializer
   * Value : JsonDeserializer
   */
  @Bean
  public ConsumerFactory<String, Map<String, Object>> consumerFactory() {
    return new DefaultKafkaConsumerFactory<>(consumerConfig());
  }

  @Bean
  public Map<String, Object> consumerConfig() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, keyDeserializer);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, valueDeserializer);
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, autoOffsetReset);
    return props;
  }
}
```

- **AckMode** : 메시지를 처리한 후에 Broker에게 해당 메시지의 처리를 어떻게 알리는지 설정합니다. 높은 안정성을 요구하는 경우 **MANUAL** 모드를 고려할 수 있으며, 높은 처리 성능을 요구하는 경우 **NONE** 또는 **BATCH** 모드를 고려할 수 있습니다.
  - **AckMode.NONE** : Consumer는 메시지 처리를 Broker에게 알리지 않습니다. 이 모드에서는 메시지 손실의 가능성이 있습니다. 대신 처리 중에 발생하는 장애를 관리하고 다시 처리할 수 있는 방법을 고려해야 합니다.
  - **AckMode.RECORD** : Consumer는 개별적인 메시지가 처리될 때마다 Broker에게 ACK를 전송합니다. 이 모드에서는 메시지 손실의 가능성을 줄일 수 있지만, 처리 속도가 느려질 수 있습니다.
  - **AckMode.BATCH** : Consumer는 일괄 처리된 메시지의 그룹이 처리될 때마다 Broker에게 ACK를 전송합니다. 이 모드에서는 메시지 처리의 효율성을 높일 수 있지만, 일괄 처리되는 메시지의 크기와 처리 지연을 고려해야 합니다.
  - **AckMode.MANUAL** : Consumer는 메시지 처리 후에 명시적으로 ACK 또는 NACK를 전송합니다. 이 모드에서는 메시지 처리를 완료한 후에만 ACK를 보내므로 메시지 손실을 줄일 수 있습니다. 하지만 개발자가 ACK를 관리해야 하므로 처리 로직에 복잡성이 추가됩니다.

**StockConsumerListener.java**

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class StockConsumerListener implements AcknowledgingMessageListener<String, Map<String, Object>> {

  private static final String REQUEST_TOPIC = "stock-valid-request";

  private final ProductRepository productRepository;

  @KafkaListener(groupId = "order-product", topics = REQUEST_TOPIC, containerFactory = "kafkaListenerContainerFactory")
  public void onMessage(ConsumerRecord<String, Map<String, Object>> record, Acknowledgment ack) {
    Map<String, Object> map = record.value();
    try {
      if(map != null && map.get("productId") != null && map.get("orderCount") != null && (Integer) map.get("orderCount") > 0){
        Integer orderCount = (Integer) map.get("orderCount");
        /*
          Producer에서 발송한 값의 타입이 Long 이지만, 
          Consumer에서 값을 받을 때 Long으로 변환이 불가능하여 String변환 후 Long으로 변환
        */
        Long productId = Long.parseLong(String.valueOf(map.get("productId")));

        //Dirty Checking 재고량 감소
        Product product = productRepository.findById(productId).orElse(null);
        if (product != null && product.getProductCount() != null && product.getProductCount() >= orderCount) {
          product.updateProductCount(product.getProductCount() - orderCount);
        }
        ack.acknowledge(); //Offset Commit(커밋 시기를 수동으로 제어)
      }
    }
    catch (Exception e){
      e.printStackTrace();
    }
  }
}
```

- **@KafkaListener** : Kafka 토픽에서 메시지를 구독하여 처리할 메서드를 지정할 수 있습니다.
  - **groupId** : Kafka Consumer의 그룹 ID를 설정할 수 있습니다. 같은 그룹 ID를 가진 여러 Consumer가 같은 토픽의 파티션을 분배하여 메시지를 처리합니다.
  - **topics** : Kafka Consumer가 구독할 토픽을 지정합니다. 토픽 이름은 **`topics`** 속성을 통해 지정하며, 하나 이상의 토픽을 지정할 수 있습니다.

### **주문 서비스에서 주문 및 메시지 발송 - Producer 재고 확인 요청**

**로그 확인**

![Kafka Producer Test Log1]({{site.url}}/assets/img/Spring-Boot-Kafka/5_KAFKA_PRODUCER_TEST_LOG1.png)

메시지를 발송 하면 이와 같이 Producer Config의 설정 값들을 확인할 수 있습니다.  

![Kafka Producer Test Log2]({{site.url}}/assets/img/Spring-Boot-Kafka/6_KAFKA_PRODUCEER_TEST_LOG2.png)
![Kafka Producer Test Log3]({{site.url}}/assets/img/Spring-Boot-Kafka/7_KAFKA_PRODUCEER_TEST_LOG3.png)
![Kafka Producer Test Log4]({{site.url}}/assets/img/Spring-Boot-Kafka/8_KAFKA_PRODUCER_TEST_LOG4.png)

메시지 발송 과정에서 초기 트랜잭션 설정을 시작으로 발송전 Interceptor에 onSend에 설정한 로그를 확인할 수 있습니다. 트랜잭션이 준비상태가 되면 메시지를 발송 후 Interceptor의 onAcknowledgement에 설정한 로그 및 Listener의 onSuccess의 로그를 확인할 수 있습니다.

**Kafka UI 확인**

![Kafka Producer Test UI]({{site.url}}/assets/img/Spring-Boot-Kafka/9_KAFKA_PRODUCER_TEST_UI.png)

Topic을 선택하고 Messages를 확인하면 위와 같이 발송한 메시지를 확인할 수 있습니다.

### **상품 서비스에서 재고 확인 메시지 구독 및 처리 - Consumer 재고 확인 및 감소**

![Kafka Consumer Test Log1]({{site.url}}/assets/img/Spring-Boot-Kafka/11_KAFKA_CONSUMER_TEST_LOG1.png)
![Kafka Consumer Test Log2]({{site.url}}/assets/img/Spring-Boot-Kafka/12_KAFKA_CONSUMER_TEST_LOG2.png)

Consumer에서 로그를 확인하면 위와 같이 재고 확인 및 감소 비즈니스 로직을 실행하는 것을 볼 수 있습니다.

## **Kafka Transaction 테스트**

Kafka Producer에서 메시지 발송 후 이후 로직에서 오류가 발생했다는 가정하에 아래와 같이 강제로 예외를 발생시켜보겠습니다.

**OrderService.java**

```java
@Transactional
public HttpStatus order(final OrderDto orderDto) {
  String orderId = orderRepository.save(orderDto.toEntity()).getOrderId();
  stockProducer.sendMessage(
      "ORDER_" + orderDto.getProductId(),
      new ProductCountDto(
          orderId,
          orderDto.getProductId(),
          orderDto.getOrderCount(),
          LocalDateTime.now()
      )
  );
  
  throw new RuntimeException("런타임 예외");
  //return orderId != null ? HttpStatus.OK : HttpStatus.INTERNAL_SERVER_ERROR;
}
```

![Kafka Producer Transaction Test Log1]({{site.url}}/assets/img/Spring-Boot-Kafka/13_KAFKA_PRODUCER_TRANSACTION_TEST_LOG1.png)

**RuntimeException**을 발생시켰지만 예상과 다르게 메시지는 정상적으로 발송된 것을 볼 수 있습니다. 롤백이 되지 않고 Kafka UI에도 데이터가 확인됩니다.

![Kafka Producer Transaction Test UI]({{site.url}}/assets/img/Spring-Boot-Kafka/14_KAFKA_PRODUCER_TRANSACTION_TEST_UI.png)

Producer 트랜잭션 롤백 처리를 위해 설정에 아래와 같이 트랜잭션 설정 값을 추가합니다.

**KafkaProducerConfig.java**

```java

...

@Value("${spring.kafka.producer.transactional.id}")
private String transactionalId;

@Bean
public ProducerFactory<String, Map<String, Object>> producerFactory() {
  Map<String, Object> configs = new HashMap<>();
  configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
  configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, keySerializer);
  configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, valueSerializer);
  
  //추가
  configs.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, transactionalId);  //Producer factory does not support transactions 오류 발생 해결법

  DefaultKafkaProducerFactory<String, Map<String, Object>> producerFactory = new DefaultKafkaProducerFactory<>(configs);
  return producerFactory;
}

@Bean
public KafkaTemplate<String, Map<String, Object>> kafkaTemplate() {
  KafkaTemplate<String, Map<String, Object>> kafkaTemplate = new KafkaTemplate<>(producerFactory());
  kafkaTemplate.setProducerInterceptor(producerInterceptor);
  kafkaTemplate.setProducerListener(producerListener);
  
  //추가
  kafkaTemplate.setTransactionIdPrefix(transactionalId+"-tx-");
  
  return kafkaTemplate;
}
```

- **TRANSACTIONAL_ID_CONFIG** : Producer에 트랜잭션 지원을 위해 설정합니다. 하나 이상의 메시지를 안전하게 전송하고, 모든 메시지가 성공적으로 처리되거나 실패한 경우 롤백하기 위해 설정합니다.
- **setTransactionIdPrefix** : 트랜잭션 ID의 접두사를 설정하는 데 사용됩니다. 이를 통해 Kafka Producer는 여러 트랜잭션을 구분하기 위해 트랜잭션 ID에 접두사를 추가할 수 있습니다.

![Kafka Producer Transaction Test Log2]({{site.url}}/assets/img/Spring-Boot-Kafka/15_KAFKA_PRODUCER_TRANSACTION_TEST_LOG2.png)
![Kafka Producer Transaction Test Log3]({{site.url}}/assets/img/Spring-Boot-Kafka/16_KAFKA_PRODUCER_TRANSACTION_TEST_LOG3.png)

설정 후 다시 테스트를 하게되면 이번에는 RuntimeException 발생이 되면서 트랜잭션이 Aborting 되면서 Commit 상태가 False 인것을 볼 수 있으며,  Listener에서 onError가 호출되면서 정상적으로 오류가 발생한 것을 확인할 수 있습니다. Kafka UI에서도 오류 발생으로 인해 앞에서 테스트한 1개 메시지 외 다른 메시지가 추가되지 않는 것을 볼 수 있습니다.

![Kafka Producer Transaction Test UI2]({{site.url}}/assets/img/Spring-Boot-Kafka/17_KAFKA_PRODUCER_TRANSACTION_TEST_UI2.png)

## **상품 서비스(Consumer)에서 재고 확인 후 주문 서비스(Producer)에 응답 및 처리**

재고 확인에 대한 응답 처리를 하기위한 과정은 상품 서비스에서 Broker로 다시 메시지를 발송하여 주문 서비스에서 Consumer로 응답 메시지를 읽어 응답에 대한 로직을 처리할 수 있도록 합니다.

### **상품 서비스에 응답에 대한 Producer 추가**

**application.yml**

```java
...
spring:
  kafka:
    bootstrap-servers: kafka:9092
    consumer:
      group-id: order-product
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
    # Producer 추가
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      transactional:
        id: kafka-product-id

```

**KafkaProducerConfig.java 생성**

KafkaProducerConfig의 경우 앞의 Producer와 동일합니다. JsonSerializer의 경우 주문 시 저장된 데이터에 대해 재고가 없는 경우에 해당 주문 건에 대한 삭제 처리를 위해 주문 정보를 Broker에 저장하기 위함입니다.

```java
@Configuration
@RequiredArgsConstructor
public class KafkaProducerConfig {

  @Value("${spring.kafka.bootstrap-servers}")
  private String bootstrapServers;

  @Value("${spring.kafka.producer.key-serializer}")
  private String keySerializer;

  @Value("${spring.kafka.producer.value-serializer}")
  private String valueSerializer;

  @Value("${spring.kafka.producer.transactional.id}")
  private String transactionalId;

  private final KafkaProducerInterceptor producerInterceptor;
  private final KafkaProducerListener producerListener;

  /**
   * BOOTSTRAP_SERVERS_CONFIG
   *   - Kafka의 기본 서버는 Bootstrap 서버로 Docker 환경에서 설정한 Kafka 서버 정보를 지정합니다.
   * KEY_SERIALIZER_CLASS_CONFIG
   *   - 토픽의 메시지 Key 타입을 지정해줍니다.
   * VALUE_SERIALIZER_CLASS_CONFIG
   *   - 토픽의 메시지 Value 타입을 지정해줍니다.
   * TRANSACTIONAL_ID_CONFIG
   *   - Producer에 트랜잭션 지원을 위해 설정합니다.
   *   - 하나 이상의 메시지를 안전하게 전송하고, 모든 메시지가 성공적으로 처리되거나 실패한 경우 롤백하기 위해 설정합니다.
   */
  @Bean
  public ProducerFactory<String, Map<String, Object>> producerFactory() {
    Map<String, Object> configs = new HashMap<>();
    configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
    configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, keySerializer);
    configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, valueSerializer);
    
    configs.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, transactionalId);  //Producer factory does not support transactions 오류 발생 해결법

    DefaultKafkaProducerFactory<String, Map<String, Object>> producerFactory = new DefaultKafkaProducerFactory<>(configs);
    return producerFactory;
  }

  /**
   * setProducerInterceptor
   *   - 메시지 발송 전/후에 대한 확인을 위해 Interceptor를 등록합니다.
   * setProducerListener
   *   - 메시지 발송 결과 확인을 위해 Listener를 등록합니다.
   * setTransactionIdPrefix
   *   - 트랜잭션 ID의 접두사를 설정하는 데 사용됩니다.
   *   - 이를 통해 Kafka Producer는 여러 트랜잭션을 구분하기 위해 트랜잭션 ID에 접두사를 추가할 수 있습니다.
   */
  @Bean
  public KafkaTemplate<String, Map<String, Object>> kafkaTemplate() {
    KafkaTemplate<String, Map<String, Object>> kafkaTemplate = new KafkaTemplate<>(producerFactory());
    kafkaTemplate.setProducerInterceptor(producerInterceptor);
    kafkaTemplate.setProducerListener(producerListener);
    kafkaTemplate.setTransactionIdPrefix(transactionalId+"-tx-");
    return kafkaTemplate;
  }
}
```

**StockConsumerListener.java 수정**

```java

//Response Topic 추가
private static final String RESPONSE_TOPIC = "stock-valid-response";

//KafkaTemplate 주입
private final KafkaTemplate<String, Map<String, Object>> kafkaTemplate;

//ObjectMapper 주입
private final ObjectMapper objectMapper;

@Transactional
@KafkaListener(groupId = "order-product", topics = REQUEST_TOPIC, containerFactory = "kafkaListenerContainerFactory")
public void onMessage(ConsumerRecord<String, Map<String, Object>> record, Acknowledgment ack) {
  log.info("Order Product Consumer receiveProductCheckResponse -> {}" + record.value());
  Map<String, Object> map = record.value();
  //추가
  ResponseDto response = ResponseDto.toResponse(HttpStatus.INTERNAL_SERVER_ERROR, map);
  try {
    if(map != null && map.get("orderCount") != null && map.get("productId") != null && (Integer) map.get("orderCount") > 0){
      Integer orderCount = (Integer) map.get("orderCount");
      Long productId = Long.parseLong(String.valueOf(map.get("productId")));
      Product product = productRepository.findById(productId).orElse(null);

      if (product != null && product.getProductCount() != null && product.getProductCount() >= orderCount) {
        product.updateProductCount(product.getProductCount() - orderCount);
        //추가
        response = ResponseDto.toResponse(HttpStatus.OK, map);
      }
    }
    ack.acknowledge(); //Offset Commit(커밋 시기를 수동으로 제어)
  }
  catch (Exception e){
    e.printStackTrace();
  }
  //추가 - 응답 메시지 전송
  kafkaTemplate.send(RESPONSE_TOPIC, objectMapper.convertValue(response, Map.class));
}
```

**ResponseDto.java**

```java
@Getter
@Setter
@AllArgsConstructor
public class ResponseDto {

  private int statusCode;

  private HttpStatus.Series series;

  private String reason;

  private Map<String, Object> record;

  public static ResponseDto toResponse(HttpStatus status, Map<String, Object> record){
    return new ResponseDto(status.value(), status.series(), status.getReasonPhrase(), record);
  }

  @Override
  public String toString() {
    return "ResponseDto{" +
        "statusCode=" + statusCode +
        ", series=" + series +
        ", reason='" + reason + '\'' +
        ", record=" + record +
        '}';
  }
}
```

### **주문 서비스에 응답에 대한 Consumer 추가**

**application.yml**

```yaml
...

spring:
  kafka:
    bootstrap-servers: kafka:9092
    ...
    consumer:
      group-id: order-product
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
```

**KafkaConsumerConfig.java 생성**

```java
@Configuration
@RequiredArgsConstructor
public class KafkaConsumerConfig {

  @Value("${spring.kafka.bootstrap-servers}")
  private String bootstrapServers;

  @Value("${spring.kafka.consumer.auto-offset-reset}")
  private String autoOffsetReset;

  @Value("${spring.kafka.consumer.key-deserializer}")
  private String keyDeserializer;

  @Value("${spring.kafka.consumer.value-deserializer}")
  private String valueDeserializer;

  @Bean
  public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, Map<String, Object>>> kafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, Map<String, Object>> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    factory.setConcurrency(2);
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
    return factory;
  }

  @Bean
  public ConsumerFactory<String, Map<String, Object>> consumerFactory() {
    return new DefaultKafkaConsumerFactory<>(consumerConfig());
  }

  @Bean
  public Map<String, Object> consumerConfig() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
		props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, keyDeserializer);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, valueDeserializer);
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, autoOffsetReset);
    props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, READ_COMMITTED.toString().toLowerCase(Locale.ROOT));
    return props;
  }
}
```

**StockConsumerListener.java 생성**

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class StockConsumerListener implements AcknowledgingMessageListener<String, Map<String, Object>> {

  private static final String RESPONSE_TOPIC = "stock-valid-response";

  private final OrderRepository orderRepository;

  @KafkaListener(groupId = "order-product", topics = RESPONSE_TOPIC, containerFactory = "kafkaListenerContainerFactory")
  public void stockResponse(ConsumerRecord<String, Map<String, Object>> record){
    try {
      Map<String, Object> map = record.value();
      if(map == null || map.get("statusCode") == null || map.get("record") == null) {
        return;
      }

      int statusCode = (Integer)map.get("statusCode");
      Map<String, Object> data = (Map<String, Object>)map.get("record");
      log.info("응답 코드 ->  {} " + statusCode);
      log.info("응답 데이터 ->  {} " + data);
      if (statusCode != 200 && data.get("orderId") != null) {
        orderRepository.deleteById((String) data.get("orderId"));
      }
      ack.acknowledge(); //Offset Commit(커밋 시기를 수동으로 제어)
    }
    catch (Exception e){
      e.printStackTrace();
    }
  }
}
```

- 정상 처리되지 않은 응답의 경우 주문 처리 과정에서 들어간 주문 데이터를 삭제합니다.

### **재고 확인에 따른 주문 후 처리 테스트**

**재고 데이터**

![제품 데이터]({{site.url}}/assets/img/Spring-Boot-Kafka/18_PRODUCT_DATA.png)

**주문 처리 및 정상 재고 처리 후 데이터와 로그 확인**

![제품 데이터2]({{site.url}}/assets/img/Spring-Boot-Kafka/19_PRODUCT_DATA2.png)
![재고 처리 후 주문 서비스 로그]({{site.url}}/assets/img/Spring-Boot-Kafka/20_KAFKA_CONSUMER_SUCCESS_LOG.png)
![재고 처리 후 Kafka UI 응답 데이터 확인]({{site.url}}/assets/img/Spring-Boot-Kafka/21_KAFKA_RESPONSE_UI.png)

2개 제품을 주문 했다는 가정하에 PRODUCT_COUNT가 감소되었고 Kafka UI에 응답 토픽을 확인하면 상품 서비스에서 처리 후 응답한 데이터를 확인할 수 있습니다.

주문 서비스의 Consumer에서 응답 토픽의 데이터를 읽어 정상 응답에 따라 이후에 삭제 과정은 진행되지 않았습니다. 

이 상태에서 다시 동일한 제품에 대해 2건의 주문을 요청해보겠습니다.

![재고 처리 실패 로그1]({{site.url}}/assets/img/Spring-Boot-Kafka/22_KAFKA_RESPONSE_FAIL_LOG1.png)
![재고 처리 실패 로그2]({{site.url}}/assets/img/Spring-Boot-Kafka/23_KAFKA_RESPONSE_FAIL_LOG2.png)
![재고 처리 후 Kafka UI 응답 실패 데이터 확인]({{site.url}}/assets/img/Spring-Boot-Kafka/24_KAFKA_RESPONSE_FAIL_UI2.png)

로그를 확인하면 주문 서비스 Consumer에서 토픽으로 부터 응답 데이터를 읽어 응답 코드에 따라 기존 주문건 삭제처리가 된 것을 확인하실 수 있습니다.

## **트러블 슈팅**

### **문제**

Kafka에서 Consumer를 테스트 과정에서 만약 Consumer 서비스에서 장애가 발생해 서버가 다운되었다는 가정하에 Consumer 서비스를 재시작했는데 해당 Topic의 메시지를 다시 읽어 실행했다.

![Kafka UI]({{site.url}}/assets/img/Spring-Boot-Kafka/25_KAFKA-UI.png)

1개 메시지가 현재 요청된 상태이고 Consumer는 이미 해당 메시지를 처리했지만, 재시작되면서 다시 요청을 읽어 처리되었다.

### **원인 파악**

Consumer의 경우 메시지를 구독하는 기준은 offset이기에 offset에 관련 내용에서 아래 2가지 경우에 대해 생각했다.

- **auto.offset.reset 확인** : auto.offset.reset 설정의 경우 offset 오류 (Consumer가 Topic의 Offset 정보를 가지고 있지 않음)의 경우 발생한다.
- **Consumer Lag 확인** : Consumer Lag는 Producer의 전송 속도가 Consumer가 구독하여 처리하는 속도보다 빠르다면 Consumer가 마지막으로 읽은 offset과 Producer가 마지막으로 넣은 offset의 차이이다.

### **원인 확인**

위 2가지 예상 원인에 대해 **`Kafka UI`{: .text-blue}** 혹은 **`Burrow`{: .text-blue}**로 Offset의 여부 확인

**Kafka UI**

![Kafka Offset Test UI]({{site.url}}/assets/img/Spring-Boot-Kafka/26_OFFSET_TEST_KAFKA-UI.png)

→ 재고 확인 요청에 대한 토픽(stock-valid-request)의 경우 Consumer Lag의 값이 0으로 확인된다.

**Burrow Data**

![Burrow Consumer Data]({{site.url}}/assets/img/Spring-Boot-Kafka/27_BURROW_CONSUMMER_JSON_DATA.png)

→ Burrow에서 Consumer를 확인했을 때 **`stock-valid-request`{: .text-blue}** 토픽 정보가 없음

→ Consumer Lag의 경우 Offset이 있어야하기에 연관이 없다고 판단

**auto.offset.reset** 현재 설정되어있는 값은 **`earliest`{: .text-blue}**이기에 해당 오류 발생시 처음부터 다시 읽어서 발생하는 것으로 확인되어 왜 Offset이 없는지에 대해 다시 확인이 필요하다.

### **왜?**

**AckMode 설정으로 인한 Offset 커밋 누락**

원인에 대해 찾다보니 AckMode와 Offset이 어떤 연관이 있는지 몰랐었다… 

AckMode에 정리한 내용을 보면서  **`MANUAL_IMMEDIATE`{: .text-blue}** 의 설정은  **`AcknowledgingMessageListener`{: .text-blue}**를 통해 **`acknowledge()`{: .text-blue}** 메서드를 호출한 즉시 커밋한다. 관련된 부분들을 따라가다보면 **KafaMessageListenerContainer.class**에 아래 메서드를 볼 수 있다.

```java
private void ackImmediate(ConsumerRecord<K, V> cRecord) {
  Map<TopicPartition, OffsetAndMetadata> commits = Collections.singletonMap(new TopicPartition(cRecord.topic(), cRecord.partition()), this.createOffsetAndMetadata(cRecord.offset() + 1L));
  this.commitLogger.log(() -> {
    return "Committing: " + commits;
  });
  if (this.producer != null) {
    this.doSendOffsets(this.producer, commits);
  } else if (this.syncCommits) {
    this.commitSync(commits);
  } else {
    this.commitAsync(commits);
  }
}
```

commits 객체에서 offset 메타데이터를 설정하고 해당 설정에 대해서 Offset을 증가시키는 것을 볼 수 있다. 결국, **`acknowledge`{: .text-blue}**를 호출하지 않아 발생했다.

### **문제의 Consumer 코드**

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class StockConsumerListener {

  private static final String REQUEST_TOPIC = "stock-valid-request";

  private final ProductRepository productRepository;

  private final KafkaTemplate<String, Map<String, Object>> kafkaTemplate;

  private final ObjectMapper objectMapper;


  @Transactional
  @KafkaListener(groupId = "order-product", topics = REQUEST_TOPIC, containerFactory = "kafkaListenerContainerFactory")
  public void onMessage(ConsumerRecord<String, Map<String, Object>> record) {
    Map<String, Object> map = record.value();
    try {
      if(map != null && map.get("productId") != null && map.get("orderCount") != null && (Integer) map.get("orderCount") > 0){
        Integer orderCount = (Integer) map.get("orderCount");
        /*
          Producer에서 발송한 값의 타입이 Long 이지만, 
          Consumer에서 값을 받을 때 Long으로 변환이 불가능하여 String변환 후 Long으로 변환
        */
        Long productId = Long.parseLong(String.valueOf(map.get("productId")));

        //Dirty Checking 재고량 감소
        Product product = productRepository.findById(productId).orElse(null);
        if (product != null && product.getProductCount() != null && product.getProductCount() >= orderCount) {
          product.updateProductCount(product.getProductCount() - orderCount);
        }
      }
    }
    catch (Exception e){
      e.printStackTrace();
    }
    kafkaTemplate.send(RESPONSE_TOPIC, objectMapper.convertValue(response, Map.class));
  }
}
```

### **처리 후 Consumer 코드**

```java
@Slf4j
@Service
@Transactional
@RequiredArgsConstructor
public class StockConsumerListener implements AcknowledgingMessageListener<String, Map<String, Object>> {

  private static final String REQUEST_TOPIC = "stock-valid-request";

  private final ProductRepository productRepository;

  private final KafkaTemplate<String, Map<String, Object>> kafkaTemplate;

  private final ObjectMapper objectMapper;

  @Transactional
  @KafkaListener(groupId = "order-product", topics = REQUEST_TOPIC, containerFactory = "kafkaListenerContainerFactory")
  public void onMessage(ConsumerRecord<String, Map<String, Object>> record, Acknowledgment ack) {
    Map<String, Object> map = record.value();
    try {
      if(map != null && map.get("productId") != null && map.get("orderCount") != null && (Integer) map.get("orderCount") > 0){
        Integer orderCount = (Integer) map.get("orderCount");
        /*
          Producer에서 발송한 값의 타입이 Long 이지만, 
          Consumer에서 값을 받을 때 Long으로 변환이 불가능하여 String변환 후 Long으로 변환
        */
        Long productId = Long.parseLong(String.valueOf(map.get("productId")));

        //Dirty Checking 재고량 감소
        Product product = productRepository.findById(productId).orElse(null);
        if (product != null && product.getProductCount() != null && product.getProductCount() >= orderCount) {
          product.updateProductCount(product.getProductCount() - orderCount);
        }
        ack.acknowledge();
      }
    }
    catch (Exception e){
      e.printStackTrace();
    }
    kafkaTemplate.send(RESPONSE_TOPIC, objectMapper.convertValue(response, Map.class));
  }
}
```

### **처리 결과**

![Burrow Consumer Ok Data]({{site.url}}/assets/img/Spring-Boot-Kafka/28_BURROW_CONSUMER_OK_DATA.png)

stock-valid-request 토픽과 Offset 정보가 정상적으로 확인된다.

Consumer 서비스를 재시작해도 더이상 전체 메시지를 구독하여 처리하지 않는다.

### **추가 확인 사항**

Spring Kafka의 Consumer 설정 중 Commit과 관련된 설정이 있다.

- enable.auto.commit : Auto-Commit 여부 **(default : true)**
- auto.commit.interval.ms : Auto-Commit 사용 시 Commit을 수행할 시간(ms) **(defaut : 5000)**

자동 커밋이 기본으로 설정이 되어이지만 **`MANUAL_IMMEDIATE`{: .text-blue}** 의 경우 위 옵션이 동작하지 않고 수동으로 커밋 메서드를 호출해야한다.