# UEFI规范到底规定了什么？五分钟读懂核心概念

> 🔥 UEFI/BSP 开发系列第 006 篇 | 难度：⭐ 入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

如果你去搜"UEFI 规范"，会找到一本 **2800 多页**的 PDF：

```
UEFI Specification Version 2.10
Published: August 2022
Pages: 2,818

你：？？？这是规范还是百科全书？？
```

这本"天书"是 UEFI Forum（统一可扩展固件接口论坛）发布的官方标准文档。Intel、AMD、微软、苹果、ARM 等几十家公司共同参与制定。

**但你真的需要读完 2800 页吗？不需要。** 今天我们用五分钟，把它的核心概念提炼出来。

---

## 一、UEFI 规范的本质：一份"合同"

UEFI 规范本质上就是一份**合同**，规定了三方之间的接口：

```
+-----------------------------------------------------------+
|                      UEFI 规范 = 合同                       |
+-----------------------------------------------------------+
|                                                           |
|  甲方: 操作系统 (Windows/Linux)                             |
|        "我需要这些服务，你必须按这个格式提供"                    |
|                                                           |
|  乙方: 固件 (UEFI BIOS)                                    |
|        "行，我保证实现你要的接口"                              |
|                                                           |
|  丙方: 硬件厂商 (芯片公司、板卡厂商)                           |
|        "我的硬件按这个标准提供驱动"                             |
|                                                           |
+-----------------------------------------------------------+
```

> 就像 USB 标准规定了插头的形状和传输协议，UEFI 规范规定了**固件必须提供哪些功能、操作系统该怎么调用它们**。

这意味着：
- **Windows 不需要关心**你的 BIOS 是 AMI 做的还是 Insyde 做的，只要它实现了 UEFI 规范的接口
- **Linux 不需要关心**你的板子是 Intel 还是 AMD，只要固件按规范提供了启动服务
- **你写的 UEFI 应用**可以在任何符合 UEFI 规范的平台上运行

---

## 二、UEFI 规范的五大核心内容

2800 页的规范，其实核心就这五大块：

```
UEFI 规范的五大支柱:

  ┌─────────────────────────────────────────────────┐
  │                                                 │
  │  1. Boot Services         (启动服务)             │
  │     ↓ 操作系统启动前的服务                        │
  │                                                 │
  │  2. Runtime Services      (运行时服务)            │
  │     ↓ 操作系统启动后仍然可用的服务                  │
  │                                                 │
  │  3. Protocol               (协议/接口)            │
  │     ↓ 固件模块之间、固件与OS之间的通信方式          │
  │                                                 │
  │  4. Boot Manager           (启动管理器)            │
  │     ↓ 管理启动项、选择从哪个设备启动               │
  │                                                 │
  │  5. Driver Model           (驱动模型)             │
  │     ↓ UEFI 驱动的标准写法                         │
  │                                                 │
  └─────────────────────────────────────────────────┘
```

我们一个一个来看。

---

## 三、Boot Services（启动服务）—— 开机期间的打工人

Boot Services 是操作系统**启动之前**，固件提供给所有 UEFI 模块和 OS Loader 使用的一组服务。

把它想象成一个**临时营业的服务中心**——只在开机期间营业，OS 一接管就关门。

```
Boot Services 服务清单:

┌────────────────────────────────────────────────────────────┐
│                    Boot Services 服务大厅                    │
├──────────────────┬─────────────────────────────────────────┤
│ 柜台1: 内存管理   │ AllocatePages()  - 分配内存页            │
│                  │ FreePages()      - 释放内存页            │
│                  │ AllocatePool()   - 分配内存块            │
│                  │ GetMemoryMap()   - 查看内存地图           │
├──────────────────┼─────────────────────────────────────────┤
│ 柜台2: Protocol  │ InstallProtocol()  - 注册新服务          │
│        管理      │ LocateProtocol()   - 查找服务            │
│                  │ HandleProtocol()   - 获取服务            │
│                  │ OpenProtocol()     - 打开服务             │
├──────────────────┼─────────────────────────────────────────┤
│ 柜台3: 事件管理   │ CreateEvent()    - 创建事件              │
│                  │ SetTimer()       - 设置定时器            │
│                  │ WaitForEvent()   - 等待事件              │
│                  │ SignalEvent()    - 触发事件              │
├──────────────────┼─────────────────────────────────────────┤
│ 柜台4: 镜像管理   │ LoadImage()      - 加载 EFI 程序         │
│                  │ StartImage()     - 启动 EFI 程序         │
│                  │ ExitBootServices()- 操作系统接管！         │
│                  │                    ⬆️ 调用这个后服务中心关门│
├──────────────────┼─────────────────────────────────────────┤
│ 柜台5: 杂项      │ Stall()          - 延时等待              │
│                  │ SetWatchdogTimer() - 看门狗               │
│                  │ ConnectController() - 连接控制器          │
└──────────────────┴─────────────────────────────────────────┘
```

**关键概念：`ExitBootServices()`**

这个函数是一个"分水岭"。当操作系统准备好后，它调用 `ExitBootServices()`，意思是："好了固件兄弟们，谢谢你们的服务，从现在开始我自己来了。" 调用之后：
- 所有 Boot Services **永久失效**，内存被操作系统回收
- 固件的事件、定时器全部失效
- 只有 Runtime Services 还继续工作

> Boot Services 就像婚礼上的伴郎伴娘——典礼期间帮忙张罗一切，仪式完成后就可以撤了。Runtime Services 则像双方的父母——婚后还得一直在。

---

## 四、Runtime Services（运行时服务）—— 开机后还在值班的几个人

操作系统启动后，绝大部分固件代码都"下班"了，但有少数服务必须继续提供：

```
Runtime Services (操作系统启动后仍然可用):

┌───────────────────────────────────────────────────────────┐
│                Runtime Services (永不关门)                  │
├─────────────────┬─────────────────────────────────────────┤
│ UEFI 变量       │ GetVariable()    - 读取 UEFI 变量       │
│                 │ SetVariable()    - 写入 UEFI 变量       │
│                 │ 用途: 保存启动顺序、安全启动密钥等          │
├─────────────────┼─────────────────────────────────────────┤
│ 时间管理        │ GetTime()        - 读取 RTC 时间         │
│                 │ SetTime()        - 设置 RTC 时间         │
├─────────────────┼─────────────────────────────────────────┤
│ 系统重置        │ ResetSystem()    - 重启/关机              │
│                 │ ColdReset / WarmReset / Shutdown          │
├─────────────────┼─────────────────────────────────────────┤
│ 虚拟地址        │ SetVirtualAddressMap()                    │
│                 │ 让 OS 把固件的物理地址映射到虚拟地址        │
├─────────────────┼─────────────────────────────────────────┤
│ 胶囊更新        │ UpdateCapsule()  - 固件在线更新           │
│                 │ 操作系统下发新固件，下次重启自动刷入          │
└─────────────────┴─────────────────────────────────────────┘
```

**为什么操作系统还需要固件的服务？**

- **读写 UEFI 变量**：Windows 的 `bcdedit` 修改启动项、Linux 的 `efivarfs` 读写安全启动密钥，底层都是调用 Runtime Services 的 `GetVariable()` / `SetVariable()`
- **系统重启**：你点"重启"时，OS 最终调用的是固件的 `ResetSystem()`
- **固件更新**：Windows Update 推送 BIOS 更新时，通过 `UpdateCapsule()` 把新固件交给固件处理

> Runtime Services 就像物业管理——业主（OS）入住后，大部分装修工人都撤了，但物业还得 24 小时值班，管水电（变量读写）、安保（安全启动）、维修（固件更新）。

---

## 五、Protocol（协议）—— UEFI 的灵魂机制

**Protocol 是 UEFI 中最核心、最重要的概念。** 没有之一。

Protocol 本质上就是一个**函数指针表**（struct里面全是函数指针），每个 Protocol 通过一个 **GUID**（全局唯一标识符）来标识。

```c
// 一个 Protocol 长什么样？以块设备 IO 为例:

struct EFI_BLOCK_IO_PROTOCOL {
    UINT64          Revision;
    EFI_BLOCK_IO_MEDIA  *Media;           // 设备信息
    EFI_BLOCK_RESET     Reset;            // 函数指针: 重置设备
    EFI_BLOCK_READ      ReadBlocks;       // 函数指针: 读数据块
    EFI_BLOCK_WRITE     WriteBlocks;      // 函数指针: 写数据块
    EFI_BLOCK_FLUSH     FlushBlocks;      // 函数指针: 刷缓存
};

// 对应的 GUID:
// EFI_BLOCK_IO_PROTOCOL_GUID =
//   {964e5b21-6459-11d2-8e39-00a0c969723b}
```

**为什么这么设计？** 因为 UEFI 要实现**松耦合**：

```
传统 BIOS 的做法 (紧耦合):
  应用程序 → 直接调用 INT 13h → 直接操作硬盘寄存器
  问题: 换个硬盘型号, 整个调用链可能都得改

UEFI 的做法 (松耦合):
  应用程序 → 通过 GUID 查找 BlockIO Protocol → 调用 ReadBlocks()
  好处: 不管底层是 NVMe、SATA、还是 UFS，
       只要驱动实现了 BlockIO Protocol，上层代码不用改
```

Protocol 的使用模式：

```
1. 驱动安装 Protocol:
   gBS->InstallProtocolInterface(
     &Handle,
     &gEfiBlockIoProtocolGuid,    // GUID
     EFI_NATIVE_INTERFACE,
     &mBlockIoProtocol            // 函数指针表
   );

2. 使用者查找 Protocol:
   gBS->LocateProtocol(
     &gEfiBlockIoProtocolGuid,    // 通过 GUID 找
     NULL,
     &BlockIo                     // 拿到函数指针表
   );

3. 使用者调用:
   BlockIo->ReadBlocks(BlockIo, MediaId, Lba, BufferSize, Buffer);
```

> Protocol 就像外卖平台——你不需要知道做饭的是谁，只需要在平台上搜"宫保鸡丁"（GUID），找到一家店（Handle），下单（调用函数指针）就行。底层是大厨还是AI做菜，你不关心。

---

## 六、Boot Manager（启动管理器）—— 决定从哪里启动

UEFI 规范定义了一套标准的启动管理机制：

```
UEFI 启动管理器的工作流程:

  1. 读取 UEFI 变量 "BootOrder"
     → 值: 0003, 0001, 0002, 0000
     → 含义: 先试第3个启动项, 再试第1个, 再第2个, 最后第0个

  2. 根据编号读取对应的启动项变量:
     Boot0000: Windows Boot Manager  → \EFI\Microsoft\Boot\bootmgfw.efi
     Boot0001: Ubuntu                → \EFI\ubuntu\shimx64.efi
     Boot0002: USB Drive             → PciRoot(0x0)/Pci(0x14,0x0)/USB(...)
     Boot0003: UEFI Shell            → \EFI\Shell\Shell.efi

  3. 按 BootOrder 顺序尝试启动:
     → 尝试 Boot0003 (UEFI Shell)... 不存在, 跳过
     → 尝试 Boot0001 (Ubuntu)... 找到了! 启动!

  如果全部失败:
     → 搜索每个 FAT 分区的 \EFI\BOOT\BOOTX64.EFI (默认启动文件)
```

**关键变量**：

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `BootOrder` | UINT16[] | 启动项的尝试顺序 |
| `Boot0000` ~ `BootFFFF` | 结构体 | 每个启动项的详细信息（名称、设备路径、参数） |
| `BootCurrent` | UINT16 | 当前正在启动的项目编号 |
| `BootNext` | UINT16 | 下次重启临时使用的启动项（一次性） |
| `Timeout` | UINT16 | 启动菜单等待时间（秒） |

> 你在 BIOS 设置里调整"启动顺序"时，实际上就是在修改 `BootOrder` 这个 UEFI 变量。Windows 的 `bcdedit` 命令、Linux 的 `efibootmgr` 命令，底层都是在读写这些变量。

---

## 七、Driver Model（UEFI 驱动模型）—— 驱动的标准写法

UEFI 规范定义了一套标准的驱动编写模型，核心是三个 Protocol：

```
UEFI 驱动模型的三巨头:

┌──────────────────────────────────────────┐
│  EFI_DRIVER_BINDING_PROTOCOL              │
│                                          │
│  Supported()  - "这个设备我能不能驱动？"    │
│  Start()      - "好的，我开始驱动它了"      │
│  Stop()       - "行，我不管它了"            │
└──────────────────────────────────────────┘

工作流程:

  DXE Core 发现新硬件 (Handle)
      ↓
  遍历所有已加载的驱动，调用每个驱动的 Supported()
      ↓
  驱动A: Supported() → "不认识这设备" → 跳过
  驱动B: Supported() → "不认识这设备" → 跳过
  驱动C: Supported() → "这是 NVMe 控制器！我认识！" → 匹配！
      ↓
  DXE Core 调用驱动C 的 Start()
      ↓
  驱动C 初始化 NVMe 控制器，安装 BlockIO Protocol
      ↓
  其他模块/OS Loader 就可以通过 BlockIO 读写 NVMe 了
```

这个模型和 Windows 的 PnP（即插即用）驱动模型非常像——操作系统发现设备后，找到匹配的驱动来管理它。

> 这就像招聘会：新来一个岗位需求（设备），所有求职者（驱动）排队面试（Supported），谁说"这活我能干"就安排谁上岗（Start），干得不好就辞退（Stop）。

---

## 八、规范之外：PI 规范

你可能注意到了，前面讲的 SEC → PEI → DXE 四阶段启动流程在 UEFI 规范里**找不到**。

这是因为启动流程定义在另一份规范里——**PI 规范（Platform Initialization Specification）**：

```
两份规范的分工:

┌────────────────────────────────────┐
│  PI 规范 (平台初始化规范)            │
│                                    │
│  规定: 固件内部怎么启动              │
│  SEC → PEI → DXE → BDS            │
│  PPI、HOB、FV 等内部机制            │
│                                    │
│  面向: 固件开发者 (你!)              │
└────────────────────────────────────┘
          ↕ 合在一起 = 完整的固件标准
┌────────────────────────────────────┐
│  UEFI 规范 (统一可扩展固件接口规范)   │
│                                    │
│  规定: 固件对外提供什么服务           │
│  Boot Services / Runtime Services  │
│  Protocol / Driver Model           │
│  Boot Manager                      │
│                                    │
│  面向: 固件开发者 + OS 开发者        │
└────────────────────────────────────┘
```

简单记忆：
- **PI 规范** = 固件内部的游戏规则（后厨怎么做菜）
- **UEFI 规范** = 固件对外的服务标准（餐厅菜单和服务流程）

> PI 规范管的是"厨房里怎么切菜炒菜"，UEFI 规范管的是"顾客点餐、上菜、结账的流程"。你是厨师（BSP 工程师），两本规范都得看。

---

## 九、实际代码中的规范对应

以你可能接触到的 EDK2 代码为例，各个概念在代码中的位置：

| UEFI 概念 | EDK2 代码位置 | 说明 |
|-----------|--------------|------|
| Boot Services 实现 | `MdeModulePkg/Core/Dxe/DxeMain.c` | 那个巨大的 `mBootServices` 函数指针表 |
| Runtime Services 实现 | `MdeModulePkg/Core/RuntimeDxe/` | 运行时服务的 DXE 驱动 |
| Protocol 定义 | `MdePkg/Include/Protocol/` | 几百个 `.h` 文件，每个定义一个 Protocol |
| Boot Manager | `MdeModulePkg/Universal/BdsDxe/BdsEntry.c` | 读 BootOrder、枚举启动项 |
| Driver Model | 每个 DXE 驱动的 `Supported/Start/Stop` | 驱动绑定三件套 |
| GUID 定义 | `MdePkg/Include/Guid/` | 各种 GUID 常量定义 |
| UEFI 变量实现 | `MdeModulePkg/Universal/Variable/` | 变量的读写和 NV 存储 |

---

## 十、一张图总结

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   ┌─────────── PI 规范的地盘 ────────────┐                       │
│   │                                     │                       │
│   │  SEC → PEI → DXE                   │                       │
│   │  (固件内部初始化)                     │                       │
│   │                                     │                       │
│   └─────────────┬───────────────────────┘                       │
│                 │                                               │
│   ┌─────────── UEFI 规范的地盘 ──────────────────────────┐       │
│   │             ↓                                       │       │
│   │         BDS (启动管理器)                              │       │
│   │             ↓                                       │       │
│   │    ┌─── Boot Services ───┐   ┌─ Runtime Services ─┐ │       │
│   │    │ 内存管理             │   │ UEFI 变量          │ │       │
│   │    │ Protocol 管理        │   │ 时间管理           │ │       │
│   │    │ 事件管理             │   │ 系统重置           │ │       │
│   │    │ 镜像加载             │   │ 胶囊更新           │ │       │
│   │    └────────┬────────────┘   └────────┬───────────┘ │       │
│   │             │                         │             │       │
│   │      ExitBootServices()               │(继续运行)    │       │
│   │             ↓                         ↓             │       │
│   │     OS Loader → OS Kernel → 操作系统运行中            │       │
│   └─────────────────────────────────────────────────────┘       │
│                                                                 │
│   ┌── UEFI 驱动模型 ──┐    ┌── Protocol 机制 ──┐                │
│   │ Supported/Start/  │    │ Install/Locate/   │                │
│   │ Stop 三件套       │    │ Open Protocol     │                │
│   └──────────────────┘    └──────────────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 十一、小结

| 概念 | 一句话 |
|------|--------|
| **UEFI 规范** | 固件与操作系统之间的接口合同 |
| **Boot Services** | 开机期间的临时服务，OS 接管后失效 |
| **Runtime Services** | OS 接管后仍然在工作的固件服务 |
| **Protocol** | 通过 GUID 查找的函数指针表，UEFI 的灵魂 |
| **Boot Manager** | 管理 BootOrder，决定从哪个设备启动 |
| **Driver Model** | 驱动的标准三件套：Supported/Start/Stop |
| **PI 规范** | 固件内部启动流程的规范（SEC/PEI/DXE） |

> 记住：UEFI 规范 2800 页看起来吓人，但核心逻辑就这五块。理解了这些概念，后面读规范就是在补细节了。

---

## 下一篇预告

**007. EDK2是什么？为什么它是UEFI开发的事实标准**

UEFI 规范定义了"要做什么"，EDK2 就是"怎么做"的参考实现。来看看这个开源项目到底是什么来头。

---

> 📌 本系列共 300 篇，从入门到芯片原厂 BSP 能耗优化
>
> 👨‍💻 作者：在职 BSP 开发工程师，持续更新中
>
> 🏷️ 标签：UEFI 规范、Boot Services、Runtime Services、Protocol、GUID、Driver Model、PI 规范
>
> 📁 GitHub：[UEFI-BSP-300](https://github.com/kebin988/UEFI-BSP-300)
