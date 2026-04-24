# UEFI Shell 是什么？它能干什么？

> 🔥 UEFI/BSP 开发系列第 011 篇 | 难度：⭐ 入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们在 UEFI Shell 里运行了 HelloWorld.efi。

但你有没有好奇：**这个 Shell 到底是什么？它和 Windows 的 CMD、Linux 的 Bash 有什么关系？**

UEFI Shell 可能是你用过的**最底层的命令行工具**——它运行在操作系统启动之前，直接和硬件打交道。在这个环境里，你可以读写内存、查看 PCI 设备、操作磁盘分区、跑 EFI 程序……甚至可以写脚本自动化操作。

对于固件工程师来说，UEFI Shell 是**调试利器中的利器**，地位相当于 Linux 工程师手里的 Bash + GDB + strace 的合体。

---

## 一、UEFI Shell 的定位

先搞清楚 UEFI Shell 在整个系统架构中的位置：

```
┌─────────────────────────────────┐
│  Windows / Linux / macOS        │  ← 操作系统
├─────────────────────────────────┤
│  OS Boot Loader                 │  ← 引导加载程序
├─────────────────────────────────┤
│  UEFI Shell                     │  ← 就在这层！
├─────────────────────────────────┤
│  UEFI 固件（DXE 驱动、Protocol）  │  ← 固件核心
├─────────────────────────────────┤
│  硬件（CPU、内存、PCIe、USB...）  │  ← 物理硬件
└─────────────────────────────────┘
```

UEFI Shell 运行在 BDS 阶段（启动设备选择阶段），这时候：
- ✅ 内存已经初始化完毕
- ✅ 所有 DXE 驱动已经加载（PCI、USB、网络、文件系统都可用）
- ❌ 操作系统还没有启动

> 💡 **一句话理解**：UEFI Shell = 操作系统启动前的命令行环境，能直接访问硬件。

---

## 二、UEFI Shell 长什么样？

在 QEMU 或者真机中进入 UEFI Shell 后，你会看到：

```
UEFI Interactive Shell v2.2
EDK II
UEFI v2.70 (EDK II, 0x00010000)
Mapping table
      FS0: Alias(s):F0:;BLK1:
          PciRoot(0x0)/Pci(0x1,0x1)/Ata(0x0)/HD(1,MBR,0xBE1AFDFA,0x3F,0xFBFC1)
      BLK0: Alias(s):
          PciRoot(0x0)/Pci(0x1,0x1)/Ata(0x0)
Press ESC in 5 seconds to skip startup.nsh or any other key to continue.
Shell>
```

那个 `Mapping table` 列出了当前识别到的存储设备。`FS0:` 表示第一个文件系统（类似 Windows 的 C: 盘）。

---

## 三、常用命令速查

UEFI Shell 的命令和 DOS/Linux 有些相似，但又有自己的特色。

### 文件和目录操作

| 命令 | 作用 | 类似 |
|------|------|------|
| `ls` 或 `dir` | 列出文件 | Linux `ls` / Windows `dir` |
| `cd` | 切换目录 | 一样 |
| `cp` | 复制文件 | 一样 |
| `mv` | 移动/重命名 | 一样 |
| `rm` | 删除文件 | 一样 |
| `mkdir` | 创建目录 | 一样 |
| `cat` 或 `type` | 查看文件内容 | 一样 |
| `edit` | 内置文本编辑器 | 类似 nano |

### 系统信息

| 命令 | 作用 |
|------|------|
| `ver` | 显示 UEFI Shell 和固件版本 |
| `map` | 显示设备映射表（存储设备、网络设备等） |
| `memmap` | 显示内存映射（哪些地址被谁占用） |
| `smbiosview` | 查看 SMBIOS 表（主板、CPU、内存等硬件信息） |
| `dmpstore` | 查看 UEFI 变量 |

### 硬件调试（固件工程师最爱）

| 命令 | 作用 |
|------|------|
| `pci` | 列出所有 PCI 设备 |
| `devtree` | 以树形结构显示设备 |
| `dh` | 显示所有 Handle 和 Protocol |
| `drivers` | 列出所有已加载的驱动 |
| `connect` | 手动连接驱动到设备 |
| `disconnect` | 断开驱动连接 |
| `mm` | 读写内存/IO 端口/PCI 配置空间 |

### 执行和调试

| 命令 | 作用 |
|------|------|
| `xxx.efi` | 直接运行 EFI 程序 |
| `load xxx.efi` | 加载 EFI 驱动 |
| `unload` | 卸载驱动 |
| `set` | 设置/查看环境变量 |
| `help` | 查看帮助 |
| `help <cmd>` | 查看某个命令的详细用法 |

---

## 四、实用操作示例

### 4.1 查看 PCI 设备列表

```
Shell> pci
   Seg  Bus  Dev  Func
   ---  ---  ---  ----
    00   00   00    00 ==> Bridge Device - Host/PCI bridge
                         Vendor 8086 Device 1237 Prog Interface 0
    00   00   01    00 ==> Bridge Device - ISA bridge
                         Vendor 8086 Device 7000 Prog Interface 0
    00   00   01    01 ==> Mass Storage Controller - IDE controller
                         Vendor 8086 Device 7010 Prog Interface 80
    ...
```

Vendor `8086` 就是 Intel（8086 处理器的致敬）。在真机上，你能看到所有 PCIe 设备——显卡、网卡、NVMe 控制器……

### 4.2 读写内存

```
Shell> mm 0xFED40000  # 读取某个 MMIO 地址
MEM  0x00000000FED40000 : 0x00060014

Shell> mm 0xFED40000 0x12345678 -w 4  # 写入 4 字节
```

> ⚠️ **危险操作！** 随意写内存/MMIO 可能导致系统死机。但对于固件工程师来说，这就是调试硬件寄存器的日常。

### 4.3 查看 UEFI 变量

```
Shell> dmpstore -all
Variable NV+RT+BS 'EFIGlobalVariable:Boot0000' DataSize = 0x64
  00000000: 01 00 00 00-62 00 55 00-62 00 75 00-6E 00 74 00  *....b.U.b.u.n.t.*
  ...

Variable NV+RT+BS 'EFIGlobalVariable:BootOrder' DataSize = 0x06
  00000000: 00 00 01 00-02 00                                *......*
```

这些就是 UEFI 的 NVRAM 变量。`BootOrder` 控制启动顺序，`Boot0000` 定义第一个启动项。你在 BIOS 设置界面调整启动顺序，本质上就是修改这些变量。

### 4.4 查看设备树

```
Shell> devtree
Ctrl[00] PciRootBridge
  Ctrl[01] PCI(0,0)
  Ctrl[02] PCI(1,0)
    Ctrl[03] PCI(1,1) - IDE Controller
      Ctrl[04] ATA Channel 0
        Ctrl[05] HD(1,MBR)
          Ctrl[06] FAT File System
  Ctrl[07] PCI(2,0) - VGA
  ...
```

---

## 五、UEFI Shell 脚本

UEFI Shell 支持脚本（`.nsh` 文件），语法类似 DOS 批处理：

```nsh
# startup.nsh - 自动启动脚本
@echo off
echo "=== Auto Boot Script ==="
echo "Current time:"
date
time
echo "PCI devices:"
pci
echo "Booting HelloWorld..."
fs0:\HelloWorld.efi
```

把 `startup.nsh` 放在 FAT 分区根目录，UEFI Shell 启动时会自动执行它——就像 Linux 的 `.bashrc` 或 Windows 的 `autoexec.bat`。

### 脚本语法要点

```nsh
# 变量
set myvar "hello"
echo %myvar%

# 条件判断
if exist fs0:\test.efi then
  fs0:\test.efi
endif

# 循环
for %i in 1 2 3
  echo "Loop %i"
endfor

# 跳转
goto :label
:label
echo "jumped here"
```

---

## 六、UEFI Shell 在实际工作中的用途

### 固件工程师的日常场景

| 场景 | 用法 |
|------|------|
| **验证 PCI 设备是否枚举成功** | `pci` 查看设备列表 |
| **调试 NVMe 识别问题** | `map` 查看是否出现 FS 设备 |
| **读写硬件寄存器** | `mm` 读 MMIO 地址 |
| **查看内存布局** | `memmap` 确认内存分配 |
| **刷写 BIOS 变量** | `dmpstore` 查看/修改 NVRAM |
| **运行测试工具** | 直接跑 `.efi` 测试程序 |
| **自动化测试** | `.nsh` 脚本批量执行 |

### 一个真实场景

你刚拿到一颗新芯片的开发板，BIOS 刚点亮（能进 Shell），但 NVMe 硬盘没被识别。

调试步骤：
1. `pci` —— 看 NVMe 控制器是否出现在 PCI 设备列表
2. `dh -d -v` —— 看是否有 NVMe PassThru Protocol
3. `drivers` —— 看 NVMe 驱动是否加载
4. `connect -r` —— 尝试重新连接所有驱动
5. `map -r` —— 刷新设备映射
6. `mm` —— 读 NVMe 控制器的 BAR 寄存器，看 Link Training 状态

5 分钟定位问题，比进 OS 调试快得多。

---

## 七、如何进入 UEFI Shell？

### UEFI Shell 的输出方式：显示器 or 串口？

这是一个初学者常有的疑问：UEFI Shell 到底是在哪里"显示"的？

**答案：两个都行，看你怎么连接。**

```
┌──────────────────────────────────────────────┐
│              UEFI Shell 输出                   │
│                                              │
│  方式 1：显示器 + 键盘                         │
│  ┌──────────┐     ┌──────────┐              │
│  │ 显示器    │ ←── │ GOP 驱动  │  通过显卡输出  │
│  │ Shell>    │     │          │              │
│  └──────────┘     └──────────┘              │
│                                              │
│  方式 2：串口 + 终端软件                        │
│  ┌──────────┐     ┌──────────┐              │
│  │ PuTTY    │ ←── │ SerialIO  │  通过串口线    │
│  │ Shell>    │     │ 驱动     │              │
│  └──────────┘     └──────────┘              │
│                                              │
│  方式 3：两个同时输出（最常见的开发配置）         │
└──────────────────────────────────────────────┘
```

**消费级电脑**（你的笔记本/台式机）：直接连显示器 + 键盘，在屏幕上看到 Shell 界面并操作。你在 QEMU 里看到的弹出窗口就是模拟的"显示器"。

**开发板/服务器**：BSP 工程师通常通过**串口**（Serial Port）连接另一台电脑，用终端软件（PuTTY、SecureCRT、minicom）查看 Shell 输出。原因有三：
1. 开发板可能没有显卡/显示器接口
2. 串口可以完整记录所有日志（包括启动过程中的 DEBUG 打印），显示器只能看到最终画面
3. 串口可以远程操作——开发板放在实验室，你在工位上通过串口连过去

**实际工作中最常见的配置**：显示器和串口**同时开启**。显示器看界面，串口抓日志。

### 在不同设备上进入 UEFI Shell

### QEMU 虚拟机

直接用 OVMF 固件启动，默认就进入 UEFI Shell。

### 真实电脑

方法因主板/品牌而异：

1. **开机按 F2/Del 进入 BIOS 设置** → 找到 Boot 选项 → 选择 "UEFI Shell" 或 "Built-in EFI Shell"
2. **准备一个 U 盘**，格式化为 FAT32，把 `Shell.efi` 放到 `\EFI\BOOT\BOOTX64.EFI`，从 U 盘启动
3. 某些主板支持开机按 **F12** 直接选择 UEFI Shell

> Shell.efi 可以从 EDK2 编译产物中获取：`Build/OvmfX64/DEBUG_GCC5/X64/Shell.efi`

---

## 八、总结

| 维度 | 内容 |
|------|------|
| **是什么** | UEFI 固件环境中的命令行工具 |
| **运行时机** | 操作系统启动前（BDS 阶段） |
| **核心能力** | 文件操作、硬件调试、驱动管理、脚本执行 |
| **杀手级功能** | `pci`、`mm`、`memmap`、`dh` —— 直接和硬件对话 |
| **对标工具** | Linux Shell + 硬件调试器的合体 |
| **实际用途** | BSP 开发中的首选调试环境 |

---

下一篇：**#012 在 QEMU 中运行你的第一个 UEFI 固件**——详细讲解 QEMU 的各种启动参数和调试技巧。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
