---
title: Jira & Github 연동을 통한 이슈 처리 자동화
# author: dowonl2e
date: 2025-03-15 07:00:00 +0800
categories: [Jira]
tags: [Jira, Github]
pin: true
img_path: "/assets"
image:
  path: /commons/Jira.png
  alt: Jira
---

## **Jira란?**

Jira는 Atlassian에서 개발한 프로젝트 관리 및 이슈 추적 도구이다. 소프트웨어 개발은 물론이며 팀 협업이나 업무 관리에서도 폭넓게 이용할 수 있다. 특히, Agile(애자일) 개발 방법론을 지원하며 다양한 템플릿을 지원한다.

### **Jira의 구성 요소**

- **이슈(Issue)**: Jira에서 작업 항목을 나타내는 기본 단위입니다. 이슈는 버그, 스토리, 태스크 등 다양한 형태로 관리됩니다.
- **프로젝트(Project)**: 이슈를 그룹화하는 단위로, 일반적으로 팀이나 부서에 의해 관리됩니다.
- **워크플로우(Workflow)**: 이슈가 진행되는 상태의 흐름을 정의합니다. 예를 들어, "To Do → In Progress → Done"처럼 상태를 정의할 수 있습니다.
- **보드(Board)**: 스크럼 보드 또는 칸반 보드를 통해 이슈의 진행 상태를 시각적으로 확인할 수 있습니다.
- **스프린트(Sprint)**: Agile 방식의 스크럼에서 일정 기간 동안 완료할 작업을 계획하는 단위입니다.

### **Jira의 활용 예시**

- **소프트웨어 개발**: 개발 팀에서 버그, 기능 추가, 스프린트 등을 관리에 사용
- **지원 및 고객 서비스**: 고객 지원팀이 고객 요청이나 문제를 추적하고 해결에 사용
- **프로젝트 관리**: 다양한 업무를 관리하고 팀 간의 협업을 촉진하기 위한 도구로 활용

### **Jira의 장점**

- **협업 기능**: 팀원 간의 협업을 지원하고, 실시간으로 이슈를 관리할 수 있습니다.
- **맞춤화 가능**: 워크플로우, 대시보드, 알림 등을 사용자의 필요에 맞게 조정할 수 있습니다.
- **강력한 보고서 및 대시보드**: 프로젝트 성과를 분석하고 팀의 생산성을 높이는 데 도움이 됩니다.
- **통합 기능**: 다양한 개발 도구 및 팀 협업 툴과의 원활한 통합을 제공합니다.

### **Jira의 단점**

- **복잡성**: 다양한 기능을 제공하는 만큼, 초보자에게는 설정이나 사용이 복잡할 수 있습니다.
- **비용**: 유료 버전에서 많은 기능이 제공되기 때문에, 예산이 제한된 경우에는 비용 부담이 있을 수 있습니다.

### **이슈(Issue)의 유형**

| 유형 | 설명 |
| --- | --- |
| 에픽(Epic) | 큰 규모의 작업을 나타낸다. 대개 여러개의 이슈 모음을 나타낸다. |
| 스토리(Story) | 사용자 관점에서 표현한 요구사항을 나타낸다. |
| 작업(Task) | 해야 할 작업을 나타낸다. 작업은 포괄적으로 사용되며 다른 이슈 유형으로는 작업을 정확하게 나타낼 수 없을 때 사용한다. |
| 버그(Bug) | 해결해야 할 문제를 나타낸다. |
| 하위 작업 | 표준 이슈를 완료하는 데 필요한 작업을 더 세부적으로 분해한 것을 나타냅니다. 모든 이슈 유형에 대해 하위 작업을 만들 수 있습니다. <br/> 이슈를 만든 후 추가할 수 있는 하위 이슈이다. |

> 참고 : [https://www.atlassian.com/ko/software/jira/guides/issues/overview#what-are-issue-types](https://www.atlassian.com/ko/software/jira/guides/issues/overview#what-are-issue-types){:target="\_blank"}

설명만으로는 와닫지 않아 진행중인 여행 플래너 서비스를 기반으로 각 유형을 구분해야한다면 아래와 같지 않을까 싶다.

- **에픽(Epic)**: 여행 플래너 서비스 개발/관리
- **스토리(Story)**: 여행 플랜 등록, 수정, 삭제
- **작업(Task)**: 서비스 로직 변경, 성능 개선
- **버그(Bug)**: 일정 거리순 정렬시 순서가 틀림

## **Jira와 Github 연동**

Jira와 Github를 연동으로 **소프트웨어 개발** 및 **프로젝트 관리**를 더 효율적으로 할 수 있게된다.

이 연동을 통해 **개발자**는 Github에서 코드 변경 작업을 할 때, Jira에서 이슈를 추적하고 **프로젝트 관리** 기능을 활용할 수 있습니다. 또한, **이슈 상태 업데이트** 및 **자동화**를 통해 보다 **효율적인 개발 프로세스**를 구현할 수 있습니다.

### **개발자 관점에서의 Jira와 Github의 프로젝트 관리**

1. Jira 프로젝트에서 이슈와 Branch를 생성한다.
2. IDE에서 해당 이슈를 처리 한 후 Github에 Push한다.
3. Github에서는 Pull Request을 하여 코드 병합을 진행한다.
4. 코드 병합이 완료되면 Jira에서 생성한 이슈를 완료 처리한다.

프로젝트를 더 효율적으로 관리할 수 있는 방법이 있으나, 목표하는 프로젝트 관리 방법은 위와 같이 진행하고자 한다.

## **Jira와 Github 설정**

### **Github 앱 설정**

앱 메뉴에서 Github 검색 후 **`Github for Jira`**를 추가해준다.

![Jira Github App]({{site.url}}/assets/img/Jira/1-Github_App.png)

연결하고자 하는 Github Repository를 추가한다.

![Jira Github Repository]({{site.url}}/assets/img/Jira/2_1-Github_Repository.png)

### **Github에서 확인**

`Repository → Settings → Integrations → Github Apps`에 Jira가 연동된 것을 확인할 수 있으며 세부 구성을 보면 엑세스된 Repositories들을 볼 수 있다.

![Jira Github Integration]({{site.url}}/assets/img/Jira/2_3-Github_Integration_2.png)

![Jira Github Integration Jira Configuration]({{site.url}}/assets/img/Jira/2_4-Github_Integration_3.png)

## **Jira 프로젝트 만들기**

### **프로젝트 템플릿 및 생성**

애자일(Agile) 방법론에 대해서 적절한 템플릿으로 칸반(Kanban)을 설정했다.

![Jira Project]({{site.url}}/assets/img/Jira/3-Jira_Project.png)

MSA에 대한 환경 구성과 과정으로 팀에서 관리하는 프로젝트를 선택했다.

![Jira Project Create 1]({{site.url}}/assets/img/Jira/4-Jira_Project_Create_1.png)

프로젝트명, 키, 엑세스를 설정해준다. 키는 이슈 생성시 키를 기반으로 이슈의 키가 생성된다. 

`ex) 키가 TP인 경우 이슈 생성시 키는 TP-1로 자동 생성된다.`

![Jira Project Create 2]({{site.url}}/assets/img/Jira/5-Jira_Project_Create_2.png)

프로젝트에서 관리할 Github Repository를 선택해준다. 필요에 따라 다수 리포지토리를 선택할 수 있다.

![Jira Project Create 3]({{site.url}}/assets/img/Jira/6-Jira_Project_Create_3.png)

## **Jira 이슈 생성 및 관리**

### **이슈 생성**

`프로젝트 → 타임라인` 메뉴에서 `에픽`{: .text-purple}과 `하위이슈`{: .text-blue}를 등록할 수 있다.

![Jira Project Time Line]({{site.url}}/assets/img/Jira/7-Jira_Time_Line.png)

`프로젝트 → 보드` 메뉴에서 **할일(To Do)**, **진행중(In Progress)**, **완료(Done)**로 이슈를 등록 후 상태를 변경할 수 있다.

![Jira Project Board]({{site.url}}/assets/img/Jira/8-Jira_Board.png)

`프로젝트 → 이슈` 메뉴에서 타임라인 혹은 보드 등 생성한 에픽과 이슈를 확인할 수 있으며 프로젝트 관리에 필요한 브랜치와 커밋을 만들 수 있다.

![Jira Project Issue]({{site.url}}/assets/img/Jira/9-Jira_Issue.png)

### **이슈 브랜치 만들기**

이슈 메뉴에서 선택한 이슈에서 브랜치 만들기를 선택하면 리포지토리와 브랜치 정보를 확인할 수 있으며 생성할 브랜치명은 **`Issue 키와 이름`{: .text-blue}**으로 자동 생성되며, Github Repository에서 생성된 브랜치를 확인할 수 있다.

![Jira Project Issue Branch]({{site.url}}/assets/img/Jira/10-Jira_Issue_Branch.png)

## **이슈 자동화 임시 테스트**

### **Workflow Dispatch 생성**

```bash
# .github/workflows/dispatch-jira-update-on-merge.yml

name: Update Jira Issue on PR Merge

on: 
  workflow_dispatch:
    inputs:
      branch:
        description: "Branch to run workflow"
        required: true
        default: "main"

jobs:
  update-jira:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout selected branch
        uses: actions/checkout@v4
        with:
          ref: $[[ github.event.inputs.branch ]]
          
      - name: Debug Input Branch
        run: |
          echo "GitHub Input Branch: $[[ github.event.inputs.branch ]]"
        
      - name: 1. Extract Jira Issue Key
        id: extract_jira_issue_key
        run: |
          JIRA_KEY=$(echo "$[[ github.event.inputs.branch ]]" | grep -oE '[A-Z]+-[0-9]+')
          echo "JIRA_KEY=$JIRA_KEY" >> $GITHUB_ENV
          
      - name: 2. Echo Jira Key
        run: |
          echo "$JIRA_KEY"
```

- 생성한 Workflow를 수동으로 실행하려면 Workflow Dispatch를 통해 실행할 수 있다. PR의 경우 특정 Branch Commit된 이력을 Merge하기에 대상 Branch를 찾을 수 있으나, Workflow Dispatch에서 테스트해보았을 때 main branch만 조회되어 Inputs을 통해 직접 대상 Branch를 입력할 수 있도록 하였다.
- `grep -oE '[A-Z]+-[0-9]+'`{: .text-blue}:  리눅스 기반 시스템에서 문자열에서 관련 패턴을 찾아주는 명령어로 입력받은 Branch에 패턴과 일치하는 문자열을 찾아 JIRA_KEY 환경 변수로 값을 넣어준다.

> "[[", "]]"를 중괄호로 변경 필요

![Github Workflow Dispatch Test 1]({{site.url}}/assets/img/Jira/11-Jira_Workflow_Dispatch_Test_1.png)

![Github Workflow Dispatch Test 2]({{site.url}}/assets/img/Jira/12-Jira_Workflow_Dispatch_Test_2.png)

## **Jira & Github 이슈 자동화**

브랜치가 생성된 이후에 사용중인 IDE에서 브랜치를 받은 후 작업을 완료하고 Github에 Push한다. 이후 PR(Pull Request)를 통해 Merge하는 과정을 거치게 된다. 이 과정에서 Github에서 PR이후 Jira에 의해 생성된 브랜치에 대해서 **`완료`**처리를 진행해야한다.

일반적인 상태에서는 수동으로 완료처리를 할 수 있지만 Github와 Jira 연동을 통해 이슈 처리를 자동화할 수 있다.

### **Github Repository Secrets 추가**

`Settings → Secrets and variables → Actions`에서 JIRA에 대한 Secrets를 추가해준다.

- **JIRA_BASE_URL** : Jira 기본 URL `예) https://test.atlassian.net`
- **JIRA_USER_EMAIL** : Jira 사용자 이메일
- **JIRA_API_TOKEN** : Jira API 토큰
  - Jira API 토큰의 경우 아래 링크를 통해 생성 후 따로 저장하면된다.
  - [https://id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens){:target="\_blank"}

### **Jira 상태 변경을 위한 API 체크**

#### **완료 상태 Transitions Id 조회 API**

```bash
curl --request GET \
  --url "{Jira Base Url}/rest/api/3/issue/{Issue Key}/transitions" \
  --user "{Jira 계정 이메일}:{Jira API 토큰}" \
  --header "Accept: application/json"
```

위 명령어를 통해 완료 상태에 대한 Transition Id를 조회하면 아래와 같은 형식의 Json 응답 데이터를 확인할 수 있으며 완료 상태의 Transition ID를 찾는다.

```json
{
  "expand": "transitions",
  "transitions": [

    {
      "id": "{Transition Id}",
      "name": "완료",
      "to": {

      }
    },
  ]
}
```

#### **완료 상태 변경 API**

```bash
curl --request POST \
--url "{Jira Base Url}/rest/api/3/issue/{Issue Key}/transitions" \
--user "{Jira 사용자 이메일}:{Jira API 토큰}" \
--header "Accept: application/json" \
--header "Content-Type: application/json" \
--data '{
    "transition": {
      "id": “{Transition Id}“ 
    }
  }’

# 정상 처리에 대한 응답코드: 204
```

### **PR(Pull Request) Workflow 생성**

```yaml
# .github/workflows/jira-update-on-merge.yml

name: Update Jira Issue on PR Merge

on:
  pull_request:
    types:
      - closed  # PR이 닫힐 때 (Merged 포함)

jobs:
  update-jira:
    runs-on: ubuntu-latest
    if: github.event.pull_request.merged == true  # PR이 머지된 경우에만 실행됨

    steps:
      - name: 1. Extract Jira Issue Key
        id: extract_jira_issue_key
        run: |
          JIRA_KEY=$(echo "$[[ github.head_ref ]]" | grep -oE '[A-Z]+-[0-9]+')
          echo "JIRA_KEY=$JIRA_KEY" >> $GITHUB_ENV

      - name: 2. Update Jira Issue to Done
        id: update_jira_issue_to_done
        if: env.JIRA_KEY != '' # 환경변수 JIRA_KEY 값이 있을 경우에만 실행
        run: |
          response=$(curl --write-out "%{http_code}" --silent --output /dev/null --request POST \
            --url "$[[ secrets.JIRA_BASE_URL ]]/rest/api/3/issue/${JIRA_KEY}/transitions" \
            --user "$[[ secrets.JIRA_USER_EMAIL ]]:$[[ secrets.JIRA_API_TOKEN ]]" \
            --header "Accept: application/json" \
            --header "Content-Type: application/json" \
            --data '{
              "transition": {
                "id": "{Transition id}"
              }
            }')

          # 응답 코드가 204이 아니면 오류 처리
          if [ "$response" -ne 204 ]; then
            echo "Error: Failed to transition Jira issue $JIRA_KEY. Received HTTP status code $response"
            exit 1
          else
            echo "Jira issue $JIRA_KEY successfully transitioned to Done."
          fi
```

1. **Pull Request**가 될 때 작업을 진행할 수 있으며 closed전 merged 상태의 경우에 작업을 진행할 수 이도록 한다.
2. **Extract Jira Issue Key**: PR 병합 대상 브랜치에서 Issue Key 추출한다. `예: TP-123`
  1. **`github.head_ref`{: .text-blue}**: Pull Request가 생성될 때의 원본 브랜치를 나타낸다.
3. **Update Jira Issue to Done**: Jira 상태 변경에 대한 Request를 진행하며 이후 응답 결과에 따라 Workflow의 결과를 처리한다.
  1. **Transition id**: **`완료`{: .text-blue}**에 대한 Transition Id 값으로 지정해준다.

> "[[", "]]"를 중괄호로 변경 필요

#### **Workflow 결과**

![Github Workflow Jira Update On Merge Result 1]({{site.url}}/assets/img/Jira/13_Jira_Update_On_Merge_Result_1.png)

![Github Workflow Jira Update On Merge Result 2]({{site.url}}/assets/img/Jira/14_Jira_Update_On_Merge_Result_2.png)

![Github Workflow Jira Update On Merge Result 3]({{site.url}}/assets/img/Jira/15_Jira_Update_On_Merge_Result_3.png)
