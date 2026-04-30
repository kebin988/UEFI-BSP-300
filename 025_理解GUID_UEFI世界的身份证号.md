# 理解 GUID：UEFI 世界的身份证号

> 🔥 UEFI/BSP 开发系列第 025 篇 | 难度：⭐ 入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们搞明白了 UEFI 的事件和定时器——固件里的异步机制。

不知道你有没有发现，在前面的文章里，有一个东西反复出现：

```c
EFI_GUID gMyProtocolGuid = { 0x12345678, 0xabcd, ... };
EFI_GUID gEfiGlobalVariableGuid = { 0x8BE4DF61, 0x93CA, ... };
```

Protocol 有 GUID，变量有 GUID，事件有 GUID，驱动有 GUID……在 UEFI 的世界里，**GUID 无处不在**。

但 GUID 到底是什么？为什么不用简单的字符串或数字来标识？怎么保证全球唯一？今天一次讲清楚。

---

## 一、GUID 是什么？

GUID 的全称是 **Globally Unique Identifier**（全球唯一标识符），也叫 UUID（Universally Unique Identifier）。

本质上，它就是一个 **128 位（16 字节）的数字**，通常表示为这种格式：

```
8BE4DF61-93CA-11D2-AA0D-00E098032B8C
^^^^^^^^ ^^^^ ^^^^ ^^^^ ^^^^^^^^^^^^
 8位     4位  4位  4位    12位
```

总共 32 个十六进制字符，分成 5 段，中间用 `-` 分隔。

### 有多少种可能？

128 位 = $2^{128} \approx 3.4 \times 10^{38}$ 种可能。

这个数有多大？打个比方：

- 地球上的沙粒数量 ≈ $7.5 \times 10^{18}$
- 可观测宇宙中的原子数量 ≈ $10^{80}$
- GUID 的数量 ≈ $10^{38}$

也就是说，GUID 的数量比地球上沙粒的数量还多 **20 个数量级**。两个随机生成的 GUID 碰撞（重复）的概率，比你连续被闪电击中两次还低。

> 💡 用行话说：GUID 在实践中可以视为**绝对不会重复**的。

---

## 二、UEFI 为什么选择 GUID？

你可能会问：用字符串不行吗？用整数编号不行吗？

### 字符串的问题

```
方案 A：用字符串标识 Protocol
  "SimpleTextOutput"          → 谁来管理命名？
  "SimpleTextOutputProtocol"  → 不同厂商可能取同名
  "ACME_SimpleTextOutput"     → 命名规范谁来制定？
```

字符串需要一个**中心化的命名管理机构**来避免重名。但 UEFI 的世界里有几百家芯片厂商、OEM、ODM，不可能让所有人都去一个地方注册名字。

### 整数编号的问题

```
方案 B：用整数编号
  Protocol #1, Protocol #2, Protocol #3...
  → 谁来分配编号？
  → 两家厂商同时分配到 #42 怎么办？
```

同样需要中心化管理。

### GUID 的优势

```
方案 C：GUID
  每个人自己随机生成 → 不需要中心管理机构
  128 位空间 → 碰撞概率 ≈ 0
  标准算法 → 所有平台通用
```

GUID 的最大优势是**去中心化**——任何人、在任何时候、在任何地方都可以独立生成一个 GUID，不用跟任何人商量，也不用担心重复。

> 打个比方：字符串就像给孩子起名——你叫"张伟"，他也叫"张伟"，满大街重名。GUID 就像身份证号——18 位数字，全国唯一，你不需要跟别人商量就能生成一个不重复的。

---

## 三、UEFI 中 GUID 的数据结构

在 EDK2 代码中，GUID 的定义如下：

```c
typedef struct {
    UINT32  Data1;     // 4 字节
    UINT16  Data2;     // 2 字节
    UINT16  Data3;     // 2 字节
    UINT8   Data4[8];  // 8 字节
} EFI_GUID;            // 总共 16 字节 = 128 位
```

对应到可读格式：

```
{Data1}-{Data2}-{Data3}-{Data4[0..1]}-{Data4[2..7]}

例如：
{ 0x8BE4DF61, 0x93CA, 0x11D2, 
  { 0xAA, 0x0D, 0x00, 0xE0, 0x98, 0x03, 0x2B, 0x8C } }

对应：8BE4DF61-93CA-11D2-AA0D-00E098032B8C
```

### 在代码中定义 GUID

```c
// 方式 1：直接定义
EFI_GUID gMyProtocolGuid = {
    0x12345678, 0xABCD, 0xEF01,
    { 0x23, 0x45, 0x67, 0x89, 0xAB, 0xCD, 0xEF, 0x01 }
};

// 方式 2：用宏（EDK2 推荐方式）
#define MY_PROTOCOL_GUID \
    { 0x12345678, 0xABCD, 0xEF01, \
      { 0x23, 0x45, 0x67, 0x89, 0xAB, 0xCD, 0xEF, 0x01 } }

// 方式 3：在 .DEC 文件中声明（最标准的方式）
// [Protocols]
//   gMyProtocolGuid = { 0x12345678, 0xABCD, 0xEF01, ... }
```

---

## 四、UEFI 中 GUID 的使用场景

GUID 在 UEFI 中无处不在，这里列出最常见的用途：

### 1. Protocol 标识

```c
// EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL 的 GUID
#define EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL_GUID \
    { 0x387477C2, 0x69C7, 0x11D2, \
      { 0x8E, 0x39, 0x00, 0xA0, 0xC9, 0x69, 0x72, 0x3B } }

// 用 GUID 查找 Protocol
gBS->LocateProtocol(&gEfiSimpleTextOutputProtocolGuid, NULL, (VOID**)&TextOut);
```

### 2. UEFI 变量命名空间

```c
// 全局变量 GUID
#define EFI_GLOBAL_VARIABLE_GUID \
    { 0x8BE4DF61, 0x93CA, 0x11D2, \
      { 0xAA, 0x0D, 0x00, 0xE0, 0x98, 0x03, 0x2B, 0x8C } }

// 读取 BootOrder 变量
gRT->GetVariable(L"BootOrder", &gEfiGlobalVariableGuid, ...);
```

### 3. 固件卷（FV）和文件标识

```
固件卷中的每个文件都有一个 GUID：
  FFS File GUID: A8B8249A-2D10-4C5F-BBCE-50A9C81A3D61  → DxeCore
  FFS File GUID: 9B680FCE-AD6B-4F3A-B60B-F59899003443  → DeviceManager
```

### 4. 事件组标识

```c
// ExitBootServices 事件组的 GUID
#define EFI_EVENT_GROUP_EXIT_BOOT_SERVICES \
    { 0x27ABF055, 0xB1B8, 0x4C26, \
      { 0x80, 0x48, 0x74, 0x8F, 0x37, 0xBA, 0xA2, 0xDF } }
```

### 5. PCD（Platform Configuration Database）Token Space

```c
// Token Space GUID
[Guids]
  gEfiMdePkgTokenSpaceGuid = { 0x914AEBE7, 0x4635, 0x459B, ... }
```

### 6. HOB（Hand-Off Block）类型

```c
// 内存分配 HOB 的 GUID
EFI_HOB_GUID_TYPE  *GuidHob;
GuidHob = GetFirstGuidHob(&gEfiMemoryTypeInformationGuid);
```

### 一张表总结

| 使用场景 | 例子 | 数量级 |
|---------|------|--------|
| Protocol | `gEfiBlockIoProtocolGuid` | 几百个 |
| 变量命名空间 | `gEfiGlobalVariableGuid` | 几十个 |
| 固件文件 | 每个 .efi 模块的 GUID | 几百个 |
| 事件组 | `ExitBootServices` 事件 | 十几个 |
| PCD Token Space | 每个 Package 一个 | 几十个 |
| HOB | 各种 GUID HOB | 几十个 |

> 💡 一个典型的 UEFI 固件里，会涉及 **上千个不同的 GUID**。

---

## 五、怎么生成 GUID？

### 方法 1：在线工具

最简单——打开网站，点一下就生成：

- https://www.guidgenerator.com
- https://www.uuidgenerator.net

### 方法 2：命令行工具

**Windows PowerShell**：
```powershell
[guid]::NewGuid()
# 输出：3f2504e0-4f89-11d3-9a0c-0305e82c3301
```

**Linux**：
```bash
uuidgen
# 输出：3f2504e0-4f89-11d3-9a0c-0305e82c3301
```

**Python**：
```python
import uuid
print(uuid.uuid4())
# 输出：3f2504e0-4f89-11d3-9a0c-0305e82c3301
```

### 方法 3：Visual Studio

在 VS 的"工具"菜单里有"Create GUID"工具，可以直接生成各种格式的 GUID。

### 方法 4：EDK2 工具

```bash
# EDK2 自带的 GUID 生成脚本（Python）
python -c "import uuid; g=uuid.uuid4(); print('{0x%08X, 0x%04X, 0x%04X, {0x%02X, 0x%02X, 0x%02X, 0x%02X, 0x%02X, 0x%02X, 0x%02X, 0x%02X}}' % (g.time_low, g.time_mid, g.time_hi_version, g.clock_seq_hi_variant, g.clock_seq_low, *g.node.to_bytes(6,'big')))"
```

> ⚠️ **重要**：每个 Protocol、变量、模块都必须使用**独一无二**的 GUID。绝对不要复制别人的 GUID 来用——那就像用别人的身份证号一样，会出大问题。

---

## 六、GUID 的版本

UUID/GUID 有好几个版本，最常用的是：

| 版本 | 生成方式 | 特点 | 在 UEFI 中的使用 |
|------|---------|------|-----------------|
| V1 | 时间戳 + MAC 地址 | 可追溯生成时间和机器 | 早期 UEFI 规范中常用 |
| V4 | 纯随机数 | 最简单，隐私友好 | 新开发中推荐 |
| V5 | 基于名称的 SHA-1 哈希 | 相同输入 → 相同 GUID | 较少使用 |

怎么看版本？看第三段的第一位数字：

```
8BE4DF61-93CA-11D2-AA0D-00E098032B8C
                   ^
                   1 = V1（基于时间）

550E8400-E29B-41D4-A716-446655440000
                   ^
                   4 = V4（随机）
```

> 💡 UEFI 规范中很多"官方" GUID 是 V1 的（因为最早制定规范时 V4 还不流行），你自己新写的代码用 V4 就行——随机生成，简单安全。

---

## 七、GUID 比较：怎么判断两个 GUID 相等？

在 EDK2 中，比较两个 GUID 的方式：

```c
#include <Library/BaseMemoryLib.h>

// 方式 1：用 CompareGuid（推荐）
if (CompareGuid(&Guid1, &Guid2)) {
    // 相等
}

// 方式 2：用 CompareMem（通用）
if (CompareMem(&Guid1, &Guid2, sizeof(EFI_GUID)) == 0) {
    // 相等
}
```

**千万不要这样比较**：

```c
// ❌ 错误！结构体不能直接用 == 比较
if (Guid1 == Guid2) { ... }  // 编译错误或未定义行为
```

---

## 八、著名的 GUID 们

下面列出几个 UEFI 开发中最常遇到的 GUID，混个脸熟：

```c
// Boot Services 中最常用的 Protocol
gEfiSimpleTextOutputProtocolGuid    // 屏幕输出
gEfiSimpleTextInputProtocolGuid     // 键盘输入
gEfiBlockIoProtocolGuid             // 块设备（硬盘/U盘）
gEfiDevicePathProtocolGuid          // 设备路径
gEfiDriverBindingProtocolGuid       // 驱动绑定
gEfiGraphicsOutputProtocolGuid      // 图形输出（GOP）

// UEFI 变量
gEfiGlobalVariableGuid              // 全局变量命名空间

// 固件文件系统
gEfiFirmwareFileSystem2Guid         // FFS v2 文件系统
gEfiFirmwareVolumeTopFileGuid       // 固件卷的"目录"

// 事件
gEfiEventExitBootServicesGuid       // ExitBootServices 事件组
gEfiEventReadyToBootGuid            // ReadyToBoot 事件组
```

> 💡 干久了你会发现，很多 GUID 的前几位你一眼就能认出来——就像老司机一眼认出车牌号归属地一样。

---

## 九、常见疑问

### Q1：GUID 真的不会重复吗？

理论上有极小概率重复。V4 UUID 使用 122 位随机数（128 位减去 6 位版本和变体标记），碰撞概率需要生成约 $2^{61}$（约 23 亿亿）个 GUID 才有 50% 的概率出现一次碰撞。在实际工程中可以认为不会重复。

### Q2：为什么不用哈希值？

哈希值（如 MD5、SHA-1）是对输入数据的摘要，长度固定但有碰撞风险。GUID 是专门为"标识"设计的，有标准的生成算法和格式规范。实际上 UUID V5 就是基于 SHA-1 的，所以两者并不矛盾。

### Q3：我在 .DEC 文件和 .INF 文件中都看到 GUID，它们有什么关系？

- **.DEC 文件**：声明 GUID（"我这个包定义了这些 GUID"）
- **.INF 文件**：引用 GUID（"我这个模块要用这些 GUID"）

```ini
# SomePkg.dec 中声明
[Protocols]
  gSomeProtocolGuid = { 0x1234... }

# SomeDriver.inf 中引用
[Protocols]
  gSomeProtocolGuid    ## CONSUMES  表示该模块使用这个 Protocol
```

### Q4：两个不同厂商碰巧生成了一样的 GUID 怎么办？

不会发生。但如果你**手动复制**了别人的 GUID（偷懒没有自己生成），那就可能冲突。所以规则很简单：**永远自己生成新的 GUID，不要复制别人的**。

---

## 番外：UEFI GUID vs Windows 驱动 GUID，到底有什么区别？

如果你同时接触过 UEFI 开发和 Windows 驱动开发（WDM/WDF/KMDF），你会发现两边都大量使用 GUID。它们到底一不一样？

### 数据结构完全一样

```c
// UEFI 定义（在 MdePkg/Include/Uefi/UefiBaseType.h）
typedef struct {
  UINT32  Data1;
  UINT16  Data2;
  UINT16  Data3;
  UINT8   Data4[8];
} EFI_GUID;

// Windows 定义（在 guiddef.h）
typedef struct _GUID {
  unsigned long  Data1;
  unsigned short Data2;
  unsigned short Data3;
  unsigned char  Data4[8];
} GUID;
```

二者在内存中布局**完全相同**，都是 16 字节，只是类型名不同（`EFI_GUID` vs `GUID`）。

### 用法对比

| 场景 | UEFI | Windows 驱动 |
|------|------|-------------|
| **标识协议/接口** | Protocol GUID（如 `gEfiSimpleTextOutProtocolGuid`） | Device Interface GUID（如 `GUID_DEVINTERFACE_DISK`） |
| **标识设备** | Handle + Protocol | Device Instance ID + Interface GUID |
| **变量命名空间** | Variable GUID + Name | 注册表 Key + Value Name |
| **事件/通知** | Event Group GUID | PnP Event GUID / WMI GUID |
| **驱动匹配** | DepEx 中声明依赖的 GUID | INF 中声明的 Class GUID |
| **IPC 通信** | 无直接机制 | DeviceIoControl + IOCTL |

### 核心思想对比

| | UEFI | Windows |
|---|---|---|
| **Protocol** | 一张函数指针表，通过 GUID 查找 | COM 接口 / Device Interface，通过 GUID 查找 |
| **发现机制** | `LocateProtocol(GUID)` | `SetupDiGetClassDevs(GUID)` / `IoGetDeviceInterfaces(GUID)` |
| **注册机制** | `InstallProtocolInterface(Handle, GUID, Interface)` | `IoRegisterDeviceInterface(PDO, GUID)` |

### 能不能互通？

技术上可以！因为二进制格式完全一致。例如：

- UEFI 的 Secure Boot 证书 GUID，和 Windows 内核读取 UEFI 变量时用的是**同一个 GUID**
- Windows 通过 `GetFirmwareEnvironmentVariable()` API 读取 UEFI 变量时，传入的 GUID 就是 UEFI 变量的 Namespace GUID

```c
// Windows 用户态读取 UEFI 变量
GetFirmwareEnvironmentVariable(
    L"SecureBoot",                    // 变量名
    L"{8be4df61-93ca-11d2-aa0d-00e098032b8c}",  // 这就是 EFI_GLOBAL_VARIABLE GUID！
    &value,
    sizeof(value)
);
```

### 一句话总结

**UEFI 的 GUID 和 Windows 的 GUID 是同一个东西（RFC 4122 UUID），只是在各自生态中扮演不同角色。** 就像身份证号在银行系统和医保系统里格式一样，但用途不同。

---

## 十、总结

```
GUID 在 UEFI 中的角色：

什么是 GUID？
  → 128 位全球唯一标识符
  → 去中心化：谁都可以独立生成，不怕重复

为什么用 GUID？
  → 不需要中心管理机构
  → 几百家厂商可以独立开发，互不干扰
  → 跨平台、跨编译器通用

在哪里用？
  ├── Protocol 标识
  ├── 变量命名空间
  ├── 固件文件标识
  ├── 事件组标识
  ├── PCD Token Space
  └── HOB 类型标识

怎么生成？
  → PowerShell / uuidgen / Python / 在线工具
  → 务必每次生成新的，不要复制别人的
```

| 你以为的 GUID | 实际的 GUID |
|--------------|------------|
| 随便写的一串数字 | 128 位全球唯一标识符 |
| 用于调试，不重要 | UEFI 的基础设施，到处都是 |
| 容易重复 | 碰撞概率低到可以忽略 |
| 难以生成 | 一行命令就能生成 |

---

下一篇：**#026 什么是 FV、FFS、FD？UEFI 固件的打包格式**——你编译出的 .efi 文件，最终是怎么打包进一个完整的 BIOS 镜像的？FV（固件卷）、FFS（固件文件系统）、FD（Flash Device）——这三层套娃，今天拆开看。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
