# 学习 UEFI 开发的路线图和推荐资源

> 🔥 UEFI/BSP 开发系列第 035 篇 | 难度：⭐⭐ 进阶入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系
>
> 前一篇：[#034 芯片原厂 vs OEM vs ODM：固件开发的三个角色](034_芯片原厂vsOEMvsODM固件开发的三个角色.md)
> 下一篇：[#036 UEFI启动的四个阶段：SEC→PEI→DXE→BDS 全景图](036_UEFI启动的四个阶段SEC_PEI_DXE_BDS全景图.md)

---

## 写在前面

恭喜你看到这里。

前面 34 篇，我们从"UEFI 是什么"聊到了"固件行业里谁干什么"。你应该已经建立起了对 UEFI/BSP 世界的基本认知——至少下次面试时别人说"Protocol"你不会想成"外交协议"了。

但如果你真的想**做这一行**，光有认知还远远不够。

今天这篇是入门篇的收官之作。我要给你画一张**从零基础到能独立干活的 BSP 工程师**的学习路线图，以及我认为最值得投入时间的学习资源。

> ⚠️ 先打个预防针：**UEFI 开发的学习曲线很陡**。你会发现搜索引擎上中文资料少得可怜，Stack Overflow 上相关问题也不多。这是一个"小众但高薪"的领域——学习成本高，但回报也对得起你的投入。

---

## 一、先搞清楚你要走哪条路

UEFI/BSP 不是一个方向——而是**至少三个方向**，进去之前先选赛道：

```
                        你要走哪条路？
                             │
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
       x86 BIOS 工程师  ARM BSP 工程师    固件安全研究员
       (Intel/AMD 平台)  (高通/MTK/NXP)   (安全攻防/审计)
            │                │                │
    核心技能：          核心技能：          核心技能：
    • EDK2 框架        • ARM 启动流程      • 逆向工程
    • FSP/AGESA        • TF-A / XBL       • 漏洞挖掘
    • ACPI/ASL         • 设备树(DTS)       • 签名/密钥链
    • Setup/HII        • Linux BSP 联调    • CHIPSEC
    • AMI Aptio        • PSCI/TEE         • 固件 Fuzzing
```

### 我该选哪条？

| 如果你… | 推荐方向 |
|---------|---------|
| 本科/研究生刚毕业，零基础 | x86 BIOS（资料最多、生态最成熟） |
| 有嵌入式/Linux 驱动经验 | ARM BSP（过渡最自然） |
| 有安全/CTF 背景 | 固件安全研究（这条赛道缺人，薪资溢价高） |
| 想进大厂（Intel/高通） | 对应平台的 BSP |
| 想进 AMI/Insyde（IBV） | x86 BIOS，必须精通 EDK2 |

> 💡 **三条路的技能底座是相通的**：C 语言、计算机体系结构、UEFI 规范。区别只是上层的"平台特定知识"。所以前期不用纠结，先把基础打好。

---

## 二、学习路线图（全景版）

### 阶段 0：前置技能（1-2 个月）

```
┌─────────────────────────────────────────────────┐
│  如果你这些不熟，先补。不然后面寸步难行。         │
│                                                 │
│  ① C 语言（必须熟练）                            │
│     • 指针、函数指针、结构体、位操作              │
│     • 链表、回调函数、内存管理                   │
│     • 预处理器宏（UEFI 里大量使用）               │
│                                                 │
│  ② 计算机体系结构基础                            │
│     • 内存模型、缓存、MMU 概念                   │
│     • 中断、I/O 端口、MMIO                       │
│     • 寄存器操作                                 │
│                                                 │
│  ③ Linux 基本操作                                │
│     • 命令行、GCC 编译、Makefile                  │
│     • Git 版本管理（EDK2 用 Git 管理）           │
│                                                 │
│  ④ 可选但加分：x86 汇编 / ARM 汇编基础           │
│     • 不需要精通，能读懂就行                     │
└─────────────────────────────────────────────────┘
```

> **我 C 语言学到什么程度才够？** 能手写一个链表库、能看懂 Linux 内核的 `container_of` 宏、能用函数指针实现一个简单的"面向对象"——差不多就够了。UEFI 的核心设计模式就是"结构体 + 函数指针"（回顾本系列 [#016](016_从C语言角度理解UEFI Protocol就是函数指针表.md)）。

### 阶段 1：UEFI 基础认知（1 个月）

这个阶段的目标：**能在脑子里画出 UEFI 启动的完整流程，理解核心概念。**

```
学习内容（对应本系列 #001 - #035）：
  ✅ UEFI vs Legacy BIOS 的区别
  ✅ 启动流程：SEC → PEI → DXE → BDS
  ✅ 核心概念：Protocol、Handle、GUID、FV/FFS
  ✅ UEFI 变量和安全启动
  ✅ 固件行业的角色分工

学习方式：
  → 读本系列的前 35 篇（你已经快读完了 👈）
  → 配合翻阅 UEFI 规范的目录和核心章节（不用全读）
```

### 阶段 2：搭环境 + 动手编码（1-2 个月）

这个阶段的目标：**能编译 EDK2，能写 UEFI 应用和简单驱动，能在 QEMU 里跑起来。**

```
学习路径：

Week 1-2：搭建 EDK2 环境
  → 回顾 #008（Windows）或 #009（Linux）
  → 编译 HelloWorld（#010）
  → 在 QEMU 里跑（#012）

Week 3-4：理解 EDK2 构建系统
  → .INF / .DEC / .DSC / .FDF 四大文件
  → PCD 机制和 Library Class
  → 动手改一个现有模块，编译，看效果

Week 5-8：写代码
  → 写一个 UEFI Application（在 Shell 里跑）
  → 写一个 DXE Driver（注册一个自定义 Protocol）
  → 读懂一个现有的简单 Driver（比如 HelloWorldDxe）
  → 用 GDB + QEMU 调试（下断点、看变量）
```

> 🔥 **这个阶段最关键的一句话**：别光看，动手写。哪怕你的代码很丑，能编译能跑就是进步。UEFI 开发最大的坑不是代码写不好——是环境搭不起来。第一次编译通过的那一刻，你已经超过了 90% 想学 UEFI 的人。

### 阶段 3：深入 UEFI 规范 + 平台代码（2-3 个月）

```
x86 方向：                          ARM 方向：
├── 学习 Intel FSP 架构              ├── 学习 ARM Trusted Firmware (TF-A)
├── 理解 MRC（内存初始化）            ├── 学习 BL1/BL2/BL31 启动链
├── ACPI 表和 ASL 语言               ├── 理解设备树（DTS）
├── Setup 菜单（HII/VFR）            ├── 高通 XBL / QcomPkg 代码结构
├── 读 Intel 参考平台代码             ├── PSCI 和 TEE 的交互
└── 尝试在真板子上刷自编译固件         └── 在开发板上跑自编译固件

两条路都要学的：
├── UEFI 规范核心章节（下面有清单）
├── PI（Platform Initialization）规范
├── UEFI Shell 命令深入
└── Secure Boot 密钥管理
```

### 阶段 4：专项深入 + 实际项目（3-6 个月）

```
到这里你应该已经能在团队里干活了。接下来根据岗位需要深入：

芯片原厂 BSP 岗：
  → 内存训练（DDR/LPDDR Training）
  → PCIe/USB/Display 初始化
  → 电源管理与 S0ix/C-states
  → Secure Boot 全链路（efuse → BL → UEFI → OS）
  → 能耗优化

OEM/IBV BIOS 岗：
  → ACPI 调优（热管理、电池、传感器）
  → Setup/HII 开发
  → Capsule Update 和 Recovery
  → Windows WHQL 认证流程
  → 客户化需求实现

安全研究岗：
  → 固件逆向（UEFITool + IDA/Ghidra）
  → SMM 安全
  → CHIPSEC 实战
  → 已知 CVE 复现
  → Fuzzing 框架搭建
```

---

## 三、必读资料清单

### 3.1 规范文档（官方一手资料）

| 文档 | 说明 | 优先级 |
|------|------|-------|
| [UEFI Specification](https://uefi.org/specifications) | UEFI 规范本体，2800+ 页。别全读，当字典查 | ⭐⭐⭐⭐⭐ |
| [PI Specification](https://uefi.org/specifications) | 平台初始化规范，定义 SEC/PEI/DXE/BDS | ⭐⭐⭐⭐ |
| [ACPI Specification](https://uefi.org/specifications) | ACPI 规范，电源管理和设备描述 | ⭐⭐⭐⭐ |
| [UEFI Shell Specification](https://uefi.org/specifications) | Shell 命令和脚本规范 | ⭐⭐⭐ |

> 💡 **怎么读 UEFI 规范？** 不要从第一页读到最后一页（你会死的）。正确姿势：
> 1. 先看目录，知道有哪些章节
> 2. 遇到具体问题时翻对应章节
> 3. 重点读：Boot Services、Runtime Services、Protocol 定义、Image 加载、变量服务

### 3.2 书籍

| 书名 | 说明 | 适合谁 |
|------|------|-------|
| 《UEFI 原理与编程》（戴正华） | 中文，国内唯一系统讲 UEFI 编程的书 | 入门必买 |
| 《Beyond BIOS》（Intel 出版） | EDK2 架构设计思想，英文 | 进阶必读 |
| *Harnessing the UEFI Shell*（Intel 出版） | UEFI Shell 编程指南 | Shell 开发者 |
| 《ARM Architecture Reference Manual》 | ARM 体系结构圣经 | ARM BSP 必读 |
| 《Intel 64 and IA-32 Architectures SDM》 | x86 体系结构圣经 | x86 BIOS 必读 |

> 戴正华那本书虽然年代有点久了（基于 UDK2014），但核心概念没变。国内中文 UEFI 资料实在太少，这本书是雪中送炭级别的。

### 3.3 源码仓库

| 仓库 | 说明 | 怎么用 |
|------|------|-------|
| [tianocore/edk2](https://github.com/tianocore/edk2) | EDK2 核心代码 | 每天都要翻的，当参考书看 |
| [tianocore/edk2-platforms](https://github.com/tianocore/edk2-platforms) | 各平台的参考代码 | 找你对应平台的 BSP 参考 |
| [ARM-software/arm-trusted-firmware](https://github.com/ARM-software/arm-trusted-firmware) | TF-A，ARM 启动链参考 | ARM BSP 必看 |
| [nicklela/edk2-platforms (Qualcomm)](https://github.com/nicklela/edk2-platforms/tree/master/Silicon/Qualcomm) | 高通 EDK2 参考代码 | 高通 BSP 开发参考 |
| [coreboot](https://github.com/coreboot/coreboot) | 开源固件项目 | 对比学习，理解不同固件哲学 |
| [projectmu](https://github.com/microsoft/mu) | 微软 Project Mu | 模块化 UEFI 的新思路 |

> 🔥 **阅读源码的诀窍**：不要试图"从头到尾"读 EDK2。它有几十万行代码，你会迷路。正确方法是**带着问题读**：
> - "Protocol 是怎么安装的？" → 读 `MdeModulePkg/Core/Dxe/Hand/Handle.c`
> - "Variable 是怎么读写的？" → 读 `MdeModulePkg/Universal/Variable/`
> - "BDS 是怎么选启动项的？" → 读 `MdeModulePkg/Universal/BdsDxe/`

### 3.4 在线资源

| 资源 | 说明 |
|------|------|
| [TianoCore Wiki](https://github.com/tianocore/tianocore.github.io/wiki) | EDK2 官方文档，搭建环境必看 |
| [firmware.intel.com](https://firmware.intel.com) | Intel 固件相关培训资料 |
| [UEFI Forum Learning Center](https://uefi.org/learning_center) | UEFI 官方培训视频 |
| [osdev.org - UEFI](https://wiki.osdev.org/UEFI) | OS 开发社区的 UEFI 资料，信息密度高 |
| [知乎：UEFI 话题](https://www.zhihu.com/topic/20042062) | 中文社区，有几个大佬的回答质量很高 |
| [BIOS 修炼之道（公众号/博客）](https://blog.csdn.net/luobing4365) | 国内少有的系统性 UEFI 博客 |

---

## 四、UEFI 规范重点章节导读

2800 页的规范不用全读。下面是**必读章节**和**按需查阅章节**：

### 必读（通用基础）

| 章节 | 内容 | 为什么必读 |
|------|------|----------|
| Ch 1-2 | 概述与约定 | 理解文档风格和术语 |
| Ch 4 | EFI System Table | 所有服务的入口，相当于 UEFI 的"main 参数" |
| Ch 7 | Boot Services | 内存分配、Protocol 操作、事件机制——UEFI 编程的核心 |
| Ch 8 | Runtime Services | 变量读写、时间、重置——OS 启动后固件还能干的事 |
| Ch 9 | Protocol 规则 | 理解 Protocol 的发现和使用模式 |
| Ch 10 | Image Services | .efi 文件怎么被加载和执行 |
| Ch 13 | UFS/Block IO/File System | 存储协议栈 |
| Ch 32 | Secure Boot | PK/KEK/db/dbx 的完整定义 |

### 按需查阅

| 章节 | 内容 | 什么时候需要 |
|------|------|------------|
| Ch 11 | EFI Loaded Image | 写驱动时 |
| Ch 12 | Console I/O | 做 UI/文本输出时 |
| Ch 14 | PCI Bus | PCI 驱动开发时 |
| Ch 16-17 | USB | USB 驱动开发时 |
| Ch 24 | Network | 网络栈开发时 |
| Ch 33-34 | HII/Forms | Setup 菜单开发时 |

---

## 五、推荐的练手项目

光学不练假把式。以下是按难度递增的练手项目：

### Level 1：Hello World 系列（1-2 周）

```
项目 1：UEFI Hello World Application
  → 在 UEFI Shell 里打印 "Hello UEFI"
  → 目标：验证环境搭建成功

项目 2：系统信息查看器
  → 读取 EFI System Table，打印固件版本、内存映射
  → 目标：熟悉 Boot Services API

项目 3：变量读写工具
  → 创建、读取、修改 UEFI 变量
  → 目标：理解 Runtime Services
```

### Level 2：驱动和 Protocol（2-4 周）

```
项目 4：自定义 Protocol 驱动
  → 写一个 DXE Driver 安装自定义 Protocol
  → 写一个 Application 使用这个 Protocol
  → 目标：理解 UEFI 的核心编程模型

项目 5：简易 Shell 命令
  → 写一个 UEFI Shell 命令，实现 hexdump 功能
  → 目标：熟悉文件系统和 Block IO

项目 6：按键检测 + 简易菜单
  → 用 SimpleTextInput 做一个交互式菜单
  → 目标：理解事件和输入处理
```

### Level 3：平台级练手（1-2 个月）

```
项目 7：[x86] 在 QEMU + OVMF 上定制启动流程
  → 修改 BDS 策略，添加自定义启动项
  → 添加一个自定义 Setup 页面（HII）
  → 目标：理解 BDS 和 HII 框架

项目 8：[ARM] 在 QEMU virt 平台上跑 EDK2
  → 编译 ArmVirtPkg，在 QEMU ARM64 上启动
  → 对比 x86 OVMF 的差异
  → 目标：理解 ARM 平台的 EDK2 适配

项目 9：Secure Boot 实验
  → 生成自定义 PK/KEK/db 密钥
  → 给自己的 .efi 签名
  → 在 QEMU 的 OVMF 上验证 Secure Boot 流程
  → 目标：理解安全启动的完整链路
```

### Level 4：实战级（持续积累）

```
项目 10：给 EDK2 社区提交一个 Patch
  → 修复一个 Bugzilla 上的简单 bug
  → 或者改进一段文档
  → 目标：学习开源协作流程（这个写在简历上是加分项）

项目 11：在真实开发板上做 BSP
  → 买一块 ARM 开发板（树莓派 4/5、RockChip 等）
  → 尝试在上面跑 EDK2
  → 目标：体验真实硬件 bring-up 的痛苦和快乐
```

---

## 六、x86 和 ARM 学习路径对比

| 维度 | x86 (Intel/AMD) | ARM (高通/NXP/RockChip) |
|------|-----------------|------------------------|
| **入门难度** | ⭐⭐（资料多，生态成熟） | ⭐⭐⭐（资料少，厂商各搞各的） |
| **开发环境** | QEMU + OVMF，开箱即用 | QEMU ARM64 + ArmVirtPkg，或买开发板 |
| **核心规范** | UEFI Spec + PI Spec + ACPI Spec | UEFI Spec + ARM SMCCC + PSCI + 设备树(DT) |
| **启动链** | Reset Vector → SEC → PEI → DXE → BDS | BL1 → BL2 → BL31 → BL33(UEFI) → BDS |
| **闭源 Blob** | Intel FSP / AMD AGESA | 高通 XBL/PBL、厂商 TZ/HYP |
| **ACPI vs DTS** | 全面依赖 ACPI | ACPI + 设备树混合（服务器偏 ACPI，嵌入式偏 DT） |
| **社区活跃度** | 高（TianoCore 主力） | 中（Linaro 推动中，但碎片化严重） |
| **求职市场** | PC/服务器 BIOS 岗 | 手机/IoT/车机 BSP 岗 |

> **如果你两个都不熟，先从 x86 OVMF 入手。** 原因很简单：不需要硬件、不需要买板子、QEMU 上直接就能跑。等你在 OVMF 上把 EDK2 的核心概念搞明白了，再转 ARM 事半功倍。

---

## 七、常见误区

### ❌ "我要先把 UEFI 规范全部读完再开始写代码"

**别。** 2800 页的规范不是用来"读完"的，是用来查的。正确方法：动手写代码 → 遇到问题 → 翻规范对应章节 → 理解了 → 继续写。

### ❌ "UEFI 用 C 语言写，所以我只要会 C 就够了"

C 是必要条件，不是充分条件。你还需要：
- 理解硬件（寄存器、总线、时钟、电源域）
- 理解启动流程（不是 OS 层面的启动，是固件层面的）
- 读英文文档的能力（规范和代码注释全是英文）

### ❌ "去找个培训班学 UEFI 最快"

国内几乎没有靠谱的 UEFI 培训班。这个领域太小众了，不像 Java/Python 有成熟的培训产业。**最好的学习方式是：读代码 + 写代码 + 读规范 + 问社区。** 如果你英语可以，TianoCore 的邮件列表（edk2-devel@groups.io）上问问题响应很快。

### ❌ "学 UEFI 不需要了解硬件"

**大错特错。** UEFI 存在的意义就是初始化硬件、把硬件信息传给 OS。你可以不会设计电路，但你必须看得懂原理图、知道寄存器怎么配、理解 PCIe/USB/DDR 的基本原理。**固件是软件和硬件之间的翻译官——两边都不懂就没法翻译。**

### ❌ "ARM 和 x86 的 UEFI 完全一样"

框架一样（都是 EDK2），但差异不小：
- 启动链不同（x86 没有 BL1/BL2，ARM 没有 FSP）
- 硬件发现方式不同（x86 用 ACPI，ARM 部分用设备树）
- 安全模型不同（x86 用 SMM + Boot Guard，ARM 用 TrustZone + SMMU）
- 内存初始化实现完全不同

---

## 八、一份实际可执行的 90 天计划

给真正想入行的人，一份**具体到周**的学习计划：

```
Week 1-2：补 C 语言基础
  → 重点：函数指针、结构体嵌套、宏展开、位操作
  → 产出：能手写一个函数指针表实现多态

Week 3-4：搭建 EDK2 环境
  → Windows 或 Linux 都行
  → 编译 OVMF，在 QEMU 里启动 UEFI Shell
  → 编译运行 HelloWorld
  → 产出：截图证明你的环境能跑

Week 5-6：读 EDK2 构建系统
  → 理解 .INF/.DEC/.DSC/.FDF
  → 修改一个现有模块的 PCD，重新编译，验证效果
  → 产出：能解释 build 命令的完整流程

Week 7-8：写第一个 DXE Driver
  → 实现一个自定义 Protocol
  → 另写一个 App 调用这个 Protocol
  → 在 QEMU 里跑通
  → 产出：两个能联动的 .efi 文件

Week 9-10：读 UEFI 规范核心章节
  → Boot Services（Ch 7）和 Runtime Services（Ch 8）
  → 边读边对照 EDK2 源码
  → 产出：能画出 Protocol 安装和发现的时序图

Week 11-12：选方向做一个平台级项目
  → x86：定制 OVMF 的 BDS + 加一个 Setup 页面
  → ARM：编译 ArmVirtPkg 并在 QEMU ARM64 上启动
  → 产出：一个可展示的项目（面试用）

Week 13：整理笔记 + 写博客
  → 把学习过程写成 3-5 篇博客发到知乎/掘金/CSDN
  → 产出：技术影响力 + 面试加分项
```

> 💡 **这个计划假设你每天能投入 2-3 小时。** 如果是全职学习，时间可以压缩到一半。

---

## 九、工具清单

不管走哪条路，以下工具建议提前装好：

| 工具 | 用途 | 平台 |
|------|------|------|
| **QEMU** | 虚拟机，跑 UEFI 固件 | x86 + ARM 通用 |
| **VS Code + C/C++ 插件** | 代码阅读和编辑 | 通用 |
| **UEFITool** | BIOS 镜像解析 | 通用 |
| **iasl** | ACPI ASL 编译/反编译 | x86 为主 |
| **GDB** | 源码级调试 | 通用 |
| **Python 3** | EDK2 构建工具链依赖 | 通用 |
| **OpenSSL** | Secure Boot 密钥和签名 | 通用 |
| **Git** | 代码管理 | 通用 |
| **IDA Pro / Ghidra** | 固件逆向 | 安全方向 |
| **CHIPSEC** | 固件安全审计 | 安全方向 |
| **sbsign / mokutil** | Secure Boot 签名工具 | Linux |

---

## 十、总结：入门篇收官

```
入门篇（#001 - #035）你应该收获了：

✅ 知道 UEFI 是什么、为什么替代了 Legacy BIOS
✅ 理解启动流程的大框架：SEC → PEI → DXE → BDS
✅ 理解 Protocol/Handle/GUID 的核心编程模型
✅ 知道 DMA/IOMMU、内存管理、变量系统等基本概念
✅ 理解 Secure Boot 两层模型：Platform + UEFI
✅ 理解固件打包格式：FV/FFS/FD
✅ 了解固件行业角色：芯片原厂/IBV/ODM/OEM
✅ 有一份清晰的学习路线图和资源清单

接下来的"第二阶段：启动流程深入"会开始讲硬核内容了：
  SEC 阶段的 Reset Vector、CAR（Cache As RAM）
  PEI 阶段的内存初始化
  DXE 的调度器算法
  BDS 的启动策略
  ...

技术深度会明显上一个台阶 📈
```

| 入门篇的你 | 深入篇的你（目标） |
|-----------|-------------------|
| "UEFI 好像是新一代 BIOS" | "SEC 阶段 CAR 的实现原理我能讲清楚" |
| "Protocol 大概是个函数指针" | "我能手写一个 DXE Driver 注册 Protocol" |
| "Secure Boot 是验证签名的" | "PK/KEK/db/dbx 我自己能配" |
| "BIOS 工程师在干什么我不太清楚" | "给我一个 EVB 板子我能做 bring-up" |

---

## 参考资料

- [UEFI Forum Specifications](https://uefi.org/specifications) — UEFI/PI/ACPI 规范下载
- [TianoCore EDK2 GitHub Wiki](https://github.com/tianocore/tianocore.github.io/wiki) — EDK2 官方文档
- [Beyond BIOS: Developing with the Unified Extensible Firmware Interface](https://www.amazon.com/Beyond-BIOS-Developing-Extensible-Interface/dp/1501514784) — Intel 官方 UEFI 架构书
- 戴正华《UEFI 原理与编程》 — 中文 UEFI 编程教材
- [ARM Trusted Firmware Documentation](https://trustedfirmware-a.readthedocs.io/) — TF-A 官方文档
- [Intel FSP Overview](https://www.intel.com/content/www/us/en/developer/topic-technology/firmware/fsp/overview.html) — FSP 架构介绍
- [EDK2 Mailing List (edk2-devel)](https://edk2.groups.io/g/devel) — EDK2 社区邮件列表
- [OSDev Wiki - UEFI](https://wiki.osdev.org/UEFI) — OS 开发视角的 UEFI 资料

---

> **下一篇预告**：[#036 UEFI启动的四个阶段：SEC→PEI→DXE→BDS 全景图](036_UEFI启动的四个阶段SEC_PEI_DXE_BDS全景图.md) —— 正式进入第二阶段"启动流程深入"。我们来给 SEC/PEI/DXE/BDS 每个阶段画一张详细的流程图，搞清楚每个阶段到底在干什么、为什么要这么分。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
