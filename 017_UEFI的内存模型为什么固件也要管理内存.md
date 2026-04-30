# UEFI 的内存模型：为什么固件也要管理内存？

> 🔥 UEFI/BSP 开发系列第 017 篇 | 难度：⭐⭐ 进阶入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们搞明白了 Protocol 的本质——就是函数指针表。

但你有没有想过一个很基础的问题：

> "这些 Protocol 结构体、Handle 数据库、驱动代码……它们住在哪里？"

答案当然是——**内存**。

但问题来了：**内存不是上电就能用的**。DDR 芯片刚通电的时候，里面是一坨乱码，你往里写什么都白搭。必须经过一系列复杂的训练（Memory Training），内存控制器才能正确读写。

这意味着 UEFI 固件有一段时间是**没有内存可用**的。那它是怎么活下来的？又是怎么一步步从"内存荒"走到"内存自由"的？

今天就把这个故事讲清楚。

---

## 一、操作系统的内存管理你很熟

先回忆一下你熟悉的世界。在 Linux / Windows 里：

```
应用程序：malloc(1024)  →  操作系统：好的，给你一块虚拟内存
                         →  MMU + 页表：虚拟地址 → 物理地址
                         →  物理内存：真正的 DDR 芯片上分配
```

操作系统启动的时候，内存已经初始化好了，虚拟内存机制也建好了。应用程序只管 `malloc / free`，完全不用操心底层。

但固件不一样——**固件是在操作系统之前运行的**。它面临的局面是：

```
CPU 上电
  ↓
内存？还没初始化呢！
  ↓
硬盘？驱动还没加载呢！
  ↓
页表？MMU 还没配呢！
  ↓
固件：就靠我自己了……
```

> 💡 操作系统是"含着金钥匙出生"的——内存、文件系统、驱动全都准备好了。固件是"白手起家"——啥都没有，全靠自己一个个搞出来。

---

## 二、固件的三个内存阶段

UEFI 固件的内存使用经历了三个截然不同的阶段，就像一个人从穷到富的过程：

```
阶段 1：SEC       → 赤贫期      → 只有 CPU 寄存器，啥都没有
阶段 2：PEI       → 创业初期    → 靠 Cache 当内存勉强度日（CAR）
阶段 3：DXE/BDS   → 财务自由    → 真正的 DDR 内存随便用
```

### 阶段 1：SEC — 赤贫期（只有寄存器）

CPU 刚上电，跳到 Reset Vector（第 032/033 篇会详细讲）。此时：

- ❌ DDR 内存不可用
- ❌ Cache 还没配置
- ❌ 栈（Stack）不存在
- ✅ 只有 CPU 内部的几十个寄存器

这时候能干的事极其有限——基本就是用汇编做一些最低限度的初始化（比如切换 CPU 模式、设置 Cache 策略），然后尽快把 Cache 搞成可以当内存用的状态。

> 打个比方：你刚来到一座荒岛，身上只有一把瑞士军刀（寄存器），连个帐篷都没有。

### 阶段 2：PEI — 创业初期（Cache As RAM）

SEC 阶段完成后，进入 PEI（Pre-EFI Initialization）。此时 DDR 内存**依然不可用**，但我们有了一个救命的技巧：

**CAR（Cache As RAM）**—— 把 CPU 的 L1/L2 Cache 配置成普通内存来用。

```
正常情况：
  CPU ←→ Cache ←→ DDR
         (缓存)   (主存)

CAR 模式：
  CPU ←→ Cache ←→ ✕ (DDR 还没好)
         ↑
         当内存用！
```

Cache 通常只有几百 KB 到几 MB，虽然少得可怜，但足够 PEI 阶段完成最关键的任务：

- 建立一个小栈（Stack），可以跑 C 语言了
- 加载并执行 PEIM（PEI 模块）
- **初始化 DDR 内存**（Memory Reference Code / MRC / FSP-M）

PEI 阶段的核心使命就一个：**把真正的内存搞好**。

> 打个比方：你用树枝搭了个简易棚子（CAR），然后在棚子里造砖窑、烧砖，最终盖出一栋真正的房子（DDR）。

### 阶段 3：DXE — 财务自由（真正的内存）

DDR 内存初始化完成后，PEI 通过 HOB（Hand-Off Block）把内存信息传递给 DXE，系统进入"有钱人"阶段：

- ✅ 几 GB 甚至几十 GB 的 DDR 内存可用
- ✅ 完整的内存分配器（`AllocatePool` / `AllocatePages`）
- ✅ 可以加载几百个 DXE 驱动
- ✅ Protocol、Handle Database、UEFI 变量全住在内存里

> 打个比方：砖房盖好了，你可以在里面开公司、招员工、装修办公室——想怎么折腾都行。

---

## 三、UEFI 的内存分配 API

进入 DXE 阶段后，UEFI 提供了一套自己的内存管理 API（在 Boot Services 中）：

### 1. 按页分配（Page 级别，4KB 对齐）

```c
EFI_STATUS
gBS->AllocatePages(
    AllocateAnyPages,           // 分配类型
    EfiLoaderData,              // 内存类型
    NumberOfPages,              // 页数
    &PhysicalAddress            // 输出：分配到的物理地址
);

gBS->FreePages(PhysicalAddress, NumberOfPages);
```

### 2. 按池分配（任意大小，类似 malloc）

```c
VOID *Buffer;

gBS->AllocatePool(
    EfiLoaderData,              // 内存类型
    BufferSize,                 // 字节数
    &Buffer                     // 输出：指针
);

gBS->FreePool(Buffer);
```

> 💡 `AllocatePool` 就是 UEFI 版的 `malloc`，`FreePool` 就是 `free`。用法几乎一样，只是多了一个"内存类型"参数。

### 日常开发中怎么选？

| 场景 | 选择 | 原因 |
|------|------|------|
| 分配一小块数据结构 | `AllocatePool` | 方便，不需要对齐 |
| 分配 DMA 缓冲区 | `AllocatePages` | 需要页对齐，且要指定物理地址范围 |
| 分配设备映射区域 | `AllocatePages` + `AllocateAddress` | 需要精确的物理地址 |

---

## 四、UEFI 的内存类型（Memory Type）

UEFI 给每块内存打了"标签"，这些标签告诉操作系统："这块内存是干什么用的，你能不能动它。"

```c
typedef enum {
    EfiReservedMemoryType,      // 0  保留，谁都别动
    EfiLoaderCode,              // 1  引导程序代码（OS Loader）
    EfiLoaderData,              // 2  引导程序数据
    EfiBootServicesCode,        // 3  Boot Services 代码
    EfiBootServicesData,        // 4  Boot Services 数据
    EfiRuntimeServicesCode,     // 5  Runtime Services 代码（OS 不能回收！）
    EfiRuntimeServicesData,     // 6  Runtime Services 数据（OS 不能回收！）
    EfiConventionalMemory,      // 7  可用内存（OS 随便用）
    EfiUnusableMemory,          // 8  有硬件错误的内存
    EfiACPIReclaimMemory,       // 9  ACPI 表占用（OS 读完后可回收）
    EfiACPIMemoryNVS,           // 10 ACPI NVS（OS 不能回收！）
    EfiMemoryMappedIO,          // 11 MMIO 区域
    EfiMemoryMappedIOPortSpace, // 12 MMIO 端口空间
    EfiPalCode,                 // 13 IA-64 专用
    EfiMaxMemoryType
} EFI_MEMORY_TYPE;
```

### 关键理解：哪些内存操作系统能回收？

这是面试高频题，也是实际开发中最容易踩坑的地方：

| 内存类型 | ExitBootServices 后 | 操作系统能回收？ |
|---------|---------------------|----------------|
| EfiBootServicesCode/Data | 不再需要 | ✅ 可以回收 |
| EfiRuntimeServicesCode/Data | 还在使用 | ❌ 绝对不能动 |
| EfiConventionalMemory | 空闲 | ✅ 直接接管 |
| EfiACPIReclaimMemory | ACPI 表读完 | ✅ 可以回收 |
| EfiACPIMemoryNVS | ACPI 运行时需要 | ❌ 不能回收 |
| EfiLoaderCode/Data | OS Loader 决定 | ✅ 通常可回收 |

> 💡 **核心规则**：带 "Runtime" 字样的内存，操作系统启动后**依然活着**——因为 Runtime Services（如读写 UEFI 变量、获取时间）在 OS 运行时还会被调用。

---

## 五、内存映射表（Memory Map）

UEFI 维护了一张**内存映射表**，记录了每一块物理内存的起始地址、大小和类型。操作系统启动前，必须调用 `GetMemoryMap()` 拿到这张表：

```c
UINTN                  MemoryMapSize = 0;
EFI_MEMORY_DESCRIPTOR  *MemoryMap = NULL;
UINTN                  MapKey;
UINTN                  DescriptorSize;
UINT32                 DescriptorVersion;

// 第一次调用：获取所需的 buffer 大小
gBS->GetMemoryMap(&MemoryMapSize, MemoryMap, &MapKey, 
                  &DescriptorSize, &DescriptorVersion);

// 分配 buffer（多加一点余量，因为 AllocatePool 本身也会改变 Memory Map）
MemoryMapSize += 2 * DescriptorSize;
gBS->AllocatePool(EfiLoaderData, MemoryMapSize, (VOID **)&MemoryMap);

// 第二次调用：真正获取 Memory Map
gBS->GetMemoryMap(&MemoryMapSize, MemoryMap, &MapKey, 
                  &DescriptorSize, &DescriptorVersion);
```

拿到的 Memory Map 大概长这样：

```
Physical Start    Pages    Type
0x0000_0000       0x0A0    EfiConventionalMemory    (640 KB 可用)
0x000A_0000       0x060    EfiReservedMemoryType    (384 KB VGA/Legacy)
0x0010_0000       0x700    EfiBootServicesData      (7 MB Boot Services)
0x0080_0000       0x100    EfiRuntimeServicesCode   (1 MB Runtime Code)
0x0090_0000       0x37000  EfiConventionalMemory    (880 MB 大块可用)
0xFE00_0000       0x200    EfiMemoryMappedIO        (2 MB MMIO)
...
```

> 💡 Linux 内核在启动时打印的 `e820` 内存表，本质上就是从 UEFI Memory Map 转换过来的。你在 `dmesg` 里看到的那些 `usable`、`reserved`、`ACPI data`，来源就在这里。

---

## 六、ExitBootServices：内存的大交接

当操作系统准备接管硬件时，OS Loader 会调用一个关键函数：

```c
gBS->ExitBootServices(ImageHandle, MapKey);
```

这是 UEFI 和操作系统之间的"权力交接仪式"。一旦调用成功：

1. **Boot Services 全部失效**——`AllocatePool`、`LocateProtocol` 等函数再也不能调用
2. **Boot Services 占用的内存可以回收**——操作系统可以拿来自用
3. **Runtime Services 继续有效**——但只能在约定的内存区域运行
4. **中断、定时器全部关闭**——操作系统要自己重新设置

```
ExitBootServices 前：                ExitBootServices 后：
┌──────────────────────┐            ┌──────────────────────┐
│ Boot Services Code   │  → 回收 →  │ OS 可用内存           │
│ Boot Services Data   │  → 回收 →  │ OS 可用内存           │
│ Runtime Services     │  → 保留 →  │ Runtime Services     │
│ ACPI Tables          │  → 保留 →  │ ACPI Tables          │
│ Conventional Memory  │  → 接管 →  │ OS 可用内存           │
│ MMIO                 │  → 保留 →  │ MMIO                 │
└──────────────────────┘            └──────────────────────┘
```

> 💡 **经典面试题**："ExitBootServices 之后，固件还在运行吗？"
> 答：Boot Services 不在了，但 **Runtime Services 还活着**。操作系统随时可能调用 `GetVariable()`、`GetTime()` 等 Runtime 函数。

---

## 七、PEI 阶段的内存管理：穷人版

DXE 阶段的内存管理很"正规"，但 PEI 阶段就简陋多了：

```c
// PEI 阶段分配内存（只能分配，不能释放！）
EFI_STATUS
(*PeiAllocatePages)(
    IN CONST EFI_PEI_SERVICES  **PeiServices,
    IN EFI_MEMORY_TYPE         MemoryType,
    IN UINTN                   Pages,
    OUT EFI_PHYSICAL_ADDRESS   *Memory
);
```

PEI 阶段的内存分配有个显著特点：**只能分配，不能释放**。

为什么？因为 PEI 的内存分配器极其简单——就是一个指针往前推。分配 N 页，指针就往后移 N 页，不支持回收。反正 PEI 阶段很短暂，分配的内存也不多，等到 DXE 阶段接管后，会重新整理所有内存。

```
CAR 内存布局（简化）：
┌─────────────────────────┐  高地址
│       Stack（向下增长）    │
├─────────────────────────┤
│       空闲区域            │  ← 这块越来越小
├─────────────────────────┤
│       堆分配（向上增长）   │
├─────────────────────────┤
│       PEI Core 数据      │
├─────────────────────────┤
│       HOB 列表           │
└─────────────────────────┘  低地址
```

> 打个比方：PEI 的内存管理就像在信用卡上刷一刷——只管花，不还款。到了 DXE 阶段，新任管家会把整个账本重新整理一遍。

---

## 八、常见疑问

### Q1：为什么不在固件里开虚拟内存？

成本太高，收益太低。固件运行时间很短（通常只有几秒），模块之间又不存在互相防护的需求（不像操作系统要隔离进程）。用物理地址直接寻址够用了。

> 不过 Runtime Services 的内存在 ExitBootServices 之后需要重新映射——操作系统会把它映射到自己的虚拟地址空间中，通过 `SetVirtualAddressMap()` 通知固件。

### Q2：DDR 没初始化时，Cache 那么小够用吗？

PEI 阶段对代码大小有严格约束。Intel FSP-M（负责内存初始化的固件模块）被优化到能在几百 KB 的 Cache 中运行。这也是为什么 PEI 阶段的代码风格很"抠门"——变量能少就少，栈能浅就浅。

### Q3：AllocatePool 和 AllocatePages 在底层是什么关系？

`AllocatePool` 底层调用的就是 `AllocatePages`。UEFI 维护了一个简单的内存池管理器——先用 `AllocatePages` 拿到一大块内存，然后在里面做子分配。类似于 C 库的 `malloc` 底层调 `mmap/sbrk`。

---

## 九、总结

```
UEFI 内存管理的演进：

SEC 阶段 ─→ 只有寄存器，靠汇编活着
    │
    ▼
PEI 阶段 ─→ Cache As RAM（几百 KB）
    │        只能分配不能释放
    │        核心任务：把 DDR 内存初始化好
    │
    ▼
DXE 阶段 ─→ 真正的 DDR 内存（几 GB）
    │        AllocatePool / AllocatePages
    │        完整的内存类型系统
    │
    ▼
ExitBootServices ─→ 移交给操作系统
             Boot Services 内存被回收
             Runtime Services 内存保留
```

| 你以为的固件内存管理 | 实际的固件内存管理 |
|-------------------|-------------------|
| 上电就有内存 | 上电没有内存，得自己初始化 |
| malloc / free | AllocatePool / FreePool |
| 虚拟内存 | 直接用物理地址 |
| 所有内存归 OS 管 | 有些内存 OS 不能碰（Runtime）|

---

下一篇：**#018 什么是 UEFI 变量？它和注册表有什么像？**——固件也有"持久化存储"，你的启动顺序、安全启动密钥、甚至 BIOS 设置界面的每一项配置，都存在 UEFI 变量里。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
