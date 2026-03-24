---
title: AEWS 2주차 - EKS Networking
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

## 학습 참고 자료
- EKS Best Practice Docs: [EN](https://docs.aws.amazon.com/eks/latest/best-practices/introduction.html), [KR](https://aws.github.io/aws-eks-best-practices/ko/networking/index/)
- EKS 네트워킹 소개 영상[Link](https://youtu.be/E49Q3y9wsUo?si=wH7fGLTdRs2rPxKl&t=2521)

## 실습 환경 배포

## AWS VPC CNI 소개

## 노드에서 기본 네트워크 정보 확인

## 노드 간 파드 통신

## 파드에서 외부 통신

## AWS VPC CNI 설정 변경

## 노드에 파드 생성 갯수 제한

## Service & Amazon EKS 네트워킹 지원

## AWS LoadBalancer Controller (LBC) & Service(L4)

## Ingress (L7:HTTP)

## ExternalDNS

## Gateway API

## CoreDNS

### 2주차 과제

- **2주차 스터디 내용**을 정리하거나 혹은 **AWS EKS 네트워크 관련 내용**을 **정리**해서 작성해주기 바랍니다 🙇🏻‍♂️🙇🏻‍♀️

**도전 해보세요!**

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