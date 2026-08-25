---
title: "System Call"
hideSummary: true
---

### User Mode / Kernel Mode

Process의 thread는 일반적인 application code를 User Mode에서 실행하며, file이나 network socket과 같이 kernel이 관리하는 resource에 접근해야 할 경우 Kernel Mode로 전환하여 kernel code를 실행한다. User Mode와 Kernel Mode를 구분하는 가장 큰 이유는 privilege를 분리하여 application이 kernel memory나 hardware resource를 임의로 조작하지 못하도록 하기 위해서이다. System Call은 user-space program이 kernel에 특정 작업을 요청하기 위해 kernel이 제공하는 interface이며, `open`, `read`, `write`, `close`, `socket`, `bind` 등이 대표적이다. System Call을 호출하면 현재 thread가 Kernel Mode로 진입하여 해당 kernel code를 수행하고, 작업이 끝나면 다시 User Mode로 돌아온다. 이 과정에서 반드시 다른 process로 전환되는 것은 아니기 때문에 User Mode와 Kernel Mode 사이의 mode switch와 process/thread 사이의 context switch는 서로 다른 개념이다. System Call 외에도 hardware interrupt나 현재 instruction을 수행하는 과정에서 발생하는 exception/fault를 통해 Kernel Mode로 진입할 수 있다.

### System Call and File Descriptor

각 process는 자신만의 file descriptor table을 가지고 있으며, file descriptor는 process가 사용하고 있는 file, terminal, pipe, socket 등의 kernel resource를 참조하기 위한 integer identifier이다. 예를 들어 `open()`을 통해 file을 열거나 `socket()`을 통해 socket을 생성하면 kernel은 해당 process의 file descriptor table에 새로운 entry를 만들고 그 번호를 반환한다. 이후 `read()`, `write()`, `close()` 등의 System Call은 전달받은 file descriptor를 통해 어떤 resource에 작업을 수행할지 식별한다. File Descriptor 자체에 대한 자세한 내용은 [Process and File Descriptor]({{< relref "process_and_file_descriptor.md" >}})에서 정리하였다.

### Privileged Port

Network socket 역시 file descriptor를 통해 관리되며, server는 `socket()`으로 socket을 생성한 뒤 `bind()`를 통해 특정 address와 port에 연결할 수 있다. Linux에서는 낮은 범위의 port를 privileged port로 두고 일반 process가 임의로 bind하지 못하도록 제한할 수 있다. 전통적으로 1024보다 작은 port가 privileged port로 사용되어 왔지만, 실제 Linux에서는 `net.ipv4.ip_unprivileged_port_start` 값이 일반 user가 사용할 수 있는 첫 번째 port를 결정한다. 이 값보다 낮은 port에 bind하려면 일반적으로 root 권한이나 `CAP_NET_BIND_SERVICE` capability가 필요하다. 따라서 privileged port는 단순히 특정 process가 사용하기 때문에 막아둔 port라기보다, 낮은 port를 privileged service 영역으로 두고 일반 process의 bind를 제한하는 mechanism으로 보는 것이 더 정확하다.

현재 system에서 privileged port의 boundary를 확인해보면

```bash
sysctl net.ipv4.ip_unprivileged_port_start
# net.ipv4.ip_unprivileged_port_start = 1024
```

`sysctl`은 kernel의 runtime parameter를 조회하거나 변경할 때 사용하는 command이다. 위 결과에서 현재 system에서는 1024가 첫 번째 unprivileged port이므로, 그보다 낮은 1023 port에 일반 user로 `bind()`를 시도해보면

```python
# bind.py

import socket

PORT = 1023

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

try:
    sock.bind(("127.0.0.1", PORT))
    print(f"bind({PORT}) succeeded")
except OSError as e:
    print(f"bind({PORT}) failed: errno={e.errno}, {e}")
finally:
    sock.close()
```

```bash
python3 bind.py
# bind(1023) failed: errno=13, [Errno 13] Permission denied
```

`socket()`을 통해 TCP socket을 생성한 뒤 `bind()`로 1023 port에 연결하려 하면, 현재 user에게 해당 privileged port를 bind할 권한이 없기 때문에 kernel이 `Permission denied`를 반환한다.

### strace

`strace`는 process가 실행하면서 호출하는 System Call과 signal을 확인할 수 있는 tool이다. Program에서 `Permission denied`와 같은 error가 발생했을 때 application에서 보이는 error message뿐 아니라 실제로 어떤 System Call이 실패했고 kernel이 어떤 error를 반환했는지를 확인할 수 있다. 여기서는 `strace` 자체의 자세한 사용법보다는 System Call을 직접 관찰하기 위한 용도로 간단하게 사용해보면

```bash
echo "test" > permission_denied.txt
cat permission_denied.txt
# test

chmod 000 permission_denied.txt

cat permission_denied.txt
# Permission denied

strace -e trace=openat cat permission_denied.txt 2>&1 | grep permission_denied.txt
# openat(AT_FDCWD, "permission_denied.txt", O_RDONLY) = -1 EACCES (Permission denied)
```

`chmod 000`으로 file의 permission을 모두 제거한 뒤 `cat`을 실행하면 접근이 거부되며, `strace`를 통해 확인했을 때 실제로 `openat()` System Call이 `EACCES`를 반환하는 것을 볼 수 있다.
