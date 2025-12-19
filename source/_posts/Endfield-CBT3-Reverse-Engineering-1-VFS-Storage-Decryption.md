---
title: 《终末地》CBT3逆向工程1：VFS资源存储解密
toc: true
categories:
  - Reverse Engineering
tags: [逆向工程, 密码学, 反编译, Unity]
date: 2025-12-05 17:09:00
updated: 2025-12-07 17:33:00
thumbnail: /Reverse-Engineering/Endfield-CBT3-Reverse-Engineering-1-VFS-Storage-Decryption/BLC.jpg
---

本文记录了我对 Unity 游戏[《明日方舟：终末地》](https://endfield.hypergryph.com/)（简称《终末地》）全面测试版本中的 VFS 资源存储逻辑的逆向工程过程。主要介绍了如何通过对反编译代码的静态分析，结合调试器的动态分析，解析《终末地》的 VFS 资源存储格式，并实现解析工具。

## 前言

### 目标

对某个游戏做逆向工程，通常出于以下几个目的：

- 分析游戏资源的存储格式，以便提取或修改资源；
- 分析游戏的网络协议，以便制作私服；
- 分析游戏的代码逻辑，以供学习或制作辅助工具。

在“《终末地》CBT3 逆向工程”系列文章中，我们将着重分析《终末地》的**资源存储逻辑**，最终目的是实现美术、音乐资源和数值数据的提取。这里的 CBT3 指的是第三次封闭测试（Closed Beta Test 3），其内部代号是 `EndFieldTBeta2`。

<!-- more -->

### 初步分析

《终末地》是基于 [Unity 2021.3](https://docs.unity3d.com/2021.3/Documentation/Manual/) 开发的跨平台游戏，使用了 **il2cpp** 作为脚本后端。游戏在发行时分为了中国版（CN）和海外版（OS）两个版本进行分发。游戏的资源文件采用了自定义的 VFS（虚拟文件系统，Virtual File System）进行存储，并且对资源数据进行了加密处理。

游戏文件在文件系统中的存储结构是：

```
EndFieldTBeta2_Data/
|- index_initial.json
|- index_main.json
|- VFS/
   |- 07A1BB91/
   |  |- 07A1BB91.blc
   |  |- 24F006196A004C8E3A259EADA8F45818.chk
   |  |- 69B1B82E779ECFD39DD0B4F94EAFB39D.chk
   |- 0CE8FA57/
   |  |- 0CE8FA57.blc
   |  |- 10EFD763A7B3D5536C4268DE20B2A1C2.chk
   |- 19E3AE45/
   |  |- ...
   |- 1CDDBF1F/
   |  |- ...
   |- ...
```

其中，`.blc` 文件的大小通常较小，而 `.chk` 文件则较大。据此初步推测，BLC 文件可能是某种索引或元数据文件，而 CHK 文件则存储了实际的资源数据。

对 BLC 文件进行二进制检视，发现其开头 4 Bytes 始终为 `03 00 00 00`，而其余部分的香农熵高达 8.00，说明其内容极有可能是加密的。

对 CHK 文件进行二进制检视，发现：

- `55FC21C6` 目录下的文件的开头始终为 `CRID`，说明是 [Criware](https://game.criware.jp/) USM 文件（进行常规 USM 文件解码后可正常获得视频资源，表明此类文件未加密）。
- `07A1BB91` 等若干目录下的文件的开头始终为 `:)xD`，考虑是某种自定义的资源包（事后确认是加密后的 [AudioKinetic](https://www.audiokinetic.com/) Package）。
- 其余目录下的文件开头均无显著特征，且香农熵大部分为 8.00，少部分低于 4，表明这些资源大部分是加密资源。在部分加密文件中，搜索到 `2021.3.34f5` 字符串，很可能是 Unity 引擎的版本号。

要想弄清楚这些加密资源是什么，我们需要从 BLC 文件入手，尝试解析其内容。

## 执行 Il2cpp 反编译

为了了解游戏程序内是如何解析 BLC 文件的，我们需要对 il2cpp 程序行反编译。在 Windows 平台上，《终末地》的 il2cpp 程序被命名为 `GameAssembly.dll`，在 Android 平台上则是 `libil2cpp.so`。

### 准备 global-metadata

若直接使用 [IDA Pro](https://hex-rays.com/ida-pro) 对其进行反编译，得到的结果是没有任何符号信息的汇编代码，难以阅读和分析。为此，我们需要获取一个记录了符号信息的 `global-metadata.dat` 文件，并使用 Il2cppInspector 工具对 il2cpp 程序进行**符号化处理**，才能在 IDA Pro 中降低分析的难度。

当我们打开 CN Win 和 OS Win 版本的 `global-metadata.dat` 文件时，发现该文件的内容是：

```
T_T T_T
```

显然，程序员开了一个玩笑，直接将符号信息文件替换成了颜文字。

但是，CN Android 和 OS Android 版本中，`global-metadata.dat` 文件并没有被这样替换掉。其中，CN Android 版本的 `global-metadata.dat` 文件的开头 4 Bytes 是 `94 43 72 12`，不符合正常的 `global-metadata.dat` 的开头（应为 `AF 1B B1 FA`）。经社区成员提示，我了解到这是腾讯的 **ACE 反作弊系统**对 `global-metadata.dat` 文件的头部数据所进行的反逆向工程的保护处理，参见[这篇文章](https://intl.anticheatexpert.com/resource-center/content-91.html)。幸运的是，OS Android 版本的 `global-metadata.dat` 文件并没有被 ACE 保护，开头 4 Bytes 是正常的。

因此，现状是，我们可以对 Android 版本的 so 库进行反编译和静态分析，但是如果我们想要调试 so 库的话，难免会遇到一些麻烦。为此，能不能通过一些简单的手段，获取 Windows 版本的 `global-metadata.dat` 文件呢？

作为一个 il2cpp 程序，`global-metadata.dat` 文件会被加载到内存中，以供程序运行时使用。如果内存中的 `global-metadata.dat` 文件没有被加密处理，那么我们就可以通过**内存转储**的方式，获取到未被破坏的 `global-metadata.dat` 文件。为此，我们可以尝试对游戏进行内存转储。

然而，当我们试图使用工具对 Windows 版本的游戏进行内存转储时，发现遭到了某些进程级机制的阻止。经社区成员提示，这一阻止机制可以通过替换掉游戏启动程序 `EndFieldTBeta2.exe` 来绕过。具体而言，我们需要创建一个空白的同名 Unity 项目，并生成一份干净的 `EndFieldTBeta2.exe`，用它来替换掉游戏目录下的启动程序。实践表明，这样做确实可以绕过内存转储的阻止机制。

在转储文件中搜索二进制 `AF 1B B1 FA`，我们很快找到了内存中的一处疑似 `global-metadata.dat` 的数据。将其与 Android 版本的 `global-metadata.dat` 文件进行对比后，可以轻易地导出完整的 `global-metadata.dat` 文件。这一点是幸运的，要是内存中的 `global-metadata.dat` 文件也被加密处理的话，那么我们就只能使用 Android 版本的 so 库进行分析了。

### 使用 Il2cppInspector 生成符号化文件

至此我们已经获得了 Windows 版本的 `global-metadata.dat` 文件，接下来我们就可以使用 Il2cppInspector 工具，对 `GameAssembly.dll` 进行符号化处理。

由于原来的 Il2cppInspector 项目已于 2021 年停止维护，这里我们使用的是 [LukeFZ 维护的 Il2cppInspector](https://github.com/LukeFZ/Il2CppInspectorRedux)。运行以下命令行：

```bash
.\Il2CppInspector.exe process -m "path\to\global-metadata.dat" -i "path\to\libil2cpp.so" --unity-version 2021.3.34f5 -t IDA
```

我们就可以生成供 IDA Pro 使用的符号化文件，包括：`dll` 文件夹，`cpp` 文件夹，`metadata.json` 文件，`types.cs` 文件和 `il2cpp.py` 文件。

### 使用 IDA Pro 进行符号化的反编译

接下来，在 IDA Pro 中打开 `GameAssembly.dll` 文件，并在菜单栏的 `File -> Script file` 中选择 `il2cpp.py` 脚本运行。这一过程可能需要较长时间。脚本运行完成后，便可以在伪代码试图中阅读到符号化的 C++ 代码了。

## 分析 VFS 相关代码

我们能很快在函数表中找到与 VFS 读取相关的函数。下面我整理了几个关键的函数片段，快速浏览代码即可，无需细究。

### 生成 VFBlockMainInfo 实例的函数

可以发现 `VFBlockMainInfo` 是最顶层的结构，每个 `VFBlockMainInfo` 代表一个**单独的 BLC 文件**。以下函数负责生成 `VFBlockMainInfo` 结构体实例：

```cpp
// VFBlockMainInfo ReadFromByteBuf(Byte[], Int32)
VFBlockMainInfo *Beyond::VFS::VFBlockMainInfo::ReadFromByteBuf(
    Byte__Array *cfgBytes,
    int32_t startOffset,
    MethodInfo *method)
{
    int v5;
    int32_t Int, version, groupFileInfoNum, v34;
    struct FVFBlockChunkInfo__Array *allChunks_1, *allChunks;
    __int64 i_1, v10, rgctxDataDummy_1, *v15, v33;
    VFBlockMainInfo *v9;
    struct MethodInfo *_ZN10MethodInfo6System5Array5EmptyIN6Beyond3VFS17FVFBlockChunkI;
    _BYTE *rgctxDataDummy;
    signed __int64 v16;
    ByteBufStream v19;
    unsigned int v28, i;
    FVFBlockChunkInfo *v32, v43;
    Byte__Array *bs[2];
    // ...

    // ...
    if (IFix::WrappersManagerImpl::IsPatched(1162, 0))
    {
        // ...
    }
    else
    {
        if (cfgBytes)
        {
            v5 = LODWORD(cfgBytes->max_length) - startOffset - 4;
            if (v5 > 0)
            {
                Int = Beyond::Byte::ByteHelper::ReadInt(cfgBytes, LODWORD(cfgBytes->max_length) - 4, 1, 0);
                // ...
                if (Int == sub_7FFD1790B5F0(cfgBytes, startOffset, v5))
                {
                    v9 = sub_7FFD156E1D80(TypeInfo::Beyond::VFS::VFBlockMainInfo);
                    if (!v9)
                        goto LABEL_66;
                    // ...
                    _ZN10MethodInfo6System5Array5EmptyIN6Beyond3VFS17FVFBlockChunkI = MethodInfo::System::Array::Empty<Beyond::VFS::FVFBlockChunkInfo>;
                    // ...
                    rgctxDataDummy_1 = _ZN10MethodInfo6System5Array5EmptyIN6Beyond3VFS17FVFBlockChunkI->rgctx_data->rgctxDataDummy;
                    // ...
                    v9->fields.allChunks = **(rgctxDataDummy_1 + 184);
                    // ...
                    if (IFix::WrappersManagerImpl::IsPatched(45, 0))
                    {
                        // ...
                    }
                    else
                    {
                        // ...
                    }
                    *bs = v19;
                    if (IFix::WrappersManagerImpl::IsPatched(13, 0))
                    {
                        // ...
                    }
                    else
                    {
                        version = Beyond::Byte::ByteHelper::ReadInt(bs[1], bs[0], 1, 0);
                        LODWORD(bs[0]) += 4;
                    }
                    v9->fields.version = version;
                    v9->fields.groupCfgName = Beyond::Byte::ByteBufStream::ReadUTF8(bs, 0);
                    sub_7FFD156BBC20(&v9->fields.groupCfgName);
                    v9->fields.groupCfgHashName = Beyond::Byte::ByteBufStream::ReadLong(bs, 0);
                    if (IFix::WrappersManagerImpl::IsPatched(13, 0))
                    {
                        // ...
                    }
                    else
                    {
                        groupFileInfoNum = Beyond::Byte::ByteHelper::ReadInt(bs[1], bs[0], 1, 0);
                        LODWORD(bs[0]) += 4;
                    }
                    v9->fields.groupFileInfoNum = groupFileInfoNum;
                    v9->fields.groupChunksLength = Beyond::Byte::ByteBufStream::ReadLong(bs, 0);
                    v9->fields.blockType = Beyond::Byte::ByteBufStream::ReadByte(bs, 0);
                    if (IFix::WrappersManagerImpl::IsPatched(13, 0))
                    {
                        // ...
                    }
                    else
                    {
                        v28 = Beyond::Byte::ByteHelper::ReadInt(bs[1], bs[0], 1, 0);
                        LODWORD(bs[0]) += 4;
                    }
                    v9->fields.allChunks = il2cpp_array_new_specific_1(TypeInfo::Beyond::VFS::FVFBlockChunkInfo, v28);
                    sub_7FFD156BBC20(&v9->fields.allChunks);
                    i_1 = 0;
                    for (i = 0;; i_1 = i)
                    {
                        allChunks = v9->fields.allChunks;
                        if (!allChunks)
                            goto LABEL_66;
                        if (i_1 >= SLODWORD(allChunks->max_length))
                            break;
                        // ...
                        v32 = Beyond::VFS::FVFBlockChunkInfo::ReadFromByteBuf(&v43, bs, 0);
                        allChunks_1 = v9->fields.allChunks;
                        if (!allChunks_1)
                            goto LABEL_66;
                        if (i >= LODWORD(allChunks_1->max_length))
                            sub_7FFD15783440(i_1, allChunks_1, v10);
                        v33 = i++ << 6;
                        *(&allChunks_1->vector[0].md5Name + v33) = v32->md5Name;
                        *(&allChunks_1->vector[0].contentMD5 + v33) = v32->contentMD5;
                        *(&allChunks_1->vector[0].length + v33) = *&v32->length;
                        *(&allChunks_1->vector[0].files.m_Length + v33) = *&v32->files.m_Length;
                    }
                    if (IFix::WrappersManagerImpl::IsPatched(44, 0))
                    {
                        // ...
                    }
                    else
                    {
                        if (!bs[1])
                            goto LABEL_66;
                        v34 = LODWORD(bs[1]->max_length) - LODWORD(bs[0]);
                        if (v34 <= 0)
                            v34 = 0;
                    }
                    if (v34 <= 0)
                        return v9;
                    if (!IFix::WrappersManagerImpl::IsPatched(13, 0))
                    {
                        Beyond::Byte::ByteHelper::ReadInt(bs[1], bs[0], 1, 0);
                        return v9;
                    }
                    v36 = IFix::WrappersManagerImpl::GetPatch(13, 0);
                    if (v36)
                    {
                        IFix::ILFixDynamicMethodWrapper::__Gen_Wrap_7(v36, bs, 0);
                        return v9;
                    }
                LABEL_66:
                    sub_7FFD15783430(i_1, allChunks_1, v10);
                }
                // ...
            }
        }
        return 0;
    }
}
```

这个函数从一个字节数组中读取数据，并填充到 `VFBlockMainInfo` 结构体的各个字段中。代码中，`Beyond::Byte::ByteBufStream::ReadUTF8` 函数负责读取 UTF-8 字符串，具体代码在此不再赘述，其逻辑是先读取一个 2 Bytes 整数作为长度 n，随后读取 n Bytes 的字符串内容；`v28` 变量是 4 Bytes 整数表示 `allChunks` 数组的长度。

于是，我们可以整理出 `VFBlockMainInfo` 的数据结构定义：

```cpp
struct VFBlockMainInfo
{
    int32_t version;
    String *groupCfgName; // int16 (length) + n Bytes (content)
    __int64 groupCfgHashName;
    int32_t groupFileInfoNum;
    __int64 groupChunksLength;
    uint8_t blockType; 
    struct FVFBlockChunkInfo__Array *allChunks; // int32 (length) + n * FVFBlockChunkInfo
};
```

### 生成 FVFBlockChunkInfo 实例的函数

一个 `VFBlockMainInfo` 包含多个 `FVFBlockChunkInfo`，每个 `FVFBlockChunkInfo` 代表一个**单独的 CHK 文件**。以下函数负责生成 `FVFBlockChunkInfo` 结构体实例：

```cpp
// FVFBlockChunkInfo ReadFromByteBuf(ByteBufStream ByRef)
FVFBlockChunkInfo *Beyond::VFS::FVFBlockChunkInfo::ReadFromByteBuf(
    FVFBlockChunkInfo *__return_ptr retstr,
    ByteBufStream *cfg,
    MethodInfo *method)
{
    UInt128 md5Name_1, bs__1, bs_, md5Name;
    int32_t Int;
    int Int_1;
    __int64 Int_2;
    struct Void *m_Buffer;
    FVFBlockFileInfo *v14;
    struct UInt128 fileChunkMD5Name, fileDataMD5, contentMD5;
    __int128 v17, v18, v20, v24, v25;
    FVFBlockChunkInfo *v22, v29, v31;
    struct NativeArray_1_Beyond_VFS_FVFBlockFileInfo_ files;
    // ...

    // ...
    if (!IFix::WrappersManagerImpl::IsPatched(1163, 0))
    {
        *(&v29.blockType + 1) = 0;
        *(&v29.blockType + 5) = 0;
        *(&v29.blockType + 7) = 0;
        if (IFix::WrappersManagerImpl::IsPatched(21, 0))
        {
            // ...
        }
        else
        {
            md5Name_1 = *Beyond::Byte::ByteHelper::ReadUInt128(&bs_, cfg->datas, cfg->currentIdx, 1, 0);
            cfg->currentIdx += 16;
        }
        md5Name = md5Name_1;
        if (IFix::WrappersManagerImpl::IsPatched(21, 0))
        {
            // ...
        }
        else
        {
            bs__1 = *Beyond::Byte::ByteHelper::ReadUInt128(&bs_, cfg->datas, cfg->currentIdx, 1, 0);
            cfg->currentIdx += 16;
        }
        bs_ = bs__1;
        v29.length = Beyond::Byte::ByteBufStream::ReadLong(cfg, 0);
        v29.blockType = Beyond::Byte::ByteBufStream::ReadByte(cfg, 0);
        if (!IFix::WrappersManagerImpl::IsPatched(13, 0))
        {
            Int = Beyond::Byte::ByteHelper::ReadInt(cfg->datas, cfg->currentIdx, 1, 0);
            cfg->currentIdx += 4;
        LABEL_16:
            Int_1 = Int;
            files = 0;
            sub_7FFD14942900(
                &files,
                Int,
                4,
                1,
                MethodInfo::Unity::Collections::NativeArray<Beyond::VFS::FVFBlockFileInfo>::NativeArray);
            Int_2 = Int_1;
            v29.files = files;
            if (Int_1 > 0)
            {
                m_Buffer = files.m_Buffer;
                do
                {
                    // ...
                    v14 = Beyond::VFS::FVFBlockFileInfo::ReadFromByteBuf(&v31, cfg, 0);
                    fileChunkMD5Name = v14->fileChunkMD5Name;
                    fileDataMD5 = v14->fileDataMD5;
                    v17 = *&v14->offset;
                    v18 = *&v14->ivSeed;
                    *m_Buffer = *&v14->m_fileName.handle;
                    *&m_Buffer[16] = fileChunkMD5Name;
                    *&m_Buffer[32] = fileDataMD5;
                    *&m_Buffer[48] = v17;
                    *&m_Buffer[64] = v18;
                    m_Buffer += 80;
                    --Int_2;
                } while (Int_2);
            }
            v20 = *&v29.files.m_Length;
            retstr->md5Name = md5Name;
            retstr->contentMD5 = bs_;
            *&retstr->length = *&v29.length;
            *&retstr->files.m_Length = v20;
            return retstr;
        }
        // ...
    LABEL_25:
        sub_7FFD15783430();
    }
    // ...
    contentMD5 = v22->contentMD5;
    retstr->md5Name = v22->md5Name;
    v24 = *&v22->length;
    retstr->contentMD5 = contentMD5;
    v25 = *&v22->files.m_Length;
    *&retstr->length = v24;
    *&retstr->files.m_Length = v25;
    return retstr;
}
```

这个函数从一个字节流中读取数据，并填充到 `FVFBlockChunkInfo` 结构体的各个字段中。代码中，`Int_1` 变量是 4 Bytes 整数表示 `files` 数组的长度。

可以整理出 `FVFBlockChunkInfo` 的数据结构定义：

```cpp
struct FVFBlockChunkInfo
{
    UInt128 md5Name;
    UInt128 contentMD5;
    __int64 length;
    byte blockType;
    struct NativeArray_1_Beyond_VFS_FVFBlockFileInfo_ files; // int32 (length) + n * FVFBlockFileInfo
};
```

### 生成 FVFBlockFileInfo 实例的函数

一个 `FVFBlockChunkInfo` 包含多个 `FVFBlockFileInfo`，每个 `FVFBlockFileInfo` 代表着**某个虚拟文件的在 CHK 中的元数据**。以下函数负责生成 `FVFBlockFileInfo` 结构体实例：

```cpp
// FVFBlockFileInfo ReadFromByteBuf(ByteBufStream ByRef)
FVFBlockFileInfo *Beyond::VFS::FVFBlockFileInfo::ReadFromByteBuf(
    FVFBlockFileInfo *__return_ptr retstr,
    ByteBufStream *cfg,
    MethodInfo *method)
{
    int32_t currentIdx, v12, currentIdx_1;
    struct UInt128 fileChunkMD5Name_1, fileDataMD5_1, fileChunkMD5Name, fileDataMD5;
    uint8_t Byte;
    String *UTF8;
    __int128 v16, v17, v23, v24;
    ILFixDynamicMethodWrapper_3 *Patch;
    FVFBlockFileInfo *v20, v27;
    UInt128 bs_, bs__1;
    GCHandle m_fileName;
    // ...

    // ...
    if (IFix::WrappersManagerImpl::IsPatched(1164, 0))
    {
        Patch = IFix::WrappersManagerImpl::GetPatch(1164, 0);
        if (Patch)
        {
            v20 = IFix::ILFixDynamicMethodWrapper::__Gen_Wrap_484(&v27, Patch, cfg, 0);
            fileChunkMD5Name = v20->fileChunkMD5Name;
            *&retstr->m_fileName.handle = *&v20->m_fileName.handle;
            fileDataMD5 = v20->fileDataMD5;
            retstr->fileChunkMD5Name = fileChunkMD5Name;
            v23 = *&v20->offset;
            retstr->fileDataMD5 = fileDataMD5;
            v24 = *&v20->ivSeed;
            *&retstr->offset = v23;
            *&retstr->ivSeed = v24;
            return retstr;
        }
        goto LABEL_33;
    }
    *&v27.ivSeed = 0;
    v27.m_fileName.handle = 0;
    if (IFix::WrappersManagerImpl::IsPatched(8, 0))
    {
        // ...
    }
    else
    {
        currentIdx = cfg->currentIdx;
    }
    Beyond::Byte::ByteBufStream::SkipReadUTF8(cfg, 0);
    v27.fileNameHash = Beyond::Byte::ByteBufStream::ReadLong(cfg, 0);
    if (IFix::WrappersManagerImpl::IsPatched(21, 0))
    {
        // ...
    }
    else
    {
        fileChunkMD5Name_1 = *Beyond::Byte::ByteHelper::ReadUInt128(&bs_, cfg->datas, cfg->currentIdx, 1, 0);
        cfg->currentIdx += 16;
    }
    if (IFix::WrappersManagerImpl::IsPatched(21, 0))
    {
        // ...
    }
    else
    {
        fileDataMD5_1 = *Beyond::Byte::ByteHelper::ReadUInt128(&bs__1, cfg->datas, cfg->currentIdx, 1, 0);
        cfg->currentIdx += 16;
    }
    v27.offset = Beyond::Byte::ByteBufStream::ReadLong(cfg, 0);
    v27.len = Beyond::Byte::ByteBufStream::ReadLong(cfg, 0);
    Byte = Beyond::Byte::ByteBufStream::ReadByte(cfg, 0);
    v27.blockType = Byte;
    v12 = Beyond::Byte::ByteBufStream::ReadByte(cfg, 0);
    v27.bUseEncrypt = v12 != 0;
    if (v12)
        v27.ivSeed = Beyond::Byte::ByteBufStream::ReadLong(cfg, 0);
    // ...
    if (!Beyond::VFS::VirtualFileSystem::IsUseStringFileNameInfo(Byte, v27.bIsDirectFileName, 0))
        goto LABEL_29;
    if (!IFix::WrappersManagerImpl::IsPatched(8, 0))
    {
        currentIdx_1 = cfg->currentIdx;
        goto LABEL_25;
    }
    // ...
LABEL_33:
    sub_7FFD15783430();
    // ...
LABEL_25:
    cfg->currentIdx = currentIdx;
    UTF8 = Beyond::Byte::ByteBufStream::ReadUTF8(cfg, 0);
    if (UTF8 && UTF8->fields._stringLength)
    {
        m_fileName.handle = 0;
        System::Runtime::InteropServices::GCHandle::GCHandle(&m_fileName, UTF8, 0);
        v27.m_fileName = m_fileName;
    }
    cfg->currentIdx = currentIdx_1;
LABEL_29:
    v16 = *&v27.ivSeed;
    *&retstr->m_fileName.handle = *&v27.m_fileName.handle;
    v17 = *&v27.offset;
    retstr->fileChunkMD5Name = fileChunkMD5Name_1;
    retstr->fileDataMD5 = fileDataMD5_1;
    *&retstr->offset = v17;
    *&retstr->ivSeed = v16;
    return retstr;
}
```

这个函数从一个字节流中读取数据，并填充到 `FVFBlockFileInfo` 结构体的各个字段中。

可以整理出 `FVFBlockFileInfo` 的定义：

```cpp
struct FVFBlockFileInfo
{
    String fileName; // int16 (length) + n Bytes (content)
    __int64 fileNameHash;
    UInt128 fileChunkMD5Name;
    UInt128 fileDataMD5;
    __int64 offset;
    __int64 len;
    byte blockType;
    bool bUseEncrypt; // non-zero byte is true
    __int64 ivSeed;
};
```

### 负责 BLC 数据解密的函数

实际上，传入以上函数的字节数据都是已经被解密过的。为此，我们需要从第一个函数向上追溯，找到解密数据的函数。代码如下所示：

```cpp
// VFBlockMainInfo DecryptCreateBlockGroupInfo(Byte[], String)
VFBlockMainInfo *Beyond::VFS::VFSUtils::DecryptCreateBlockGroupInfo(
    Byte__Array *convertedBytes,
    String *inputPath,
    MethodInfo *method)
{
    struct VFSDefine__Class *_ZN8TypeInfo6Beyond3VFS9VFSDefineE;
    unsigned int BLOCK_HEAD_LEN, VFS_PROTO_VERSION_1;
    struct BitConverter__Class *_ZN8TypeInfo6System12BitConverterE;
    unsigned __int64 KEY_LEN;
    __int64 v12;
    void *v13;
    char *_value, v26;
    int32_t length, VFS_PROTO_VERSION;
    struct MethodInfo *_ZN10MethodInfo6System4SpanIhE11op_ImplicitEN6System4SpanIhEE;
    XXE1 *v19, *v20;
    Span_1_Byte_ nonce_, key_, v29;
    // ...

    // ...
    if (IFix::WrappersManagerImpl::IsPatched(1202, 0))
    {
        // ...
    }
    else
    {
        if (!convertedBytes)
            return 0;
        _ZN8TypeInfo6Beyond3VFS9VFSDefineE = TypeInfo::Beyond::VFS::VFSDefine;
        if (!TypeInfo::Beyond::VFS::VFSDefine->_1.cctor_finished_or_no_cctor)
        {
            // ...
            _ZN8TypeInfo6Beyond3VFS9VFSDefineE = TypeInfo::Beyond::VFS::VFSDefine;
        }
        if (SLODWORD(convertedBytes->max_length) > _ZN8TypeInfo6Beyond3VFS9VFSDefineE->static_fields->VFS_VFB_HEAD_LEN)
        {
            // ...
            BLOCK_HEAD_LEN = TypeInfo::Beyond::VFS::VFSDefine->static_fields->BLOCK_HEAD_LEN;
            // ...
            VFS_PROTO_VERSION_1 = *convertedBytes->vector;
            _ZN8TypeInfo6System12BitConverterE = TypeInfo::System::BitConverter;
            // ...
            if (!_ZN8TypeInfo6System12BitConverterE->static_fields->IsLittleEndian)
                VFS_PROTO_VERSION_1 = _byteswap_ulong(VFS_PROTO_VERSION_1);
            if (VFS_PROTO_VERSION_1 == TypeInfo::Beyond::VFS::VFSDefine->static_fields->VFS_PROTO_VERSION)
            {
                // ...
                KEY_LEN = TypeInfo::Beyond::VFS::VFSDefine->static_fields->KEY_LEN;
                if (KEY_LEN)
                {
                    v12 = KEY_LEN + 15;
                    if (KEY_LEN + 15 < KEY_LEN)
                        v12 = 0xFFFFFFFFFFFFFF0LL;
                    v13 = alloca(v12 & 0xFFFFFFFFFFFFFFF0uLL);
                    _value = &v26;
                }
                else
                {
                    _value = 0;
                }
                sub_7FFD1579B400(_value, 0, KEY_LEN);
                *(&nonce_._length + 1) = 0;
                // ...
                nonce_._pointer._value = _value;
                nonce_._length = KEY_LEN;
                // ...
                key_ = nonce_;
                nonce_ = *Beyond::VFS::VirtualFileSystem::GetCommonChachaKeyBs(&v29, &key_, 0);
                length = nonce_._length;
                *(&key_._length + 1) = 0;
                _ZN10MethodInfo6System4SpanIhE11op_ImplicitEN6System4SpanIhEE = MethodInfo::System::Span<unsigned char>::op_Implicit;
                // ...
                key_._pointer._value = nonce_._pointer._value;
                key_._length = length;
                *(&nonce_._length + 1) = 0;
                // ...
                nonce_._pointer._value = convertedBytes->vector;
                nonce_._length = BLOCK_HEAD_LEN;
                v19 = sub_7FFD156E1D80(TypeInfo::Beyond::XXEnc::XXE1);
                v20 = v19;
                if (!v19)
                    sub_7FFD15783430();
                Beyond::XXEnc::XXE1::XXE1(v19, &key_, &nonce_, 1u, 0);
                Beyond::XXEnc::XXE1::WorkBytes(
                    v20,
                    convertedBytes,
                    TypeInfo::Beyond::VFS::VFSDefine->static_fields->BLOCK_HEAD_LEN,
                    convertedBytes,
                    TypeInfo::Beyond::VFS::VFSDefine->static_fields->BLOCK_HEAD_LEN,
                    LODWORD(convertedBytes->max_length) - TypeInfo::Beyond::VFS::VFSDefine->static_fields->BLOCK_HEAD_LEN,
                    0);
                return Beyond::VFS::VFBlockMainInfo::ReadFromByteBuf(
                    convertedBytes,
                    TypeInfo::Beyond::VFS::VFSDefine->static_fields->BLOCK_HEAD_LEN,
                    0);
            }
            else
            {
                // ...
                return 0;
            }
        }
        else
        {
            return 0;
        }
    }
}
```

容易从代码中发现，BCL 文件使用了 [ChaCha20 加密算法](https://datatracker.ietf.org/doc/html/rfc7539)。解密函数 `Beyond::XXEnc::XXE1::XXE1` 需要输入 4 个参数：

- **input**（加密数据），整个文件的数据；
- **key**（解密密钥），通过 `Beyond::VFS::VirtualFileSystem::GetCommonChachaKeyBs` 函数获取；
- **nonce**（有时称作 iv），是数据的前 `KEY_LEN` Bytes，查找发现 `KEY_LEN` 常量的值是 12；
- **count**（初始计数），固定是 1。

因此，关键是要找到**密钥**。

### 获取 Chacha20 密钥

在阅读 `GetCommonChachaKeyBs` 函数的代码后，我发现生成密钥的算法过于复杂，涉及多种拼接和转换操作。因此我们选择通过**动态分析**的方式，使用调试器在运行时直接获取密钥。

我们不需要阅读 `GetCommonChachaKeyBs` 函数具体的密钥运算逻辑，我们只需要阅读它的返回语句附近的片段，如下所示：

```cpp
// Span`1[Byte] GetCommonChachaKeyBs(Span`1[Byte])
Span_1_Byte_ *Beyond::VFS::VirtualFileSystem::GetCommonChachaKeyBs(
    Span_1_Byte_ *__return_ptr retstr,
    Span_1_Byte_ *keyBuffer,
    MethodInfo *method)
{
    Span_1_Byte_ v28;
    // ...

    // some complicated operations to generate the key

    // ...
    v28 = *keyBuffer;
    *retstr = v28;
    return retstr;
}
```

我们可以发现，该函数返回的是变量 `v28`，而 `v28` 的值正是传入的参数 `keyBuffer`。因此，我们只需要在调试器中设置断点，查看 `v28` 的内容即可。具体如何绕过《终末地》的反调试机制，可以参考前面的章节（替换可执行文件），这里不再重复。

断点命中后，我们发现 `v28` 指向一个长度为 32 Bytes 的数据区域，其内容如下所示：

```
Stack[00004058]:00000009A64EEBF0 db 0E9h
Stack[00004058]:00000009A64EEBF1 db  5Bh ; [
Stack[00004058]:00000009A64EEBF2 db  31h ; 1
Stack[00004058]:00000009A64EEBF3 db  7Ah ; z
Stack[00004058]:00000009A64EEBF4 db 0C4h
Stack[00004058]:00000009A64EEBF5 db 0F8h
Stack[00004058]:00000009A64EEBF6 db  28h ; (
Stack[00004058]:00000009A64EEBF7 db  56h ; V
Stack[00004058]:00000009A64EEBF8 db  9Dh
Stack[00004058]:00000009A64EEBF9 db  23h ; #
Stack[00004058]:00000009A64EEBFA db 0A8h
Stack[00004058]:00000009A64EEBFB db  6Bh ; k
Stack[00004058]:00000009A64EEBFC db 0F2h
Stack[00004058]:00000009A64EEBFD db  71h ; q
Stack[00004058]:00000009A64EEBFE db 0DCh
Stack[00004058]:00000009A64EEBFF db 0B5h
Stack[00004058]:00000009A64EEC00 db  3Eh ; >
Stack[00004058]:00000009A64EEC01 db  84h
Stack[00004058]:00000009A64EEC02 db  6Fh ; o
Stack[00004058]:00000009A64EEC03 db 0A7h
Stack[00004058]:00000009A64EEC04 db  5Ch ; \
Stack[00004058]:00000009A64EEC05 db  92h
Stack[00004058]:00000009A64EEC06 db  4Dh ; M
Stack[00004058]:00000009A64EEC07 db  67h ; g
Stack[00004058]:00000009A64EEC08 db  1Dh
Stack[00004058]:00000009A64EEC09 db 0BAh
Stack[00004058]:00000009A64EEC0A db  8Eh
Stack[00004058]:00000009A64EEC0B db  38h ; 8
Stack[00004058]:00000009A64EEC0C db 0F4h
Stack[00004058]:00000009A64EEC0D db 0CAh
Stack[00004058]:00000009A64EEC0E db  52h ; R
Stack[00004058]:00000009A64EEC0F db 0E1h
```

我们可以将其整理为十六进制字节数组：

```cpp
unsigned char key[32] = {
  0xE9, 0x5B, 0x31, 0x7A, 0xC4, 0xF8, 0x28, 0x56,
  0x9D, 0x23, 0xA8, 0x6B, 0xF2, 0x71, 0xDC, 0xB5,
  0x3E, 0x84, 0x6F, 0xA7, 0x5C, 0x92, 0x4D, 0x67,
  0x1D, 0xBA, 0x8E, 0x38, 0xF4, 0xCA, 0x52, 0xE1,
};
```

或者 Base64 字符串格式：

```txt
6VsxesT4KFadI6hr8nHctT6Eb6dckk1nHbqOOPTKUuE=
```

## 完成 VFS 资源解析

### 解析 BLC 文件

至此，我们已经掌握了 BLC 文件的解密方法和数据结构定义，接下来我们就可以实现一个简单的 BLC 解析工具。解析工具的具体代码就在此省略了。

此外，在实际操作过程中发现一些细节问题：

1. 解密后的 BLC 文件的 `version` 后面（也就是 `groupCfgName` 前面）还存在 12 Bytes 的未知数据，在解析时应该跳过；
2. `groupCfgHashName` 的长度是 8 Bytes，但是它的有效数据只有 4 Bytes，因此后面 4 Bytes 可以跳过；
3. 如果你正在使用 Python 的 [pycryptodome](https://github.com/Legrandin/pycryptodome) 提供的 Chacha20 解密函数来进行解密，你会发现它的解密函数并没有传入 counter 这个参数（在它的实现里面，counter 参数是固定为 0 的）。根据 Chacha20 的原理，我们只需要将密钥流跳过 64 Bytes 即可等效地实现 counter 参数值为 1 的效果（示例代码如下）。
   ```python
   from Crypto.Cipher import ChaCha20

   CHACHA_KEY = b"\xe9[1z\xc4\xf8(V\x9d#\xa8k\xf2q\xdc\xb5>\x84o\xa7\\\x92Mg\x1d\xba\x8e8\xf4\xcaR\xe1"
   # nonce = ...
   # data = ...

   cipher = ChaCha20.new(key=CHACHA_KEY, nonce=nonce)
   cipher.seek(64)
   decrypted_data = cipher.decrypt(data)
   ```

最终，我成功对 19 个 BLC 文件进行了解析。以下是其中一个已解析的 BLC 文件的 `VFBlockMainInfo` JSON 序列化示例：

```json
{
  "version": 3,
  "groupCfgName": "BundleManifest",
  "groupCfgHashName": 532667676,
  "groupFileInfoNum": 1,
  "groupChunksLength": 167727212,
  "blockType": 3,
  "allChunks": [
    {
      "md5Name": 165989112415202324370777827216542827635,
      "contentMd5": 247380710951872531186795803451203924162,
      "length": 167727212,
      "blockType": 3,
      "files": [
        {
          "fileName": "Data/Bundles/Windows/manifest.hgmmap",
          "fileNameHash": 4361581287989299209,
          "fileChunkMd5Name": 165989112415202324370777827216542827635,
          "fileDataMd5": 247380710951872531186795803451203924162,
          "offset": 0,
          "len": 167727212,
          "blockType": 3,
          "bUseEncrypt": true,
          "ivSeed": 285540354
        }
      ]
    }
  ]
}
```

### 理解 BLC 的数据结构

以下是使用 Nano Banana 绘制的 BLC 数据结构图，可供参考：

![Arknights Endfield CBT3 BLC Structure](./BLC.jpg)

为了方便起见，我对某些哈希字段的读取方式进行了如下调整：

1. `groupCfgHashName` 改为大端读取 4 Bytes 之后转化为十六进制字符串（随后跳过 4 Bytes）；
2. `md5Name`、`contentMd5`、`fileChunkMd5Name` 和 `fileDataMd5` 都改为大端读取 16 Bytes 之后转化为十六进制字符串；
3. `fileNameHash` 改为大端读取 8 Bytes 之后转化为十六进制字符串。

调整后，我们可以得到类似这样的数据：

```json
{
  "version": 3,
  "groupCfgName": "BundleManifest",
  "groupCfgHashName": "1CDDBF1F",
  "groupFileInfoNum": 1,
  "groupChunksLength": 167727212,
  "blockType": 3,
  "allChunks": [
    {
      "md5Name": "7384B24D7C4BE3D6FE6D84A01757E07C",
      "contentMd5": "C23092E89D51DFBEFB661D36B9CA1BBA",
      "length": 167727212,
      "blockType": 3,
      "files": [
        {
          "fileName": "Data/Bundles/Windows/manifest.hgmmap",
          "fileNameHash": "09C897A11273873C",
          "fileChunkMd5Name": "7384B24D7C4BE3D6FE6D84A01757E07C",
          "fileDataMd5": "C23092E89D51DFBEFB661D36B9CA1BBA",
          "offset": 0,
          "len": 167727212,
          "blockType": 3,
          "bUseEncrypt": true,
          "ivSeed": 285540354
        }
      ]
    }
  ]
}
```

经观察，可以发现以下规律：

1. `groupCfgName` 正是 BLC 文件的名称，也是 BLC 文件所在的文件夹的名称；
2. `md5Name` 和 `fileChunkMd5Name` 正是每个 CHK 文件的名称；
3. `blockType`、`groupCfgName` 和 `groupCfgHashName` 三者的值是**有对应关系的**。

下表列出了各个 BLC 文件中的 `blockType` 等字段的关系：

| `blockType` | `groupCfgName`    | `groupCfgHashName` | 虚拟文件格式              | `bUseEncrypt` |
| ----------- | ----------------- | ------------------ | ------------------------- | ------------- |
| 1           | InitAudio         | `07A1BB91`         | AK Package (`.pck`)       | × `false`     |
| 2           | InitBundle        | `0CE8FA57`         | Asset Bundle (`.ab`)      | × `false`     |
| 3           | BundleManifest    | `1CDDBF1F`         | Manifest Map (`.hgmmap`)  | ✓ `true`      |
| 5           | InitialExtendData | `3C9D9D2D`         | Binary Data (`.bin`)      | × `false`     |
| 11          | Audio             | `24ED34CF`         | AK Package (`.pck`)       | × `false`     |
| 12          | Bundle            | `7064D8E2`         | Asset Bundle (`.ab`)      | × `false`     |
| 13          | DynamicStreaming  | `23D53F5D`         | Binary Data (`.bytes`)    | × `false`     |
| 14          | Table             | `42A8FCA6`         | Binary Data (`.bytes`)    | ✓ `true`      |
| 15          | Video             | `55FC21C6`         | Criware USM File (`.usm`) | × `false`     |
| 16          | IV                | `A63D7E6A`         | Binary Data (`.bytes`)    | × `false`     |
| 17          | Streaming         | `C3442D43`         | Binary Data (`.bytes`)    | × `false`     |
| 18          | JsonData          | `775A31D1`         | JSON Data (`.json`)       | ✓ `true`      |
| 19          | Lua               | `19E3AE45`         | Lua Script (`.lua`)       | ✓ `true`      |
| 21          | IFixPatchOut      | `DAFE52C9`         | *No files*                | × `false`     |
| 22          | ExtendData        | `D6E622F7`         | Binary Data (`.bin`)      | × `false`     |
| 30          | AudioChinese      | `E1E7D7CE`         | AK Package (`.pck`)       | × `false`     |
| 31          | AudioEnglish      | `A31457D0`         | AK Package (`.pck`)       | × `false`     |
| 32          | AudioJapanese     | `F668D4EE`         | AK Package (`.pck`)       | × `false`     |
| 33          | AudioKorean       | `E9D31017`         | AK Package (`.pck`)       | × `false`     |

根据 BLC 文件中记录的 `offset` 和 `len`，我们就可以从 CHK 文件中提取出每个虚拟文件的原始数据了，然后根据文件格式进行后续处理。

需要注意的是，这里的 `bUseEncrypt` 字段并不表示虚拟文件的原本内容是否被加密，而是表示该文件在 CHK 文件中是否被**外层加密**。通常而言，如果虚拟文件本身是非二进制格式（例如 `.json` 和 `.lua`），那么它在 CHK 文件中通常会被外层加密；而如果虚拟文件本身已经是加密或压缩后的格式（例如 `.ab` 和 `.pck`），那么它在 CHK 文件中通常不会被外层加密。

### 深入探究 CHK 的外层加密

为了将外层加密后的虚拟文件进行解密，我们可以在 `GameAssembly.dll` 中搜索与 `bUseEncrypt` 相关的代码片段，最终排查到以下函数：

```cpp
// FVFSTrackedLowIOHandle(FVFBlockFileInfo&, Boolean)
void Beyond::VFS::FVFSTrackedLowIOHandle::FVFSTrackedLowIOHandle(
    FVFSTrackedLowIOHandle *this,
    FVFBlockFileInfo *info,
    bool async,
    MethodInfo *method)
{
    String *ChunkFileRedirectedLoaderPath;
    bool isEnc;
    int64_t len, offset, ivSeed;
    struct VFSDefine__Class *_ZN8TypeInfo6Beyond3VFS9VFSDefineE;
    unsigned __int64 BLOCK_HEAD_LEN, KEY_LEN;
    Span_1_Byte_ *p_nonce, *p_key, *CommonChachaKeyBs, key_1, key_ivSeed_1, v63, key, key_ivSeed, v71, v73;
    struct VFSDefine__StaticFields *static_fields;
    ReadOnlySpan_1_Byte_ nonce__1, nonce__2, key__3, nonce_, key__1;
    XXE1 *m_xxe1_1;
    FVFSUntrackedLowIOReadHandle v67;

    // ...
    ChunkFileRedirectedLoaderPath = Beyond::VFS::FVFBlockFileInfo::GetChunkFileRedirectedLoaderPath(info, 0);
    // ...
    if (ChunkFileRedirectedLoaderPath && ChunkFileRedirectedLoaderPath->fields._stringLength)
    {
        isEnc = info->bUseEncrypt;
        len = info->len;
        offset = info->offset;
        memset(&v67, 0, sizeof(v67));
        Beyond::VFS::FVFSUntrackedLowIOReadHandle::FVFSUntrackedLowIOReadHandle(
            &v67,
            ChunkFileRedirectedLoaderPath,
            offset,
            len,
            async,
            isEnc,
            0);
        // ...
        _ZN8TypeInfo6Beyond3VFS9VFSDefineE = TypeInfo::Beyond::VFS::VFSDefine;
        // ...
        BLOCK_HEAD_LEN = _ZN8TypeInfo6Beyond3VFS9VFSDefineE->static_fields->BLOCK_HEAD_LEN;
        if (BLOCK_HEAD_LEN)
        {
            // ...
            p_nonce = &key_1;
        }
        else
        {
            p_nonce = 0;
        }
        sub_7FFCE0DDB400(p_nonce, 0, BLOCK_HEAD_LEN);
        // ...
        if (BLOCK_HEAD_LEN < 4)
            goto LABEL_69;
        *(&key_1._length + 1) = 0;
        // ...
        key_1._pointer._value = p_nonce;
        key_1._length = 4;
        static_fields = TypeInfo::Beyond::VFS::VFSDefine->static_fields;
        key = key_1;
        sub_7FFCE31DD900(&key, static_fields->VFS_PROTO_VERSION);
        if ((BLOCK_HEAD_LEN - 4) < 8)
            goto LABEL_69;
        *(&key_ivSeed_1._length + 1) = 0;
        // ...
        ivSeed = info->ivSeed;
        key_ivSeed_1._pointer._value = &p_nonce->_pointer._value + 4;
        key_ivSeed_1._length = 8;
        key_ivSeed = key_ivSeed_1;
        sub_7FFCE31DD9E0(&key_ivSeed, ivSeed);
        KEY_LEN = TypeInfo::Beyond::VFS::VFSDefine->static_fields->KEY_LEN;
        if (KEY_LEN)
        {
            // ...
            p_key = &key_1;
        }
        else
        {
            p_key = 0;
        }
        sub_7FFCE0DDB400(p_key, 0, KEY_LEN);
        *(&v63._length + 1) = 0;
        // ...
        if ((KEY_LEN & 0x80000000) != 0LL)
        LABEL_69:
            System::ThrowHelper::ThrowArgumentOutOfRangeException(0);
        v63._pointer._value = p_key;
        v63._length = KEY_LEN;
        // ...
        v71 = v63;
        CommonChachaKeyBs = Beyond::VFS::VirtualFileSystem::GetCommonChachaKeyBs(&v73, &v71, 0);
        // ...
        *(&key__3._length + 1) = 0;
        nonce__1 = *CommonChachaKeyBs;
        // ...
        nonce_ = nonce__1;
        // ...
        key__3._pointer._value = nonce_._pointer._value;
        // ...
        key__3._length = nonce__1._length;
        *(&nonce__2._length + 1) = 0;
        // ...
        nonce__2._pointer._value = p_nonce;
        nonce__2._length = BLOCK_HEAD_LEN;
        m_xxe1_1 = sub_7FFCE0D21D80(TypeInfo::Beyond::XXEnc::XXE1);
        // ...
        if (m_xxe1_1)
        {
            nonce_ = nonce__2;
            key__1 = key__3;
            Beyond::XXEnc::XXE1::XXE1(m_xxe1_1, &key__1, &nonce_, 1u, 0);
            // ...
            return;
        }
    LABEL_70:
        sub_7FFCE0DC3430();
    }
    // ...
}
```

可以发现，外层加密的解密过程与 BLC 的解密过程非常类似，依然是使用 ChaCha20 算法进行解密。相同之处是，它们都使用了同一个密钥（通过 `GetCommonChachaKeyBs` 函数获取）；不同之处在于，外层加密的解密所需的 **nonce** 需要通过以下方式来拼接：

- 前 4 Bytes 是将 BLC 文件的 `version`（代码中的 `VFS_PROTO_VERSION` 常量）进行转换后的结果（进一步分析显示，这种转换是将 `version` 转换为小端序 4 Bytes 字节）；
- 接下来的 8 Bytes 是将虚拟文件的 `ivSeed` 进行转换后的结果（进一步分析显示，这种转换也是小端序转换）；

~~CHK 解密，轻而易举啊。~~ 完美！根据这些信息，就可以顺利地将所有虚拟文件都还原出来了。

### 下一步

我很快地将各类虚拟文件进行了提取，并做了简单的分析：

1. `.usm` 文件未见修改，可以直接用相关工具进行解码，得到视频；
2. `.pck` 文件不是常规的格式，且极有可能本身被加密；
3. `.ab` 文件不是常规的格式，且极有可能本身被加密；
4. BundleManifest 类别只有一个文件 `manifest.hgmmap`，是自定义格式，推测是某种索引文件；
5. DynamicStreaming 和 Streaming 类别可能与某些高性能的流式加载有关，包括但不限于游戏场景的加载；
6. Table 类别可能与游戏数据表有关，是自定义格式；
7. IV 类别的全称是 Irradiance Volume，与光照有关；
8. JsonData 类别的文件存疑；
9. Lua 类别的文件存储着 Base64 字符串，且可能被加密；
10. ExtendData 类别只有一个文件 `InitStringPathHash.bin`，是自定义格式。

需要注意的是，本文是针对《终末地》**CBT3** 进行的分析，而游戏正式版的资源存储逻辑可能会与此不同，因此需要注意以上信息的时效性。

我们可以发现，鹰角网络对《终末地》CBT3 的资源保护机制做得非常完善——不仅对游戏文件做了虚拟化，还对各种类别的游戏资源都做了**不同的**格式自定义或加密处理，甚至连虚拟文件系统本身（BLC 与 CHK 文件）都设有处心积虑的加密，极大地增加了逆向工程的难度。

下一步，我会优先想要研究 `.pck` 和 `.ab` 文件的加密方式（因为包含主要的音频和美术资源）。

至此，本文就先告一段落，感谢阅读！也感谢 UnityPy 社区的各位朋友提供的帮助！
