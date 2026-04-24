# UEFI 中的 Handle 和 Protocol：理解 UEFI 的面向对象思想

> 🔥 UEFI/BSP 开发系列第 015 篇 | 难度：⭐ 入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

前面几篇我们反复提到了两个词：**Handle** 和 **Protocol**。

"在 Handle 上安装 Protocol"、"通过 Protocol 调用功能"——第一次听到这些术语，脑子里全是问号。

但这两个概念是理解整个 UEFI 架构的**钥匙**。搞懂了它们，你就能理解 UEFI 为什么能用 C 语言实现一套面向对象系统，以及为什么 UEFI 的驱动模型比传统 BIOS 强大得多。

今天就把这两个概念掰开了揉碎了讲清楚。

---

## 一、先用生活中的例子理解

想象你走进一家大型医院。

- **Handle（句柄）= 每位医生的工牌**
  - 工牌上有唯一编号，代表这个人的身份
  - 一个工牌可以挂多个头衔："内科主任"、"心脏专家"、"博士生导师"

- **Protocol（协议）= 医生的技能/服务**
  - "内科诊断"是一项 Protocol——它定义了"能看病、能开药、能写病历"
  - "心脏手术"是另一项 Protocol——它定义了"能做搭桥、能装支架"
  
- **一个 Handle 上可以安装多个 Protocol**
  - 张医生（Handle）同时具备"内科诊断"和"心脏手术"两项 Protocol

- **你想看病时，不需要认识张医生本人**
  - 你只需要告诉前台："我需要一个能做心脏手术的医生"
  - 前台根据 Protocol 类型，找到挂了这项 Protocol 的 Handle，把你引过去

映射到 UEFI：

```
医院            = UEFI 系统
医生工牌        = Handle（句柄）
医生的技能      = Protocol（协议）
前台            = HandleProtocol() / LocateProtocol() 函数
你（患者）      = 调用方（其他驱动或应用）
```

---

## 二、Handle：UEFI 世界的"身份证"

Handle 在代码中是一个**不透明的指针**：

```c
typedef VOID *EFI_HANDLE;
```

你不需要（也不应该）知道 Handle 内部长什么样。你只需要拿着这个"身份证"去查询它上面有哪些 Protocol。

### Handle 的种类

| Handle 类型 | 代表什么 |
|------------|---------|
| **Image Handle** | 一个已加载的 EFI 程序/驱动 |
| **Device Handle** | 一个硬件设备 |
| **Agent Handle** | 管理某个 Protocol 的驱动 |

比如，你的 NVMe 硬盘在 UEFI 系统中会有一个 Device Handle，上面挂着 Block IO Protocol、Device Path Protocol 等。

```
NVMe 设备 Handle
├── Block IO Protocol        （读写磁盘）
├── Device Path Protocol     （设备路径标识）
├── NVM Express PassThru     （NVMe 命令直通）
└── Disk Info Protocol       （磁盘信息）
```

---

## 三、Protocol：UEFI 世界的"接口"

Protocol 就是一个**结构体，里面装着函数指针和数据**。

以最常见的 Block IO Protocol 为例：

```c
typedef struct _EFI_BLOCK_IO_PROTOCOL {
    UINT64                  Revision;
    EFI_BLOCK_IO_MEDIA      *Media;          // 磁盘媒体信息
    EFI_BLOCK_RESET         Reset;           // 重置设备
    EFI_BLOCK_READ          ReadBlocks;      // 读块
    EFI_BLOCK_WRITE         WriteBlocks;     // 写块
    EFI_BLOCK_FLUSH         FlushBlocks;     // 刷新缓冲
} EFI_BLOCK_IO_PROTOCOL;
```

其中 `ReadBlocks`、`WriteBlocks` 等都是**函数指针**。NVMe 驱动会把自己实现的读写函数赋值给这些指针，然后把这个 Protocol 挂到 Handle 上。

文件系统驱动想读磁盘时：
1. 找到一个有 Block IO Protocol 的 Handle
2. 调用 `BlockIo->ReadBlocks()` 
3. 底层是 NVMe 还是 SATA？不知道也不需要知道

> 💡 这就是**多态**——同一个接口（`ReadBlocks`），不同实现（NVMe/SATA/USB），调用者无感知。纯 C 语言实现的面向对象。

### 每个 Protocol 都有一个 GUID

GUID（全局唯一标识符）是 Protocol 的"身份证号"：

```c
// Block IO Protocol 的 GUID
#define EFI_BLOCK_IO_PROTOCOL_GUID \
  { 0x964e5b21, 0x6459, 0x11d2, \
    { 0x8e, 0x39, 0x00, 0xa0, 0xc9, 0x69, 0x72, 0x3b } }
```

在整个 UEFI 世界里，这个 GUID 唯一标识 Block IO Protocol。你想找 Block IO，就用这个 GUID 去查。

---

## 四、核心操作：安装和查找 Protocol

### 4.1 安装 Protocol（驱动端）

驱动初始化时，把 Protocol 安装到 Handle 上：

```c
// NVMe 驱动在初始化时安装 Block IO Protocol
Status = gBS->InstallProtocolInterface(
    &DeviceHandle,                    // Handle
    &gEfiBlockIoProtocolGuid,         // Protocol 的 GUID
    EFI_NATIVE_INTERFACE,
    &mMyBlockIoProtocol               // Protocol 实例（函数指针表）
);
```

### 4.2 查找 Protocol（调用端）

其他模块想使用 Block IO：

```c
EFI_BLOCK_IO_PROTOCOL *BlockIo;

// 方法 1：知道具体 Handle，直接从 Handle 上取
Status = gBS->HandleProtocol(
    DeviceHandle,
    &gEfiBlockIoProtocolGuid,
    (VOID **)&BlockIo
);

// 方法 2：不知道 Handle，让系统帮我找第一个有 Block IO 的 Handle
Status = gBS->LocateProtocol(
    &gEfiBlockIoProtocolGuid,
    NULL,
    (VOID **)&BlockIo
);

// 方法 3：找出所有有 Block IO Protocol 的 Handle
EFI_HANDLE *HandleBuffer;
UINTN HandleCount;
Status = gBS->LocateHandleBuffer(
    ByProtocol,
    &gEfiBlockIoProtocolGuid,
    NULL,
    &HandleCount,
    &HandleBuffer
);
// HandleCount 个 Handle，每个都有 Block IO
```

---

## 五、用一个完整的例子串起来

当你在 UEFI Shell 里输入 `fs0:\HelloWorld.efi` 时，背后发生了什么：

```
UEFI Shell 需要读取 HelloWorld.efi 文件
    ↓
Shell 调用 File System Protocol → OpenVolume() 打开文件系统
    ↓
FAT 文件系统驱动收到请求
    ↓
FAT 驱动需要读磁盘 → 调用 Block IO Protocol → ReadBlocks()
    ↓
NVMe 驱动的 ReadBlocks() 函数被调用
    ↓
NVMe 驱动操作 NVMe 控制器寄存器，读取物理数据
    ↓
数据一层层返回 → FAT 驱动解析文件 → Shell 拿到 HelloWorld.efi
    ↓
Shell 调用 Image Load Protocol 加载并执行 HelloWorld.efi
```

每一层之间的通信都是通过 Protocol。没有直接的函数调用，没有全局变量耦合。

```
Shell
  ↕ File System Protocol
FAT Driver
  ↕ Block IO Protocol
NVMe Driver
  ↕ 硬件寄存器操作
NVMe Controller
```

---

## 六、Handle-Protocol 机制 vs 其他设计模式

| UEFI 概念 | 面向对象类比 | Java 类比 | Linux 内核类比 |
|-----------|------------|----------|--------------|
| Handle | 对象实例 | Object | struct device |
| Protocol | 接口 | Interface | struct file_operations |
| GUID | 接口名称 | Interface 全限定名 | - |
| InstallProtocol | 实现接口 | implements | 注册 ops |
| LocateProtocol | 查找接口实现 | ServiceLoader | 查找 driver |

如果你熟悉设计模式，Handle-Protocol 机制本质上就是**服务定位器模式（Service Locator Pattern）**——Protocol 是服务接口，Handle 是服务实例，UEFI 的 Handle Database 是服务注册表。

---

## 七、总结

| 概念 | 本质 | 类比 |
|------|------|------|
| **Handle** | 不透明指针，代表一个实体（设备/程序） | 对象指针 |
| **Protocol** | 函数指针表（结构体） | Interface / API |
| **GUID** | Protocol 的唯一标识 | 接口名称 |
| **Install** | 把 Protocol 挂到 Handle 上 | 实现接口 |
| **Locate** | 根据 GUID 找到 Protocol 实例 | 服务发现 |

> **一句话总结**：Handle 是"谁"，Protocol 是"能做什么"，GUID 是"做的具体是哪件事"。

---

下一篇：**#016 从 C 语言角度理解 UEFI Protocol——就是函数指针表**——用代码实例彻底搞懂 Protocol 的实现原理。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
