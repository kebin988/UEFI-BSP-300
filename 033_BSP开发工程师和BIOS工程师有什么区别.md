# 033 - BSP开发工程师和BIOS工程师有什么区别？

> **UEFI/EDK2 BSP 开发系列第 33 篇**
> 前一篇：[#032 固件工程师的日常工作是什么样的？](032_固件工程师的日常工作是什么样的.md)
> 下一篇：[#034 芯片原厂 vs OEM vs ODM：固件开发的三个角色](034_芯片原厂vsOEMvsODM固件开发的三个角色.md)

---

## 一、这两个岗位经常被混淆

在招聘网站上搜"BIOS工程师"和"BSP工程师"，职位描述看起来差不多——都要懂 C 语言、都要懂 UEFI、都要跟硬件打交道。但实际上，这两个岗位的**侧重点、技术栈、目标平台**都有显著差异。

一句话版本：
- **BIOS 工程师** = 以 **x86 平台** 为主，做**标准化固件**
- **BSP 工程师** = 以 **ARM/RISC-V 等嵌入式平台** 为主，做**板级适配**

当然现实中有交叉，但这是最常见的分工。

---

## 二、名字从哪来？

### BIOS 工程师

"BIOS"这个词自带 x86 基因——它从 IBM PC 时代就有了。虽然现在叫 UEFI BIOS，但**行业习惯仍然把做 x86 固件的人叫 BIOS 工程师**。

典型雇主：AMI、Insyde、Phoenix（三大 BIOS 厂商）、联想/Dell/HP 的固件团队、Intel 的固件团队。

### BSP 工程师

BSP = Board Support Package（板级支持包）。这个概念来自**嵌入式 Linux**——要让 Linux 跑在一块新的 ARM 开发板上，得提供 bootloader、设备树、内核驱动、外设初始化代码，这一整套就叫 BSP。

后来这个概念扩展了：**任何让操作系统能在特定硬件上跑起来的底层软件工作，都可以叫 BSP 开发**。

典型雇主：高通、联发科、NXP（芯片原厂），中科创达、Thundercomm（方案公司），各种 IoT/车机/手机厂商。

---

## 三、核心差异对比

| 对比维度 | BIOS 工程师 | BSP 工程师 |
|---------|-------------|-----------|
| **主要平台** | x86（Intel/AMD） | ARM（高通/联发科/NXP）、RISC-V |
| **核心框架** | EDK2 + AMI Aptio / Insyde H2O | EDK2 + 芯片原厂 BSP SDK |
| **启动链** | SEC→PEI→DXE→BDS（纯 UEFI） | XBL/SPL→U-Boot/ABL/EDK2→OS |
| **内存初始化** | Intel FSP / AMD AGESA | 芯片原厂闭源 Blob |
| **ACPI vs DT** | 重度 ACPI（几千行 ASL） | ACPI + Device Tree 混用 |
| **操作系统** | Windows / Linux / VMware ESXi | Android / Linux / WoA / RTOS |
| **外设类型** | PCIe NVMe、SATA、USB、集成显卡 | UFS/eMMC、MIPI DSI、Camera、Modem |
| **安全** | Secure Boot + TPM (dTPM 芯片) | Secure Boot + TrustZone (fTPM) |
| **标准化程度** | 高（UEFI/ACPI 标准统一） | 低（每家芯片原厂方案不同） |
| **调试工具** | 串口 + 80 Port + Intel DCI | 串口 + JTAG/SWD + Trace32 |
| **编译工具** | VS2019/GCC + EDK2 build | GCC/Clang + 芯片原厂编译系统 |

### 直观感受

**BIOS 工程师**的世界更"标准化"：Intel 出了新平台，AMI/Insyde 适配 EDK2，OEM 客户在此基础上定制。整个流程有成熟的框架，多数工作是在框架内做配置和调优。

**BSP 工程师**的世界更"荒野"：每家芯片原厂的 BSP 架构不同，高通的和联发科的完全是两套东西。从 bootloader 到内核驱动，很多时候得自己从头适配，标准化程度低但自由度高。

---

## 四、技术栈差异详解

### 4.1 启动链的不同

**x86 BIOS 工程师的世界：**

```
CPU Reset → Reset Vector (0xFFFFFFF0)
    → SEC (Cache as RAM)
    → PEI (FSP-T + FSP-M 初始化内存)
    → DXE (加载几百个 UEFI 驱动)
    → BDS (找启动设备，交给 OS Loader)
    → Windows/Linux 启动
```

整个流程全在 EDK2 框架内，标准化程度非常高。

**ARM BSP 工程师的世界（以高通为例）：**

```
CPU Reset → PBL (芯片内置，不可修改)
    → XBL (高通闭源 Bootloader，初始化 DDR/UFS)
    → ABL 或 EDK2 (UEFI 环境，可修改)
    → HLOS Loader (加载 Android/Linux/Windows)
    → 操作系统启动
```

注意看：**ARM 平台的前半段（PBL + XBL）是芯片原厂锁死的**，BSP 工程师能改的主要是后半段。而且不同芯片原厂的启动链完全不同——高通、联发科、NXP 各有各的玩法。

### 4.2 ACPI vs Device Tree

**BIOS 工程师最头疼的事之一：ACPI 表**。

x86 平台几乎所有硬件描述都靠 ACPI：CPU 拓扑、电源管理、设备枚举、热管理、GPIO、I2C 设备…一个复杂平台的 ACPI ASL 代码可以有**上万行**。

```asl
// x86 ACPI 示例（处理器性能状态）
Scope (\_SB.CPU0) {
    Name (_PCT, Package () {
        ResourceTemplate () { Register (FFixedHW, 0, 0, 0) },
        ResourceTemplate () { Register (FFixedHW, 0, 0, 0) }
    })
    Name (_PSS, Package () {
        Package () { 3600, 35000, 10, 10, 0x0024, 0x0024 },  // 3.6GHz
        Package () { 2400, 20000, 10, 10, 0x0018, 0x0018 },  // 2.4GHz
        Package () { 1200, 10000, 10, 10, 0x000C, 0x000C },  // 1.2GHz
    })
}
```

**BSP 工程师更常用 Device Tree**（尤其是 Linux/Android 场景）：

```dts
// ARM Device Tree 示例（高通 UART）
uart0: serial@a84000 {
    compatible = "qcom,geni-uart";
    reg = <0xa84000 0x4000>;
    interrupts = <GIC_SPI 354 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&gcc GCC_QUPV3_WRAP0_S0_CLK>;
    pinctrl-0 = <&uart0_default>;
    status = "okay";
};
```

不过当 ARM 平台跑 Windows 时，就必须用 ACPI 了（Windows 不认 Device Tree）。所以做 Windows on ARM (WoA) 的 BSP 工程师，反而需要同时精通 ACPI 和 Device Tree——两边都得会。

### 4.3 调试手段的不同

| 调试工具 | BIOS 工程师 | BSP 工程师 |
|---------|-------------|-----------|
| **串口** | ✅ 必备 | ✅ 必备 |
| **80 Port Debug Card** | ✅ x86 特有（POST Code 显示） | ❌ ARM 没有 |
| **JTAG/SWD** | 少用（Intel DCI 替代） | ✅ 必备（Lauterbach Trace32、J-Link） |
| **Intel DCI/CCA** | ✅ Intel 平台专用调试接口 | ❌ |
| **UEFITool** | ✅ 分析 BIOS 固件镜像 | ✅ 也能用 |
| **QEMU** | ✅ OVMF 模拟 | ⚠️ ARM QEMU 支持有限 |
| **逻辑分析仪** | 偶尔 | ✅ 总线调试常用 |

> 💡 **一个冷知识**：x86 的 "80 Port Debug Card" 是一个插在 PCI/LPC 总线上的小卡，显示两位十六进制数（POST Code）。BIOS 在启动的每个关键步骤都会向 Port 80h 写一个数字，卡上的数码管就显示出来。如果机器卡在某个 POST Code 不动了，查手册就知道卡在哪个阶段。ARM 平台没有这个传统，一般靠串口或 GPIO 点灯。

---

## 五、职业发展路径对比

### BIOS 工程师的晋升路线

```
初级 BIOS 工程师
├── 能在现有框架上改 Setup 选项、加驱动
│
├── 中级 BIOS 工程师
│   ├── 能独立做平台 Bring-up（新 Intel 平台适配）
│   ├── 精通 ACPI、FSP 集成
│   │
│   ├── 高级 BIOS 工程师 / 架构师
│   │   ├── 设计固件架构、安全方案
│   │   ├── 领导平台 Bring-up 团队
│   │   │
│   │   └── 转型方向
│   │       ├── 安全架构师（Secure Boot / TEE）
│   │       ├── 云计算/虚拟化固件（OpenBMC、IPMI）
│   │       └── 技术管理
```

### BSP 工程师的晋升路线

```
初级 BSP 工程师
├── 能改芯片原厂 BSP 代码、调外设驱动
│
├── 中级 BSP 工程师
│   ├── 能独立做新板 Bring-up（DDR、eMMC/UFS、Display）
│   ├── 精通至少一家芯片原厂的 BSP 架构
│   │
│   ├── 高级 BSP 工程师 / 架构师
│   │   ├── 跨平台能力（高通 + NXP 或 高通 + x86）
│   │   ├── 固件安全（TrustZone、Secure Boot）
│   │   │
│   │   └── 转型方向
│   │       ├── 芯片原厂 FAE / 技术支持
│   │       ├── 车载 BSP（域控制器、智能座舱）
│   │       ├── UEFI 安全研究
│   │       └── 技术管理
```

### 薪资参考（仅供参考，2024 年一线城市）

| 级别 | BIOS 工程师 | BSP 工程师 |
|------|------------|-----------|
| 初级（1-3 年） | 15-25k | 15-25k |
| 中级（3-5 年） | 25-40k | 25-40k |
| 高级（5-10 年） | 40-60k | 35-55k |
| 架构师/专家 | 60-100k+ | 50-80k+ |

> ⚠️ 以上数字非常粗略，实际薪资取决于公司（Intel/高通等外企 vs 国内公司差异很大）、城市、个人能力。BIOS 工程师在高端服务器领域（Intel/AMD/NVIDIA BMC）薪资较高；BSP 工程师在车载/安全领域薪资上升空间大。

---

## 六、我该选哪个方向？

| 如果你… | 建议方向 |
|---------|---------|
| 喜欢标准化、规范明确的工作 | BIOS 工程师 |
| 喜欢折腾各种硬件、接受"荒野求生" | BSP 工程师 |
| 想进外企（Intel、AMD、Dell） | BIOS 方向 |
| 想进芯片公司（高通、联发科、NXP） | BSP 方向 |
| 对安全感兴趣（Secure Boot、TEE） | 两个都行，但 ARM TrustZone 方向目前更热 |
| 对汽车电子感兴趣 | BSP 工程师（车载域控制器大量使用 ARM） |
| 想转做 Linux 内核开发 | BSP 更近（BSP 经常涉及内核驱动） |
| 想最大化"不可替代性" | 两个都学——既懂 x86 BIOS 又懂 ARM BSP 的人极为稀缺 |

---

## 七、其实它们正在融合

一个重要趋势：**ARM 和 x86 的固件开发正在融合**。

1. **ARM 平台越来越多地使用 UEFI/ACPI**：Windows on ARM 要求 ACPI，ARM 服务器（如 Ampere Altra）完全使用 UEFI+ACPI
2. **EDK2 是两边的公共框架**：不管做 x86 BIOS 还是 ARM BSP，底层都是 EDK2
3. **安全需求统一**：Secure Boot、Measured Boot、固件更新机制，两个平台的实现越来越接近
4. **RISC-V 崛起**：第三个阵营也在采用 UEFI+ACPI（或 UEFI+DT），未来固件工程师可能需要三个平台都懂一点

所以，与其纠结"选 BIOS 还是 BSP"，不如**把 UEFI/EDK2 框架本身学扎实**——这是两边都通用的核心。

---

## 八、本篇小结

```
┌─────────────────────────────────────────────────────────┐
│         BIOS 工程师 vs BSP 工程师 核心区别               │
├─────────────────────────────────────────────────────────┤
│ BIOS 工程师 = x86 为主 + 标准化框架 + ACPI 重度用户    │
│ BSP 工程师  = ARM 为主 + 芯片原厂 SDK + DT/ACPI 混用   │
│                                                         │
│ 共同点 = EDK2 框架 + C 语言 + 硬件调试 + Secure Boot   │
│                                                         │
│ 趋势 = ARM 平台越来越多用 UEFI/ACPI，两者正在融合      │
│ 建议 = 学好 UEFI/EDK2 核心，平台知识可以叠加            │
└─────────────────────────────────────────────────────────┘
```

> 下一篇：[#034 芯片原厂 vs OEM vs ODM：固件开发的三个角色](034_芯片原厂vsOEMvsODM固件开发的三个角色.md) —— 同一款芯片，从原厂到终端产品，固件经过了几手？每一手的人分别做什么？

---

**参考资料：**
- [AMI Aptio V BIOS 开发文档](https://www.ami.com/aptio-v/)
- [Qualcomm BSP Architecture](https://www.qualcomm.com/developer)
- [UEFI Specification 2.10](https://uefi.org/specifications)
- [ARM SystemReady 认证](https://www.arm.com/architecture/system-architectures/systemready-certification-program)
