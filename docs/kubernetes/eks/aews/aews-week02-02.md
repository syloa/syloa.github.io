---
title: AEWS 2주차 - EKS Networking Hands-on
date: 2026-03-21
draft: true
categories:
  - Kubernetes
  - AWS
  - EKS
  - AEWS
tags:
  - AEWS
---
> *CloudNet 팀의 [2026년 AWS EKS Workshop Study 4기](https://gasidaseo.notion.site/26-AWS-EKS-Hands-on-Study-4-31a50aec5edf804b8294d8d512c43370) 2주차 학습 내용을 담고 있습니다.*

## 실습 환경 구성
2주차 실습 자료를 다운로드 받아 실습 환경을 구성하였습니다. <br/>
금주 Pod의 IP 할당 실습이 진행될 예정으로, Terraform 코드 내 서브넷 머스크가 22로 변경되어 있습니다. 


```Bash
# 코드 다운로드, 작업 디렉터리 이동
git clone https://github.com/gasida/aews.git
cd aews/2w

# 변수 지정
export TF_VAR_KeyName=$(aws ec2 describe-key-pairs --query "KeyPairs[].KeyName" --output text)
export TF_VAR_ssh_access_cidr=$(curl -s ipinfo.io/ip)/32
echo $TF_VAR_KeyName $TF_VAR_ssh_access_cidr

# 배포 : 12분 정도 소요

terraform init
terraform plan
nohup sh -c "terraform apply -auto-approve" > create.log 2>&1 &
tail -f create.log


# 자격증명 설정
terraform output -raw configure_kubectl
aws eks --region ap-southeast-1 update-kubeconfig --name myeks
kubectl config rename-context $(cat ~/.kube/config | grep current-context | awk '{print $2}') myeks
```

## EKS 기본 정보 확인
  
```Bash
# 클러스터 확인
kubectl cluster-info
Kubernetes control plane is running at https://F077EF3C6046F7E7513BA007BFA218F2.gr7.ap-southeast-1.eks.amazonaws.com
CoreDNS is running at https://F077EF3C6046F7E7513BA007BFA218F2.gr7.ap-southeast-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

eksctl get cluster
NAME    REGION          EKSCTL CREATED
myeks   ap-southeast-1  False

# 네임스페이스 default 변경 적용
kubens default
Context "arn:aws:eks:ap-southeast-1:{AccountID}:cluster/myeks" modified.
Active namespace is "default".

# 노드 정보 확인
kubectl get node --label-columns=node.kubernetes.io/instance-type,eks.amazonaws.com/capacityType,topology.kubernetes.io/zone
NAME                                               STATUS   ROLES    AGE   VERSION               INSTANCE-TYPE   CAPACITYTYPE   ZONE
ip-192-168-0-174.ap-southeast-1.compute.internal   Ready    <none>   22m   v1.34.4-eks-f69f56f   t3.medium       ON_DEMAND      ap-southeast-1a
ip-192-168-6-86.ap-southeast-1.compute.internal    Ready    <none>   22m   v1.34.4-eks-f69f56f   t3.medium       ON_DEMAND      ap-southeast-1b
ip-192-168-9-205.ap-southeast-1.compute.internal   Ready    <none>   22m   v1.34.4-eks-f69f56f   t3.medium       ON_DEMAND      ap-southeast-1c

kubectl get node -v=6
I0326 04:30:35.878065   77812 cmd.go:527] kubectl command headers turned on
I0326 04:30:35.938921   77812 loader.go:402] Config loaded from file:  /Users/mzc01-siyoung/.kube/config
I0326 04:30:35.943113   77812 envvar.go:172] "Feature gate default state" feature="ClientsAllowCBOR" enabled=false
I0326 04:30:35.943152   77812 envvar.go:172] "Feature gate default state" feature="ClientsPreferCBOR" enabled=false
I0326 04:30:35.943156   77812 envvar.go:172] "Feature gate default state" feature="InformerResourceVersion" enabled=false
I0326 04:30:35.943159   77812 envvar.go:172] "Feature gate default state" feature="InOrderInformers" enabled=true
I0326 04:30:35.943161   77812 envvar.go:172] "Feature gate default state" feature="WatchListClient" enabled=false
I0326 04:30:35.943164   77812 envvar.go:172] "Feature gate default state" feature="InOrderInformersBatchProcess" enabled=false
I0326 04:30:37.183790   77812 round_trippers.go:632] "Response" verb="GET" url="https://F077EF3C6046F7E7513BA007BFA218F2.gr7.ap-southeast-1.eks.amazonaws.com/api/v1/nodes?limit=500" status="200 OK" milliseconds=1210
NAME                                               STATUS   ROLES    AGE   VERSION
ip-192-168-0-174.ap-southeast-1.compute.internal   Ready    <none>   23m   v1.34.4-eks-f69f56f
ip-192-168-6-86.ap-southeast-1.compute.internal    Ready    <none>   23m   v1.34.4-eks-f69f56f
ip-192-168-9-205.ap-southeast-1.compute.internal   Ready    <none>   23m   v1.34.4-eks-f69f56f

# 노드 라벨 확인
# Terraform에서 설정된 'tier=primary' node label 확인 가능
kubectl get node --show-labels
kubectl get node -l tier=primary
ip-192-168-9-205.ap-southeast-1.compute.internal   Ready    <none>   24m   v1.34.4-eks-f69f56f   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=t3.medium,beta.kubernetes.io/os=linux,eks.amazonaws.com/capacityType=ON_DEMAND,eks.amazonaws.com/nodegroup-image=ami-07519ef34ac4a1ff4,eks.amazonaws.com/nodegroup=myeks-1nd-node-group,eks.amazonaws.com/sourceLaunchTemplateId=lt-0dd63f934ada196b4,eks.amazonaws.com/sourceLaunchTemplateVersion=1,failure-domain.beta.kubernetes.io/region=ap-southeast-1,failure-domain.beta.kubernetes.io/zone=ap-southeast-1c,k8s.io/cloud-provider-aws=5553ae84a0d29114870f67bbabd07d44,kubernetes.io/arch=amd64,kubernetes.io/hostname=ip-192-168-9-205.ap-southeast-1.compute.internal,kubernetes.io/os=linux,node.kubernetes.io/instance-type=t3.medium,**tier=primary**,topology.k8s.aws/zone-id=apse1-az3,topology.kubernetes.io/region=ap-southeast-1,topology.kubernetes.io/zone=ap-southeast-1c

# 파드 정보 확인
kubectl get pod -A
kubectl get pdb -n kube-system
NAME      MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
coredns   N/A             1                 1                     82m

# 클러스터와 노드 그룹 이름을 지정하여 노드그룹 확인
aws eks describe-nodegroup --cluster-name myeks --nodegroup-name myeks-1nd-node-group | jq

# eks addon 확인
aws eks list-addons --cluster-name myeks | jq
eksctl get addon --cluster myeks 
NAME            VERSION                 STATUS  ISSUES  IAMROLE UPDATE AVAILABLE        CONFIGURATION VALUES NAMESPACE        POD IDENTITY ASSOCIATION ROLES
coredns         v1.13.2-eksbuild.3      ACTIVE  0                                                            kube-system
kube-proxy      v1.34.5-eksbuild.2      ACTIVE  0                                                            kube-system
vpc-cni         v1.21.1-eksbuild.5      ACTIVE  0                                       {"env":{"WARM_ENI_TARGET":"1"}}       kube-system
```

---

## 노드의 네트워크 정보 확인

```Bash
# EC2 ENI IP 확인
aws ec2 describe-instances --query "Reservations[*].Instances[*].{PublicIPAdd:PublicIpAddress,PrivateIPAdd:PrivateIpAddress,InstanceName:Tags[?Key=='Name']|[0].Value,Status:State.Name}" --filters Name=instance-state-name,Values=running --output table

------------------------------------------------------------------------
|                           DescribeInstances                          |
+-----------------------+----------------+------------------+----------+
|     InstanceName      | PrivateIPAdd   |   PublicIPAdd    | Status   |
+-----------------------+----------------+------------------+----------+
|  myeks-1nd-node-group |  192.168.9.205 |  13.229.157.220  |  running |
|  myeks-1nd-node-group |  192.168.0.174 |  52.77.251.226   |  running |
|  myeks-1nd-node-group |  192.168.6.86  |  13.228.25.57    |  running |
+-----------------------+----------------+------------------+----------+

# 각 노드의 Public IP를 일시적 환경변수로 적용
N1=13.229.157.220
N2=52.77.251.226
N3=13.228.25.57

# 워커 노드 SSH 접속
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh -o StrictHostKeyChecking=no ec2-user@$i hostname; echo; done


# 파드 상세 정보 확인
kubectl get daemonset aws-node --namespace kube-system -owide
kubectl describe daemonset aws-node --namespace kube-system

# kube-proxy config 확인 : 모드 iptables 사용
kubectl describe cm -n kube-system kube-proxy-config
...
mode: "iptables"
...

kubectl describe cm -n kube-system kube-proxy-config | grep iptables: -A5
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
# ipvs는 지원중단으로 nftables 사용 권장

# aws-node 데몬셋 env 확인
kubectl get ds aws-node -n kube-system -o json | jq '.spec.template.spec.containers[0].env'

# 노드 IP 확인
aws ec2 describe-instances --query "Reservations[*].Instances[*].{PublicIPAdd:PublicIpAddress,PrivateIPAdd:PrivateIpAddress,InstanceName:Tags[?Key=='Name']|[0].Value,Status:State.Name}" --filters Name=instance-state-name,Values=running --output table

# 파드 IP 확인
kubectl get pod -n kube-system -o=custom-columns=NAME:.metadata.name,IP:.status.podIP,STATUS:.status.phase

# 파드 이름 확인
kubectl get pod -A -o name

# 파드 갯수 확인
kubectl get pod -A -o name | wc -ls

# cni log 확인
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i tree /var/log/aws-routed-eni ; echo; done
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo cat /var/log/aws-routed-eni/plugin.log | jq ; echo; done
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo cat /var/log/aws-routed-eni/ipamd.log | jq ; echo; done

# 네트워크 정보 확인 : eniY는 pod network 네임스페이스와 veth pair
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo ip -br -c addr; echo; done
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo ip -c addr; echo; done
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo ip -c route; echo; done

ssh ec2-user@$N1 sudo iptables -t nat -S
ssh ec2-user@$N1 sudo iptables -t nat -L -n -v

```

### 워커 노드의 ENI 확인

- 실습용으로 생성한 t3.medium 타입의 경우 ENI는 3개, ENI마다 최대 6개의 IP를 가질 수 있습니다. 그 중 Secondary IP만 포드에 할당이 가능하므로 노드 하나 당 최대 15개까지의 포드 생성이 가능합니다.

![](images/2026-03-26-04-51-38.png)

워커 노드의 인스턴스 네트워크 정보에서 프라이빗 IP 2개와 Secondary 프라이빗 IP 10개가 확인됩니다. 

- 보조 IPv4 주소를 coredns 파드가 사용하는지 확인 ⇒ coredns 파드가 배치되지 않은 워커 노드에 ENI 갯수 확인!

```Bash
# coredns 파드 IP 정보 확인
# 3개 중 2개 노드에만 Coredns 배치, 배치되지 않은 워커 노드도 ENI 갯수 2개로 동일 (설정에 따라 미리 ENI 추가)
kubectl get pod -n kube-system -l k8s-app=kube-dns -owide
NAME                       READY   STATUS    RESTARTS   AGE   IP              NODE                                               NOMINATED NODE   READINESS GATES
coredns-7d7954875f-55tcc   1/1     Running   0          46m   192.168.11.99   ip-192-168-9-205.ap-southeast-1.compute.internal   <none>           <none>
coredns-7d7954875f-pv7rs   1/1     Running   0          49m   192.168.2.12    ip-192-168-0-174.ap-southeast-1.compute.internal   <none>           <none>

# 노드의 라우팅 정보 확인 
# EC2 네트워크 정보의 '보조 프라이빗 IPv4 주소'에 있는 IP가 확인됨
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo ip -c route; echo; done
>> node 52.77.251.226 <<
default via 192.168.0.1 dev ens5 proto dhcp src 192.168.0.174 metric 512 
192.168.0.0/22 dev ens5 proto kernel scope link src 192.168.0.174 metric 512 
192.168.0.1 dev ens5 proto dhcp scope link src 192.168.0.174 metric 512 
192.168.0.2 dev ens5 proto dhcp scope link src 192.168.0.174 metric 512 
192.168.2.12 dev eni4d2fe644677 scope link 
192.168.3.22 dev enia7cc107bbb9 scope link 

# IpamD debugging commands
# https://github.com/aws/amazon-vpc-cni-k8s/blob/master/docs/troubleshooting.md
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i curl -s http://localhost:61679/v1/enis | jq; echo; done
```

### Network-Multitool 디플로이먼트 생성 

Network-Multitool: https://github.com/Praqma/Network-MultiTool

```Bash
# [터미널1~3] 노드 모니터링
ssh ec2-user@$N1
watch -d "ip link | egrep 'ens|eni' ;echo;echo "[ROUTE TABLE]"; route -n | grep eni"

ssh ec2-user@$N2
watch -d "ip link | egrep 'ens|eni' ;echo;echo "[ROUTE TABLE]"; route -n | grep eni"

ssh ec2-user@$N3
watch -d "ip link | egrep 'ens|eni' ;echo;echo "[ROUTE TABLE]"; route -n | grep eni"

# Network-Multitool 디플로이먼트 생성
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netshoot-pod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: netshoot-pod
  template:
    metadata:
      labels:
        app: netshoot-pod
    spec:
      containers:
      - name: netshoot-pod
        image: praqma/network-multitool
        ports:
        - containerPort: 80
        - containerPort: 443
        env:
        - name: HTTP_PORT
          value: "80"
        - name: HTTPS_PORT
          value: "443"
      terminationGracePeriodSeconds: 0
EOF

# 파드 이름 변수 지정
PODNAME1=$(kubectl get pod -l app=netshoot-pod -o jsonpath='{.items[0].metadata.name}')
PODNAME2=$(kubectl get pod -l app=netshoot-pod -o jsonpath='{.items[1].metadata.name}')
PODNAME3=$(kubectl get pod -l app=netshoot-pod -o jsonpath='{.items[2].metadata.name}')
echo $PODNAME1 $PODNAME2 $PODNAME3

# 파드 확인
kubectl get pod -o wide
kubectl get pod -o=custom-columns=NAME:.metadata.name,IP:.status.podIP
NAME                           IP
netshoot-pod-64fbf7fb5-bpj4v   192.168.6.161
netshoot-pod-64fbf7fb5-qg4zl   192.168.3.22
netshoot-pod-64fbf7fb5-z7l4v   192.168.10.66

# 노드에 라우팅 정보 확인
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo ip -c route; echo; done

# 파드가 생성되면, 워커 노드에 eniY@ifN 추가되고 라우팅 테이블에도 정보가 추가된다
# 노드3에서 네트워크 인터페이스 정보 확인
ssh ec2-user@$N3
----------------
ip -br -c addr show
ip -c link
ip -c addr
ip route # 혹은 route -n
default via 192.168.4.1 dev ens5 proto dhcp src 192.168.6.86 metric 512 
192.168.0.2 via 192.168.4.1 dev ens5 proto dhcp src 192.168.6.86 metric 512 
192.168.4.0/22 dev ens5 proto kernel scope link src 192.168.6.86 metric 512 
192.168.4.1 dev ens5 proto dhcp scope link src 192.168.6.86 metric 512 
192.168.6.161 dev eniea85eb048a4 scope link 

# 네임스페이스 정보 출력 -t net(네트워크 타입)
sudo lsns -t net
        NS TYPE NPROCS   PID USER     NETNSID NSFS                                                COMMAND
4026531840 net     115     1 root  unassigned                                                     /usr/li
4026532207 net       3 14985 65535          0 /run/netns/cni-86002c17-e81e-c751-593a-a7ee7971bfd7 /pause

# 테스트용 파드 접속(exec) 후 Shell 실행
kubectl exec -it $PODNAME1 -- bash

# 아래부터는 pod-1 Shell 에서 실행 : 네트워크 정보 확인
----------------------------
ip -c addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
3: eth0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default 
    link/ether 2a:0c:7b:0f:9c:45 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.6.161/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::280c:7bff:fe0f:9c45/64 scope link 
       valid_lft forever preferred_lft forever

ip -c route
route -n
ping -c 1 <pod-2 IP>
ps
cat /etc/resolv.conf
----------------------------

# 파드2 Shell 실행
kubectl exec -it $PODNAME2 -- ip -c addr

# 파드3 Shell 실행
kubectl exec -it $PODNAME3 -- ip -br -c addr

```

### 포드 간 통신 테스트

```Bash
# 파드 IP 변수 지정
PODIP1=$(kubectl get pod -l app=netshoot-pod -o jsonpath='{.items[0].status.podIP}')
PODIP2=$(kubectl get pod -l app=netshoot-pod -o jsonpath='{.items[1].status.podIP}')
PODIP3=$(kubectl get pod -l app=netshoot-pod -o jsonpath='{.items[2].status.podIP}')
echo $PODIP1 $PODIP2 $PODIP3

# 파드1 Shell 에서 파드2로 ping 테스트
kubectl exec -it $PODNAME1 -- ping -c 2 $PODIP2
kubectl exec -it $PODNAME1 -- curl -s http://$PODIP2
kubectl exec -it $PODNAME1 -- curl -sk https://$PODIP2

# 파드2 Shell 에서 파드3로 ping 테스트
kubectl exec -it $PODNAME2 -- ping -c 2 $PODIP3

# 파드3 Shell 에서 파드1로 ping 테스트
kubectl exec -it $PODNAME3 -- ping -c 2 $PODIP1

# 3건 ping 테스트 모두 VPC 내부 통신이 진행됨

# 워커 노드 EC2 : TCPDUMP 확인
## For Pod to external (outside VPC) traffic, we will program iptables to SNAT using Primary IP address on the Primary ENI.
sudo tcpdump -i any -nn icmp
sudo tcpdump -i ens5 -nn icmp
sudo tcpdump -i ens6 -nn icmp
sudo tcpdump -i eniYYYYYYYY -nn icmp

[워커 노드1]
# routing policy database management 확인
ip rule

# routing table management 확인
ip route show table local
ip route show table main
ip route show table 2
```

---

### Pod에서 외부 통신 (NAT 포함)

VPC CNI에서 iptable의 SNAT를 통하여 노드의 ENI IP로 변경되어 외부와 통신합니다.
- 파드 shell 실행 후 외부로 ping 테스트 & 워커 노드에서 tcpdump 및 iptables 정보 확인
```Bash
# pod-1 Shell 에서 외부로 ping
kubectl exec -it $PODNAME1 -- ping -c 1 www.google.com
kubectl exec -it $PODNAME1 -- ping -i 0.1 www.google.com
kubectl exec -it $PODNAME1 -- ping -i 0.1 8.8.8.8

# 워커 노드 EC2 : TCPDUMP 확인
sudo tcpdump -i any -nn icmp
sudo tcpdump -i ens5 -nn icmp

# 퍼블릭IP 확인
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i curl -s ipinfo.io/ip; echo; echo; done

# 작업용 EC2 : pod-1 Shell 에서 외부 접속 확인 - 공인IP는 어떤 주소인가?
## The right way to check the weather - 링크
for i in $PODNAME1 $PODNAME2 $PODNAME3; do echo ">> Pod : $i <<"; kubectl exec -it $i -- curl -s ipinfo.io/ip; echo; echo; done
kubectl exec -it $PODNAME1 -- curl -s wttr.in/seoul
kubectl exec -it $PODNAME1 -- curl -s wttr.in/seoul?format=3
kubectl exec -it $PODNAME1 -- curl -s wttr.in/Moon
kubectl exec -it $PODNAME1 -- curl -s wttr.in/:help


# 워커 노드 EC2
## 출력된 결과를 보고 어떻게 빠져나가는지 고민해보자!
ip rule
ip route show table main
sudo iptables -L -n -v -t nat
sudo iptables -t nat -S

# 파드가 외부와 통신시에는 아래 처럼 'AWS-SNAT-CHAIN-0' 룰(rule)에 의해서 SNAT 되어서 외부와 통신!
# 참고로 뒤 IP는 eth0(ENI 첫번째)의 IP 주소이다
# --random-fully 동작 - 링크1  링크2
sudo iptables -t nat -S | grep 'A AWS-SNAT-CHAIN'
-A AWS-SNAT-CHAIN-0 ! -d 192.168.0.0/16 -m comment --comment "AWS SNAT CHAIN" -j RETURN
-A AWS-SNAT-CHAIN-0 ! -o vlan+ -m comment --comment "AWS, SNAT" -m addrtype ! --dst-type LOCAL -j SNAT --to-source 192.168.1.251 --random-fully

## 아래 'mark 0x4000/0x4000' 매칭되지 않아서 RETURN 됨!
-A KUBE-POSTROUTING -m mark ! --mark 0x4000/0x4000 -j RETURN
-A KUBE-POSTROUTING -j MARK --set-xmark 0x4000/0x0
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -j MASQUERADE --random-fully
...

# 카운트 확인 시 AWS-SNAT-CHAIN-0에 매칭되어, 목적지가 192.168.0.0/16 아니고 외부 빠져나갈때 SNAT 192.168.1.251(EC2 노드1 IP) 변경되어 나간다!
sudo iptables -t filter --zero; sudo iptables -t nat --zero; sudo iptables -t mangle --zero; sudo iptables -t raw --zero
watch -d 'sudo iptables -v --numeric --table nat --list AWS-SNAT-CHAIN-0; echo ; sudo iptables -v --numeric --table nat --list KUBE-POSTROUTING; echo ; sudo iptables -v --numeric --table nat --list POSTROUTING'

# conntrack 확인 : EC2 메타데이터 주소(169.254.169.254) 제외 출력
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo conntrack -L -n |grep -v '169.254.169'; echo; done
conntrack v1.4.5 (conntrack-tools): 
icmp     1 28 src=172.30.66.58 dst=8.8.8.8 type=8 code=0 id=34392 src=8.8.8.8 dst=172.30.85.242 type=0 code=0 id=50705 mark=128 use=1
tcp      6 23 TIME_WAIT src=172.30.66.58 dst=34.117.59.81 sport=58144 dport=80 src=34.117.59.81 dst=172.30.85.242 sport=80 dport=44768 [ASSURED] mark=128 use=1
```

---

### AWS VPC CNI 주요 설정값 튜닝

- WARM_ENI_TARGET 
여분의 ENI를 미리 붙여놓는 것, Pod의 IP를 조밀하게 관리하고 싶다면 0으로 관리 권장

- WARM_IP_TARGET

- MINIMUM_IP_TARGET

---

### 노드당 Pod 수 제한과 maxPods 파라미터

---

## AWS LoadBalancer Controller (LBC) & Service(L4)
1. 인스턴스 유형 : 노드에 NodePort로 전달
2. IP 유형 ⇒ 반드시 AWS LoadBalancer 컨트롤러 파드 및 정책 설정이 필요함!

- **AWS LBC with IRSA 설치** - [Docs](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html) , [Github](https://github.com/kubernetes-sigs/aws-load-balancer-controller) *→ 아래 8번 LBC 파드 그림 확인!*
    - AWS LBC(파드)가 AWS Service 를 이용하는 방법 : **방안1(IRSA)**, 방안2(Pod Identity), 방안3(EC2 Instance Profile - 비권장), 방안4(Static credentials - 절대 금지!) - [악분일상](https://malwareanalysis.tistory.com/723)*

---

## Ingress (L7:HTTP)

인그레스 소개 : 클러스터 내부의 서비스(ClusterIP, NodePort, Loadbalancer)를 외부로 노출(HTTP/HTTPS) - Web Proxy 역할
AWS Load Balancer Controller + Ingress (ALB) IP 모드 동작 with AWS VPC CNI

---

## ExternalDNS

쿠버네티스 서비스/인그레스/Gateway API 생성 시 도메인을 설정하면 클라우드 프로바이더의 DNS 서비스에 A 레코드(TXT)로 자동 생성/삭제

---

## Ingress(**ALB** + **HTTPS**) + 도메인 연동(ExternalDNS)

---

## Gateway API

---

## CoreDNS

쿠버네티스는 CoreDNS라는 DNS 서비스를 통해 모든 서비스를 이름으로 등록합니다. 내부적으로 CoreDNS는 서비스 이름을 호스트명으로 저장하고 이를 클러스터 IP 주소에 매핑합니다. 

IP 주소는 일시적이고 예측이 불가능한 반면, 레이블은 선언적이며 알려져있습니다.

정확한 서비스 검색을 확인하려면 동일한 네임스페이스에서 호스트명과 인바운드 포트를 사용하여 서비스에 호출을 수행하는 Pod를 실행해볼 수 있습니다.

동일 네임스페이스에서 호스트명과 수신 포트를 사용하여 서비스를 호출하는 Pod를 실행함으로써 올바른 서비스 검색을 확인할 수 있습니다. 

---

## 2주차 도전 과제 정리

- `도전과제` 두 번째 관리형 노드 추가 하여, **노드 부트스트랩 과정에 kubelet maxPods 110개 적용** 후 해당 노드에 **디플로이먼트 배포** by 테라폼 - [참고](https://kkamji.net/posts/eks-max-pod-limit/)
- `도전과제` Custom Networking 설정 및 동작 확인 해보기 by 테라폼 - [Docs](https://docs.aws.amazon.com/ko_kr/eks/latest/best-practices/custom-networking.html) , [Workshop](https://www.eksworkshop.com/docs/networking/vpc-cni/custom-networking/)
    - Automating custom networking to solve IPv4 exhaustion in Amazon EKS - [Link](https://aws.amazon.com/blogs/containers/automating-custom-networking-to-solve-ipv4-exhaustion-in-amazon-eks/)
- `도전과제` eks 모듈로 배포 시, 관리형 노드 그룹에 시작 템플릿에 `imds hop limit = 2` 적용 설정 해보기
- `도전과제` Amazon EKS 멀티 클러스터 로드밸런싱으로 고가용성 애플리케이션 구성하기 - [Link](https://aws.amazon.com/ko/blogs/tech/build-highly-available-application-with-amazon-eks-multi-cluster-loadbalancing/)
    - AWS LB Controller 에서 Service(NLB)와 Ingress(ALB)를 MultiCluster Target Groups 사용 - [Docs](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.11/guide/use_cases/multi_cluster/)
    - A deeper look at Ingress Sharing and Target Group Binding in AWS Load Balancer Controller - [링크](https://aws.amazon.com/blogs/containers/a-deeper-look-at-ingress-sharing-and-target-group-binding-in-aws-load-balancer-controller/)
- `도전과제` Service(NLB + **TLS**) + 도메인 연동(ExternalDNS)
- `도전과제` Ingress(**ALB** + **HTTPS**) + 도메인 연동(ExternalDNS)
- `도전과제` AWS LBC를 통해서 Gateway API 의 HTTPRoute 사용 시, 아래 기능들 적용 해보기 - [Docs](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/gateway/l7gateway/#examples)
    - Modifying Request Headers : 요청 헤더 조작
    - HTTP Header Matching : 헤더 매칭을 통한 백엔드 라우팅
    - Source IP Condition : 요청 소스 IP 통제
- `도전과제` AWS LBC를 통해서 Gateway API 의 TLSRoute 를 적용 해보기 - [Docs](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/gateway/l4gateway/)
- `도전과제` CoreDNS 모니터링 및 최적화 가이드 내용으로 실습 환경 구성 및 테스트 해보기 - [Github](https://devfloor9.github.io/engineering-playbook/docs/infrastructure-optimization/coredns-monitoring-optimization)
    
    ![image.png](attachment:2c7ebdeb-52a9-42dc-8c20-a824a3bdb156:image.png)
    
    ```bash
    .:53 {
      kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        **ttl 30**           # Service/POD 레코드 TTL
      }
    
      **cache 30 {         # 최대 30초 보존
        success 10000 30 # capacity 10k, maxTTL 30s
        denial 2000 10   # negative cache 2k, maxTTL 10s
        prefetch 5 60s   # 동일 질의 5회↑면 60s 전에 갱신
      }**
    
      forward . /etc/resolv.conf {
        **max_concurrent 2000**
        prefer_udp
      }
    
      prometheus :9153
      health {
        **lameduck 30s**
      }
      ready
      reload
      log
    }
    ```

---

## 참고 자료
- EKS Best Practice Docs: [EN](https://docs.aws.amazon.com/eks/latest/best-practices/introduction.html), [KR](https://aws.github.io/aws-eks-best-practices/ko/networking/index/)
- EKS 네트워킹 소개 영상[Link](https://youtu.be/E49Q3y9wsUo?si=wH7fGLTdRs2rPxKl&t=2521)