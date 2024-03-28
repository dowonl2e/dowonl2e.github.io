---
title: Redis Master/Replica(Slave), Sentinel, AOF With Docker
# author: dowonl2e
date: 2024-03-25 13:20:00 +0800
categories: [Redis]
tags: [Redis, Sentinel, AOF, Docker]
pin: true
img_path: "/assets"
image:
  path: /commons/Redis.png
  alt: Redis
---

작년에 재고 관리 시스템 버전을 업그레이드하면서 Redis를 사용했습니다. 당시에는 단순히 토큰관리 용도로만 사용했지만, 인메모리 DB Redis를 이용해 서비스를 운영할 때 어떠한 기술을 이용할 수 있는지 그리고 AOF 매커니즘으로 어떻게 백업을 해야하는지에 대한 공부를 하면서 내용을 정리해보았습니다.

## **Redis란?**

Redis란 오픈 소스 인메모리 데이터 구조 저장소입니다. 키-값(Key-Value) 구조의 저장소로서 strings, hashes, lists, sets, sorted sets 등 다양한 데이터 유형을 지원합니다.

Redis는 메모리 내에 데이터를 보관하므로 MySQL, Oracle 등 Database와 비교해 빠른 응답 시간을 제공합니다. 별도 SQL 작성이 필요없는 NoSQL 방식 입니다.

## **Redis 장단점**

### **장점**

1. **성능** : 메모리 내에서 데이터를 저장하고 액세스하기 때문에 매우 빠른 응답 속도를 제공합니다.
2. **다양한 데이터 구조** : 단순한 문자열부터 복잡한 해시맵, 리스트, 집합, 정렬된 집합 등 다양한 데이터 구조를 지원합니다. 이는 애플리케이션 요구사항에 적절한 데이터 타입을 활용할 수 있습니다.
3. **영속성** : 디스크에 데이터를 영구적으로 저장할 수 있는 옵션을 제공합니다. 따라서 시스템 장애 시에도 데이터 손실을 방지할 수 있습니다.
4. **높은 가용성** : Master/Slave와 같은 고가용성 구성을 지원하여 시스템 장애 시에도 중단되지 않고 계속해서 서비스를 제공할 수 있습니다.
5. **클라이언트 측 캐싱** : 클라이언트 측 캐싱을 지원하여 서버의 부하를 줄이고 응답 시간을 최적화할 수 있습니다.
6. **다양한 기능** : 트랜잭션, Pub/Sub(발행/구독), Lua 스크립팅, 비동기 처리 등 다양한 기능을 제공하여 개발자가 다양한 요구 사항을 처리할 수 있습니다.

### **단점**

1. **메모리 제약** : 모든 데이터를 메모리에 저장하기 때문에 메모리 제약이 있는 환경에서는 사용이 제한될 수 있습니다. 매우 큰 데이터셋을 다루는 경우 메모리 비용이 증가할 수 있습니다.
2. **단일 스레드 구조** : 기본적으로 단일 스레드로 동작합니다. 따라서 하나의 명령이 처리될 때 다른 명령은 대기해야 합니다. 이로 인해 매우 빠른 응답 시간을 제공하지만, 특정 작업이 블로킹될 수 있습니다.
3. **데이터 일관성 문제** : Redis의 Master-Slave는 비동기식이므로 Slave의 데이터가 항상 최신이 아닐 수 있어 데이터 일관성에 영향을 줄 수 있습니다.
4. **영구성 설정의 복잡성** : 영구적으로 데이터를 저장하려면 Redis를 디스크에 저장하도록 구성해야 합니다. 이 과정은 구성 및 관리의 복잡성을 증가시킬 수 있습니다.
5. **복잡한 캐싱 정책 구현** : 캐시 정책을 유지하고 구현하는 것은 Redis에서 어려울 수 있습니다. 캐시의 만료 및 무효화를 관리하기 위해 추가적인 로직을 구현해야 합니다.
6. **기능 부족** : 몇몇 기능들은 다른 데이터베이스 시스템에 비해 부족한 경우가 있습니다. 예를 들어, 복잡한 쿼리나 조인을 처리하기에는 적합하지 않을 수 있습니다.
7. **보안 취약점**: Redis는 기본적으로 인증을 사용하지 않기 때문에 적절한 보안 구성이 필요합니다. 또한 Redis 인스턴스가 외부에서 접근 가능하다면 보안 위험이 증가할 수 있습니다.

## **Redis Master / Replica(Slave) 환경 구축**

### **docker-compose.yml 설정**

```yaml
version: '3'
services:
  master:
    hostname: redis-master
    container_name: master
    image: 'bitnami/redis:latest'
    environment:
      - REDIS_REPLICATION_MODE=master
      - ALLOW_EMPTY_PASSWORD=yes
    networks:
      - redis-network
    ports:
      - 6379:6379

  replica-1:
    hostname: redis-replica-1
    container_name: replica-1
    image: 'bitnami/redis:latest'
    environment:
      - REDIS_REPLICATION_MODE=slave
      - REDIS_MASTER_HOST=redis-master
      - ALLOW_EMPTY_PASSWORD=yes
    ports:
      - 6380:6379
    networks:
      - redis-network
    depends_on:
      - master

  replica-2:
    hostname: redis-replica-2
    container_name: replica-2
    image: 'bitnami/redis:latest'
    environment:
      - REDIS_REPLICATION_MODE=slave
      - REDIS_MASTER_HOST=redis-master
      - ALLOW_EMPTY_PASSWORD=yes
    ports:
      - 6381:6379
    networks:
      - redis-network
    depends_on:
      - master

  redis-commander:
    container_name: redis-commander
    hostname: redis-commander
    image: rediscommander/redis-commander:latest
    restart: always
    environment:
      - REDIS_HOSTS=master:master,replica-1:replica-1,replica-2:replica-2
    ports:
      - "8081:8081"
    networks:
      - redis-network
    depends_on:
      - master

networks:
  redis-network:
    driver: bridge
```

- Master / Replica 이미지는 redis를 이용했으며, redis-commander는 rediscommander/redis-commander를 이용했습니다.
- redis-commander : redis 관리 툴을 위해서 추가했습니다.

**환경변수(environment variables)**의 경우 아래 링크를 통해 확인할 수 있습니다.

> [https://hub.docker.com/r/bitnami/redis](https://hub.docker.com/r/bitnami/redis){:target="\_blank"} 

### **Master / Replica(Slave) 정보 확인**

**Master / Replica redis-cli 접속**

```bash
$ redis-cli -p 6379 => master 접속
$ redis-cli -p 6380 => replica-1 접속
$ redis-cli -p 6381 => replica-2 접속
```

**Master / Replica redis-cli 정보 확인**

```bash
> info Replication

# Replication
role:master
connected_slaves:2
slave0:ip=192.168.48.5,port=6379,state=online,offset=98,lag=1
slave1:ip=192.168.48.4,port=6379,state=online,offset=98,lag=1
...
```

redis-cli로 연결 후 위 명령어 실행시 각 노드에 대한 정보를 확인하실 수 있습니다. 아래의 경우 Master에 접속했으며, **`connected_slaves:2`{: .text-blue }** 를 통해 설정한 2개의 Replica가 연결된 것을 볼 수 있습니다.

**Master 로그 확인**

![Redis Master Node Log]({{site.url}}/assets/img/RedisWithDocker/1_REDIS_MASTER_LOG.png)

- 백업을 위한 AOF 설정을 확인할 수 있습니다.
- 2개의 Replica가 Master와 동기화된 것을 볼 수 있습니다.

**Redis-Commander 확인**

http://<hostname>:8081로 redis-commander에 접속하면 아래와 같이 master, replica-1, replica-2 노드가 정상적으로 확인되는 것을 볼 수 있습니다.

![Redis Commander Page]({{site.url}}/assets/img/RedisWithDocker/2_REDIS_COMMANDER.png)

## **Redis Sentinel 환경 구축**

Redis Sentinel이란 Redis에 문제가 발생했을 때 **모니터링**, **알람**, **자동 페일 오버**를 제공하는 고가용성 솔루션입니다.

데이터가 유실되더라도 서비스가 정상적으로 이용되는 경우 (캐시 용도로 사용) 큰 문제가 되지 않을 수 있지만, 유실되지 않아야 하는 데이터의 경우 큰 문제가 발생합니다. 만약 복구하더도 정상 운영까지의 **`다운타임`{: .text-blue }**이 발생합니다.

### **docker-compose.yml 수정**

```yaml
version: '3'
services:
  master:
    hostname: redis-master
    container_name: master
    image: 'bitnami/redis:latest'
    environment:
      - REDIS_REPLICATION_MODE=master
      - ALLOW_EMPTY_PASSWORD=yes
    
    networks:
      - redis-network
    ports:
      - 6379:6379

  replica-1:
    hostname: redis-replica-1
    container_name: replica-1
    image: 'bitnami/redis:latest'
    environment:
      - REDIS_REPLICATION_MODE=slave
      - REDIS_MASTER_HOST=redis-master
      - ALLOW_EMPTY_PASSWORD=yes
    ports:
      - 6380:6379
    networks:
      - redis-network
    depends_on:
      - master

  replica-2:
    hostname: redis-replica-2
    container_name: replica-2
    image: 'bitnami/redis:latest'
    environment:
      - REDIS_REPLICATION_MODE=slave
      - REDIS_MASTER_HOST=redis-master
      - ALLOW_EMPTY_PASSWORD=yes
    ports:
      - 6381:6379
    networks:
      - redis-network
    depends_on:
      - master

  redis-commander:
    container_name: redis-commander
    hostname: redis-commander
    image: rediscommander/redis-commander:latest
    restart: always
    environment:
      - REDIS_HOSTS=master:master,replica-1:replica-1,replica-2:replica-2
    ports:
      - "8081:8081"
    networks:
      - redis-network
    depends_on:
      - master

  sentinel-1:
    container_name: sentinel-1
    image: 'bitnami/redis-sentinel:latest'
    environment:
      - REDIS_SENTINEL_DOWN_AFTER_MILLISECONDS=3000
      - REDIS_MASTER_HOST=redis-master
      - REDIS_MASTER_PORT_NUMBER=6379
      - REDIS_MASTER_SET=master
      - REDIS_SENTINEL_QUORUM=2
    networks:
      - redis-network
    ports:
      - 26379:26379
    depends_on:
      - master
      - replica-1
      - replica-2

  sentinel-2:
    container_name: sentinel-2
    image: 'bitnami/redis-sentinel:latest'
    environment:
      - REDIS_SENTINEL_DOWN_AFTER_MILLISECONDS=3000
      - REDIS_MASTER_HOST=redis-master
      - REDIS_MASTER_PORT_NUMBER=6379
      - REDIS_MASTER_SET=master
      - REDIS_SENTINEL_QUORUM=2
    networks:
      - redis-network
    ports:
      - 26380:26379
    depends_on:
      - master
      - replica-1
      - replica-2

  sentinel-3:
    container_name: sentinel-3
    image: 'bitnami/redis-sentinel:latest'
    environment:
      - REDIS_SENTINEL_DOWN_AFTER_MILLISECONDS=3000
      - REDIS_MASTER_HOST=redis-master
      - REDIS_MASTER_PORT_NUMBER=6379
      - REDIS_MASTER_SET=master
      - REDIS_SENTINEL_QUORUM=2
    networks:
      - redis-network
    ports:
      - 26381:26379
    depends_on:
      - master
      - replica-1
      - replica-2

networks:
  redis-network:
    driver: bridge
```

- `sentinel-1`, `sentinel-2`, `sentinel-3` 서비스를 추가해줍니다. (bitnami/redis-sentinel 이미지 사용)

Failover를 판단하기 위한 기준으로 Sentinel의 동의가 과반수 이상이 되어야한다. 그래서 Sentinal을 구성할 때 최소 3개 이상을 권장하며, 과반수 결정을 위해 홀수로 생성해야 할 필요가 있다.

### **Sentinal Message 확인하기**

```bash
$ redis-cli -p 26379 # sentinel-1
$ redis-cli -p 26380 # sentinel-2
$ redis-cli -p 26381 # sentinel-3
```

```bash
> psubscribe *
```

위 명령어를 통해 Sentinal Message를 확인할 수 있습니다.

```bash
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "*"
3) (integer) 1
```

새로운 터미널을 열어 마스터를 중지해보면 아래와 같이 Sentinel에서 메세지를 확인할 수 있습니다.

```bash
3) "+sdown"
4) "master master 192.168.128.2 6379"
1) "pmessage"
2) "*"
3) "+odown"
4) "master master 192.168.128.2 6379 #quorum 2/2"
1) "pmessage"
2) "*"
3) "+new-epoch"
4) "1"
1) "pmessage"
2) "*"
3) "+try-failover"
4) "master master 192.168.128.2 6379"
1) "pmessage"
2) "*"
3) "+vote-for-leader"
4) "1cfc2ca5e547c63253cb7a362f2c5ece6743f162 1"

...

3) "+switch-master"
4) "master 192.168.128.2 6379 192.168.128.4 6379"
1) "pmessage"

...

3) "-role-change"
4) "slave :6379 192.168.128.2 6379 @ master 192.168.128.4 6379 new reported role is master"
1) "pmessage"
```

- 마스터 장애여부 판단시간이 되면 Sentinel에서 장애여부를 인지하고 failover를 시도합니다. 이 과정에서 3개의 Sentinel에서 새로운 마스터를 투표한 후에 새로운 마스터 노드를 판단 후 지정해줍니다.
- Replica의 정보를 확인하면 master가 변경된 것을 확인할 수 있습니다.

### **기존 Master를 중단 후 실행하면?**

장애가 발생했다는 가정하에 기존 Master를 재시작 후 **`redis-cli -p 6379`{: .text-blue }** 연결 후 정보를 확인하면 role이 slave가 되며, 변경된 master로 연결됩니다.

![Redis Replica Information]({{site.url}}/assets/img/RedisWithDocker/3_REPLICA_INFO.png)

### **redis-commander에서 확인**

**1) Master 컨테이너 중단 후 replica-2 노드의 role이 master로 변경**
![Redis Master Stop]({{site.url}}/assets/img/RedisWithDocker/4_INIT_MASTER_STOP.png)

**2) 기존 Master 노드 실행 후 master 노드의 role이 slave로 변경**
![Redis Master Start]({{site.url}}/assets/img/RedisWithDocker/5_INIT_MASTER_START.png)

기존 master 컨테이너를 다시 실행하면 위 이미지에서 보듯이 데이터는 현재 master의 데이터와 동일하게 유지됩니다.

## **Redis 데이터 백업 매커니즘**

Redis 데이터를 백업하기 위한 매커니즘으로 **`RDB(Redis Database)`{: .text-blue }**, **`AOF(Append Only File)`{: .text-blue }**가 있습니다.

### **RDB(Redis Database)**

특정 시점(snapshot)의 메모리에 있는 데이터 전체를 바이너리 파일로 저장합니다. AOF와 비교했을 때 보다 작은 사이즈로 로딩 속도가 빠릅니다.

하지만, 특정 조건에서 백업하므로 스냅샷이 찍히기 전에 Redis가 종료되면 그 사이의 데이터는 복원할 수 없습니다.

Redis 설정 파일(redis.conf)에서 SNAPSHOOTING 부분을 보면 아래와 같은 설정으로 SNAPSHOT을 지정할 수 있으며, 스냅샷은 디폴트로 **dump.rdb** 파일에 저장됩니다.

- **save 3600 1** : 3600초 안에 1개 이상의 데이터가 변경되면 저장
- **save 300 100** : 300초 안에 100개 이상의 데이터가 변경되면 저장
- **save 60 10000** : 60초 안에 10000개 이상의 데이터가 변경되면 저장

### **AOF(Append Only File)**

데이터 변경작업을 기록하는 매커니즘으로 수행된 모든 작업이 파일에 연속적으로 기록됩니다.

계속해서 기록하는 방식이다보니 파일 사이즈가 커지며, 로딩 속도가 느려집니다.

그렇지만, rewrite를 통해 용량이 지정된 사이즈를 넘어갈 경우 **base.rdb**에 저장되며, **incr.aof(데이터 변경이 기록되는 파일)** 파일을 초기화하게 됩니다.

Redis 설정에서 AOF에 데이터를 기록하는 시점을 아래 3개 중 하나로 지정할 수 있습니다.

- **appendfsync always** : 명령어를 실행할 때 마다 기록
  - 안전하나 성능이 떨어질 수 있습니다.
- **appendfsync everysec** : 매 초마다 기록
  - 1초 사이에 데이터 유실 있을 수 있으나, 성능에 거의 영향을 미치지 않고 데이터를 보존할 수 있기에 권장합니다. (default)
  - 2.4 버전 이후 부터 Snapshot 만큼 충분히 빠르다.
- **appendfsync no** : 기록 시점을 OS가 정함
  - 기본적으로 Linux의 경우 매 30초마다 flush하지만, 커널의 설정에 따라 다릅니다.

rewrite 기준은 **`auto-aof-rewrite-percentage`**, **`auto-aof-rewrite-min-size`**이며, redis-cli 접속 후 아래 명령어를 통해 확인할 수 있으며, Redis 설정파일에서 변경 가능합니다.

```bash
> config get auto-aof-rewrite-percentage # DEFAULT = 100
> config get auto-aof-rewrite-min-size # DEFAULT = 67108864 (= 64MB)
```

## **AOF(Append Of File)을 이용한 데이터 백업**

Docker 환경에서 **`bitnami/redis`{: .text-blue }** 이미지를 사용하는 경우 AOF는 환경변수를 통해 설정 가능합니다.

### **docker-compose.yml 수정**

`master`, `replica-1`, `replica-2` 서비스에 environment 변수 값과 volumes를 추가 후 컨테이너를 재시작합니다.

```yaml
environment:
  ...
  - REDIS_AOF_ENABLED=yes
volumes:
  - {로컬 스토리지 경로}:/bitnami/redis/data
```

- **REDIS_AOF_ENABLED** : AOF 사용 여부를 설정합니다. (default : no)
- **volumes** : AOF 파일을 확인하기 위해 bitnami에서의 컨테이너 data를 로컬 스토리지에 마운트 해줍니다.

5개의 데이터가 있으며 데이터를 모두 삭제하고 복구해보겠습니다.

![Redis Data Delete All]({{site.url}}/assets/img/RedisWithDocker/6_DATA_DELETE_ALL.png)

docker-compose에 설정한 **volume**의 **로컬 스토리지 경로**를 보면 **incr.aof** 로 끝나는 파일을 볼 수 있습니다. 에디터로 확인하면 아래와 같이 실행된 명령어가 기록되는 것을 확인할 수 있습니다.

![Redis Data Flush All Commandar Remove]({{site.url}}/assets/img/RedisWithDocker/7_FLUSHALL_REMOVE.png)

복구 과정에서는 aof 파일을 이용해 처리되므로, 로컬 스토리지 경로로 이동 후 위 파일에서 마지막에 실행된 **`flushall`** 을 삭제합니다.

컨테이너를 재시작하면 아래와 같이 로그를 확인 할 수 있으며, **incr.aof** 파일을 읽는 것을 볼 수 있습니다.
![Redis Data Back Up]({{site.url}}/assets/img/RedisWithDocker/8_DATA_BACKUP.png)