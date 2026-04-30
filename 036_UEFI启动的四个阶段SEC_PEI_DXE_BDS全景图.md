# UEFI 启动的四个阶段：SEC→PEI→DXE→BDS 全景图

> 🔥 UEFI/BSP 开发系列第 036 篇 | 难度：⭐⭐ 进阶
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

在第 003 篇我们简单提过"开机后到底发生了什么"。那时候是科普向，点到为止。

现在你已经学了 Protocol、Handle、内存模型、变量、固件打包格式……有了这些基础，是时候把 UEFI 启动的**完整流程**摊开来看了。

UEFI 把从通电到操作系统接管的整个过程分成了**四个阶段**（其实是五个，但 TSL 阶段是操作系统的事，我们先不管）：

```
SEC → PEI → DXE → BDS → [操作系统]
```

每个阶段干的事不同，拥有的资源也不同。就像人的成长：

```
SEC  = 婴儿期（刚出生，啥都没有，连内存都用不了）
PEI  = 幼儿期（终于有了一点内存，开始初始化硬件）
DXE  = 青年期（满血状态，所有资源就位，Protocol 满天飞）
BDS  = 成年期（选择人生道路——启动哪个操作系统？）
```

---

## 一、全景图：一图胜千言

```
 电源开启
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│  SEC (Security Phase)                                     │
│  • CPU 从 Reset Vector 开始执行                           │
│  • 设置临时 RAM（CAR: Cache as RAM）                      │
│  • 验证 PEI Core 的完整性                                 │
│  • 切换到保护模式 / 64位模式                              │
│  时间：微秒级 | 内存：无（仅 CPU Cache 模拟的几十 KB）     │
└──────────────────────────────────────────────────────────┘
    │ 传递 PEI Core 入口地址 + 临时 RAM 信息
    ▼
┌──────────────────────────────────────────────────────────┐
│  PEI (Pre-EFI Initialization)                             │
│  • 初始化内存控制器 → 真正的 DRAM 可用了！                │
│  • 发现并初始化关键硬件（CPU/芯片组/内存）                │
│  • 通过 PPI (PEIM-to-PEIM Interface) 通信                 │
│  • 为 DXE 准备 HOB (Hand-Off Block) 列表                  │
│  时间：毫秒~百毫秒 | 内存：CAR → 真实 DRAM              │
└──────────────────────────────────────────────────────────┘
    │ 传递 HOB List（内存布局、平台信息）
    ▼
┌──────────────────────────────────────────────────────────┐
│  DXE (Driver Execution Environment)                       │
│  • DXE Core 初始化 Boot Services / Runtime Services       │
│  • 加载并执行所有 DXE 驱动（几十到几百个）                │
│  • 建立完整的 Protocol 数据库                             │
│  • 枚举所有硬件设备、初始化外设                           │
│  时间：秒级 | 内存：完整 DRAM                            │
└──────────────────────────────────────────────────────────┘
    │ 所有驱动加载完毕，设备就绪
    ▼
┌──────────────────────────────────────────────────────────┐
│  BDS (Boot Device Selection)                              │
│  • 读取启动顺序（UEFI 变量 BootOrder）                    │
│  • 连接必要的设备驱动                                     │
│  • 加载 OS Boot Loader（如 GRUB、Windows Boot Manager）   │
│  • 如果找不到 → 进入 UEFI Shell 或显示错误               │
│  时间：毫秒~秒 | 内存：完整 DRAM                        │
└──────────────────────────────────────────────────────────┘
    │ ExitBootServices() → 操作系统接管
    ▼
  操作系统内核启动
```

---

## 二、SEC 阶段：从虚无中苏醒

### 2.1 CPU Reset Vector

CPU 上电后做的第一件事是从一个**固定地址**取第一条指令：

| 架构 | Reset Vector 地址 |
|------|------------------|
| x86 (Real Mode) | `0xFFFFFFF0`（4GB - 16 字节） |
| ARM AArch64 | `0x00000000` 或可配置 |
| RISC-V | 由硬件决定 |

这个地址指向 Flash ROM 中的 SEC 模块代码。

### 2.2 CAR (Cache As RAM)

此时 DRAM 还没初始化（内存控制器还没配置），CPU 的 Cache 是唯一可用的"内存"。

SEC 做的一件关键事：**把 CPU Cache 配置成 No-Eviction Mode**，让它当临时 RAM 用。

```
正常模式：CPU Cache 是透明的缓存层
CAR 模式：CPU Cache 被锁住，当作一块 SRAM 用

┌──────────┐
│ CPU Core │
├──────────┤
│ L1 Cache │ ← 当作栈和数据区（几十 KB）
│ L2 Cache │ ← 可能也被用上
├──────────┤
│   ...    │
│  DRAM    │ ← 此时完全不可用（还没初始化）
└──────────┘
```

### 2.3 SEC 的输出

SEC 做完后，传递给 PEI：
- 临时 RAM 的位置和大小
- PEI Core 的入口地址
- 安全验证状态（如果支持 Secure Boot）

> 💡 SEC 代码通常很短（几百行汇编 + 少量 C），因为它能做的事太有限了。

---

## 三、PEI 阶段：初始化内存，建立基础

### 3.1 最重要的任务：初始化 DRAM

PEI 阶段最核心的一件事——**让内存控制器工作**，让系统有真正的 RAM 可用。

这件事看起来简单，实际上是整个 UEFI 启动中**最复杂**的部分之一：
- 要检测 DIMM 的 SPD 信息（什么规格、什么速度）
- 要配置内存控制器的几百个寄存器
- 要做内存训练（Memory Training）调整时序
- 要处理各种异常（坏内存条、不兼容的组合）

> 这就是为什么 Intel/AMD 的内存初始化代码通常是**闭源**的——太复杂、太核心。

### 3.2 PEIM 和 PPI

PEI 阶段的模块叫 **PEIM**（PEI Module），它们之间通过 **PPI**（PEIM-to-PEIM Interface）通信。

PPI 和 DXE 阶段的 Protocol 类似，也是函数指针表，但更轻量：

```c
// PPI 示例：内存发现通知
typedef struct {
    EFI_PEI_NOTIFY_PPI_CALLBACK  Notify;
} EFI_PEI_PERMANENT_MEMORY_INSTALLED_PPI;
```

PEI 常见的 PEIM 模块：

```
PlatformInit PEIM     → 配置芯片组基础寄存器
MemoryInit PEIM       → 初始化 DRAM（最关键！）
CpuInit PEIM          → 初始化 CPU（微码加载等）
SiliconInit PEIM      → 硅片级初始化
RecoveryPEIM          → 固件恢复支持
S3Resume PEIM         → S3 休眠唤醒支持
```

### 3.3 HOB (Hand-Off Block)

PEI 结束时，要把发现的所有信息"打包"传给 DXE，这个包裹就是 **HOB List**：

```
HOB List 内容：
├── Memory Allocation HOB（内存布局：哪些可用，哪些保留）
├── Resource Descriptor HOB（系统资源描述）
├── Firmware Volume HOB（固件卷位置）
├── CPU HOB（CPU 信息）
├── Platform Info HOB（平台私有数据）
└── End of HOB List
```

DXE Core 收到 HOB List 后，就知道整个系统的内存布局了。

---

## 四、DXE 阶段：满血状态，百花齐放

### 4.1 DXE 是 UEFI 的核心

DXE 阶段是整个 UEFI 固件中**代码量最大、功能最丰富**的阶段。你之前学的 Protocol、Handle、Boot Services、驱动模型——全部在这个阶段发挥作用。

DXE Core 的初始化流程：

```
1. 接收 HOB List
2. 初始化内存管理服务（AllocatePages / AllocatePool）
3. 初始化 Handle Database
4. 创建 EFI_BOOT_SERVICES 和 EFI_RUNTIME_SERVICES
5. 创建 EFI_SYSTEM_TABLE
6. 发现并调度所有 DXE 驱动
```

### 4.2 DXE Dispatcher（调度器）

DXE Core 里有一个调度器，它的工作是：

1. 扫描所有 Firmware Volume 中的 DXE 驱动
2. 检查每个驱动的依赖关系（Dependency Expression）
3. 按依赖顺序加载和执行驱动

```
依赖关系示例：
  USB 驱动 依赖 PCI Bus Protocol
  PCI Bus 驱动 依赖 PCI Root Bridge IO Protocol
  PCI Root Bridge 驱动 无依赖（最先加载）

执行顺序：
  PCI Root Bridge → PCI Bus → USB Host Controller → USB Bus → ...
```

### 4.3 DXE 驱动的类型

| 类型 | 特点 | 示例 |
|------|------|------|
| 早期 DXE 驱动 | 无依赖，最先加载 | CPU Arch Protocol、Timer |
| 总线驱动 | 枚举子设备 | PCI Bus、USB Bus |
| 设备驱动 | 驱动具体设备 | NVMe、SATA、网卡 |
| 服务驱动 | 提供系统服务 | Console、Variable、Security |
| Runtime 驱动 | OS 之后仍可用 | RTC、Variable（Flash） |

### 4.4 DXE 结束的标志

当 BDS 找到启动项并准备启动操作系统时，会调用 `ExitBootServices()`。这个调用是 DXE 的终点：

- Boot Services 被销毁（`AllocatePool`、`LocateProtocol` 等不再可用）
- 所有 Boot Services 类型的驱动被卸载
- 只有 Runtime Services 存活（`GetVariable`、`GetTime` 等）
- 中断被关闭
- 内存所有权移交给操作系统

---

## 五、BDS 阶段：选择启动目标

### 5.1 启动顺序

BDS 的核心工作是根据 UEFI 变量决定启动什么：

```
UEFI 变量 BootOrder: 0003, 0001, 0002, 0000
                      ↓
Boot0003 → "Windows Boot Manager" → \EFI\Microsoft\Boot\bootmgfw.efi
Boot0001 → "Ubuntu"              → \EFI\ubuntu\grubx64.efi
Boot0002 → "USB Drive"           → 可移动设备
Boot0000 → "UEFI Shell"          → 内置 Shell
```

BDS 按 BootOrder 顺序尝试，第一个成功的就启动。

### 5.2 BDS 的流程

```
1. 读取 BootOrder 变量
2. 遍历每个 BootXXXX 变量
3. 连接必要的设备驱动（Connect）
4. 加载 Boot Loader 的 EFI 文件
5. 调用 gBS->StartImage() 执行 Boot Loader
6. Boot Loader 加载 OS 内核
7. Boot Loader 调用 ExitBootServices()
8. 操作系统内核获得控制权
```

### 5.3 如果所有启动项都失败

- 如果固件内置了 UEFI Shell → 启动 Shell
- 如果设置了 PlatformRecovery → 进入恢复模式
- 否则 → 显示错误信息，系统挂起

---

## 六、四个阶段的资源对比

| | SEC | PEI | DXE | BDS |
|--|-----|-----|-----|-----|
| **内存** | 无（CAR 几十KB） | CAR → DRAM | 完整 DRAM | 完整 DRAM |
| **通信机制** | 无 | PPI | Protocol | Protocol |
| **代码大小** | 极小（几KB） | 中等（几十~几百KB） | 大（几MB） | 中等 |
| **典型模块数** | 1-2 | 5-20 | 50-200+ | 1-5 |
| **主要语言** | 汇编 + C | C | C | C |
| **Flash/RAM** | 从 Flash XIP | Flash XIP → RAM | RAM | RAM |
| **安全状态** | 建立 Root of Trust | 验证后续模块 | 可选验证 | 可选验证 |

> **XIP (Execute in Place)**：直接从 Flash 执行代码，不拷贝到 RAM。PEI 早期必须 XIP，因为 RAM 还没有。

---

## 七、用时间线看真实的启动过程

以一台典型的服务器为例（从通电到 OS 接管）：

```
时间轴（不精确，示意用）：

0 ms      ─── 电源稳定 ───
                │
0-5 ms    ─── SEC ───
              CPU Reset, CAR Setup
                │
5-200 ms  ─── PEI ───
              内存初始化（最耗时！）
              CPU 微码加载
              PCH/芯片组初始化
                │
200ms-3s  ─── DXE ───
              PCI 枚举
              USB 初始化
              存储设备就绪
              网络栈初始化
              Console 就绪（这时你才看到 Logo）
                │
3-5s      ─── BDS ───
              读取启动项
              加载 Boot Loader
              调用 ExitBootServices()
                │
5s+       ─── OS 启动 ───
```

> 💡 你在开机时看到主板 Logo 的时候，通常已经到了 DXE 阶段后期。SEC 和 PEI 太快了，你根本看不到。

---

## 八、常见疑问

### Q1：为什么要分这么多阶段，不能一步到位？

因为**资源是逐步获得的**。你不可能在没有内存的时候就跑复杂的驱动程序。分阶段是一种"量力而行"的设计：

- SEC：我只有 Cache，能做的就是初始化到一个基本状态
- PEI：我有了 Cache 当 RAM，那就去初始化真内存
- DXE：真内存有了，我可以大展拳脚了
- BDS：一切就绪，选个 OS 启动吧

### Q2：ARM 平台也是这四个阶段吗？

是的。UEFI 规范是架构无关的。ARM 平台也用 SEC→PEI→DXE→BDS 的流程，只是 SEC 阶段的 Reset Vector 地址和模式切换方式不同。

### Q3：有没有阶段可以跳过？

理论上 PEI 可以极度精简（如果内存由硬件自动初始化）。实际上 Intel/AMD/ARM 平台都完整走了四个阶段。

有些轻量级方案（如 coreboot + UEFI payload）会跳过前面的阶段，只用 DXE+BDS。

### Q4：哪个阶段是 BSP 工程师工作最多的地方？

- **芯片原厂 BSP 工程师**：PEI 阶段（内存训练、硅片初始化）
- **OEM/ODM BIOS 工程师**：DXE 和 BDS 阶段（驱动定制、Setup 界面）
- **安全固件工程师**：SEC 阶段（Root of Trust、Verified Boot）

---

## 九、总结

```
SEC  → "我能执行了"     （从 Reset Vector 启动，建立最小环境）
PEI  → "我有内存了"     （初始化 DRAM，建立基础硬件环境）
DXE  → "我什么都有了"   （完整的驱动框架，所有 Protocol 就位）
BDS  → "我要启动 OS 了" （选择并加载操作系统）
```

| 阶段 | 核心任务 | 产出 |
|------|---------|------|
| SEC | CPU 初始化 + CAR | 临时 RAM + PEI 入口 |
| PEI | DRAM 初始化 | HOB List（内存布局） |
| DXE | 全系统初始化 | 完整 Protocol 数据库 |
| BDS | 启动选择 | 加载并执行 OS Loader |

从下一篇开始，我们将**深入每个阶段**，看看里面具体干了什么、代码怎么写。

---

下一篇：**#037 SEC 阶段详解：CPU 的第一条指令从哪里来？**——看看 CPU 上电后那几微秒到底发生了什么。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
