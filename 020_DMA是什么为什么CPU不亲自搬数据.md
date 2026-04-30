# DMA 是什么？为什么 CPU 不亲自搬数据

> 🔥 UEFI/BSP 开发系列第 020 篇 | 难度：⭐⭐ 进阶入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们聊了 UEFI 的事件和定时器——固件里的异步机制。

今天要进入一个硬核但超级实用的话题——**DMA（Direct Memory Access，直接内存访问）**。

你可能在调试 BSP 的时候见过这些场景：

> "UFS 读不出数据，卡在 DMA 传输" > "PCIe 设备写内存写到了错误地址" > "开了 IOMMU 之后设备就挂了"

这些 bug 的根源，90% 跟 DMA 有关。所以搞 BSP 的人如果不懂 DMA，就像做饭不知道火候——迟早翻车。

---

## 一、先搞懂一个问题：CPU 为什么不自己搬数据？

假设你要从 SSD 读 1GB 数据到内存。

### 方案一：CPU 亲自搬（PIO 模式）

```
CPU 循环做这件事：
  1. 发命令给 SSD 控制器："给我一个字节"
  2. 等 SSD 准备好
  3. 从 SSD 的数据寄存器读一个字节
  4. 写入内存
  5. 回到第 1 步，重复 10 亿次（1GB）
```

**问题**：CPU 被完全占用，什么别的事都干不了。就像让公司 CEO 亲自搬砖——能搬，但太浪费了。

### 方案二：找个搬运工（DMA 模式）

```
CPU 只做一件事：
  1. 告诉 DMA 控制器："从 SSD 读 1GB 到内存地址 0x80000000"
  2. CPU 去忙别的事
  3. DMA 控制器自己搬数据（设备 ↔ 内存，不经过 CPU）
  4. 搬完了给 CPU 发个中断："老板，数据到了"
```

**效果**：CPU 只花了一点点时间发命令，剩下的时间可以干别的。这就是 DMA 的核心价值——**让 CPU 从繁重的数据搬运中解放出来**。

> 💡 一句话理解 DMA：CPU 是老板，DMA 控制器是搬运工。老板只需要说"搬什么、从哪搬到哪"，然后就可以去开会了。

---

## 二、DMA 是怎么工作的？

### 2.1 基本流程

```
┌────────┐         ┌──────────────┐         ┌──────────┐
│  CPU   │──①──→│ DMA 控制器    │──③──→│  内存    │
│        │        │              │         │          │
│        │←─④──│ (中断通知)    │←─②──│  设备    │
└────────┘         └──────────────┘         └──────────┘

① CPU 配置 DMA：源地址、目的地址、传输长度、方向
② 设备数据通过 DMA 控制器写入/读取内存
③ 数据直接在设备和内存之间传输，不经过 CPU
④ 传输完成，DMA 控制器触发中断通知 CPU
```

### 2.2 三种 DMA 传输方向

| 方向 | 说明 | 典型场景 |
|------|------|----------|
| **设备 → 内存** | 从设备读数据 | 从 SSD 读文件、网卡收包 |
| **内存 → 设备** | 往设备写数据 | 写数据到 SSD、网卡发包 |
| **内存 ↔ 内存** | 内存之间搬数据 | GPU 拷贝 framebuffer |

### 2.3 谁来当"搬运工"？

DMA 控制器有两种形态：

| 类型 | 说明 | 举例 |
|------|------|------|
| **集中式 DMA 控制器** | SoC 上的通用 DMA 引擎 | Intel 8237A、ARM PL330 |
| **设备内置 DMA** | 设备自带 DMA 引擎 | PCIe 设备、USB xHCI、UFS 控制器 |

现代 SoC 上几乎所有高速设备都**内置 DMA 引擎**。高通平台上的 UFS 控制器、PCIe 控制器、USB 控制器，都自带 DMA 能力——不需要额外的 DMA 控制器。

---

## 三、CPU 视角和设备视角：地址不一样！

这是 DMA 最容易踩坑的地方，也是 BSP 调试中最常见的问题之一。

### 3.1 两种地址

```
CPU 看到的地址（虚拟地址/物理地址）：
  0x80000000 ──→ 这块内存

设备看到的地址（总线地址/DMA 地址）：
  0x80000000 ──→ 可能不是同一块内存！
```

为什么会不一样？因为中间可能有**地址转换**：

```
                    CPU 地址空间
                  ┌──────────────┐
                  │  0x00000000  │
  CPU ──────────→│  0x80000000  │←── 物理内存
                  │  0xFFFFFFFF  │
                  └──────────────┘

                    设备地址空间
                  ┌──────────────┐
                  │  0x00000000  │
  设备 ─────────→│  0x00000000  │←── 同一块物理内存
                  │  0xFFFFFFFF  │     但设备看到的地址可能是 0x00000000！
                  └──────────────┘
```

### 3.2 为什么地址会不一样？

| 原因 | 说明 |
|------|------|
| **IOMMU/SMMU** | 设备访问内存需要经过地址转换（后面的文章专门讲） |
| **物理地址偏移** | 有些 SoC 的 DRAM 物理起始地址不是 0 |
| **32位设备** | 设备只能寻址 4GB 以下，但内存可能在 4GB 以上 |
| **地址重映射** | 一些平台固件会重映射 DMA 地址 |

### 3.3 经典 Bug 案例

```
// ❌ 错误：直接把 CPU 的物理地址给设备做 DMA
UfsController->DmaAddress = (UINT64)Buffer;  // Buffer 是 CPU 分配的

// ✅ 正确：通过 PciIo 或 DMA 映射接口获取设备可用的地址
Status = PciIo->Map(PciIo, ..., Buffer, &DeviceAddress, &Mapping);
UfsController->DmaAddress = DeviceAddress;   // 这才是设备能用的地址
```

> 🐛 **BSP 调试经验**：如果设备 DMA 后读到的数据全是 0 或者乱码，第一件事检查 DMA 地址是不是对的。这个 bug 能浪费你一整天。

---

## 四、DMA 的三种模式

### 4.1 Block DMA（块传输）

最基础的方式——一次搬一大块连续数据。

```
CPU 告诉 DMA：
  源地址：0x80000000
  目的地址：设备的 FIFO 寄存器
  大小：4096 字节
  方向：内存 → 设备
```

### 4.2 Scatter-Gather DMA（分散聚集）

数据在内存中不连续？没关系，给 DMA 控制器一个**描述符链表**，它会自动按列表搬运。

```
描述符链表（Descriptor List）：
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ 地址: 0x80000000 │──→│ 地址: 0x80010000 │──→│ 地址: 0x80030000 │
│ 长度: 4KB        │    │ 长度: 8KB        │    │ 长度: 2KB        │
│ 下一个: ──────────│    │ 下一个: ──────────│    │ 下一个: NULL     │
└──────────────────┘    └──────────────────┘    └──────────────────┘
```

**为什么需要 Scatter-Gather？**

操作系统/固件分配的内存往往是不连续的（尤其是大块内存），如果没有 Scatter-Gather，就要求数据必须放在连续的物理内存中，这在碎片化的内存环境里很难做到。

> 💡 Scatter-Gather 就像外卖骑手一次性取多个订单——虽然餐厅分散在各处，但骑手按地址列表挨个取，最终一起送达。

### 4.3 Coherent DMA vs Streaming DMA

| 类型 | 说明 | 使用场景 |
|------|------|----------|
| **Coherent DMA** | 分配的内存保证 CPU 和设备**看到的数据一致**（缓存一致性） | 设备描述符、环形缓冲区 |
| **Streaming DMA** | 分配的内存需要**手动刷缓存**来保证一致性 | 大数据块传输（性能更好） |

为什么有这个区别？因为 **CPU 有缓存（Cache）**！

```
问题场景：
  1. CPU 写数据到内存地址 0x80000000（数据在 Cache 里）
  2. 设备通过 DMA 从 0x80000000 读数据
  3. 设备读到的是内存里的旧数据，不是 Cache 里的新数据！

解决方案：
  ① Coherent DMA：硬件保证缓存一致性（通过 CCI/CHI 互联）
  ② Streaming DMA：软件在 DMA 前手动 Flush Cache
```

> 🐛 **BSP 调试经验**：如果 DMA 数据偶尔出错（不是每次），90% 是缓存一致性问题。ARM 平台尤其常见，因为不是所有 ARM 设备都接在 Cache Coherent Interconnect 上。

---

## 五、UEFI 中的 DMA

UEFI 固件也要做 DMA！因为固件阶段就要：

- 从 UFS/NVMe 读取启动文件
- 通过 USB 接收键盘输入
- 通过网卡进行 PXE 网络启动
- 通过 GPU 显示 Logo

### 5.1 UEFI 阶段 DMA 的特殊性

| 特点 | 说明 |
|------|------|
| **没有操作系统** | 不能用 Linux 的 `dma_alloc_coherent()` |
| **内存管理简单** | 通常用 1:1 映射（虚拟地址 = 物理地址） |
| **没有完整的 IOMMU 驱动** | 很多平台在 UEFI 阶段不开 IOMMU |
| **调试手段有限** | 主要靠串口打印 |

### 5.2 UEFI 中 DMA 相关的 API

```c
// 方法一：通过 PciIo Protocol 做 DMA
EFI_PCI_IO_PROTOCOL *PciIo;

// Map：把 CPU 地址转换成设备 DMA 可用的地址
Status = PciIo->Map(
  PciIo,
  EfiPciIoOperationBusMasterRead,  // 设备读内存
  HostAddress,                      // CPU 地址
  &NumberOfBytes,                   // 传输长度
  &DeviceAddress,                   // 输出：设备 DMA 地址
  &Mapping                          // 输出：映射句柄
);

// 告诉设备用 DeviceAddress 做 DMA
WriteDeviceRegister(DMA_ADDR_REG, DeviceAddress);
WriteDeviceRegister(DMA_LEN_REG, NumberOfBytes);
WriteDeviceRegister(DMA_START_REG, 1);

// DMA 完成后 Unmap
PciIo->Unmap(PciIo, Mapping);

// 方法二：直接分配 DMA 可用的内存
Status = PciIo->AllocateBuffer(
  PciIo,
  AllocateAnyPages,
  EfiBootServicesData,
  Pages,
  &HostAddress,
  0   // Attributes
);
```

### 5.3 为什么要用 Map/Unmap 而不是直接用地址？

```
                    不用 Map（可能出 bug）
CPU 分配内存 → 直接把物理地址给设备 → 设备 DMA → 💥 地址可能无效

                    用 Map（正确方式）
CPU 分配内存 → PciIo->Map() → 得到设备 DMA 地址 → 设备 DMA → ✅ 
               ↑ 这一步会处理：
               ├── 地址转换（如果有 IOMMU）
               ├── 缓存刷新（如果需要）
               └── Bounce Buffer（如果设备寻址受限）
```

---

## 六、Bounce Buffer：32 位设备的救星

有些老设备只能寻址 4GB 以下的内存（32 位 DMA 地址），但你的内存可能分配在 4GB 以上。怎么办？

```
问题：
  设备只能 DMA 到 0x00000000 ~ 0xFFFFFFFF（4GB）
  但数据在 0x1_80000000（6GB 位置）

解决方案：Bounce Buffer
  1. 在 4GB 以下分配一块临时缓冲区（Bounce Buffer）
  2. 把数据从 6GB 位置复制到 Bounce Buffer
  3. 设备 DMA 到 Bounce Buffer
  4. DMA 完成后把数据从 Bounce Buffer 复制回 6GB 位置
```

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  原始数据    │──复制──→│ Bounce Buffer│←──DMA──→│    设备      │
│  (>4GB)      │         │   (<4GB)     │         │  (32位DMA)   │
└──────────────┘         └──────────────┘         └──────────────┘
```

> 💡 Bounce Buffer 就像翻译——两个人语言不通（地址空间不一样），中间放个翻译（Bounce Buffer）就解决了。性能会差一点，但至少能工作。

UEFI 的 `PciIo->Map()` 会自动处理 Bounce Buffer——如果它发现设备寻址受限，就会自动分配 Bounce Buffer。这也是为什么一定要用 Map/Unmap 而不是自己直接传地址。

---

## 七、DMA 和安全：为什么 DMA 能被用来攻击？

DMA 的强大之处——设备可以直接读写内存，不需要 CPU 参与——同时也是它的安全隐患。

### 7.1 DMA 攻击

```
攻击场景：
  1. 攻击者插入一个恶意 PCIe/Thunderbolt/FireWire 设备
  2. 设备通过 DMA 直接读取内存中的敏感数据（密码、密钥）
  3. 或者通过 DMA 修改内存中的代码（跳过密码验证）
  4. CPU 全程不知情——因为 DMA 不需要 CPU 参与！
```

### 7.2 防御手段

| 防御 | 原理 | 对应固件工作 |
|------|------|-------------|
| **IOMMU/SMMU** | 给每个设备一个独立的地址空间，限制设备能访问的内存范围 | 固件阶段提前启用 IOMMU |
| **Kernel DMA Protection** | Windows/Linux 启动时锁定 IOMMU，防止新设备的 DMA 攻击 | 需要固件和 OS 配合 |
| **Thunderbolt 安全级别** | 限制哪些 Thunderbolt 设备可以使用 DMA | BIOS 设置选项 |

> 下一篇（#022 IOMMU/SMMU）会详细讲防御机制。

---

## 八、DMA 在不同平台上的实现

| 平台 | DMA 控制器 | 特点 |
|------|-----------|------|
| **高通 (ARM)** | QUP DMA / BAM (Bus Access Manager) / GPI | BAM 是高通私有的 DMA 引擎，GPI 是新一代 |
| **Intel (x86)** | 设备自带 + IOAT (Crystal Beach DMA) | IOAT 用于服务器高性能数据搬运 |
| **AMD (x86)** | 设备自带 + AMD DMA Engine | 类似 Intel |
| **瑞芯微 (ARM)** | 通用 PL330 + 设备自带 | PL330 是 ARM 公版 DMA 控制器 |
| **飞腾 (ARM Server)** | 设备自带 + SMMU | 服务器级别，完整 IOMMU 支持 |

### 高通 BAM 特别说明

在高通平台做 BSP，一定会遇到 BAM（Bus Access Manager）。它是高通自己设计的 DMA 引擎，几乎所有高通外设（UART、SPI、I2C、UFS、USB）都通过 BAM 做数据传输。

```
高通外设的数据通路：
  UART/SPI/I2C 控制器 ←→ BAM DMA 引擎 ←→ 系统内存
                                ↑
                          BAM 管理多个 Pipe（管道）
                          每个 Pipe 有自己的描述符环
```

---

## 九、常见面试题

### Q1：DMA 和 CPU 同时访问同一块内存会怎样？

数据不一致。这就是缓存一致性问题。解决方案：
- 硬件层面：使用 Cache Coherent Interconnect（如 ARM CCI/CHI）
- 软件层面：DMA 前 Flush Cache，DMA 后 Invalidate Cache

### Q2：为什么 UEFI 阶段的 DMA 问题特别多？

因为 UEFI 阶段：
1. 内存管理器比 OS 简单，Cache 配置可能不完善
2. 很多平台没有启用 IOMMU
3. 设备初始化不完整，DMA 配置容易遗漏
4. 调试手段少，DMA 问题难以定位

### Q3：Scatter-Gather 和连续 DMA 哪个性能好？

Scatter-Gather 有额外开销（读描述符链表），但在内存碎片化严重时是唯一选择。在内存充足且连续的情况下，单块连续 DMA 性能更好。

### Q4：什么是 "DMA Zone"？

在 Linux 中，DMA Zone 是 16MB 以下的内存区域，专门留给只能做 24 位地址 DMA 的老设备（ISA 总线）。在 UEFI 中没有这个概念，但 `PciIo->Map()` 会通过 Bounce Buffer 解决类似问题。

---

## 十、总结

```
DMA 知识地图：

为什么要 DMA？
  → CPU 太贵了，数据搬运交给专门的硬件

DMA 怎么工作？
  ├── Block DMA：搬连续数据块
  ├── Scatter-Gather：搬不连续数据
  └── CPU 只需发命令 + 等中断

地址问题（最大坑！）
  ├── CPU 地址 ≠ 设备 DMA 地址
  ├── 必须用 Map/Unmap 做地址转换
  └── 32 位设备需要 Bounce Buffer

缓存一致性（第二大坑！）
  ├── Coherent DMA：硬件保证
  ├── Streaming DMA：软件刷缓存
  └── ARM 平台尤其容易踩坑

UEFI 中的 DMA
  ├── PciIo->Map() / Unmap() / AllocateBuffer()
  └── UEFI 阶段 DMA 问题多，调试手段少

安全隐患
  ├── 恶意设备通过 DMA 攻击内存
  └── 防御靠 IOMMU（下篇详讲）
```

| 你以为的 DMA | 实际的 DMA |
|-------------|-----------|
| 只是个数据搬运 | CPU 不参与的高速数据通道 |
| 地址直接用就行 | CPU 地址和设备 DMA 地址可能不一样 |
| 跟缓存没关系 | Cache 一致性是最大的坑 |
| 安全无所谓 | 恶意设备可以通过 DMA 偷数据/改内存 |

---

下一篇：**#021 UEFI 中的 DMA 操作和 PciIo 协议**——深入 UEFI 代码层面，看看 `PciIo->Map()`、`PciIo->Unmap()`、`PciIo->AllocateBuffer()` 到底做了什么，以及 UEFI 固件中常见的 DMA 编程模式。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
