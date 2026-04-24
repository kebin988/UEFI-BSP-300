# 第一次搭建 EDK2 开发环境（Linux 篇）

> 🔥 UEFI/BSP 开发系列第 009 篇 | 难度：⭐ 入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们在 Windows 上搭建了 EDK2 开发环境，过程还挺"丰富"的（各种安装配置加踩坑）。

这一篇换到 Linux。

说实话，**Linux 下搭建 EDK2 反而更简单**。因为 Linux 天生就是开发者的主场——GCC 自带、Python 自带、包管理器一条命令搞定依赖。

更重要的是，在真实的 BIOS/BSP 开发工作中，**大部分团队的编译服务器都跑 Linux**。Windows 是你的桌面办公环境，Linux 才是代码编译的主战场。

所以这篇文章的含金量其实比上一篇还高。

---

## 一、选哪个 Linux 发行版？

EDK2 官方测试最多的发行版是 **Ubuntu**，社区文档和 CI 也以 Ubuntu 为主。

推荐：

| 发行版 | 推荐度 | 说明 |
|--------|-------|------|
| **Ubuntu 22.04 LTS** | ⭐⭐⭐⭐⭐ | 首选，社区支持最好 |
| Ubuntu 24.04 LTS | ⭐⭐⭐⭐ | 可用，部分工具版本更新 |
| Fedora / CentOS | ⭐⭐⭐ | 能用，包名可能不同 |
| Arch Linux | ⭐⭐ | 滚动更新，版本可能太新导致兼容问题 |

> 💡 如果你用 Windows 11，可以直接用 **WSL2（Ubuntu）**，体验和原生 Linux 几乎一样。

---

## 二、一键安装所有依赖

打开终端，复制粘贴这一段：

```bash
# Ubuntu / Debian 系
sudo apt update
sudo apt install -y \
    build-essential \
    git \
    python3 \
    python3-pip \
    python3-venv \
    uuid-dev \
    nasm \
    iasl \
    gcc-multilib
```

就这么简单。Windows 上需要手动下载安装 5 个工具，Linux 一条命令全搞定。

> 包管理器就是 Linux 的超能力 🦸

### 各包的作用

| 包名 | 作用 |
|------|------|
| `build-essential` | GCC、make 等编译工具集 |
| `git` | 版本控制 |
| `python3` / `python3-pip` | EDK2 构建脚本运行环境 |
| `python3-venv` | Python 虚拟环境支持 |
| `uuid-dev` | UUID 库，BaseTools 编译需要 |
| `nasm` | 汇编器 |
| `iasl` | ASL 编译器（ACPI 表） |
| `gcc-multilib` | 32 位和 64 位交叉编译支持 |

### 验证安装

```bash
gcc --version       # GCC 11 或 12
python3 --version   # Python 3.10+
nasm -v             # NASM 2.15+
iasl -v             # iASL 20xx
git --version       # Git 2.x
```

---

## 三、克隆 EDK2 源码

```bash
# 创建工作目录
mkdir -p ~/UEFI && cd ~/UEFI

# 克隆 EDK2
git clone https://github.com/tianocore/edk2.git

# 进入目录
cd edk2

# 初始化子模块（必须！）
git submodule update --init --recursive
```

> ⚠️ 和 Windows 篇一样，`git submodule update --init --recursive` **不能跳过**。少了子模块，编译必定失败。
>
> 如果 GitHub 访问慢，可以配置 Git 代理或者使用镜像。

---

## 四、创建 Python 虚拟环境

EDK2 的构建系统依赖 Python，为了不污染系统 Python 环境，我们用虚拟环境：

```bash
# 在 edk2 目录下创建虚拟环境
python3 -m venv .venv

# 激活虚拟环境
source .venv/bin/activate

# 安装 EDK2 需要的 Python 依赖
pip install -i https://mirrors.aliyun.com/pypi/simple/ -r pip-requirements.txt
```

> 💡 激活虚拟环境后，你的终端提示符前面会出现 `(.venv)`，表示当前在虚拟环境中。

---

## 五、配置构建环境

### 5.1 运行 edksetup.sh

```bash
# 确保你在 edk2 目录下
cd ~/UEFI/edk2

# 加载环境变量
source edksetup.sh
```

这个脚本会设置 `WORKSPACE`、`EDK_TOOLS_PATH` 等环境变量。

> 注意是 `source edksetup.sh` 不是 `./edksetup.sh`。用 source 才能让环境变量生效在当前 Shell。

### 5.2 编译 BaseTools

```bash
make -C BaseTools
```

这一步编译 EDK2 自带的 C 语言构建工具。如果成功，你会看到：

```
Finished building BaseTools C Tools ...
```

### 5.3 配置 target.txt

编辑 `Conf/target.txt`：

```bash
vim Conf/target.txt
# 或者用你喜欢的编辑器
```

修改以下字段：

```ini
ACTIVE_PLATFORM       = OvmfPkg/OvmfPkgX64.dsc
TARGET                = DEBUG
TARGET_ARCH           = X64
TOOL_CHAIN_TAG        = GCC5
```

> ⚠️ **注意**：Linux 下工具链是 `GCC5`（这是 EDK2 的工具链标签名，不是说你必须用 GCC 5 版本，GCC 11/12/13 都用这个标签）。
>
> 这是一个经典的命名误导：`GCC5` 表示"GCC 5 及以上版本的编译参数集合"，你的 GCC 12 完全没问题。

---

## 六、编译！

```bash
build
```

Linux 下 EDK2 的编译速度通常比 Windows 快（GCC 编译速度优于 MSVC，加上 Linux 文件系统IO 更快）。

编译成功的输出：

```
- Done -
Build end time: ...
Build total time: 00:05:17

Build successfully!
```

编译产物路径：

```
~/UEFI/edk2/Build/OvmfX64/DEBUG_GCC5/FV/OVMF.fd
```

### 编译出来的 OVMF.fd 到底是个啥？

这个 `OVMF.fd` 就是一个 **完整的 UEFI 固件镜像**（FD = Flash Device image）。

打个比方：你买了一块全新的主板，主板上那块 SPI Flash 芯片里烧录的东西，本质上就是这么一个 `.fd` 文件。

具体来说，OVMF.fd 里面打包了：

| 组件 | 作用 |
|------|------|
| **SEC 模块** | CPU 上电后执行的第一段代码（安全验证 + 初始化 CAR） |
| **PEI 模块们** | 内存初始化前的早期模块（在虚拟机里没有真正的内存训练） |
| **DXE 驱动们** | 各种设备驱动：PCI 枚举、USB 栈、网络栈、文件系统驱动… |
| **BDS 模块** | 启动管理器，决定从哪个设备启动操作系统 |
| **UEFI Shell** | 固件层的命令行工具（所以你 QEMU 跑起来看到的就是它） |

OVMF 是 **Open Virtual Machine Firmware** 的缩写，专门为虚拟机（QEMU/KVM）设计的 UEFI 固件。它不能直接烧进真实主板——真实主板需要芯片原厂的硅初始化代码（比如 Intel FSP），这些代码不在 EDK2 开源仓库里。

但从学习角度来说，OVMF 和真实主板固件的**架构完全一样**（SEC → PEI → DXE → BDS），只是跳过了硬件初始化的细节。所以它是入门学习的最佳起点。

```
真实主板固件 BIOS.fd        OVMF.fd（虚拟机固件）
┌──────────────────┐      ┌──────────────────┐
│ SEC              │      │ SEC              │
│ PEI + Intel FSP  │      │ PEI（简化版）     │
│ DXE 驱动（真硬件）│      │ DXE 驱动（虚拟硬件）│
│ BDS              │      │ BDS              │
│ UEFI Shell       │      │ UEFI Shell       │
└──────────────────┘      └──────────────────┘
    ↓ 烧进 SPI Flash           ↓ 给 QEMU 加载
    真实电脑启动                虚拟机启动
```

---

## 七、（可选）用 QEMU 跑起来看看

如果你想立刻看到效果，可以安装 QEMU 并运行编译出来的 UEFI 固件：

```bash
# 安装 QEMU
sudo apt install -y qemu-system-x86

# 运行 OVMF 固件
qemu-system-x86_64 \
    -bios Build/OvmfX64/DEBUG_GCC5/FV/OVMF.fd \
    -net none \
    -serial stdio
```

如果一切正常，你会看到一个 QEMU 窗口弹出来，显示 UEFI Shell 界面：

```
UEFI Interactive Shell v2.2
EDK II
UEFI v2.70 (EDK II, 0x00010000)
Mapping table
    BLK0: Alias(s):
Shell>
```

**恭喜！** 你成功在 Linux 上编译并运行了一个 UEFI 固件。

> 🎮 在 UEFI Shell 里你可以输入 `ver` 查看版本，输入 `help` 查看可用命令。这就是操作系统启动前的"命令行"。

---

## 八、完整流程脚本

把上面所有步骤浓缩成一个可以直接跑的脚本：

```bash
#!/bin/bash
# EDK2 环境搭建一键脚本（Ubuntu 22.04）

set -e

echo "=== 安装依赖 ==="
sudo apt update
sudo apt install -y build-essential git python3 python3-pip python3-venv \
    uuid-dev nasm iasl gcc-multilib qemu-system-x86

echo "=== 克隆 EDK2 ==="
mkdir -p ~/UEFI && cd ~/UEFI
git clone https://github.com/tianocore/edk2.git
cd edk2
git submodule update --init --recursive

echo "=== Python 虚拟环境 ==="
python3 -m venv .venv
source .venv/bin/activate
pip install -i https://mirrors.aliyun.com/pypi/simple/ -r pip-requirements.txt

echo "=== 配置 EDK2 ==="
source edksetup.sh
make -C BaseTools

echo "=== 配置 target.txt ==="
sed -i 's|^ACTIVE_PLATFORM.*|ACTIVE_PLATFORM = OvmfPkg/OvmfPkgX64.dsc|' Conf/target.txt
sed -i 's|^TARGET_ARCH.*|TARGET_ARCH = X64|' Conf/target.txt
sed -i 's|^TOOL_CHAIN_TAG.*|TOOL_CHAIN_TAG = GCC5|' Conf/target.txt

echo "=== 开始编译 ==="
build

echo "=== 编译完成！==="
echo "固件镜像: $(pwd)/Build/OvmfX64/DEBUG_GCC5/FV/OVMF.fd"
```

保存为 `setup_edk2.sh`，一条命令从零到编译完成。

---

## 九、Windows vs Linux 环境对比

| 维度 | Windows | Linux |
|------|---------|-------|
| 编译器 | MSVC (VS2022) | GCC |
| 工具链标签 | VS2022 | GCC5 |
| 安装依赖 | 手动下载 5 个安装包 | `apt install` 一行搞定 |
| 环境初始化 | edksetup.bat | source edksetup.sh |
| 编译速度 | 较慢 | 较快 |
| 实际工作使用 | 日常桌面办公 | 编译服务器主力 |
| 路径问题 | 中文/空格容易出事 | 通常没有 |

> 💡 **实际工作中的常见组合**：Windows 上写代码（VS Code / Source Insight），Linux 服务器上编译（SSH 远程连接）。

---

## 十、常见错误排查

| 错误信息 | 原因 | 解决方法 |
|---------|------|---------|
| `/bin/sh: nasm: not found` | 没装 NASM | `sudo apt install nasm` |
| `iasl: command not found` | 没装 ASL 编译器 | `sudo apt install iasl` |
| `No module named 'xxx'` | Python 依赖缺失 | 在虚拟环境中 `pip install xxx` |
| `make: *** BaseTools Error` | 缺少 uuid-dev | `sudo apt install uuid-dev` |
| `undefined reference to __udivdi3` | 缺少 gcc-multilib | `sudo apt install gcc-multilib` |
| `permission denied: edksetup.sh` | 权限问题 | 用 `source edksetup.sh` 而不是 `./` |

---

## 十一、总结

| 维度 | 内容 |
|------|------|
| **目标** | 在 Linux（Ubuntu）上搭建 EDK2 开发环境 |
| **依赖安装** | `apt install` 一行命令 |
| **关键步骤** | source edksetup.sh → make BaseTools → 编辑 target.txt → build |
| **工具链标签** | GCC5（适用于 GCC 5 及以上所有版本） |
| **编译产物** | Build/OvmfX64/DEBUG_GCC5/FV/OVMF.fd |
| **额外福利** | 装个 QEMU 就能直接跑固件 |

环境搭好了（Windows 和 Linux 双平台），下一篇我们终于要写代码了——编译运行你人生中第一个 UEFI 程序：HelloWorld。

---

下一篇：**#010 用 EDK2 编译你的第一个 UEFI 程序：HelloWorld**——从零写一个能在固件环境运行的程序。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
