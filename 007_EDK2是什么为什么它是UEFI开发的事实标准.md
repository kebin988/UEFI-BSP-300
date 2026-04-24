# EDK2 是什么？为什么它是 UEFI 开发的事实标准

> 🔥 UEFI/BSP 开发系列第 007 篇 | 难度：⭐ 入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

前几篇我们聊了 UEFI 是一份**规范**——它规定了固件和操作系统之间应该怎么对话，但它不是代码，不是工具，更不是你能直接用来开机的东西。

那问题来了：**谁把这份规范变成了真正能跑的代码？**

答案就是今天的主角：**EDK2**。

如果说 UEFI 规范是一套建筑设计图纸，那 EDK2 就是全球最权威的施工团队，把图纸变成了你每次按下电源键都在悄悄运行的固件程序。

---

## 一、EDK2 是什么？

EDK2 的全称是 **EFI Development Kit II**，是由 Intel 主导、开源社区共同维护的 **UEFI 固件开发框架**。

> 💡 **一句话理解**：EDK2 是实现 UEFI 规范的开源代码库，也是目前全球 PC/服务器固件开发的事实标准框架。

它的本质是一套：

- **代码库**：几十万行 C 代码，涵盖从 CPU 上电到操作系统启动的完整实现
- **构建系统**：一套工具链，把你写的代码编译成能烧进主板 Flash 的固件镜像
- **开发规范**：规定了模块怎么写、驱动怎么组织、接口怎么定义

简单类比：

```
UEFI 规范  ≈  HTTP/HTTPS 协议文档
EDK2       ≈  Nginx / Apache（把协议变成真正运行的服务器）
```

---

## 二、EDK2 的前世今生

EDK2 不是凭空出现的，它有一段颇为曲折的历史。

### 第一代：EFI 诞生（1998 年）

Intel 在开发安腾（Itanium）处理器时，遇到了 BIOS 的巨大限制。安腾是 64 位架构，而 BIOS 还活在 16 位实模式的石器时代。

Intel 内部开发了 **EFI（Extensible Firmware Interface）**，这是 UEFI 的前身。同期，Intel 也发布了第一版 **EDK（EFI Development Kit）**，作为 EFI 的参考实现。

### 第二代：开源与 TianoCore（2004 年）

Intel 意识到一家公司单打独斗搞不定全行业，于是把 EDK 的核心代码捐给了开源社区，项目名称叫 **TianoCore**，托管在 SourceForge 上。

这是一个关键决定——它让 EDK 从 Intel 的私有项目变成了行业共同维护的基础设施。

### 第三代：EDK2 正式发布（2006 年）

随着 UEFI Forum 正式成立、UEFI 规范完善，Intel 发布了全面重写的 **EDK II（EDK2）**，主要改进：

- 完整支持 UEFI 2.x 规范
- 更清晰的模块化架构
- 跨平台支持（x86、x64、ARM、RISC-V）
- 迁移到 GitHub，社区贡献更方便

```
EFI + EDK（1998）
    ↓ 开源
TianoCore（2004）
    ↓ 重写
EDK2（2006～至今）← 你现在用的就是这个
```

---

## 三、EDK2 长什么样？

克隆下来 EDK2 仓库，你会看到几十个文件夹。别慌，核心目录结构其实很清晰：

```
edk2/
├── MdePkg/          # UEFI 规范定义的基础接口（Protocol、GUID、数据类型）
│                    # 等价于 UEFI 规范的"头文件翻译"
├── MdeModulePkg/    # Microsoft 和 Intel 通用驱动模块（USB、PCI、网络栈...）
├── NetworkPkg/      # UEFI 网络协议栈（TCP/IP、PXE、TLS）
├── SecurityPkg/     # Secure Boot、TCG/TPM 安全相关
├── FatPkg/          # FAT 文件系统驱动（UEFI Shell 需要它）
├── ShellPkg/        # UEFI Shell：固件里的命令行工具
├── OvmfPkg/         # QEMU/KVM 虚拟化平台固件（开发调试利器）
├── ArmPkg/          # ARM 平台支持
├── BaseTools/       # 构建工具集（编译器、链接器、固件打包工具）
└── Conf/            # 构建配置（目标架构、工具链选择）
```

> 💡 刚上手时只需要关注 **MdePkg**（学接口）和 **OvmfPkg**（用 QEMU 跑起来看看）就够了。

---

## 四、为什么 EDK2 是"事实标准"？

"事实标准"（de facto standard）的意思是：没有任何法律或组织强制规定必须用它，但全行业都在用它。

EDK2 能拿到这个地位，有几个关键原因：

### 1. Intel 背书，行业跟进

Intel 是全球最大的 CPU 厂商，它用什么，主板厂商就得跟着用什么。  
更重要的是，Intel 的 **FSP（Firmware Support Package）**——芯片内存初始化的核心代码——就是基于 EDK2 框架封装的。

你想用 Intel 的 CPU，绕不开 FSP，绕不开 EDK2。

### 2. 完整实现，拿来就用

EDK2 不是一个学术项目，而是一个**可以直接用于生产的完整实现**：

| 功能 | EDK2 提供的模块 |
|------|---------------|
| 内存初始化 | 通过 FSP 集成 |
| PCI/USB 枚举 | MdeModulePkg |
| 网络启动（PXE） | NetworkPkg |
| 安全启动 | SecurityPkg |
| UEFI Shell | ShellPkg |
| 虚拟机固件 | OvmfPkg（QEMU 默认固件就是它） |

大厂（AMI、Phoenix、Insyde）的商业 BIOS 也都是以 EDK2 为基础二次开发的。

### 3. 开源 + BSD 许可证，没有法律风险

EDK2 采用 **BSD-2-Clause + Patent License** 双重授权，允许商业使用、允许修改、允许闭源。

这对 OEM/ODM 厂商来说至关重要——他们可以在 EDK2 基础上加入自己的私有定制，而不需要开源自己的代码。

### 4. 跨平台，支持全主流架构

```
x86 / x64（台式机、笔记本、服务器）
ARM / AArch64（手机 SoC、嵌入式、苹果 M 系列前身参考）
RISC-V（新兴开源架构）
```

一套框架，多个平台通用。对于跨平台的 BSP 开发来说，这是巨大的优势。

---

## 五、谁在用 EDK2？

不夸张地说，你现在手边的电脑，99% 的概率正在运行基于 EDK2 的固件：

| 公司/产品 | 使用方式 |
|---------|---------|
| **Intel** | FSP 核心实现、参考平台固件 |
| **AMI** | Aptio 商业 BIOS（全球市占率最高的 BIOS 供应商）基于 EDK2 |
| **Insyde** | InsydeH2O 商业 BIOS |
| **Phoenix** | SecureCore BIOS |
| **Arm** | 参考固件平台 |
| **微软** | Project Mu（UEFI 平台 Surface/HoloLens 固件）基于 EDK2 |
| **Google** | Coreboot + UEFI Payload（Chromebook 固件） |
| **QEMU** | OVMF 虚拟机固件 |

> 微软自己也在用 EDK2 做 Surface 的固件，这大概是"事实标准"最有力的证明。

---

## 六、EDK2 和 UEFI 规范是什么关系？

初学者容易混淆这两个概念，一张图说清楚：

```
                UEFI Forum
                （标准组织）
                    |
                    ↓ 发布
            UEFI Specification
            （规范文档，纯文字）
                    |
                    ↓ 实现
    ┌───────────────┼───────────────┐
    │               │               │
   EDK2          AMI Aptio      Insyde H2O
（开源参考实现）  （商业 BIOS）   （商业 BIOS）
    │
    ↓ 基础
各 OEM/ODM 平台固件
（联想、戴尔、华为、百度...）
```

UEFI 规范规定"要有什么"，EDK2 实现"怎么做"。两者相互独立，但深度绑定——UEFI 规范的每次更新，EDK2 都是第一批对齐实现的框架。

---

## 七、总结

| 维度 | 内容 |
|------|------|
| **是什么** | UEFI 固件开发框架，Intel 主导，开源社区维护 |
| **历史** | EFI（1998）→ TianoCore（2004）→ EDK2（2006） |
| **核心优势** | Intel 背书 + 完整实现 + BSD 许可 + 跨平台 |
| **谁在用** | Intel、AMI、Insyde、微软、Google……几乎全行业 |
| **与 UEFI 关系** | UEFI = 规范，EDK2 = 规范的开源参考实现 |

---

下一篇：**#008 第一次搭建 EDK2 开发环境（Windows 篇）**——从零开始配环境，踩坑记录全在里面。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
