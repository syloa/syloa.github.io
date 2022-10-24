---
title: 1 April 2022, VirtualBox
author: hailey
date: 2022-04-01 12:32:00 +0900
categories: [Solutions Architect Course, On-premise Infrastructure]
tags: [VirtualBox]
---


### 인터넷 속도 확인
---
<https://fast.com>

# VirtualBox 설치
---

설치 링크: <https://www.virtualbox.org/wiki/Downloads>

![Host Key Combination](/posts/2022-04-01-08.png)  
호스트 키 조합의 단축키를 Ctrl + Alt로 바꾸어준다.  
가상머신에서 호스트 키를 누르면 호스트 마우스 포인터와 키보드를 잡아 가상머신 밖으로 나오게 된다. 

VirtualBox를 설치하고 나서 HostOS의 네트워크를 확인하면 물리적 네트워크 인프라 위의 가상 레이어인 overlay 네트워크가 생성되어있다.

# VirtualBox 실습
---

Oracle사의 VirtualBox를 이용하여 VM 생성 실습

서울에서 가장 가까운 위치의 미러사이트: <https://mirror.kakao.com/>

먼저 미러사이트에서 `CentOS7Minimal*.iso`, `CentOS7DVD*.iso`를 다운받는다.

## 가상 머신 생성
--- 

![Virtual Machine](/posts/2022-04-01-01.png)

- Oricle VirtualBox에 CentOS를 설치. 버전은 Red Hat으로 선택해준다.


## 가상 머신 사양
--- 

**QuestVM1(GUI) 사양(CentOS7Desktop)**  
CPU: 2Core(vCPU- Hyperthreading)  
RAM: 4G, Shared Memory: 128MB VGA0  
SDD: 32G  
NETWORK: NAT (Network Address Translation)  
IMG: CentOS7DVD*.iso  

**QuestVM2(CLI) 사양(CentOS7Minimal)**  
CPU: 2C  
RAM: 4G  
SDD: 100G  
NETWORK: NAT  
IMG: CentOS7Minimal*.iso  


→ 게스트VM은 호스트VM보다 사양이 좋을 수 없다. 유의해서 CPU와 RAM을 설정해준다.  
→ GUI는 CLI보다 iso 파일의 용량이 크다.  
→ 가상화는 자원 소모하므로 사용하지 않으면 꺼도 된다.
→ VM은 D드라이브에 저장해도 된다.

## 가상 하드 디스크 생성
--- 
Motherboard == Mainboard

![Computer Hard Drive](/posts/2022-04-01-05.png)

Hard disk: 중간에 동적으로 돌아가는 모터 있고, 쓸수록 성능 저하

SSD: 메모리 방식, 성능 저하 X

- VDI: VirtualBox 디스크 이미지
- VHD: MS의 HyperV
- VMDK: VMWare에서 쓰는 가상머신 디스크(가상머신의 하드 디스크)


## 동적 할당과 고정 크기
---

![Virtual Hard disk](/posts/2022-04-01-02.png)

동적 할당: 내가 가진 디스크의 한도 넘을 수 있다. 성능은 고정 크기보다 떨어지는 편이다.

고정 크기: 호스트OS 디스크의 용량을 넘을 수 없다.

![동적 할당 하드 디스크(dynamic disk)]({{"/posts/2022-04-01-03.png"| relative_url}})*동적 할당 하드 디스크(dynamic disk)*

2TB로 만들었지만 파일크기가 9MB

OS 설치가 되면 파일이 점점 커짐. 하지만 2TB까지는 커지지 못한다. 호스트 OS 디스크의 용량 넘으면 에러 뜸.

![고정 크기 하드 디스크(fixed disk)]({{"/posts/2022-04-01-04.png"| relative_url}})*고정 크기 하드 디스크(fixed disk)*

고정크기 vdi의 파일 크기가 11GB임. 

고정 크기 디스크의 용량 일부만 사용하면 나머지는 다른데서 못씀.

비즈니스에서는 퍼포먼스 때문에 고정크기를 쓴다. (매번 CPU 연산해서 공간할당을 하지 않음.)



## 가상머신 설정
---

![Display](/posts/2022-04-01-06.png)

디스플레이는 기본적인 시스템 메모리에서 떼서 Shard Memory를 사용한다.  
RAM: 4G, Shared Memory: 128MB VGA0  
여기선 4GB - 128MB한 것이 시스템 메모리가 된다.

![Storage](/posts/2022-04-01-07.png) 

컨트롤러 IDE는 호환성을 위해 존재한다. 호환이 될 경우에는 SATA 컨트롤러만 사용해도 무관하다.

![USB](/posts/2022-04-01-09.png)

Virtual Machine에서 USB는 요즘 의미가 없다.

USB3.0버전을 이용하기 위해선 VirtualBox의 extension pack 필요하다. 
여기선 1.1이라 성능이 좋지 않다.

