---
title: "Concurrent Proxy Server"
hideSummary: true
---

### Overview

이 프로젝트의 출발점은 과거 C++로 게임 서버를 구현했던 경험이다. 당시 프로젝트인 [SOKC](https://github.com/irumi1206/SOKC)는 하나의 방에 접속한 여러 사용자가 서로 통신하는 구조를 가지고 있었고, TCP 연결마다 별도의 thread를 생성하여 각 client의 요청을 수신했다. 수신된 요청은 하나의 processing queue에 넣은 뒤 별도의 thread에서 순차적으로 처리하도록 하여, 여러 thread가 동시에 게임 로직을 수정하면서 발생할 수 있는 동시성 문제를 피하고자 했다. UDP는 위치처럼 자주 갱신되는 정보를 처리하기 위해 TCP와 분리하고, 별도의 receive/send thread를 통해 정보를 주고받도록 구성했다. 이 구조는 당시의 비교적 작은 규모에서는 충분히 동작했지만, 서버의 동시성과 성능을 체계적으로 다뤘다고 보기는 어려웠다. 연결 수가 늘어날수록 thread 수와 context switch가 어떻게 변하는지, 하나의 queue가 어느 시점부터 병목이 되는지, 많은 connection과 request가 들어왔을 때 throughput과 latency가 어떻게 달라지는지 등을 실제로 측정해보지는 않았다. 이번 프로젝트에서는 이 부분을 처음부터 다시 확인해보고자 한다. 처음에는 간단한 C++ Echo Server를 구현하고, 많은 동시 connection과 traffic이 발생할 때 서버 구조에 따라 어떤 차이가 생기는지 확인한다. 이후 이를 Reverse Proxy로 확장하고자 한다.

### Echo server 

먼저 네트워크 connection 처리 자체에 집중하기 위해 가장 단순한 Blocking TCP Echo Server에서 시작한다. 이후 Thread-per-Connection, Thread Pool, epoll 기반 Event-driven 구조로 하나씩 변경하면서 각 방식이 많은 connection을 처리할 때 어떤 차이를 가지는지 확인한다.
각 단계에서는 단순히 다음 구조로 넘어가는 것이 아니라, 같은 형태의 workload를 주고 throughput, latency, CPU usage, context switch 등의 변화를 비교하면서 현재 구조의 한계가 어디에서 발생하는지를 확인한다.


### Reverse Proxy

epoll 기반 서버까지 구현한 이후에는 단순한 echo server에서 끝내지 않고 실제 request를 backend server로 전달하는 Reverse Proxy로 확장한다. Reverse Proxy가 되면 client connection만 처리하는 것 외에도 고려해야 할 부분이 생긴다. client에서 받은 request를 backend로 전달하고 다시 response를 돌려주는 과정에서 backend connection을 어떻게 관리할지, backend가 느릴 때 데이터를 어떻게 처리할지, timeout이나 connection failure가 발생했을 때 어떻게 대응할지 등을 직접 다뤄보고자 한다. 또한 부하가 커졌을 때 request가 계속 쌓이는 상황을 어떻게 처리할지도 함께 확인한다.


### Load Generator and Backend Simulator

이러한 차이를 실제로 확인하기 위해 서버와 별도로 Load Generator와 Backend Simulator를 구성한다. Load Generator는 많은 TCP connection을 생성하고 request를 반복해서 보내면서 connection 수, request 수, request가 들어오는 속도 등을 조절한다. 이를 통해 평상시의 부하뿐 아니라 connection이나 traffic이 순간적으로 증가하는 상황도 만들어본다. Backend Simulator는 Reverse Proxy가 연결할 backend 역할을 하며, 정상적으로 빠르게 응답하는 경우뿐 아니라 응답이 느려지거나 일부 request가 실패하는 상황도 만들어본다. 이를 통해 proxy가 backend 상태에 따라 어떻게 동작하는지 확인할 수 있다. 필요한 경우 기존 load testing tool이나 network emulation tool도 함께 사용하여 직접 만든 실험 환경과 결과를 비교한다.

### Evaluation

각 구조에서는 주로 다음과 같은 항목을 비교한다.

- Throughput
- Latency
- CPU usage
- Context switch
- Success / Failure

최종적으로는 동일한 Load Generator와 Backend Simulator를 사용하여 직접 구현한 Reverse Proxy와 Nginx Reverse Proxy를 같은 조건에서 비교하고, 성능 차이가 발생하는 원인을 확인한 뒤 가능한 범위에서 직접 구현한 서버의 성능을 개선해보고자 한다.
