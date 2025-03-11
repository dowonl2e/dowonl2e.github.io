---
title: KinD(Kubernetes in Docker)를 이용한 K8S 클러스터 구축
# author: dowonl2e
date: 2025-03-11 08:00:00 +0800
categories: [Kubernetes]
tags: [Kubernetes, MSA, Docker]
pin: true
---

K8S를 실제 운영서버에 설정해보기 전에 K8S가 어떤 것인지 알아보며 로컬 환경에서 K8S를 구축해보려 한다. 로컬 환경에서 구축하는 방법 중 **Minikube**, **KinD(Kubernetes in Docker)**, **Docker Desktop** 등을 이용해 로컬 환경에서 Kubernetes를 실행하는 방법이 있으며, 이 중 `KinD`{: .text-blue}를 이용해 K8S 클러스터를 구축해보려한다.

## **K8S(Kubernetes)란?**

K8S는 컨테이너화된 애플리케이션을 자동으로 배포, 확장 및 관리하는 오픈소스 컨테이너 오케스트레이션 시스템이다.

### **핵심 기능 및 특징**

- **컨테이너 오케스트레이션:**
  - 여러 호스트에 걸쳐 컨테이너를 자동으로 배포하고 관리합니다.
  - 컨테이너의 생명주기를 관리하고, 필요한 경우 자동으로 재시작하거나 복제합니다.
- **자동 확장 및 복구:**
  - 트래픽 변화에 따라 컨테이너 수를 자동으로 확장하거나 축소합니다.
  - 컨테이너나 호스트에 장애가 발생하면 자동으로 복구하여 애플리케이션의 가용성을 높입니다.
- **서비스 디스커버리 및 로드 밸런싱:**
  - 컨테이너 간의 통신을 위한 네트워크를 자동으로 구성합니다.
  - 트래픽을 여러 컨테이너에 분산시켜 애플리케이션의 성능을 향상시킵니다.
- **스토리지 오케스트레이션:**
  - 컨테이너에 필요한 스토리지를 자동으로 프로비저닝하고 관리합니다.
  - 다양한 스토리지 솔루션을 지원하여 유연한 스토리지 관리를 가능하게 합니다.
- **자동화된 롤아웃 및 롤백:**
  - 애플리케이션 업데이트를 안전하게 배포하고, 문제가 발생하면 이전 버전으로 롤백합니다.
  - CI/CD 파이프라인과 통합하여 애플리케이션 배포를 자동화할 수 있습니다.
- **구성 관리:**
  - 애플리케이션의 설정을 중앙에서 관리하고, 변경 사항을 자동으로 배포합니다.
  - ConfigMap과 Secret을 사용하여 중요한 설정 정보와 비밀 정보를 안전하게 관리합니다.

### **주요 구성 요소**

- **컨트롤 플레인(Control Plane)**
  - 클러스터의 전반적인 상태를 관리하고, 워크로드를 스케줄링합니다.
  - API 서버, 스케줄러, 컨트롤러 관리자, etcd 등의 구성 요소로 이루어져 있습니다.
- **노드(Node)**
  - 쿠버네티스 클러스터에서 워크로드를 실행하는 물리적 또는 가상 머신입니다.
  - 마스터 노드와 워커 노드를 모두 포함하는 개념입니다.
  - kubelet, kube-proxy, 컨테이너 런타임 등의 구성 요소로 이루어져 있습니다.
- **파드(Pod)**
  - 하나 이상의 컨테이너를 묶어서 관리하는 Kubernetes의 기본 배포 단위입니다.
- **서비스(Service)**
  - Pod 집합에 대한 단일 접근점을 제공하여 네트워크를 통해 접근할 수 있도록 합니다.
- **배포(Deployment)**
  - Pod의 복제본을 관리하고, 업데이트를 자동화합니다.

### **세부 구성 요소**

- **마스터 노드(Master Node)**
  - Control Plane 구성 요소들이 실행되는 물리적 또는 가상 서버입니다.
  - 하나 이상의 마스터 노드를 사용하여 Control Plane의 가용성을 높일 수 있습니다.
  - 마스터 노드는 Control Plane 구성 요소들 외에도 다른 시스템 구성 요소들을 포함할 수 있습니다.
- **워커 노드(Worker Node)**
  - 실제로 애플리케이션 컨테이너(Pod)가 실행되는 노드입니다.
  - 마스터 노드로부터 작업을 할당받아 수행하고, 클러스터의 마스터 노드에게 작업 결과를 보고하는 역할을 합니다.

### **장점**

- **애플리케이션 개발 및 배포 속도 향상:** 컨테이너 기반으로 개발, 배포 과정이 간편하고 빨라집니다.
- **리소스 효율성 향상:** 필요에 따라 자동으로 리소스를 할당하고 회수하여 리소스 활용도를 높입니다.
- **애플리케이션 가용성 향상:** 자동 복구 및 확장을 통해 애플리케이션의 안정성을 높입니다.
- **클라우드 환경에 대한 높은 이식성:** 다양한 클라우드 환경에서 동일한 방식으로 애플리케이션을 실행할 수 있습니다.

### **단점**

- 다양한 개념과 구성으로 러닝 커브가 가파르다.
- K8S 클러스터 운영에는 마스터 및 워커 노드에 대한 추가적인 리소스가 필요하여 추가적인 비용이 발생한다.
- 소규모 애플리케이션의 경우 리소스 오버헤드가 부담이 된다.

## **K8S CLI 도구**

### **Kubelet**

K8S에서 Kubelet은 워커 노드(Worker Node)에서 실행되는 에이전트이며 핵심 구성 요소이다. 워커 노드에서 실행되는 컨테이너(Pod)를 실행하고 관리하는 역할을 수행하며, kubelet이 없으면 워커 노드는 Pod를 실행할 수 없으며, 쿠버네티스 클러스터는 정상적으로 작동할 수 없다. 

kubelet은 워커 노드의 상태를 마스터 노드에게 보고하여 클러스터의 안정성을 유지하는 데 중요한 역할을 한다.

- **Pod 관리:**
  - kubelet은 마스터 노드(Control Plane)의 API 서버로부터 Pod 생성, 실행, 삭제 요청을 받아 해당 워커 노드에서 Pod를 관리합니다.
  - Pod의 상태를 지속적으로 모니터링하고, 문제가 발생하면 자동으로 재시작하거나 복구합니다.
- **컨테이너 런타임과의 상호 작용:**
  - 컨테이너 런타임(예: Docker, containerd)과 통신하여 컨테이너를 생성하고 관리합니다.
  - 컨테이너의 이미지 다운로드, 실행, 중지 등의 작업을 수행합니다.
- **노드 상태 보고:**
  - kubelet은 워커 노드의 상태(CPU, 메모리 사용량 등)를 마스터 노드에게 주기적으로 보고합니다.
  - 이를 통해 마스터 노드는 클러스터의 전체적인 상태를 파악하고 스케줄링 결정을 내릴 수 있습니다.
- **볼륨 관리:**
  - Pod에 필요한 스토리지를 연결하고 관리합니다.
  - Persistent Volume을 사용하여 데이터의 영속성을 보장합니다.

### **Kubectl**

kubectl은 K8S 클러스터를 관리하는 CLI(Command line Interface) 도구이다. Kubernetes API 서버와 통신하여 `클러스터` `노드` `파드(Pod)` `서비스(Service)` 등을 상태 확인, 배포, 삭제, 로그 확인 등을 확인할 수 있다.

#### **기본 명령어**

| 명령어 | 설명 |
| --- | --- |
| kubectl get nodes | 클러스터 내 노드 목록 |
| kubectl get pods -A | 모든 네임스페이스의  파드 목록 |
| kubectl get services | 서비스 목록 |
| kubectl describe pod <POD_NAME> | 특정 파드 상세 정보 확인 |
| kubectl logs <POD_NAME> | 특정 파드의 로그 확인 |
| kubectl exec -it <POD_NAME> -- /bin/sh | 파드 내부로 접속 |
| kubectl apply -f <파일명>.yaml | YAML 파일을 이용한 리소스 생성 |
| kubectl delete pod <POD_NAME> | 특정 파드 삭제 |
| kubectl delete -f <파일명>.yaml | YAML 파일을 이용한 리소스 삭제 |

### **Kubeadm**

kubeadm은 K8S 클러스터 실행과 배포를 도와주는 CLI(Command line Interface) 도구이다. Kubernetes 공식에서 제공하는 방법 중 하나로, **마스터 노드 및 워커 노드를 빠르게 설정**할 수 있습니다.

- **마스터 노드 초기화 (**`kubeadm init`**)**
  - API 서버, 컨트롤러 매니저, 스케줄러 및 etcd 설정
  - TLS 인증서 및 kubeconfig 생성
- **워커 노드 추가 (**`kubeadm join`**)**
  - 마스터 노드의 `kubeadm token`을 사용하여 클러스터에 합류
- **클러스터 설정 검사 및 업그레이드 (**`kubeadm upgrade`**)**

> Kubernetes의 클러스터 관리를 자동화하지만, 네트워크 플러그인(CNI) 설치는 별도로 필요.

## **컨테이너 런타임(Container Runtime)**

컨테이너 런타임이란 컨테이너를 실행하고 관리하는 도구이며, 컨테이너의 **생성, 실행, 중지, 삭제, 네트워크 연결, 리소스 관리(CPU, 메모리)** 등의 역할을 갖는다. 컨테이너 자체를 만드는 저수준 런타임과 컨테이너를 논리적으로 실행하는 고수준의 런타임으로 나뉜다.

### **컨테이너 런타임 종류**

#### **저수준 런타임 (Open Container Initative, OCI)**

- 컨테이너 실행을 담당하는 가장 기본적인 런타임
- OCI(Open Container Initative) 표준을 따른다.
- 예 : `runC`{: .text-blue} `gVisor`{: .text-blue} `Kata Containers`{: .text-blue}

#### **고수준 런타임(Container Runtime Interface, CRI)**

- 쿠버네티스에서 제공하는 컨테이너 런타임 추상화 계층
- Kubernetes 및 Docker 같은 오케스트레이션 툴과 통신
- 컨테이너 이미지 관리, 네트워크 설정 등 수행
- 예 : `containerd`{: .text-blue} `CRI-O`{: .text-blue} `Docker`{: .text-blue}

고수준 런타임 이용을 목적으로 하며 containerd, CRI-O에 대해서 알아보자.

### **Containerd**

containerd는 Docker에서 분리된 컨테이너 런타임이다. Docker의 핵심 컴포넌트 중 하나로 컨테이너 생애주기와 이미지 관리를 담당한다.

컨테이너 시작과 종료에 필요한 핵심 동작을 담당하며, Docker외 다른 컨테이너 관리 도구 K8S에도 이용할 수 있다. 

K8S 이용시 도커 자체를 런타임으로 이용하는 것과 containerd를 이용하는 부분에서 차이가 있다고 한다.

- **K8S + containerd** : containerd가 중지된 상태에서도 명령어를 통해 pods 확인, 생성, 삭제할 수 있다.

  → API 서버가 정상적으로 동작하게된다.

- **K8S + Docker** : Docker를 중지하면 Pods 명령어를 이용하더라도 해당 API 서버가 Command를 받지 해 실행되지 않는다.

> 참고 : [https://velog.io/@rockwellvinca/도커와-Containerd-런타임](https://velog.io/@rockwellvinca/%EB%8F%84%EC%BB%A4%EC%99%80-Containerd-%EB%9F%B0%ED%83%80%EC%9E%84){:target="\_blank"}

### **CRI-O**

Kubernetes 전용 컨테이너 런타임이며, **컨테이너를 실행하고 관리하는 데 필요한 최소한의 기능만을 제공**하며, **Kubernetes의 CRI**와 직접 통신합니다. 

- **Kubernetes 전용**
  - Docker와 달리 Kubernetes 환경에서만 최적화되어 사용됩니다.
  - Kubernetes의 CRI(Container Runtime Interface) 표준을 따르며, Kubernetes와 직접 상호작용한다.
- **가벼운 런타임**
  - CRI-O는 **불필요한 기능을 제거하고 Kubernetes에 필요한 기능만 제공**하여 **가볍고 빠른** 실행을 목표로 합니다.
- **컨테이너 실행에 필요한 기본적인 기능만 제공**
  - CRI-O는 **컨테이너 실행과 이미지를 다루는 데 필요한 최소한의 기능**만 제공하며, Docker와 같은 추가적인 기능은 포함하지 않습니다.
- **OCI(Open Container Initiative) 호환**
  - CRI-O는 OCI 표준을 따르는 런타임으로, 다양한 컨테이너 런타임(`runC` `gVisor` 등)과 호환됩니다.

### **Containerd와 CRI-O 차이**

| containerd | CRI-O |
| --- | --- |
| 범용 컨테이너 런타임 (Kubernetes, Docker 등) | Kubernetes 전용 런타임 |
| 컨테이너 실행을 위한 다양한 기능을 제공 | Kubernetes와 통합을 목적으로 간소화된 기능 제공 |
| 컨테이너 빌드, 이미지 저장소, 네트워크 플러그인 등 추가 기능 제공 | Kubernetes에서 필요한 기본적인 기능만 제공 |
| 여러 기능을 포함한 더 복잡한 구성 요소 (예: 이미지 관리, 네트워크 설정) | Kubernetes CRI에 맞게 구성 |
| Kubernetes 및 다른 시스템(Docker 등)에도 사용 가능 | Kubernetes만 사용 가능 |
| 많은 기능을 제공하지만 상대적으로 더 무겁고 복잡함 | 가벼움, 빠름 |

Containerd와 CRI-O는 위와 같은 차이가 있다. K8S에 컨테이너 런타임이 있으며 처음에는 Docker를 이용해 진행해보려 했다. 하지만,이번에 학습 및 사용해보려는 컨테이너 런타임은 Containerd이다. 그 이유는 아래와 같다.

1. K8S 공식 블로그에서 V1.28 이후로 dockershim을 통한 Docker를 지원하지 않는다고 한다. 이유는 dockershim의 추가적인 유지 및 관리를 필요로 하며 오류의 가능성이 높으며, 도커는 CRI를 준수하지 않기 때문이라고 한다.

  > 참고 : https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/

2. CRI-O는 Kubernetes를 위한 컨테이너 런타임으로 가볍고 빠르다는 이점이 있다. 다만, 이전부터 도커를 이용하면서 배운 지식을 이번 기회로 조금 더 심화 학습과정을 거칠 수 있다고 생각하며, containerd에서의 유연한 기능을 이용을 목표로 한다.

## **KinD(Kubernetes in Docker)란?**

Docker 컨테이너 내부에서 Kubernetes 클러스터를 실행할 수 있도록 해주는 경량 Kubernetes 솔루션이며, VM 없이 로컬에서 간편하게 K8S 클러스터를 실행할 수 있는 도구이다.

### **핵심 개념**

- Docker 컨테이너 기반의 Kubernetes 클러스터
  - KinD는 Kubernetes 노드를 Docker 컨테이너로 실행하며, 각 노드는 실제 머신이 아니라 하나의 Docker 컨테이너이다.
  - 예를 들어, 1개의 Control Plane과 2개의 Worker 노드를 갖춘 클러스터를 만들면 Docker 컨테이너 3개가 생성된다.
- 빠른 테스트 및 개발 환경 구축 가능
  - 로컬 환경에서 Kubernetes 클러스터를 실행할 때 **Minikube**처럼 **VM 없이 가볍게 실행 가능**
  - CI/CD 환경에서도 유용 (GitHub Actions, Jenkins 등에서 Kubernetes 테스트 가능)
  - Production 환경에서는 사용하지 않음 (로컬 개발 및 테스트 목적)
- 멀티 노드 클러스터 지원
  - 기본적으로 싱글 노드(1개 컨테이너) 클러스터를 실행하지만, 여러 개의 노드를 포함하는 멀티 노드 클러스터도 설정 가능하다.

## **KinD를 이용해 K8S 클러스터 구축**

### **KinD 설치 (MacOS - Homebrew)**

```bash
$ brew update # homebrew 업데이트
$ brew install kind
```

![KinD 설치]({{site.url}}/assets/img/KinD/1-KinD_Install.png)

![KinD 버전]({{site.url}}/assets/img/KinD/2-KinD_Version.png)

### **클러스터 생성**

#### **단일 노드 클러스터 생성**

```bash
kind create cluster --name {클러스터 명}
```

![K8S 클러스터 생성]({{site.url}}/assets/img/KinD/3-K8S_Cluster_Create.png)

![K8S 클러스터 단일 노드 조회]({{site.url}}/assets/img/KinD/4-K8S_Cluster_Single_Node.png)

#### **커스텀 클러스터 생성**

단일 클러스터 생성시에는 마스터 노드 역할인 control-plane노드만 생성된다. Worker Node 추가는 Yaml을 통해 커스텀으로 가능하다.

```yaml
# kind-cluster.yaml

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
networking:
  apiServerAddress: "127.0.0.1"
  apiServerPort: 6443
```

```yaml
kind create cluster --name {클러스터 명} --config kind-cluster.yaml
```

#### **Docker의 MySQL 이용시 네트워크 설정**

```bash
$ docker network connect {mysql-network} {control-plane 컨테이너명}
$ docker network connect {mysql-network} {worker 컨테이너명}
```

#### **Node 확인**

```bash
$ kubectl get nodes
```

![K8S 클러스터 노드 조회]({{site.url}}/assets/img/KinD/5-K8S_Cluster_Custom_Nodes.png)

#### **전체 정보 확인**

```bash
$ kubectl get all -A
```

#### **도커 이미지 로드**

로컬에서의 도커 이미지를 이용하기 위해 클러스터에 이미지를 가져온다.

```bash
$ kind load docker-image {repository-name}/{image-name}:{tag} --name {cluster-name}
```

### **설정 및 배포 테스트**

ConfigMap을 통해 배포에 필요한 환경 변수 정보를 설정하여 이용할 수 있다.

#### **ConfigMap 리소스 설정**

```yaml
# tp-config.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: tp-config
data:
  DB_URL: "{mysql-docker-container-name}"
  DB_PORT: "{mysql-port}"
  DB_NAME: "{mysql-db-name}"
  DB_USER: "{mysql-username}"
  DB_PWD: "{mysql-password}"
  SPRING_PROFILES_ACTIVE: "{project-profiles}"
```

#### **Deployment 리소스 설정**

```yaml
# travel-planner-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: travel-planner-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: travel-planner
  template:
    metadata:
      labels:
        app: travel-planner
    spec:
      hostPID: true # 호스트의 PID 네임스페이스 사용
      containers:
      - name: travel-planner-app
        image: {docker-repository-name}/{docker-container-name}:${tag}
        ports:
        - containerPort: 8080 # 컨테이너에서 열리는 포트 (예: 8080)
        env:
        - name: DB_URL
          valueFrom:
            configMapKeyRef:
              name: tp-config
              key: DB_URL
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: tp-config
              key: DB_PORT
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: tp-config
              key: DB_NAME
        - name: DB_USER
          valueFrom:
            configMapKeyRef:
              name: tp-config
              key: DB_USER
        - name: DB_PWD
          valueFrom:
            configMapKeyRef:
              name: tp-config
              key: DB_PWD
        - name: SPRING_PROFILES_ACTIVE
          valueFrom:
            configMapKeyRef:
              name: tp-config
              key: SPRING_PROFILES_ACTIVE
        resources:
          requests:
            cpu: "2000m" # CPU 2 Core
            memory: "1024Mi" # Memory 1GiB
          limits:
            cpu: "4000m" # CPU 4 Core Limit
            memory: "2048Mi" # Memory 2GiB Limit
```

- hostPID : 각각의 독립적인 PID를 이용하는 Pod가 호스트 노드의 PID 네임스페이스를 공유하여 Pod 내에서 실행되는 프로세스가 호스트 노드의 모든 프로세스를 볼 수 있다.
  - 컨테이너에서 `ps aux`, `top`, `kill` 명령어 등을 실행하면 호스트의 모든 프로세스를 관리 가능하며 `시스템 모니터링`, `컨테이너가 호스트의 특정 프로세스를 제어`, `네트워크 및 보안 툴` 사용 가능하다.

#### **Service 리소스 설정**

```yaml
# travel-planner-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: travel-planner-service
spec:
  selector:
    app: travel-planner
  ports:
  - protocol: TCP
    port: 8080 # 서비스 내에서 사용할 포트
    targetPort: 8080 # 컨테이너에서 사용할 포트
```

#### **리소스(ConfigMap, Service, Deployment) 생성**

```bash
$ kubectl apply -f tp-config.yaml
$ kubectl apply -f travel-planner-service.yaml
$ kubectl apply -f travel-planner-deployment.yaml
```

#### **Pods 세부 정보**

```bash
$ kubectl get pods -o wide
```

#### **서비스의 엔드포인트 정보**

```bash
$ kubectl get endpoints
```

- 서비스가 라우팅하는 Pod의 내부 IP 및 포트 확인을 할 수 있다.

#### **서비스 Port Forward**

아래 명령어를 이용해 로컬에서 8080 포트로 서비스를 요청할 수 있다.

```bash
$ kubectl port-forward service/travel-planner-service 8080:8080
```

![K8S Port Forward]({{site.url}}/assets/img/KinD/6-K8S_Local_Port_Forward.png)

![K8S Port Forward Result]({{site.url}}/assets/img/KinD/7-K8S_Local_Port_Forward_Result.png)

### **NodePort를 통한 서비스 접근**

`kind-cluster.yaml`{: .text-blue}로 다시 돌아가 로컬 Host OS에서 접속할 **worker**에 **extraPortMappings** 설정을 추가해준다.

```yaml
# kind-cluster.yaml

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
  extraPortMappings:
  - containerPort: 30080  # 클러스터 내 서비스의 노드포트
    hostPort: 30080      # 외부에서 요청할 포트
    protocol: TCP
networking:
  apiServerAddress: "127.0.0.1"
  apiServerPort: 6443
```

cluster 재시작 이후 **Docker의 MySQL 이용시 네트워크 설정, 클러스터 도커 로컬이미지 로드** 작업을 진행한다.

```yaml
# travel-planner-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: travel-planner-service
spec:
  selector:
    app: travel-planner
  ports:
  - protocol: TCP
    port: 8080 # 서비스 내에서 사용할 포트
    targetPort: 8080 # 컨테이너에서 사용할 포트
    nodePort: 30080 # 외부에서 접근할 노드 포트
  type: NodePort # NodePort 서비스 타입
```

- 서비스 구성에서 `nodePort`{: .text-blue}와 `type: NodePort`{: .text-blue}를 설정 후 service를 다시 적용한다.

이후 아래 명령어를 통해 PORT(S)에 8080 내부 포트가 30080 외부포트로 설정된 것을 볼 수 있다.

```bash
$ kubectl get svc travel-planner-service
```

![K8S NodePort 설정]({{site.url}}/assets/img/KinD/8-K8S_NodePort_Setting.png)

### **Pod의 Node 설정**

NodePort를 이용할 경우 Worker 노드가 2개 이상일 때 각 노드에 대해서 외부 포트를 지정해야한다. 이럴 때 각 노드의 외부 포트를 30080, 30081로 설정하면 Pod는 랜덤으로 두 노드 중 하나가 지정될 것이며, 서비스 이용에 포트 또한 바뀌어야한다. 이에 아래와 같이 간단하게 Pod의 Node를 설정해줄 수 있다.

```yaml
# travel-planner-deployment.yaml

apiVersion: apps/v1
 ...
    spec:
      hostPID: true
      nodeSelector:
        kubernetes.io/hostname: {Worker 노드 이름}
      containers:
      ...
```

![K8S Pod Worker 설정]({{site.url}}/assets/img/KinD/9-K8S_Pod_Worker_Setting.png)

## **롤링 업데이트(Rolling Update)**

K8S에서 롤링 업데이트(Rolling Update)는 기존 Pod를 하나씩 새로운 버전으로 교체하는 방식으로 무중단 배포를 지원한다. 즉, 애플리케이션을 배포할 때 전체 서비스를 중단하지 않고 순차적으로 새로운 버전으로 업데이트하는 방식이다.

### **롤링 업데이트 동작 방식**

1. 새로운 버전의 애플리케이션을 포함한 **새로운 Pod를 생성**
2. 기존 Pod를 하나씩 제거하면서 새로운 Pod로 교체
3. 모든 Pod가 새로운 버전으로 변경될 때까지 위 과정을 반복

### **롤링 업데이트 설정**

Deployment의 `spec.strategy`에서 **RollingUpdate** 전략을 사용하면 자동으로 롤링 업데이트가 수행됩니다. Service는 기존 설정에서 변경 사항이 없으며 Deployment에 strategy를 설정해준다.

```yaml
# travel-planner-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: travel-planner-app
spec:
  replicas: 2 # 2개의 Pod 운영
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # 업데이트 중 최대 1개의 Pod만 Down
      maxSurge: 1         # 새로운 Pod를 추가하면서 배포 진행
  selector:
    matchLabels:
      app: travel-planner
  template:
    metadata:
      labels:
        app: travel-planner
    spec:
      hostPID: true
      nodeSelector:
        kubernetes.io/hostname: {Worker 노드 이름} # worker 노드의 이름
      terminationGracePeriodSeconds: 20 # 종료시 Gracefull Down 시간 설정
      containers:
      - name: travel-planner-app
        ...
        readinessProbe:  # Pod가 정상적으로 준비될 때까지 트래픽을 받지 않도록 설정
          httpGet:
            path: /actuator/health/readiness # Healthy Check Path
            port: 8080
          initialDelaySeconds: 3 # 초기 딜레이 시간
          periodSeconds: 5 # 반복 주기
        livenessProbe:   # 컨테이너가 정상 동작하는지 확인
          httpGet:
            path: /actuator/health/liveness # Healthy Check Path
            port: 8080
          initialDelaySeconds: 5 # 초기 딜레이 시간
          periodSeconds: 10 # 반복 주기
          successThreshold: 3 # 설정된 횟수만큼 실패가 반복되면 Pod가 재시작
          failureThreshold: 5 # 설정된 횟수만큼 성공한 후 Pod가 정상으로 간주
```

### **Health Check를 위한 Spring Boot Actuator 설정**

Spring Boot Actuator 의존성 추가 후 아래 설정 추가

```yaml
# application.yaml

...
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics
  endpoint:
    health:
      enabled: true
      show-components: ALWAYS
      show-details: NEVER # 세부 정보가 출력되기에 미사용
    metrics:
      enabled: true
  health:
    db:
      enabled: true
    diskspace:
      enabled: true  # 디스크 공간 체크 활성화
      threshold: 1073741824  # 1GB 이하이면 DOWN 처리
    ping:
      enabled: true  # Ping 체크
```

`http://127.0.0.1:8080/actuator/health`로 접속하여 서비스 상태 확인할 수 있다.

![K8S Service Port Forward 결과1]({{site.url}}/assets/img/KinD/10_1-Service-Port-Forward.png)

`show-details: ALWAYS`{: .text-blue} 설정 후 출력 결과

![K8S Service Port Forward 결과2]({{site.url}}/assets/img/KinD/10_2-Actuator_Details.png)

### **이미지 및 롤링 업데이트**

K8S에 이미 실행중인 Deployment의 컨테이너 이미지 업데이트에 사용한다. 이를 통해 애플리케이션의 버전 업그레이드나 수정된 코드 배포를 쉽게 처리할 수 있다. 해당 명령어의 경우 롤링 업데이트가 바로 진행된다.

#### **이미지 업데이트를 통한 재배포**

```bash
$ kubectl set image deployment {deployment-name} {deployment-name}={deployment-name}:{new-tag}
```

#### **롤링 업데이트 재배포**

```bash
$ kubectl rollout restart deployment {deployment-name}
```

![K8S Rolling Update Deployment]({{site.url}}/assets/img/KinD/11-Pod_Rolling_Update.png)

총 2개의 Pod에서 기존 Replica 1개 Pod가 종료되고 신규로 Pod가 2개 추가되며 신규 Pod들이 정상 상태가 되면 남은 Pod 1개 또한 종료된다.

#### **롤링 업데이트 과정에 대한 의문점**

rolling update 설정에서 1개씩 파드를 종료하고 배포하는 방식으로 설정했는데 왜 동시에 2개의 파드가 올라올까?

> 쿠버네티스가 `availableReplicas` 수를 계산할 때 종료 중인(terminating) 파드는 포함하지 않으며, 이 수는 `replicas - maxUnavailable` 와 `replicas + maxSurge` 사이에 존재한다. 그 결과, **롤아웃 중에는 파드의 수가 예상보다 많을 수 있으며**, 종료 중인 파드의 `terminationGracePeriodSeconds`가 만료될 때까지는 디플로이먼트가 소비하는 총 리소스가 `replicas + maxSurge` 이상일 수 있다.
참고 : [https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment](https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment){:target="\_blank"}

### **롤백 방법**

신규 버전을 배포 후 이슈 발생시 롤백을 해야할 필요가 있다. 이 경우 아래 명령어를 통해 롤백을 할 수 있다.

```bash
# 이전 버전으로 롤백
$ kubectl rollout undo deployment my-app
```

```bash
# 버전 리비전 확인
$ kubectl rollout history deployment {deployment-name}

# 특정 버전으로 롤백
$ kubectl rollout undo deployment {deployment-name} --to-revision={버전}
```

## **K8S 대시보드**

### **K8S 대시보드 설치**

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

### **서비스 계정 생성 및 Cluster 바인딩**

```yaml
# k8s-account-binding.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

```bash
$ kubectl apply -f k8s-account-binding.yaml
```

### **Proxy 서버 실행**

```bash
kubectl proxy
```

> http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

### **토큰 생성**

```bash
$ kubectl -n kubernetes-dashboard create token admin-user
```

### **대시보드 화면**

![K8S Dashboard_1]({{site.url}}/assets/img/KinD/12-K8S_Dashboard_1.png)

![K8S Dashboard_2]({{site.url}}/assets/img/KinD/13-K8S_Dashboard_2.png)

## **트러블 슈팅**

### **문제 발생**

Spring Acturator 설정 이후 아래 Metric 빈 생성이 계속해서 실패하는 원인이 발생

> org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'processorMetrics’
…
Caused by: java.lang.NullPointerException: Cannot invoke "jdk.internal.platform.CgroupInfo.getMountPoint()" because "anyController" is null

### **원인 분석**

JDK 17 이슈 : JDK 17에서 Cgroup V2(컨테이너 리소스 제한 관련)의 경우 K8S, Docker와 같은 컨테이너 환경에서 Cgroup 정보를 읽지 못하는 경우가 발생할 수 있으며, 특히 jdk 17 버전에서 자주 발생한다고 한다.

> JVM이 `/proc/self/mountinfo`{: .text-red}에서 `/sys/fs/cgroup`{: .text-red} 혹은 파일이 없거나 권한 문제로 접근이 불가능할 때 발생

실제 Docker 컨테이너 내부에 접속하여 `/sys/fs/cgroup`{: .text-red} 를 확인해보았을 때 내부 관련 파일들은 있었다. 그렇다면, 파일들에 대한 접근 권한의 문제로 이슈가 발생하는 것으로 생각하게 되어 해결방법을 찾아보았다.

![JDK 17 버전 Metrics 이슈]({{site.url}}/assets/img/KinD/14-JDK_17_ea_Cgroup.png)

### **해결방안**

1. JVM 옵션으로 Cgroup 감지 비활성화 (`JAVA_OPTS : "-XX:-UseContainerSupport”`{: .text-blue})
  - 모니터링을 위해 리소스 정보를 확인할 필요가 있어 보이기에 활성화 필요

2. JDK 버전을 17.0.9 이상으로 변경 필요
  - 현재 JDK : openjdk:17-alpine (JDK 버전 : 17-ea)
  - 변경 JDK : bellsoft/liberica-openjdk-alpine:17.0.9 (JDK 버전 : 17.0.9)

### **결과**

JDK 버전 변경 이후 Cgroup 내부 파일을 확인과 K8S에 정상적인 배포가 이루어진 것으로 보아 파일 접근 권한이 원인인 것으로 이슈를 해결할 수 있었다.

![JDK 17 버전 Metrics 이슈 해결]({{site.url}}/assets/img/KinD/15-JDK_17_0_9_Cgroup.png)
