---
title: Spring Boot + MyBatis Transaction
# author: dowonl2e
date: 2024-02-21 12:30:00 +0800
categories: [Blogging, Tutorial]
tags: [Transaction]
pin: true
img_path: "/posts/20240221"
---

## **Intro**

회사 생활 초반에 자주 발생한 문제였던 Transaction에 대해서 다루고자 한다. 당시에는 JavaBean 기반의 프로젝트에서 Spring으로 넘어간지 오래되지 않았을 때이다. 트랜잭션는 처리가 되어있었으나, 시간이 지날수록 DB Pool 이슈와 롤백 이슈가 있는 시스템이 있어 확인했을 때 잘못된 사용으로 개선하기도 했었다.

Spring Boot는 아니었지만, 당시에 공부하고 정리했던 Transaction을 Spring Boot를 이용하여 비슷한 환경에서 트랜잭션을 어떻한 방식으로 처리했었는지를 작성하고 다음으로 JPA 환경에서도 Transaction를 처리하는 방식을 정리해보려 한다.

## **개발 환경**

- IDE : IntelliJ IDEA (Community Edition)
- JDK : Java 17
- Framework : Spring Boot 3, MyBatis
- DB : MySQL 8.0
- DBCP : Apache Common DBCP2

### **Spring Project 생성**

Spring Initializr 웹 도구 사이트[(https://spring.io)](https://start.spring.io/)에서 Spring Project를 생성해줍니다.

**Dependency 추가**

- Spring Web
- JDBC API
- MySQL Driver
- MyBatis Framework
- Lombok

### **Apache Common DBCP2 추가**

build.gradle dependency에 Apache Common DBCP2을 추가해준다. Spring Boot 2.0 부터 디폴트 DBCP는 HikariCP가 되었지만, 당시의 비슷한 상황으로 테스트해보기 위해 Apache Commons DBCP2를 사용하겠습니다.

`implementation 'org.apache.commons:commons-dbcp2:2.11.0'`

### **DataSource 옵션 설정**

**application.yml**

```yaml
# DataSource 옵션
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://{DB호스트명}:{DB포트}/{DB명}
    username: { DB 계정명 }
    password: { DB 비밀번호 }

#MyBatis 옵션
mybatis:
  configuration:
    map-underscore-to-camel-case: true
```

- `mybatis.configuration.map-underscore-to-camel-case`은 MyBatis 설정시에 카멜 기법을 추가하기 위한 용도로 설정된 옵션

### **Apache Common DBCP2 DataSource 설정**

**DatabaseConfig.java**

```java
@Configuration
public class DatabaseConfig {

  @Bean
  public DataSource dataSource() {  // DataSource 객체 생성
    BasicDataSource dataSource = new BasicDataSource();

    dataSource.setDriverClassName(context.getEnvironment().getProperty("spring.datasource.driver-class-name"));
    dataSource.setUrl(context.getEnvironment().getProperty("spring.datasource.url"));
    dataSource.setUsername(context.getEnvironment().getProperty("spring.datasource.username"));
    dataSource.setPassword(context.getEnvironment().getProperty("spring.datasource.password"));

    return dataSource;
  }

  @Bean
  public TransactionManager transactionManager() {
    return new DataSourceTransactionManager(dataSource());
  }

}
```

- application.yml의 옵션에서 spring.datasource 옵션을 가져와 데이터 소스를 구성합니다.

### **MyBatis 설정**

MyBatis 구성 설정을 위해 DatabaseConfig에 아래 코드를 추가해줍니다.

```java
@Configuration
public class DatabaseConfig {

	//추가
  @Autowired
  private ApplicationContext context;

  ...

	//추가
  @Bean
  public SqlSessionFactory sqlSessionFactory() throws Exception {
    SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
    factoryBean.setDataSource(dataSource());
    factoryBean.setMapperLocations(context.getResources("classpath:/mappers/**/*Mapper.xml"));
    factoryBean.setTypeAliasesPackage("com.study.*");
    factoryBean.setConfiguration(mybatisConfig());
    return factoryBean.getObject();
  }

	//추가
  @Bean
  public SqlSessionTemplate sqlSession() throws Exception {
    return new SqlSessionTemplate(sqlSessionFactory());
  }
}
```

- ApplicationContext : 스프링 컨테이너 중 하나로 애플리케이션 리소스를 가져오기위해 사용
- sqlSessionFactory() : DB 커넥션, SQL 실행의 역할이며, SqlSessionFactoryBean을 통해 MyBatis와 Spring의 연동
  - setTypeAliasesPackage : mapper의 parameter, result 타입을 DTO에 설정된 alias로 사용하기 위해 설정

### **DatabaseConfig.java JUnit 테스트**

**Apache Common DBCP2 JUnit 테스트**

![DBCP2 JUnit 테스트]({{site.url}}/assets/img/transaction/1_DBCP2 Test Result.png)

**SqlSessionFactory JUnit 테스트**

![SqlSessionFactory JUnit 테스트]({{site.url}}/assets/img/transaction/2_SqlSessionFactory JUnit Test.png)

트랜잭션 테스트를 위한 설정이 완료되었습니다. 지금부터 트랜잭션 테스트를 해보겠습니다.

## **트랜잭션 테스트**

### **추가설정**

우선 트랜젝션 및 JDBC 로그 출력을 위해 아래 옵션을 추가합니다.

**application.yml**

```yaml
# 트랜젝션 로깅 설정
logging:
  level:
    org:
      springframework:
        jdbc: debug
        transaction:
          interceptor: trace
```

- 데이터 추가에 대한 트랜잭션 테스트를 위해 Service 클래스 파일의 구현 메서드에 아래와 같이 애너테이션을 추가해줍니다.

**BoardServiceImpl.java**

```java
@Override
@Transactional // => 추가
public void save(BoardDto boardDto) {
  ...
}
```

### **첫 번째 테스트**

![싱글 데이터추가 트랜잭션 로그.png]({{site.url}}/assets/img/transaction/3_Single Insert Transaction Log.png)
결과를 보면 Transactional 애너테이션을 추가함으로서 트랜잭션 시작과 완료되었다는 것을 확인 할 수 있다.

### **두 번째 테스트**

두 번째는 간단하게 Loop를 이용해서 10개의 데이터를 추가하는 경우를 보겠습니다.
![멀티 데이터추가 트랜잭션 로그.png]({{site.url}}/assets/img/transaction/4_Multi Insert Transaction Log.png)
이 경우도 정상적으로 트랜잭션이 시작되고 종료 되는 것을 볼 수 있다.

### **예외가 발생하면 어떻게 될까?**

개발을 하다보면 다양한 오류를 마주하게 됩니다. 예를 들어 SQLException, NullPointerException, IOException 등

다중 DB 트랜잭션의 경우 몇번의 데이터 처리가 되더라도 모두 정상처리가 되어야합니다. 하지만, 프로세스 진행 중 오류가 발생하면 앞에서 수행했던 것들은 롤백을 통해 원래의 데이터로 유지되어야합니다. 트랜잭션 4가지 특성 중 <span class="text-blue">**원자성(Atomicity)**</span>에 대한 내용입니다.

**1) NullPointerException 발생**

테스트를 위해 강제로 예외를 발생시키기는 코드를 추가해줍니다.
**BoardServiceImpl.java**

```java
@Override
@Transactional
public void multiSave() {
  for(int i = 0 ; i < 10 ; i++){

    if(i == 5) throw new NullPointerException(); // => 추가

    BoardDto boardDto = new BoardDto();
    boardDto.setTitle("트랜젝션 테스트 제목" + (i + 1));
    boardDto.setContents("트랜젝션 테스트 내용" + (i + 1));
    boardMapper.insertBoard(boardDto);

  }
}
```

**테스트 결과**
![NullPointerException 발생 결과.png]({{site.url}}/assets/img/transaction/5_NullPointerException Result.png)
![NullPointerException 발생 데이터 결과.png]({{site.url}}/assets/img/transaction/6_NullPointerException Data Result.png)
첫 5개의 데이터 추가 처리는 정상 프로세스이지만, 예외가 발생으로 롤백되는 것을 확인할 수 있다.

**2) IOException 발생**

```java
@Override
@Transactional
public void multiSave() throws Exception {
  for(int i = 0 ; i < 10 ; i++){

    if(i == 5) throw new IOException(); // => IOException으로 변경

    BoardDto boardDto = new BoardDto();
    boardDto.setTitle("트랜젝션 테스트 제목" + (i + 1));
    boardDto.setContents("트랜젝션 테스트 내용" + (i + 1));
    boardMapper.insertBoard(boardDto);
  }
}
```

![IOException 발생 결과.png]({{site.url}}/assets/img/transaction/7_IOException Result.png)
IOException은 NullPointerException와 다르게 롤백을 하지 않고 커밋되었습니다.

### 왜?

Transactional.java 파일을 보면 주석에 아래 내용이 있다.
![트랜잭션 롤백 주석 내용.png]({{site.url}}/assets/img/transaction/9_Transaction Rollback Description.png)
이 애너태이션에 설정한 롤백 규칙이 없으면 트랜잭션은 <span class="text-red">**RuntimeException, Error**</span>에 롤백되고 <span class="text-red">**Checked Exceptions**</span>은 해당되지 않는다라고 적혀있습니다.

Java의 오류를 크게 세 가지 유형으로 나뉩니다.

- Checked Exception
- Unchecked Exception
- Error
![Java 예외 종류]({{site.url}}/assets/img/transaction/10_Kind Of Exception.png)
<center><a href="https://marrrang.tistory.com/56" target="_blank">https://marrrang.tistory.com/56</a></center>
<br/>
UncheckedException은 런타임 때 발생하는 예외이다. NullPointerException는 코딩을 하다보면 특정 객체가 인스턴스화가 되지 않았음에도 컴파일시에는 오류로 확인하지 않는다. ArrayIndexOutOfBoundsException도 마찬가지이다.

반대로 CheckedException은 컴파일 중에 발생하는 예외이다. 테스트 코드에 IOException의 경우는 메서드 선언부에서 예외를 처리하거나 구현부에 try/catch로 감싸 처리를 해줘야한다.

> 롤백이 정상적으로 되지 않는 경우 발생한 예외가 어떤 오류에 포함되는지 확인할 필요가 있다.
> {: .prompt-warning }

### **CheckedException의 경우 롤백은 어떻게 할까?**

첫 번째 방법은 try/catch로 감싸 catch 절에서 강제로 롤백을 하는 것이다.

```java
TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
```

TransactionExecution.java파일에 setRollbackOnly 메서드를 보면 아래와 같다
![강제 롤백 설명]({{site.url}}/assets/img/transaction/11_Rollback Force Description.png)
트리거를 통한 예외 발생을 대안으로, 현 트랜잭션에 강제로 예외를 던져 트랜잭션 매니저에 롤백을 지시하는 내용으로 보인다. 실제로 구현부에서 RuntimeException를 상속받은 UnsupportedOperationException 예외를 던지고 있다.

**강제 롤백 테스트**

**BoardServiceImpl.java**

```java
@Override
@Transactional
public void multiSave() {
  try {
    for (int i = 0; i < 10; i++) {
      BoardDto boardDto = new BoardDto();
      boardDto.setTitle("트랜젝션 테스트 제목" + (i + 1));
      boardDto.setContents("트랜젝션 테스트 내용" + (i + 1));
      boardMapper.insertBoard(boardDto);

      if(i == 5) throw new IOException();

    }
  }
  catch (Exception e){
    TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
  }
}
```

**테스트 결과**
![강제 롤백 결과]({{site.url}}/assets/img/transaction/12_Rollback Force Result.png)

테스트할 메서드 구현부를 try/catch로 감싼 뒤 catch절에 강제 롤백 코드를 적용시켜 테스트해보면 IOException 발생시 롤백 요청을 받아 강제로 롤백이 진행 되는 것을 볼 수 있다.

### **다른 방법은 없을까?**

Transactional 애너테이션에 롤백 속성을 사용하는 방법이 있다.

**BoardServiceImpl.java**

```java
@Override
@Transactional(rollbackFor = {Exception.class}) // => rollbackFor 속성 추가
public void multiSave() throws Exception {
  for (int i = 0; i < 10; i++) {
    BoardDto boardDto = new BoardDto();
    boardDto.setTitle("트랜젝션 테스트 제목" + (i + 1));
    boardDto.setContents("트랜젝션 테스트 내용" + (i + 1));
    boardMapper.insertBoard(boardDto);

    if(i == 5) throw new IOException();

  }
}
```

**테스트 결과**
![rollbackFor 속성 사용 결과]({{site.url}}/assets/img/transaction/13_Transaction rollbackFor Attribute Result.png)
위 코드 적용 후 로그를 보게 되면 롤백이 처리 되는 것을 볼 수 있다.

롤백 속성은 Exception으로 사용할 수 있고, 아래와 같이 구현부에 발생할 세부 예외를 여러개 선언할 수도 있다.

```java
@Transactional(
    rollbackFor = {
        IOException.class,
        SQLException.class
    }
)
```

특정 예외가 발생하면 롤백이 되지 않도록 지정할 수 있는 속성도 있으니 필요에 따라 사용하면 될 것 같다.

```java
@Transactional(
    noRollbackFor = {
        DataFormatException.class,
    }
)
```

### **그 외 예외**

지정된 예외가 아닌 시스템에 별도로 필요한 예외를 만들어 해당 예외 발생시에 롤백이 되어야하는 상황도 있을 것이다. 이 경우 RuntimeException을 상속하는 하위 클래스를 생성하여 이용할 수 있다.

**CustomException.java**

```java
public class CustomException extends RuntimeException {

  public CustomException(String message) {
    super(message);
  }

}
```

**BoardServiceImpl.java**

```java
@Override
@Transactional(rollbackFor = { CustomException.class }) // => 예외 클래스 변경
public void multiSave() {
  for (int i = 0; i < 10; i++) {
    BoardDto boardDto = new BoardDto();

    if(i == 5) throw new CustomException("커스텀 예외 발생");

    boardDto.setTitle("트랜젝션 테스트 제목" + (i + 1));
    boardDto.setContents("트랜젝션 테스트 내용" + (i + 1));
    boardMapper.insertBoard(boardDto);
  }
}
```

**테스트 결과**
![커스텀예외를 통해 rollbackFor 속성 사용 결과]({{site.url}}/assets/img/transaction/14_Custom Exception rollbackFor Attribute Result.png)

## **정리**

- 선언적 트랜잭션 사용법
- 트랜잭션에서는 Unchecked Exception, Error는 Transactional 설정만으로 롤백 가능하며, CheckedException은 개발자가 try/catch절을 통한 강제 롤백 또는 Transactional의 롤백설정을 해주어야한다. (**참고로 try/catch를 남발하는 것 보다 커스텀화 하여 처리하는 것이 좋아보인다.**)
- 모든 오류 처리는 중요하지만, 트랜잭션의 롤백 기준에서 CheckedException은 개발과정에 별도 작업이 필요할 수 있기에, 정상 프로세스에 대한 로그 확인은 필수로 해야한다.

> Github : [https://github.com/dowonl2e/transaction](https://github.com/dowonl2e/transaction)
> {: .prompt-info }
