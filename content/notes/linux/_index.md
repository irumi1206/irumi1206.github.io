---
title: "Linux"
hideSummary: true
---

### Program / Executable / Process / Terminal / Bash / PATH


**Program**은 컴퓨터가 수행할 명령과 로직을 표현한 일반적인 개념이다. C++ source code, Python program, shell script 모두 넓은 의미에서 program이라고 볼 수 있다. 하지만 program이 작성되어 있다는 것과 운영체제가 해당 파일을 실제로 실행할 수 있다는 것은 별개의 문제다. **Executable**은 운영체제에서 실행 가능한 형태의 파일을 의미한다. Executable에는 실행해야 할 machine code뿐만 아니라 code와 data가 어떻게 구성되어 있는지, 프로그램을 메모리에 어떤 형태로 적재해야 하는지, 실행을 어디서 시작해야 하는지 등 프로그램을 실행하기 위해 필요한 구조적인 정보도 포함될 수 있다. Linux에서 C/C++ program을 컴파일하여 생성한 executable은 일반적으로 ELF(Executable and Linkable Format) 형식을 사용한다. 실제로 간단한 C++ program을 컴파일한 뒤 생성된 executable을 살펴보면 이러한 구조를 확인할 수 있다.

```text
$ g++ main.cpp -o main

$ file ./main
./main: ELF 64-bit LSB pie executable, x86-64

$ readelf -h ./main

ELF Header:
  Class:                             ELF64
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Entry point address:               0x1080
  Start of program headers:          64 (bytes into file)
  Start of section headers:          14416 (bytes into file)
  Number of program headers:         13
  Number of section headers:         31

$ readelf -l ./main

Elf file type is DYN (Position-Independent Executable file)
Entry point 0x1080
There are 13 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000002d8 0x00000000000002d8  R      0x8

  INTERP         0x0000000000000318 0x0000000000000318 0x0000000000000318
                 0x000000000000001c 0x000000000000001c  R      0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]

  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000760 0x0000000000000760  R      0x1000

  LOAD           0x0000000000001000 0x0000000000001000 0x0000000000001000
                 0x0000000000000241 0x0000000000000241  R E    0x1000

  LOAD           0x0000000000002000 0x0000000000002000 0x0000000000002000
                 ...

$ objdump -d ./main

./main:     file format elf64-x86-64

Disassembly of section .init:

0000000000001000 <_init>:
    1000:       f3 0f 1e fa             endbr64
    1004:       48 83 ec 08             sub    $0x8,%rsp
    1008:       48 8b 05 e1 2f 00 00    mov    0x2fe1(%rip),%rax
    100f:       48 85 c0                test   %rax,%rax
    1012:       74 02                   je     1016 <_init+0x16>
    1014:       ff d0                   call   *%rax
    1016:       48 83 c4 08             add    $0x8,%rsp
    101a:       c3                      ret
```

file 명령을 통해 생성된 파일이 x86-64용 ELF executable이라는 것을 확인할 수 있다. 

readelf -h는 ELF header를 보여준다. Machine에는 executable이 대상으로 하는 CPU architecture가 나타나고, Entry point address에는 프로그램 실행이 시작되는 위치가 기록되어 있다. 이외에도 program header와 section header의 위치와 개수 등 ELF 파일 자체의 구조에 대한 정보가 포함되어 있다. 

readelf -l에서는 프로그램이 실행될 때 사용되는 program header들을 확인할 수 있다. 특히 LOAD는 실행 과정에서 메모리에 적재될 영역을 나타내며, R, W, E는 각각 해당 영역이 Read, Write, Execute 가능한지를 나타낸다. 예를 들어 R E가 설정된 LOAD 영역에는 실행 가능한 code가 포함될 수 있다. 또한 INTERP에는 이 executable을 실행하는 과정에서 사용되는 dynamic linker의 경로가 기록되어 있는 것도 확인할 수 있다.

objdump -d는 executable에 들어 있는 machine code를 disassemble하여 보여준다. 왼쪽의 f3 0f 1e fa, 48 83 ec 08과 같은 값들이 실제 executable에 저장된 machine-code byte이고, 오른쪽의 endbr64, sub, mov, call, ret 등이 이를 사람이 읽을 수 있도록 표현한 x86-64 assembly instruction이다. 따라서 Linux의 ELF executable은 단순히 machine code만 나열한 파일이 아니다. CPU가 실행할 machine code와 program에서 사용하는 data뿐만 아니라 CPU architecture, entry point, 메모리에 적재할 영역과 그 속성 등 운영체제가 프로그램을 적재하고 실행하기 위해 필요한 정보도 일정한 형식으로 함께 가지고 있다.

이러한 C/C++ executable은 native executable이라고 하며, 특정 CPU architecture와 실행 환경을 대상으로 만들어진다. 위의 출력에서도 x86-64와 Advanced Micro Devices X86-64가 표시되는 것을 볼 수 있다. 이는 이 executable이 x86-64 architecture를 대상으로 생성되었다는 의미이며, 내부의 machine code 역시 x86-64 instruction set에 맞게 만들어져 있다. 같은 C++ source code라도 ARM64를 대상으로 컴파일하면 ARM64 instruction set에 맞는 다른 machine code가 생성된다. 이러한 의미에서 native executable은 CPU architecture에 dependent하다. 반면 Python program이나 shell script처럼 program 자체가 native machine code를 포함하지 않는 경우도 있다. 이들은 Python interpreter나 Bash와 같은 별도의 native executable이 program의 내용을 읽고 실행한다. Script에 실행 권한과 적절한 interpreter 정보가 설정되어 있다면 script 파일 자체에 직접 실행을 요청할 수도 있지만, 실제 CPU에서 실행되는 machine code는 Python interpreter나 Bash와 같은 interpreter 쪽에 포함되어 있다.

Executable은 파일 시스템에 저장되어 있는 실행 가능한 파일이지만, 이를 실제로 실행하면 운영체제는 해당 program을 실행하기 위한 **process**를 생성한다. 즉 process는 실행 중인 program의 instance라고 볼 수 있으며, 같은 executable을 여러 번 실행하더라도 각각 독립적인 process가 생성될 수 있다. Linux kernel은 각 process를 구분하기 위해 PID(Process ID)를 부여하고, process별 virtual address space, CPU register 상태, stack과 heap, 열린 file과 socket을 관리하는 file descriptor table, scheduling을 위한 상태 등 실제 실행에 필요한 정보를 함께 관리한다.

Linux에서 program을 실행하기 위해 흔히 사용하는 **Terminal**과 **Shell**은 서로 다른 역할을 한다. Terminal은 사용자의 text input을 전달하고 program의 text output을 화면에 보여주는 interface이다. 즉 Terminal 자체가 ls, cd, g++과 같은 command의 의미를 해석하여 실행하는 것은 아니며, 사용자가 입력한 command를 실제로 읽고 해석하는 역할은 Shell이 담당한다. Shell은 사용자의 command를 해석하고 필요한 동작을 수행하는 command interpreter이며, Bash는 Linux에서 널리 사용되는 shell 중 하나이다. Bash 역시 하나의 program이며 Linux에서는 native executable 형태로 존재한다. 현재 사용하고 있는 shell과 Bash executable을 한번 살펴보면 다음과 같다.

```text
$ echo $SHELL
/bin/bash

$ file /bin/bash
/bin/bash: ELF 64-bit LSB pie executable, x86-64, ...

$ type cd
cd is a shell builtin

$ type pwd
pwd is a shell builtin

$ type -P ls
/usr/bin/ls

$ type -P g++
/usr/bin/g++

$ type -P cmake
/usr/bin/cmake
```

사용자에게 설정된 shell이 /bin/bash이며, 앞에서 살펴본 C++ executable과 마찬가지로 Bash 자체도 x86-64용 ELF native executable이라는 것을 확인할 수 있다. Terminal에서 Bash가 실행되면 하나의 Bash process가 존재하고, 사용자가 Terminal에 command를 입력하면 Bash가 이를 읽어 어떤 동작을 수행해야 하는지 판단한다. 이때 모든 command가 같은 방식으로 처리되는 것은 아니다. cd와 pwd처럼 Bash 내부에 직접 구현되어 실행되는 command를 shell builtin이라고 하며, ls, g++, cmake와 같은 command는 별도의 executable file로 존재한다. Bash는 입력받은 command가 builtin이라면 내부에서 직접 처리하고, external command라면 해당 executable을 찾아 새로운 process로 실행한다. 특히 cd가 shell builtin으로 구현되어 있는 이유는 current working directory가 process가 가지고 있는 상태이기 때문이다. cd는 현재 실행 중인 Bash process의 working directory 자체를 변경해야 한다. 만약 cd가 별도의 executable로 실행되어 새로운 child process의 working directory만 변경한다면, 실행이 종료된 뒤 기존 Bash process의 working directory에는 아무런 변화가 없게 된다.

그렇다면 Bash는 사용자가 ls라고 입력했을 때 /usr/bin/ls에 executable이 존재한다는 것을 어떻게 찾을까. External command를 실행할 때 Shell은 **PATH environment variable**에 등록된 directory들을 순서대로 검색한다. 실제 PATH에 대허서 살펴보면 다음과 같다.

```text
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:...
```

PATH에는 executable을 검색할 directory들이 colon(:)으로 구분되어 저장되어 있다. Bash는 external command가 입력되면 PATH에 등록된 directory들을 순서대로 검색하여 해당 executable을 찾아 실행한다.