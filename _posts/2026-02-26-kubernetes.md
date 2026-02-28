---
title: Kubernetes 실습, 기초 정리
description: desired state 관점에서 Kubernetes 구성요소(Control Plane/Worker Node)와 기본 리소스(Pod/Deployment/Service/Ingress), 실습 사례까지 정리
author: lucas
date: 2026-02-26 18:00:00 +0900
categories: [Backend, DevOps]
tags: [kubernetes, k8s, control-plane, kubelet, service, deployment, ingress]
pin: false
math: false
mermaid: false
---

## 1. 쿠버네티스 개요

> 쿠버네티스는 컨테이너 기반 애플리케이션의 **원하는 상태(desired state)** 를 유지하는 오케스트레이션 시스템이다.
> 
- Docker: 컨테이너 실행 도구
- Kubernetes: 컨테이너의 배치, 복제, 복구, 확장을 관리하는 시스템

![image](https://kubernetes.io/images/docs/kubernetes-cluster-architecture.svg)

[ Kubernetes Cluster 구조]

```
Cluster
 ├── Control Plane
 │     ├── API Server
 │     ├── Scheduler
 │     ├── Controller Manager
 │     └── etcd
 │
 └── Worker Node
       ├── kubelet
       ├── container runtime (containerd)
       └── kube-proxy
```

Control Plane은 **원하는 상태를 저장·판단·배치**하고, Worker Node는 **실행·보고**한다.

> 예시: SpecRAG 프로젝트 (k3s)  
> - EC2 한 대에 k3s 설치 → 단일 노드에서 Control Plane + Worker가 같이 동작  
> - 이후 `kubectl apply`로 리소스를 선언하고 상태를 쿠버네티스가 유지  
{: .prompt-tip }

---

## 2. Control Plane

쿠버네티스의 모든 리소스는 YAML(manifest) 형태로 선언된다. 
이 매니페스트는 공통적으로 다음 구조를 가진다.

- **apiVersion / kind** → 어떤 리소스인지 정의
- **metadata** → 이름, 네임스페이스, 라벨 등 식별 정보
- **spec** → 원하는 상태(desired state)
- **status** → 현재 상태(actual state, 시스템이 채움)

Control Plane은 이 manifest의 **spec을 기준으로 상태를 유지하는 역할**을 한다. 

---

### 2.1 구성 요소

---

| 구성요소 | 역할 |
| --- | --- |
| API Server | 모든 요청 수신, 상태 변경의 단일 진입점 |
| Scheduler | Pod가 실행될 Worker Node 선택 |
| Controller Manager | spec(원하는 상태)과 status(현재 상태)를 맞추는 루프 |
| etcd | 클러스터 리소스의 원하는 상태 저장 |

### 2.2 동작 흐름

---

- STEP 1 : `kubectl apply`
- STEP 2 : API Server가 YAML(manifest)을 수신
- STEP 3 : 리소스의 spec이 etcd에 저장됨
- STEP 4 : Controller가 spec과 status를 비교
- STEP 5 : 차이가 있으면 리소스를 생성/수정
- STEP 6 : Scheduler가 Pod를 실행할 Worker Node 선택

```
kubectl apply -k k8s/
```

- `k8s/` 아래의 여러 YAML(manifest)을 한 번에 적용  
- 각 리소스의 spec이 etcd에 저장  
- chat-server Deployment, redis, kafka 등의 spec(replicas, image 등)에 따라 ReplicaSet/Pod 생성  
- Scheduler가 해당 Pod를 실행할 노드를 선택  

{: .prompt-tip }

---

## 3. Worker Node

### 3.1 구성 요소

---

| 구성요소 | 역할 |
| --- | --- |
| kubelet | Pod 실행/상태 보고 |
| container runtime (containerd) | 컨테이너 실행 |
| kube-proxy | Service 트래픽 라우팅 |

### 3.2 동작 흐름

---

- STEP 1: Control Plane의 Scheduler가 특정 노드에 Pod를 배치
- STEP 2: kubelet이 API Server 상태를 watch
- STEP 3: containerd가 이미지 pull 후 컨테이너 실행
- STEP 4: kubelet이 상태를 API Server에 보고

**예시: SpecRAG 프로젝트 — Pod Status : ImagePullBackOff 발생**

```
실행 명령어 : kubectl get pods -n specrag
결과 : chat-server-xxxxx   ImagePullBackOff
```

- Scheduler는 Pod 배치를 완료
- kubelet은 실행을 시도
- containerd가 ECR에서 이미지를 pull하지 못함
- 재시도를 반복하다가 back-off 상태로 전환

```
kubectl describe pod chat-server-xxxxx -n specrag
```

Event 영역에서 다음 메시지를 확인했다.

- `pull access denied`
- `authorization token has expired`

문제는 ECR 인증이었다. 기존 Deploy 스크립트를 이용하지 않고 새로 작성하면서 ECR 인증 로직이 빠져 있었고, 스크립트를 수정하는 것이 맞지만 즉시 해결을 위해 직접 로그인하였다.

```
aws ecr get-login-password | docker login ...
```

이후 Pod를 재시작했고, 이미지 pull이 정상 수행되며 `Running` 상태로 전환되었다.

이 과정은 Worker Node의 container runtime 단계에서 발생한 문제이며, kubelet이 해당 상태를 API Server에 보고한 결과가 `ImagePullBackOff`로 나타난 것이다.

{: .prompt-tip }

---

## 4. 쿠버네티스 기본 리소스 구조

### 4.1 Pod

---

Pod는 쿠버네티스의 최소 실행 단위다.

- 하나 이상의 컨테이너를 포함 가능
- Pod 내부 컨테이너는 네트워크/IP를 공유
- Pod는 교체 가능한 단위(죽으면 새로 생성)
- 상태
    - `Running`: 정상 실행
    - `CrashLoopBackOff`: 프로세스 반복 종료
    - `ImagePullBackOff`: 이미지 pull 실패

> 내 프로그램이 실제로 실행되는 최소 단위  
{: .prompt-tip }

### 4.2 Deployment

---

Deployment는 애플리케이션 배포 단위다. 

- Pod 복제 관리(replicas)
- 롤링 업데이트
- 장애 시 자동 복구
- 롤백 가능

> 내 프로그램을 몇 개 실행할지 관리하는 객체  
{: .prompt-tip }

### 4.3 Service

---

Service는 **Pod들의 집합에 고정된 네트워크 주소를 부여하는 리소스**다.

각 pod의 IP는 재생성 시 변경되므로, pod 집합을 가리키는 고정 주소가 필요하다.

- 특정 label을 가진 Pod들을 묶고
- 하나의 **고정된 가상 IP(ClusterIP)** 를 제공한다.
- 내부에서 L4 로드밸런싱 수행

```
Pod A (10.0.1.3)
Pod B (10.0.1.7)
Pod C (10.0.1.9)

        ↓

Service (ClusterIP: 10.96.0.15)

        ↓

요청은 내부적으로 Pod들 중 하나로 전달됨
```

| 타입 | 의미 |
| --- | --- |
| ClusterIP | 클러스터 내부에서만 접근 가능 |
| NodePort | 각 노드의 특정 포트를 열어 외부 접근 허용 |
| LoadBalancer | 클라우드 LB 생성 |

```
kind: Service
spec:
  selector:
    app: chat-server 
  type: NodePort
```

- `app: chat-server` 라벨을 가진 Pod들을 묶음
- NodePort 30080으로 외부 접근 허용
- EC2IP:30080으로 외부 접근 시 내부 로드밸런싱 수행

{: .prompt-tip }

> 실행된 프로그램들에 어떻게 접근할지 관리하는 객체  
{: .prompt-tip }

### 4.4 Ingress

---

Ingress는 **HTTP 레벨(L7)에서 요청을 어떤 Service로 보낼지 결정하는 리소스**다.

```
Client
 ↓
Ingress (L7)
 ↓
Service (L4)
 ↓
Pod
```

- 도메인 기반 분기
- 경로 기반 분기
- HTTPS(TLS) 처리
- 여러 Service를 하나의 진입점으로 통합

> SpecRAG 예시 : Ingress를 사용하지 않고 단일 Node에서 NodePort만 노출  
{: .prompt-tip }

---

## 5. 백엔드 개발자가 알아야 하는 포인트

> 쿠버네티스를 사용한다는 것은 단순히 배포 도구를 바꾸는 것이 아니다. 애플리케이션이 **단일 서버가 아니라 분산 환경에서 실행된다**는 의미다.

### 5.1 인스턴스는 여러 개다

- 요청은 어떤 인스턴스로 갈지 예측할 수 없다.
- 특정 인스턴스에 상태를 의존할 수 없다.

> 백엔드 개발자가 고려해야 할 것  
> - 요청은 항상 멱등하게 설계  
> - 중복 요청 대비  
> - 상태는 외부 저장소에 둔다  
> - 단일 실행 가정 금지  
{: .prompt-tip }

### 5.2 네트워크는 항상 실패할 수 있다

- 동일 클러스터 내부라도 네트워크다.

> 백엔드 개발자가 고려해야 할 것  
> - timeout 필수  
> - retry 전략 설계  
> - 부분 실패 전제 설계  
{: .prompt-tip }

### 5.3 로깅과 추적은 필수다

> - 로그는 구조화  
> - traceId 사용  
> - 서비스 간 trace 전파  
{: .prompt-tip }

### 5.4 트래픽은 분산된다

> - 세션은 외부 저장소  
> - 캐시 공유 전략 필요  
> - WebSocket/SSE 스케일 고려  
{: .prompt-tip }



### 참고 자료

1. [SpecRAG BE Project](https://github.com/Spec-RAG/Server)