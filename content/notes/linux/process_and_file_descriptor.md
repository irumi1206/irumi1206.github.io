---
title: "Process and File Descriptor"
hideSummary: true
---

### Process

Executable과 Process의 기본적인 관계는 [Executable and Shell]({{< relref "executable_and_shell.md" >}})에서 정리했다. Executable을 실행하면 운영체제는 해당 program을 실행하기 위한 process를 생성한다. Process는 단순히 실행 중인 code만을 의미하는 것이 아니라 Linux kernel이 하나의 실행 단위로 관리하기 위해 필요한 실행 상태와 resource를 함께 가진다. 각 process에는 kernel이 부여하는 identifier인 PID(Process ID)가 존재하며, 같은 executable을 여러 번 실행하더라도 각각 다른 PID를 가지는 독립적인 process가 생성된다. PPID(Parent Process ID)는 해당 process를 생성한 parent process의 PID이며, Bash에서 external command를 실행하는 경우 Bash가 parent가 되고 실행된 program이 child process가 될 수 있다. Process는 또한 독립적인 virtual address space를 가지며, process가 사용하는 virtual address는 page table을 통해 physical memory와 mapping된다. 이 address space에는 executable code와 data, heap, stack, shared library 등의 영역이 배치될 수 있다. Process가 실행되는 동안 program counter, stack pointer, general-purpose register와 같은 CPU register state가 계속 변화하며, context switch가 발생하면 kernel은 실행을 이어갈 수 있도록 필요한 상태를 보존하고 복원한다. Stack은 function call frame, local variable, return address 등 함수 호출 과정에서 필요한 정보를 저장하는 영역이고, heap은 program 실행 중 동적으로 할당되는 memory를 위한 영역이다. 이외에도 process는 relative path의 기준이 되는 current working directory, 열린 file이나 socket과 같은 resource를 관리하기 위한 file descriptor table, scheduler가 관리하는 process state와 scheduling 정보 등을 가진다. 실제로 `pause()`를 통해 종료되지 않고 대기하도록 만든 간단한 C++ executable을 실행한 뒤 해당 process의 정보를 확인해보았다.

```text
$ g++ main.cpp -o main
$ ./main &
[1] 74872
Process started

$ MAIN_PID=$!

$ ps -p $MAIN_PID -o pid,ppid,user,stat,comm,args
    PID    PPID USER     STAT COMMAND         COMMAND
  74872    1248 sungha   S    main            ./main

$ cat /proc/$MAIN_PID/status
Name:   main
State:  S (sleeping)
Pid:    74872
PPid:   1248
Uid:    1000    1000    1000    1000
Gid:    1000    1000    1000    1000
FDSize: 256
VmSize:     6352 kB
VmRSS:      3200 kB
VmStk:       132 kB
VmExe:         4 kB
VmLib:      3712 kB
Threads:        1
voluntary_ctxt_switches:        2
nonvoluntary_ctxt_switches:     0

$ readlink /proc/$MAIN_PID/exe
/home/sungha/main

$ readlink /proc/$MAIN_PID/cwd
/home/sungha

$ cat /proc/$MAIN_PID/maps
5cf0ddcae000-5cf0ddcaf000 r--p ... /home/sungha/main
5cf0ddcaf000-5cf0ddcb0000 r-xp ... /home/sungha/main
5cf0ddcb2000-5cf0ddcb3000 rw-p ... /home/sungha/main
5cf116b2b000-5cf116b4c000 rw-p ... [heap]
7414d4600000-7414d4628000 r--p ... /usr/lib/x86_64-linux-gnu/libc.so.6
7414d4a00000-7414d4a9d000 r--p ... /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.33
7ffd87443000-7ffd87464000 rw-p ... [stack]

$ ls -l /proc/$MAIN_PID/fd
0 -> /dev/pts/5
1 -> /dev/pts/5
2 -> /dev/pts/5

$ cat /proc/$MAIN_PID/limits
Limit                     Soft Limit           Hard Limit
Max stack size            8388608              unlimited
Max processes             30485                30485
Max open files            1048576              1048576
Max address space         unlimited            unlimited

$ cat /proc/$MAIN_PID/sched
main (74872, #threads: 1)
se.vruntime                    : 103051.546424
se.sum_exec_runtime            : 2.362100
nr_switches                    : 2
nr_voluntary_switches          : 2
nr_involuntary_switches        : 0
policy                         : 0
prio                           : 120

$ kill $MAIN_PID

$ ps -p $MAIN_PID -o pid,ppid,user,stat,comm,args
    PID    PPID USER     STAT COMMAND         COMMAND
```

Linux에서는 `/proc` virtual filesystem을 통해 kernel이 관리하는 process 정보를 확인할 수 있다. 실행 중인 process에는 `/proc/<PID>` directory가 존재하며, `status`에서는 PID와 PPID, UID와 GID, memory 사용량, thread 수, context switch 횟수 등의 metadata를 확인할 수 있다. 여기서 실행한 `main`의 PID는 74872이고 PPID는 1248이며, Bash에서 실행했기 때문에 parent는 해당 Bash process가 된다. `exe`는 현재 process가 실행 중인 `/home/sungha/main` executable을, `cwd`는 current working directory인 `/home/sungha`를 가리킨다. `maps`에서는 executable과 `[heap]`, `[stack]`, `libc`, `libstdc++` 등이 서로 다른 virtual address range에 mapping되어 있는 것을 확인할 수 있으며, `r`, `w`, `x`는 각 mapping의 read, write, execute permission을 나타낸다. 또한 `fd`에서는 process가 가지고 있는 file descriptor를, `limits`에서는 process에 적용되는 resource limit을, `sched`에서는 scheduler가 관리하는 정보를 확인할 수 있다. 이때 process는 `pause()`를 통해 signal을 기다리고 있었기 때문에 `S (sleeping)` 상태이다. 마지막으로 `kill`을 통해 process를 종료한 뒤 같은 PID를 다시 조회하면 더 이상 process가 나타나지 않으며 `/proc/74872` directory도 사라진다. 따라서 `/proc/<PID>`에 존재하는 정보는 executable file 자체에 저장된 정보가 아니라 현재 kernel이 관리하고 있는 실행 중인 process의 상태와 resource를 나타낸다.

### Process State

Process는 항상 CPU에서 실행되고 있는 것은 아니다. CPU에서 instruction을 실행하고 있을 수도 있고, 실행 가능한 상태로 CPU 할당을 기다리거나 I/O와 같은 특정 event가 발생하기를 기다릴 수도 있다. Linux kernel은 이러한 실행 상태를 process state로 관리한다.

`ps`의 `STAT`이나 `/proc/<PID>/status`에서는 대표적으로 다음과 같은 process state를 확인할 수 있다.

```text
R   Running or Runnable
S   Interruptible Sleep
D   Uninterruptible Sleep
T   Stopped
Z   Zombie
```

`R`은 현재 CPU에서 실행 중이거나 CPU를 할당받으면 바로 실행할 수 있는 runnable 상태를 의미한다. 여러 process가 동시에 runnable 상태일 수 있지만 CPU core의 수는 제한되어 있으므로 모든 process가 동시에 실행되는 것은 아니다. `S`는 interruptible sleep 상태로, process가 특정 event나 I/O 등의 완료를 기다릴 때 CPU를 점유하지 않고 대기하는 상태이다. 앞에서 실행한 `main` process도 `pause()`를 통해 signal을 기다리고 있었기 때문에 `S (sleeping)`으로 나타났다. `D`는 주로 kernel 내부에서 I/O 등의 완료를 기다리는 uninterruptible sleep 상태이며, `T`는 signal이나 debugger 등에 의해 실행이 정지된 상태이다. `Z`는 process 자체의 실행은 종료되었지만 parent process가 아직 종료 상태를 회수하지 않아 일부 정보가 남아 있는 zombie 상태이다. Runnable한 process 중 어떤 process가 실제 CPU를 사용할지는 scheduler가 결정한다. Process state의 변화, context switch, scheduling policy와 Linux scheduler의 동작은 [Scheduling]({{< relref "scheduling.md" >}})에서 별도로 정리했다.


### File Descriptor

Process는 실행 중에 file, terminal, pipe, socket과 같은 여러 kernel resource를 사용할 수 있다. Linux에서는 process가 이러한 resource를 직접 가리키는 대신 process별로 file descriptor table을 관리하고, 각 entry를 식별하는 0 이상의 integer인 file descriptor를 통해 resource에 접근한다. File descriptor 자체가 file이나 socket인 것은 아니며, 현재 process가 kernel이 관리하는 열린 resource를 참조하기 위한 identifier이다. File descriptor table은 process마다 독립적으로 존재하므로 서로 다른 process에서 같은 번호의 file descriptor가 서로 다른 resource를 가리킬 수도 있다. 개념적으로 하나의 process는 다음과 같은 file descriptor table을 가질 수 있다.

```text
Process
│
└── File Descriptor Table
      ├── 0  → terminal
      ├── 1  → terminal
      ├── 2  → terminal
      ├── 3  → file
      ├── 4  → socket
      └── 5  → socket
```

일반적으로 process가 실행될 때 file descriptor 0, 1, 2는 각각 standard input, standard output, standard error로 사용된다. Standard input은 process가 기본적으로 input을 읽는 곳이고, standard output은 일반적인 program output이 전달되는 곳이며, standard error는 error나 diagnostic message를 위한 별도의 출력 경로이다.

실제로 file을 하나 열어 둔 C++ process를 만든 뒤 file descriptor table을 확인해보았다. 사용한 `main.cpp`는 다음과 같다.

```cpp
#include <fcntl.h>
#include <unistd.h>

int main() {
    int fd = open("fd_test.txt", O_CREAT | O_WRONLY, 0644);

    pause();

    close(fd);
    return 0;
}
```

`open()`으로 `fd_test.txt`를 연 뒤 `pause()`를 호출하여 process가 종료되지 않은 상태에서 file descriptor를 확인했다.

```text
$ g++ main.cpp -o main

$ ./main &
[1] 83158

$ MAIN_PID=$!

$ ls -l /proc/$MAIN_PID/fd
total 0
lrwx------ 1 sungha sungha 64 Aug 21 18:10 0 -> /dev/pts/5
lrwx------ 1 sungha sungha 64 Aug 21 18:10 1 -> /dev/pts/5
lrwx------ 1 sungha sungha 64 Aug 21 18:10 2 -> /dev/pts/5
l-wx------ 1 sungha sungha 64 Aug 21 18:10 3 -> /home/sungha/fd_test.txt

$ readlink /proc/$MAIN_PID/fd/0
/dev/pts/5

$ readlink /proc/$MAIN_PID/fd/1
/dev/pts/5

$ readlink /proc/$MAIN_PID/fd/2
/dev/pts/5

$ readlink /proc/$MAIN_PID/fd/3
/home/sungha/fd_test.txt

$ cat /proc/$MAIN_PID/limits | grep "Max open files"
Max open files            1048576              1048576              files

$ kill $MAIN_PID
```

File descriptor 0, 1, 2는 모두 현재 terminal device인 `/dev/pts/5`를 가리키고 있다. 각각 standard input, standard output, standard error로 사용된다. `open()`을 통해 새로운 `fd_test.txt`를 열자 기존 descriptor 다음의 사용 가능한 번호인 3이 반환되었으며, 실제로 `/proc/<PID>/fd/3`이 해당 file을 가리키는 것을 확인할 수 있다. 즉 process는 file을 연 이후 file name을 계속 사용하는 것이 아니라 반환된 file descriptor를 통해 kernel이 관리하는 열린 resource를 참조한다.

File descriptor가 가리키는 대상은 process를 실행할 때 변경될 수도 있다. 이를 확인하기 위해 이번에는 standard output과 standard error를 각각 출력하는 `main.cpp`를 사용했다.

```cpp
#include <iostream>
#include <unistd.h>

int main() {
    std::cout << "standard output\n";
    std::cerr << "standard error\n";

    pause();
    return 0;
}
```

이번에는 Bash에서 `> output.txt`를 사용하여 program을 실행하고 같은 방식으로 process의 PID를 저장한 뒤 file descriptor를 확인했다.

```text
$ g++ main.cpp -o main

$ ./main > output.txt &
[1] 83599
standard error

$ MAIN_PID=$!

$ ls -l /proc/$MAIN_PID/fd
total 0
lrwx------ 1 sungha sungha 64 Aug 21 18:11 0 -> /dev/pts/5
l-wx------ 1 sungha sungha 64 Aug 21 18:11 1 -> /home/sungha/output.txt
lrwx------ 1 sungha sungha 64 Aug 21 18:11 2 -> /dev/pts/5

$ cat output.txt
standard output

$ kill $MAIN_PID
```

일반적으로 실행했을 때는 file descriptor 0, 1, 2가 모두 terminal을 가리켰지만, `> output.txt`를 사용하자 standard output인 file descriptor 1만 `output.txt`를 가리키도록 변경되었다.

```text
./main

FD 0 → terminal
FD 1 → terminal
FD 2 → terminal

./main > output.txt

FD 0 → terminal
FD 1 → output.txt
FD 2 → terminal
```

Bash가 `main`을 실행하기 전에 file descriptor 1이 `output.txt`를 가리키도록 설정한다. 따라서 `std::cout`으로 출력한 `standard output`은 file에 기록되었지만, `std::cerr`가 사용하는 file descriptor 2는 그대로 terminal을 가리키고 있어 `standard error`는 terminal에 출력되었다.

File descriptor를 사용하는 대상은 regular file에 한정되지 않는다. Linux에서는 terminal, pipe, device, network socket 등 여러 kernel resource를 file descriptor를 통해 참조할 수 있다. 이러한 공통적인 참조 방식을 사용하기 때문에 process는 서로 다른 종류의 resource를 file descriptor를 기준으로 다룰 수 있다. `read()`, `write()`, `close()`와 같은 system call과 file descriptor의 관계는 [System Call]({{< relref "system_call.md" >}})에서 별도로 정리했다.

File descriptor는 진행했었던 프로젝트 [Concurrent c++ proxy server]({{< relref "../../projects/concurrent_proxy_server.md" >}})와 같이 직접 서버를 만드는데에서 주의깊게 다뤄져야 한다. 소켓 프로그래밍에서 `socket()`이 성공하면 socket을 참조하는 file descriptor가 반환되고, listening socket에서 `accept()`를 통해 client connection을 받으면 해당 connection을 위한 새로운 file descriptor가 반환된다.

```cpp
int server_fd = socket(AF_INET, SOCK_STREAM, 0);
int client_fd = accept(server_fd, ...);
```

따라서 여러 client와 연결된 server process의 file descriptor table은 개념적으로 다음과 같이 구성될 수 있다.

```text
0   stdin
1   stdout
2   stderr
3   listening socket
4   client socket
5   client socket
6   client socket
...
```

`server_fd`와 `client_fd`가 integer인 이유도 이 때문이다. 이 값들은 socket 자체가 아니라 현재 process가 kernel의 socket resource를 참조하기 위한 file descriptor이다. 하지만 File descriptor는 무한히 생성할 수 있는 resource는 아니다. 실제 process에서 확인한 것처럼 현재 환경의 `Max open files`는 1,048,576으로 설정되어 있다. 각 client connection의 socket 역시 file descriptor를 사용하므로 동시에 많은 connection을 유지할 때 application의 concurrency 구조뿐만 아니라 이러한 system resource limit도 고려해야 한다. 따라서 file descriptor는 process가 file, terminal, socket 등의 서로 다른 kernel resource를 공통적인 integer identifier를 통해 참조할 수 있도록 하는 구조이며, 이후 multiple connection, blocking I/O, `epoll`을 이해하는 데에도 기본이 되는 개념이다.