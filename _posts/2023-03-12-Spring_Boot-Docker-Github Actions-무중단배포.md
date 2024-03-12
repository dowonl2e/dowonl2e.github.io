---
title: Spring Boot + Docker + Github Actions + Nginx를 이용한 무중단 배포
# author: dowonl2e
date: 2024-03-12 10:00:00 +0800
categories: [DevOps, CI/CD]
tags: [Spring Boot, AWS, Docker, Github Actions, CI/CD, 무중단 배포]
pin: true
---

이전 글에서 Github Actions CI/CD 환경을 구축하고 배포를 해보았습니다. 이번 글에서는 이전 글에서 구축한 CI/CD 파이프라인을 이용해 Nginx를 이용한 무중단 배포를 적용한 경험을 적어봅니다.

> [Spring Boot + Docker + Github Actions를 활용한 CI/CD 환경 구축](/posts/Docker-Github-Action-CICD-%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%B6%95/){:target="\_blank"}

먼저, 무중단 배포란 무엇이고 어떤 종류가 있는지 보겠습니다.

## **무중단 배포란?**

무중단 배포란 **서비스를 중단하지 않고 새로운 버전을 배포**한다는 의미입니다.

실제 서비스를 서버 1대만으로 운영한다고 보면 새로운 버전을 배포하기위해서는 서버를 중단해야하는 상황이 발생한다. 특히, Java 언어의 경우 컴파일 과정이 반드시 필요하기에 서버 재시작은 필수이며, 이 과정에서 서버를 사용할 수 없는 시간이 생기는데 이를 **다운타임**{: .text-blue }이라고 합니다.

## **무중단 배포 전략**

### **블루그린 배포 (Blue/Green Deployment)**

가장 대표적인 방식 중 하나이며, 기존 서비스가 동작하는 **블루**{: .text-blue } 그리고 새로운 업데이트 버전이 동작하는 **그린**{: .text-green } 2개의 환경이 필요하다.

블루그린 배포를 적용하면, 새로운 업데이트 버전이 그린 환경에서 먼저 동작하게 되며, 이후에 블루 환경에서 동작하던 기존 버전이 순차적으로 그린 환경으로 이전된다. 이러한 방식으로 사용자는 언제나 정상적인 서비스 이용이 가능하다.

### **카나리아 배포(Canary Deployment)**

블루그린 배포와 비슷한 방식으로, 새로운 업데이트 버전을 먼저 일부 사용자에게 노출시키는 방식이다. 이를 통해, 새로운 업데이트 버전에 대한 사용자들의 반응을 먼저 파악하고, 문제가 발생할 경우 빠르게 대처할 수 있다.

### **폴링 배포(Polling Deployment)**

기존 서비스와 새로운 업데이트 버전을 번갈아가며 동작시키는 방식으로, 이를 위해서는 두 개의 버전이 동시에 운영되어야 하며, 사용자의 요청이 들어오면 이를 번갈라서 처리하게 된다.

## **Nginx를 이용한 블루그린 배포 (Blue/Green Deployment)**

이번에 작성할 무중단 배포 전략은 블루그린 배포 전력이며, 기존에 구축한 CI/CD 환경을 기반으로 Nginx를 추가로 활용해보았습니다.

블루그린 배포의 경우 블루환경에서 동작하는 1개 서버, 그린 환경에서 동작하는 또 다른 서버 이렇게 2개의 인프라가 필요하지만, Nginx를 이용하면 굳이 2개의 인프라가 필요하지 않아 저렴하게 무중단 배포가 가능합니다.

> **프로젝트, AWS, Nginx 환경은 구축 되어있다는 가정으로 진행하겠습니다.**

## **Nginx를 활용한 무중단 배포**

### **배포 프로세스**

Nginx를 활용한 무중단 배포 프로세스는 아래와 같습니다.
![무중단 배포 프로세스]({{site.url}}/assets/img/None-Stop-Deployment/1_Process.png)

1. 배포할 코드를 Github에 Push 합니다.
2. Github Actions에서 Build & Test 작업을 수행합니다.
3. Build & Test이 정상적으로 완료되면 Docker에 배포할 프로젝트 이미지 Build 후 Push 하게됩니다.
4. Docker Repository Push가 완료되면 EC2 연결 후 배치 스크립트를 진행해 배포합니다. 배포 스크립트 과정은 아래와 같으며, Workflow에서 배포 과정을 확인하실 수 있습니다.
    1. docker-compose에 설정된 green 서비스를 통해 도커의 신규 버전인 이미지를 받은 후 green 컨테이너를 실행해줍니다.
        ![Docker Image Pull]({{site.url}}/assets/img/None-Stop-Deployment/2_Docker Image Pull.png)

    2. green 컨테이너가 정상 상태가 될때 까지 Healthy Check를 진행합니다.
        ![Grean Container Start And Health Check]({{site.url}}/assets/img/None-Stop-Deployment/3_Green Start Health Check.png)

    3. green 컨테이너가 정상 상태가 되면 Nginx의 프록시를 green 컨테이너로 변경 후 리로드 후 blue 컨테이너를 중지해줍니다.
        ![Blue Container Stop]({{site.url}}/assets/img/None-Stop-Deployment/4_Blue Stop.png)

> blue의 포트는 8081, green의 포트는 8082로 설정해주었습니다. 다른 서비스와 충돌이 없는 포트를 사용하시면됩니다.
{: .prompt-info }

### **Github Actions Workflow**

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
      # 지정한 저장소(현재 REPO)에서 코드를 워크플로우 환경으로 가져오도록 하는 github action
      - uses: actions/checkout@v3
      # open jdk 17 버전 환경을 세팅
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      # Github secrets로부터 데이터에 저장된 애플리케이션 설정 값들을 워크 플로우를 통해서 생성
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
      # 배포 스크립트 실행 : EC2의 실행중인 컨테이너를 중단, Docker Repoistory에 Push한 내용을 받아 컨테이너를 실행
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

### **docker-compose 설정**

```yaml
version: '3'
services:

  green:
    container_name: jewelry-green
    image: ${DOCKER_USERNAME}/jewelry:${IMAGE_TAG}
    ports:
      - 8082:8082
    environment:
      - SPRING_PROFILES_ACTIVE=green

  blue:
    container_name: jewelry-blue
    image: ${DOCKER_USERNAME}/jewelry:${IMAGE_TAG}
    ports:
      - 8081:8081
    environment:
      - SPRING_PROFILES_ACTIVE=blue
```

- ${DOCKER_USRNAME}, ${IMAGE_TAG} 의 경우 Workflow 마지막에 작성한 배포스크립트 실행시 넘긴 파라미터입니다.
- 프로젝트 애플리케이션 설정 내용의 프로파일은 blue, green 2개가 있어야합니다.

### **Nginx 설정**

```
  
  ...

  server {
      listen       80;
      listen       [::]:80;
      server_name  _;
      root         /usr/share/nginx/html;

      # Load configuration files for the default server block.
      include /etc/nginx/default.d/*.conf;

      include /etc/nginx/conf.d/service-url.inc;

      location / {
              proxy_pass $service_url;
              proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
              proxy_set_header Host $host;
      }

      error_page 404 /404.html;
      location = /404.html {
      }

      error_page 500 502 503 504 /50x.html;
      location = /50x.html {
      }
  }

  ...
  
```

- Nginx 설치 후 nginx.conf 파일 내용에서 server 부분을 위와 같이 설정했습니다. (사용하는 도메인이 없어 SSL(443)없이 진행했습니다.)
- 무중단 배포 과정에서 위 코드에 **`include /etc/nginx/conf.d/service-url.inc;`** 파일 안에 값이 변경되며, 
**`$service_url`**의 값이 **location**의 **proxy_pass**에 적용됩니다.

**proxy 설정에 필요한 URL 설정**

Nginx를 설치 후 /etc/nginx 경로에 conf.d 파일을 생성해주고 아래 3개 파일을 생성해 줍니다.

- service-url-blue.inc : proxy_pass로 설정할 Blue 서비스 URL 지정
    
    ```
    set $service_url http://127.0.0.1:8081;
    ```
    
- service-url-green.inc : proxy_pass로 설정할 Green 서비스 URL 지정
    
    ```
    set $service_url http://127.0.0.1:8082;
    ```
    
- service-url.inc : 최초는 빈 파일 또는 **service-url-blue.inc**의 내용이 설정되어있으면 됩니다. 아래의 배포 스크립트 실행시 **service-url-green** 또는 **service-url-blue**의 내용이 덮어씌어지게 됩니다.

> http://localhost:{포트}로 설정할 경우 Nginx에서 오류가 발생합니다. 로그를 확인해보면 “no resolver defined to resolve localhost”를 확인할 수 있습니다. URL의 IP를 알 수 없다는 의미로 localhost 대신 127.0.0.1 또는 IP를 찾을 수 있는 도메인 DNS를 입력해야 합니다.
{: .prompt-warning }

### **배포 스크립트**

```bash
#!/bin/bash

IS_GREEN=$(docker ps | grep jewelry-green)
IMAGE_TAG=$1
DOCKER_USERNAME=$2

if [ -z "$IMAGE_TAG" ]; then
  echo "ERROR: Image tag argument is missing."
  exit 1
fi

if [ -z "$IS_GREEN"  ]; then

  echo "################################"
  echo "### BLUE To GREEN Deployment ###"
  echo "################################"

  echo "1. Green Image Pull"
  docker-compose -f /home/ec2-user/docker/docker-compose.yml pull green

  echo "2. Green Container Run"
  docker-compose -f /home/ec2-user/docker/docker-compose.yml up -d green

  while [ 1 = 1 ]; do
  echo "3. ### Green Health Check ###"
  sleep 3

  REQUEST=$(curl http://127.0.0.1:8082)
    if [ -n "$REQUEST" ]; then
            echo "Green Health Check Success"
            break ;
            fi
  done;

  echo "4. Nginx Reload (Change Proxy Blue To Green)"
  sudo cp /etc/nginx/conf.d/service-url-green.inc /etc/nginx/conf.d/service-url.inc
  sudo nginx -s reload

  echo "5. Blue Container Stop"
  docker-compose -f /home/ec2-user/docker/docker-compose.yml stop blue
else
  echo "################################"
  echo "### Green To Blue Deployment ###"
  echo "################################"

  echo "1. Blue Image Pull"
  docker-compose -f /home/ec2-user/docker/docker-compose.yml pull blue

  echo "2. Blue Container Start"
  docker-compose -f /home/ec2-user/docker/docker-compose.yml up -d blue

  while [ 1 = 1 ]; do
    echo "3. ### Blue Health Check ###"
    sleep 3
    REQUEST=$(curl http://127.0.0.1:8081)

    if [ -n "$REQUEST" ]; then
      echo "Blue Health Check Success"
      break ;
    fi
  done;

  echo "4. Nginx Reload (Change Proxy Green To Blue)"
  sudo cp /etc/nginx/conf.d/service-url-blue.inc /etc/nginx/conf.d/service-url.inc
  sudo nginx -s reload

  echo "5. Green Container Stop"
  docker-compose -f /home/ec2-user/docker/docker-compose.yml stop green
fi

```

- Nginx 리로드 전에 **`sudo cp /etc/nginx/conf.d/service-url-green.inc /etc/nginx/conf.d/service-url.inc`** 이 명령어를 통해 Nginx의 프록시 설정을 변경하게 됩니다.
