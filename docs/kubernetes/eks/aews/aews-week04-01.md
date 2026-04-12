---
title: AEWS 4주차 - 1) EKS AuthN/AuthZ
date: 2026-04-12
draft: true
categories:
  - Kubernetes
  - AWS
  - EKS
  - AEWS
tags:
  - AEWS
---
> *CloudNet 팀의 [2026년 AWS EKS Workshop Study 4기](https://gasidaseo.notion.site/26-AWS-EKS-Hands-on-Study-4-31a50aec5edf804b8294d8d512c43370) 4주차 학습 내용을 담고 있습니다.*

## 1. 들어가며

EKS 운영 관점에서의 트래픽은 요청 경로에 따라 크게 두 가지로 나눌 수 있습니다.

1. 운영 트래픽 (Inbound: 클러스터 API 접근)
      1. 주체: 관리자·개발자·CI/CD 시스템 등
      2. 경로: 외부 → `kubectl` / `eksctl` / API 호출 → EKS Control Plane
      3. 특징: AWS IAM을 통해 '신원'을 먼저 확인(AuthN)한 후, 쿠버네티스 내부의 RBAC을 통해 '권한'을 결정(AuthZ)
2. 워크로드 트래픽 (Outbound: AWS API 호출)
      1. 주체: 클러스터 내부에서 실행 중인 Pod(애플리케이션)
      2. 경로: Pod → AWS API (S3, DynamoDB 등)
      3. IRSA(IAM Roles for Service Accounts)나 EKS Pod Identity 같은 자격 증명 경로를 통해 AWS 리소스에 접근

EKS는 AWS 인프라 위에서 동작하는 관리형 Kubernetes이기 때문에, Kubernetes API 접근이 AWS IAM 인증과 연결됩니다. 즉 운영자가 `kubectl`을 사용할 때는 보통 IAM 기반 토큰으로 먼저 인증되고, 그 다음 Kubernetes 권한 체계인 RBAC 기반의 인가가 진행됩니다.

반면 Pod가 AWS API를 호출할 때는 Kubernetes API 인증 체계가 아니라 AWS 자격증명 경로(IRSA, Pod Identity, Node Role)를 통해 IAM 권한이 평가됩니다.
이 차이점을 기준으로 EKS 인증/인가 구조를 정리해보겠습니다.

> Pod → Kubernetes API 경로는 ServiceAccount(AuthN) + RBAC(AuthZ)로 내부에서 동작하여 구조가 단순한 편입니다.

## 2. AuthN vs AuthZ


| 구분 | 질문 | EKS 예시 |
| --- | --- | --- |
| Authentication (AuthN) | "당신은 누구인가?" | IAM Principal 검증, STS Token 검증 |
| Authorization (AuthZ) | "당신은 무엇을 할 수 있는가?" | K8s RBAC, EKS access policy, IAM policy |

- **인증(Authentication)**: 요청자의 신원을 확인한다.
- **인가(Authorization)**: 인증된 요청자가 리소스/동작에 대한 권한을 가지는지 판정한다.
- **핵심 포인트**: EKS에서는 어느 시점의 어떤 API(K8s API vs AWS API)에 대한 인증/인가인지 문맥을 반드시 붙여야 의미가 생긴다.


> 인증에 성공했다고 해서 모든 것을 할 수 있는 것은 아닙니다. 로그인은 되는데 명령어가 거부(Forbidden)된다면 인가(AuthZ)의 문제입니다.

EKS 보안 모델은 호출 대상(K8s API vs AWS API)과 요청의 방향에 따라 인증·인가 주체가 완전히 달라집니다. 따라서 트러블슈팅 시에는 '누가(Identity)', '어떤 엔드포인트(API Server vs AWS Endpoint)'에 접근하는지를 정확히 특정해야 합니다.

**예시**

- EKS에서 `kubectl` 요청의 인증은 IAM 기반이다.
- Kubernetes 오브젝트에 대한 인가는 RBAC 또는 EKS access policy가 담당한다.
- Pod의 AWS API 호출 권한은 IRSA/Pod Identity/Node Role이 담당한다.


## 3. EKS 인증/인가

EKS에서는 IAM, Kubernetes RBAC, ServiceAccount, STS, OIDC 같은 용어가 등장합니다. 이들이 모두 "권한"과 관련 있어 보이지만, 실제로는 서로 다른 경로에서 서로 다른 질문에 답합니다. 아래 세 문장은 전부 독립적인 문제입니다. 

- 어떤 IAM 사용자/역할이 `kubectl`로 클러스터에 들어올 수 있는가 (인증) 
- 들어온 사용자가 `pods`, `deployments`, `secrets`에 대해 무엇을 할 수 있는가 (인가)
- 어떤 Pod가 `S3`, `DynamoDB`, `Route53` 같은 AWS API를 호출할 수 있는가 (워크로드의 AWS API 호출 권한)

### 3.1. AWS IAM과 K8s 두 인증/인가 체계의 결합

EKS는 본질적으로 서로 다른 **AWS IAM 세계**와 **K8s 세계**를 연결합니다. 

| 구분 | AWS IAM 세계 | Kubernetes 세계 |
| --- | --- | --- |
| 인증 주체 | IAM User/Role | UserInfo, ServiceAccount |
| 인가 단위 | AWS API Action + ARN | Resource + Verb + Namespace |
| 대표 정책 | IAM Policy | Role/ClusterRole + RoleBinding |

EKS 운영에서는 **어느 세계에서 누구를 어떤 주체로 해석할지**를 고려하여 권한을 설정해야 합니다.

> K8s에는 User 리소스가 없습니다. UserInfo는 API 서버가 요청을 처리할 때 사용하는 가상의 신원 정보입니다. EKS는 IAM 자격 증명을 확인한 뒤, "이 IAM 유저는 K8s에서 alice라는 이름표(Username)를 가진다"라고 API 서버에 알려 RBAC과 연결합니다.

### 3.2. K8s API (사용자 → K8s API) 동작 방식

운영 트래픽은 컨트롤 플레인의 API server로 들어가며, 여기서 인증과 인가, 정책 검사가 수행됩니다. Pod 생성, ConfigMap 업데이트, Namespace 삭제 등의 K8s API server에 도달하는 모든 요청은 아래의 세 단계를 통과합니다.

![](images/i46g-rpmn0th46.png)


- **Authentication**: "당신은 누구인가?"
    - 유효한 자격증명으로 요청 주체의 신원을 확인합니다. 인증 실패 시 401 에러가 발생합니다. 

- **Authorization**: "이 작업을 할 수 있는가?"
    - 인증된 주체가 해당 리소스와 동작에 대해 권한이 있는지 확인합니다. 허용 규칙이 없으면 403 에러가 발생합니다.

- **Admission**: "이 리소스를 받아들여도 되는가?"
    - 인증과 인가를 통과했더라도, 조직 정책에 맞지 않으면 최종적으로 거부할 수 있습니다.

`kubectl` 명령어는 일반적으로 다음과 같이 처리됩니다. 

![](images/i38g-rpmn0th38-rpmn0.png)

1. Client에서 kubectl 명령어 실행
2. `kubeconfig` 파일 내 정의된 exec plugin 실행 → `aws eks get-token` 명령 호출
3. AWS STS GetCallerIdentity 호출을 위한 pre-signed URL 기반 bearer token 생성
4. EKS API server는 전달받은 bearer token을 Webhook 형태로 `aws-iam-authenticator`에 전달
5. 토큰이 유효하다면 해당 IAM principal의 ARN이 반환됨(AuthN)
6. 인증된 IAM Principal의 ARN을 Access Entry 또는 aws-auth를 활용하여 K8s subject(username/groups)에 매핑
7. IAM 사용자 'alice'는 K8s 사용자 'alice', 그룹 'platform-engineers'라는 K8s 내부 신원을 획득
8. 매핑된 신원을 바탕으로 Authorizer 체인(RBAC 등)을 돌려 해당 API(get, list, watch 등)의 실행 권한을 판정 (AuthZ)
9. Admission Webhook 단계에서 Policy 준수 여부를 확인
10. `etcd` 저장 및 요청 실행


### 3.3. AWS API (Pod → AWS API) 동작 방식

Pod가 S3, DynamoDB, Route53 같은 AWS 서비스 API를 호출할 때는 AWS 엔드포인트와 직접 통신합니다. (Kubernetes API server의 인증 체인을 사용하지 않습니다.)

이 통신의 핵심은 **Pod에 어떤 IAM 자격증명이 전달되는가**와 **그 자격증명을 AWS가 어떤 신뢰 관계로 받아들이는가**입니다.

![](images/fi4/12/20260Npm290-i29g-rpmn0th29-rpmn0.png)

Pod 내의 애플리케이션(AWS SDK)은 자체적인 Default Credential Provider Chain을 통해 자격증명을 탐색하며, 설정된 환경에 따라 다음 세 가지 경로 중 하나로 임시 자격증명(Temporary Credentials)을 발급받아 AWS API를 호출합니다.

- IRSA: ServiceAccount 토큰과 OIDC 신뢰를 이용해 STS `AssumeRoleWithWebIdentity`로 임시 자격증명을 발급
- Pod Identity: EKS 관리형 association을 통해 Pod에 IAM Role 자격증명을 전달
- Node IAM Role: 노드 역할을 Pod가 공유하는 단순 모델(Node 내의 모든 Pod가 동일한 권한을 갖게 되므로 최소 권한 원칙에 취약)

세 방식의 차이는 자격증 발급 주체와, 신뢰 경로, 권한 범위를 어디까지 줄 수 있는가에 있습니다.

| 방식 | 자격증명 발급 방식 | 신뢰 기반 | 권장 포인트 |
| --- | --- | --- | --- |
| IRSA | ServiceAccount 토큰 + OIDC + STS | OIDC provider + IAM trust policy | 가장 널리 쓰이는 표준 방식 |
| Pod Identity | EKS managed association | EKS 컨트롤 플레인이 백엔드에서 인증을 중개 | 설정이 단순하고 운영 편의성이 좋음 |
| Node IAM Role | 노드 인스턴스 프로파일 공유 | EC2 instance profile | 간단하지만 Pod 단위 분리가 어려움 |

Pod → AWS API는 Kubernetes RBAC가 아닌 AWS IAM 정책 평가 문제입니다.

이 경로는 아래 세 단계로 볼 수 있습니다.

1. 신뢰 수립(Trust Relationship): AWS가 해당 Pod/토큰을 신뢰할 수 있는가
2. SA 매핑: 어떤 ServiceAccount가 어떤 IAM Role을 쓸 수 있는가
3. 최소 권한(IAM Policy): IAM Role에 어떤 AWS API 권한을 줄 것인가

> Trust Relationship은 이 Role을 누가 Assume 할 수 있는가, IAM Policy는 Assume 된 Role이 어떤 AWS API를 실행할 수 있는가를 정의합니다.


## 4. EKS 인증/인가 세 개의 권한 영역
ㄴ
앞서 설명한 두 세계의 결합을 시스템 아키텍처 관점에서 보면, EKS의 인증/인가는 크게 3개의 영역(Layer)으로 나눌 수 있습니다.

### 4.1. AWS IAM

- 주요 역할:
    - IAM User/Role/Policy 관리
    - STS를 통한 임시 자격증명 발급
    - 신뢰 정책(trust policy) 기반의 role assumption 제어

### 4.2. EKS Control Plane

- 주요 역할:
    - Kubernetes API endpoint 제공
    - 인증된 IAM principal을 Kubernetes subject로 연결
    - Access Entry / access policy 관리
    - audit/authenticator 로그 생성

### 4.3. Kubernetes 인가 체계

- 주요 역할:
    - Role / ClusterRole 관리
    - RoleBinding / ClusterRoleBinding 관리
    - ServiceAccount 관리
    - Admission Controller 작동

EKS는 이 세 계층을 엮어주는 서비스입니다. 따라서 권한 오류가 발생했을 때도 **어느 영역에서 막혔는가**를 먼저 구분하면 원인 파악이 빠릅니다. 이 흐름을 이해하면 다음과 같은 오해를 피할 수 있습니다.

- ❌ kubectl은 장기 토큰을 넣고 다니는 도구가 아니다. (STS를 통해 매번 단기 토큰을 발급받습니다.)
- ❌ 로그인 가능(AuthN)과 작업 가능(AuthZ)은 서로 다른 단계다.
- ❌ IAM 권한(AdministratorAccess 등)이 있다고 해서 Kubernetes 작업 권한까지 자동으로 생기지 않는다.

## 5. 요약 및 트러블슈팅 가이드

EKS 보안은 AWS IAM과 K8s RBAC의 조합입니다. 두 세계의 차이를 명확히 인지하고, 어느 계층의 문제인지 문맥을 파악하는 것이 중요합니다.

| 문제 상황 | 레이어 | 확인 사항 | 
|---|---|---|
| Unauthorized </br>(로그인 실패) | 인증 </br>(AuthN) | `aws sts get-caller-identity`, </br>Access Entry(또는 `aws-auth`) 매핑 확인 | 
| Forbidden </br>(명령어 거부) | 인가</br>(AuthZ) | Role/ClusterRole, </br>RoleBinding/ClusterRoleBinding, </br>Access Policy 범위 확인 |
| AWS API Error </br>(S3 등 접근 불가) | 워크로드 권한 | ServiceAccount 매핑(IRSA/Pod Identity), </br>IAM Role의 Trust Policy/Permission Policy 확인 | 
