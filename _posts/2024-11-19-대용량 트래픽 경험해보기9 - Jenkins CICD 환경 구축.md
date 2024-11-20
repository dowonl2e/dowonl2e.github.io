---
title: 대용량 트래픽 경험해보기(9/10) - Jenkins CI/CD 환경 구축
# author: dowonl2e
date: 2024-11-19 13:00:00 +0800
categories: [대용량 트래픽]
tags: [대용량 트래픽, Jenkins, CI/CD]
pin: true
---

## **Jenkins**

Jenkins란 오픈 소스 자동화 서버로, 주로 지속적 통합(Continuous Integration, CI) 및 지속적 배포(Continuous Delivery, CD)를 지원하는 도구이다. 자동화 프로세스를 설정하고, 빌드, 테스트, 배포 등을 자동으로 수행하여 개발 주기를 단축시키고 품질을 향상시킬 수 있다.

### **주요특징**

- 자동화된 빌드 및 테스트	: 코드 변경이 감지될 때마다 자동으로 빌드 및 테스트를 실행할 수 있다. 이를 통해 개발자는 코드를 자주 병합하고, 코드가 정상적으로 작동하는지 편리하게 확인할 수 있다.
- 플러그인 확장성	: 많은 플러그인을 제공하여 다양한 기능을 추가할 수 있다. 
  - 소스 코드 관리 시스템(SCM) 연동, 빌드 도구, 테스트 프레임워크, 배포 도구 등
- 다양한 플랫폼 지원 : Windows, Mac OS, Linux 등 다양한 운영 체제에서 실행할 수 있다. 또한, Docker 컨테이너에서도 실행할 수 있어 유연한 배포가 가능하다.
- 분산 빌드 : 여러 노드에서 빌드를 분산하여 수행할 수 있다. 이를 통해 빌드 시간을 단축하고, 대규모 프로젝트에서도 효율적으로 작업을 수행할 수 있다.
- 파이프라인 지원 : 선언적 파이프라인과 스크립트 파이프라인을 지원한다. 이를 통해 복잡한 빌드, 테스트, 배포 과정을 코드로 정의하고 관리할 수 있다.

## **DinD & DooD**

### **DinD(Docker in Docker)**

DinD는 Docker 내부에 Docker 데몬을 실행하는 것이다. 컨테이너 내에서 다른 컨테이너를 실행하거나 Docker 명령을 사용할 수 있다. 

![DinD]({{site.url}}/assets/img/High-Volume-Traffic-9/1-DinD.png)

#### **주요 특징**

- 컨테이너 내부에서 Docker 데몬(dockerd)을 실행함으로써, 컨테이너 안에서도 Docker 명령(**`docker build`**, **`docker run`** 등)을 수행할 수 있다.
- Docker를 실행하기 위해 독립적인 데몬을 실행하므로, 호스트의 Docker 환경과 격리된다. 서로 다른 컨테이너 간 충돌을 방지할 수 있지만, 리소스 사용 측면에서 주의가 필요하다.
- DinD를 구현하려면 보통 docker:dind 이미지나 Docker 데몬을 컨테이너 내부에서 실행하도록 권한을 설정해야 한다.

### **DooD(Docker Outside of Docker)**

DooD는 Docker 호스트 위에 자신과 동일한(sibling) 관계의 Docker container를 생성하는 수평적 계층 구조이다.

![DooD]({{site.url}}/assets/img/High-Volume-Traffic-9/2-DooD.png)

#### **주요 특징**

- 컨테이너에서 호스트의 Docker 데몬 사용한다.
- 호스트의 네트워크 및 파일 시스템을 사용한다.
- Docker 데몬을 추가로 실행하지 않기에 리소스 사용량이 낮다는 점에서 CPU 및 메모리 자원을 절약할 수 있다.

### **DinD, DooD 비교**

<table>
  <thead>
    <tr class="header">
      <th style="text-align:center">특징</th>
      <th style="text-align:center">DinD (Docker in Docker)</th>
      <th style="text-align:center">DooD (Docker Outside of Docker)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td markdown="span" style="text-align:center">Docker 데몬 실행</td>
      <td markdown="span" style="text-align:center">컨테이너 내에서 독립된 Docker 데몬 실행</td>
      <td markdown="span" style="text-align:center">호스트 Docker 데몬 사용</td>
    </tr>
    <tr>
      <td markdown="span" style="text-align:center">리소스 사용</td>
      <td markdown="span" style="text-align:center">비교적 비효율적 (데몬 중복 실행)</td>
      <td markdown="span" style="text-align:center">효율적 (데몬 중복 없음)</td>
    </tr>
    <tr>
      <td markdown="span" style="text-align:center">보안</td>
      <td markdown="span" style="text-align:center">보안 취약점 높음 (privilieged mode로 실행)</td>
      <td markdown="span" style="text-align:center">보안 취약점 높음 (Docker 소켓 노출)</td>
    </tr>
  </tbody>
</table>

### **DooD 사용 이유**

Jenkins을 Docker를 이용해 컨테이너 기반으로 환경을 구축해보려한다. API 서버의 경우에도 Docker Container 기반으로 환경이 구축되고 배포되는 상태이기에 DinD 혹은 DooD 방식 중 하나를 선택해야 했다.

DinD, DooD의 경우 둘다 보안적으로 취약하다는 단점이 있지만, 리소스 사용면에서 DooD가 이점이라는 것에 이번 Jenkins 환경 구축시에 이용해보려한다. 게다가, 몇몇 블로그를 찾아보고 DinD와 관련된 내용을 찾아보면서 DinD의 경우 기술적인 결함과 보안이슈로 인해서 사용하지 않는 것을 권장한다고 한다. 이에, Jenkins 환경을 DooD 방식으로 구축해보려한다.

## **Jenkins 환경 구축**

API 서버의 경우 테스트 전 최소 사양으로 설정해둔 상태이며 API 서버 내에 Jenkins를 구축하기에 애매한 부분이 있어 별도 서버로 운영하려한다.

### **Jenkins 최소 환경**

소규모 프로젝트를 기준으로 아래 기준의 서버 사양이 필요하며, 비용을 최소화하는 목적으로 최소 사양에 맞는 서버를 선택하였다.

<table>
  <thead>
    <tr class="header">
      <th style="text-align:center">구분</th>
      <th style="text-align:center">최소 사양</th>
      <th style="text-align:center">서버 사양(t3a.small)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td markdown="span" style="text-align:center">CPU</td>
      <td markdown="span" style="text-align:center">1Core</td>
      <td markdown="span" style="text-align:center">2(1Core)</td>
    </tr>
    <tr>
      <td markdown="span" style="text-align:center">Memory</td>
      <td markdown="span" style="text-align:center">1Gb</td>
      <td markdown="span" style="text-align:center">2Gb</td>
    </tr>
    <tr>
      <td markdown="span" style="text-align:center">Storage</td>
      <td markdown="span" style="text-align:center">Jenkins를 Docker 컨테이너로 실행하는 경우 최소 10GB가 권장</td>
      <td markdown="span" style="text-align:center">10Gb</td>
    </tr>
  </tbody>
</table>

### **Docker & docker-compose 설치**

- [Docker 설치](/posts/%EB%8C%80%EC%9A%A9%EB%9F%89-%ED%8A%B8%EB%9E%98%ED%94%BD-%EA%B2%BD%ED%97%98%ED%95%B4%EB%B3%B4%EA%B8%B04-%EC%B4%88%EA%B8%B0-%EC%9D%B8%ED%94%84%EB%9D%BC-%EA%B5%AC%EC%B6%95/#docker-설치){:target="\_blank"}

- [docker-compose 설치](/posts/%EB%8C%80%EC%9A%A9%EB%9F%89-%ED%8A%B8%EB%9E%98%ED%94%BD-%EA%B2%BD%ED%97%98%ED%95%B4%EB%B3%B4%EA%B8%B04-%EC%B4%88%EA%B8%B0-%EC%9D%B8%ED%94%84%EB%9D%BC-%EA%B5%AC%EC%B6%95/#docker-compose-설치){:target="\_blank"}

### **Docker로 Jenkins 환경 구축**

#### **Jenkins 이미지 구축 및 볼륨 생성**

```bash
# vi Dockerfile
FROM jenkins/jenkins:lts-jdk17
USER root

RUN apt-get update && \
    apt-get -y install apt-transport-https \
      ca-certificates \
      curl \
      gnupg2 \
      software-properties-common && \
    curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg > /tmp/dkey; apt-key add /tmp/dkey && \
    add-apt-repository \
      "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
      $(lsb_release -cs) \
      stable" && \
   apt-get update && \
   apt-get -y install docker-ce
   
RUN usermod -u {호스트 유저 아이디} jenkins && \
    groupmod -g {호스트 도커 그룹 아이디} docker && \
    usermod -aG docker jenkins

USER jenkins
```

- **`FROM jenkins/jenkins:lts-jdk17`** : Jenkins LTS(Long-Term Support) 버전 중 JDK 17인 런타임 환경 이미지 사용
- **`USER root`** : Docker 설치 시 root 사용자로 전환하여 설치
- **`RUN ...`** : Docker CLI 설치
  - **`apt-get -y install apt-transport-https ca-certificates curl gnupg2 software-properties-common`** : 필수 패키지 설치
    - **apt-transport-https** : HTTPS로 패키지를 다운로드
    - **ca-certificates** : SSL 인증서 처리
    - **curl** : Docker GPG 키 다운로드를 위함
    - **gnupg2** : GPG 키를 처리하는 도구
    - **software-properties-common** : 추가 소프트웨어 리포지토리를 관리
  - **`curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg > /tmp/dkey`** : Docker 패키지의 신뢰성을 확인하기 위한 GPG 키를 `/tmp/dkey` 파일에 다운로드
  - **`apt-key add /tmp/dkey`** : 다운로드한 GPG 키를 시스템의 APT 키 목록에 추가하여 Docker 패키지를 신뢰할 수 있도록 등록
  - **`add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"`** : Docker의 공식 APT 리포지토리를 시스템의 패키지 소스 목록에 추가
  - **`apt-get -y install docker-ce`** : Docker Community Edition 설치하여 Jenkins 내부에서 Docker 명령어를 실행할 수 있도록 함
- **`usermod -u {호스트 유저 아이디} jenkins`** : Jenkins 사용자의 UID를 호스트 시스템에 지정된 UID로 변경
  - Jenkins 컨테이너 내부의 파일이나 디렉토리가 호스트와 공유될 때 파일 권한 문제를 방지
- **`groupmod -g {호스트 도커 그룹 아이디} docker`** : docker 그룹의 GID를 호스트 시스템에 지정된 도커 GID로 변경
  - 컨테이너 내부에서 Docker 명령어를 실행할 때 파일 권한 충돌을 방지
- **`usermod -aG docker jenkins`** : jenkins 사용자를 docker 그룹에 추가
  - jenkins 사용자가 Docker 소켓에 접근할 수 있도록 설정하여 Docker 명령어 실행 가능

```bash
# Jenkins 이미지 구축(위 Dockerfile의 경로에서 실행)
$ docker build -t jenkins-dood:1.0 .
```

```bash
# Jenkins 볼륨 생성
$ sudo docker volume create jenkins-volume

# 볼륨 정보에서 마운트 경로 확인
$ docker volume inspect {볼륨명}

# 마운트 경로 소유자 확인
$ sudo ls -al {볼륨 마운트 경로}

# Dockerfile에서 호스트 UID로 설정한 jenkins의 UID를 마운트된 volume에 접근이 가능하도록 변경
$ sudo chown -R {jenkins 호스트 UID}:{jenkins 호스트 그룹 UID} {볼륨 마운트 경로}
```

#### **명령어를 통한 Jenkins Container 실행**

```bash
$ docker run -d \
-p 8080:8080 \
-p 50000:50000 \
-v jenkins-volume:/var/jenkins_home \
-v /var/run/docker.sock:/var/run/docker.sock:ro \
-v /var/lib/docker/containers:/var/lib/docker/containers:ro \
--name jenkins \
--network jenkins-network \
--user jenkins \
jenkins-dood:1.0
```

- **`-p 8080`** : 기본 웹 UI 접속 포트
- **`-p 50000`** : Jenkins는 Master와 Agent(Worker)와 같이 분산 빌드 환경을 구성할 수 있으며 Master와 Agent(Worker)의 빌드 작업 통신 용도
- **`-v jenkins-volume:/var/jenkins_home`** : Jenkins 볼륨 마운트
- **`-v /var/run/docker.sock:/var/run/docker.sock:ro`** : Docker Container에서 호스트의 Docker 데몬을 읽기 전용으로 연결되도록 볼륨을 마운트
- **`-v /var/lib/docker/containers:/var/lib/docker/containers:ro`** : Docker 컨테이너 내부에서 호스트 시스템의 Docker 컨테이너 데이터 디렉토리를 읽기 전용으로 볼륨을 마운트

#### **docker-compose를 이용한 Jenkins Container 실행**

```yaml
# docker-compose.yml
services:
  jenkins:
    container_name: jenkins
    image: jenkins-dood:1.0
    environment:
      - TZ=Asia/Seoul
    ports:
      - 8080:8080
      - 50000:50000
    volumes:
      - jenkins-volume:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    networks:
      - jenkins-network
    user: jenkins

networks:
  jenkins-network:
    external: true
volumes:
  jenkins-volume:
    external: true
```

Container를 실행하게되면 초기 비밀번호를 입력하는 화면을 확인할 수 있다.

```bash
$ docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

위 명령어를 통해 초기 비밀번호를 확인하면된다. 

첫 시작은 플러그인 설치 과정이며 **`Install suggested plugins`**을 통해 기본 플러그인을 설치한다.

플러그인 설치 후 신규 계정을 추가화면에서 정보를 입력하면 젠킨스 대시보드 화면을 볼 수 있다.

## **Jenkins 기본 설정**

### **Jenkins Location 설정**

**`Dashboard > Jenkins 관리 > System`**에서 Jenkins URL을 입력해준다.

![Jenkins URL]({{site.url}}/assets/img/High-Volume-Traffic-9/3-Jenkins_URL.png)


### **Credentials 추가**

**`Dashboard > Jenkins 관리 > Credentials > System > Global credentials (unrestricted)`**에서 CI/CD 과정에서 필요한 주요 정보(Github, Docker, property 파일 등)를 등록한다.

![Jenkins Credentials]({{site.url}}/assets/img/High-Volume-Traffic-9/4-Jenkins_Credentials.png)

### **Pipeline 생성**

**`Dashboard > 새로운 Item > Pipeline`**을 생성하면 아래와 같이 파이프라인 스크립트를 작성할 수 있으며 추가로 Github Branch, WebHook 등 다양한 설정을 할 수 있다.

![Jenkins Pipeline]({{site.url}}/assets/img/High-Volume-Traffic-9/5-Jenkins_Pipeline.png)

#### **Pipeline Script 자동생성**

파이프라인을 생성했다면 **`Dashboard > {생성한 Pipeline} > Pipeline Syntax`**에서 아래와 같이 필요한 스텝을 선택하고 스탭에 맞는 필요 정보를 입력하면 파이프라인 스크립트를 쉽게 생성할 수 있다. 

다만, Credentials에 등록한 정보를 스크립트에 따로 호출해서 사용해야하는 경우 스크립트 수정이 필요하다.

![Jenkins Pipeline Syntax]({{site.url}}/assets/img/High-Volume-Traffic-9/6-Jenkins_Pipeline_Syntax.png)

#### **Docker Plugin 추가**

파이프라인 스크립트를 생성할 때 도커 관련 스크립트를 작성해야할 필요가 있으면 **`Dashboard > Jenkins 관리 > Plugins`**에서 Docker를 검색하여 **Docker API Plugin**, **Docker Common Plugin**을 받으면되며, 필요에 따라 플러그인 찾아 설치하면된다.

![Jenkins Add Docker Plugin]({{site.url}}/assets/img/High-Volume-Traffic-9/7-Jenkins_Add_Docker_Plugins.png)

## **CI/CD**

### **Github Webhook 설정**

**`Webhook을 설정할 Github Repository > Settings > Webhooks`**에 Webhook 설정을 해준다.

![Github Webhook]({{site.url}}/assets/img/High-Volume-Traffic-9/8-Github_Webhook.png)

- **Payload URL** : **http(s)://{Jenkins 호스트 DNS or IP}:{Port}/github-webhook/** 입력
- **Content type** : application/json 설정
- **SSL verification** : SSL 설정이 있는 경우 선택
- **Which events would you like to trigger this webhook?** : Just the push event.로 푸시가 발생했을 때로 설정

Jenkins로 돌아와 생성한 파이프라인에서 구성으로 들어가면 아래와 같이 **GitHub project**, **GitHub hook trigger for GITScm polling**를 설정해준다.

![Jenkins Github Webhook Settings]({{site.url}}/assets/img/High-Volume-Traffic-9/9-Jenkins_Github_Webhook_Setting.png)

### **Github Repository Clone Script**

```tsx
pipeline {
    agent any

    environment {
        github_credential_id = '{Credentials에 등록된 Github ID}'
        git_branch = credentials('{Credentials에 등록된 Github Branch ID}')
        git_url = credentials('{Credentials에 등록된 Github URL ID}')
    }
        
    stages {
        // git에서 repository clone
        stage('Github Repository Clone') {
            steps {
                echo 'Clonning Repository'
                git credentialsId: github_credential_id, 
                    branch: git_branch, 
                    url: git_url
            }
            post {
                success { 
                    echo '======================================================'
                    echo '============Successfully Cloned Repository============'
                    echo '======================================================'
                }
                failure {
                    error 'This pipeline stops here... -> Prepare Error'
                }
            }
        }
    }
}
```

### **Spring Boot Application 설정 다운로드 Script**

```tsx
pipeline {
    agent any

    environment {
        application_property_file_id = '{Credentials에 등록된 Application File ID}`
        spring_profiles_active = 'prod'
    }
        
    stages {
        stage('Downloading Application Config File') {
            steps {
              withCredentials([file(credentialsId: application_property_file_id, variable: 'applicationConfigFile')]) {
                  script {
                      // application 설정 파일 프로젝트 resource 경로로 이동하기 위해 권한을 수정한다.
                      sh 'chmod 755 /var/jenkins_home/workspace/{파이프라인 명}/src/main/resources'
                      sh 'cp $applicationConfigFile /var/jenkins_home/workspace/{파이프라인 명}/src/main/resources/application-${spring_profiles_active}.yml'
                  }
              }
          }
          post {
                success { 
                    echo '======================================================='
                    echo '=====Successfully Download Application Config File====='
                    echo '======================================================='
                }
                failure {
                  error 'This pipeline stops here... -> Download Application Config File Error'
                }
            }
        }
    }
}
```

### **Project Gradle Build Script**

```tsx
pipeline {
    agent any

    environment {
        pipeline_root_dir = '/var/jenkins_home/workspace/{파이프라인 명}'
    }
        
    stages {
        // gradle build
        stage('Bulid Gradle') {
            steps {
                echo 'Bulid Gradle'
                // 파이프라인 루트 경로로 이동 후 gradle build 실행
                dir (pipeline_root_dir){
                    sh '''
                        echo 'Start Gradle Clean&Build'
                        chmod +x gradlew
                        ./gradlew clean build -x test
                    '''
                }
            }
            post {
                success { 
                    echo '======================================================'
                    echo '===============Successfully Gradle Build=============='
                    echo '======================================================'
                }
                failure {
                error 'This pipeline stops here... -> Build Gradle Error'
                }
            }
        }
    }
}
```

### **Docker Build Script**

```tsx
pipeline {
    agent any

    environment {
        pipeline_root_dir = '/var/jenkins_home/workspace/{파이프라인 명}'
        imagename = '{Docker 계정}/{Docker Repository 명}'
    }
        
    stages {
        // docker build
        stage('Bulid Docker') {
            steps {
                echo 'Bulid Docker'
                script {
                    dockerImage = docker.build("${imagename}", "${pipeline_root_dir}")
                }
            }
            post {
                success { 
                    echo '====================================================='
                    echo '==============Successfully Docker Build=============='
                    echo '====================================================='
                }
                failure {
                    error 'This pipeline stops here... -> Build Docker Error'
                }
            }
        }
    }
}
```

### **Docker Repository Push Script**

```tsx
pipeline {
    agent any

    environment {
        docker_credential_id = '{Credentials에 등록된 Docker Credential ID}'
        docker_repository_tag = 'latest'
    }
        
    stages {
        // docker push
        stage('Push Docker') {
            steps {
                echo 'Push Docker'
                script {
                    docker.withRegistry( '', docker_credential_id) {
                        dockerImage.push(docker_repository_tag)
                    }
                }
            }
            post {
                success { 
                    echo '======================================================'
                    echo '================Successfully Docker Push=============='
                    echo '======================================================'
                }
                failure {
                    error 'This pipeline stops here... -> Docker Push Error'
                }
            }
        }
    }
}
```

### **Deploy Script 파일 다운로드 Script**

#### **deploy.sh 실행 순서**

> **Docker Image Pull → Container Stop → Container Start**

```tsx
pipeline {
    agent any

    environment {
        deployment_file_id = '{Credentials에 등록된 배포 스크립트 파일 ID}'
        jenkins_pipeline_path = '/var/jenkins_home/workspace/{파이프라인 명}/'
        jenkins_deploy_path = 'jenkins/deploy'
    }
        
    stages {
        stage('Downloading Deployment File') {
            steps {
              withCredentials([file(credentialsId: deployment_file_id, variable: 'deploymentFile')]) {
                  script {
                      sh 'mkdir -p ${jenkins_pipeline_path}${jenkins_deploy_path}'
                      sh 'chmod 755 ${jenkins_pipeline_path}${jenkins_deploy_path}'
                      sh 'cp $deploymentFile ${jenkins_pipeline_path}${jenkins_deploy_path}/deploy.sh'
                      sh 'chmod 777 ${jenkins_pipeline_path}${jenkins_deploy_path}/deploy.sh'
                  }
              }
          }
          post {
                success { 
                    echo '======================================================='
                    echo '=====Successfully Download Deployment File====='
                    echo '======================================================='
                }
                failure {
                  error 'This pipeline stops here... -> Download Application Config File Error'
                }
            }
        }
    }
}
```

### **API 서버 원격 접속, Deploy Script 파일 업로드, Deploy**

#### **API 서버 SSH 접속 설정**

![Jenkins SSH Config]({{site.url}}/assets/img/High-Volume-Traffic-9/10-Jenkins_SSH_Config_1.png)

- **Name** : 접속 서버 명으로 아무거나 입력
- **Hostname** : API 서버 호스트 DNS or IP 입력
- **Username** : SSH 접속 계정 입력
- **Remote Directory** : SSH 접속 후 이동할 디렉토리 경로

설정 후 **Test Configuration**으로 정상적으로 접속되는지 확인한다.

#### **Deploy Script**

```tsx
pipeline {
    agent any

    environment {
        docker_username = '{Docker 계정}'
        docker_imagename = 'Docker Repository 명'
        docker_tag = 'latest'
        jenkins_pipeline_path = '/var/jenkins_home/workspace/{파이프라인 명}/'
        jenkins_deploy_path = '{파이프라인 경로에 저장할 디렉토리 경로}'
    }
        
    stages {
        // Trasfer File & Deployment
        stage('Trasfer Deployment File & Deploy') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'api-server', 
                            transfers: [
                                sshTransfer(
                                    cleanRemote: false, 
                                    excludes: '', 
                                    execCommand: """
                                        sudo chmod 777 {업로드 할 배포 스크립트 디렉토리}/deploy.sh
                                        {업로드 할 배포 스크립트 디렉토리}/deploy.sh ${docker_username} ${docker_imagename} ${docker_tag}
                                    """, 
                                    execTimeout: 120000, 
                                    flatten: false, 
                                    makeEmptyDirs: false, 
                                    noDefaultExcludes: false, 
                                    patternSeparator: '[, ]+', 
                                    remoteDirectory: './{원격 서버 배포 스크립트 디렉토리 경로}', // SSH 설정에서 Remote Directory 기준으로 원격 디렉토리 경로 설정
                                    remoteDirectorySDF: false, 
                                    removePrefix: '${jenkins_deploy_path}', 
                                    sourceFiles: '${jenkins_deploy_path}/deploy.sh'
                                )
                            ], 
                            usePromotionTimestamp: false, 
                            useWorkspaceInPromotion: false, 
                            verbose: true
                        )
                    ]
                )
            }
            post {
                success { 
                    echo '========================================================'
                    echo '=====Successfully Transfer Deployment File & Deploy====='
                    echo '========================================================'
                }
                failure {
                  error 'This pipeline stops here... -> Trasfer Deployment File & Deploy Error'
                }
            }
        }
    }
}
```

## **효율적인 배포를 위한 파이프라인 나누기**

실제 운영에서 CI/CD는 위와 같이 진행될 것이다. 만약, 소스 변경이 아닌 API 서버가 어떠한 문제로 다운이 된 상태 혹은 재시작만 해야하는 상황일 때 위 과정이 모두 필요할까? 라는 생각이 들었다.

이에, 아래와 같이 총 3개의 파이프라인으로 나누었고 전체 CI/CD 과정 또한 할 수 있도록 하나의 Pipeline에서 두 파이프라인을 실행하도록 변경했다.

1. Github WebHook ~ Docker Repository Push 파이프라인
2. API 서버 원격 접속 ~ 배포 스크립트 파일 업로드 및 배포 파이프라인
3. 1, 2번을 순서대로 실행하는 파이프라인

### **N개 파이프라인 실행 스크립트**

```tsx
pipeline {
    agent any
    stages {
        stage('1. Run Build Docker Push Pipeline') {
            steps {
                build job: '{파이프라인 명}' // 첫 번째 파이프라인 실행
            }
        }
        stage('2. Run Deploy Pipeline') {
            steps {
                build job: '{파이프라인 명}' // 두 번째 파이프라인 실행
            }
        }
    }
}
```

다른 파이프라인을 실행해야 할 때 **`stage`**를 추가해 실행할 파이프라인을 설정해주면 된다.

> 위 과정에서 변경해야할 설정으로 1번 파이프라인에서 설정한 Github Webhook 설정을 해제 후 3번 파이프라인에 설정

## **References**

> [https://velog.io/@showui96/Docker-DinD-vs-DooD](https://velog.io/@showui96/Docker-DinD-vs-DooD){:target="\_blank"}
>
> [https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/){:target="\_blank"}