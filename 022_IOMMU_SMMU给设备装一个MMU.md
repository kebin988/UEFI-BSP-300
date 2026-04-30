# IOMMU/SMMU：给设备装一个 MMU

> 🔥 UEFI/BSP 开发系列第 022 篇 | 难度：⭐⭐⭐ 进阶
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们深入了 UEFI 中的 DMA 编程——`PciIo->Map()`、`Unmap()`、`AllocateBuffer()` 那一套。

你可能注意到了一个反复出现的词：**IOMMU**。Map() 内部会检查有没有 IOMMU，有 IOMMU 的平台地址转换方式完全不同。

但 IOMMU 到底是个什么东西？它和 CPU 的 MMU 有什么关系？在 ARM 上叫 SMMU，在 Intel 上叫 VT-d，它们是一回事吗？

今天一次讲清楚。

---

## 一、先回忆：CPU 的 MMU 是什么？

在讲 IOMMU 之前，先快速回忆 CPU 的 MMU（Memory Management Unit）：

```
CPU 视角：
  程序访问虚拟地址 0x7FFE0000
        ↓
  MMU 查页表（Page Table）
        ↓
  翻译成物理地址 0x80010000
        ↓
  访问物理内存
```

MMU 做了两件事：
1. **地址翻译**：虚拟地址 → 物理地址
2. **权限控制**：这个地址是否可读/可写/可执行

有了 MMU，每个进程都以为自己独占 4GB（32 位）或 256TB（64 位）的地址空间，进程之间互不干扰。

> 💡 MMU 就是 CPU 的"门卫"——谁能进哪个门，一查页表就知道。

---

## 二、问题来了：设备的 DMA 谁来管？

上一篇讲了，DMA 允许设备**绕过 CPU 直接访问内存**。这意味着：

```
没有 IOMMU 的情况：
  设备发起 DMA → 直接访问物理内存 → 想读哪就读哪！

  恶意设备：嘿嘿，我要读 0x80000000（操作系统内核代码）
  → 成功读取！没人拦我！
```

CPU 有 MMU 保护，每个进程只能访问自己的地址空间。但**设备没有 MMU**——设备的 DMA 访问直接用物理地址，想读哪就读哪。

这就像公司大楼：员工（进程）出入需要刷门禁卡（MMU），但快递员（DMA 设备）可以直接进任何办公室——显然不安全。

**IOMMU 就是给设备装的 MMU**——给快递员也发一张门禁卡。

---

## 三、IOMMU 是什么？

IOMMU（Input/Output Memory Management Unit）= **I/O 设备的内存管理单元**。

```
有 IOMMU 的情况：
  设备发起 DMA → 经过 IOMMU → 地址翻译 + 权限检查 → 访问物理内存

  恶意设备：我要读 0x80000000
  IOMMU：你没有这个地址的权限，拒绝！→ 产生 IOMMU Fault
```

### IOMMU 的两大能力

| 能力 | 说明 | 类比 |
|------|------|------|
| **地址翻译** | 设备 DMA 地址 → 物理地址 | 类似 CPU MMU 的虚拟→物理 |
| **权限控制** | 限制设备只能访问指定的内存区域 | 类似 CPU MMU 的页权限 |

### IOMMU 在硬件中的位置

```
                     ┌─────────┐
                     │   CPU   │
                     └────┬────┘
                          │
                     ┌────┴────┐
                     │   MMU   │ ← CPU 的 MMU
                     └────┬────┘
                          │
              ┌───────────┴───────────┐
              │     内存控制器         │
              │    (Memory Controller) │
              └───────────┬───────────┘
                          │
                     ┌────┴────┐
                     │  DRAM   │ ← 物理内存
                     └────┬────┘
                          │
              ┌───────────┴───────────┐
              │      IOMMU/SMMU       │ ← 设备的 MMU
              └───────────┬───────────┘
                          │
              ┌───────┬───┴───┬───────┐
              │       │       │       │
           ┌──┴──┐ ┌──┴──┐ ┌──┴──┐ ┌──┴──┐
           │PCIe │ │ USB │ │ UFS │ │ GPU │  ← I/O 设备
           └─────┘ └─────┘ └─────┘ └─────┘
```

### 不同平台的 IOMMU 叫法

| 平台 | 名称 | 全称 |
|------|------|------|
| **Intel** | VT-d | Virtualization Technology for Directed I/O |
| **AMD** | AMD-Vi | AMD I/O Virtualization Technology |
| **ARM** | SMMU | System Memory Management Unit |
| **IBM POWER** | PHB with TCE | PCI Host Bridge with Translation Control Entry |

> 💡 虽然名字不同，但本质都是一回事：**给 I/O 设备做地址翻译和权限控制**。

---

## 四、IOMMU 的工作原理

### 4.1 地址翻译过程

```
设备发起 DMA 请求（使用 IOVA）
        ↓
    ┌──────────┐
    │  IOMMU   │
    │          │
    │ 1. 根据设备 ID 找到对应的页表
    │ 2. 查页表：IOVA → 物理地址
    │ 3. 检查权限：读/写是否允许
    │          │
    └────┬─────┘
         │
    ┌────┴────┐
    │  成功？  │
    ├── 是 ──→ 访问物理内存
    └── 否 ──→ IOMMU Fault（类似 CPU 的 Page Fault）
```

### 4.2 关键概念

| 概念 | 说明 | 类比 CPU MMU |
|------|------|-------------|
| **IOVA** | I/O Virtual Address，设备看到的地址 | 虚拟地址 |
| **物理地址** | 实际的 DRAM 地址 | 物理地址 |
| **I/O 页表** | IOMMU 用来做地址翻译的页表 | CPU 页表（PGD/PUD/PMD/PTE） |
| **设备 ID** | 标识是哪个设备（PCI 用 BDF，ARM 用 StreamID） | 进程 ID（ASID） |
| **IOMMU Fault** | 设备访问了没有映射或没有权限的地址 | Page Fault |

### 4.3 每个设备一张页表

和 CPU 为每个进程维护一张页表类似，IOMMU 可以为**每个设备**（或设备组）维护一张独立的页表：

```
           IOMMU
         ┌──────────────────────┐
         │  设备 A (BDF 01:00.0)│──→ 页表 A → 只能访问 0x80000000~0x80010000
         │  设备 B (BDF 02:00.0)│──→ 页表 B → 只能访问 0x90000000~0x90020000
         │  设备 C (BDF 03:00.0)│──→ 页表 C → 只能访问 0xA0000000~0xA0008000
         └──────────────────────┘

设备 A 尝试 DMA 到 0x90000000：
  → IOMMU 查 A 的页表 → 没有映射 → Fault！
  → 设备 A 只能访问自己被分配的内存区域
```

> 💡 这就像酒店房卡——每个住客的房卡只能打开自己的房间，不能进别人的。

---

## 五、IOMMU 的三大作用

### 5.1 安全隔离

防止恶意设备通过 DMA 读写任意内存。

```
攻击场景：
  1. 攻击者插入恶意 Thunderbolt 设备
  2. 设备尝试 DMA 读取内核内存中的密码
  
没有 IOMMU：读取成功 → 密码泄露
有 IOMMU：IOMMU 拒绝访问 → Fault → 攻击失败
```

### 5.2 地址空间虚拟化

让设备看到的地址空间和物理地址不一样，这在虚拟化场景中至关重要：

```
虚拟化场景：
  虚拟机 1 分配给设备 A 的物理内存：0x80000000~0x80100000
  虚拟机 2 分配给设备 B 的物理内存：0x90000000~0x90100000

  但两个设备都以为自己的 DMA 地址从 0x00000000 开始

  IOMMU 做翻译：
    设备 A: 0x00000000 → 0x80000000
    设备 B: 0x00000000 → 0x90000000
```

这让 **PCIe 设备直通（Passthrough）** 成为可能——虚拟机里的设备可以直接 DMA，不需要虚拟机管理器（Hypervisor）介入，性能接近物理机。

### 5.3 解决 32 位 DMA 限制

有些设备只能做 32 位 DMA（地址 < 4GB），但物理内存可能在 4GB 以上：

```
没有 IOMMU：
  需要 Bounce Buffer（在 4GB 以下分配临时缓冲区，来回复制）
  → 性能差

有 IOMMU：
  IOMMU 把设备的 DMA 地址 0x10000000 翻译到物理地址 0x1_80000000
  → 设备以为自己在访问 4GB 以下，实际上是 6GB 位置
  → 不需要 Bounce Buffer，性能好
```

---

## 六、ARM SMMU 详解

在高通平台做 BSP，SMMU 是绕不开的东西。

### 6.1 SMMU 架构概览

ARM 的 SMMU（System MMU）是 IOMMU 的 ARM 实现。目前主流版本：

| 版本 | 说明 | 典型平台 |
|------|------|----------|
| SMMUv1 | 旧版，功能有限 | 老 ARM 平台 |
| **SMMUv2** | 主流版本，支持 Stage 1/2 翻译 | 高通骁龙 845/855/778 等 |
| **SMMUv3** | 最新版，支持 PCIe ATS/PRI | ARM Neoverse N1/N2 服务器 |

### 6.2 SMMU 的 StreamID 和 Context Bank

```
SMMU 的设备识别机制：

每个 I/O 设备有一个 StreamID（类似 PCIe 的 BDF）
  ├── StreamID → 查 Stream Table → 找到 Context Bank
  └── Context Bank → 包含该设备的页表基地址和配置

Context Bank 就像酒店的房间号：
  StreamID 1 → Context Bank 3 → 页表 A（这个设备能访问的内存）
  StreamID 2 → Context Bank 7 → 页表 B（另一个设备能访问的内存）
```

### 6.3 Stage 1 和 Stage 2 翻译

SMMUv2/v3 支持两级地址翻译——这和 ARM CPU 的 EL1/EL2 虚拟化配合：

```
Stage 1（类似 EL1 的页表）：
  设备虚拟地址 (VA) → 中间物理地址 (IPA)
  → 由操作系统/驱动配置

Stage 2（类似 EL2 的页表）：
  中间物理地址 (IPA) → 物理地址 (PA)
  → 由 Hypervisor 配置

完整路径：
  设备 DMA 地址 → Stage 1 → IPA → Stage 2 → 物理内存

非虚拟化场景：
  通常只用 Stage 1，或者 Bypass（直通不翻译）
```

### 6.4 高通平台的 SMMU

在高通 SoC（包括你的 QCS6490/RB3Gen2）上：

```
高通 SMMU 配置层次：
  
  XBL/TZ（闭源）
    ├── 配置 SMMU 的安全域（Secure Context Banks）
    ├── 为 TrustZone 设备配置 SMMU
    └── 锁定安全相关的 SMMU 配置

  UEFI/ABL（开源 EDK2）
    ├── 为启动设备配置 SMMU（UFS、USB 等）
    └── 可能 Bypass SMMU（让设备直接用物理地址）

  Linux 内核
    ├── arm-smmu 驱动接管 SMMU
    ├── 为每个设备建立 IOMMU 域
    └── 配合 DMA API 做地址映射
```

> ⚠️ **BSP 调试经验**：如果在 UEFI 阶段 UFS 或 PCIe 能工作，但到了 Linux 阶段设备挂了，很可能是 SMMU 的 Context Bank 配置在 UEFI 和 Linux 之间不一致。

---

## 七、Intel VT-d 简介

为了对比，简要说一下 Intel 的 IOMMU 实现：

### 7.1 VT-d 架构

```
Intel VT-d 的地址翻译：

  设备 (BDF: Bus/Device/Function)
        ↓
  Root Table → 根据 Bus 号找到 Context Table
        ↓
  Context Table → 根据 Device/Function 找到页表
        ↓
  页表翻译：DMA 地址 → 物理地址
```

| 组件 | 说明 | 类比 ARM SMMU |
|------|------|-------------|
| **Root Table** | 按 PCI Bus 号索引 | Stream Table |
| **Context Table** | 按 Device/Function 索引 | Context Bank |
| **DMAR Table** | ACPI 表，描述 VT-d 硬件信息 | Device Tree 中的 SMMU 节点 |
| **Fault Recording** | 记录 DMA 地址翻译错误 | SMMU Fault |

### 7.2 DMAR ACPI 表

Intel VT-d 的硬件信息通过 ACPI 的 **DMAR 表**（DMA Remapping）告诉操作系统：

```
DMAR 表内容：
  ├── DMA Remapping Hardware Unit（DRHD）
  │   ├── 基地址：VT-d 硬件寄存器的 MMIO 地址
  │   └── 设备范围：哪些 PCI 设备归这个 DRHD 管
  ├── Reserved Memory Region（RMRR）
  │   └── 某些设备需要的固定 DMA 内存区域（如 USB EHCI）
  └── ...
```

> 💡 如果你做 x86 BIOS 开发，配置 DMAR ACPI 表是 IOMMU 工作的关键步骤。

---

## 八、UEFI 中的 IOMMU

### 8.1 EDKII_IOMMU_PROTOCOL

EDK2 定义了一个通用的 IOMMU 协议接口：

```c
struct _EDKII_IOMMU_PROTOCOL {
  UINT64                       Revision;
  EDKII_IOMMU_SET_ATTRIBUTE    SetAttribute;   // 设置设备 DMA 属性
  EDKII_IOMMU_MAP              Map;            // DMA 地址映射
  EDKII_IOMMU_UNMAP            Unmap;          // 取消映射
  EDKII_IOMMU_ALLOCATE_BUFFER  AllocateBuffer; // 分配 DMA 内存
  EDKII_IOMMU_FREE_BUFFER      FreeBuffer;     // 释放 DMA 内存
};
```

### 8.2 PciIo 和 IOMMU 的协作

```
当你调用 PciIo->Map() 时，内部流程：

PciIo->Map()
    ↓
检查是否有 EDKII_IOMMU_PROTOCOL
    ├── 有 → 调用 IoMmu->Map() → IOMMU 建立映射
    │        DeviceAddress = IOVA（IOMMU 分配的设备虚拟地址）
    └── 无 → 直接使用物理地址（或分配 Bounce Buffer）
             DeviceAddress = 物理地址（或 Bounce Buffer 地址）
```

### 8.3 UEFI 阶段是否启用 IOMMU？

| 场景 | 是否启用 | 原因 |
|------|---------|------|
| **大多数客户端平台** | ❌ 不启用 | UEFI 阶段简单，不值得启用 IOMMU 的复杂性 |
| **服务器平台** | ✅ 可能启用 | 安全要求高 |
| **支持 Secure Boot + DMA Protection** | ✅ 启用 | Windows 的 Kernel DMA Protection 要求 |
| **高通 ARM 平台** | ⚠️ 部分启用 | TZ 配置 Secure SMMU，UEFI 阶段可能 Bypass |

---

## 九、IOMMU 不是万能的：常见问题

### 9.1 性能开销

```
每次 DMA 访问多了一步地址翻译：
  设备 DMA → IOMMU 查页表 → 物理内存

额外开销：
  ├── IOMMU 页表查找延迟（TLB miss 时更大）
  ├── IOMMU 页表占用内存
  └── 建立/拆除映射的 CPU 开销
```

### 9.2 兼容性问题

有些老设备的 DMA 行为不符合规范，开了 IOMMU 后会出问题：

```
常见问题：
  ├── 设备 DMA 地址越界 → IOMMU Fault
  ├── 设备访问了未映射的地址 → Fault
  ├── 设备在 DMA 完成后还在访问已取消映射的地址 → Fault
  └── 设备 firmware bug 导致错误的 DMA 地址
```

> 🐛 **BSP 调试技巧**：如果开 IOMMU 后设备不工作，第一步看 IOMMU Fault 日志。ARM 平台用 `dmesg | grep smmu`，Intel 用 `dmesg | grep dmar`。

### 9.3 UEFI 阶段的 IOMMU 挑战

```
UEFI 阶段开 IOMMU 的难点：
  1. 所有设备驱动都要适配 IOMMU（Map/Unmap）
  2. IOMMU 页表管理增加复杂性
  3. 和 TrustZone/SMM 的安全配置要协调
  4. 调试困难——IOMMU Fault 信息在 UEFI 阶段不好看
```

---

## 十、面试题

### Q1：IOMMU 和 MMU 的核心区别是什么？

- MMU 翻译 **CPU** 的虚拟地址 → 物理地址
- IOMMU 翻译 **I/O 设备** 的 DMA 地址 → 物理地址
- MMU 用进程 ID（ASID）区分不同进程
- IOMMU 用设备 ID（StreamID/BDF）区分不同设备

### Q2：ARM SMMU 的 Stage 1 和 Stage 2 分别由谁配置？

- Stage 1：操作系统或固件（EL1 级别）
- Stage 2：Hypervisor（EL2 级别）
- 非虚拟化场景通常只用 Stage 1

### Q3：如果不开 IOMMU，PciIo->Map() 还能工作吗？

能。没有 IOMMU 时，Map() 退化为直接返回物理地址，或者在需要时分配 Bounce Buffer。IOMMU 不是 Map() 的必须依赖。

---

## 十一、总结

```
IOMMU 知识地图：

是什么？
  → 给 I/O 设备装的 MMU
  → 做地址翻译 + 权限控制

为什么需要？
  ├── 安全：防止 DMA 攻击
  ├── 虚拟化：设备直通（Passthrough）
  └── 兼容性：解决 32 位 DMA 限制

不同平台的实现
  ├── Intel：VT-d（Root Table → Context Table → 页表）
  ├── AMD：AMD-Vi
  └── ARM：SMMU（StreamID → Context Bank → 页表）

SMMU 特有概念
  ├── StreamID：设备标识
  ├── Context Bank：设备的配置上下文
  └── Stage 1/2：两级翻译（配合虚拟化）

在 UEFI 中
  ├── EDKII_IOMMU_PROTOCOL
  ├── PciIo->Map() 内部调用 IOMMU
  └── 大多数平台在 UEFI 阶段不启用 IOMMU
```

| 你以为的 IOMMU | 实际的 IOMMU |
|---------------|-------------|
| 可有可无的高级功能 | 安全和虚拟化的基础设施 |
| 只在服务器上用 | 手机、笔记本、IoT 都在用 |
| 和固件无关 | 固件阶段是否启用 IOMMU 影响整个启动安全 |
| 开了就行 | 配错了比不开更糟糕 |

---

下一篇：**#023 ARM SMMU vs Intel VT-d：两大阵营的 IOMMU 对比**——同样是给设备做地址翻译，ARM 和 Intel 的实现在架构、配置方式、固件接口上有哪些差异？搞 BSP 的人两边都要懂。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
