---
title: "Development Environment"
hideSummary: true
weight: 1
---

이 프로젝트는 처음에는 편의성 때문에 Windows의 WSL2 Ubuntu 환경에서 시작했지만, 현재 프로젝트 자체는 프로그램 자체를 개발하는 것 뿐만 아니라 개발도구들을 활용하면서 Blocking TCP server에서부터 시작해 Thread per connection, Thread pool, 그리고 Epoll로 구조를 개선하게 되면서 성능차이를 측정하고, 병목을 확인하고 개선하는것까지 포함하기 때문에 Linux환경으로 넘어와 작업하게 되었다. 이글은 이와 관련해서 WSL과 리눅스환경 자체의 차이점 그리고 가상 머신의 구조에 대해서 정리하고자 한다.


### WSL1과 WSL2

WSL1과 WSL2는 Linux 프로그램을 Windows에서 실행한다는 점은 같지만 내부 구조가 다르다. WSL1에는 실제 Linux Kernel이 존재하지 않고, Linux 프로그램이 요청한 system call을 Windows에서 처리할 수 있도록 compatibility layer를 제공하는 방식이다. 반면에 WSL2에서는 경량 VM 안에서 실제 linux kernel을 실행하고, system call을 윈도우에서 처리할 수 있도록 linux kernel -> virtualized hardware -> hypervisor -> physical hardware의 과정을 통해서 이를 수행하게 된다. 이를 정리해보자면

```text
WSL1

Linux Program → Linux System call → WSL1 Compatibility Layer → Windows Kernel

WSL2

Linux Program → Linux System  call → Linux Kernel → Virtualized Hardware → Hypervisor → Physical Hardware
    

```

따라서 WSL2에서 Linux 프로그램이 `read()`, `write()`, `socket()`과 같은 system call을 호출하면 이를 Windows system call로 하나씩 번역하는 것이 아니라 실제 Linux Kernel이 처리한다. 이 구조 덕분에 WSL1보다 Linux system call 호환성이 높고 Linux 환경과 유사한 개발이 가능하다. 대신 WSL2에는 Windows Host와 Linux Guest 사이의 VM 경계가 존재하며 파일시스템, 네트워크와 같은 부분에 있어서 경계를 짓기 위해서 가상화가 들어가게 된다.


### VM과 Hypervisor

먼저 가상머신에 대해서 알아보면 가상머신은 하나의 physical machine 위에서 여러 운영체제가 서로 독립된 환경처럼 실행될 수 있도록 만든다. 이를 가능하게 하는 핵심이 Hypervisor이다. Hypervisor는 CPU, Memory, I/O device와 같은 physical resource를 여러 Virtual Machine이 공유할 수 있도록 관리하고 각각의 VM을 서로 격리한다. 각 Guest OS는 자신이 CPU, Memory, Network Device 등을 직접 사용하는 것처럼 보지만 실제로는 Hypervisor가 제공하는 virtualized resource를 사용한다. WSL2가 사용하는 Hyper-V는 Type-1 Hypervisor 구조이다. 이 구조에서는 Hypervisor가 hardware 바로 위에서 동작하고, Windows 역시 Hypervisor 위의 Root Partition에서 실행된다. WSL2의 Linux 환경은 별도의 Child Partition에 해당하는 경량 VM에서 실행된다. Hyper-V에서 Root Partition은 실제 device에 직접 접근할 수 있고, Child Partition은 virtual device를 통해 resource에 접근한다.

구조를 단순화하면 다음과 같이 볼 수 있다.

```text
               Windows
            Root Partition
                  │
Linux Guest       │
Child Partition   │
       │          │
       └── Hyper-V Hypervisor
                  │
           Physical Hardware
```

Linux 프로그램의 모든 동작이 매번 Hypervisor를 거치지는 않는다. 예를 들어 Linux program이 system call을 호출하면 먼저 Guest Linux Kernel이 이를 처리한다. Kernel 내부에서 해결할 수 있는 작업은 Guest 안에서 처리된다. 반면 disk I/O나 network I/O처럼 실제 hardware resource가 필요한 작업은 Guest에게 제공된 virtual device를 통해 Host 측의 실제 device로 전달된다. Hyper-V에서는 이러한 I/O를 효율적으로 처리하기 위해 Guest와 Root Partition 사이에 VMBus라는 통신 구조를 사용한다. Guest가 직접 physical NIC나 disk controller를 제어하는 것이 아니라 virtual device를 통해 요청하고, Root Partition의 실제 device driver가 hardware와 통신한다.


### Performance Profiling in a Virtualized Environment

Virtualization이 이 프로젝트에서 문제가 된 지점은 시스템 성능을 측정하고 원인을 분석하는 과정이다. 이 프로젝트에서는 이후 perf와 같은 도구를 이용해 cycles, instructions, cache-misses 등의 hardware event를 측정할 예정인데, 이러한 값은 CPU 내부의 PMU(Performance Monitoring Unit)를 통해 수집한다. Native Linux에서는 Linux Kernel이 physical CPU의 PMU를 직접 사용할 수 있지만, Virtual Machine에서는 Guest Kernel이 physical PMU에 직접 접근하는 것이 아니라 Hypervisor가 제공하는 virtual PMU를 사용해야 한다. 이때 perf는 Guest Kernel의 perf subsystem을 통해 PMU driver에 접근하고, 해당 driver가 Hypervisor가 노출한 virtual PMU를 사용할 수 있어야 하므로 perf가 설치되어 있다고 해서 모든 hardware event를 항상 측정할 수 있는 것은 아니다. 실제 WSL2 환경에서도 Microsoft WSL Kernel과 perf package의 version mismatch를 겪었고, 이를 해결한 이후에도 필요한 hardware counter를 정상적으로 사용할 수 없는 환경임을 확인했다. 또한 Virtual Machine에서는 Guest의 vCPU가 physical CPU에 어떻게 scheduling되는지, Host와 CPU와 I/O resource를 어떻게 공유하는지 등 Native 환경에는 없는 요소가 추가되므로 성능 측정 결과를 해석할 때 이러한 영향을 함께 고려해야 한다. 따라서 이후 server 구조별 성능 비교와 병목 분석에서는 측정 환경 자체에서 발생하는 변수를 줄이고 필요한 profiling 기능을 안정적으로 사용하기 위해 Native Linux를 기준 환경으로 사용한다.


### Need for separate machine in reverse proxy phase

Native Linux로 전환하더라도 Load Generator와 Server를 같은 machine에서 실행하면 network 측정에는 여전히 한계가 있다. Loopback 환경은 physical NIC를 거치지 않기 때문에 Blocking, Thread-per-Connection, Thread Pool, epoll과 같은 server architecture 자체의 처리 비용을 비교하기에는 적절하지만, 실제 network에서 발생하는 bandwidth limitation, latency, packet loss와 같은 요소는 포함하지 않는다. 특히 이후 Reverse Proxy 단계에서는 여러 backend와 통신하고 Backend Simulator를 통해 delay와 failure를 의도적으로 발생시키기 때문에 server process의 CPU 성능뿐 아니라 network path와 backend 상태까지 전체 system 성능에 영향을 준다. 따라서 초기 구조 비교는 Native Linux의 loopback 환경에서 수행하고, 이후 Reverse Proxy와 Failure / Overload 실험에서는 Load Generator와 Server를 별도의 machine으로 분리하여 throughput, tail latency, saturation point, bandwidth, failure rate와 같은 값들을 측정한다.

