---
title: AEWS 1주차 - EKS 소개 및 배포
date: 2026-03-14
categories:
  - Cloud
  - AWS
  - EKS
  - AEWS
tags:
  - AEWS
---
> *CloudNet 팀의 [2026년 AWS EKS Workshop Study 4기](https://gasidaseo.notion.site/26-AWS-EKS-Hands-on-Study-4-31a50aec5edf804b8294d8d512c43370) 1주차 학습 내용을 담고 있습니다.*

## Kubernetes의 핵심 구성요소
![Kubernetes Components](https://kubernetes.io/images/docs/components-of-kubernetes.svg)
/// caption
[https://kubernetes.io/docs/concepts/overview/components/](https://kubernetes.io/docs/concepts/overview/components/)
///

마이크로서비스 아키텍처로 애플리케이션을 운영하고자 한다면 서비스가 많아질 수록 확장성, 보안, 지속성, 부하 분산을 체계적으로 관리해야 합니다.
쿠버네티스는 물리적 머신, 가상 머신 또는 클라우드 환경에서 수백 개, 수천 개 단위까지의 대규모 컨테이너 기반 워크로드를 운영하는 데 도움을 주는 컨테이너 오케스트레이션 소프트웨어입니다. 

쿠버네티스 클러스터는 최소한 하나씩의 Control Plane과 Worker Node로 구성되어 있습니다.
Control Plane은 워크로드 스케줄링 및 기능 관리를 위한 진입점 역할을 하며, Worker Node는 Control Plane으로부터 할당받은 워크로드를 처리합니다. 

마이크로서비스 아키텍처에서는 애플리케이션 스택을 여러 개의 독립적인 서비스(Pod)로 분리해 배포하며, 이 서비스들은 서로 네트워크 통신을 통해 상호 작용합니다. 이때 Control Plane과 Worker Node의 구성 요소가 각각 어떤 역할을 하는지 이해하면 클러스터 동작을 더 잘 추적하고 문제를 빠르게 진단할 수 있습니다.

각 노드의 주요 구성 요소와 역할에 대해 간략하게 알아보겠습니다.

### Control Plane
컨트롤 플레인은 워크로드 스케줄링 및 클러스터 전체의 기능 관리를 위한 진입점 역할을 합니다. 관리자는 커맨드라인 도구 `kubectl`을 사용하여 쿠버네티스 클러스터와 상호작용할 수 있습니다.

- kube-apiserver: API 서버는 클라이언트가 쿠버네티스와 통신하는 데 사용하는 API 엔드포인트를 노출하는 핵심 컴포넌트입니다. 예를 들어, 쿠버네티스 클라이언트인 kubectl을 통해 명령어를 실행하면 API 노출된 엔드포인트에 RESTful API 호출을 수행하게 됩니다. API 서버는 API 처리 절차에서 인증, 권한 부여, 승인 제어를 보장합니다. 

- etcd: [etcd](https://etcd.io/)는 쿠버네티스 클러스터와 관련된 모든 데이터를 백업하는데 사용되는 key-value 형태의 저장소입니다. 클러스터 데이터는 노드나 전체 클러스터가 재시작 되더라도 재구성될 수 있도록 지속적으로 유지됩니다.

- kube-scheduler: 워커노드에 아직 할당(Binding)되지 않은 새로운 Pod를 찾고, 각 Pod를 가장 적절한 워커 노드에 할당하는 백그라운드 프로세스입니다.

- kube-controller-manager: 클러스터의 현재 상태를 모니터링하고, 사용자가 정의한 원하는 상태(Desired state)와 일치하도록 변경 사항을 적용합니다. 예를 들어, 기존 쿠버네티스 객체(Pod, ReplicaSet, Deployment 등)의 구성이 변경되면 Controller manager가 이를 감지하고 전환을 시도합니다. 

- cloud-controller-manager: Cloud Provider와 통합

### Worker Node
워커 노드는 애플리케이션 워크로드의 실행을 책입집니다. Control Plane의 명령을 받아 컨테이너를 시작·중지하고, 상태를 보고하며, 네트워크 트래픽을 적절한 Pod로 전달합니다. 클러스터의 워커 노드 세트를 데이터 플레인이라고 합니다. 각 워커 노드에는 공통적으로 아래의 구성 요소가 사용됩니다. 

- kubelet: kubelet은 각 노드에서 실행되는 에이전트로, Pod 내에서 필요한 컨테이너가 실행되고 상태를 유지하도록 보장합니다. 쿠버네티스와 컨테이너 런타임 엔진 사이의 접착제 역할을 하며 컨테이너가 실행되고 정상 상태를 유지하도록 보장한다고 할 수 있습니다. 컨트롤 플레인에서도 실행은 가능하지만, 컨트롤 플레인은 워크로드 처리가 주 목적이 아니므로 일반적으로 워커 노드에서 주로 실행됩니다.

- kube-proxy(선택 사항): 각 워커 노드에서 실행되는 네트워크 프록시입니다. 네트워크 규칙을 유지하고, 클러스터 내부 또는 외부에서 들어온 트래픽이 적절한 Pod로 라우팅될 수 있습니다.

- Container runtime: 컨테이너 관리를 책임지는 소프트웨어입니다. kubelet은 설정에 따라 다양한 컨테이너 런타임 엔진(예: containerd, CRI-O 등) 중에서 선택하도록 구성할 수 있습니다. 


## Amazon EKS
Amazon EKS는 AWS 클라우드와 온프레미스 환경에서 쿠버네티스를 실행하기 위한 AWS의 완전 관리형 서비스입니다. 핵심은 쿠버네티스의 컨트롤 플레인을 AWS가 직접 관리한다는 점입니다.

- **컨트롤 플레인(AWS 관리 범위)**: EKS에서 컨트롤 플레인은 AWS 백단에서 자체적으로 관리하는 VPC에서 실행되며, 컨트롤 플레인의 API 서버, etcd와 같은 핵심 컴포넌트들을 3개의 가용 영역에 걸쳐 최소 2개의 API 서버와 3개의 `etcd` 노드를 분산 배치합니다. 
- **데이터 플레인**: 워크로드가 배포되고 실행되는 환경으로, 사용자의 AWS 계정 내 VPC 또는 온프레미스(EKS Hybrid Nodes 사용 시)에 구성됩니다. 사용자는 목적에 맞게 데이터 플레인을 구성할 수 있습니다.
- **컨트롤 플레인과 데이터 플레인 간 통신**: 두 영역은 네트워크 상으로 완전히 분리 되어있으나, 컨트롤 플레인과 워커 노드가 통신할 때는 외부 인터넷 망을 거치지 않고 AWS가 생성하는 Cross-Account ENI를 통하여 사설망으로 통신하여 보안을 유지합니다.

### 도전과제1
> `도전과제1`: aws eks vs vanilla k8s 간 control plane , data plane 비교 정리 해보기

#### Control Plane

- Vanilla K8s는 자체 서버에 컨트롤 플레인을 직접 설치 및 구성하는 방식입니다. 마스터 노드에 SSH 접속이 가능하여 컴포넌트를 직접 프로세스 단위로 모니터링할 수 있으며 /etc/kubernetes/manifests, etcd 데이터 경로, 로그 파일 등을 직접 확인하고 편집할 수 있습니다.
- EKS는 AWS가 컨트롤 플레인을 관리합니다. 사용자는 AWS가 제공하는 API 서버 엔드포인트(Private/Public DNS)를 kubectl로 접근하는 것만 가능합니다. 

| 구성 요소 | Vanilla K8S | Amazon EKS |
|---|---|---|
| 고가용성(HA) 구성 | 사용자가가 직접/설계 구축 | AWS가 최소 2개의 API Server 관리 | 
| etcd 저장소 관리 | 사용자가 직접 백업/복구/테스트/모니터링 진행 | AWS가 3개의 etcd 관리 |
| 인증서 라이프사이클 관리 | 인증서 수동 발급 | 자동 발급 및 로테이션/재배포 |
<!-- | Add-on 및 네트워크/스토리지 플러그인 관리 |  |  |
| 무중단 버전 업그레이드 | 사용자가 롤백 전략 |  |


EKS는 AWS가 컨트롤 플레인(API 서버, etcd, scheduler, controller-manager)를 멀티 AZ에서 자동 HA로 운영하며, etcd 백업, 인증서 로테이션, 패치, 업그레이드를 처리합니다. 

Vanilla Kubernetes에서는 이러한 컴포넌트를 사용자가 직접 EC2 등에 설치하고 관리해야 하며, etcd 쿼럼 설계나 컨트롤 플레인 장애 복구가 추가 작업입니다. -->

<!-- 
#### Data Plane

데이터 플레인은 워커 노드가


데이터 플레인은 EKS에서 사용자 VPC 내 옵션(Self-managed EC2, Managed Node Group, Fargate, Auto Mode)을 선택할 수 있으며, 노드 프로비저닝/ASG/패치 일부를 AWS가 지원합니다.

Vanilla에서는 모든 워커 노드를 직접 프로비저닝하고 kubelet/kube-proxy를 설치하며, Cluster Autoscaler 등을 별도 구성해야 합니다. -->


## EKS Hands-on

EKS 클러스터는 다음과 같은 방법으로 프로비저닝 및 관리할 수 있습니다.

Hands-on은 제공된 Terraform 실습코드를 통해 배포하였습니다.

| 프로비저닝 방법 | 설명 |
|---|---|
| AWS Management Console | AWS 콘솔에서 GUI 기반으로 EKS 클러스터를 생성합니다. 직접 버튼을 클릭하고 값을 입력하여 클러스터를 생성하는 방식으로, 코드로 상태를 관리하거나 저장하지 않고 Live objects를 직접 조작합니다. 과정을 코드로 추적할 수 없어 프로덕션으로 적합하지 않으나 러닝 커브가 낮아 학습 용도로 접근하기 좋습니다. |
| eksctl | `eksctl create cluster --name my-cluster`와 같은 CLI 명령어를 통해 EKS 클러스터를 생성합니다. 사용자는 명령형으로 다루지만, eksctl 내부적으로는 CloudFormation으로 변환하여 배포를 수행하며 실제로 CloudFormation 스택이 생성되거나 업데이트 됩니다. |
| IaC(예: Terraform, Pulumi, CloudFormation) | `.tf`와 같은 IaC 도구 관련 파일들이 모인 디렉토리에 인프라를 정의해두고, 시스템 엔진이 리소스 블록을 읽고 조합하여 목표 상태와 일치하도록 인프라를 자동 배포하는 방식입니다. 인프라를 코드로 관리하므로 버전 관리와 협업이 용이하여 프로덕션 환경에 가장 적합합니다. |

### 로컬 환경 설정
macOS 사용자는 터미널, Windows OS 사용자는 WSL 터미널을 사용하여 로컬 PC에 Hands-on 환경을 설정합니다. 

1) AWS CLI 설치
```Zsh
# Install aws cli
brew install awscli
aws --version

# iam (주체) 자격 증명 설정
aws configure
AWS Access Key ID : <액세스 키 입력>
AWS Secret Access Key : <시크릿 키 입력>
Default region name : ap-northeast-2

# 확인
aws sts get-caller-identity
```

2) IAM User 생성 및 자격증명 설정(`aws configure`), EC2 Key Pair 생성

3) K8s 관리 도구 설치(kubectl, helm) 설치
```Zsh
# Install kubectl
brew install kubernetes-cli
kubectl version --client=true

# Install Helm
brew install helm
helm version
```

4) K8s 관리에 유용한 툴(krew, k9s, kube-ps1, kubectx) 설치(권장)
```Zsh
# Install krew
brew install krew

# Install k9s
brew install k9s

# Install kube-ps1
brew install kube-ps1

# Install kubectx
brew install kubectx

# kubectl 출력 시 하이라이트 처리
brew install kubecolor
echo "alias k=kubectl" >> ~/.zshrc
echo "alias kubectl=kubecolor" >> ~/.zshrc
echo "compdef kubecolor=kubectl" >> ~/.zshrc

# k8s krew path : ~/.zshrc 아래 추가
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

5) 여러 버전의 Terraform을 설치하고 쉽게 관리할 수 있도록 tfenv 설치

```Zsh
# tfenv 설치
brew install tfenv

# 설치 가능 버전 리스트 확인
tfenv list-remote

# 테라폼 특정 버전 설치
tfenv install 1.14.6

# 테라폼 특정 버전 사용 설정 
tfenv use 1.14.6

# tfenv로 설치한 버전 확인
tfenv list

# 테라폼 버전 정보 확인
terraform version

# 자동완성
terraform -install-autocomplete

## 참고 .zshrc 에 아래 추가됨
cat ~/.zshrc
autoload -U +X bashcompinit && bashcompinit
complete -o nospace -C /usr/local/bin/terraform terraform
```

### Terraform 실습 코드 배포
1) Github 저장소에서 실습 코드 다운로드 

```Zsh
# 코드 다운로드
git clone https://github.com/gasida/aews.git
cd aews
tree aews

# 작업 디렉터리 이동
cd 1w
```

2) Terraform 배포 및 K8s 자격증명 설정

```Zsh
# 변수 지정
aws ec2 describe-key-pairs --query "KeyPairs[].KeyName" --output text
export TF_VAR_KeyName=$(aws ec2 describe-key-pairs --query "KeyPairs[].KeyName" --output text)
export TF_VAR_ssh_access_cidr=$(curl -s ipinfo.io/ip)/32
echo $TF_VAR_KeyName $TF_VAR_ssh_access_cidr

# 배포 : 12분 정도 소요
terraform init
terraform plan
nohup sh -c "terraform apply -auto-approve" > create.log 2>&1 &
tail -f create.log


# 자격증명 설정
aws eks update-kubeconfig --region ap-northeast-2 --name myeks

# k8s config 확인 및 rename context
cat ~/.kube/config
cat ~/.kube/config | grep current-context | awk '{print $2}'
kubectl config rename-context $(cat ~/.kube/config | grep current-context | awk '{print $2}') myeks
cat ~/.kube/config | grep current-context
```

3) 테라폼 모듈 버전 정보 확인
테라폼에서 레지스트리의 모듈을 가져올 때는 코드 호환성을 유하기 위해 버전 제약 조건([Version Constraints](https://developer.hashicorp.com/terraform/language/expressions/version-constraints))을 설정할 수 있습니다. 

| 연산자 | 설명 |
|---|---|
| `=` | 특정한 버전을 지정합니다. <br/> `=` 기호를 쓰거나 기호를 생략하여 표기합니다. <br/> - version = "= 1.2.3"  <br/> - version = "1.2.3" |
| 부등호(`>`, `>=`, `<`, `<=`) | 특정 버전 이상이거나 미만인 버전을 허용합니다. |
| `~>` | 지정된 버전에서 가장 오른쪽에 있는 자리의 버전 업데이트만 허용합니다. <br/>- version = "~> 1.2": 1.2.0 이상, 2.0.0 미만의 최신 버전을 설치합니다. <br/>- version = "~> 1.2.3": 1.2.3 이상, 1.3.0 미만의 최신 버전을 설치합니다. |
| 조건 결합(`,` | 여러 조건을 쉼표로 결합하여 더 세밀한 범위를 지정합니다. <br/> - version = ">= 1.2.0, != 1.2.5": 1.2.0 이상을 허용하되, 버그가 있는 특정 버전(1.2.5)만 제외합니다. | 

예제 코드에는 `~>` 조건이 설정되어 있습니다.

```Zsh
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~>6.5"

module "eks" {
  
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 21.0" 
```

`terraform init` 명령어를 실행한 후, 실제로 다운로드 및 적용된 모듈의 정확한 버전은 .terraform/modules/modules.json 파일을 통해 다음과 같이 확인할 수 있습니다.

vpc는 6.5 이상의 최신 버전인 6.6.0이, eks는 21.0 이상의 최신 버전인 21.15.1 버전이 설치된 것을 확인하였습니다.

```Zsh
cat .terraform/modules/modules.json | jq
{
  "Modules": [
    {
      "Key": "vpc",
      "Source": "registry.terraform.io/terraform-aws-modules/vpc/aws",
      "Version": "6.6.0",
      "Dir": ".terraform/modules/vpc"
    },
    {
      "Key": "eks",
      "Source": "registry.terraform.io/terraform-aws-modules/eks/aws",
      "Version": "21.15.1",
      "Dir": ".terraform/modules/eks"
    }
  ]
}
```
### EKS 클러스터 정보 확인 

#### 컨트롤 플레인
1) EKS 클러스터 정보 확인
```Zsh
kubectl cluster-info
Kubernetes control plane is running at https://F06234AE0397E05608EB21DAED43151D.gr7.ap-southeast-1.eks.amazonaws.com
CoreDNS is running at https://F06234AE0397E05608EB21DAED43151D.gr7.ap-southeast-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

2) Endpoint Access 설정 확인

**Public Cluster Endpoint만 활성화, IP 접근 제한 미적용**

```Zsh
aws eks describe-cluster --name $CLUSTER_NAME | jq .cluster.resourcesVpcConfig
{
  "endpointPublicAccess": true,
  "endpointPrivateAccess": false,
  "publicAccessCidrs": [
    "0.0.0.0/0"
  ]
}
```
**API 서버 Endpoint 조회**
```Zsh
aws eks describe-cluster --name $CLUSTER_NAME | jq -r .cluster.endpoint
https://F06234AE0397E05608EB21DAED43151D.gr7.ap-southeast-1.eks.amazonaws.com
```

3) API 서버 Endpoint의 DNS 질의
```Zsh
APIDNS=$(aws eks describe-cluster --name $CLUSTER_NAME | jq -r .cluster.endpoint | cut -d '/' -f 3)
dig +short $APIDNS

18.136.245.227
18.141.20.137
```

```Zsh
curl -s ipinfo.io/18.136.245.227
{
  "ip": "18.136.245.227",
  "hostname": "ec2-18-136-245-227.ap-southeast-1.compute.amazonaws.com",
  "city": "Singapore",
  "region": "Singapore",
  "country": "SG",
  "loc": "1.2897,103.8501",
  "org": "AS16509 Amazon.com, Inc.",
  "postal": "018989",
  "timezone": "Asia/Singapore",
  "readme": "https://ipinfo.io/missingauth"
}
```

4) eks 노드 그룹 정보 확인
```Zsh
aws eks describe-nodegroup --cluster-name $CLUSTER_NAME --nodegroup-name $CLUSTER_NAME-node-group | jq
{
  "nodegroup": {
    "nodegroupName": "myeks-node-group",
    "nodegroupArn": "arn:aws:eks:ap-southeast-1:447659471488:nodegroup/myeks/myeks-node-group/d4ce7f6d-1dee-e4f0-5188-ce82c3754bac",
    "clusterName": "myeks",
    "version": "1.34",
    "releaseVersion": "1.34.4-20260304",
    "createdAt": "2026-03-18T11:50:44.386000+09:00",
    "modifiedAt": "2026-03-19T02:51:21.461000+09:00",
    "status": "ACTIVE",
    "capacityType": "ON_DEMAND",
    "scalingConfig": {
      "minSize": 1,
      "maxSize": 4,
      "desiredSize": 2
    },
    "instanceTypes": [
      "t3.medium"
    ],
    "subnets": [
      "subnet-0378f8589255a2d7f",
      "subnet-0df35014dbd94b704",
      "subnet-0b88a18c994f9aba6"
    ],
    "amiType": "AL2023_x86_64_STANDARD",
    "nodeRole": "arn:aws:iam::447659471488:role/myeks-node-group-eks-node-group-20260318024325206100000005",
    "labels": {},
    "resources": {
      "autoScalingGroups": [
        {
          "name": "eks-myeks-node-group-d4ce7f6d-1dee-e4f0-5188-ce82c3754bac"
        }
      ]
    },
    "health": {
      "issues": []
    },
    "updateConfig": {
      "maxUnavailablePercentage": 33
    },
    "launchTemplate": {
      "name": "default-20260318025034909900000008",
      "version": "1",
      "id": "lt-0c5106b59b7b4875e"
    },
    "tags": {
      "Terraform": "true",
      "Environment": "cloudneta-lab",
      "Name": "myeks-node-group"
    }
  }
}
```

5) 노드 상세 정보 확인
```Zsh
kubectl get node --label-columns=node.kubernetes.io/instance-type,eks.amazonaws.com/capacityType,topology.kubernetes.io/zone
kubectl get node --label-columns=node.kubernetes.io/instance-type
kubectl get node --label-columns=eks.amazonaws.com/capacityType # 노드의 capacityType 확인
kubectl get node
kubectl get node -owide
NAME                                               STATUS   ROLES    AGE   VERSION               INTERNAL-IP     EXTERNAL-IP      OS-IMAGE                        KERNEL-VERSION                   CONTAINER-RUNTIME
ip-192-168-1-116.ap-southeast-1.compute.internal   Ready    <none>   15h   v1.34.4-eks-f69f56f   192.168.1.116   52.221.196.142   Amazon Linux 2023.10.20260216   6.12.68-92.122.amzn2023.x86_64   containerd://2.1.5
ip-192-168-3-89.ap-southeast-1.compute.internal    Ready    <none>   15h   v1.34.4-eks-f69f56f   192.168.3.89    18.141.156.72    Amazon Linux 2023.10.20260216   6.12.68-92.122.amzn2023.x86_64   containerd://2.1.5
```
<!-- 6) 노드 정보 확인

7) 인증 정보 확인 -->

<!-- 

## EKS Cluster Endpoint Access
EKS의 API 서버는 사용자가 관리하지 않고, AWS 관리형 VPC에 API 서버가 배치됩니다.
이 클러스터의 API 엔드포인트를 어떻게 구성할지 선택하는 항목입니다. 

### Public Cluster Endpoint
- API 서버가 공인 IP로 노출됩니다.
- 워커 노드(kubelet)와 외부 관리자(kubectl) 모두 인터넷을 통해 API 서버와 통신합니다.
- API 서버가 퍼블릭으로 열려있어 DDoS와 같은 보안 위협에 노출될 수 있습니다.

### Public & Private Endpoint
- 워커 노드는 Public IP가 아닌 EKS-owned ENI를 통해 Private IP로 API 서버와 내부 통신을 합니다.
- 외부 관리자와 CI/CD 도구는 Public IP를 통해 API 서버에 접근합니다.
- 보안을 위해 특정 IP에서만 접근할 수 있도록 통제해야 합니다.

### Fully Private Endpoint
 

## 도전과제2
>`도전과제2`: EKS Fully Private Cluster 환경에서 nginx 디플로이먼트를 배포하고, 외부 노출 설정을 해보자

Fully Private Cluster(공개 엔드포인트 비활성화, Private만 활성화)에서는 클러스터 API가 VPC 내부에서만 접근 가능하며, Nginx를 배포한 후 외부 노출을 위해 ALB Ingress Controller나 NLB를 사용합니다. VPC 내 bastion/EC2나 VPN(예: Client VPN)으로 접근 후  kubectl apply 로 배포하세요.
	외부 노출 (ALB Ingress 추천):
	•	AWS Load Balancer Controller 설치:  helm install aws-load-balancer-controller eks/aws-load-balancer-controller ... 
	•	Ingress 리소스:

  접근: ALB DNS 확인 ( kubectl get ingress ), 브라우저로 테스트. Private LB 필요 시  internal  scheme 사용.


## 도전과제3
>`도전과제3`: EKS 보안 그룹 : 각 보안 그룹이 어떻게 적용되는지 정리해보기!

### Cluster Security Group

### Node Security Group -->