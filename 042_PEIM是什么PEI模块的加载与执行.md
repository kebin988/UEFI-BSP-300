# PEIM 是什么？PEI 模块的加载与执行

> 🔥 UEFI/BSP 开发系列第 042 篇 | 难度：⭐⭐⭐ 中级
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们从全景视角看了 PEI 阶段。今天深入一个具体问题：**PEIM 模块到底是什么？怎么被发现、加载和执行的？**

如果说 DXE 阶段的"演员"是 DXE 驱动，那 PEI 阶段的"演员"就是 **PEIM (PEI Module)**。每个 PEIM 负责一项具体的初始化任务。

---

## 一、PEIM 是什么

PEIM = **PEI Module**，是 PEI 阶段的基本执行单元。

类比：

```
操作系统里：
  Application / Service → 完成特定功能

DXE 阶段：
  DXE Driver → 驱动硬件或提供服务

PEI 阶段：
  PEIM → 初始化硬件或提供 PEI 级服务
```

每个 PEIM 都是一个独立的二进制文件，存储在 Firmware Volume (FV) 中。

---

## 二、PEIM 的文件格式

PEIM 在 Flash 中以 **FFS (Firmware File System)** 格式存储：

```
Firmware Volume (FV)
├── FFS File: PEI Core          (Type = 0x04: PEI Core)
├── FFS File: PlatformInit PEIM (Type = 0x06: PEIM)
├── FFS File: MemoryInit PEIM   (Type = 0x06: PEIM)
├── FFS File: CpuInit PEIM      (Type = 0x06: PEIM)
└── FFS File: DXE Core          (Type = 0x05: DXE Core)
```

每个 PEIM 的 FFS 文件内部结构：

```
┌────────────────────────────────────┐
│  FFS File Header                   │
│  - File GUID (唯一标识)            │
│  - File Type = 0x06 (PEIM)        │
│  - File Size                       │
│  - State / Attributes             │
├────────────────────────────────────┤
│  Section: PE32 (或 TE)            │  ← 可执行代码
│  - 就是一个 PE/COFF 格式的可执行文件│
├────────────────────────────────────┤
│  Section: DXE_DEPEX (依赖表达式)   │  ← 告诉调度器何时执行
├────────────────────────────────────┤
│  Section: UI (用户界面名称)        │  ← 可选，如 "MemoryInit"
├────────────────────────────────────┤
│  Section: VERSION                  │  ← 可选，版本信息
└────────────────────────────────────┘
```

### TE (Terse Executable) 格式

PEI 阶段为了节省 Flash 空间，很多 PEIM 使用 **TE 格式**而不是完整 PE 格式：

```
PE/COFF 头：约 200+ 字节的头部信息
TE 头：只有 40 字节（去掉了 PE 中不需要的字段）

在 Flash 寸土寸金的 PEI 阶段，每个 PEIM 省 160 字节，
几十个 PEIM 加起来就省了好几 KB。
```

---

## 三、PEIM 的入口函数

每个 PEIM 都有一个标准的入口函数：

```c
/**
  PEIM 的入口点
  
  @param[in]  FileHandle    指向 PEIM 在 FV 中的文件句柄
  @param[in]  PeiServices   PEI Services 表指针

  @retval EFI_SUCCESS       初始化成功
**/
EFI_STATUS
EFIAPI
MyPeimEntryPoint(
    IN EFI_PEI_FILE_HANDLE   FileHandle,
    IN CONST EFI_PEI_SERVICES **PeiServices
)
{
    EFI_STATUS Status;
    
    // 做这个 PEIM 该做的事
    // 比如初始化某个硬件...
    
    // 可以安装 PPI 给其他 PEIM 使用
    Status = (*PeiServices)->InstallPpi(PeiServices, &mMyPpiList);
    
    return Status;
}
```

### INF 文件中的声明

```ini
# MyPeim.inf

[Defines]
  INF_VERSION    = 0x00010005
  BASE_NAME      = MyPeim
  FILE_GUID      = 12345678-ABCD-EF01-2345-678901234567
  MODULE_TYPE    = PEIM            # ← 声明这是 PEIM
  ENTRY_POINT    = MyPeimEntryPoint

[Sources]
  MyPeim.c

[Packages]
  MdePkg/MdePkg.dec

[LibraryClasses]
  PeimEntryPoint
  PeiServicesLib
  BaseMemoryLib

[Ppis]
  gMyOutputPpiGuid          # 本 PEIM 要安装的 PPI

[Depex]
  gEfiPeiMemoryDiscoveredPpiGuid  # 依赖：内存已初始化
```

---

## 四、PEIM 的依赖表达式 (DEPEX)

每个 PEIM 可以声明依赖——"我需要哪些 PPI 已经被安装了才能运行"。

### 4.1 基本语法

```ini
# INF 文件中
[Depex]
  gPpiAGuid AND gPpiBGuid   # 需要 PPI A 和 PPI B 都已安装

# 或者用 .depex 文件
[Depex]
  TRUE                       # 无依赖，随时可执行
```

### 4.2 DEPEX 操作符

| 操作符 | 含义 | 示例 |
|--------|------|------|
| `TRUE` | 无条件可执行 | 最早期的 PEIM |
| `AND` | 所有条件满足 | `A AND B` |
| `OR` | 任一条件满足 | `A OR B` |
| `NOT` | 条件不满足时 | `NOT A` |
| `BEFORE guid` | 在某 PEIM 之前执行 | 优先级控制 |
| `AFTER guid` | 在某 PEIM 之后执行 | 顺序控制 |

### 4.3 实际的依赖链

```
PlatformInit PEIM
  DEPEX: TRUE（最先执行，无依赖）
  安装: PlatformConfig PPI

CpuInit PEIM  
  DEPEX: gPlatformConfigPpiGuid（依赖 PlatformInit）
  安装: CpuInitDone PPI

MemoryInit PEIM
  DEPEX: gPlatformConfigPpiGuid AND gCpuInitDonePpiGuid
  安装: gEfiPeiMemoryDiscoveredPpiGuid（标志性 PPI：内存好了！）

SiliconInit PEIM
  DEPEX: gEfiPeiMemoryDiscoveredPpiGuid（依赖内存）
  安装: SiliconInitDone PPI
```

执行顺序被自动解析为：
```
PlatformInit → CpuInit → MemoryInit → SiliconInit
```

---

## 五、PEI Dispatcher 的调度过程

PEI Core 中的调度器按如下逻辑工作：

```c
// PEI Dispatcher 伪代码
VOID PeiDispatcher(VOID)
{
    BOOLEAN Dispatched;
    
    do {
        Dispatched = FALSE;
        
        // 遍历所有 FV 中的 PEIM
        for (UINTN i = 0; i < PeimCount; i++) {
            
            // 跳过已执行的
            if (Peims[i].AlreadyDispatched) continue;
            
            // 计算 DEPEX
            if (EvaluateDepex(Peims[i].Depex) == TRUE) {
                
                // 加载 PEIM
                LoadPeim(&Peims[i]);
                
                // 执行 PEIM 入口函数
                Status = Peims[i].EntryPoint(FileHandle, PeiServices);
                
                // 标记为已执行
                Peims[i].AlreadyDispatched = TRUE;
                Dispatched = TRUE;
                
                // PEIM 可能安装了新 PPI
                // → 其他 PEIM 的依赖可能变为满足
                // → 需要重新遍历
            }
        }
    } while (Dispatched);
    // 当没有新 PEIM 可以执行时，PEI 调度结束
}
```

### 关键点：

1. **多轮遍历**：每次有 PEIM 被执行（可能安装了新 PPI），就重新扫描
2. **依赖驱动**：不是按文件顺序，而是按依赖满足顺序
3. **直到稳定**：没有新 PEIM 可执行时停止

---

## 六、PEIM 的加载方式

### 6.1 Pre-Memory：XIP 执行

DRAM 初始化之前，PEIM 直接从 Flash 执行（XIP）：

```
Flash 中的 PEIM 代码
      │
      │ CPU 直接从 Flash 地址取指令执行
      ▼
CPU 执行 PEIM（速度慢，但不需要 RAM）
```

### 6.2 Post-Memory：Shadow 到 DRAM

DRAM 可用后，新加载的 PEIM 会被复制到 RAM（Shadow）：

```
Flash 中的 PEIM 代码
      │
      │ 复制到 DRAM
      ▼
DRAM 中的 PEIM 副本
      │
      │ CPU 从 DRAM 地址执行（快得多！）
      ▼
CPU 执行 PEIM（速度快）
```

PEI Core 判断逻辑：

```c
if (MemoryAvailable) {
    // DRAM 可用 → 先把 PEIM 复制到 RAM 再执行
    CopyMem(RamBuffer, FlashAddress, PeimSize);
    EntryPoint = RelocatePeim(RamBuffer);
} else {
    // DRAM 不可用 → XIP 执行
    EntryPoint = GetEntryPointFromFlash(FlashAddress);
}

EntryPoint(FileHandle, PeiServices);
```

---

## 七、PEIM 的生命周期

```
┌─────────────────────────────────────────┐
│           PEIM 生命周期                   │
├─────────────────────────────────────────┤
│                                         │
│  1. 存储在 Flash（FFS 文件）            │
│           ↓                             │
│  2. PEI Dispatcher 发现                 │
│           ↓                             │
│  3. 检查 DEPEX（依赖满足？）            │
│           ↓ YES                         │
│  4. 加载（XIP 或 Shadow 到 RAM）         │
│           ↓                             │
│  5. 调用入口函数 EntryPoint()           │
│           ↓                             │
│  6. PEIM 执行初始化任务                  │
│     - 初始化硬件                         │
│     - 安装 PPI                           │
│     - 创建 HOB                           │
│           ↓                             │
│  7. 返回 EFI_STATUS                     │
│           ↓                             │
│  8. PEI Core 标记为已执行               │
│     （PEIM 代码区域不再需要）            │
│                                         │
└─────────────────────────────────────────┘
```

**注意**：PEIM 执行完就"死了"——它的入口函数返回后不会再被调用。PEIM 的"遗产"是它安装的 PPI 和创建的 HOB。

---

## 八、常见的 PEIM 模块

| PEIM 名称 | 功能 | DEPEX |
|-----------|------|-------|
| **PlatformInitPreMem** | 芯片组最早期配置 | TRUE |
| **CpuMpPei** | CPU 多核初始化 | PlatformInit |
| **SiliconPolicyInitPreMem** | 硅片策略配置 | TRUE |
| **MemoryInit (MRC/FSP-M)** | DRAM 初始化 | SiliconPolicy |
| **SiliconInitPreMem** | 硅片 Pre-Memory 初始化 | MemoryDiscovered |
| **PlatformInitPostMem** | 平台 Post-Memory 配置 | MemoryDiscovered |
| **CapsulePei** | 固件更新胶囊处理 | MemoryDiscovered |
| **DxeIpl** | DXE 加载器 | 所有关键 PPI |

---

## 九、写一个最简单的 PEIM

```c
// SimplePeim.c
#include <PiPei.h>
#include <Library/DebugLib.h>
#include <Library/IoLib.h>

EFI_STATUS
EFIAPI
SimplePeimEntry(
    IN EFI_PEI_FILE_HANDLE   FileHandle,
    IN CONST EFI_PEI_SERVICES **PeiServices
)
{
    // Post Code 标记执行到这里
    IoWrite8(0x80, 0x42);
    
    DEBUG((DEBUG_INFO, "SimplePeim: Hello from PEI!\n"));
    
    // 做一些硬件初始化...
    // 比如配置一个 GPIO
    
    return EFI_SUCCESS;
}
```

```ini
# SimplePeim.inf
[Defines]
  INF_VERSION    = 0x00010005
  BASE_NAME      = SimplePeim
  FILE_GUID      = AABBCCDD-1234-5678-9012-AABBCCDDEEFF
  MODULE_TYPE    = PEIM
  ENTRY_POINT    = SimplePeimEntry

[Sources]
  SimplePeim.c

[Packages]
  MdePkg/MdePkg.dec

[LibraryClasses]
  PeimEntryPoint
  DebugLib
  IoLib

[Depex]
  TRUE    # 无依赖，尽早执行
```

---

## 十、总结

| 概念 | 说明 |
|------|------|
| **PEIM** | PEI 阶段的执行模块（类似 DXE Driver） |
| **存储格式** | FFS 文件中的 PE/TE 可执行文件 |
| **入口函数** | `EntryPoint(FileHandle, PeiServices)` |
| **依赖控制** | DEPEX 表达式（基于 PPI GUID） |
| **调度方式** | PEI Dispatcher 多轮依赖解析 |
| **执行方式** | Pre-Memory: XIP / Post-Memory: Shadow |
| **生命周期** | 执行一次就结束，遗产是 PPI 和 HOB |
| **TE 格式** | 精简版 PE 头，省 Flash 空间 |

> **一句话总结**：PEIM 就是 PEI 阶段的"一次性工人"——被叫到后干完活就走，留下的 PPI 和 HOB 才是持久贡献。

---

下一篇：**#043 PPI (PEI-to-PEI Interface)：PEI 阶段的通信机制**——PEIM 之间怎么互相发现和调用。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
