# 什么是 FV、FFS、FD？UEFI 固件的打包格式

> 🔥 UEFI/BSP 开发系列第 026 篇 | 难度：⭐⭐ 进阶入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们搞明白了 GUID——UEFI 世界的身份证号。

现在你已经知道 UEFI 固件是由几百个模块组成的，每个模块编译出来是一个 `.efi` 文件。但问题来了：

> "这几百个 .efi 文件最终是怎么打包成一个完整的 BIOS 镜像，烧到主板上的 SPI Flash 芯片里的？"

答案涉及三个关键概念：**FD**、**FV**、**FFS**。

它们的关系就像一个三层套娃：

```
FD（Flash Device）      → 整个 SPI Flash 芯片的镜像文件
  └── FV（Firmware Volume）  → 一个"逻辑分区"
        └── FFS File            → 分区里的"文件"（一个 .efi 模块）
              └── Section          → 文件里的"段"（代码段、数据段等）
```

今天就把这个套娃一层层拆开。

---

## 一、从你熟悉的概念说起

先用你熟悉的操作系统概念做类比：

```
操作系统世界：                    UEFI 固件世界：
├── 硬盘镜像（.img/.iso）        ├── FD（Flash Device Image）
│   ├── 分区 1（C盘）            │   ├── FV（Firmware Volume）PEI
│   │   ├── Windows\System32\    │   │   ├── FFS: PeiCore.efi
│   │   ├── Program Files\       │   │   ├── FFS: MemoryInit.efi
│   │   └── ...                  │   │   └── FFS: ...
│   ├── 分区 2（D盘）            │   ├── FV（Firmware Volume）DXE
│   │   └── ...                  │   │   ├── FFS: DxeCore.efi
│   └── 分区 3（EFI分区）        │   │   ├── FFS: PciBusDxe.efi
│       └── ...                  │   │   └── FFS: ...
└──                              │   └── NV Variable Store
                                 └──
```

| 操作系统概念 | UEFI 固件概念 | 说明 |
|-------------|-------------|------|
| 硬盘镜像 | FD（Flash Device） | 整块存储的完整镜像 |
| 分区 | FV（Firmware Volume） | 逻辑上的划分 |
| 文件系统（NTFS/ext4） | FFS（Firmware File System） | 组织文件的格式 |
| 文件 | FFS File | 一个固件模块 |
| 文件内容（代码/数据） | Section | 模块内的各个段 |

---

## 二、FD — Flash Device Image（闪存设备镜像）

FD 是**最外层的容器**——它代表整个 SPI Flash 芯片的完整内容。

```
一个典型的 16MB SPI Flash 的 FD 布局：

偏移地址        大小        内容
0x0000_0000    4MB        Intel ME Region（管理引擎固件）
0x0040_0000    64KB       Flash Descriptor（闪存描述符）
0x0041_0000    64KB       GbE Region（千兆网卡固件，可选）
0x0042_0000    ~12MB      BIOS Region
  ├── 0x0042_0000  2MB      FV_PEI（PEI 阶段的模块）
  ├── 0x0062_0000  8MB      FV_DXE（DXE 阶段的模块）
  ├── 0x00E2_0000  256KB    NV_VARIABLE（UEFI 变量存储）
  ├── 0x00E6_0000  256KB    FTW（容错写入备份区）
  └── 0x00EA_0000  ~1.4MB   FV_RECOVERY / FV_ADVANCED / ...
```

### FD 在 EDK2 中怎么定义？

在 `.FDF`（Flash Description File）文件中：

```ini
[FD.MyPlatform]
BaseAddress   = 0xFF000000    # FD 映射到 CPU 地址空间的基地址
Size          = 0x01000000    # 16 MB
ErasePolarity = 1             # 擦除后的值是 0xFF（全 1）
BlockSize     = 0x00010000    # Flash 块大小 = 64 KB

# 按偏移量定义各个区域
0x00000000|0x00400000         # 0~4MB: ME Region（不由 EDK2 管理）
0x00400000|0x00200000         # 4MB~6MB: FV_PEI
FV = FV_PEI

0x00600000|0x00800000         # 6MB~14MB: FV_DXE
FV = FV_DXE

0x00E00000|0x00040000         # 14MB~14.25MB: NV 变量
DATA = {
  # NV Variable Store Header...
}

0x00E40000|0x001C0000         # 14.25MB~16MB: 其他 FV
FV = FV_ADVANCED
```

> 💡 **FD = 整块 Flash 的"地图"**。它告诉构建系统：Flash 从哪到哪是什么内容。

---

## 三、FV — Firmware Volume（固件卷）

FV 是 FD 里的"逻辑分区"。每个 FV 是一个独立的容器，里面装着一组相关的固件模块。

### 为什么要分成多个 FV？

```
不分区（全部放一起）：
  FD = [PeiCore][MemInit][DxeCore][PciBus][Usb][Network][Setup]...
  问题：PEI 阶段要从几百个文件中找到自己的模块，效率低

分区后：
  FD = [FV_PEI: PeiCore|MemInit|...] [FV_DXE: DxeCore|PciBus|...] [NV_VAR]
  好处：PEI 阶段只扫描 FV_PEI，DXE 阶段只扫描 FV_DXE
```

常见的 FV 划分：

| FV 名称 | 内容 | 说明 |
|---------|------|------|
| FV_SECPEI / FV_PEI | SEC 和 PEI 阶段的模块 | 开机最先执行的代码 |
| FV_DXE / FV_MAIN | DXE 驱动和 BDS 模块 | 固件的"主体" |
| FV_ADVANCED | 高级功能（网络启动等） | 可选功能 |
| FV_FVRECOVERY | 恢复用的最小固件 | BIOS 损坏时的救命稻草 |
| NV_VARIABLE_STORE | UEFI 变量 | 上一篇讲过的内容 |

### FV 的内部结构

```
一个 FV 的二进制布局：

┌──────────────────────────────────────┐
│ FV Header（固件卷头部）                │
│   Signature: "_FVH"                  │
│   Length: 整个 FV 的大小              │
│   FileSystem GUID: FFS v2 的 GUID    │
│   Revision: 版本号                    │
│   Attributes: 属性标志                │
├──────────────────────────────────────┤
│ FFS File 1 (PeiCore.efi)            │
├──────────────────────────────────────┤
│ FFS File 2 (MemoryInit.efi)         │
├──────────────────────────────────────┤
│ FFS File 3 (...)                    │
├──────────────────────────────────────┤
│ ... 更多文件 ...                     │
├──────────────────────────────────────┤
│ Free Space (0xFF 填充)               │
└──────────────────────────────────────┘
```

### FV 在 .FDF 中的定义

```ini
[FV.FV_DXE]
FvAlignment        = 16         # 对齐要求
ERASE_POLARITY     = 1          # 擦除极性
MEMORY_MAPPED      = TRUE       # 可以内存映射访问
STICKY_WRITE       = TRUE
LOCK_CAP           = TRUE
LOCK_STATUS        = TRUE
WRITE_DISABLED_CAP = TRUE
WRITE_STATUS       = TRUE
WRITE_LOCK_CAP     = TRUE
WRITE_LOCK_STATUS  = TRUE
READ_DISABLED_CAP  = TRUE
READ_STATUS        = TRUE
READ_LOCK_CAP      = TRUE
READ_LOCK_STATUS   = TRUE

# 包含哪些模块
INF  MdeModulePkg/Core/Dxe/DxeMain.inf
INF  MdeModulePkg/Universal/PCD/Dxe/Pcd.inf
INF  MdeModulePkg/Bus/Pci/PciBusDxe/PciBusDxe.inf
INF  MdeModulePkg/Bus/Usb/UsbBusDxe/UsbBusDxe.inf
# ... 几十甚至几百个 INF
```

---

## 四、FFS — Firmware File System（固件文件系统）

FFS 是 FV 内部组织文件的格式——类似于硬盘上的 NTFS 或 ext4。

### FFS File 的结构

每个 FFS File 代表一个固件模块（.efi），它有自己的头部：

```c
typedef struct {
    EFI_GUID    Name;           // 文件的 GUID（唯一标识）
    UINT8       IntegrityCheck; // 完整性校验
    UINT8       Type;           // 文件类型
    UINT8       Attributes;     // 属性
    UINT8       Size[3];        // 文件大小（24 位，最大 16MB）
    UINT8       State;          // 文件状态
} EFI_FFS_FILE_HEADER;
```

### FFS File 的类型

| 类型值 | 名称 | 说明 |
|--------|------|------|
| 0x01 | RAW | 原始二进制数据 |
| 0x02 | FREEFORM | 自由格式（资源文件等） |
| 0x03 | SECURITY_CORE | SEC 阶段核心 |
| 0x04 | PEI_CORE | PEI 阶段核心 |
| 0x05 | DXE_CORE | DXE 阶段核心 |
| 0x06 | PEIM | PEI 模块 |
| 0x07 | DRIVER | DXE 驱动 |
| 0x08 | COMBINED_PEIM_DRIVER | PEI/DXE 双模块 |
| 0x09 | APPLICATION | UEFI 应用程序 |
| 0x0A | SMM | SMM 驱动 |
| 0x0B | FIRMWARE_VOLUME_IMAGE | 嵌套的 FV |

> 💡 注意最后一个——FV 里面可以嵌套 FV！这就是为什么有时候你用工具打开 BIOS 镜像，会看到"套娃"结构。

---

## 五、Section — 文件里的段

每个 FFS File 内部又分成若干个 **Section**（段）：

```
FFS File（例如 PciBusDxe.efi）
  ├── PE32 Section          → 可执行代码（.efi 本体）
  ├── UI Section            → 模块名称字符串（"PciBusDxe"）
  ├── VERSION Section       → 版本信息
  └── DEPEX Section         → 依赖表达式（先决条件）
```

### 常见的 Section 类型

| 类型 | 说明 | 内容 |
|------|------|------|
| PE32 | PE32+ 可执行文件 | .efi 代码本身 |
| TE | Terse Executable | 压缩版的 PE（PEI 阶段用） |
| DXE_DEPEX | DXE 依赖表达式 | 模块加载的先决条件 |
| PEI_DEPEX | PEI 依赖表达式 | 同上 |
| UI | 用户界面名 | 模块的可读名称 |
| VERSION | 版本信息 | 版本号字符串 |
| RAW | 原始数据 | 任意二进制数据 |
| COMPRESSION | 压缩段 | 包含被压缩的子 Section |
| GUID_DEFINED | GUID 定义的段 | 签名/加密的子 Section |

### 压缩和签名

为了节省 Flash 空间，FFS File 内的 Section 通常会被压缩：

```
未压缩：
  FFS File
    └── PE32 Section (800 KB)

压缩后：
  FFS File
    └── COMPRESSION Section
          └── PE32 Section (300 KB，压缩比 ~60%)
```

有些安全要求高的平台还会对 Section 做签名：

```
签名后：
  FFS File
    └── GUID_DEFINED Section (RSA2048/SHA256 签名)
          └── PE32 Section (签名保护的代码)
```

---

## 六、完整的套娃示意图

```
┌─────────────────────────────────────────────────────────────┐
│  FD（Flash Device Image）— 完整的 SPI Flash 镜像             │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  FV_PEI（Firmware Volume）                            │   │
│  │                                                      │   │
│  │  ┌─────────────────────────────────────────────┐     │   │
│  │  │  FFS File: PeiCore (GUID: xxx)               │     │   │
│  │  │  ├── TE Section (代码)                        │     │   │
│  │  │  ├── UI Section ("PeiCore")                   │     │   │
│  │  │  └── PEI_DEPEX Section (TRUE)                │     │   │
│  │  └─────────────────────────────────────────────┘     │   │
│  │  ┌─────────────────────────────────────────────┐     │   │
│  │  │  FFS File: MemoryInit (GUID: yyy)            │     │   │
│  │  │  ├── TE Section (代码)                        │     │   │
│  │  │  └── PEI_DEPEX Section (依赖条件)             │     │   │
│  │  └─────────────────────────────────────────────┘     │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  FV_DXE（Firmware Volume）                            │   │
│  │                                                      │   │
│  │  ┌─────────────────────────────────────────────┐     │   │
│  │  │  FFS File: DxeCore (GUID: aaa)               │     │   │
│  │  │  ├── COMPRESSION Section                      │     │   │
│  │  │  │   └── PE32 Section (代码)                  │     │   │
│  │  │  └── UI Section ("DxeCore")                   │     │   │
│  │  └─────────────────────────────────────────────┘     │   │
│  │  ┌─────────────────────────────────────────────┐     │   │
│  │  │  FFS File: PciBusDxe (GUID: bbb)             │     │   │
│  │  │  ├── COMPRESSION Section                      │     │   │
│  │  │  │   └── PE32 Section (代码)                  │     │   │
│  │  │  ├── UI Section ("PciBusDxe")                 │     │   │
│  │  │  └── DXE_DEPEX Section (需要 PCI Root Bridge)│     │   │
│  │  └─────────────────────────────────────────────┘     │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  NV Variable Store（UEFI 变量存储区）                  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  FTW Working / Spare（容错写入工作区）                  │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 七、动手看一看：用 UEFITool 打开真实 BIOS 镜像

理论讲了这么多，不如动手看看实物。下一篇（#022）我们会详细讲 UEFITool 的用法，这里先给你一个预览：

```
UEFITool 打开一个 BIOS 镜像后看到的树形结构：

▼ BIOS region (0x500000~0xFFFFFF)
  ▼ Padding (0x500000~0x5FFFFF)
  ▼ Volume (0x600000~0xDFFFFF) — 这就是 FV_DXE
    ▼ File: DxeCore {D6A2CB7F-6A18-4E2F-B43B-9920A733700A}
      ► PE32 image section
      ► UI section "DxeCore"
    ▼ File: PcdDxe {80CF7257-87AB-47F9-A3FE-D50B76D89541}
      ► PE32 image section
      ► DXE dependency section
      ► UI section "PcdDxe"
    ▼ File: PciBusDxe {93B80004-9FB3-11D4-9A3A-0090273FC14D}
      ► Compressed section
        ► PE32 image section
      ► DXE dependency section
      ► UI section "PciBusDxe"
    ... (几百个文件)
```

> 💡 如果你有兴趣，现在就可以去 GitHub 下载 UEFITool（https://github.com/LongSoft/UEFITool），然后从主板厂商网站下载一个 BIOS 更新包，直接用 UEFITool 打开看看。

---

## 八、构建过程：从源码到 FD

```
EDK2 构建流程（简化）：

源码文件 (.c / .h)
    ↓  编译
目标文件 (.obj)
    ↓  链接
EFI 文件 (.efi) — 一个个独立的模块
    ↓  GenFfs — 按 Section 打包
FFS 文件 (.ffs) — 加上 FFS 头部、UI、DEPEX
    ↓  GenFv — 多个 FFS 组装
固件卷 (.fv) — 一个 FV
    ↓  GenFd — 多个 FV + NV + Descriptor 组装
固件镜像 (.fd) — 完整的 Flash 镜像
    ↓  烧录
SPI Flash 芯片 — 焊在主板上
```

对应的 EDK2 工具：

| 工具 | 作用 | 输入 → 输出 |
|------|------|------------|
| `GenSec` | 生成 Section | .efi → .sec |
| `GenFfs` | 生成 FFS 文件 | .sec → .ffs |
| `GenFv` | 生成 FV | .ffs → .fv |
| `GenFd` | 生成 FD | .fv + layout → .fd |

> 💡 实际上你直接跑 `build` 命令，这些工具会被自动调用。但了解底层流程有助于你排查构建错误。

---

## 九、常见疑问

### Q1：FV 里面为什么还能嵌套 FV？

因为有些场景需要"FV 中的 FV"。比如：
- 整个 FV_DXE 可能被压缩后放在 FV_PEI_COMPRESS 里（节省空间）
- Intel FSP 二进制自带一个独立的 FV，被嵌入到平台的 FD 中

```
FV_MAIN
  └── FFS File (type=FIRMWARE_VOLUME_IMAGE)
        └── FV_DXE_COMPRESSED (嵌套的 FV)
              ├── DxeCore.ffs
              ├── PciBusDxe.ffs
              └── ...
```

### Q2：为什么 PEI 用 TE 格式而不是 PE32？

TE（Terse Executable）是 PE32 的精简版——去掉了 PE 头部的冗余信息，通常能节省几百字节。在 PEI 阶段，Cache 空间极其有限（还在 CAR 模式），每个字节都很珍贵。

### Q3：FFS 文件的最大大小是多少？

标准 FFS 文件头的 Size 字段是 24 位，最大 $2^{24}$ = 16 MB。对于超过 16 MB 的文件，使用扩展头（`EFI_FFS_FILE_HEADER2`），Size 字段扩展到 32 位。但实际上很少有单个模块超过 16 MB。

### Q4：UEFI 变量存储为什么不放在 FV 里？

虽然 NV Variable Store 在 Flash 上也有类似的头部结构，但它不是一个标准 FV——因为它需要频繁擦写（每次修改变量都要写 Flash），而 FV 是只读的。把变量区和代码区分开，是为了保护代码不被意外修改。

---

## 十、总结

```
UEFI 固件的打包层次：

FD（Flash Device Image）
  → 整个 SPI Flash 的镜像
  → 由 .FDF 文件定义布局
  → 包含多个 FV + NV 变量区 + 描述符
  
  FV（Firmware Volume）
    → 一个逻辑分区
    → 按启动阶段划分（PEI/DXE/Advanced）
    → 有自己的头部和文件系统
    
    FFS File（Firmware File System File）
      → 一个固件模块
      → 由 GUID 唯一标识
      → 有文件类型（PEIM/DRIVER/APPLICATION）
      
      Section
        → 文件内的数据段
        → PE32/TE（代码）、DEPEX（依赖）、UI（名称）
        → 支持压缩和签名
```

| 你以为的 BIOS 镜像 | 实际的 BIOS 镜像 |
|-------------------|-----------------|
| 一整坨二进制 | 精心组织的多层结构 |
| 随便拼在一起 | FD → FV → FFS → Section 四层套娃 |
| 全部同等重要 | 按启动阶段分区，各司其职 |
| 不可读 | 用 UEFITool 一目了然 |

---

下一篇：**#027 用 UEFITool 打开一个真实的 BIOS 镜像看看里面有什么**——理论讲够了，下一篇我们动手操作——从主板厂商下载一个 BIOS 文件，用 UEFITool 打开，亲眼看看 FD/FV/FFS/Section 长什么样。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
