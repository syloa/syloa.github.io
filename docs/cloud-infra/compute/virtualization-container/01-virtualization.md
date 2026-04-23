---
title: 가상화 기초와 하이퍼바이저 유형
date: 2026-03-14
tags:
  - Virtualization & Container
---
> `git checkout 2022`

2022년에 작성했던 가상화 글을 바탕으로 개념을 다시 한번 환기하고 새롭게 글을 작성해보았습니다. 클라우드 인프라의 기본이 되는 가상화와, 이를 가능하게 하는 하이퍼바이저의 역할과 유형, 구현 방식에 대해 간략하게 짚어보는 글입니다.

## 가상화(Virtualization)
가상화는 CPU, 메모리, 스토리지, 네트워크 등의 물리적 자원을 추상화하여 여러 개의 독립적인 가상환경(VM/컨테이너)을 동시에 실행할 수 있게 해주는 기술입니다. 이를 통해 하드웨어 리소스 활용률을 극대화하고, 환경을 격리하여 안전성과 유연성을 확보할 수 있습니다.

이러한 가상화를 구성하고 관리하는 핵심 소프트웨어를 하이퍼바이저 또는 VMM(Virtual Machine Manager)이라고 합니다.

## 하이퍼바이저 타입
하이퍼바이저는 하드웨어와 운영체제 사이의 어느 위치에서 동작하는지에 따라 크게 두 가지로 나눌 수 있습니다. 
<!-- 와  I/O 경로 특성 -->
![HyperVisor Type](../../../images/hypervisor_type.png)

### Type 1 (Bare Metal/Native Hypervisors)
- Type 1 하이퍼바이저는 운영체제 없이 하이퍼바이저가 하드웨어 계층 위에서 직접 동작하며, 여러 게스트 운영체제와 그에 할당된 하드웨어 자원(가상 CPU, 가상 메모리, 가상 디스크, 가상 네트워크 등)을 관리합니다. 
- 게스트 OS가 하드웨어 자원을 요청할 때, 하이퍼바이저가 직접적으로 하드웨어에 명령어를 직접 전달하여 오버헤드가 적습니다. 
- 일반적으로 서버 가상화(데이터센터/클라우드)에서 표준적으로 사용됩니다.
- 대표 솔루션으로 VMWare ESXi, Microsoft Hyper-V, Liunx KVM, Citrix Hypervisor(구 XenServer, 젠서버) 등이 있습니다.

#### *KVM / Hyper-V는 왜 Type 1일까?
- 리눅스 기반의 KVM과 윈도우 기반의 Hyper-V는 하드웨어 위에서 하이퍼바이저가 직접 실행되는 Type 1에 속하지만 Host OS가 존재하여 각각 리눅스와 윈도우 위에 Guest OS가 동작하는 것처럼 보일 수 있습니다. 실제로는 하이퍼바이저가 하드웨어 자원을 직접 제어하는 구조이므로 Type 1으로 분류합니다.
- KVM은 Linux 커널의 가상화 기능을 활용해 커널 자체가 하이퍼바이저 역할을 수행합니다.
- Hyper-V는 기능 활성화 시 기존 Windows OS가 Parent Partition으로 동작하고, 다른 Guest OS들은 별도 Child Partition에서 실행되는 구조입니다.

### Type 2 (Embedded/Hosted Hypervisors):
- 구조: Hardware → Host OS → Hypervisor → Guest OS
- Windows/macOS/Linux 같은 일반 목적 Host OS 위에 애플리케이션 형태로 설치됩니다.
- 하드웨어 계층 위에 Host OS 계층이 있고, 그 위에 하이퍼바이저가 존재합니다. 하이퍼바이저 위에서 여러 Guest OS가 독립적으로 실행됩니다.
- 게스트 OS의 요청이 하이퍼바이저 → 호스트 OS 커널 → 하드웨어라는 복잡한 경로를 거쳐 오버헤드가 커질 수 있습니다.
- 개발/테스트/학습 용도에서 활용도가 높습니다.
- Oracle VirtualBox, VMware Workstation, VMware Fusion 등의 솔루션이 있습니다.


## 가상화 구현 방식
Guest OS를 어떻게 실행하고 I/O를 어떻게 최적화하는지 가상화 구현 방식에 따라 다릅니다. 실제 운영 환경에서는 이 방식들이 섞여 동작하는 경우가 많습니다.

### 호스트 가상화(Hosted Hypervisors)
- 구조: Hardware → Host OS → Hypervisor → Guest OS
- 일반 목적 OS 위에서 하이퍼바이저가 동작하는 방식으로, 위의 Type 2와 동일한 개념입니다. 개발용 VM이나 데스크톱 가상화에서 흔히 볼 수 있습니다.

### 전가상화 (Full Virtualization)
- 구조: Hardware → Hypervisor(Type 1) → Guest OS
- 하이퍼바이저가 하드웨어 전체를 완벽하게 추상화하여 Guest OS에 제공하는 방식입니다. Guest OS는 자신이 가상 환경에서 돌아가고 있다는 사실을 인지하지 못하며, 베어 메탈 서버에 설치되어있는 것처럼 동작합니다.
- 초기에는 하이퍼바이저가 Guest OS의 특권 명령을 처리하기 위해 이진 번역(Binary Translation)으로 처리해 성능 오버헤드가 발생할 수 있었습니다.
- 게스트 OS를 수정하지 않고도 실행 가능한 방식입니다.
- 현재는 하드웨어 지원 가상화(Intel VT-x/AMD-V)가 결합되어, 과거 대비 오버헤드가 크게 완화되었습니다.

### 반가상화 (Paravirtualization)
- 구조: Hardware → Hypervisor(Type 1) → Modified Guest OS
- 게스트 OS의 커널을 수정하여, 번역 과정 없이 '하이퍼콜(Hyper Call)'이라는 인터페이스를 통해 하이퍼바이저와 통신합니다. 게스트 OS는 자신이 가상환경에서 돌아가고 있다는 것을 인식하고 있으며, 하이퍼바이저와 소통하기 위한 수단인 하이퍼콜을 사용합니다.
- 전가상화와 다르게 하이퍼바이저가 명령을 가로채어 이진 번역할 필요 없이, Guest OS가 하이퍼콜(Hypercall)이라는 전용 API를 통해 하이퍼바이저에게 직접 자원 제어를 요청하므로 오버헤드가 크게 줄어듭니다.
- 반가상화를 지원하지 않는 운영 체제는 커널 수정이 필요합니다. 또는, Windows와 같이 소스코드가 공개되지 않은 특정 OS에는 반가상화 인식 드라이버(Paravirtualization-aware drivers, 또는 PV 드라이버)가 제공될 수 있습니다.
- 하이퍼바이저나 게스트 OS를 업데이트할 때는 호환성 이슈가 생길 수 있으므로 주의가 필요합니다.
- 대표적으로 오픈 소스 하이퍼바이저인 Citrix XEN server가 반가상화에 특화되어 있습니다.
- 실무 환경에서는 완전한 반가상화보다는 PV 드라이버(VirtIO 등)를 통해 디스크와 네트워크 I/O를 최적화하는 형태가 더 일반적입니다.


### 하드웨어 지원 가상화 (Hardware-assisted Virtualization)
- 구조: Hardware → Hypervisor → Guest OS
- 하드웨어 지원 가상화는 성능 저하를 일으키던 이진 번역의 필요성을 없애고, CPU 하드웨어 차원에서 가상화 전용 실행 공간을 제공하여 Guest OS의 명령이 하이퍼바이저 개입 없이 CPU에서 직접 실행되도록 돕는 방식입니다.
- CPU가 가상화 실행 모드와 메모리 가상화 기능을 제공해 하이퍼바이저 구현 부담을 줄이고 성능을 개선합니다.
- 하드웨어 지원 가상화를 지원하는 대표적인 하드웨어로 Intel VT, AMD-V, ARM virtualization extension이 있습니다.


## 사용 사례
- 테스트/학습 목적이면 접근성이 좋은 Type 2 하이퍼바이저로 빠르게 시작하고, 운영 환경 설계는 Type 1 기준으로 검토하는 것이 좋습니다.
- 현재 대부분의 클라우드의 IaaS 서비스는 **'Type 1 기반 + 전가상화 + 하드웨어 지원 가상화 + PV 드라이버 최적화'**가 혼합된 형태로 동작합니다.
- 운영 관점에서는 성능 외 격리 경계, 라이브 마이그레이션 가능 여부, 백업/복구 시간(RTO/RPO), 관측성까지 함께 고려해야 합니다.