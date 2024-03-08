---
title: Spring Boot + Docker + Github Actions를 활용한 CI/CD 환경 구축
# author: dowonl2e
date: 2024-03-07 13:00:00 +0800
categories: [DevOps, CI/CD]
tags: [Spring Boot, AWS, Docker, Github Actions, CI/CD]
pin: true
---

재고관리 시스템을 운영할 때 초기에는 개발 기간이 짧아 급한대로 수동배포를 했으나, 빌드/테스트 과정에서 체크해야하며 배포 과정에서 불필요한 설정도 같이 배포되어 재작업으로 인한 많은 시간소요와 불편함이 있어 당시에 구축한 CI/CD 환경을 정리해보고자 합니다.

환경 구축 전 CI/CD에 대한 정의를 보겠습니다.

## **CI/CD**

### **CI란?**

CI는 간단히 요약하자면 **빌드/테스트 자동화 과정**{: .text-blue }입니다. CI는 개발자를 위한 자동화 프로세스인 **지속적인 통합(Continuous Integration)**{: .text-blue }을 의미합니다. CI를 성공적으로 구현할 경우 애플리케이션에 대한 새로운 코드 변경 사항이 정기적으로 빌드 및 테스트되어 공유 리포지토리에 통합되므로 여러 명의 개발자가 동시에 애플리케이션 개발과 관련된 코드 작업을 할 경우 서로 충돌할 수 있는 문제를 해결할 수 있습니다.

지속적 통합의 실행은 소스/버전 관리 시스템에 대한 변경 사항을 정기적으로 커밋하여 모든 사람에게 동일 작업 기반을 제공하는 것으로 시작합니다. 커밋할 때마다 빌드와 일련의 자동 테스트가 이루어져 동작을 확인하고 변경으로 인해 문제가 생기는 부분이 없도록 보장합니다. 지속적 통합은 그 자체로 유익하지만 **CI/CD 파이프라인을 구현하기 위한 첫 번째 단계**이기도 합니다.

### **CD란?**

CD는 간단히 말하면 **배포 자동화 과정**{: .text-blue }입니다. CD는 **지속적인 서비스 제공(Continuous Delivery)**{: .text-blue } 또는 **지속적인 배포(Continuous Deployment)**{: .text-blue }를 의미합니다.

> 이 두 용어는 상호 교환적으로 사용됩니다. 두 가지 의미 모두 파이프라인의 추가 단계에 대한 자동화를 뜻하지만 때로는 얼마나 많은 자동화가 이루어지고 있는지를 설명하기 위해 별도로 사용되기도 합니다.

지속적 배포는 빌드, 테스트 및 배포 단계를 자동화하는 DevOps 방식을 논리적 극한까지 끌어 올립니다. 코드 변경이 파이프라인 첫 단계인 빌드/테스트 과정을 모두 성공적으로 통과하면 해당 변경 사항이 프로덕션에 자동으로 배포됩니다. 지속적 배포를 채택하면 품질 저하 없이 최대한 빨리 사용자에게 새로운 기능을 제공할 수 있습니다.

또한 입증된 지속적 통합 및 지속적인 전달 단계를 기반으로 합니다. 간단한 코드 변경이 정기적으로 커밋되고, 자동화된 빌드 및 테스트 프로세스를 거치며 다양한 사전 프로덕션 환경으로 승격되며, 문제가 발견되지 않으면 최종적으로 배포됩니다.

### **CI/CD 종류**

종류로는 Jenkins, **Github Actions**, CircleCI, TravisCI가 있으며 재고관리 시스템을 개발할때 Github를 사용하면서 친숙한 **Github Actions**를 활용했습니다.

## **프로젝트 환경**

- **Framework** : Spring Boot 2.7, JPA, Spring Data JPA, Querydsl
- **DevOps** : AWS EC2(Amazon Linux 2), Docker, Github Action, Gradle
- **Network** : Nginx
- **Database** : MySQL 8.0

배포할 프로젝트, 서버에 Docker, docker-compose 환경이 구성되어있다는 전제로 진행하겠습니다.

## **Docker + Github Actions CI/CD 과정**

리눅스 환경에서 Docker와 Github Actions를 활용한 CI/CD 과정을 정리해보았습니다.
![Docker + Github Actions CI/CD Flow]({{site.url}}/assets/img/Docker-GithubActions-CICD/1_CICD Flow.png)

CI/CD를 적용한 간략한 흐름은 위와 같습니다.

1. 배포할 소스를 Github에 Push 합니다.
2. Github Actions에서 빌드 및 테스트 과정을 시작합니다. 빌드 과정에서 문제가 없으면 Docker 빌드 작업을 시작합니다.
3. Docker에서 프로젝트의 Dockerfile 설정을 기반으로 빌드 후 Docker Repository에 Push하게 됩니다.
4. Docker Build & Push 과정에 문제가 없으면 Github Actions에서 AWS EC2 연결 후 배포 스크립트를 실행합니다. 배포 스크립트 과정은 아래와 같습니다.<br/>
    1) Docker 컨테이너 중지<br/>
    2) Docker 이미지 가져오기<br/>
    3) Docker 컨테이너 실행

> **컨테이너 실행이 완료되기 전까지 서비스는 중단 상태에 있습니다.**
{: .prompt-warning }

## **CI/CD 과정에 필요한 설정**

### **1) Dockerfile 생성**

```
FROM openjdk:17-alpine
ARG JAR_FILE=build/libs/jewelry-3.1.1-SNAPSHOT.jar
COPY ${JAR_FILE} jewelry.jar
ENTRYPOINT ["java", "-jar", "jewelry.jar"]
```

- **FROM** : Docker Image를 지정할 수 있습니다. 이미지 종류는 아래와 같으며 프로젝트 및 서버 구성에 맞게 설정하시면됩니다.
    - [https://hub.docker.com/search?q=IMAGE](https://hub.docker.com/search?q=IMAGE){:target="\_blank"}
- **ARG** : Dockerfile내에 변수를 지정할 수 있으며 저는 Spring 프로젝트이므로 jar 파일 경로 및 명칭을 지정해주었습니다.
- **ENV** : ARG와 비슷하지만 컨테이너에서 환경 변수로도 사용 가능합니다.
- **COPY** : Docker 컨테이너에 파일을 복사합니다.
- **ENTRYPOINT** : 컨테이너가 시작될 때 항상 실행할 명령어를 지정해줍니다.
    - **java** : Java 애플리케이션 실행
    - **-jar** : JAR 파일을 실행
    - **jewelry.jar** : 실행할 파일 명을 설정 (COPY에서 컨테이너에 jewelry.jar를 복사해주었기에 동일하게 적용했습니다.)
- **-Dspring.profiles.active** : 사용할 프로파일을 설정. 애플리케이션 속성을 로컬(local), 개발용(dev), 운영용(prod)으로 구분해 사용할 경우 `-Dspring.profiles.active=prod`를 추가하면 프로파일을 prod로 설정됩니다.

### **2) Github Action Workflow 생성**
![Github Actions Workflow]({{site.url}}/assets/img/Docker-GithubActions-CICD/2_Github Actions Workflow1.png)

Repository의 상단에 **Actions**통해 Workflow를 구성할 수 있으며 프로젝트 환경에 맞게 선택하면 됩니다.

> 프로젝트 및 서버 환경에 맞는 Workflow를 선택해야하며 필요한 Job 추가 또는 수정이 필요합니다.
{: .prompt-info }

### **3) Workflow 작성**

```
# This workflow uses actions that are not certified by GitHub.
# They are provided by a third-party and are governed by
# separate terms of service, privacy policy, and support
# documentation.
# This workflow will build a Java project with Gradle and cache/restore any dependencies to improve the workflow execution time
# For more information see: https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-java-with-gradle

name: Java CI with Gradle

# main branch에 push 또는 pull_request가 발생하면 실행
on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

# 파일을 실행하여 action을 수행하는 주체(Github Actions에서 사용하는 VM)가 읽을 수 있도록 허용
permissions:
  contents: read

jobs:
  build:
    # ubuntu 최신 버전에서 script를 실행
    runs-on: ubuntu-latest

    steps:
      # 지정한 저장소(현재 REPO)에서 코드를 워크플로우 환경으로 가져온다.
      - uses: actions/checkout@v3
      # open jdk 17 버전 환경을 세팅
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      # Github secrets로부터 데이터에 저장된 애플리케이션 설정 값들을 생성
      - name: make application.yml
        run: |
          cd ./src/main/resources
          touch ./application.yml
          echo "[ secrets.APPLICATION_PROD ]" > ./application.yml
          
      # Gradle을 통한 빌드 및 테스트
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Build with Gradle
        run: |
          chmod +x ./gradlew
          ./gradlew clean build -x test

      # Dockerfile을 통해 이미지를 빌드 후 Docker Repository로 Push
      - name: Docker build & push to docker repository
        run: |
          docker login -u [ secrets.DOCKER_USERNAME ] -p [ secrets.DOCKER_PASSWORD ]
          docker build -f Dockerfile -t [ secrets.DOCKER_REPO ]/jewelry:[ secrets.DOCKER_IMAGE_TAG ] .
          docker push [ secrets.DOCKER_REPO ]/jewelry:[ secrets.DOCKER_IMAGE_TAG ]
      
      # appleboy/ssh-action@master 액션을 사용하여 지정한 서버에 ssh로 접속하고, 배포 스크립트를 실행
      # 배포 스크립트 실행 : EC2의 실행중인 컨테이너를 중단, Docker Repoistory에 Push한 이미지를 가져와 컨테이너를 실행
      - name: Deploy
        uses: appleboy/ssh-action@master
        id: deploy
        with:
          username: ec2-user
          host: [ secrets.HOST ]
          key: [ secrets.PRIVATE_KEY ]
          port: [ secrets.REMOTE_SSH_PORT ]
          script: |
            export IMAGE_TAG=[ secrets.DOCKER_IMAGE_TAG ]
            export DOCKER_USERNAME=[ secrets.DOCKER_REPO ]
            chmod 777 /home/ec2-user/docker/deploy.sh
            /home/ec2-user/docker/deploy.sh $IMAGE_TAG $DOCKER_USERNAME
```
- **`[ secrets.OOO ]`** 표기를 **${\{ secrets.OOO }}** 로 변경해야 변수 사용이 가능합니다.
- **`secrets.OOO`**을 Repository > Settings > Secrets and variables > Actions에 **Repository secrets**에 필요 값을 추가하여 사용하시면됩니다.

### **4) 배포 스크립트 작성**

**docker-compose.yml**

```bash
version: '3'
services:

  jewelry:
    container_name: jewelry
    image: ${DOCKER_USERNAME}/jewelry:${IMAGE_TAG}
    ports:
      - 8080:8080
    environment:
      - SPRING_PROFILES_ACTIVE=[설정한 프로파일 명]

```

**deploy.sh**

```bash
#!/bin/bash

IMAGE_TAG=$1
DOCKER_USERNAME=$2

if [ -z "$IMAGE_TAG" ]; then
  echo "ERROR: Image tag argument is missing."
  exit 1
fi

echo "1. current jewelry container down"
docker-compose -f /home/ec2-user/docker/docker-compose.yml stop jewelry

echo "2. get jewelry image"
docker-compose -f /home/ec2-user/docker/docker-compose.yml pull jewelry

echo "3. jewelry container up"
docker-compose -f /home/ec2-user/docker/docker-compose.yml up -d jewelry

while [ 1 = 1 ]; do
  echo "3. jewelry health check..."
  sleep 3
  REQUEST=$(curl http://127.0.0.1:8080) # 8080포트 request

  if [ -n "$REQUEST" ]; then # 서비스 가능하면 health check 중지
    echo "health check success"
```

### **5) CI/CD 테스트**

커밋을 하게되면 아래와 같이 Workflow에 작성한 작업이 순서대로 진행되는 것을 볼 수 있습니다.
![Github Actions Workflow Result]({{site.url}}/assets/img/Docker-GithubActions-CICD/3_Workflow Result.png)

## **CI/CD 환경을 구축하면서**

굉장히 편리했습니다. CI/CD를 해보지 않았을 당시에는 프로젝트를 배포하는데 있어 빌드하고 문제 없으면 프로젝트 또는 수정한 파일만 올리고 서버를 재시작하고 이 과정을 계속해서 반복해야 했습니다.

회사를 다닐때는 배포 시간만 잘 지키면 문제가 없는 상황이여서 굳이 CI/CD를 구축한다는 것을 중요하게 생각하지 않습니다. 그 과정에서 빌드와 테스트를 더 신경써야했고 백업을 해두지 않으면 오류가 발생 시 다시 찾아 복구시키거나 수정하고 이 과정에 시간을 많이 들었습니다.

CI/CD 환경을 구축하면서 아쉬웠던 점은, Workflow 작업에 배포 스크립트 같이 올려서 진행해보려했지만, 방법을 찾지 못했습니다. 그래서 스크립트 파일을 서버에 생성하고 실행하는 방식으로 처리했는데, 수정해야하는 경우가 생기면 서버에 직접 접속해야하는 번거로움이 있었습니다.

AWS의 S3, AWS CodeDeploy를 통해 CI/CD 환경을 구축하는 방법도 있지만, Docker를 좀 더 사용해보고자 Docker, Github Action을 활용해서 CI/CD 환경을 구축해보았습니다. 추후에 S3, AWS CodeDeploy를 통한 CI/CD 환경도 구축하겠습니다.