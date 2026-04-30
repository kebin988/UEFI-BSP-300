# 实战：DMA 攻击与 UEFI 固件的防御

> 🔥 UEFI/BSP 开发系列第 024 篇 | 难度：⭐⭐⭐ 进阶
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

前三篇我们把 DMA 和 IOMMU 的原理讲清楚了。今天换个口味——聊聊**黑客怎么用 DMA 搞事情**。

安全攻防是固件领域最刺激的话题之一。上一篇提到过，DMA 允许设备绕过 CPU 直接读写内存——这个"便利"被黑客盯上了。

插一个"U盘"就能偷走你的 BitLocker 密钥？用一根 Thunderbolt 线就能读取内存中的密码？这不是电影情节，是真实存在的攻击手段。

今天从攻击者视角出发，再讲固件怎么防。

> ⚠️ 本文仅用于安全研究和防御目的。所有内容基于公开的安全研究论文和会议演示。

---

## 一、DMA 攻击是什么？

### 1.1 一句话版本

> **DMA 攻击 = 通过外部设备的 DMA 能力，绕过 CPU 和操作系统的安全控制，直接读写物理内存。**

### 1.2 为什么能攻击？

```
正常情况下的安全模型：
  ├── 用户程序 → 没有内核权限 → 不能读内核内存
  ├── 内核模块 → 需要签名验证 → 恶意代码难以加载
  └── CPU 的 MMU → 强制执行内存权限

DMA 攻击为什么能绕过：
  └── DMA 不经过 CPU → 不受 MMU 控制
      → 设备直接访问物理内存
      → CPU 的所有安全机制都无效！
```

这就像一个公司，所有员工进门要刷卡、过安检。但有个后门——快递通道（DMA），快递员（设备）可以直接把东西送进任何办公室，不需要刷卡。如果快递员是间谍呢？

---

## 二、经典 DMA 攻击手段

### 2.1 FireWire 攻击（元祖级）

FireWire（IEEE 1394）是最早被利用来做 DMA 攻击的接口：

```
攻击流程：
  1. 攻击者携带一台笔记本 + FireWire 线
  2. 插入目标电脑的 FireWire 接口
  3. 利用 FireWire 的 DMA 能力读取目标电脑的全部物理内存
  4. 从内存 dump 中提取密码、密钥、敏感数据

工具：inception（开源 DMA 攻击工具）
  → 能直接从内存中提取 Windows/macOS/Linux 的登录密码
  → 甚至可以修改内存中的认证逻辑，直接绕过锁屏
```

> 🎬 2012 年安全会议演示：研究人员用一根 FireWire 线，在 3 秒内绕过了 Windows 锁屏。台下观众：😱

### 2.2 Thunderbolt 攻击（现代版）

Thunderbolt 继承了 PCIe 的 DMA 能力，成为最常用的 DMA 攻击接口：

```
Thunderclap 攻击（2019 年发表）：
  ├── 研究人员用 FPGA 模拟一个恶意 Thunderbolt 设备
  ├── 设备伪装成网卡/显示器
  ├── 通过 DMA 读取内存中的敏感数据
  ├── 即使目标系统开了 IOMMU，也能通过 RMRR/ACS 漏洞突破
  └── 影响 macOS、Linux、Windows、FreeBSD

Thunderspy 攻击（2020 年）：
  ├── 可以重写 Thunderbolt 控制器的固件
  ├── 禁用安全检查
  ├── 获得 DMA 访问权限
  └── 物理接触 5 分钟内完成攻击
```

### 2.3 PCILeech（通用 DMA 攻击框架）

```
PCILeech = DMA 攻击的"瑞士军刀"

支持的攻击硬件：
  ├── FPGA 开发板（Xilinx Artix-7 等）
  ├── USB3380 芯片方案
  ├── Thunderbolt 接口设备
  └── 甚至可以用 Raspberry Pi

能做的事情：
  ├── 读取全部物理内存
  ├── 写入任意物理内存
  ├── 提取 BitLocker/FileVault 密钥
  ├── 绕过 Windows 登录密码
  ├── 注入内核代码
  └── 获取最高权限的 shell
```

> 🐛 **有趣的事实**：PCILeech 的作者 Ulf Frisk 是一位瑞典安全研究员，他把工具完全开源在 GitHub 上。这个项目有 5000+ 星——证明"安全靠的是公开审视，不是隐藏"。

---

## 三、DMA 攻击在 UEFI 启动阶段更危险

你可能觉得"我装了最新 Windows/Linux，有 IOMMU 保护"就安全了。但有个窗口期——**从开机到 OS 启用 IOMMU 之间**，没有任何 DMA 保护：

```
启动时间线：
                                                   
  电源键 ──→ PBL ──→ XBL ──→ UEFI DXE ──→ OS Boot ──→ OS 就绪
                                                        ↑
  ← 这段时间没有 IOMMU 保护 →          OS 启用 IOMMU ─────┘

攻击时机：
  ├── UEFI DXE 阶段：所有 PCIe 设备的 DMA 不受限制
  ├── OS 早期启动：IOMMU 还没配置好
  └── 攻击者在这个窗口期内就能读写内存
```

### Pre-boot DMA 攻击

```
攻击场景：
  1. 攻击者在电脑的 M.2 或 Thunderbolt 口插入恶意设备
  2. 电脑开机进入 UEFI 阶段
  3. UEFI 阶段没有 IOMMU 保护
  4. 恶意设备通过 DMA 修改 UEFI 启动代码
  5. 修改后的启动代码绕过 Secure Boot 验证
  6. 加载恶意 OS Loader
  7. 系统"看起来正常启动"但已被完全控制
```

> 😱 这意味着即使你有 Secure Boot + BitLocker + TPM，如果固件阶段没有 DMA 保护，一个物理访问就能搞定一切。

---

## 四、防御手段

### 4.1 Pre-boot DMA Protection（固件级防御）

**核心思路**：在固件阶段就启用 IOMMU，限制设备 DMA。

#### Intel 平台

```
Intel 的 Pre-boot DMA Protection：
  1. BIOS/UEFI 启动时立即启用 VT-d
  2. 把所有外部端口（Thunderbolt/USB4/PCIe M.2）的设备 DMA 范围限制为 0
     → 这些设备在 OS 启用 IOMMU 之前不能做任何 DMA
  3. 只允许内部设备（NVMe SSD 等）正常 DMA
  4. OS 启动后接管 VT-d，按需分配 DMA 权限
```

```
固件代码逻辑（简化）：
  UEFI PEI 阶段：
    → 启用 VT-d 硬件
    → 配置 Root Table：外部端口设备 → 空页表（禁止 DMA）
    → 配置 Root Table：内部设备 → Identity Mapping（1:1 映射）
    
  UEFI DXE 阶段：
    → 生成 DMAR ACPI 表
    → 标记哪些设备需要 DMA 保护
    
  OS 启动：
    → 读取 DMAR 表
    → 接管 VT-d
    → 为每个设备配置独立的 IOMMU 域
```

#### ARM 平台

```
ARM 的 Pre-boot DMA Protection：
  1. ATF/TZ（Trusted Firmware）阶段配置 SMMU
  2. 默认阻止所有设备的 DMA（白名单模式）
  3. UEFI 阶段只为必要设备（UFS、USB）打开 DMA 通道
  4. 外部接口设备的 SMMU Context Bank 设为 Fault 模式
  5. Linux 内核启动后接管 SMMU 配置
```

### 4.2 Windows Kernel DMA Protection

Windows 10 1803+ 引入了 **Kernel DMA Protection**：

```
工作原理：
  1. Windows 检查固件是否启用了 Pre-boot DMA Protection
     → 读取 ACPI 表中的标志位
  2. 如果已启用：
     ├── 所有外部设备默认 DMA 被阻止
     ├── 用户插入 Thunderbolt 设备时需要授权
     └── 只有经过认证的设备才能获得 DMA 权限
  3. 如果未启用：
     └── 弹出安全警告或降级保护

如何检查你的电脑是否支持：
  → 运行 msinfo32 → 搜索 "Kernel DMA Protection"
  → 值为 "On" = 你的固件做对了
```

### 4.3 Thunderbolt 安全级别

```
BIOS/UEFI 中的 Thunderbolt 安全设置：

Level 0：无安全（所有设备直接 DMA）
Level 1：用户授权（弹窗确认）
Level 2：安全连接（只允许已授权的设备）
Level 3：禁用 Thunderbolt（最安全但最不方便）

推荐：至少 Level 1 + 开启 IOMMU Pre-boot Protection
```

### 4.4 IOMMU 本身的局限

```
IOMMU 不是万能的：
  ├── RMRR 漏洞：某些设备声称需要固定 DMA 区域，IOMMU 被迫放行
  ├── ACS 问题：PCIe 拓扑中某些设备可能共享 IOMMU 域
  ├── Peer-to-Peer DMA：两个 PCIe 设备之间直接 DMA，不经过 IOMMU
  ├── 设备本身被攻陷：恶意固件可以从设备内部发起攻击
  └── 早期启动窗口：在 IOMMU 配置完成前的短暂窗口仍然危险
```

---

## 五、作为 BSP/固件工程师，你需要做什么？

### 5.1 在 UEFI 固件中实现 DMA 保护

```
你的工作清单：

□ PEI 阶段早期启用 IOMMU 硬件
□ 配置默认策略：外部接口设备 DMA = 阻止
□ 为启动设备（NVMe/UFS）配置最小权限的 DMA 映射
□ 生成正确的 DMAR/IORT ACPI 表
□ 设置 ACPI 标志位告诉 OS "Pre-boot DMA Protection 已启用"
□ Thunderbolt 安全级别可配置（BIOS Setup 选项）
□ ExitBootServices 时清理 IOMMU 配置，交给 OS
```

### 5.2 在你的 RB3Gen2 上可以实践什么？

```
实践计划：
  1. 检查 RB3Gen2 的 SMMU 当前配置
     → 看 Linux 启动 log 中的 arm-smmu 信息
     → dmesg | grep smmu

  2. 查看 SMMU Context Bank 分配
     → 哪些设备走了 SMMU，哪些 Bypass 了

  3. 如果有 Bypass 的设备，尝试给它配置 SMMU
     → 修改 UEFI 代码或 Device Tree

  4. 制造一个 SMMU Fault 看看什么现象
     → 故意给设备一个无效的 DMA 地址
```

---

## 六、各平台 DMA 安全对比

| 安全特性 | Intel (VT-d) | AMD (AMD-Vi) | ARM (SMMU) | 高通 (SMMU + TZ) |
|---------|-------------|-------------|-----------|-----------------|
| **Pre-boot DMA Protection** | ✅ 支持 | ✅ 支持 | ✅ 可实现 | ⚠️ 部分（TZ 配置） |
| **Kernel DMA Protection** | ✅ Windows 支持 | ✅ Windows 支持 | ⚠️ Linux 支持 | ⚠️ 取决于平台 |
| **中断重映射** | ✅ | ✅ | ✅ (SMMUv3) | ⚠️ 取决于版本 |
| **Thunderbolt 安全** | ✅ 可配置 | ✅ 可配置 | N/A | N/A |
| **默认安全级别** | 中等 | 中等 | 取决于固件 | 取决于 TZ 配置 |

---

## 七、面试题

### Q1：Pre-boot DMA Protection 是什么？

在 UEFI 固件启动阶段就启用 IOMMU，限制外部设备的 DMA 权限，防止在操作系统接管之前的 DMA 攻击。

### Q2：为什么 UEFI 阶段特别容易受到 DMA 攻击？

因为大多数 UEFI 固件不启用 IOMMU（复杂度高），所有设备的 DMA 都不受限。从开机到操作系统启用 IOMMU 之间存在一个安全窗口。

### Q3：IOMMU 能否完全防止 DMA 攻击？

不能。局限性包括：RMRR 预留区域、ACS 拓扑问题、Peer-to-Peer DMA、设备固件被攻陷、IOMMU 配置前的启动窗口等。IOMMU 是重要的防御层，但需要和其他安全措施（Secure Boot、TPM、设备认证）配合使用。

---

## 八、总结

```
DMA 攻击与防御知识地图：

攻击手段：
  ├── FireWire DMA 攻击（经典）
  ├── Thunderbolt DMA 攻击（现代主流）
  ├── PCILeech 框架（通用攻击工具）
  └── Pre-boot DMA 攻击（固件阶段最危险）

防御手段：
  ├── IOMMU/SMMU（核心防御）
  │   ├── Pre-boot DMA Protection（固件阶段启用）
  │   └── OS 级 IOMMU（运行时保护）
  ├── Thunderbolt 安全级别（接口级别控制）
  ├── Windows Kernel DMA Protection
  └── 设备认证 + Secure Boot + TPM（多层防御）

固件工程师的责任：
  ├── 早期启用 IOMMU
  ├── 正确生成 DMAR/IORT 表
  ├── 外部接口默认 DMA 阻止
  └── 和 OS 安全特性配合
```

| 你以为的 DMA 安全 | 实际的 DMA 安全 |
|-----------------|---------------|
| 装了杀毒软件就安全了 | DMA 攻击绕过所有软件防护 |
| IOMMU 开了就万事大吉 | IOMMU 有漏洞（RMRR/ACS），不是万能的 |
| 只有黑客电影才有 DMA 攻击 | PCILeech 开源免费，淘宝能买到攻击硬件 |
| 跟固件工程师无关 | Pre-boot DMA Protection 就是固件工程师的活 |

---

下一篇：**#025 理解 GUID：UEFI 世界的身份证号（含 Windows 驱动 GUID 对比）**——Protocol 有 GUID，变量有 GUID，驱动有 GUID……128 位的全球唯一标识符到底是怎么保证唯一的？和 Windows 驱动里的 GUID 又是什么关系？

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
