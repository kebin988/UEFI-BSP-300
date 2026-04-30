# 什么是 UEFI 变量？它和注册表有什么像？

> 🔥 UEFI/BSP 开发系列第 018 篇 | 难度：⭐ 入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们搞明白了 UEFI 的内存模型——从 SEC 的"赤贫期"到 DXE 的"财务自由"。

但你有没有想过一个问题：

> "BIOS 设置界面里那些选项——启动顺序、安全启动开关、风扇策略——这些配置存在哪里？断电后为什么不会丢？"

在 Windows 的世界里，应用程序的配置存在**注册表**（Registry）里。在 Linux 的世界里，配置存在 `/etc` 下面的各种文件里。

那在固件的世界里呢？答案是——**UEFI 变量（UEFI Variables）**。

今天就把这个"固件的注册表"讲清楚。

---

## 一、UEFI 变量是什么？一句话版本

**UEFI 变量 = 一个存储在 SPI Flash（非易失性存储）上的键值对数据库。**

```
Windows 注册表：  HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run → "SomeApp.exe"
UEFI 变量：       {GUID}"BootOrder" → { 0x0001, 0x0003, 0x0002 }
```

本质上就是一个 **Key-Value 存储**：

- **Key** = GUID + 变量名（Unicode 字符串）
- **Value** = 一坨二进制数据（你想存什么都行）

> 💡 跟 Windows 注册表最大的相似点：都是操作系统/固件用来存"配置"的地方，都能在重启后保持不变。

---

## 二、为什么需要 UEFI 变量？

想象一下没有 UEFI 变量的世界：

```
用户：我要改启动顺序，先从 U 盘启动
BIOS：好的，改了
        —— 关机 ——
        —— 开机 ——
BIOS：启动顺序？什么启动顺序？我什么都不记得啊！
用户：？？？
```

固件每次启动都是"失忆"状态——因为 DDR 内存断电就清空了。要想让固件"记住"一些东西，就必须有一个**断电不丢失**的存储机制。

这就是 UEFI 变量的核心使命：**让固件拥有"记忆力"**。

具体存了些什么？举几个最常见的：

| 变量名 | 用途 | 你在哪见过 |
|--------|------|-----------|
| `BootOrder` | 启动设备的优先顺序 | BIOS 设置界面的 Boot 选项 |
| `Boot0001` | 第 1 个启动项的详细信息 | 对应某个硬盘/U盘/网络启动 |
| `SecureBoot` | 安全启动是否开启 | BIOS 安全设置里的开关 |
| `PK` | Platform Key（平台密钥） | Secure Boot 的信任根 |
| `Timeout` | 启动菜单等待时间（秒） | 开机那个倒计时 |
| `ConOut` | 控制台输出设备路径 | 固件选择在哪里显示输出 |
| `Lang` | 当前语言设置 | BIOS 界面语言选择 |

> 打个比方：UEFI 变量就像固件的"便签纸"——贴在 SPI Flash 上，关机也不会掉。每次开机，固件第一件事就是去读这些便签，恢复之前的设置。

---

## 三、UEFI 变量的命名规则

一个 UEFI 变量由两部分唯一标识：

```
变量标识 = GUID + 变量名（Unicode 字符串）
```

为什么需要 GUID？因为不同的固件模块可能定义同名变量。GUID 就像"命名空间"，防止撞车：

```
Intel 的模块定义了 "Config" 变量 → GUID = {11111111-...}
AMD 的模块也定义了 "Config" 变量 → GUID = {22222222-...}
```

两个 "Config" 互不干扰，因为 GUID 不同。

UEFI 规范定义了一个全局变量的 GUID：

```c
#define EFI_GLOBAL_VARIABLE_GUID \
  { 0x8BE4DF61, 0x93CA, 0x11D2, \
    { 0xAA, 0x0D, 0x00, 0xE0, 0x98, 0x03, 0x2B, 0x8C } }
```

像 `BootOrder`、`SecureBoot`、`Timeout` 这些"官方"变量，都挂在这个 GUID 下面。

---

## 四、UEFI 变量的属性（Attributes）

每个 UEFI 变量还有一组**属性标志**，决定了它的"脾气"：

```c
// 常用属性组合
#define EFI_VARIABLE_NON_VOLATILE                          0x00000001
#define EFI_VARIABLE_BOOTSERVICE_ACCESS                    0x00000002
#define EFI_VARIABLE_RUNTIME_ACCESS                        0x00000004
#define EFI_VARIABLE_HARDWARE_ERROR_RECORD                 0x00000008
#define EFI_VARIABLE_AUTHENTICATED_WRITE_ACCESS             0x00000010  // 已废弃
#define EFI_VARIABLE_TIME_BASED_AUTHENTICATED_WRITE_ACCESS  0x00000020
#define EFI_VARIABLE_APPEND_WRITE                          0x00000040
```

### 最重要的三个属性

| 属性 | 含义 | 比喻 |
|------|------|------|
| `NON_VOLATILE` (NV) | 断电不丢失，存在 Flash 上 | 写在石碑上 |
| `BOOTSERVICE_ACCESS` (BS) | Boot Services 阶段可访问 | 办公时间可进 |
| `RUNTIME_ACCESS` (RT) | Runtime（OS 运行时）可访问 | 7×24 小时可进 |

### 常见的属性组合

```
NV + BS + RT  →  最常见！断电不丢 + 固件阶段能读 + OS 也能读写
                 例如：BootOrder、SecureBoot

BS + RT       →  易失性变量，重启就没了，但固件和 OS 都能用
                 例如：某些临时状态变量

NV + BS       →  断电不丢，但 OS 启动后就不能访问了
                 例如：某些仅供固件使用的内部配置
```

> 💡 **重点理解**：`RUNTIME_ACCESS` 决定了操作系统能不能碰这个变量。如果没有这个属性，OS 调用 `GetVariable()` 会被拒之门外。

---

## 五、读写 UEFI 变量的 API

UEFI 提供了两个核心函数来读写变量，它们属于 **Runtime Services**——也就是说，**操作系统启动后依然可以调用**：

### 读取变量：GetVariable

```c
EFI_STATUS
gRT->GetVariable(
    L"BootOrder",                    // 变量名（Unicode）
    &gEfiGlobalVariableGuid,         // GUID
    &Attributes,                     // 输出：属性
    &DataSize,                       // 输入输出：数据大小
    Data                             // 输出：数据缓冲区
);
```

### 写入变量：SetVariable

```c
UINT16 NewBootOrder[] = { 0x0003, 0x0001, 0x0002 };

EFI_STATUS Status = gRT->SetVariable(
    L"BootOrder",                    // 变量名
    &gEfiGlobalVariableGuid,         // GUID
    EFI_VARIABLE_NON_VOLATILE |      // 属性：断电不丢
    EFI_VARIABLE_BOOTSERVICE_ACCESS |
    EFI_VARIABLE_RUNTIME_ACCESS,
    sizeof(NewBootOrder),            // 数据大小
    NewBootOrder                     // 数据
);
```

### 枚举所有变量：GetNextVariableName

```c
CHAR16    VariableName[256];
EFI_GUID  VendorGuid;
UINTN     NameSize;

// 初始化：传入空字符串开始枚举
VariableName[0] = L'\0';

while (TRUE) {
    NameSize = sizeof(VariableName);
    Status = gRT->GetNextVariableName(&NameSize, VariableName, &VendorGuid);
    if (EFI_ERROR(Status)) break;
    
    Print(L"变量: %g : %s\n", &VendorGuid, VariableName);
}
```

> 💡 `GetNextVariableName` 就是 UEFI 变量数据库的"遍历迭代器"。用法有点像 `readdir()`——每次调用返回下一个变量的名字和 GUID。

---

## 六、UEFI 变量存在哪里？物理层面

逻辑上，UEFI 变量是个 Key-Value 数据库。但物理上，它们存在 **SPI NOR Flash** 芯片上：

```
主板上的 SPI Flash 芯片（通常 8~32 MB）

┌────────────────────────────────────┐
│  Descriptor Region                 │  ← Intel ME 描述符
├────────────────────────────────────┤
│  ME Region                         │  ← Intel ME 固件
├────────────────────────────────────┤
│  BIOS Region                       │
│  ┌──────────────────────────────┐  │
│  │  FV_MAIN (DXE 驱动)          │  │
│  ├──────────────────────────────┤  │
│  │  FV_PEI (PEI 模块)           │  │
│  ├──────────────────────────────┤  │
│  │  NVRAM / NV Variable Store   │  │  ← UEFI 变量就在这里！
│  │  ┌──────────────────────┐    │  │
│  │  │ Var: BootOrder       │    │  │
│  │  │ Var: SecureBoot      │    │  │
│  │  │ Var: Boot0001        │    │  │
│  │  │ Var: Setup           │    │  │
│  │  │ ...                  │    │  │
│  │  │ FTW (Fault Tolerant  │    │  │
│  │  │      Write Backup)   │    │  │
│  │  └──────────────────────┘    │  │
│  └──────────────────────────────┘  │
├────────────────────────────────────┤
│  GbE Region                        │  ← 网卡固件（可选）
└────────────────────────────────────┘
```

### SPI Flash 的特殊性质

SPI Flash 不是普通的硬盘，它有一些"脾气"：

1. **读取**：随意，想读哪就读哪
2. **写入**：只能把 1 变成 0，不能把 0 变成 1
3. **擦除**：必须整块擦除（通常 4KB 一块），擦完全是 0xFF（全 1）
4. **寿命**：擦写次数有限（通常 10 万次）

这意味着变量的存储设计要特别小心：

```
变量更新流程（简化）：
1. 新变量写到当前 Block 的空闲区域（追加写入）
2. 老变量标记为"已删除"（改一个标志位，1→0）
3. 当 Block 快满了 → 触发"垃圾回收"
4. 把所有有效变量搬到新 Block
5. 擦除旧 Block
```

> 💡 这种"追加写入 + 垃圾回收"的设计，跟 SSD 的 FTL（闪存转换层）思路异曲同工。都是为了适应 Flash 的物理特性。

### FTW（Fault Tolerant Write）

如果变量写到一半掉电了怎么办？数据岂不是损坏了？

UEFI 设计了 **FTW（容错写入）** 机制：

```
正常写入流程：
1. 先把要修改的数据备份到 FTW 工作区
2. 标记"正在写入"
3. 执行真正的擦除和写入
4. 写完后标记"完成"

如果中途掉电：
→ 下次开机检测到"正在写入"标记
→ 从 FTW 备份区恢复数据
→ 变量区回到掉电前的一致状态
```

> 打个比方：就像你用 Word 文档时的"自动恢复"功能——每次保存前先存个备份，万一电脑崩了也不至于丢文件。

---

## 七、操作系统怎么读写 UEFI 变量？

UEFI 变量不只是固件自己用——操作系统也能读写它们。

### Linux 下

```bash
# 列出所有 UEFI 变量
ls /sys/firmware/efi/efivars/

# 查看 BootOrder
hexdump -C /sys/firmware/efi/efivars/BootOrder-8be4df61-93ca-11d2-aa0d-00e098032b8c

# 更友好的工具
efivar -l          # 列出所有变量
efivar -n 8be4df61-93ca-11d2-aa0d-00e098032b8c-BootOrder -p  # 打印变量

# 管理启动项
efibootmgr -v      # 查看启动项（最常用！）
efibootmgr -o 0003,0001,0002   # 修改启动顺序
```

### Windows 下

```powershell
# Windows 原生没有直接读取 UEFI 变量的 PowerShell 命令
# 需要通过 Win32 API 或第三方工具

# 方式 1：用 bcdedit 查看固件启动项（间接读取 Boot 相关变量）
bcdedit /enum firmware     # 查看固件层面的启动项

# 方式 2：通过 P/Invoke 调用 Win32 API（高级用法）
# GetFirmwareEnvironmentVariable / SetFirmwareEnvironmentVariable
# 示例：
# Add-Type @"
#   using System;
#   using System.Runtime.InteropServices;
#   public class FirmwareEnv {
#     [DllImport("kernel32.dll", SetLastError=true)]
#     public static extern uint GetFirmwareEnvironmentVariable(
#       string lpName, string lpGuid, IntPtr pBuffer, uint nSize);
#   }
# "@
# $buf = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(1024)
# [FirmwareEnv]::GetFirmwareEnvironmentVariable(
#   "SecureBoot", "{8be4df61-93ca-11d2-aa0d-00e098032b8c}", $buf, 1024)

# 方式 3：第三方工具
# UEFITool — 查看/编辑 BIOS 镜像
# EasyUEFI — 图形化管理启动项
# UEFI-variable-tool — 命令行读写 UEFI 变量
```

> ⚠️ **注意区分**：硬盘上的 EFI System Partition（ESP，通常是一个 FAT32 小分区）存放的是 OS Bootloader（如 `bootmgfw.efi`），那不是 UEFI 变量。UEFI 变量存在主板上的 **SPI Flash 芯片**里，跟硬盘没有关系。

### 底层原理

操作系统读写 UEFI 变量时，实际上是调用 **Runtime Services** 中的 `GetVariable()` / `SetVariable()`：

```
Linux 内核：
  用户空间 → /sys/firmware/efi/efivars/ → efi_get_variable()
  → efi.runtime->get_variable() → UEFI Runtime Services
  → 最终读写 SPI Flash 上的变量区

整个调用链：
  用户 App → 内核 → UEFI Runtime → SPI Flash 驱动 → SPI Flash 芯片
```

> 💡 这也是为什么 Runtime Services 内存在 `ExitBootServices` 之后不能被回收——因为操作系统随时可能调用它来读写 UEFI 变量。

---

## 八、UEFI 变量 vs Windows 注册表

既然开头说它们很像，我们来做个详细对比：

| 对比维度 | UEFI 变量 | Windows 注册表 |
|---------|----------|---------------|
| 本质 | 键值对数据库 | 键值对数据库 |
| 存储位置 | SPI Flash | 硬盘（`C:\Windows\System32\config`） |
| 命名空间 | GUID | Registry Hive（HKLM/HKCU 等） |
| Key 格式 | GUID + Unicode 字符串 | 路径字符串 |
| Value 格式 | 任意二进制 | 类型化（REG_SZ/REG_DWORD 等） |
| 断电保持 | ✅ 非易失性（NV 属性） | ✅（存在硬盘上） |
| 访问控制 | 属性标志 + 认证变量 | ACL（访问控制列表） |
| 容量 | 很小（通常几百 KB） | 很大（几百 MB） |
| 读写速度 | 慢（SPI Flash） | 快（硬盘/SSD） |
| 树形结构 | ❌ 扁平 | ✅ 树形层级 |
| 事务保护 | FTW 容错写入 | Registry Transaction |

### 最关键的区别：容量

注册表可以几百 MB 甚至几 GB 乱造，但 UEFI 变量区通常只有 **256KB ~ 1MB**。

这意味着：
- 不能在 UEFI 变量里存大数据
- 变量的数量和大小都要精打细算
- 写入次数也要注意（Flash 有寿命）

> 打个比方：注册表是个大仓库，你想堆多少东西都行。UEFI 变量是个保险箱——地方很小，只放最重要的东西。

---

## 九、认证变量（Authenticated Variables）

有些 UEFI 变量非常敏感——比如 Secure Boot 的密钥。如果谁都能改，安全启动就形同虚设了。

UEFI 规范定义了**认证变量**机制：修改这些变量时，必须提供**数字签名**。

```
普通变量：
  SetVariable("MyConfig", data)  →  直接写入  ✅

认证变量（如 PK、KEK、db）：
  SetVariable("PK", signed_data)
  → 验证签名是否有效
  → 签名不对？拒绝写入  ❌
  → 签名正确？允许写入  ✅
```

Secure Boot 相关的变量链：

```
PK（Platform Key）        → 信任链的根，由 OEM 设置
  ↓ PK 签名授权
KEK（Key Exchange Key）   → 可以更新 db/dbx
  ↓ KEK 签名授权
db（Signature Database）  → 允许启动的签名列表
dbx（Forbidden Sigs）     → 禁止启动的签名黑名单
```

> 💡 认证变量 = 给保险箱加了密码锁。不是你说改就能改的——得有钥匙（私钥签名）。

---

## 十、常见疑问

### Q1："恢复 BIOS 默认设置" 到底干了什么？

清除（或重置）NV 变量区中的用户设置变量，让固件回到出厂默认值。具体实现方式有两种：

1. **擦除整个 NV 变量区**，让固件下次启动时使用代码中的默认值
2. **仅删除特定变量**，保留 Secure Boot 密钥等关键变量

大多数主板 BIOS 选的是方式 2。

### Q2：UEFI 变量区满了会怎样？

变量区快满时，固件会触发**垃圾回收**——把被标记为已删除的旧变量清理掉，腾出空间。如果垃圾回收完了还是满的……那 `SetVariable()` 就返回 `EFI_OUT_OF_RESOURCES`，写入失败。

有些固件 Bug 会导致变量区被垃圾填满，这就是为什么你偶尔看到"BIOS 更新后设置被重置"——有时是故意清空变量区来修复问题。

### Q3：为什么 Linux 下 rm -rf 删 UEFI 变量能把电脑搞砖？

2016 年轰动一时的事件：在某些联想笔记本上执行 `rm -rf /sys/firmware/efi/efivars/*` 直接导致电脑变砖——因为把所有 UEFI 变量（包括启动项）都删光了，固件下次启动找不到任何引导设备。

后来 Linux 内核加了保护，不让你轻易删除关键变量。但这个故事完美说明了 UEFI 变量的重要性——**它是固件的"记忆"，删掉等于固件失忆**。

### Q4：UEFI 变量和 CMOS/NVRAM 是什么关系？

传统 BIOS 时代，设置存在 **CMOS RAM**（由纽扣电池供电的小容量存储）里。UEFI 时代，变量存在 **SPI Flash** 上，不需要电池。但人们习惯性地还是把 UEFI 变量区叫"NVRAM"（非易失性存储器）。

```
传统 BIOS：  CMOS RAM（电池供电，128 字节~几 KB）
UEFI：      SPI Flash（不需要电池，几百 KB~1MB）
```

---

## 十一、实战：在 EDK2 中读写一个自定义变量

```c
#include <Library/UefiRuntimeServicesTableLib.h>
#include <Library/UefiLib.h>

// 定义自己的变量 GUID（命名空间）
EFI_GUID gMyAppVariableGuid = 
  { 0xAABBCCDD, 0x1122, 0x3344, 
    { 0x55, 0x66, 0x77, 0x88, 0x99, 0xAA, 0xBB, 0xCC } };

// 写入变量
EFI_STATUS WriteMyVariable(VOID)
{
    UINT32 MyData = 42;
    
    return gRT->SetVariable(
        L"MyFavoriteNumber",           // 变量名
        &gMyAppVariableGuid,           // GUID
        EFI_VARIABLE_NON_VOLATILE |    // 断电不丢
        EFI_VARIABLE_BOOTSERVICE_ACCESS |
        EFI_VARIABLE_RUNTIME_ACCESS,
        sizeof(MyData),                // 数据大小
        &MyData                        // 数据指针
    );
}

// 读取变量
EFI_STATUS ReadMyVariable(VOID)
{
    UINT32  MyData = 0;
    UINTN   DataSize = sizeof(MyData);
    UINT32  Attributes;
    
    EFI_STATUS Status = gRT->GetVariable(
        L"MyFavoriteNumber",
        &gMyAppVariableGuid,
        &Attributes,
        &DataSize,
        &MyData
    );
    
    if (!EFI_ERROR(Status)) {
        Print(L"读到的值: %d\n", MyData);  // 输出：读到的值: 42
    } else {
        Print(L"变量不存在或读取失败: %r\n", Status);
    }
    
    return Status;
}

// 删除变量（把 DataSize 设为 0）
EFI_STATUS DeleteMyVariable(VOID)
{
    return gRT->SetVariable(
        L"MyFavoriteNumber",
        &gMyAppVariableGuid,
        0,       // 属性设为 0
        0,       // 大小设为 0
        NULL     // 数据为 NULL
    );
}
```

> 💡 删除变量的方法有点反直觉——不是调 `DeleteVariable()`（没有这个 API），而是调 `SetVariable()` 时把大小设为 0。简约设计。

---

## 十二、总结

```
UEFI 变量系统：

存储层：  SPI Flash 上的 NV Variable Store
         ├── 追加写入 + 垃圾回收
         └── FTW 容错保护

接口层：  Runtime Services
         ├── GetVariable()      → 读
         ├── SetVariable()      → 写 / 删
         └── GetNextVariableName() → 遍历

访问控制：
         ├── BS / RT 属性 → 谁能访问
         ├── NV 属性 → 是否持久化
         └── 认证变量 → 签名保护（Secure Boot 密钥）

典型用途：
         ├── 启动顺序（BootOrder）
         ├── 安全启动密钥（PK/KEK/db/dbx）
         ├── BIOS 设置选项（Setup）
         └── OEM 自定义配置
```

| 你以为的 UEFI 变量 | 实际的 UEFI 变量 |
|-------------------|-----------------|
| 和普通变量差不多 | 存在 Flash 上的持久化键值对 |
| 随便读写 | 有访问属性控制，敏感变量需要签名 |
| 容量无限 | 通常只有几百 KB，寸土寸金 |
| 和操作系统无关 | OS 运行时依然可以读写（Runtime Services） |

---

下一篇：**#019 UEFI 事件和定时器：固件里的异步机制**——固件也有"回调函数"和"定时器"？没错，UEFI 的事件系统让固件可以做异步操作，虽然它没有操作系统那样的线程。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
