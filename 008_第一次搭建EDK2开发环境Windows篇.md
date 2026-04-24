# 第一次搭建 EDK2 开发环境（Windows 篇）

> 🔥 UEFI/BSP 开发系列第 008 篇 | 难度：⭐ 入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们聊了 EDK2 是什么，这一篇我们动手。

学编程有句老话：**"配环境一天，写代码五分钟。"**

UEFI 开发也不例外。EDK2 的环境搭建虽然不算复杂，但坑多且隐蔽——版本不对、路径有中文、Python 版本不兼容……随便一个都能让你折腾一晚上。

这篇文章的目标很简单：**让你在 Windows 上从零搭建 EDK2 开发环境，并成功编译出第一个固件镜像。**

全程截图级教程，踩过的坑我都帮你标出来了。

---

## 一、你需要准备什么？

先别急着下载，看看你的电脑满不满足最低要求：

| 条件 | 最低要求 | 推荐配置 |
|------|---------|---------|
| 操作系统 | Windows 10 64 位 | Windows 11 |
| 磁盘空间 | 10GB 以上 | SSD，20GB+ |
| 内存 | 4GB | 8GB+ |
| 网络 | 能访问 GitHub | 如果慢就用镜像/代理 |

然后，我们需要安装以下工具：

```
1. Visual Studio 2019 或更高版本（提供 C 编译器）
2. Python 3.8+（EDK2 构建脚本依赖 Python）
3. NASM（汇编器，某些模块需要）
4. ASL 编译器（iasl，编译 ACPI 表用）
5. Git（拉取 EDK2 源码）
```

> ⚠️ **踩坑警告**：不要用 MinGW 或 Cygwin 的 gcc 来编译 EDK2，官方只推荐 VS2019/VS2022 的 MSVC 工具链。用 gcc 走 Windows 编译会掉进无穷无尽的坑。

---

## 二、Step 1：安装 Visual Studio

EDK2 在 Windows 上编译需要 MSVC 编译器。你不需要安装完整的 Visual Studio IDE（那玩意几十个 GB），只需要 **Build Tools**。

### 下载地址

前往 [Visual Studio 下载页面](https://visualstudio.microsoft.com/downloads/)，选择 **"Build Tools for Visual Studio 2022"**（免费）。

### 安装时选择这些组件

在安装界面勾选 **"使用 C++ 的桌面开发"** 工作负载，确保以下组件被包含：

- MSVC v143 - VS 2022 C++ x64/x86 生成工具
- Windows 11 SDK（或 Windows 10 SDK）
- C++ CMake tools for Windows（可选，但推荐）

```
                    ┌─────────────────────────┐
                    │ Visual Studio Installer  │
                    │                         │
                    │ ☑ 使用 C++ 的桌面开发     │
                    │   ☑ MSVC v143           │
                    │   ☑ Windows SDK         │
                    │   ☐ 其他可选组件...      │
                    └─────────────────────────┘
```

> 💡 **Tips**：安装路径最好不要有中文和空格。`C:\BuildTools` 或 `C:\VS2022` 是个好选择。

---

## 三、Step 2：安装 Python

EDK2 的构建系统（BaseTools）大量使用 Python 脚本。

### 下载地址

前往 [Python 官网](https://www.python.org/downloads/) 下载 **Python 3.10 或 3.11**。

> ⚠️ **踩坑警告**：
> - Python 3.12+ 可能有兼容性问题（某些第三方库还没跟上），建议用 3.10 或 3.11
> - 安装时务必勾选 **"Add Python to PATH"**（底部那个小复选框！）
> - 不要安装到带中文的路径

### 验证安装

打开命令行（cmd 或 PowerShell）：

```powershell
python --version
# 应该输出 Python 3.10.x 或 3.11.x

pip --version
# 确认 pip 可用
```

---

## 四、Step 3：安装 NASM

NASM（Netwide Assembler）是 EDK2 中某些底层模块（比如 SEC 阶段的汇编代码）需要的汇编器。

### 下载地址

前往 [NASM 官网](https://www.nasm.us/) 下载最新的 Windows 安装包。

### 安装和配置

1. 安装到 `C:\nasm`（路径简单，后面配置方便）
2. 把 `C:\nasm` 添加到系统 **PATH** 环境变量

### 验证安装

```powershell
nasm -v
# 应该输出类似 NASM version 2.16.xx
```

> 💡 **不想手动改 PATH？** 也可以等到后面配置 EDK2 时通过 `NASM_PREFIX` 环境变量指定路径。

---

## 五、Step 4：安装 ASL 编译器（iasl）

ACPI 相关的模块需要 ASL 编译器。EDK2 构建过程中会自动调用 `iasl`。

### 下载地址

前往 [ACPICA 下载页面](https://acpica.org/downloads) 下载 **Windows Binary Tools**。

### 安装方法

1. 解压到 `C:\ASL`
2. 把 `C:\ASL` 添加到系统 PATH

### 验证安装

```powershell
iasl -v
# 应该输出 Intel ACPI Component Architecture ASL+ Optimizing Compiler/Disassembler version xxxxxxxx
```

---

## 六、Step 5：安装 Git 并克隆 EDK2

### 安装 Git

如果你还没装 Git：前往 [Git 官网](https://git-scm.com/) 下载 Windows 安装包，一路 Next 即可。

### 克隆 EDK2 仓库

找一个没有中文和空格的目录（比如 `C:\UEFI`），打开 Git Bash 或 PowerShell：

```powershell
cd C:\
mkdir UEFI
cd UEFI

# 克隆 EDK2 主仓库
git clone https://github.com/tianocore/edk2.git

# 进入目录
cd edk2

# 初始化子模块（很重要！EDK2 有一些依赖是 git submodule）
git submodule update --init --recursive
```

> ⚠️ **踩坑警告**：
> - `git submodule update --init --recursive` 这一步**不能跳过**！否则后面编译会报一堆找不到文件的错
> - 如果 GitHub 下载很慢，可以用 `https://ghproxy.com/` 等镜像加速，或者配置 Git 代理
> - 完整克隆（含子模块）大约需要 2~3GB 磁盘空间

克隆完成后，目录结构应该长这样：

```
C:\UEFI\edk2\
├── BaseTools\
├── MdePkg\
├── MdeModulePkg\
├── OvmfPkg\
├── ShellPkg\
├── SecurityPkg\
├── ...
├── edksetup.bat      <-- 关键脚本！
└── Conf\
```

---

## 七、Step 6：配置 EDK2 构建环境

这是最关键的一步。

### 6.1 打开 VS 命令行环境

不要直接用普通的 cmd，要用 Visual Studio 提供的开发者命令行工具。

在开始菜单搜索：**"x64 Native Tools Command Prompt for VS 2022"**，打开它。

> 这个命令行自动配置了 MSVC 编译器的环境变量，直接在普通 cmd 里编译会找不到 `cl.exe`。

### 6.2 运行 edksetup.bat

```cmd
cd C:\UEFI\edk2
edksetup.bat
```

这个脚本会：
1. 设置 EDK2 相关的环境变量（`WORKSPACE`、`EDK_TOOLS_PATH` 等）
2. 编译 BaseTools 的 C 工具（如果是第一次运行）

### 6.3 编译 BaseTools

首次使用需要编译 EDK2 的构建工具：

```cmd
edksetup.bat Rebuild
```

如果一切顺利，你会看到类似这样的输出：

```
Building ... C Tools ...
Finished building BaseTools C Tools.
```

> ⚠️ 如果报错 `nmake not found`，说明你没用 VS 开发者命令行。回到 6.1 重新来。

### 6.4 配置 target.txt

编辑 `C:\UEFI\edk2\Conf\target.txt`（第一次运行 edksetup.bat 后会自动生成）：

```ini
ACTIVE_PLATFORM       = OvmfPkg/OvmfPkgX64.dsc
TARGET                = DEBUG
TARGET_ARCH           = X64
TOOL_CHAIN_TAG        = VS2022

# 如果你用的是 VS2019，改为 VS2019
```

> 💡 **关键字段说明**：
> - `ACTIVE_PLATFORM`：指定编译哪个平台，OVMF 是 QEMU 虚拟机的固件
> - `TARGET`：DEBUG 版本会输出调试日志，RELEASE 版本会做优化
> - `TARGET_ARCH`：X64 表示编译 64 位固件
> - `TOOL_CHAIN_TAG`：编译器版本，和你安装的 VS 对应

---

## 八、Step 7：编译！

万事俱备，一行命令开始编译：

```cmd
build
```

第一次编译 OVMF 大约需要 5~15 分钟（取决于你的 CPU 和硬盘速度）。

如果看到类似这样的输出，恭喜你，编译成功了：

```
- Done -
Build end time: ...
Build total time: 00:08:23

Build successfully!
```

编译产物在这里：

```
C:\UEFI\edk2\Build\OvmfX64\DEBUG_VS2022\FV\OVMF.fd
```

这个 `OVMF.fd` 就是你编译出来的 UEFI 固件镜像！虽然它是虚拟机用的，但它是一个货真价实的 UEFI 固件。

---

## 九、常见错误排查

| 错误信息 | 原因 | 解决方法 |
|---------|------|---------|
| `'nmake' is not recognized` | 没有用 VS 开发者命令行 | 用 "x64 Native Tools Command Prompt" |
| `'python' is not recognized` | Python 没加入 PATH | 重装 Python，勾选 "Add to PATH" |
| `'nasm' is not recognized` | NASM 没安装或没加 PATH | 安装 NASM 并加到 PATH |
| `No module named 'xxx'` | Python 缺少依赖包 | `pip install xxx` |
| `submodule xxx not found` | 没有初始化子模块 | `git submodule update --init --recursive` |
| 路径中有中文导致乱码 | EDK2 工具链不支持中文路径 | 把 edk2 放到纯英文路径下 |
| `Build Failed` + `IA32` 相关错误 | target.txt 架构配置不对 | 确认 `TARGET_ARCH = X64` |

---

## 十、搭建过程一览

怕你看晕了，这里整理一个 checklist：

```
☐ 1. 安装 Visual Studio Build Tools 2022
       勾选 "使用 C++ 的桌面开发"
☐ 2. 安装 Python 3.10/3.11
       勾选 "Add to PATH"
☐ 3. 安装 NASM
       添加到 PATH
☐ 4. 安装 iasl（ASL 编译器）
       添加到 PATH
☐ 5. 安装 Git
☐ 6. git clone edk2 + submodule init
☐ 7. 用 VS 开发者命令行运行 edksetup.bat
☐ 8. 编辑 Conf/target.txt
☐ 9. 运行 build，得到 OVMF.fd
```

全部打勾，你就拥有了一个可以编译 UEFI 固件的开发环境。

---

## 十一、总结

| 维度 | 内容 |
|------|------|
| **目标** | 在 Windows 上搭建 EDK2 开发环境 |
| **必装工具** | VS Build Tools + Python + NASM + iasl + Git |
| **关键步骤** | edksetup.bat → 编辑 target.txt → build |
| **编译产物** | Build\OvmfX64\DEBUG_VS2022\FV\OVMF.fd |
| **最大的坑** | 中文路径、忘记子模块初始化、用错命令行 |

环境搭好了，接下来我们在 Linux 上也搭一遍——因为真正的 BSP 开发大部分时间都在 Linux 下。

---

下一篇：**#009 第一次搭建 EDK2 开发环境（Linux 篇）**——Linux 的环境搭建反而比 Windows 简单。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
