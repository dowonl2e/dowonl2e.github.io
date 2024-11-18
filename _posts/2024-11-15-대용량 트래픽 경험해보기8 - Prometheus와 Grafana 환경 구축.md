---
title: 대용량 트래픽 경험해보기(8/10) - Prometheus & Grafana 환경 구축
# author: dowonl2e
date: 2024-11-18 13:00:00 +0800
categories: [대용량 트래픽]
tags: [대용량 트래픽, Prometheus, Grafana, Monitoring]
pin: true
---

시나리오 테스트 전 게별 API를 테스트하면서 사용한 모니터링 도구로 **`Spring Boot Admin`**, **`Visual VM`**을 이용했다.

**Spring Boot Admin**

![Spring Boot Admin 1]({{site.url}}/assets/img/High-Volume-Traffic-8/1-SPRING_BOOT_ADMIN_1.png)

![Spring Boot Admin 2]({{site.url}}/assets/img/High-Volume-Traffic-8/2-SPRING_BOOT_ADMIN_2.png)

Spring Boot Admin을 Spring Boot 기반 애플리케이션을 모니터링하고 관리하기 위한 웹 기반 UI 애플리케이션이다. Spring Boot 애플리케이션의 상태를 시각화하고, JVM Heap Dump, 메트릭, 로그, 환경 정보 등을 실시간으로 확인할 수 있다.

Spring Boot Admin은 Client와 Server로 구성된다.

- Client : 서버에 등록하고 actuator 엔드포인트에 액세스하는데 사용되는 클라이언트이다.
- Server : Spring Boot Actuator를 표시하고 상호 작용하기 위한 사용자 인터페이스를 제공하는 서버이다.

Spring Boot 애플리케이션에 적합한 모니터링 도구이지만, Server용 프로젝트를 생성해야한다는 점에서 컨테이너 혹은 서버가 필요하다.
추가적인 서버를 생성하면 비용이 발생하고, API 서버 내 Server용 컨테이너 운영은 리소스를 공유하게 되는 문제가 있다.

**Visual VM**

![Visual VM]({{site.url}}/assets/img/High-Volume-Traffic-8/3-VISUAL_VM.png)

Visual VM의 경우 CPU, Memory에 대한 사용률을 기본으로 내부 스레드의 리소스 사용률 또한 모니터링 가능하다.

무료로 사용 가능하지만 Spring Boot Admin에 비해 알림과 같은 자동화 기능이 없으며 수집한 메트릭을 장기간 저장하거나 시각화하는 기능이 제한적이다.

## **Prometheus & Grafana란**

### **Prometheus**

Prometheus는 시스템 및 서비스의 상태를 모니터링하는 오픈소스 모니터링 도구이다. 주요 특징은 다음과 같다.

- Prometheus는 key-value 쌍(label) 기반의 메트릭 데이터를 수집하고 저장합니다.
- Prometheus는 타겟에서 데이터를 풀링(Pull)한다. Prometheus Pushgateway를 이용하면 Push 방식도 구현할 수 있다.
- Kubernetes, Docker, Grafana와 같은 인기 있는 도구와 쉽게 통합된다.
- Exporter를 통해 다양한 시스템과 애플리케이션의 메트릭을 수집할 수 있습니다.

단점으로는 아래와 같다.

- 대규모 데이터 수집 시 스토리지 관리 부담이 있을 수 있다.
- 풀 방식이므로 네트워크 방화벽 설정 필요.
- 수집 메트릭과 룰이 많아질수록 화면 구성이 복잡해져 Grafana와 함께 사용해야 효과적이다.

#### **Prometheus UI**

![Prometheus UI 1]({{site.url}}/assets/img/High-Volume-Traffic-8/4-PROMETHEUS_UI_1.png)

![Prometheus UI 2]({{site.url}}/assets/img/High-Volume-Traffic-8/5-PROMETHEUS_UI_2.png)

Metrics Explorer를 통해서 원하는 리소스 정보를 확인할 수 있다. 다만, 패널 구성이 자유롭지 못해 다양한 데이터 정보를 보기위해서 패널을 추가하면 화면이 복잡해진다.

### **Grafana**

Grafana는 오픈 소스 데이터 시각화 및 모니터링 플랫폼으로 다양한 데이터 소스에서 데이터를 수집하여 대시보드 형태로 시각화하는 데 사용되며, 주요 특징으로 아래와 같다.

- Prometheus, Graphite, InfluxDB, Elasticsearch, MySQL, PostgreSQL, OpenTSDB 등 다양한 데이터 소스와 통합 가능하며, 대시보드에 조합하여 시각화할 수 있다.
- 사용자가 원하는 메트릭을 기반으로 대시보드를 쉽게 생성할 수 있으며, 다양한 시각화 옵션(라인 그래프, 바 차트, 히트맵, 게이지 등)을 제공한다.
- 사용자는 JSON 형식으로 대시보드를 정의할 수 있어 쉽게 백업 및 공유가 가능하다.
- 데이터를 실시간으로 스트리밍하여 빠르게 변하는 상황을 모니터링할 수 있다.
- [Grafana Labs](https://grafana.com/grafana/dashboards/){:target="\_blank"}에서 운영하는 서비스에 적합한 대시보드를 이용하여 모니터링할 수 있다.

## **Spring Boot Prometheus 설정 및 환경 구축**

### **의존성 추가**

```gradle
dependencies {
  ...

  implementation 'org.springframework.boot:spring-boot-starter-actuator'
  implementation 'io.micrometer:micrometer-registry-prometheus'


  ...
}
```

### **애플리케이션 속성 추가**

```yaml
...

management:
  endpoints:
    web:
      exposure:
        include: prometheus
  endpoint:
    prometheus:
      enabled: true

...
```

- **`management.endpoints.web.exposure.include`** : Spring Boot Actuator의 설정 중 하나로 웹 인터페이스에 노출할 엔드포인트를 정의한다.
  - **health** : 애플리케이션 상태 확인
  - **info** : 애플리케이션 정보 제공
  - **metrics** : 메트릭 데이터 확인
  - **prometheus** : Prometheus 형식으로 메트릭 데이터 제공

## **Docker Prometheus 환경 구성**

```yaml
# prometheus.yml

global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ["{API 서버 컨테이너명}:{PORT}"]

```

- **global 섹션**
  - **scrape_interval** : 데이터를 수집하는 기본 주기(스크랩 간격)를 설정
- **scrape_configs 섹션** : 여러 job을 추가하여 다양한 데이터 소스를 모니터링
  - **job_name** : 작업(Job)의 이름을 정의, Prometheus 메트릭 데이터에 job 라벨로 추가되어 나중에 데이터 분석 시 구분하는 데 사용
  - **metrics_path** : 메트릭 데이터를 수집하기 위해 접근할 경로(URL의 일부)를 지정(기본값은 `/metrics`이지만, Spring Boot Actuator에서는 `/actuator/prometheus`를 사용)
  - **static_configs** : 고정된(targets) 데이터 수집 대상을 정의
    - **targets** : Prometheus가 데이터를 스크랩할 서버의 호스트네임 또는 IP 주소와 포트를 지정

```yaml
# docker-compose.yml

services:
  API 서버 서비스명:
    container_name: {API 서버 컨테이너명}
    ...
  PROMETHEUS 서비스 명:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./{prometheus yml 파일 경로}/prometheus.yml:/etc/prometheus/prometheus.yml # prometheus.yml 마운트
    command:
      - '--config.file=/etc/prometheus/prometheus.yml' # prometheuss config file 지정
    ports:
      - "9090:9090"
    deploy:
      resources: # CPU & Memory 사용률 설정
        limits:
          cpus: 0.2
          memory: 96M
        reservations:
          cpus: 0.1
          memory: 16M

```

- **`docker-compose restart/down/start {서비스명}`** : 특정 서비스 재시작/중단/시작

http://{호스트 DNS or IP}:9090 접속 후 Prometheus Target을 확인하면 아래와 같이 API 서버 컨테이너의 **`/acurator/prometheus`{: .text-blue}**를 확인할 수 있다.

![Prometheus UI Target Result]({{site.url}}/assets/img/High-Volume-Traffic-8/6-PROMETHEUS_TARGETS.png)

## **Local에서 Docker Grafana 환경 구성**

Grafana의 경우 Prometheus와 같이 환경을 구성하여 환경울 구축할 수 있지만, API 서버와 리소스를 공유하게 되는 점에서 개별 서버로 운영해야 하는데 비용 발생으로 Local에 구축하였다.

### **docker-compose 구성**

```yaml
# docker-compose.yml

services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"

```

서비스를 실행하고 **`http://localhost:3000`{: .text-blue}**로 접속하면 로그인 화면이 뜨는데 초기 접속 정보는 **`admin/admin`**이다.

![Grafana UI]({{site.url}}/assets/img/High-Volume-Traffic-8/7-GRAFANA_UI.png)

### **DataSource 설정**

**`Home > Connection > Data sources`**에서 **Add new data source**{: .text-blue} 버튼을 누르고 **Prometheus**를 찾아 선택하면 아래와 같이 설정 화면을 확인할 수 있다.

![Grafana Prometheus Datasource]({{site.url}}/assets/img/High-Volume-Traffic-8/8-GRAFANA_PROMETHEUS_DATASOURCE.png)

필수 설정은 **`Prometheus server URL`**이다. 기존 API 서버의 호스트 DNS or IP를 통해 아래와 같이 입력 후 테스트를 통해 정상적으로 연결되었는지 확인하면된다.

![Grafana Prometheus Datasource Test]({{site.url}}/assets/img/High-Volume-Traffic-8/9-GRAFANA_PROMETHEUS_DATASOURCE_TEST.png)

### **Dashboard 구성**

**`Home > Dashboards`**에서 대시보드 생성으로 원하는 데이터 패널을 구성하여 Dashboard를 커스터마이징할 수 있다.

![Grafana Custom Dashboard]({{site.url}}/assets/img/High-Volume-Traffic-8/10-GRAFANA_CUSTOM_UI.png)

아래 링크를 통해 Grafana Cloud에 등록된 다양한 Dashboard를 이용할 수 있다.

> [https://grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards){:target="\_blank"}

사용 Dashboard → [https://grafana.com/grafana/dashboards/19004-spring-boot-statistics/](https://grafana.com/grafana/dashboards/19004-spring-boot-statistics/){:target="\_blank"}

Dashboard ID를 복사 후 **`Grafana Dashboard > Import Dashboard 화면`**에서 Dashboard를 로드 후 Import하면 아래와 같이 Prometheus에 수집되는 데이터들을 모니터링할 수 있다.

![Grafana Spring Boot Dashboard 1]({{site.url}}/assets/img/High-Volume-Traffic-8/11-GRAFANA_SPRING_BOOT_DASHBOARD_1.png)

![Grafana Spring Boot Dashboard 2]({{site.url}}/assets/img/High-Volume-Traffic-8/12-GRAFANA_SPRING_BOOT_DASHBOARD_2.png)

![Grafana Spring Boot Dashboard 3]({{site.url}}/assets/img/High-Volume-Traffic-8/13-GRAFANA_SPRING_BOOT_DASHBOARD_3.png)

![Grafana Spring Boot Dashboard 4]({{site.url}}/assets/img/High-Volume-Traffic-8/14-GRAFANA_SPRING_BOOT_DASHBOARD_4.png)

![Grafana Spring Boot Dashboard 5]({{site.url}}/assets/img/High-Volume-Traffic-8/15-GRAFANA_SPRING_BOOT_DASHBOARD_5.png)

![Grafana Spring Boot Dashboard 6]({{site.url}}/assets/img/High-Volume-Traffic-8/16-GRAFANA_SPRING_BOOT_DASHBOARD_6.png)

## **정리**

API 테스트를 진행하면서 리소스 사용에 대한 정보를 보기위해 **`Spring Boot Admin`**, **`Visual VM`**, **`Prometheus & Grafana`** 환경을 구성하여 사용해보았다.

Spring Boot Admin의 경우 Spring Boot 기반 애플리케이션 모니터링에 적합하며 애플리케이션 상태, 다양한 데이터의 메트릭화, Log, JVM Heap Dump 등을 쉽게 확인할 수 있다는 장점이 있다. 하지만, 주요 데이터 외에는 시각화가 부족하다는 점이 있었으며 JVM Heap Dump의 경우 Spring Boot Admin Server용 서버 스팩이 어느정도 갖춰지지 못하면 과도한 부하를 발생시키는 문제가 있다. 

Visual VM은 모니터링 하기에는 부족한 데이터와 애플리케이션에 성능 오버헤드가 발생하여 부하 테스트 과정에서 사용하기에 부적합했다.

Prometheus & Grafana는 다양한 데이터를 시각화가 잘되어있고 이슈가 발생한 시점을 쉽게 확인할 수 있으며 알림을 통해 실시간 대응을 할 수 있다는 이점이 있었다. JVM Heap Dump를 할 수 없어 아쉬운 점이 있지만, S3와 연동하여 OOM 발생 혹은 주기적인 Heap Dump를 통한 방법으로 개선할 수 있다.
