---
title: 某即时通讯软件的逆向工程——数据库解密
toc: true
categories:
  - Reverse Engineering
tags: [逆向工程, 密码学, 反编译, 数据库]
date: 2026-04-28 09:40:00
updated: 2026-04-28 09:40:00
---

本文记录了我对某个用于商家询盘的即时通讯（IM）软件的本地数据库的逆向工程过程。主要介绍了如何通过 IDA Pro 调试来分析其数据库文件的加密机制。

<!-- more -->

## 前言

### 目标

本次逆向工程的最终目标是：**实现该软件的数据库文件的解密**。

待解密的数据库，被存放在如下目录结构中：

```
path/to/software/data/IMServiceDir/MessageSDK/
|- some_user_id1/
|  |- database/
|  |  |- im.sqlite
|  |  |- im.sqlite-shm
|  |  |- im.sqlite-wal
|  |  |- im.sqlite_fts
|  |  |- im.sqlite_fts-shm
|  |  |- im.sqlite_fts-wal
|  |  |- sync.sqlite
|  |  |- sync.sqlite-shm
|  |  |- sync.sqlite-wal
│  |- sync/
│     |- sync
|- some_user_id2/
   |- ...
```

在该目录下，根据用户 ID 的不同，数据库会被存放在不同目录中。根据文件名称可以推测，软件采用的是 SQLite 数据库引擎。

在这里，我们研究的主要目标是 `im.sqlite` 文件。其余的数据库文件，例如 `sync.sqlite` 仅作为对照参考使用。

由于无法使用常规的 SQLite 浏览工具来查看这些文件的内容，并且这些文件的头部都不是以 `SQLite format 3\x00` 开头，我们可以初步推测这些文件是经过加密处理的。

## 文件分析

### 初步观察文件内容

在采用更激进的手段（例如对软件进行调试分析）之前，我们先对这些数据库文件的字节内容进行一些静态分析，以寻找可能的线索。

`im.sqlite` 文件（大小约 3.8 MB）的 Hex Dump 如下所示，头部 16 字节：

```
00000000  82 4D F5 44 85 CD 4C 42  |.M.D..LB|
00000008  E5 9C DC 80 4B DC DB F2  |....K...|
```

随后是一些没有规律的字节数据，在此省略。紧接着出现了一些重复的 16 字节序列：

```
00000110  21 4A 2A 6C F3 6E 48 9D  |!J*l.nH.|
00000118  73 AD 82 26 E6 AB BE ED  |s..&....|
00000120  21 4A 2A 6C F3 6E 48 9D  |!J*l.nH.|
00000128  73 AD 82 26 E6 AB BE ED  |s..&....|
00000130  21 4A 2A 6C F3 6E 48 9D  |!J*l.nH.|
00000138  73 AD 82 26 E6 AB BE ED  |s..&....|
```

再然后又是一些没有规律的字节数据。但是不久之后，上述 16 字节序列又会再次出现和重复。

### 文件中出现的重复密文

我们可以推测，**这种重复的密文可能对应了某种重复明文**。在常规 SQLite 文件中，最常出现的重复二进制内容是全零字节。因此，我们可以大胆猜测，这些重复的密文有可能对应了全零字节的明文。

如果上述猜想成立，那么数据库文件**采用的加密算法可能是某种分组加密算法**，典型的是 AES 算法的 ECB 模式。在该模式下，相同的明文块会被加密成相同的密文块。如果确实是 AES-ECB 或类似的分组加密算法，那么极有可能分组字节数是 16 字节（128 位）。

### 其他文件的对比分析

查看相同用户的 `sync.sqlite` 文件，我们发现其头部 16 字节与 `im.sqlite` 文件完全相同，并且文件内容中也出现了同样的重复密文。如果先前关于分组加密算法的推测无误，那么这个头部 16 字节对应的明文很可能就是 `SQLite format 3\x00`。

但是，查看其他用户的数据库文件，我们却发现文件的头部 16 字节完全不同了。

结合上述现象，我们可以推测，**同一用户的所有数据库文件共享相同的密钥，但是不同用户的密钥是不同的**。

### 初步结论

即使本小节的推测全部正确，考虑到 AES-ECB 对已知明文攻击（KPA）具有抗性，我们不能仅凭现有文件来直接推断出密钥。

因此，无论如何，我们都需要通过对软件进行动态分析来获取更多的线索。

## 动态分析

接下来将通过 IDA Pro 调试来分析数据库文件的加密机制。

### 截获文件访问

启动调试，等待软件进入登录界面后，在 `C:\Windows\System32\KERNEL32.DLL` 模块的 `CreateFileW` 函数开头处设置断点。

该函数的定义是：

```cpp
HANDLE CreateFileW(
  [in]           LPCWSTR               lpFileName,
  [in]           DWORD                 dwDesiredAccess,
  [in]           DWORD                 dwShareMode,
  [in, optional] LPSECURITY_ATTRIBUTES lpSecurityAttributes,
  [in]           DWORD                 dwCreationDisposition,
  [in]           DWORD                 dwFlagsAndAttributes,
  [in, optional] HANDLE                hTemplateFile
);
```

> 在 64 位 Windows 程序中，约定：函数的前 4 个整数或指针参数依次使用 RCX、RDX、R8、R9 寄存器传递。

为了仅在访问 `im.sqlite` 文件时才触发断点，我们可以设置如下条件：

```python
ptr = idc.get_reg_value("RCX")  # lpFileName
if ptr == 0:
    return False

try:
    raw = idc.read_dbg_memory(ptr, 1024)
    if not raw:
        return False
    text = raw.decode('utf-16-le', errors='ignore')
    null_idx = text.index('\x00')
    if null_idx != -1:
        name = text[:null_idx]
        print(f"! File opened: {name}")
        return name.endswith("im.sqlite")
except Exception as e:
    print(f"! Breakpoint error: {e}")

return False
```

首次命中该断点时，对应的栈帧如下：

```
Module	Function
KERNEL32.DLL	kernel32_CreateFileW
icbu_aim.dll	icbu_aim_fts_get_fts_table_names+EC92
icbu_aim.dll	icbu_aim_fts_get_fts_table_names+15CCB
icbu_aim.dll	icbu_aim_fts_get_fts_table_names+1D5D0
icbu_aim.dll	icbu_aim_fts_get_fts_table_names+988F1
icbu_aim.dll	icbu_aim_fts_get_fts_table_names+4F5
icbu_aim.dll	icbu_aim_?shared_from_this...XZ+A1826
icbu_aim.dll	icbu_aim_?shared_from_this...XZ+A8EA9
icbu_aim.dll	icbu_aim_?shared_from_this...XZ+A3078
icbu_aim.dll	icbu_aim_?shared_from_this...XZ+97D94
icbu_aim.dll	icbu_aim_?shared_from_this...XZ+90F5B
...
KERNEL32.DLL	kernel32_BaseThreadInitThunk+18
ntdll.dll	ntdll_RtlUserThreadStart+22
```

显然，软件对数据库的操作是由 `icbu_aim.dll` 模块来负责的。

在整个 `CreateFileW` 函数的函数体运行结束后，得到最后一个函数参数（`hTemplateFile`）的值是 `0x1418`，这是一个文件句柄，程序后面会用到它。

### 截获文件读取

拿到文件句柄后，我们需要在 `C:\Windows\System32\KERNEL32.DLL` 模块的 `ReadFile` 函数开头处设置断点，从而截获对数据库文件的读取操作。

该函数的定义是：

```cpp
BOOL ReadFile(
  [in]                HANDLE       hFile,
  [out]               LPVOID       lpBuffer,
  [in]                DWORD        nNumberOfBytesToRead,
  [out, optional]     LPDWORD      lpNumberOfBytesRead,
  [in, out, optional] LPOVERLAPPED lpOverlapped
);
```

断点条件是：

```python
ptr = idc.get_reg_value("RCX")  # hFile
return ptr == 0x1418
```

> 每次调试时，`hTemplateFile` 的值通常不相同。应根据实际情况修改断点条件中的文件句柄值。

该断点命中后，向上查找调用者，发现 `ReadFile` 是在下面的 A 处的里面被调用的：

```c
__int64 __fastcall sub_7FFDC792D930(__int64 a1, int a2)
{
  __int64 v2; // rdi
  unsigned int v4; // ebp
  __int64 v5; // rbx
  __int64 v6; // r11
  __int64 v7; // r9
  int v8; // r10d
  unsigned int n522; // ebx
  __m128i si128; // xmm0
  __int64 (__fastcall *v11)(_QWORD, _QWORD, _QWORD, __int64); // rax

  v2 = *(a1 + 32);
  v4 = *(a1 + 40);
  v5 = *(a1 + 8);
  v6 = *(v2 + 188);
  if ( a2 )
  {
    // ... unmet condition
  }
  else
  {
    n522 = (*(**(v2 + 72) + 16LL))(*(v2 + 72), v5, v6, v6 * (v4 - 1)); // A
    if ( n522 == 522 )
      n522 = 0;
  }
  if ( v4 == 1 )
  {
    if ( n522 )
      si128 = _mm_load_si128(xmmword_7FFDC7DAF4B0);
    else
      si128 = *(*(a1 + 8) + 24LL);
    *(v2 + 136) = si128;
  }
  v11 = *(v2 + 264);
  if ( v11 && !v11(*(v2 + 288), *(a1 + 8), v4, 3) ) // B
    return 7;
  return n522;
}
```

随后，程序进入了 B 处，调用的函数 `v11` 的定义如下：

```c
__int64 __fastcall sub_7FFDC79DAE40(__int64 a1, unsigned int *a2, __int64 a3, __int64 a4)
{
  __int64 v4; // rsi
  __int64 v6; // r8
  int v7; // edi
  int v8; // r9d
  int v9; // r9d
  __int64 v10; // r9
  __int64 v11; // rbp
  __int64 v12; // rsi
  __int64 v13; // rbx
  __int64 v14; // rdi
  __int64 v15; // rbp
  unsigned int *v16; // rbx
  __int64 v17; // rdi

  v4 = a2;
  if ( a1 && *a1 )
  {
    v6 = *(*(a1 + 16) + 8LL);
    v7 = *(v6 + 52);
    if ( a4 && (v8 = a4 - 2) != 0 && (v9 = v8 - 1) != 0 )
    {
      // ... unmet condition
    }
    else if ( *(a1 + 8) && v7 > 0 )
    {
      v15 = a1 + 204;
      v16 = a2;
      v17 = ((v7 - 1) >> 4) + 1;
      do
      {
        sub_7FFDC791BDB0(v16, v16, v15); // C
        v16 += 4;
        --v17;
      }
      while ( v17 );
    }
  }
  return v4;
}
```

在上述代码的 C 处，我们发现程序在对 `v16` 指向的内存进行某种处理，高度怀疑是**就地解密**。

### 追踪就地解密

进入 C 处的函数中：

```c
__int64 __fastcall sub_7FFDC791BDB0(_DWORD *a1, unsigned int *a2, __int64 a3)
{
  // [COLLAPSED LOCAL DECLARATIONS.]

  v3 = *(a3 + 176); // now v3 = 10
  result = (4 * v3);
  v5 = (a3 + 4 * result);
  v6 = *a1 ^ *v5;
  v7 = a1[1] ^ v5[1];
  v8 = a1[2] ^ v5[2];
  v9 = a1[3] ^ v5[3];
  v57 = v5;
  switch ( v3 )
  {
    case 10: // <-- hit this case
      goto LABEL_6;
    case 12:
LABEL_5:
    // ... unmet condition
LABEL_6:
      v19 = *(v5 - 4) ^ *(&unk_7FFDC7830000 + v6 + 1345340) ^ *(&unk_7FFDC7830000 + (v7 >> 24) + 1346108) ^ (*(&unk_7FFDC7830000 + BYTE2(v8) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v9) + 1345596));
      v20 = *(v5 - 3) ^ *(&unk_7FFDC7830000 + v7 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE1(v6) + 1345596) ^ *(&unk_7FFDC7830000 + (v8 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v9) + 1345852);
      v21 = *(v5 - 2) ^ *(&unk_7FFDC7830000 + v8 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE2(v6) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v7) + 1345596) ^ *(&unk_7FFDC7830000 + (v9 >> 24) + 1346108);
      v22 = *(v5 - 4) ^ *(&unk_7FFDC7830000 + 2 * v8 + 2690680) ^ *(&unk_7FFDC7830000 + 2 * BYTE2(v6) + 2691704) ^ *(&unk_7FFDC7830000 + 2 * BYTE1(v7) + 2691192) ^ *(&unk_7FFDC7830000 + 2 * (v9 >> 24) + 2692216);
      v23 = *(v57 - 1) ^ *(&unk_7FFDC7830000 + v9 + 1345340) ^ *(&unk_7FFDC7830000 + (v6 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v7) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v8) + 1345596);
      v24 = *(v57 - 8) ^ *(&unk_7FFDC7830000
            + (*(v5 - 16) ^ *(&unk_7FFDC7830000 + 4 * v6 + 5381360) ^ *(&unk_7FFDC7830000 + 4 * (v7 >> 24) + 5384432) ^ *(&unk_7FFDC7830000 + 4 * BYTE2(v8) + 5383408) ^ *(&unk_7FFDC7830000 + 4 * BYTE1(v9) + 5382384))
            + 1345340) ^ *(&unk_7FFDC7830000 + (v20 >> 24) + 1346108) ^ (*(&unk_7FFDC7830000 + BYTE2(v21) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v23) + 1345596));
      v25 = *(v57 - 7) ^ *(&unk_7FFDC7830000 + v20 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE1(v19) + 1345596) ^ *(&unk_7FFDC7830000 + (v21 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v23) + 1345852);
      v26 = *(v57 - 6) ^ *(&unk_7FFDC7830000 + v22 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE2(v19) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v20) + 1345596) ^ *(&unk_7FFDC7830000 + (v23 >> 24) + 1346108);
      v27 = *(v57 - 24) ^ *(&unk_7FFDC7830000 + 4 * v22 + 5381360) ^ *(&unk_7FFDC7830000 + 4 * BYTE2(v19) + 5383408) ^ *(&unk_7FFDC7830000 + 4 * BYTE1(v20) + 5382384) ^ *(&unk_7FFDC7830000 + 4 * (v23 >> 24) + 5384432);
      v28 = *(v57 - 5) ^ *(&unk_7FFDC7830000
            + (*(v57 - 4) ^ *(&unk_7FFDC7830000 + 4 * v9 + 5381360) ^ *(&unk_7FFDC7830000 + 4 * (v6 >> 24) + 5384432) ^ *(&unk_7FFDC7830000 + 4 * BYTE2(v7) + 5383408) ^ *(&unk_7FFDC7830000 + 4 * BYTE1(v8) + 5382384))
            + 1345340) ^ *(&unk_7FFDC7830000 + (v19 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v20) + 1345852) ^ *(&unk_7FFDC7830000 + HIBYTE(v22) + 1345596);
      v29 = *(v57 - 12) ^ *(&unk_7FFDC7830000 + v24 + 1345340) ^ *(&unk_7FFDC7830000 + (v25 >> 24) + 1346108) ^ (*(&unk_7FFDC7830000 + BYTE2(v26) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v28) + 1345596));
      v30 = *(v57 - 11) ^ *(&unk_7FFDC7830000 + v25 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE1(v24) + 1345596) ^ *(&unk_7FFDC7830000 + (v26 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v28) + 1345852);
      v31 = *(v57 - 10) ^ *(&unk_7FFDC7830000 + v27 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE2(v24) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v25) + 1345596) ^ *(&unk_7FFDC7830000 + (v28 >> 24) + 1346108);
      v32 = *(v57 - 9) ^ *(&unk_7FFDC7830000 + v28 + 1345340) ^ *(&unk_7FFDC7830000 + (v24 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v25) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v26) + 1345596);
      v33 = *(v57 - 16) ^ *(&unk_7FFDC7830000 + v29 + 1345340) ^ *(&unk_7FFDC7830000 + (v30 >> 24) + 1346108) ^ (*(&unk_7FFDC7830000 + BYTE2(v31) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v32) + 1345596));
      v34 = *(v57 - 15) ^ *(&unk_7FFDC7830000 + v30 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE1(v29) + 1345596) ^ *(&unk_7FFDC7830000 + (v31 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v32) + 1345852);
      v35 = *(v57 - 14) ^ *(&unk_7FFDC7830000 + v31 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE2(v29) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v30) + 1345596) ^ *(&unk_7FFDC7830000 + (v32 >> 24) + 1346108);
      v36 = *(v57 - 28) ^ *(&unk_7FFDC7830000 + 2 * v31 + 2690680) ^ *(&unk_7FFDC7830000 + 2 * BYTE2(v29) + 2691704) ^ *(&unk_7FFDC7830000 + 2 * BYTE1(v30) + 2691192) ^ *(&unk_7FFDC7830000 + 2 * (v32 >> 24) + 2692216);
      v37 = *(v57 - 13) ^ *(&unk_7FFDC7830000 + v32 + 1345340) ^ *(&unk_7FFDC7830000 + (v29 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v30) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v31) + 1345596);
      v38 = *(v57 - 20) ^ *(&unk_7FFDC7830000 + v33 + 1345340) ^ *(&unk_7FFDC7830000 + (v34 >> 24) + 1346108) ^ (*(&unk_7FFDC7830000 + BYTE2(v35) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v37) + 1345596));
      LODWORD(v24) = *(v57 - 19)
                   ^ *(&unk_7FFDC7830000 + v34 + 1345340)
                   ^ *(&unk_7FFDC7830000 + BYTE1(v33) + 1345596)
                   ^ *(&unk_7FFDC7830000 + (v35 >> 24) + 1346108)
                   ^ *(&unk_7FFDC7830000 + BYTE2(v37) + 1345852);
      v39 = *(v57 - 18) ^ *(&unk_7FFDC7830000 + v36 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE2(v33) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v34) + 1345596) ^ *(&unk_7FFDC7830000 + (v37 >> 24) + 1346108);
      v40 = *(v57 - 17) ^ *(&unk_7FFDC7830000 + v37 + 1345340) ^ *(&unk_7FFDC7830000 + (v33 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v34) + 1345852) ^ *(&unk_7FFDC7830000 + HIBYTE(v36) + 1345596);
      v41 = *(v57 - 24) ^ *(&unk_7FFDC7830000 + v38 + 1345340) ^ *(&unk_7FFDC7830000 + (v24 >> 24) + 1346108) ^ (*(&unk_7FFDC7830000 + BYTE2(v39) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v40) + 1345596));
      v42 = *(v57 - 23) ^ *(&unk_7FFDC7830000 + v24 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE1(v38) + 1345596) ^ *(&unk_7FFDC7830000 + (v39 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v40) + 1345852);
      v43 = *(v57 - 22) ^ *(&unk_7FFDC7830000 + v39 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE2(v38) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v24) + 1345596) ^ *(&unk_7FFDC7830000 + (v40 >> 24) + 1346108);
      v44 = *(v57 - 21) ^ *(&unk_7FFDC7830000 + v40 + 1345340) ^ *(&unk_7FFDC7830000 + (v38 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v24) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v39) + 1345596);
      v45 = *(v57 - 28) ^ *(&unk_7FFDC7830000 + v41 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE2(v43) + 1345852) ^ (*(&unk_7FFDC7830000 + BYTE1(v44) + 1345596) ^ *(&unk_7FFDC7830000 + (v42 >> 24) + 1346108));
      LODWORD(v24) = *(v57 - 27)
                   ^ *(&unk_7FFDC7830000 + v42 + 1345340)
                   ^ *(&unk_7FFDC7830000 + (v43 >> 24) + 1346108)
                   ^ *(&unk_7FFDC7830000 + BYTE2(v44) + 1345852)
                   ^ *(&unk_7FFDC7830000 + BYTE1(v41) + 1345596);
      v46 = *(v57 - 26) ^ *(&unk_7FFDC7830000 + v43 + 1345340) ^ *(&unk_7FFDC7830000 + (v44 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v41) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v42) + 1345596);
      v47 = *(v57 - 52) ^ *(&unk_7FFDC7830000 + 2 * v43 + 2690680) ^ *(&unk_7FFDC7830000 + 2 * (v44 >> 24) + 2692216) ^ *(&unk_7FFDC7830000 + 2 * BYTE2(v41) + 2691704) ^ *(&unk_7FFDC7830000 + 2 * BYTE1(v42) + 2691192);
      v48 = *(v57 - 25) ^ *(&unk_7FFDC7830000 + v44 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE1(v43) + 1345596) ^ *(&unk_7FFDC7830000 + (v41 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v42) + 1345852);
      v49 = *(v57 - 32) ^ *(&unk_7FFDC7830000 + v45 + 1345340) ^ *(&unk_7FFDC7830000 + (v24 >> 24) + 1346108) ^ (*(&unk_7FFDC7830000 + BYTE2(v46) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v48) + 1345596));
      v50 = *(v57 - 31) ^ *(&unk_7FFDC7830000 + v24 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE1(v45) + 1345596) ^ *(&unk_7FFDC7830000 + (v46 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v48) + 1345852);
      v51 = *(v57 - 30) ^ *(&unk_7FFDC7830000 + v47 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE2(v45) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v24) + 1345596) ^ *(&unk_7FFDC7830000 + (v48 >> 24) + 1346108);
      v52 = *(v57 - 29) ^ *(&unk_7FFDC7830000 + v48 + 1345340) ^ *(&unk_7FFDC7830000 + (v45 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v24) + 1345852) ^ *(&unk_7FFDC7830000 + HIBYTE(v47) + 1345596);
      v53 = *(v57 - 36) ^ *(&unk_7FFDC7830000 + v49 + 1345340) ^ *(&unk_7FFDC7830000 + (v50 >> 24) + 1346108) ^ (*(&unk_7FFDC7830000 + BYTE2(v51) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v52) + 1345596));
      v54 = *(v57 - 35) ^ *(&unk_7FFDC7830000 + v50 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE1(v49) + 1345596) ^ *(&unk_7FFDC7830000 + (v51 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v52) + 1345852);
      v55 = *(v57 - 34) ^ *(&unk_7FFDC7830000 + v51 + 1345340) ^ *(&unk_7FFDC7830000 + BYTE2(v49) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v50) + 1345596) ^ *(&unk_7FFDC7830000 + (v52 >> 24) + 1346108);
      v56 = *(v57 - 33) ^ *(&unk_7FFDC7830000 + v52 + 1345340) ^ *(&unk_7FFDC7830000 + (v49 >> 24) + 1346108) ^ *(&unk_7FFDC7830000 + BYTE2(v50) + 1345852) ^ *(&unk_7FFDC7830000 + BYTE1(v51) + 1345596);
      v6 = *(v57 - 40) ^ *(&unk_7FFDC7830000 + v53 + 1342268) ^ *(&unk_7FFDC7830000 + (v54 >> 24) + 1343036) ^ *(&unk_7FFDC7830000 + BYTE2(v55) + 1342780) ^ *(&unk_7FFDC7830000 + BYTE1(v56) + 1342524);
      v7 = *(v57 - 39) ^ *(&unk_7FFDC7830000 + v54 + 1342268) ^ *(&unk_7FFDC7830000 + BYTE1(v53) + 1342524) ^ *(&unk_7FFDC7830000 + (v55 >> 24) + 1343036) ^ *(&unk_7FFDC7830000 + BYTE2(v56) + 1342780);
      v8 = *(v57 - 38) ^ *(&unk_7FFDC7830000 + v55 + 1342268) ^ *(&unk_7FFDC7830000 + BYTE2(v53) + 1342780) ^ *(&unk_7FFDC7830000 + BYTE1(v54) + 1342524) ^ *(&unk_7FFDC7830000 + (v56 >> 24) + 1343036);
      result = v56;
      v9 = *(v57 - 37) ^ *(&unk_7FFDC7830000 + v56 + 1342268) ^ *(&unk_7FFDC7830000 + (v53 >> 24) + 1343036) ^ *(&unk_7FFDC7830000 + BYTE2(v54) + 1342780) ^ *(&unk_7FFDC7830000 + BYTE1(v55) + 1342524);
      break;
    case 14:
      // ... unmet condition
      goto LABEL_5;
  }
  *a2 = v6;
  a2[1] = v7;
  a2[2] = v8;
  a2[3] = v9;
  return result;
}
```

该函数是 AES 分组解密函数实现，参数 `a1`、`a2`、`a3` 分别是输入数据指针、输出数据指针、轮密钥，局部变量 `v3` 的含义是轮数。

| AES 变体 | 密钥长度        | 标准轮数 |
| -------- | --------------- | -------- |
| AES-128  | 16 字节 / 128 位 | 10       |
| AES-192  | 24 字节 / 192 位 | 12       |
| AES-256  | 32 字节 / 256 位 | 14       |

调试中发现，这里的**轮数值是 10，即采用的是 AES-128 的标准轮数**。

### 获取密钥并尝试解密

回到之前的 C 处：

```c
// ...
v15 = a1 + 204;
v16 = a2;
v17 = ((v7 - 1) >> 4) + 1;
do
{
  sub_7FFDC791BDB0(v16, v16, v15); // C
  v16 += 4;
  --v17;
}
while ( v17 );
// ...
```

那么这里的 `v15` 就是我们要找的密钥了，它的长度是 16 字节，通过调试器可以直接获取到它的值：

```
dd38a8f61e726d94
```

现在我们可以撰写一份简单的 Python 解密代码来验证一下这个密钥是否正确：

```python
from Crypto.Cipher import AES

key = b"dd38a8f61e726d94"

assert len(key) == 16, "AES key must be 16 bytes long"

with open("im.sqlite", "rb") as f:
    encrypted_db = f.read()

cipher = AES.new(key, AES.MODE_ECB)
decrypted_db = cipher.decrypt(encrypted_db)

with open("im_decrypted.sqlite", "wb") as f:
    f.write(decrypted_db)
```

运行上述代码后，我们得到的 `im_decrypted.sqlite` 文件确实被完整地还原了，并且可以被 SQLite 浏览工具正常打开。

## 下一步

我们已经找到了指定用户的数据库文件的加密密钥，并成功解密了数据库文件。下一步需要做的是**找到密钥的生成办法**，因为不同的用户对应了不同的密钥。

经过若干分析尝试，发现密钥的生成办法较为复杂，暂时没有找到明确的线索。但是，别忘了我们可以**直接在软件的内存中搜索密钥**。~~事实上就是懒得继续找了，反正能 dump 内存.jpg~~ 

根据已知的密钥值，我们猜测数据库密钥满足正则表达式 `[0-9a-f]{16}`。只需要在内存中搜索所有可能的密钥，随后依次对数据库的首 16 字节进行 AES-ECB-128 解密，看看能不能得到 `SQLite format 3\x00` 的明文头部即可。

实践表明，该方法确实可行，搜索耗时约几秒钟，属于可接受范围。至此，本次逆向工程的目标已经完成了。
