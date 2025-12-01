---
title: AWS Lambda Cold Start 최적화 
# author: dowonl2e
date: 2025-12-01 05:00:00 +0900
categories: [AWS Lambda]
tags: [AWS, Lambda, Cold Start]
pin: true
---

새로 입사한 회사에서의 이슈 중 서비스 이용에 있어 가장 많이 언급된 문제는 **`AWS Lambda의 ColdStart`{: .text-blue}**였다. 우선, AWS Lambda를 처음 사용해본 입장에서 AWS Lambda가 무엇이고 왜 Cold Start가 발생하는지 그리고 Cold Start로 인한 이슈 해소에 대한 내용을 정리해보았다.

## **AWS Lambda**

### **AWS Lambda란?**

AWS Lambda는 서버리스(Serverless) 컴퓨팅 서비스라고 한다. 좀 더 풀어서 말하자면, 서버를 프로비저닝 또는 관리하지 않고도 실제로 모든 유형의 애플리케이션과 백엔드 서비스에 대한 코드를 실행할 수 있는 이벤트 중심의 서버리스 컴퓨팅 서비스이다.

### **주요 특징**

- **서버리스:** 서버의 프로비저닝, 유지 보수, 확장 및 축소 등 인프라 관리에 대한 부담이 없다.
- **이벤트 기반:** 특정 이벤트(예: API 호출, S3에 파일 업로드, 데이터베이스 변경 등)가 발생할 때 함수가 실행된다.
- **자동 확장:** 이벤트 발생량에 따라 자동으로 확장 및 축소된다.
- **사용량 기준 과금:** 코드가 실행된 시간과 요청 수에 따라 비용이 부과되며, 실행되지 않을 때는 요금이 발생하지 않는다.
- **다양한 언어 지원:** Node.js, Python, Java 등 다양한 프로그래밍 언어를 지원하여 개발자는 익숙한 언어로 코드를 작성할 수 있습니다.

### **제약 사항**

- **리소스**: 람다 함수는 사용할 수 있는 리소스에 제한이 있다. 이러한 제약에는 **512MB의 디스크 공간(일시적)과 최대 10240MB의 메모리 허용량**이 포함됩니다. 이는, 기본적으로 크기가 큰 Spring Boot, Django, Next.js 등과 같은 프레임워크를 이용함에 있어 적합하지 않다.
- **실행 시간 초과:** 람다 함수의 **처리 시간은 최대 15분**으로 제한된다. 이러한 점은 확장된 처리가 필요한 프로세스나 작업을 처리할 때 영향이 있을 수 있다. 따라서, 15분 이상의 긴 작업의 경우 제한되어 장기 실행 프로세스에는 적합하지 않다.
- **Cold Start**: 서버리스 서비스로 효율적인 리소스 사용을 위해 오랫동안 사용하지 않는 경우 컴퓨팅 파워를 꺼두게 됩니다. 함수가 처음 호출될 때 람다함수를 실행시키기 위해 부수적인 설정이 필요한데, **`이 설정 과정에서 발생하는 Delay를 Cold Start`**라고 합니다. 콜드 스타트 현상은 사용하는 언어, 설정한 메모리에 따라 다르게 나타나게 됩니다.
- **동시성 문제**: 기본적으로 람다는 동시에 실행할 수 있는 람다 함수의 개수를 각 리전별 최대 1,000개로 제한하고 있습니다. 따라서 Request의 수가 이를 넘어가게 되면 람다가 수행되지 않는 문제가 발생할 수도 있습니다.
이러한 단점이 있음에도 현재 AWS Lambda를 이용하는 이유로는 적은 트래픽에서 EC2 서버와 비교해 비용 그리고 러닝커브에 대한 부분으로 인지하고 있다.

## **AWS Lambda Cold Start**

AWS Lambda의 Cold Start는 함수가 실행되기 전에 발생하는 지연 현상으로, Lambda의 **서버리스(Serverless)** 및 **비용 효율적인** 아키텍처 설계 방식 때문에 필연적으로 발생한다.

Cold Start가 발생하는 주된 이유는 Lambda가 함수 실행을 위해 **새로운 실행 환경(Execution Environment), 즉 컨테이너를 처음으로 준비하는 과정**을 거쳐야 하기 때문이다.

### **Lambda Handler 실행 순서**

람다 함수의 동작 과정을 설명하자면 아래와 같다.

![Lambda Handler Execution Process]({{site.url}}/assets/img/AWS-Lambda-ColdStart/1-Lambda_Handler_Process.png)

> 참고: [https://inpa.tistory.com/entry/AWS-📚-람다-성능-개선-Cold-Start-해결](https://inpa.tistory.com/entry/AWS-📚-람다-성능-개선-Cold-Start-해결){:target="\_blank"}

- **Download your code**: Lambda 핸들러가 실행 될 AWS 내부 인스턴스에서 개발자가 작성하고 업로드한 코드를 다운로드 하는 단계
- **Start new execution environment**: 새로운 실행 환경(execution environment)을 생성하는 단계
  - execution environment란 Lambda 핸들러가 실행될 환경으로써 메모리, 런타임 등을 구성
- **Execute initialization code**: Lambda 핸들러 바깥의전역 코드를 실행하는 단계
- **Execute handler code**: Lambda 핸들러의내부 함수 코드를 실행하는 단계

위 그림에서 Cold Start가 발생하는 구간은 **`Initialization`{: .text-blue}**이고 Warm Start가 발생하는 구간은 **`Invocation duration`{: .text-orange}**이다.

이렇게 AWS Lambda에서는 Cold Start는 필연적으로 발생하게되고, 여기서 발생하는 Delay를 최소한으로 줄일 필요가 있다. 이에 아래와 같이 팀원과 함께 아래 2가지 개선 작업을 진행하였다.

1. 불필요한 외부 의존성 제거
2. 지연 시간 최소화를 위한 초기화/캐싱

## **불필요한 외부 의존성 제거**

### **외부 의존성 제거로 인한 효과**

AWS Lambda Handler 실행 순서에서 가장 먼저 확인할 수 있는 것은 코드를 다운로드하는 단계이고, 이 점에서 먼저 생각해본 내용이 불필요한 외부 의존성 제거이며, 이를 통해서 얻을 수 있는 개선 사항은 아래와 같다.

- **다운로드 시간 감소:** Lambda 실행 환경(컨테이너)이 할당될 때, AWS 내부 네트워크를 통해 코드 패키지를 다운로드해야 합니다. 패키지 크기가 작으면 이 다운로드 시간이 단축된다.
- **압축 해제 시간 감소:** 다운로드 후 패키지를 메모리에 로드하기 위해 압축을 해제(Unzip)하는 과정이 필요합니다. 파일 수가 줄고 전체 크기가 작아지면 이 I/O 집약적인 작업 시간이 줄어든다.
- **메모리 절약:** 불필요한 라이브러리가 메모리에 로드되지 않기 때문에, 함수에 할당된 메모리를 실제 비즈니스 로직 실행에 더 효율적으로 사용할 수 있습니다.

### **외부 의존성 체크**

Cold Start로 인한 이슈가 생긴 프로젝트를 확인해볼 때 Lambda에 불필요한 외부 의존성이 너무 많이 포함되어있는 것을 확인했다.

**첫 번째**, 작업의 편리성으로 빌드 과정에서 개발에만 필요한 의존성을 모두 포함시키다보니 필요없는 의존성이 추가되는 현상이 있었다.

**두 번째**, AWS SDK(Software Development Kit) 제거이다. AWS Lambda는 실행 환경에 이미 SDK가 기본적으로 내장되어 있기 때문에 의존성을 추가할 필요가 없다. 게다가, AWS SDK(`@aws-sdk/*`)의 경우 약 5MB(다른 외부 의존성 대비 약 50배 차이) 정도의 크기를 차지하고 있다.

### 의존성 제거에 따른 Cold Start 시간 테스트

| 제거 의존성 | 제거 전/후 차이(ms) |
| --- | --- |
| 약 10개 Dependencies | 약 50ms |
| AWS SDK Dependency | 약 400ms (약 2.1초 → 약2.5초)  |

위 결과를 보면 약 10개 의존성을 제거했을 때에 대한 효과는 크게 차이가 없지만, 크기가 큰 의존성(AWS SDK)에 대한 의존성 제거의 경우 눈에 띄는 효과를 보이고 있으며, 의존성 제거를 통한 Cold Start 

> **두 케이스를 체크했을 때 100Kb의 의존성은 약 5ms 정도의 지연을 발생 시킴** <br/>**(의존성 추가시 해당 라이브러리 크기 체크 필요!)**{: .text-red}

### **불필요 외부 의존성 처리 결과**

| 구분 | 제거 전 | 제거 후 |
| --- | --- | --- |
| API 요청/응답 처리시간 | 약 2,500ms | 약 2,000ms |

> **약 20%의 속도 개선**{: .text-blue}

## **지연 시간 최소화를 위한 초기화/캐싱**

지연시간 최소화를 위한 초기화 및 캐싱 방법은 초기화 비용이 높은 작업(코드)를 애플리케이션 수명 주기 초기에 한 번만 수행하기 위해 전역(람다 핸들러 외부)으로 설정한 후 이후 요청 처리 시 발생하는 **반복적인 지연을 최소화**하는 방법이다.

### 초기화/캐싱 처리 확인

초기 비용이 높은 작업은 DB 설정 밖에 없었고, 이미 설정되어 있는 것으로 확인하였음에도, Cold Start에서의 지연시간은 변함이 없었다. 이 점에서 또 다른 방법을 생각하고 찾아볼 필요가 있어, 여러가지 코드를 보면서 아래의 문제를 발견하게 되었다.

### **DB Connection Pool**

그 동안 DB Connection Pool의 경우 최소/최대 커넥션이 기본적으로 설정되어 있다고 생각하고 있었고 자동 생성이 되었을 것이라 생각했다. 하지만, PostgreSQL에서 pg-promise 커넥션 풀을 찾아보았을 때 **`지연 연결(Lazy Connection)`{: .text-blue}** 방식을 사용하는 것을 확인했다. 

정리하면, 기존에 지연 시간 최소화를 위한 설정은 DBClient 객체 생성까지만 되어있고 초기 커넥션은 생성하지 않고 있다.

#### **초기 커넥션 생성 코드 추가**

초기 커넥션 생성의 경우 핸들러 외부에 전역으로 실행되도록 설정한다.

```tsx
const DB_CONFIG: IConnectionParameters<IClient> = {
  host: DB_HOST,
  port: Number(DB_PORT),
  database: DB_NAME,
  user: DB_USER,
  password: DB_PASSWORD,

  // PG Pool 옵션
  max: DB_MAX_CONNECTION, // 최대 커넥션 수
  idleTimeoutMillis: DB_IDLE_TIMEOUT_MILLIS, // 유휴 커넥션 유지 시간
  connectionTimeoutMillis: DB_CONNECTION_TIMEOUT_MILLIS, // 커넥션 연결 최대 시간
  statement_timeout: STATEMENT_TIMEOUT, // SQL 처리 시간
};

let pgp: pgPromise.IMain = pgPromise();
let dbClient: pgPromise.IDatabase<{}, IClient> | null = null;

export function getDbClient() {
  if (!dbClient) {
    dbClient = pgp(DB_CONFIG);
  }
  return dbClient;
}

async function createInitConnections(initCount: number) {
  getDbClient();
  const dummyQueries = Array.from({ length: initCount }, async () => {
    if (dbClient) {
      try {
        await dbClient.any("SELECT 1");
        return true;
      }
      catch(error){
        console.error(`Create initial connections error: ${(error as Error)error.message}`);
        return false;
      }
    }
  });

  const results = await Promise.all(dummyQueries);
  const connectionCount = results.filter(result => result === true).length;
  console.info(`createInitConnections - 초기 ${connectionCount}개 DB 커넥션 확보 완료`);
  if(dbClient){
    const pool = dbClient.$pool;
    console.info(`createInitConnections - 총 커넥션 수: ${pool.totalCount}`);
    console.info(`createInitConnections - 유휴 커넥션 수: ${pool.idleCount}`);
    console.info(`createInitConnections - 커넥션 대기 중인 요청 수: ${pool.waitingCount}`);
  }
}
await createInitConnections(DB_MIN_CONNECTION);
```

### 초기화/캐싱 처리 결과

| 구분 | 차이 |
| --- | --- |
| API 요청/응답 처리속도 | 약 400ms |

#### **CloudWatch에서의 Cold Start 시간 비교**

##### **초기 커넥션 미적용**

![Initial Connection Does Not Applied Result]({{site.url}}/assets/img/AWS-Lambda-ColdStart/2-No_Apply_Connection.png)

##### **초기 커넥션 적용**

![Initial Connection Applied Result]({{site.url}}/assets/img/AWS-Lambda-ColdStart/3-Apply_Connection.png)

위 내용에서 초기 커넥션 적용시 Cold Start 시간은 약 150ms의 차이로 초기 커넥션 적용했을 때 조금 더 지연되는 것을 볼 수 있다. 하지만, 핸들러 내부에서 DB 처리시에는 **`초기 커넥션 미적용시 700ms, 초기 커넥션 적용시 100ms 약 7배 차이`{: .text-blue}**가 난다.

| 구분 | 초기 커넥션 생성 미적용(ms) | 초기 커넥션 생성 적용(ms) |
| --- | --- | --- |
| Cold Start Time | 760 | 900 |
| DB Process Time | 700 | 100 |
| 비교 | 1460 | 1000 |

위 결과는 Cold Start의 지연시간을 단축시킨 결과는 아니지만, API 응답시간 성능 개선으로 볼 수 있다. 결과적으로, 성능 개선의 관점에서 초기 커넥션 적용시 **`약 450ms 정도 API 응답시간이 감소`{: .text-blue}**되는 것을 확인할 수 있었고 **`20% 이상 성능 개선의 효과`{: .text-blue}**를 보았다.

### **DB Connection Pool - Lazy Connection에 대한 의문점**

AWS Lambda에서 DB Connection 초기화 및 캐싱 효과를 통해 효과를 보았지만, 한가지 의문점이 생겼다.

요청이 1번 발생하는 상황에서 아래 2가지 경우에 대해서는 시간 차이가 생기지 않아야한다고 생각이 들었다.

1. **람다 핸들러 실행 → 핸들러 내부에서 커넥션 생성 → DB 데이터 조회 → API 응답 = 1,460ms**
2. **핸들어 외부에서 커넥션 생성 → 람다 핸들러 실행 → DB 데이터 조회 → API 응답 = 1,000ms**

여러 요청에 대해서는 당연히 외부 커넥션 생성 후 커넥션을 재사용함에 있어 좋다고 생각하지만, 1번의 요청에 대해서는 속도 차이가 없어야하지 않나?!

이에 대해 찾아 보았을 때, 아래 내용을 확인 할 수 있었다.

```text
위의 시간 차이는 DB 연결 자체의 시간이 아니라, 
DB 연결을 Lambda 컨테이너 초기화 시간과 겹치게 만드는 최적화 때문이다. 
즉, 두 작업이 병렬 처리로 인해 I/O 대기 시간 중 일부가 겹쳐 숨겨지게 보이는 효과가 발생한다.
```

핸들러 외부 초기화는 고비용의 네트워크 작업을 콜드 스타트의 필수 초기화 과정 속에 포함시켜 사용자에게 체감되는 총 지연 시간을 줄이는 것이다.

## **Cold Start 지연시간 최소화 및 성능 개선 결과**

- 최초 API 응답시간 : 약 2,500ms
- 불필요 의존성 제거 후 API 응답시간 : 약 2,050ms (2,500ms - 450ms)
- DB Connection 초기화/캐싱 : 약 1,590ms (2,050ms - 460ms)

> **API 응답시간 약 36% 성능 개선**{: .text-blue}