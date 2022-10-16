---
title: Oracle VM VirtualBox 실습
author: hailey
date: 2022-04-01 18:32:00 -0500
categories: [솔루션즈아키텍트양성과정, 온프레미스 인프라스트럭쳐]
tags: [VirtualBox]
---


Oracle사의 VirtualBox를 이용하여 VM 만들어보기


미러사이트: https://mirror.kakao.com/

미러사이트에서 CentOS7Minimal*.iso, CentOS7DVD*.iso를 다운받는다.




QuestVM1(GUI) 사양(CentOS7Desktop) → 게스트VM은 호스트VM보다 사양이 좋을 수 없음.

CPU: 2Core(vCPU- Hyperthreading)
RAM: 4G, Shared Memory: 128MB VGA0
SDD: 32G
NETWORK: NAT (Network Address Translation)
IMG: CentOS7DVD*.iso

QuestVM2(CLI) 사양(CentOS7Minimal) → 게스트VM은 호스트VM보다 사양이 좋을 수 없음.
CPU: 2C
RAM: 4G
SDD: 100G
NETWORK: NAT
IMG: CentOS7Minimal*.iso

동적 할당: 내가 가진 디스크의 한도 넘을 수 있다. 성능은 고정 크기보다 떨어지는 편

고정 크기: 호스트OS 디스크의 용량을 넘을 수 없다.