---
title: Terraform을 이용한 Serverless CI/CD 구축
# author: dowonl2e
date: 2026-10-10 15:00:00 +0900
categories: [Terraform]
tags: [Terraform, CDKTF, GitLab CI, TypeScript, AWS, API Gateway, Lambda]
pin: true
---

## **개요**

회사에서 진행하는 프로젝트의 인프라는 **`서버리스(Serverless)`{: .text-blue}**를 지향한다. 그래서 **`AWS API Gateway`{: .text-blue}**와 **`AWS Lambda`{: .text-blue}**를 중심으로 서비스를 구성하고, 이 리소스들은 **`CDKTF(CDK for Terraform)`{: .text-blue}** 기반의 IaC 코드로 관리하고 있었다.

### **로컬에서 이루어지던 배포**

문제는 배포 방식이었다. 백엔드 담당자가 **`로컬에서 직접 배포하고 그 다음에 커밋`{: .text-red}**하는 형식으로 작업이 이루어지고 있었다.

이 방식에서는 다음과 같은 문제가 생긴다.

- **배포된 코드와 레포지토리의 코드가 일치한다는 보장이 없다.** 로컬에서 수정한 내용을 먼저 배포하고 커밋을 누락하면, 현재 운영 중인 코드가 어느 시점의 코드인지 추적할 수 없다.
- **누가 언제 무엇을 배포했는지 남지 않는다.** 배포 이력이 개인 로컬에만 존재하기 때문에, 문제가 생겼을 때 되돌릴 기준점이 없다.
- **배포 환경이 담당자마다 다르다.** Node, Terraform, CDKTF 버전이 로컬마다 다르면 같은 코드를 배포해도 결과가 달라질 수 있다.

결국 **`프로젝트 버전 관리가 제대로 이루어질 수 없는 구조`{: .text-red}**였다.

### **작업의 목적**

> **배포의 기준을 개인 로컬이 아니라 레포지토리로 옮겨 프로젝트 버전 관리가 가능하도록 하는 것**{: .text-blue}
{: .prompt-info }

이 목적을 기준으로 레포지토리 구조와 배포 과정을 다시 정리했고, 그 과정에서 겪은 문제와 개선 내용을 정리해보았다.

이 글은 아래 순서로 이어진다.

1. **레포지토리 분리** - IaC와 서비스 코드를 분리하고 CI/CD를 새로 구성
2. **배포 시간 개선** - 첫 배포에서 길어진 파이프라인 시간을 단축
3. **프로젝트 구조 개선** - 분리 이후 드러난 IaC 프로젝트의 배포 스크립트·설정 구조 정리
4. **Terraform 전환** - CDKTF의 업데이트 중단에 따른 기반 변경

## **기존 구조의 문제점**

### **하나의 레포지토리에 공존하는 IaC와 서비스 코드**

기존 레포지토리는 아래와 같이 인프라 정의와 서비스 로직이 같은 공간에 있었다.

```text
aws-cdktf-iac
├── infra          # CDKTF 기반 IaC 코드 (API Gateway, Lambda, Layer 정의)
│   └── config     # Lambda 설정
├── app            # 서비스 구현 코드
└── package.json
```

이 구조에서는 서비스 로직 하나를 수정하더라도 IaC 코드와 같은 레포지토리에서 작업해야 했다.

### **레포지토리를 분리한 이유**

첫 번째는 **`관심사의 분리`{: .text-blue}**이다. 서비스 로직을 개발하는 데 굳이 IaC 설정과 구현부까지 알아야 할 필요가 있을까 하는 의문이 있었다. 인프라를 담당하지 않는 팀원 입장에서 IaC 코드는 작업 범위를 넓히기만 하고 실제로는 건드릴 일이 없는 영역이었다.

두 번째는 **`형상관리 차원의 버전관리 이슈`{: .text-blue}**이다. 인프라 변경과 서비스 변경이 같은 커밋 히스토리에 섞이면서, 특정 시점의 서비스 버전을 추적하는 것이 어려워졌다.

AWS는 API Gateway의 스테이지, Lambda와 Layer의 버전 관리 기능을 제공한다. 하지만 이 리소스 단위의 버전을 관리하는 방안보다, 팀 차원에서는 **`서비스 자체의 버전 관리가 우선순위가 높다`{: .text-blue}**고 판단했다.

> **IaC와 서비스 레포지토리를 분리하고, 서비스 레포지토리의 CI/CD가 IaC 레포지토리를 Clone해와서 배포하는 구조를 선택했다.**{: .text-blue}
{: .prompt-info }

## **첫 번째 파이프라인 구성**

분리 직후 구성한 GitLab CI 파이프라인은 작업 단위별로 스테이지를 모두 쪼갠 형태였다.

```yaml
stages:
  - clone-cdktf          # IaC 레포지토리 Clone
  - download-config-file # Secure Files 다운로드
  - prepare-source-file  # 서비스 코드를 IaC 레포지토리로 복사
  - install-dependencies # npm install
  - test                 # 테스트
  - build                # Layer, Lambda 빌드
  - deploy-dev           # 개발 환경 배포
```

![First CI/CD Result]({{site.url}}/assets/img/Serverless-Terraform/1-Slow_CICD_Result.png)

배포는 성공했지만 전체 파이프라인에 **`약 8분 49초`{: .text-red}**가 소요되었다.

| **스테이지** | **소요 시간** |
| --- | --- |
| clone-cdktf | 00:00:10 |
| download-config-file | 00:00:09 |
| prepare-source-file | 00:00:08 |
| install-dependencies | 00:00:44 |
| test | 00:00:44 |
| build | 00:01:01 |
| **deploy-{env}** | **00:05:53** |
| **합계** | **00:08:49** |

전체의 **`약 67%를 deploy-{env} 스테이지가 차지`{: .text-red}**하고 있었고, 나머지 스테이지들도 실제 작업량에 비해 시간이 길었다.

## **CI/CD 파이프라인 개선**

### **1. 스테이지 통합 (7개 → 4개)**

GitLab CI는 스테이지가 바뀔 때마다 **`새로운 러너 컨테이너를 띄우고, 이전 스테이지의 artifact를 업로드했다가 다시 다운로드`{: .text-blue}**한다. 즉, 스테이지 개수 자체가 오버헤드이다.

`download-config-file`, `prepare-source-file`, `install-dependencies`는 모두 "배포 준비"라는 하나의 맥락이었고, `test`와 `build`도 같은 소스를 대상으로 연속 실행되는 작업이었다. 이를 묶어 스테이지를 4개로 줄였다.

```yaml
stages:
  - clone-cdktf    # cdktf repository clone
  - prepare        # 소스 파일 및 설정 파일 구성
  - test-and-build # 테스트 및 빌드
  - deploy-{env}   # 환경별 배포 (develop/main 브랜치 전용)
```

> **스테이지 3개 축소 = artifact 업로드/다운로드 왕복 3회 제거**{: .text-blue}
{: .prompt-info }

### **2. Artifact에서 `.git` 디렉토리 제외**

가장 단순하면서 효과가 컸던 부분이다. 첫 구성에서는 Clone한 IaC 레포지토리를 artifact로 통째로 넘기면서 **`.git 디렉토리까지 포함`{: .text-red}**되어 있었다. 커밋 히스토리 전체가 매 스테이지마다 압축되고 전송되고 있었다.

```yaml
artifacts:
  paths:
    - ${CDKTF_REPO_PATH}/
  exclude:
    - ${CDKTF_REPO_PATH}/.git/ # .git 폴더 제외
```

배포에 `.git`은 필요하지 않으므로 모든 스테이지의 artifact에서 제외했다.

### **3. Terraform Provider 재다운로드 제거**

`deploy-{env}`가 **`5분 53초`{: .text-red}**나 걸린 핵심 원인이었다.

첫 구성에서는 배포 단계에서 `cdktf get`만 실행하고 바로 배포로 넘어갔다.

```yaml
# 개선 전
script:
  - npm run get --prefix ../${CDKTF_REPO_INFRA_PATH}
  - npm run deploy:dev:all
```

이 경우 `cdktf deploy`가 내부적으로 synth를 수행하면서 `cdktf.out/stacks/` 아래에 스택별 디렉토리를 생성하는데, 각 스택 디렉토리에 `.terraform.lock.hcl`이 없다 보니 **`스택마다 Terraform Provider를 새로 다운로드`{: .text-red}**하고 있었다. 스택이 늘어날수록 이 시간은 그대로 누적된다.

개선 후에는 배포 전에 synth를 명시적으로 실행하고, **레포지토리에 따로 커밋해둔 lock 파일**을 생성된 모든 스택 디렉토리에 복사하도록 했다.

```yaml
# 개선 후
before_script:
  - |
    cd ${CDKTF_REPO_INFRA_PATH}
    npm run get
    npm run synth
    cd ..

  - |
    TARGET_STACKS_ROOT="../${CDKTF_REPO_INFRA_PATH}/cdktf.out/stacks"
    TERRAFORM_LOCK_HCL_FILE_PATH="../${CDKTF_REPO_INFRA_PATH}/terraform/.terraform.lock.hcl"

    find "$TARGET_STACKS_ROOT" -mindepth 1 -maxdepth 1 -type d | while read TARGET_STACK_DIR; do
      cp -f ${TERRAFORM_LOCK_HCL_FILE_PATH} ${TARGET_STACK_DIR}/
    done
script:
  - npm run deploy:dev:all
```

**`.terraform.lock.hcl`{: .text-blue}**이 있으면 Terraform은 이미 고정된 Provider 버전을 사용하므로 재다운로드가 발생하지 않는다.

> **이 변경만으로 deploy-{env} 5분 53초 → 3분 4초, 약 48% 단축**{: .text-blue}
{: .prompt-info }

## **CI/CD 개선 결과**

![Improved CI/CD Result]({{site.url}}/assets/img/Serverless-Terraform/2-Improved_CICD_Result.png)

| **스테이지** | **소요 시간** |
| --- | --- |
| clone-cdktf | 00:00:14 |
| prepare | 00:00:41 |
| test-and-build | 00:01:01 |
| **deploy-dev** | **00:03:04** |
| **합계** | **00:05:00** |

| **구분** | **개선 전** | **개선 후** | **단축률** |
| --- | --- | --- | --- |
| 스테이지 수 | 7개 | 4개 | - |
| deploy-dev | 00:05:53 | 00:03:04 | **약 48%** |
| **전체** | **00:08:49** | **00:05:00** | **약 43%** |

> **전체 파이프라인 8분 49초 → 5분, 약 43% 단축**{: .text-blue}
{: .prompt-info }

## **여기까지의 정리**

레포지토리를 분리한 것은 성능이 아니라 **`관심사의 분리와 형상관리`{: .text-blue}**를 위한 선택이었다. 서비스 개발자는 더 이상 IaC 코드를 신경 쓰지 않아도 되고, 서비스의 커밋 히스토리에는 서비스 변경만 남게 되었다.

다만 분리 구조에서는 CI/CD가 IaC 레포지토리를 Clone해오는 과정이 추가되기 때문에, 파이프라인을 어떻게 구성하느냐에 따라 배포 시간이 크게 달라진다. 이번 개선에서 확인한 것은 결국 두 가지이다.

- **스테이지는 작업 단위가 아니라 전송 비용 단위로 나눠야 한다.** 스테이지를 잘게 쪼개면 가독성은 좋아지지만, **`artifact 왕복 비용`{: .text-red}**이 그대로 추가된다.
- **반복되는 다운로드는 캐시나 lock 파일로 고정해야 한다.** **`Terraform Provider 재다운로드`{: .text-red}** 하나가 전체 배포 시간의 절반 가까이를 차지하고 있었다.

## **프로젝트 구조 개선**

IaC 레포지토리를 들여다보니 배포 스크립트와 설정 양쪽에 **`분기가 과도하게 쌓여 가독성 이슈`{: .text-red}**가 있었다.

### **1. 하나의 스크립트가 모든 배포를 분기**

기존에는 `deploy.mts` 파일 하나가 배포, 삭제, synth, diff를 모두 처리했다. 명령은 서브커맨드로, 대상은 위치 인자로, 나머지는 플래그로 받는 구조였다.

```json
// 기존 package.json
{
  "deploy:dev":     "tsx scripts/deploy.mts --env dev",
  "deploy:dev:all": "tsx scripts/deploy.mts all --env dev",
  "destroy:dev":    "tsx scripts/deploy.mts destroy all --env dev",
  "diff:dev":       "tsx scripts/deploy.mts diff --env dev"
}
```

문제는 인자를 해석하는 단계부터 분기가 겹친다는 점이었다. 입력 축이 네 개였고, 그 중 일부는 서로를 참조했다.

| **입력 축** | **받는 방식** | **해석 조건** |
| --- | --- | --- |
| 명령 | 첫 번째 위치 인자 | 미리 정한 값이 아니면 기본값 `deploy`로 간주 |
| 타겟 | 위치 인자 | **명령이 무엇이냐에 따라 읽는 위치가 밀린다** |
| 서브 타겟 | 위치 인자 | 위와 동일 |
| 동작 옵션 | 플래그 3종 | 이후 단계의 실행 여부를 각각 토글 |

**`같은 자리의 인자가 명령에 따라 다른 의미를 가지는 구조`{: .text-red}**였다. 그래서 인자를 읽는 시점에 명령 종류를 먼저 판단해야 했고, 여기서 만들어진 값들이 이후 "배포일 때만", "삭제가 아닐 때만", "배포이면서 빌드를 건너뛰지 않을 때만" 같은 조건으로 계속 조합되었다.

| **항목** | **값** |
| --- | --- |
| 파일 | `scripts/deploy.mts` |
| 전체 줄 수 | **634줄** |
| `if` 분기 | **40개** |
| 삼항 연산자 | 12개 |

동작은 하지만 **`지금 이 명령이 어떤 경로를 타는지 알려면 파일 전체를 따라가야 하는 상태`{: .text-red}**였다.

### **2. 역할별 스크립트로 분리**

기준을 하나로 잡았다.

> **하나의 스크립트는 하나의 책임만 가진다. `--stack`, `--step` 같은 파라미터로 한 스크립트가 여러 역할을 분기하지 않는다.**{: .text-blue}
{: .prompt-info }

**`환경(env)만 인자로 받고, 역할은 파일로 나눈다`{: .text-blue}**는 방향이다. 스택 하나에 배포 스크립트 하나를 1:1로 대응시켰다.

```text
scripts/
├─ deploy-bootstrap.mts   # tfstate S3 + lock DynamoDB + artifact 버킷
├─ deploy-iam.mts         # Lambda 실행 role, API GW 로깅 role
├─ deploy-layer.mts       # Layer 배포
├─ deploy-lambda.mts      # 핸들러별 Lambda 배포
├─ deploy-apigw.mts       # API Gateway 경로 매핑
├─ deploy-all.mts         # 순서와 조건만 담당하는 오케스트레이터
└─ lib.mts                # 공통 유틸 (인자 파싱, cdktf 실행, 네이밍)
```

개별 배포 스크립트는 이 정도로 줄었다.

```typescript
/** deploy-lambda — 핸들러별 Lambda(단일 스택) 배포. 라우트 추출·빌드가 선행돼야 한다. */
import { cdktf, loadEnvFile, parseArgs, stackId } from "./lib.mts";

const { env } = parseArgs();
loadEnvFile(env);
cdktf("deploy", [stackId(env, "lambda")], { env, autoApprove: true });
```

분기가 사라지고 **`무엇을 배포하는지가 파일 이름과 6줄 안에서 끝난다`{: .text-blue}**. 순서와 조건은 오케스트레이터인 `deploy-all.mts`가 전담하고, 실제 동작은 각 역할 스크립트에 위임한다.

| **구분** | **개선 전** | **개선 후** |
| --- | --- | --- |
| 배포 스크립트 | `deploy.mts` 1개 (634줄) | 역할별 5개 (각 5~6줄) |
| 오케스트레이션 | 같은 파일 안에서 분기 | `deploy-all.mts` (52줄) |
| 공통 로직 | 파일 내부에 혼재 | `lib.mts` (83줄) |

부분 배포도 쉬워졌다. Lambda만 바뀌었으면 `build-lambda` → `deploy-lambda`만 호출하면 된다.

### **3. 설정 값의 환경 분기 제거**

설정 쪽에도 같은 문제가 있었다. 기존에는 **`값 하나 안에 환경별 분기 맵`{: .text-red}**을 둘 수 있었다.

```json
// 기존 - 값 안에서 환경을 분기
{
  "lambda": {
    "memory":  { "dev": 256, "prod": 1024, "default": 512 },
    "timeout": { "dev": 10,  "prod": 30,   "default": 15 },
    "reservedConcurrency": { "prod": 50 }
  }
}
```

문제는 여기에 `common.json → {env}.json → lambdas/{function}.json` 이라는 **파일 계층 override가 함께 존재**했다는 점이다. 분기가 두 축으로 겹치다 보니, 특정 환경의 특정 함수에 최종적으로 어떤 값이 들어가는지 확인하려면 세 개의 파일과 값 내부의 맵을 동시에 따라가야 했다.

개선본에서는 값 내부 분기를 없앴다.

> **모든 값은 단일 플랫 값이다. 환경 차이는 `{env}.json` 한 곳에서만 override 한다.**{: .text-blue}
{: .prompt-info }

```json
// 개선 - common.json 은 플랫 값만 가진다
{
  "lambda": { "memory": 128, "timeout": 15, "runtime": "nodejs22.x" }
}
```

```json
// prod 만 올리려면 prod.json 을 직접 수정한다
{
  "lambda": { "memory": 256 }
}
```

분기 축이 **`값과 파일 계층 2개에서 파일 계층 1개로`{: .text-blue}** 줄었다. 최종 값을 알고 싶으면 파일을 위에서 아래로 한 번만 따라가면 된다.

### **4. 설정 파일 단위 변경**

기존에는 함수 하나마다 설정 파일을 만들었다(`lambdas/{function}.json`). 핸들러가 늘어날수록 파일 수가 그대로 늘어난다.

개선본에서는 **`컨트롤러 하나가 파일 하나`{: .text-blue}**를 담당하고, 핸들러별 차이는 파일 안의 `handlers`에서 처리한다. 파일 경로는 컨트롤러 prefix를 그대로 따른다.

```jsonc
// config/lambdas/v1/users.json - 컨트롤러 하나의 설정
{
  "memory": 256,                 // 이 컨트롤러의 모든 핸들러 기본값
  "handlers": {
    "create": { "timeout": 30, "description": "사용자 생성" },
    "remove": { "description": "사용자 삭제" }
  }
}
```

우선순위는 **`핸들러 → 컨트롤러 → 공통`{: .text-blue}** 순으로 폴백한다. 대부분의 핸들러는 공통 기본값으로 충분하므로, `config/lambdas/`에는 **다른 값이 필요한 컨트롤러만** 만든다.

### **5. 레포지토리 간 역할 경계 고정**

마지막으로 두 레포지토리가 각각 무엇을 소유하는지 못박았다.

| **구분** | **소유** | **내용** |
| --- | --- | --- |
| 인프라 정의 | IaC 레포 | CDKTF 스택, 백엔드 설정 |
| 빌드·배포 | IaC 레포 | 빌드/배포 스크립트, esbuild 설정, CI 템플릿 |
| 비즈니스 로직 | 서비스 레포 | 컨트롤러, Service, Repository |
| 함수 설정 | 서비스 레포 | `config/`의 Lambda 설정과 env 키 |

핵심은 **`빌드와 배포의 모든 역할을 IaC 레포가 가져간 것`{: .text-blue}**이다. 서비스 레포는 소스(`app/`)와 설정(`config/`)만 제공하고, CI 파일은 IaC의 CI 템플릿을 include 하는 얇은 트리거만 남겼다.

```yaml
# 서비스 레포의 .gitlab-ci.yml 전체
include:
  - project: 'lillycover/backend/cdktf-iac/serverless-cdktf-iac'
    ref: main
    file: '/ci/serverless.gitlab-ci.yml'

# prod 도 수동 승인 없이 main push 시 바로 배포한다.
deploy-prod-job:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

| **구분** | **개선 전** | **개선 후** |
| --- | --- | --- |
| 서비스 레포 CI | 431줄 (파이프라인 전체 정의) | **24줄** (include + override) |
| 파이프라인 정의 위치 | 서비스 레포마다 복사 | IaC 레포 `ci/` 한 곳 |

서비스가 늘어나도 파이프라인을 복사할 필요가 없고, 배포 방식이 바뀌면 **`IaC 레포 한 곳만 고치면 모든 서비스에 반영`{: .text-blue}**된다.

### **구조 개선 후 배포 결과**

위 구조로 실제 배포한 결과이다.

![Restructured CI/CD Result]({{site.url}}/assets/img/Serverless-Terraform/3-Restructured_CICD_Result.png)

| **스테이지** | **소요 시간** |
| --- | --- |
| clone-iac | 00:00:05 |
| test-and-build | 00:01:04 |
| deploy | 00:03:55 |
| **합계** | **00:05:04** |

스테이지가 **`4개에서 3개로`{: .text-blue}** 한 번 더 줄었다. 앞서 `deploy-dev`와 `deploy-prod`를 각각의 스테이지로 두었던 것을 `deploy` 하나로 합치고, **`배포 대상 환경은 브랜치에 따라 결정되는 변수(TARGET_ENV)로 분리`{: .text-blue}**했기 때문이다. 스테이지를 늘리지 않고도 dev와 prod를 구분할 수 있게 되었다.

IaC 레포지토리를 Clone하는 시간도 **`14초에서 5초`{: .text-blue}**로 줄었다. 클론 결과를 넘길 때 `.git`을 제외하는 설정이 IaC 레포의 CI 템플릿에 기본으로 들어가 있어, 서비스 레포가 따로 신경 쓰지 않아도 적용된다.

## **CDKTF에서 Terraform으로**

### **왜 전환했는가**

가장 직접적인 이유는 외부 변수였다. **`CDKTF는 2025년 12월 10일자로 HashiCorp가 deprecated 처리하고 아카이브`{: .text-red}**했다. 저장소는 read-only가 되었고 provider 바인딩 생성도 멈췄다.

이 말은 **`@cdktf/provider-aws 20.1.0이 영구 상한`{: .text-red}**이 된다는 뜻이다. AWS가 새 리소스나 속성을 추가해도 CDKTF 쪽에서 받을 방법이 없다.

CDKTF의 출력물(`cdk.tf.json`)은 네이티브 Terraform JSON이라 당장 배포가 깨지지는 않는다. 그래서 더 급하게 보이지 않을 수도 있다. 문제는 이 레포지토리의 성격이었다.

> **앞으로 모든 서비스 레포지토리가 CI 템플릿을 include 해서 쓰게 될 플랫폼 레포지토리다. 유지 주체가 없는 도구에 계속 의존할 수 없다.**{: .text-blue}
{: .prompt-info }

서비스 하나짜리 레포지토리였다면 미뤄도 됐겠지만, 앞서 구조를 정리하면서 **`배포 방식이 바뀌면 IaC 레포 한 곳만 고치면 되는 구조`{: .text-blue}**로 만들어 둔 상태였다. 그 한 곳이 더 이상 유지되지 않는 도구 위에 있다는 것은, 앞으로 올라갈 모든 서비스가 같은 리스크를 공유한다는 뜻이다.

여기에 더해, 전환으로 함께 해결할 수 있는 것이 두 가지 있었다.

| **목표** | **내용** |
| --- | --- |
| 1. 서비스 개발자는 HCL을 몰라도 된다 | 서비스 레포가 만지는 파일은 한 글자도 바뀌지 않는다. `.tf`는 IaC 레포 안에만 존재한다 |
| 2. 배포 시간 절감 | lock 파일을 배포마다 복사해야 하던 구조를 근본적으로 해결 |
| 3. 의존성 감소 | `node_modules`에서 CDKTF 계열이 차지하던 용량 제거 |

### **CDKTF 기반 프로젝트와의 차이**

전환 전후로 수치가 크게 바뀐 항목은 네 가지다.

| **측정 항목** | **CDKTF** | **Terraform** |
| --- | --- | --- |
| `node_modules` | 884M (CDKTF 계열 612M, **69%**) | 약 **270M** |
| Provider 플러그인 캐시 | 작업 디렉토리가 **매번 새로 생성**되어 `.terraform.lock.hcl`을 배포마다 복사해 넣어야 함 | 작업 디렉토리가 **고정**이라 lock 파일을 커밋해 두면 끝 |
| 스택당 CLI 기동 | cdktf(Node + jsii) **4~5회** | **없음** (terraform 직접 호출) |
| API Gateway 리소스 수 | 라우트 수에 비례 (라우트 4건 → **18개**) | **상수 4개** + 함수당 permission |

#### **1. lock 파일을 복사하지 않아도 된다**

가장 중요한 차이다. 앞서 CI/CD 개선에서 `.terraform.lock.hcl`을 생성된 스택 디렉토리마다 복사해서 Provider 재다운로드를 막았는데, 그건 사실 우회책이었다.

CDKTF는 배포할 때마다 `cdktf.out/stacks/` 아래에 작업 디렉토리를 **새로 만든다.** 매번 새로 생기는 디렉토리이므로 lock 파일을 그 안에 커밋해 둘 수가 없다. 그래서 아래와 같은 구조로 떠받치고 있었다.

**첫째, lock 파일을 CDKTF와 무관한 경로에 따로 보관**했다. `infra/terraform/`은 CDKTF가 쓰는 경로가 아니라 **`lock 파일 하나만 들어있는 디렉토리`{: .text-red}**였다.

```text
infra/terraform/
└─ .terraform.lock.hcl    # 이 파일 하나뿐
```

**둘째, 배포할 때마다 생성된 스택 디렉토리에 복사**해 넣었다. 이 복사가 끝난 뒤에야 `terraform init`이 고정된 버전을 인식한다.

즉 **`도구가 지원하는 방식이 아니라 CI 스크립트로 떠받치는 구조`{: .text-red}**였다. 복사 단계를 빠뜨리면 에러 없이 조용히 Provider를 다시 받는다.

Terraform HCL은 **`작업 디렉토리가 고정`{: .text-blue}**이다.

```text
infra/terraform/
├─ bootstrap/   main.tf  variables.tf  provider.tf  .terraform.lock.hcl
├─ iam/         main.tf  variables.tf  provider.tf  .terraform.lock.hcl
├─ layer/       main.tf  variables.tf  provider.tf  .terraform.lock.hcl
├─ lambda/      main.tf  variables.tf  provider.tf  .terraform.lock.hcl
└─ apigateway/  main.tf  variables.tf  provider.tf  .terraform.lock.hcl
```

**`스택이 곧 디렉토리`{: .text-blue}**가 되므로 lock 파일을 그 자리에 두고 커밋할 수 있다. 복사하는 단계 자체가 사라진다.

lock 파일을 커밋해야 하는 이유는 하나 더 있다. 이 파일에는 provider 주소·버전·체크섬만 들어가고 **서비스 고유 정보는 하나도 없다.** 커밋해 두지 않으면 같은 IaC 코드를 쓰는데도 **`서비스마다 배포 시점에 따라 다른 patch 버전의 provider가 올라간다`{: .text-red}**.

#### **2. TypeScript와 HCL의 역할 분리**

CDKTF에서는 TypeScript가 설정 해석부터 리소스 선언까지 전부 담당했다. 전환하면서 이 둘을 갈랐다.

```text
TypeScript   config 해석·검증·라우트 추출·이름 조립  →  .generated/tfvars/{stack}.json
HCL          리소스 선언 (for_each)
```

서비스마다 다른 부분은 **`전부 TypeScript가 흡수`{: .text-blue}**하고, HCL은 해석이 끝난 평평한 맵을 받기만 한다. 그래서 HCL 쪽은 서비스가 몇 개든 그대로다.

```hcl
# infra/terraform/lambda/main.tf
resource "aws_lambda_function" "fn" {
  for_each = var.functions

  function_name = each.value.lambda_name
  description   = each.value.description
  role          = data.aws_iam_role.exec.arn
  ...
}
```

배포 스크립트의 구조는 **`CDKTF 때 잡아둔 역할별 분리를 그대로 유지`{: .text-blue}**했다. 내부에서 호출하는 대상만 `cdktf`에서 `terraform`으로 바뀌었다.

#### **3. API Gateway 리소스 수**

CDKTF에서는 경로를 재귀적으로 훑어 라우트마다 Resource·Method·Integration을 만들었다. 라우트가 늘어나면 리소스도 그만큼 늘어난다.

전환하면서 **`OpenAPI body 방식`{: .text-blue}**으로 바꿨다. TypeScript가 라우트를 `경로 → 메서드 → 함수` 형태로 그룹핑해서 넘기면, HCL은 Lambda ARN만 채워 하나의 문서로 만든다.

```hcl
resource "aws_api_gateway_rest_api" "api" {
  name              = var.api_name
  body              = local.body          # OpenAPI 문서 한 덩어리
  put_rest_api_mode = "overwrite"
  endpoint_configuration { types = ["REGIONAL"] }
}
```

라우트 4건 기준으로 **`18개였던 리소스가 상수 4개`{: .text-blue}**로 줄었다. 서비스마다 경로 깊이와 라우트 수가 달라도 IaC를 수정할 필요가 없다는 점이 더 중요하다.

### **서비스 레포지토리는 바뀌지 않았다**

전환에서 가장 신경 쓴 부분이다. IaC 도구를 통째로 바꿨지만 **`서비스 레포지토리가 만지는 파일은 한 글자도 바뀌지 않았다`{: .text-blue}**.

| **구분** | **전환 영향** |
| --- | --- |
| 서비스 코드(`app/`) | 변경 없음 |
| 설정(`config/*.json`) | 변경 없음 |
| 서비스 레포 `.gitlab-ci.yml` | 변경 없음 (include 그대로) |
| CI 템플릿의 잡·변수 이름 | 변경 없음 (서비스와의 계약) |
| 라우팅 선언 방식 | 변경 없음 |

앞서 **`빌드와 배포의 모든 역할을 IaC 레포가 가져가도록`{: .text-blue}** 경계를 정리해 둔 것이 여기서 효과를 봤다. 서비스 레포가 빌드나 배포 로직을 조금이라도 들고 있었다면, IaC 도구를 바꿀 때 모든 서비스 레포를 같이 수정해야 했을 것이다.

### **Terraform 전환 결과**

전환 후 같은 서비스를 배포한 결과이다.

![Terraform CI/CD Result]({{site.url}}/assets/img/Serverless-Terraform/4-Terraform_CICD_Result.png)

| **스테이지** | **소요 시간** |
| --- | --- |
| clone-iac | 00:00:08 |
| test-and-build | 00:00:43 |
| **deploy** | **00:01:45** |
| **합계** | **00:02:36** |
| plan *(합계 제외)* | 00:01:08 |

CDKTF에는 없던 **`plan 스테이지를 추가`{: .text-blue}**했다. 배포 과정에서 **`사전 체크를 위해 넣은 단계`{: .text-blue}**로, 실제로 리소스를 바꾸지 않고 어떤 변경이 일어날지 먼저 확인한다. 배포 자체에 드는 시간이 아니므로 합계에서는 제외했다.

### **CDKTF와 Terraform 배포 시간 비교**

두 배포는 **동일한 조건**에서 진행했다. 배포한 리소스도 같다.

| **리소스** | **개수** |
| --- | --- |
| API Gateway | 1개 |
| Lambda | 7개 |
| Layer | 1개 |

| **스테이지** | **CDKTF** | **Terraform** | **차이** |
| --- | --- | --- | --- |
| clone-iac | 00:00:05 | 00:00:08 | +3초 |
| test-and-build | 00:01:04 | 00:00:43 | -21초 |
| **deploy** | **00:03:55** | **00:01:45** | **-2분 10초 (약 55%)** |
| **합계** | **00:05:04** | **00:02:36** | **약 49% 단축** |
| plan *(합계 제외)* | - | 00:01:08 | 사전 체크 목적으로 신규 추가 |

핵심은 `deploy`다. **`3분 55초에서 1분 45초로 약 55% 줄었다`{: .text-blue}**. Provider 플러그인 캐시가 제대로 동작하면서 스택마다 반복되던 Provider 다운로드가 사라졌고, 스택마다 4~5회씩 뜨던 cdktf CLI 기동도 없어진 결과다.

> **전체 배포 시간이 5분 04초에서 2분 36초로 절반 이하가 되었다.**{: .text-blue}
{: .prompt-info }

`test-and-build`가 21초 줄어든 것도 전환 효과다. `node_modules`에서 CDKTF 계열 612M이 빠지면서 의존성 설치와 캐시 압축·전송량이 함께 줄었다.

## **정리**

시작은 성능이 아니었다. **`백엔드 담당자가 개인 로컬에서 배포하고 그 다음에 커밋하는 구조`{: .text-red}**를 바꾸고, 프로젝트 버전 관리가 가능하도록 만드는 것이 목적이었다.

그 목적을 위해 레포지토리를 분리했고, 분리하면서 드러난 문제를 따라가다 보니 파이프라인과 프로젝트 구조, 그리고 IaC 도구까지 차례로 손보게 되었다.

### **배포 시간 변화**

| **단계** | **스테이지 수** | **배포 시간** |
| --- | --- | --- |
| 첫 분리 배포 | 7개 | 00:08:49 |
| CI/CD 파이프라인 개선 | 4개 | 00:05:00 |
| 프로젝트 구조 개선 | 3개 | 00:05:04 |
| **Terraform 전환** | **3개** (+ plan) | **00:02:36** |

> 앞의 두 단계는 개발(dev) 배포, 뒤의 두 단계는 운영(prod) 배포 결과라 완전히 같은 조건은 아니다. 다만 **`CDKTF와 Terraform 비교는 동일한 조건`{: .text-blue}**(API Gateway 1개, Lambda 7개, Layer 1개)에서 측정했다.
{: .prompt-info }

### **앞으로**

최종 목표까지 두 가지가 남아 있다.

#### **1. 모든 서버리스 백엔드 프로젝트를 현재 구조로 전환**

지금까지의 작업은 템플릿 레포지토리를 기준으로 진행했다. **`운영 중인 서버리스 백엔드 프로젝트를 모두 이 구조로 Migration`{: .text-blue}**하는 것이 다음 목표다.

서비스 레포지토리가 가져야 할 것은 소스(`app/`)와 설정(`config/`), 그리고 IaC의 CI 템플릿을 include 하는 얇은 `.gitlab-ci.yml`뿐이므로, 전환 자체의 부담은 크지 않을 것으로 보고 있다. 다만 서비스마다 Lambda 구성 방식이 달라 함수 단위와 엔드포인트가 바뀔 수 있어, 서비스별로 확인하면서 옮겨야 한다.

#### **2. 변경된 대상만 배포하도록 개선**

현재 구조의 가장 큰 숙제다. API Gateway, Lambda, Layer는 **`변경이 있을 때만 배포하면 되는 리소스`{: .text-blue}**인데, 지금은 **`변경이 있든 없든 매번 전부 배포`{: .text-red}**하고 있다.

Lambda 하나의 코드만 고쳐도 Layer를 다시 올리고 API Gateway를 다시 배포하는 식이다. 실제로 바뀐 것은 함수 하나뿐인데 전체 배포 시간을 그대로 지불하고 있는 셈이다.

변경 대상을 판별해서 **`해당 서비스와 해당 리소스만 배포`{: .text-blue}**하도록 바꾸면, 지금의 2분 36초에서 배포 시간이 다시 한 번 크게 줄어들 가능성이 크다.

처음 목적이었던 **`배포의 기준을 개인 로컬에서 레포지토리로 옮기는 것`{: .text-blue}**은 달성했다. 이제 어떤 코드가 어느 환경에 배포되어 있는지 커밋 하나로 확인할 수 있다.
