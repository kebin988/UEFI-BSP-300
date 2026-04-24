# 用 EDK2 编译你的第一个 UEFI 程序：HelloWorld

> 🔥 UEFI/BSP 开发系列第 010 篇 | 难度：⭐ 入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

前两篇我们在 Windows 和 Linux 上搭好了 EDK2 开发环境，编译出了 OVMF 固件。

但那是编译别人写好的代码。

今天，我们要写自己的代码——一个 **在 UEFI 环境中运行的 HelloWorld 程序**。

是的，这个程序不在 Windows 里运行，不在 Linux 里运行，它运行在**操作系统启动之前**，直接和固件对话。

如果你以前只写过跑在 OS 上面的程序，今天你将第一次触碰到"OS 之下"的世界。

很酷，对吧？

---

## 一、UEFI 程序和普通程序有什么不同？

先看一段普通 C 语言 HelloWorld：

```c
#include <stdio.h>

int main() {
    printf("Hello, World!\n");
    return 0;
}
```

这段代码依赖 `stdio.h`，依赖 C 标准库，依赖操作系统的系统调用。没有 OS，它跑不起来。

再看 UEFI 版本的 HelloWorld：

```c
#include <Uefi.h>
#include <Library/UefiApplicationEntryPoint.h>
#include <Library/UefiLib.h>

EFI_STATUS
EFIAPI
UefiMain (
  IN EFI_HANDLE        ImageHandle,
  IN EFI_SYSTEM_TABLE  *SystemTable
  )
{
  Print(L"Hello, UEFI World!\n");
  return EFI_SUCCESS;
}
```

几个关键区别：

| 维度 | 普通 C 程序 | UEFI 程序 |
|------|-----------|-----------|
| 入口函数 | `main()` | `UefiMain()` |
| 运行环境 | 操作系统上 | 固件环境（无 OS） |
| 依赖库 | C 标准库（libc） | UEFI 库（MdePkg） |
| 字符串类型 | `char*`（ASCII） | `CHAR16*`（Unicode，L"..."） |
| 返回值 | `int` | `EFI_STATUS` |
| 参数 | argc/argv | ImageHandle + SystemTable |

> 💡 UEFI 程序的入口函数接收两个参数：
> - `ImageHandle`：当前程序自己的"身份证"
> - `SystemTable`：系统服务表，通过它可以访问所有 UEFI 提供的功能（屏幕输出、内存分配、文件操作等）

`Print(L"...")` 本质上是通过 `SystemTable->ConOut->OutputString()` 在屏幕上输出文字。UEFI 用的是 Unicode（UTF-16），所以字符串前面有个 `L`。

---

## 二、创建 HelloWorld 程序

我们需要创建两个文件：
1. **源代码文件** `.c` — 程序逻辑
2. **模块描述文件** `.inf` — 告诉 EDK2 构建系统怎么编译这个模块

### 2.1 创建目录

在 EDK2 的某个 Package 下创建我们的程序目录。初学者直接放在 `MdeModulePkg` 下最简单：

```bash
mkdir -p ~/UEFI/edk2/MdeModulePkg/Application/HelloWorld
```

> Windows 下对应路径：`C:\UEFI\edk2\MdeModulePkg\Application\HelloWorld`

### 2.2 创建源代码：HelloWorld.c

```c
/** @file
  My first UEFI application - HelloWorld

  Copyright (c) 2024, My Name. All rights reserved.
  SPDX-License-Identifier: BSD-2-Clause-Patent
**/

#include <Uefi.h>
#include <Library/UefiApplicationEntryPoint.h>
#include <Library/UefiLib.h>
#include <Library/UefiBootServicesTableLib.h>

EFI_STATUS
EFIAPI
UefiMain (
  IN EFI_HANDLE        ImageHandle,
  IN EFI_SYSTEM_TABLE  *SystemTable
  )
{
  // 在屏幕上打印 Hello World
  Print(L"=================================\n");
  Print(L"  Hello, UEFI World!\n");
  Print(L"  This is my first UEFI app.\n");
  Print(L"=================================\n");
  Print(L"\nPress any key to exit...\n");

  // 等待用户按下任意键
  EFI_INPUT_KEY Key;
  gBS->WaitForEvent(1, &gST->ConIn->WaitForKey, NULL);
  gST->ConIn->ReadKeyStroke(gST->ConIn, &Key);

  return EFI_SUCCESS;
}
```

> 💡 **代码解读**：
> - `gST` = Global System Table，就是入口函数的 `SystemTable` 参数的全局版本
> - `gBS` = Global Boot Services，启动阶段的服务集合
> - `WaitForEvent` 让程序暂停，等待键盘输入事件
> - 这就像普通 C 程序里的 `getchar()` 或 `system("pause")`

### 2.3 创建模块描述文件：HelloWorld.inf

```ini
## @file
#  My first UEFI Application - HelloWorld
#
#  Copyright (c) 2024, My Name. All rights reserved.
#  SPDX-License-Identifier: BSD-2-Clause-Patent
##

[Defines]
  INF_VERSION                    = 0x00010005
  BASE_NAME                      = HelloWorld
  FILE_GUID                      = a912f198-7f0e-4803-b908-b757b806ec83
  MODULE_TYPE                    = UEFI_APPLICATION
  VERSION_STRING                 = 1.0
  ENTRY_POINT                    = UefiMain

[Sources]
  HelloWorld.c

[Packages]
  MdePkg/MdePkg.dec
  MdeModulePkg/MdeModulePkg.dec

[LibraryClasses]
  UefiApplicationEntryPoint
  UefiLib
  UefiBootServicesTableLib
```

### INF 文件各段含义

INF 文件是 EDK2 中每个模块的"身份证"——**每个 .c 文件想被编译，必须有一个对应的 .inf 文件**。

你可以把 INF 文件理解为 EDK2 版本的 `Makefile` 或 `CMakeLists.txt`，但它更偏向"声明式"——你只需要告诉构建系统"我叫什么、我依赖什么"，具体怎么编译由 EDK2 的 build 工具自动处理。

| 段名 | 作用 | 类比 |
|------|------|------|
| `[Defines]` | 模块的基本信息：名称、GUID、类型、入口函数 | CMake 的 `project()` + `add_executable()` |
| `[Sources]` | 源代码文件列表 | CMake 的源文件列表 |
| `[Packages]` | 依赖哪些包（.dec 文件）| CMake 的 `find_package()` |
| `[LibraryClasses]` | 依赖哪些库 | CMake 的 `target_link_libraries()` |

那 **DSC 文件**又是什么？DSC（Description file）是**平台级别的总管文件**——它定义了"整个固件由哪些模块组成"。一个 `.dsc` 对应一个平台（比如 OvmfPkgX64.dsc 对应 QEMU 虚拟机平台）。

简单说：**INF 描述一个模块，DSC 把一堆模块组装成一个固件**。后面第 062~064 篇会详细讲解这些文件。

> ⚠️ `FILE_GUID` 必须是唯一的。你可以用在线 GUID 生成器生成一个，也可以用 Python：
> ```python
> import uuid; print(str(uuid.uuid4()))
> ```

> ⚠️ `MODULE_TYPE = UEFI_APPLICATION` 表示这是一个 UEFI 应用程序（类似 EXE），不是驱动。

---

## 三、把程序加入编译

光创建了文件还不够，你需要告诉 EDK2 的构建系统："嘿，我有一个新模块，请编译它。"

### 方法：修改 .dsc 文件

编辑 OVMF 平台的 DSC 文件，把我们的模块添加进去：

```bash
vim ~/UEFI/edk2/OvmfPkg/OvmfPkgX64.dsc
```

找到 `[Components]` 段（文件末尾附近），添加一行：

```ini
[Components]
  # ... 已有的模块列表 ...
  MdeModulePkg/Application/HelloWorld/HelloWorld.inf    # <-- 添加这一行
```

保存退出。

---

## 四、编译

```bash
# Linux
cd ~/UEFI/edk2
source .venv/bin/activate
source edksetup.sh
build
```

```cmd
# Windows（VS 开发者命令行）
cd C:\UEFI\edk2
edksetup.bat
build
```

编译成功后，你的 HelloWorld 程序在这里：

```
# Linux
Build/OvmfX64/DEBUG_GCC5/X64/HelloWorld.efi

# Windows
Build\OvmfX64\DEBUG_VS2022\X64\HelloWorld.efi
```

这个 `.efi` 文件就是你的 UEFI 可执行程序！它的格式是 PE32+（和 Windows 的 EXE 格式很像，但是跑在固件环境里）。

---

## 五、在 QEMU 中运行

### 5.1 创建一个虚拟磁盘

我们需要把 HelloWorld.efi 放到一个虚拟磁盘里，这样 UEFI Shell 才能找到它：

```bash
# 创建一个 FAT 格式的虚拟磁盘镜像
dd if=/dev/zero of=disk.img bs=1M count=64
mkfs.fat disk.img

# 创建挂载点并挂载
mkdir -p /tmp/uefi_disk
sudo mount disk.img /tmp/uefi_disk

# 把 HelloWorld.efi 复制进去
sudo cp Build/OvmfX64/DEBUG_GCC5/X64/HelloWorld.efi /tmp/uefi_disk/

# 卸载
sudo umount /tmp/uefi_disk
```

### 5.2 用 QEMU 启动

```bash
qemu-system-x86_64 \
    -bios Build/OvmfX64/DEBUG_GCC5/FV/OVMF.fd \
    -drive file=disk.img,format=raw \
    -net none \
    -serial stdio
```

### 5.3 在 UEFI Shell 中运行

QEMU 启动后进入 UEFI Shell，执行：

```
Shell> fs0:
FS0:\> HelloWorld.efi
```

如果一切正常，你会看到：

```
=================================
  Hello, UEFI World!
  This is my first UEFI app.
=================================

Press any key to exit...
```

**🎉 恭喜！你成功运行了人生中第一个 UEFI 程序！**

这段代码在操作系统启动之前就运行了。此刻没有 Windows，没有 Linux，只有你的代码和 UEFI 固件。

---

## 六、深入理解：这个程序到底经历了什么？

从你输入 `HelloWorld.efi` 到看到输出，UEFI 做了这些事：

```
你输入 HelloWorld.efi 并回车
    ↓
UEFI Shell 在 FAT 文件系统中找到 HelloWorld.efi
    ↓
UEFI Image Loader 把 PE32+ 格式的 EFI 文件加载到内存
    ↓
检查文件头、分配内存、解析依赖
    ↓
跳转到入口函数 UefiMain()
    ↓
UefiMain 调用 Print() → 实际调用 SystemTable->ConOut->OutputString()
    ↓
GOP 驱动把字符渲染到屏幕上
    ↓
WaitForEvent 等待键盘中断
    ↓
你按下任意键 → 函数返回 EFI_SUCCESS
    ↓
UEFI Shell 回收内存，回到命令行
```

> 你以为只是打印了一行字，其实背后有 Protocol 调用、内存管理、事件机制、驱动配合……这就是 UEFI 的面向对象架构在工作。

---

## 七、总结

| 维度 | 内容 |
|------|------|
| **程序入口** | `UefiMain(ImageHandle, SystemTable)` |
| **输出函数** | `Print(L"...")` 或 `gST->ConOut->OutputString()` |
| **文件格式** | `.efi`（PE32+ 格式） |
| **模块描述** | `.inf` 文件定义模块信息和依赖 |
| **编译方式** | 修改 .dsc 添加模块 → `build` |
| **运行方式** | QEMU + OVMF + UEFI Shell |

---

下一篇：**#011 UEFI Shell 是什么？它能干什么？**——UEFI 固件里的"命令行工具"，比你想的强大得多。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
