---
title: Final Project, DevOps CI/CD Pipeline Automation
author: hailey
date: 2022-09-22 18:32:00 +0900
categories: [Solutions Architect Course, Project]
tags: [Project]
---


# 프로젝트 소개 
---

- 프로젝트 기간: Friday, 12 August 2022 - Thursday, 22 September 2022

- 프로젝트 제목: (Agile 개발 환경을 위한 DevOps CI/CD 파이프라인 자동화)

- 시나리오: 국내외로 이커머스 시장이 크게 성장하고 있고 이커머스 시장에서 빠른 확장과 효율적인 운영을 위해 클라우드가 도입되고 있습니다. 브로컬리 팀은 한 고객사로부터 클라우드 네이티브 온라인 쇼핑몰 구축 의뢰를 받습니다.

✔️What I Did:

- CI/CD Pipeline 구성도:

![CI/CD Pipeline 구성도]({{"/posts/2022-09-22-01.png"| relative_url}})*CI/CD Pipeline 구성도*  

- 요구사항:
    1. 저희 웹 개발 팀은 MSA로 서비스를 개발 중이에요. 소스코드는 Github Enterprise로 관리하고 있는데, 익숙한 것을 계속 쓰고 싶어요.
    2. 확장성을 생각했을 때 컨테이너 워크로드로 쿠버네티스를 쓰고싶어요. 하지만 저희 팀에는 쿠버네티스를 잘 모르는 팀원도 있습니다.
    3. 적은 인원으로 운영과 개발 업무를 수행해야 합니다. 개발/테스트/스테이징/프로덕션 서버 모두 운영하는 것은 너무 번거로워요! 하지만 고객이 에러를 경험하지 않았으면 좋겠어요.
    4. 새로운 요청이 들어왔을 때, 변경 사항을 빠르게 적용해야 해요.  인프라와 CI/CD 구축을 도와주세요.
- 사용 스택:
    - Github
    - Github Actions
    - ECR
    - Kustomize
    - ArgoCD
    - Argo Rollouts
    - EKS (ExternalDNS, Ingress)


---

- 구성도: 

![최종 구성도]({{"/posts/2022-09-22-04.png"| relative_url}})*최종 구성도*  

- 구성도 초안:

![구성도 초안]({{"/posts/2022-09-22-02.png"| relative_url}})*구성도 초안*  
