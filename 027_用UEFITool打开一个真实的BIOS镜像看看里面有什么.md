# 用 UEFITool 打开一个真实的 BIOS 镜像看看里面有什么

> 🔥 UEFI/BSP 开发系列第 027 篇 | 难度：⭐ 入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们搞明白了 FD、FV、FFS、Section 这四层套娃的理论知识。

理论终究是理论，就像光看菜谱不做饭——你永远不知道锅气是什么味儿。

今天我们**动手操作**：从网上下载一个真实的 BIOS 固件文件，用开源神器 **UEFITool** 打开，亲眼看看 UEFI 固件的内部世界。

> 💡 这篇文章全程可以跟着操作。你不需要有任何开发环境，只需要一台能上网的电脑。

---

## 一、UEFITool 是什么？

UEFITool 是俄罗斯大佬 **Nikolaj Schlej（LongSoft）** 开发的开源 UEFI 固件解析工具，托管在 GitHub：

🔗 https://github.com/LongSoft/UEFITool

它能做什么？

| 功能 | 说明 |
|------|------|
| 打开 BIOS 镜像 | 支持各种厂商的 BIOS 文件（.bin/.rom/.cap/.fd 等） |
| 树形浏览 | 层级展示 FD → FV → FFS → Section 结构 |
| 搜索 | 按 GUID / 文本 / 十六进制模式搜索 |
| 提取 | 把任意模块或 Section 提取为文件 |
| 替换 | 替换某个 Section 的内容（改 BIOS 用） |
| 插入/删除 | 增减固件模块 |

如果说 UEFI 固件是一栋大楼，UEFITool 就是那副**X 光透视眼镜**——让你一眼看穿每面墙后面都藏了什么。

> 在固件工程师圈子里，UEFITool 的地位大概等于前端程序员的 Chrome DevTools——没它寸步难行。

### 两个版本

UEFITool 有两个版本，别下错了：

| 版本 | 说明 | 推荐 |
|------|------|------|
| **UEFITool**（经典版） | 老版本，功能完整，可以编辑 | 需要修改 BIOS 时用 |
| **UEFITool NE**（New Engine） | 新版本，解析更准确，只能查看 | ✅ 日常分析推荐 |

> 新版本叫 NE（New Engine），解析引擎完全重写，对新平台固件的支持更好。但它是**只读**的——只能看不能改。如果你需要替换模块，还得用经典版。

---

## 二、准备工作

### 第 1 步：下载 UEFITool

打开 GitHub Release 页面：

🔗 https://github.com/LongSoft/UEFITool/releases

找到最新版本，下载对应你系统的版本：
- Windows 用户：下载 `UEFITool_NE_xxx_win64.zip`
- macOS 用户：下载 `UEFITool_NE_xxx_mac.zip`
- Linux 用户：下载 `UEFITool_NE_xxx_linux_x86_64.zip`

解压就能用，不需要安装。**绿色软件，懂的都懂**。

### 第 2 步：弄到一个 BIOS 镜像

你有三种方式拿到 BIOS 文件：

**方式 A：从主板厂商官网下载（最简单）**

1. 去你主板品牌的官网（华硕 / 微星 / 技嘉 / 华擎）
2. 找到你主板型号的支持页面
3. 下载最新的 BIOS 更新文件
4. 通常是一个 `.CAP`、`.ROM` 或 `.bin` 文件

> 举个例子：去技嘉官网，找到某款主板，下载 BIOS 更新 → 得到一个类似 `B660MAORUSPROWIFI.F25` 的文件。这就是一个完整的 BIOS 镜像。

**方式 B：从你自己的电脑导出**

如果你胆子大（且不怕翻车），可以用工具从自己电脑的 SPI Flash 上把 BIOS 读出来：
- Windows：Intel CSME System Tools 里的 `fptw64.exe -d full.bin`
- Linux：`flashrom -p internal -r backup.bin`

> ⚠️ 警告：读取 BIOS 是安全的（只读不写），但如果你接下来手贱写回去了……那就是另一个故事了。

**方式 C：用 EDK2 编译出来的**

如果你之前跟着第 010/012 篇搭建了开发环境，编译出来的 `OVMF.fd` 就是一个标准的 UEFI 固件镜像，可以直接用 UEFITool 打开。

---

## 三、打开 BIOS 镜像：第一印象

双击 UEFITool（或命令行运行），然后 `File → Open Image File`，选择你的 BIOS 文件。

打开后你会看到类似这样的树形结构（以一个 Intel 平台的 BIOS 为例）：

```
▼ Intel image                              ← 整个 Flash 镜像（FD 层面）
  ▼ Descriptor region                      ← Flash 描述符区域
  ▼ ME region                              ← Intel ME 固件区域
  ▼ BIOS region                            ← BIOS 区域（我们关心的部分）
    ▼ Padding                              ← 填充区域
    ▼ Volume                               ← 这就是一个 FV！
      ▼ File: Volume Top File              ← FV 的"入口"
      ▼ File: DxeCore                      ← DXE 核心
      ▼ File: PcdDxe                       ← PCD 数据库驱动
      ▼ File: PciBusDxe                    ← PCI 总线驱动
      ▼ File: UsbBusDxe                    ← USB 总线驱动
      ▼ File: NvmExpressDxe                ← NVMe 硬盘驱动
      ▼ File: GraphicsOutputDxe            ← 显示输出驱动
      ▼ File: ConSplitterDxe               ← 控制台分配器
      ▼ File: BdsDxe                       ← Boot Device Selection
      ... (几百个文件，密密麻麻)
    ▼ Volume                               ← 另一个 FV
      ▼ File: PeiCore                      ← PEI 核心
      ▼ File: MemoryInit                   ← 内存初始化
      ▼ File: CpuMpPei                     ← 多核初始化
      ...
    ▼ Volume (NVRAM)                       ← NV 变量区
      ▼ VSS2 store                         ← 变量存储
        ▼ BootOrder                        ← 启动顺序变量
        ▼ Boot0001                         ← 启动项 1
        ▼ SecureBoot                       ← 安全启动开关
        ...
```

第一次打开的感觉大概是这样的：

> "卧槽，原来一个 BIOS 里面有这么多东西？？"

没错。一个典型的商用 BIOS 里面有 **300~500 个模块**。你在 BIOS 设置界面看到的那些选项，背后都是真实的固件代码在支撑。

---

## 四、看懂树形结构

UEFITool 的树形结构就是上一篇讲的 FD → FV → FFS → Section。我们一层层看：

### 第一层：Region（区域）

```
▼ Intel image
  ├── Descriptor region   → Flash 描述符（4KB，定义各区域边界）
  ├── ME region           → Intel ME 固件（你打不开也看不懂，正常）
  ├── GbE region          → 千兆网卡固件（可选）
  └── BIOS region         → BIOS 代码（我们的主战场）
```

> 💡 如果你打开一个 AMD 平台的 BIOS，可能看不到 "Intel image" 字样——不同平台的 Flash 布局不同。AMD 通常有 PSP（Platform Security Processor）区域而不是 ME 区域。

### 第二层：Volume（固件卷 = FV）

在 BIOS region 下面，你会看到多个 `Volume`，每个就是一个 FV：

```
▼ BIOS region
  ├── Volume (FV_PEI)        → PEI 阶段的代码，最先执行
  ├── Volume (FV_DXE)        → DXE 阶段的代码，主力驱动都在这
  ├── Volume (FV_ADVANCED)   → 高级功能（可选）
  └── Volume (NVRAM)         → UEFI 变量存储区
```

点击某个 Volume，右边的面板会显示它的详细信息：

```
Type:           EFI Firmware Volume
Subtype:        FFSv2  (或 FFSv3)
Full size:      0x800000 (8 MB)
Header size:    0x48
Body size:      0x7FFFB8
...
```

### 第三层：File（FFS 文件）

展开一个 Volume，你会看到一堆 File。每个 File 就是一个 FFS 文件——一个独立的固件模块：

```
▼ Volume (FV_DXE)
  ├── File: DxeCore            GUID: D6A2CB7F-...   Type: DXE core
  ├── File: PcdDxe             GUID: 80CF7257-...   Type: DXE driver
  ├── File: PciBusDxe          GUID: 93B80004-...   Type: DXE driver
  ├── File: UsbBusDxe          GUID: 240612B7-...   Type: DXE driver
  ├── File: NvmExpressDxe      GUID: 5BE3BDF4-...   Type: DXE driver
  ...
```

注意看每个 File 的信息：
- **名字**：有些模块有 UI Section（可读名字），有些只能看 GUID
- **GUID**：唯一标识，上一篇讲过
- **Type**：文件类型（DXE driver / PEIM / Application / ...）

### 第四层：Section（段）

展开一个 File，你能看到它的各个 Section：

```
▼ File: PciBusDxe
  ├── DXE dependency section   → 依赖条件：需要哪些 Protocol 先就位
  ├── Compressed section       → 压缩容器
  │   └── PE32 image section   → 实际的可执行代码（.efi）
  └── User interface section   → 模块名称："PciBusDxe"
```

> 💡 **划重点**：PE32 image section 就是你编译出来的 `.efi` 文件的本体。你可以右键 → `Extract body` 把它提取出来——提取后就是一个标准的 PE32+ 可执行文件，可以用 IDA Pro、Ghidra 等工具逆向分析。

---

## 五、实战：搜索功能

UEFITool 最强大的功能之一是**搜索**。按 `Ctrl+F`（经典版）或 `Action → Search`（NE 版），你可以按三种方式搜索：

### 1. 按 GUID 搜索

知道一个 Protocol 或模块的 GUID？直接搜：

```
搜索 GUID：93B80004-9FB3-11D4-9A3A-0090273FC14D
结果：找到 PciBusDxe 模块
```

### 2. 按文本搜索（Unicode/ASCII）

想看看 BIOS 里有没有某些字符串？

```
搜索文本："Secure Boot"
结果：找到安全启动相关的模块和界面文本

搜索文本："Copyright"
结果：看到各个模块的版权信息

搜索文本：你的主板型号
结果：可能找到 SMBIOS 数据（系统信息）
```

### 3. 按十六进制搜索

搜索特定的二进制模式：

```
搜索 Hex：4D5A      (就是 "MZ"，PE 文件头的魔数)
结果：找到所有 PE32 image section
```

---

## 六、实战：提取一个模块

右键点击某个 File 或 Section，你会看到这些选项：

- **Extract as is** — 原样提取（包含 FFS/Section 头部）
- **Extract body** — 只提取内容（去掉头部包装）

**最常用的场景**：提取某个驱动的 PE32 body，然后用反汇编工具分析。

```
操作步骤：
1. 展开目标 File（比如 PciBusDxe）
2. 找到 "PE32 image section"
3. 右键 → Extract body
4. 保存为 PciBusDxe.efi
5. 用 Ghidra / IDA 打开分析
```

> ⚠️ **友情提示**：商用 BIOS 中的模块受版权保护。提取仅供个人学习和调试，请勿用于商业目的或非法传播。做一个遵纪守法的技术人——代码虽好，可不要贪杯哦。

---

## 七、实战：查看 NVRAM 变量

在 UEFITool NE 中，展开 NVRAM 区域的 Volume，你能看到 VSS（Variable Storage Store）结构：

```
▼ Volume (NVRAM)
  ▼ VSS2 store
    ├── BootOrder                   → 07 00 01 00 03 00 ...
    ├── Boot0001                    → 启动项 1 的详细信息
    ├── Boot0003                    → 启动项 3 的详细信息
    ├── Boot0007                    → 启动项 7 的详细信息
    ├── SecureBoot                  → 01 (已开启)
    ├── SetupMode                   → 00
    ├── PK                          → Platform Key (几KB)
    ├── KEK                         → Key Exchange Key
    ├── db                          → 允许的签名数据库
    ├── dbx                         → 禁止的签名数据库
    ├── ConOut                      → 控制台输出设备路径
    ├── ConIn                       → 控制台输入设备路径
    ├── Timeout                     → 05 00 (5 秒超时)
    ├── Lang                        → "eng"
    └── ... (几十到上百个变量)
```

> 💡 看到了吗？第 018 篇讲的 UEFI 变量，就在这里。`BootOrder` 里存的 `07 00 01 00 03 00` 表示启动顺序是 Boot0007 → Boot0001 → Boot0003。每两个字节是一个 UINT16 小端序的启动项编号。

---

## 八、看一眼 OVMF 固件

如果你之前编译过 OVMF（第 012 篇），可以直接拿 `OVMF.fd` 来看：

```
▼ UEFI image
  ▼ Padding
  ▼ Volume (PEIFV)
    ├── File: PeiCore
    ├── File: PcdPei
    ├── File: PlatformPei
    ├── File: MemoryInit (QEMU 版简化内存初始化)
    └── ...
  ▼ Volume (DXEFV)
    ├── File: DxeCore
    ├── File: PciBusDxe
    ├── File: QemuVideoDxe (QEMU 虚拟显卡驱动)
    ├── File: VirtioBlkDxe (Virtio 块设备驱动)
    ├── File: UefiShell (UEFI Shell)
    ├── File: BdsDxe
    └── ... (大约 100+ 个模块)
  ▼ Volume (NVRAM)
    └── VSS2 store (初始为空)
```

OVMF 比真实硬件 BIOS 简单得多——没有 Intel ME 区域，没有 Flash Descriptor，驱动也少很多。这正是它适合学习的原因。

> 就像学开车先在驾校的空旷场地练，不要一上来就走快速路——OVMF 就是你的"驾校练习场"。

---

## 九、固件工程师的日常：我平时怎么用 UEFITool

作为一个日常跟 BIOS 打交道的人，我几乎每天都会打开 UEFITool。分享几个最常见的使用场景：

**场景 1："这个模块到底在不在 BIOS 里？"**
```
新加了一个驱动，编译后不知道有没有被打包进去
→ 打开 FD，搜索模块的 GUID → 找到了就在，找不到就不在
```

**场景 2："两个版本的 BIOS 有什么区别？"**
```
客户反馈新版 BIOS 有问题，老版没有
→ 用 UEFITool 分别打开新老版本
→ 搜索可疑模块，对比 GUID 和大小
→ 提取两个版本的 PE32，用 diff 工具比较
```

**场景 3："帮我看看这个 BIOS 的变量区"**
```
客户反馈某个设置项重启后恢复默认
→ 打开 BIOS 镜像，找到 NVRAM 区域
→ 检查变量存储的完整性和可用空间
```

**场景 4："这个模块依赖了什么？"**
```
某个驱动总是加载失败
→ 打开 FD，找到该模块
→ 展开看它的 DXE dependency section
→ 依赖的 Protocol 是不是都有人安装了？
```

---

## 十、常见疑问

### Q1：UEFITool 能直接修改 BIOS 吗？

经典版可以。你可以替换某个 Section 的内容，然后 `File → Save image file` 保存修改后的镜像。但 NE 版不行——它只能查看。

> ⚠️ **修改 BIOS 有风险**。如果改坏了又刷回去，可能导致电脑无法启动（变砖）。除非你有编程器（SPI Flash Programmer）能强制恢复，否则请三思。

### Q2：为什么有些模块只显示 GUID，没有名字？

因为那些模块没有 UI Section（用户界面段）。这在商用 BIOS 里很常见——厂商为了保密，编译时去掉了 UI Section。你只能通过 GUID 去猜它是什么。

### Q3：打开后全是灰色/红色标记是什么意思？

- **绿色/正常**：解析成功
- **黄色**：有小问题但不影响功能（比如填充数据不标准）
- **红色**：解析错误（可能是加密区域、非标准格式、或损坏的数据）

Intel ME 区域通常显示为红色——因为它是加密的，UEFITool 无法解析其内部结构。这是正常现象。

### Q4：UEFITool 之外还有什么类似工具？

| 工具 | 特点 |
|------|------|
| **UEFITool** | GUI，最好用，推荐 |
| **uefi-firmware-parser** | Python 库，命令行，适合脚本自动化 |
| **CHIPSEC** | Intel 开源的固件安全审计工具（下沉到硬件层面） |
| **binwalk** | 通用固件分析工具，也能识别 UEFI 结构 |
| **Ghidra + efiXplorer** | 反汇编 + UEFI 模块识别插件 |

---

## 十一、总结

```
UEFITool 使用速查：

打开镜像 → File → Open Image File（支持 .fd/.bin/.rom/.cap）

浏览结构 → 树形展开：Region → Volume → File → Section

搜索功能 → Ctrl+F 或 Action → Search
            ├── GUID 搜索（找模块）
            ├── 文本搜索（找字符串）
            └── Hex 搜索（找二进制模式）

提取模块 → 右键 → Extract body（提取纯代码/数据）

两个版本 → NE 版看分析，经典版做修改
```

| 看完这篇之前 | 看完这篇之后 |
|-------------|-------------|
| BIOS 镜像就是一坨黑盒 | 用 UEFITool 一目了然 |
| 不知道固件里有什么 | 能看到每一个驱动模块 |
| 只会在 BIOS 设置界面点来点去 | 能看到设置界面背后的变量和代码 |
| 固件问题无从下手 | 打开镜像搜一搜，定位问题快又准 |

---

下一篇：**#028 UEFI Secure Boot 最直白的解释**——安全启动到底在防谁？为什么装 Linux 有时候要关掉它？PK、KEK、db、dbx 这些密钥是什么关系？用最通俗的话讲清楚。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
