# 在 QEMU 中运行你的第一个 UEFI 固件

> 🔥 UEFI/BSP 开发系列第 012 篇 | 难度：⭐ 入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

前面几篇我们编译了 OVMF 固件、写了 HelloWorld 程序、了解了 UEFI Shell。

但每次用 QEMU 都只是简单地 `qemu-system-x86_64 -bios OVMF.fd`，对 QEMU 的各种参数一知半解。

今天我们来系统地聊聊 **QEMU** —— 固件工程师的"虚拟实验室"。

为什么 QEMU 这么重要？因为在真实的 BSP 开发中，**不可能每次改一行代码就去烧一次真实主板的 Flash**。QEMU 让你在几秒钟内就能启动一个虚拟 UEFI 平台，快速验证代码改动。

这就像前端开发者有浏览器的热更新，固件开发者有 QEMU。

---

## 一、QEMU 是什么？

QEMU（Quick EMUlator）是一个开源的**系统级仿真器**。它可以模拟完整的计算机硬件：CPU、内存、PCI 总线、USB 控制器、网卡、显卡、硬盘……

对于 UEFI 开发来说，QEMU 配合 OVMF 固件，能模拟一个完整的 UEFI 平台：

```
┌─────────────────────────────────────┐
│            QEMU 虚拟机               │
│                                     │
│  ┌─────────┐  ┌──────┐  ┌───────┐ │
│  │虚拟 CPU  │  │虚拟内存│  │虚拟 PCI│ │
│  └─────────┘  └──────┘  └───────┘ │
│  ┌─────────┐  ┌──────┐  ┌───────┐ │
│  │虚拟显卡  │  │虚拟USB│  │虚拟网卡│ │
│  └─────────┘  └──────┘  └───────┘ │
│  ┌─────────┐  ┌──────┐            │
│  │虚拟硬盘  │  │虚拟串口│            │
│  └─────────┘  └──────┘            │
│                                     │
│  OVMF.fd（UEFI 固件）加载到虚拟 Flash │
└─────────────────────────────────────┘
```

---

## 二、安装 QEMU

### Linux（Ubuntu）

```bash
sudo apt install -y qemu-system-x86
```

### Windows

前往 [QEMU 官网](https://www.qemu.org/download/#windows) 下载 Windows 安装包，安装后将安装路径加入 PATH。

### 验证安装

```bash
qemu-system-x86_64 --version
# QEMU emulator version 7.x.x 或更高
```

---

## 三、基本启动命令

### 最简启动

```bash
qemu-system-x86_64 -bios OVMF.fd
```

这条命令启动一个虚拟机，用 OVMF.fd 作为 UEFI 固件。没有硬盘、没有网络，进入 UEFI Shell。

### 常用参数组合

```bash
qemu-system-x86_64 \
    -bios Build/OvmfX64/DEBUG_GCC5/FV/OVMF.fd \
    -m 256 \
    -smp 2 \
    -drive file=disk.img,format=raw \
    -net none \
    -serial stdio \
    -display gtk
```

### 参数详解

| 参数 | 作用 | 默认值 |
|------|------|--------|
| `-bios <file>` | 指定 UEFI 固件镜像 | QEMU 内置 SeaBIOS（传统 BIOS） |
| `-m <size>` | 虚拟机内存大小 | 128MB |
| `-smp <n>` | CPU 核心数 | 1 |
| `-drive file=<img>` | 添加虚拟磁盘 | 无磁盘 |
| `-net none` | 禁用网络 | 有默认网卡 |
| `-serial stdio` | 串口输出到终端 | 无串口输出 |
| `-display gtk/sdl/none` | 显示方式 | GTK 窗口 |
| `-nographic` | 无图形界面，所有输出走串口 | 有图形窗口 |
| `-S` | 启动后暂停 CPU（配合 GDB） | 立即运行 |
| `-s` | 开启 GDB 调试端口（localhost:1234） | 不开启 |

---

## 四、常用场景配方

### 场景 1：快速验证固件（日常开发）

```bash
qemu-system-x86_64 \
    -bios OVMF.fd \
    -m 256 \
    -serial stdio \
    -net none
```

串口输出到终端，可以看到完整的启动日志。

### 场景 2：运行 EFI 程序（带虚拟磁盘）

```bash
# 先创建磁盘并放入 EFI 程序
dd if=/dev/zero of=disk.img bs=1M count=64
mkfs.fat disk.img
mkdir -p /tmp/mnt && sudo mount disk.img /tmp/mnt
sudo cp HelloWorld.efi /tmp/mnt/
sudo umount /tmp/mnt

# 启动
qemu-system-x86_64 \
    -bios OVMF.fd \
    -drive file=disk.img,format=raw \
    -serial stdio
```

### 场景 3：安装并启动 Linux

```bash
# 下载一个 Linux ISO
qemu-system-x86_64 \
    -bios OVMF.fd \
    -m 2048 \
    -smp 4 \
    -drive file=hdd.qcow2,format=qcow2 \
    -cdrom ubuntu-22.04-live-server-amd64.iso \
    -boot d
```

这可以验证你的 UEFI 固件能否正确引导一个真实的操作系统。

### 场景 4：GDB 源码级调试

```bash
# 终端 1：启动 QEMU，暂停 CPU 并等待 GDB 连接
qemu-system-x86_64 \
    -bios OVMF.fd \
    -serial stdio \
    -S -s

# 终端 2：用 GDB 连接
gdb
(gdb) target remote localhost:1234
(gdb) break UefiMain
(gdb) continue
```

这是后面第 100 篇《用 QEMU+GDB 调试 EDK2 固件》会详细讲的内容。

### 场景 5：无图形模式（服务器/SSH 环境）

```bash
qemu-system-x86_64 \
    -bios OVMF.fd \
    -nographic \
    -serial mon:stdio
```

`-nographic` 关闭图形窗口，所有输出（包括 UEFI Shell）都通过串口发到你的终端。适合在无桌面的 Linux 服务器上使用。

---

## 五、OVMF 固件的高级用法

### 5.1 分离 CODE 和 VARS

OVMF 编译还会生成两个分开的文件：

```
OVMF_CODE.fd  —— 固件代码（只读部分）
OVMF_VARS.fd  —— NVRAM 变量（可写部分）
```

用这种方式启动，NVRAM 变量会被持久化保存：

```bash
# 先复制一份 VARS 模板
cp OVMF_VARS.fd my_vars.fd

qemu-system-x86_64 \
    -drive if=pflash,format=raw,readonly=on,file=OVMF_CODE.fd \
    -drive if=pflash,format=raw,file=my_vars.fd \
    -m 256 -serial stdio
```

> 💡 **好处**：你在 UEFI 设置中修改的启动项、Secure Boot 密钥等 NVRAM 变量会保存到 `my_vars.fd`，重启 QEMU 后仍然生效。而直接用 `-bios OVMF.fd` 的方式，每次重启变量都会丢失。

### 5.2 添加 Secure Boot 支持

```bash
# 使用带有预置密钥的 OVMF 固件
qemu-system-x86_64 \
    -drive if=pflash,format=raw,readonly=on,file=OVMF_CODE.fd \
    -drive if=pflash,format=raw,file=OVMF_VARS.fd \
    -machine q35,smm=on \
    -global driver=cfi.pflash01,property=secure,value=on \
    -m 256
```

`q35` 是一个更现代的虚拟机芯片组模型（模拟 Intel Q35），`smm=on` 启用系统管理模式——Secure Boot 需要 SMM 支持。

---

## 六、调试技巧

### 6.1 查看固件启动日志

UEFI 固件的 DEBUG 输出默认走串口。加上 `-serial stdio` 后你能看到类似这样的日志：

```
SecCoreStartupWithStack(0xFFFCC000, 0x820000)
Install PPI: 8C8CE578-8A3D-4F1C-9935-896185C32DD3
Install PPI: 5473C07A-3DCB-4DCA-BD6F-1E9689E7349A
Loading PEIM at 0x00000820110
  DxeIpl.efi
  ...
Loading driver at 0x00006000000 EntryPoint=0x0000600012C
  PciBusDxe
Loading driver at 0x00006100000 EntryPoint=0x0000610015E
  NvmExpressDxe
  ...
```

每一行 `Loading driver` 就是一个 DXE 驱动在被加载。这是固件工程师最常看的输出。

### 6.2 QEMU Monitor

在 QEMU 运行中按 `Ctrl+A` 然后按 `C`（nographic 模式），可以进入 QEMU Monitor：

```
(qemu) info registers    # 查看 CPU 寄存器
(qemu) info mem           # 查看内存映射
(qemu) info pci           # 查看 PCI 设备
(qemu) xp /16xw 0xFED40000  # 读取物理内存
(qemu) quit               # 退出 QEMU
```

QEMU Monitor 是在 UEFI 之外直接观察虚拟机状态的工具，调试时非常有用。

---

## 七、QEMU vs 真实硬件

| 维度 | QEMU + OVMF | 真实硬件 |
|------|-------------|---------|
| 启动速度 | 秒级 | 需要烧写 Flash |
| 内存初始化 | 简化（没有真实 MRC/FSP） | 完整的 DDR Training |
| PCIe 设备 | 虚拟设备 | 真实芯片 |
| 调试便利性 | GDB 源码调试 | 需要 JTAG/DCI |
| 适用场景 | 框架代码开发、驱动逻辑验证 | BSP 调试、硬件相关问题 |
| 局限性 | 无法测试真实硬件初始化 | 成本高、迭代慢 |

> 💡 **实际开发流程**：先在 QEMU 上验证逻辑 → 再到真实硬件上调试硬件相关部分。这样效率最高。

---

## 八、总结

| 维度 | 内容 |
|------|------|
| **QEMU 定位** | 固件开发的虚拟实验室 |
| **核心命令** | `qemu-system-x86_64 -bios OVMF.fd -serial stdio` |
| **关键参数** | `-m`（内存）、`-drive`（磁盘）、`-serial`（串口）、`-S -s`（GDB） |
| **高级用法** | CODE/VARS 分离、Secure Boot、QEMU Monitor |
| **最佳实践** | 先 QEMU 验证逻辑，再真机调试硬件 |

---

下一篇：**#013 UEFI 的模块化思想：为什么要把 BIOS 拆成几百个小模块？**——从软件工程角度理解 UEFI 的架构哲学。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
