# ARM SMMU vs Intel VT-d：两大阵营的 IOMMU 对比

> 🔥 UEFI/BSP 开发系列第 023 篇 | 难度：⭐⭐⭐ 进阶
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们搞清楚了 IOMMU 的基本概念——给设备装一个 MMU。

你可能已经注意到了：ARM 叫 **SMMU**，Intel 叫 **VT-d**，AMD 叫 **AMD-Vi**。

名字不同，核心功能一样——但实现细节差距很大，就像同样是火锅，重庆的牛油锅和北京的涮羊肉——都是"把东西放热水里煮"，但你不能用同一套吃法。

今天就把 ARM SMMU 和 Intel VT-d 拉出来正面 PK，搞 BSP 的人两边都要懂。

---

## 一、先放一张总览对比表

| 对比维度 | ARM SMMU | Intel VT-d |
|---------|----------|-----------|
| **全称** | System Memory Management Unit | Virtualization Technology for Directed I/O |
| **主流版本** | SMMUv2 / SMMUv3 | VT-d (定义在 Intel VT-d Spec) |
| **设备标识** | StreamID | BDF (Bus/Device/Function) |
| **翻译级数** | Stage 1 + Stage 2（两级） | 只有一级（但支持嵌套翻译） |
| **页表格式** | ARM 标准页表（兼容 CPU 页表） | Intel 定义的私有页表格式 |
| **硬件信息描述** | Device Tree (DT) 或 ACPI IORT 表 | ACPI DMAR 表 |
| **中断重映射** | SMMUv3 支持 | 支持 |
| **典型平台** | 高通骁龙、飞腾、ARM 服务器 | Intel Core/Xeon、AMD 平台 |
| **固件配置** | XBL/ATF/UEFI 配置 SMMU 寄存器 | BIOS 配置 VT-d + 生成 DMAR 表 |

> 💡 你可以把这张表截图保存——面试的时候问"SMMU 和 VT-d 有什么区别"，直接甩这张表，面试官当场好感度 +50。

---

## 二、设备标识：StreamID vs BDF

### ARM SMMU：StreamID

在 ARM 世界里，每个能做 DMA 的设备都有一个 **StreamID**——SMMU 用它来区分"是谁在做 DMA"。

```
StreamID 的来源：
  ├── PCIe 设备：StreamID = BDF（和 Intel 一样）
  ├── 平台设备（SoC 上的 IP）：StreamID 由硬件连线决定
  │   例如：UFS 控制器的 StreamID 可能是 0x1A0
  │         USB 控制器的 StreamID 可能是 0x200
  └── 在 Device Tree 或 IORT 表中描述
```

**高通平台特别说明**：高通 SoC 上的每个外设（UART、SPI、I2C、UFS、USB、GPU）都有固定的 StreamID，这些 ID 是在芯片设计时就确定的，写死在硬件里。调试时经常需要查 StreamID 对应关系。

```
# 高通平台 Device Tree 中的 SMMU 描述（简化）
smmu: iommu@15000000 {
    compatible = "qcom,sm7325-smmu-500", "arm,mmu-500";
    reg = <0x15000000 0x100000>;
    #iommu-cells = <2>;
};

ufs: ufshc@1d84000 {
    iommus = <&smmu 0x1a0 0x0>;  // StreamID = 0x1A0
};
```

### Intel VT-d：BDF

在 x86/PCIe 世界里，设备通过 **BDF**（Bus/Device/Function）三元组来标识：

```
BDF = Bus:Device.Function
  例如：02:00.0 = PCI Bus 2, Device 0, Function 0

VT-d 用 BDF 在 Root Table → Context Table 中查找：
  Root Table[Bus=02] → Context Table[Dev=00, Func=0] → 页表
```

### 对比

| | ARM SMMU | Intel VT-d |
|---|---|---|
| **PCIe 设备** | StreamID = BDF | BDF |
| **SoC 平台设备** | StreamID（硬件连线） | 不适用（x86 没有太多 SoC 平台设备） |
| **查找过程** | StreamID → Stream Table → Context | BDF → Root Table → Context Table |

> 🎮 这就像两款游戏的角色 ID 系统：ARM 用的是"统一编号"（StreamID），不管你是 PCIe 设备还是 SoC 外设都有编号；Intel 用的是"PCI 门牌号"（BDF），只给 PCI 设备用。

---

## 三、地址翻译：两级 vs 一级

### ARM SMMU：Stage 1 + Stage 2

ARM SMMU 天生支持两级翻译，这是 ARM 虚拟化架构的核心设计：

```
Stage 1（OS/驱动配置）：
  设备虚拟地址 (VA) → 中间物理地址 (IPA)
  
Stage 2（Hypervisor 配置）：
  中间物理地址 (IPA) → 物理地址 (PA)

完整路径：
  设备 DMA 地址 ──Stage1──→ IPA ──Stage2──→ 物理内存
```

```
使用场景：
  ├── 非虚拟化：只用 Stage 1（或 Bypass）
  ├── 虚拟化：Stage 1 + Stage 2
  │   ├── Stage 1 由 VM 里的驱动控制
  │   └── Stage 2 由 Hypervisor 控制（VM 不可见）
  └── 纯 Stage 2：Hypervisor 直接管理
```

### Intel VT-d：单级翻译

Intel VT-d 从设计上只有一级地址翻译，但通过 **First-Level / Second-Level / Nested** 三种模式来支持不同场景：

```
Second-Level 翻译（默认）：
  DMA 地址 → Second-Level 页表 → 物理地址
  → Hypervisor 配置，用于设备直通

First-Level 翻译（较新）：
  DMA 地址 → First-Level 页表 → 物理地址
  → 类似 CPU 进程页表格式，用于 SVA/SVM

Nested 翻译（最新）：
  DMA 地址 → First-Level → Second-Level → 物理地址
  → 等效于 ARM 的 Stage 1 + Stage 2
```

### 对比总结

| | ARM SMMU | Intel VT-d |
|---|---|---|
| **两级翻译** | 原生支持（Stage 1/2） | 后来通过 Nested 模式支持 |
| **页表格式** | 和 ARM CPU 页表共享格式 | Intel 自定义格式 |
| **虚拟化友好度** | ⭐⭐⭐⭐⭐（天生为虚拟化设计） | ⭐⭐⭐⭐（后期追加嵌套） |

> 🍜 ARM 的 SMMU 就像一出生就是双卡双待手机（两级翻译），Intel 的 VT-d 是后来通过系统更新才支持双卡的——功能上差不多，但 ARM 的设计更原生。

---

## 四、页表格式：共享 vs 私有

这是一个很有趣的设计差异。

### ARM SMMU：和 CPU 共享页表格式

ARM 的 SMMU 页表格式和 CPU 的 MMU 页表格式**完全兼容**：

```
好处：
  ├── CPU 和设备可以共享同一套页表！
  ├── 操作系统不需要维护两套页表
  ├── SVA（Shared Virtual Addressing）天然支持
  │   → 设备和 CPU 看到的虚拟地址相同
  └── 代码复用：页表管理逻辑通用

实际效果：
  CPU 进程的虚拟地址 0x7FFE0000 → 物理地址 0x80010000
  设备 DMA 到 0x7FFE0000 → 也能翻译到 0x80010000
  → 设备可以直接 DMA 到用户态进程的内存！（在安全框架内）
```

### Intel VT-d：私有页表格式

Intel VT-d 最初使用自己定义的页表格式，后来 First-Level 模式支持了和 CPU 相同的页表格式：

```
Second-Level（传统模式）：
  └── Intel 自定义的 4 级页表格式
      └── 和 CPU 的 IA-32e 页表相似但不完全一样

First-Level（新模式）：
  └── 和 CPU 的 IA-32e/x86-64 页表格式相同
      └── 支持 SVA/SVM
```

### 对比

| | ARM SMMU | Intel VT-d |
|---|---|---|
| **页表格式** | 和 ARM CPU 页表一致（Stage 1） | Second-Level 是私有格式；First-Level 兼容 CPU |
| **SVA 支持** | 原生（共享页表） | First-Level 模式支持 |
| **页表大小** | 4KB/16KB/64KB 页 | 4KB 页 |

---

## 五、固件接口：IORT vs DMAR

操作系统需要知道"IOMMU 硬件在哪里、管哪些设备"，这个信息在固件阶段就要准备好。

### ARM 平台：IORT 表（或 Device Tree）

```
IORT = I/O Remapping Table（ACPI 表）

IORT 表内容：
  ├── SMMU 节点
  │   ├── 基地址（SMMU 寄存器的 MMIO 地址）
  │   ├── 类型（SMMUv1/v2/v3）
  │   └── 中断号
  ├── Named Component 节点
  │   └── 平台设备（非 PCIe）和 SMMU 的映射关系
  ├── Root Complex 节点
  │   └── PCIe 设备和 SMMU 的 StreamID 映射
  └── PMCG 节点（性能监控）
```

在非 ACPI 平台（很多 ARM 嵌入式），用 Device Tree 描述：

```dts
// Device Tree 中的 SMMU 描述
smmu: iommu@15000000 {
    compatible = "arm,smmu-v2";
    reg = <0x15000000 0x100000>;
    #iommu-cells = <1>;
    // ...
};

// 设备引用 SMMU
ufs: ufshc@1d84000 {
    iommus = <&smmu 0x1a0>;  // StreamID
};
```

### Intel 平台：DMAR 表

```
DMAR = DMA Remapping Table（ACPI 表）

DMAR 表内容：
  ├── DRHD（DMA Remapping Hardware Unit Definition）
  │   ├── VT-d 硬件寄存器的 MMIO 基地址
  │   ├── 管理的 PCI Segment
  │   └── 关联的设备范围（Device Scope）
  ├── RMRR（Reserved Memory Region Reporting）
  │   └── 某些设备需要预留的 DMA 内存区域
  ├── ATSR（ATS Root Port）
  │   └── 支持 ATS 的 PCIe Root Port
  └── RHSA（Remapping Hardware Status Affinity）
      └── NUMA 亲和性信息
```

### 对比

| | ARM (IORT/DT) | Intel (DMAR) |
|---|---|---|
| **描述方式** | ACPI IORT 表 或 Device Tree | ACPI DMAR 表 |
| **平台设备支持** | ✅（Named Component） | ❌（只描述 PCI 设备） |
| **预留内存** | 通过 DT reserved-memory | RMRR 结构 |
| **BIOS/UEFI 责任** | 生成 IORT 表 + 初始化 SMMU | 生成 DMAR 表 + 初始化 VT-d |
| **调试方法** | `acpidump -t IORT` 或看 DT | `acpidump -t DMAR` |

> 🐛 **BSP 调试经验**：如果 Linux 启动后 IOMMU 不工作，先 `acpidump` 看 IORT/DMAR 表。很多问题出在固件生成的表不对——设备 StreamID/BDF 映射错误、基地址错误、设备范围遗漏等。

---

## 六、中断重映射

除了 DMA 地址翻译，IOMMU 还有一个重要功能：**中断重映射（Interrupt Remapping）**。

### 为什么需要中断重映射？

```
中断注入攻击：
  恶意设备伪造一个 MSI 中断 → 注入到任意 CPU → 执行恶意中断处理函数

中断重映射防御：
  IOMMU 拦截所有 MSI/MSI-X 中断
  → 验证中断来源（设备 ID）
  → 验证中断目标（是否允许向这个 CPU 发中断）
  → 合法才放行
```

| | ARM SMMU | Intel VT-d |
|---|---|---|
| **SMMUv2** | ❌ 不支持中断重映射 | — |
| **SMMUv3** | ✅ 支持（通过 Event Queue） | ✅ 支持 |

---

## 七、实际场景对比

### 场景一：虚拟机设备直通（Passthrough）

```
ARM (SMMU):
  1. Hypervisor 配置 Stage 2 页表
  2. VM 内的驱动配置 Stage 1 页表
  3. 设备 DMA → Stage 1 → Stage 2 → 物理内存
  4. Hypervisor 不需要介入每次 DMA → 性能好

Intel (VT-d):
  1. Hypervisor 配置 Second-Level 页表
  2. 设备 DMA → Second-Level → 物理内存
  3. 或用 Nested 模式：First-Level + Second-Level
```

### 场景二：固件阶段启用 IOMMU

```
ARM 固件流程：
  ATF/TZ → 配置 Secure SMMU → 锁定安全设备
  UEFI   → 配置 Non-Secure SMMU 或 Bypass
  Linux  → arm-smmu 驱动接管

Intel 固件流程：
  BIOS/UEFI → 初始化 VT-d 硬件
           → 生成 DMAR ACPI 表
           → 可能启用 Pre-boot DMA Protection
  Linux/Windows → 读 DMAR 表 → 配置 VT-d
```

### 场景三：DMA 保护（Windows Kernel DMA Protection）

```
Windows 要求固件：
  ARM：SMMU 在启动时就限制设备 DMA 范围
  x86：VT-d 在启动时就启用 Pre-boot DMA Protection

固件的责任：
  ├── 初始化 IOMMU 硬件
  ├── 为启动设备建立最小必要的映射
  ├── 阻止其他设备的 DMA（白名单模式）
  └── 把配置传递给 OS
```

---

## 八、面试题

### Q1：SMMU 的 StreamID 从哪里来？

- PCIe 设备：StreamID 来自 BDF（由 PCIe 枚举确定）
- 平台设备：StreamID 由 SoC 硬件连线决定，在 Device Tree 或 IORT 表中描述

### Q2：Intel VT-d 的 RMRR 是什么？为什么争议很大？

RMRR（Reserved Memory Region Reporting）标记了某些设备"必须能 DMA 到的固定内存区域"。争议在于：
- 很多 BIOS 的 RMRR 配置不准确
- Linux IOMMU 驱动遇到 RMRR 时需要特殊处理
- 错误的 RMRR 会导致设备无法正常工作

### Q3：为什么 ARM SMMU 页表能和 CPU 共享但 VT-d 不能？

ARM 的 SMMU 从设计之初就使用和 ARM CPU 相同的页表描述符格式（LPAE），这是架构层面的统一设计。Intel VT-d 的 Second-Level 页表是独立设计的，虽然类似但不兼容 CPU 页表。后来 Intel 在 First-Level 模式中添加了 CPU 页表兼容支持。

---

## 九、总结

```
ARM SMMU vs Intel VT-d 关键差异：

设备标识
  ARM: StreamID（通用，包括非 PCIe 设备）
  Intel: BDF（仅 PCIe 设备）

翻译架构
  ARM: 原生两级（Stage 1/2）
  Intel: 单级（Second-Level），后加 Nested

页表
  ARM: 和 CPU 共享格式 → SVA 天然支持
  Intel: 私有格式（Second-Level），后兼容（First-Level）

固件接口
  ARM: IORT 表 / Device Tree
  Intel: DMAR 表

记忆口诀：
  ARM 天生双修（两级翻译），共享页表省事多
  Intel 单修后觉悟，Nested 模式来补课
  StreamID 是万能号，BDF 只管 PCI 道
  IORT 配 ARM，DMAR 配 Intel，记住这四个就够了
```

---

下一篇：**#024 实战：DMA 攻击与 UEFI 固件的防御**——插一个恶意 Thunderbolt 设备就能偷走你的密码？DMA 攻击是怎么回事？UEFI 固件怎么防？我们来看看真实的攻防场景。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
