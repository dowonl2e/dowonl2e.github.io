---
title: 대용량 트래픽 경험해보기(7/14) - HikariCP에서 Auto Commit 이슈
# author: dowonl2e
date: 2024-11-15 14:00:00 +0800
categories: [대용량 트래픽]
tags: [대용량 트래픽, Spring Boot, HikariCP, nGrinder]
pin: true
---

시나리오 테스트를 진행하기 전에 이슈가 발생할 수 있는 경우가 어떤게 있을까 해서 비즈니스 로직 중 DB 엑세스가 어느정도 있으며 다수의 데이터를 조회해야하는 API에 대해서 테스트를 진행해보았다.

## **사전 테스트 - 여행정보 현황 조회**

- VUser : 30
- Test Time : 10m

![nGrinder Prev Test Result]({{site.url}}/assets/img/High-Volume-Traffic-7/1-PREV_TEST_RESULT.png)

시나리오 테스트 전이기에 TPS와 Mean Test Time, Error 발생률 그리고 API 서버 부하로 인한 리소스 사용률은 시나리오 테스트에서 진행하기로 했다. 사전 테스트를 진행하면서 위와 같은 결과를 확인했다. 하지만 테스트 과정에서 문제가 될 수 있다고 판단한 부분이 있었다.

### **트러블 슈팅**

#### **문제 발생**

RDS에 Long Query Time을 1초로 설정하고 Slow Query 로그를 확인했을 때 아래와 같이 **`set auto_commit = 0 or 1`** 쿼리가 반복적으로 실행되고 있는 것을 확인했다. 

테스트 당시의 RDS 스팩은 부하가 발생할 수 있지만, Auto Commit 쿼리가 왜 발생하는지가 의아했고 반복적으로 실행되는 것에 부하가 걸리는 것에 영향을 미칠 수 있다고 생각했다.

![Auto Commit Issue]({{site.url}}/assets/img/High-Volume-Traffic-7/2-AUTO_COMMIT_ISSUE.png)

#### **원인 추론 및 분석**

Spring Boot에서 HikariCP에서의 set auto_commit에 대해 찾아보면서 Connection 로그를 다시 확인해보았다.

![Auto Commit Log]({{site.url}}/assets/img/High-Volume-Traffic-7/3-AUTO_COMMIT_LOG.png)

로그에서 **`Auto Commit False → Execute SQL → Connection Commit → Auto Commit True`{: .text-blue}** 순으로 로직이 로그를 확인할 수 있었으며 Auto Commit 설정으로 **`set auto_commit = 0 or 1`** 쿼리가 실행되고 있었다.

##### **상세 로그를 확인**

> TRACE org.hibernate.resource.jdbc.internal.AbstractLogicalConnectionImplementor.begin:line.71 - Preparing to begin transaction via JDBC Connection.setAutoCommit(false)
>
> DEBUG jdbc.audit.methodReturned:line.171 - 1. Connection.setAutoCommit(false) returned   com.zaxxer.hikari.pool.ProxyConnection.setAutoCommit(ProxyConnection.java:402)
>
> TRACE org.hibernate.resource.jdbc.internal.AbstractLogicalConnectionImplementor.begin:line.73 - Transaction begun via JDBC Connection.setAutoCommit(false)

위 로그에서 Auto Commmit 관련 클래스는 **LogicalConnectionManagedImpl**, **AbstractLogicalConnectionImplementor**이다.

##### **LogicalConnectionManagedImpl.begin()**

![LogicalConnectionManagedImpl begin Method]({{site.url}}/assets/img/High-Volume-Traffic-7/4-LogicalConnectionManagedImpl_begin.png)

##### **AbstractLogicalConnectionImplementor.begin()**

![AbstractLogicalConnectionImplementor begin Method]({{site.url}}/assets/img/High-Volume-Traffic-7/5-AbstractLogicalConnectionImplementor_begin.png)

**providerDisablesAutoCommit**값에 따라 Auto Commit을 설정하게 되는데 해당 필드는 **`hibernate.connection.provider_disables_autocommit`{: .text-blue}**이다.

더 자세한 내용을 확인하기위해 **HibernateJpaConfiguration** 파일에서 아래의 두 메서드를 보면

![HibernateJpaConfiguration resetConnection Method]({{site.url}}/assets/img/High-Volume-Traffic-7/6-Hibernate_Provider_Disables_Autocommit.png)

DataSourcePoolMetadata에서 Default Auto Commit 값을 이용하는데 이는 **`spring.datasource.hikari.auto-commit`{: .text-blue}**이다. **`true`**로 설정하면 **`hibernate.connection.provider_disables_autocommit(providerDisablesAutoCommit)`{: .text-blue}**는 **`false`**가 되어 **setAutoCommit(false)** 코드가 실행된다.

##### **AbstractLogicalConnectionImplementor.resetConnection()**

그리고 **`initiallyAutoCommit(초기 Auto Commit 설정 값)`{: .text-blue}**이 **`true`**이기에 비즈니스 로직을 종료한 후에 **setAutoCommit(true)** 코드가 실행된다.

![AbstractLogicalConnectionImplementor resetConnection Method]({{site.url}}/assets/img/High-Volume-Traffic-7/7-AbstractLogicalConnectionImplementor_resetConnection.png)

정리하면, HikariCP에서 Auto Commit을 활성화(default)할 경우 비즈니스 로직 전/후 **`set auto_commit=0 or 1`** SQL이 실행된다.

#### **해결방안**

application 설정에서 HikariCP에 **`auto-commit: false`{: .text-blue}** 옵션을 추가한다.

```yaml
# application.yml
spring: 
  datasource: 
    hikari: 
      auto-commit: false #추가
```

위 설정 후 로그를 다시 확인해보면 **`Connection 획득 → SQL 실행 → Connection Commit`{: .text-blue}**순으로 로그가 출력되며 SQL 또한 실행되지 않았다.

Auto Commit에 대한 이슈를 처리하면서 성능 향상에도 영향이 있다고하여 다시 테스트를 해보았을 때 아래와 같은 결과를 확인할 수 있었다.

![Auto Commit Trouble Shooting result]({{site.url}}/assets/img/High-Volume-Traffic-7/8-Touble_Shooting_Result.png)

- **TPS/Peak TPS** : 142/173 → 298/348 **`약 2배 상승`{: .text-blue}**
- **Mean Test Time** : 205 → 100 **`약 2배 감소`{: .text-blue}**
- **Executed Tests** : 84,871 → 174,679 **`약 2배 증가`{: .text-blue}**

## **References**

> [[HikariCP] set autocommit 개선을 통한 성능 최적화](https://velog.io/@haron/Spring-set-autocommit-%EA%B0%9C%EC%84%A0%EC%9D%84-%ED%86%B5%ED%95%9C-%EC%84%B1%EB%8A%A5-%EC%B5%9C%EC%A0%81%ED%99%94%EB%A5%BC-%ED%95%B4%EB%B3%B4%EC%9E%90){:target="\_blank"}
