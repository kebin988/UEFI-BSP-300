

# UEFI 中的 DMA 操作和 PciIo 协议

> 🔥 UEFI/BSP 开发系列第 021 篇 | 难度：⭐⭐⭐ 进阶
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们从原理层面搞清楚了 DMA——CPU 把数据搬运的活外包给了专门的硬件。

但原理归原理，代码归代码。今天要回答一个实际问题：

> "在 UEFI 固件开发中，DMA 到底怎么写代码？用哪些 API？有什么坑？"

如果你写过 Linux 内核驱动，你一定用过 `dma_alloc_coherent()`、`dma_map_single()` 这些 API。在 UEFI 里，对应的就是 **`EFI_PCI_IO_PROTOCOL`** 里的 DMA 操作接口。

今天就把它拆开讲清楚。

---

## 一、UEFI DMA 的核心接口：EFI_PCI_IO_PROTOCOL

在 UEFI 里，大多数需要做 DMA 的设备都是 PCI/PCIe 设备（或者挂在 PCI 总线模型下的设备）。所以 DMA 操作被封装在了 `EFI_PCI_IO_PROTOCOL` 里。

### 1.1 PciIo 中 DMA 相关的三个函数

```c
// EFI_PCI_IO_PROTOCOL 的 DMA 相关成员
struct _EFI_PCI_IO_PROTOCOL {
  // ... 其他成员（Pci.Read, Mem.Read, Io.Read 等）
  
  EFI_PCI_IO_PROTOCOL_MAP            Map;            // 映射 DMA 地址
  EFI_PCI_IO_PROTOCOL_UNMAP          Unmap;          // 取消映射
  EFI_PCI_IO_PROTOCOL_ALLOCATE_BUFFER AllocateBuffer; // 分配 DMA 内存
  EFI_PCI_IO_PROTOCOL_FREE_BUFFER    FreeBuffer;     // 释放 DMA 内存
  EFI_PCI_IO_PROTOCOL_FLUSH          Flush;          // 刷新 DMA 缓存
  
  // ...
};
```

就这五个函数，搞定 UEFI 中 90% 的 DMA 需求。

---

## 二、Map() 和 Unmap()——DMA 地址映射

### 2.1 为什么需要 Map？

上一篇讲过：CPU 地址和设备 DMA 地址可能不一样。`Map()` 就是干这个的——把 CPU 分配的内存地址转换成设备 DMA 可以使用的地址。

### 2.2 Map() 函数原型

```c
typedef
EFI_STATUS
(EFIAPI *EFI_PCI_IO_PROTOCOL_MAP)(
  IN  EFI_PCI_IO_PROTOCOL            *This,
  IN  EFI_PCI_IO_PROTOCOL_OPERATION  Operation,     // DMA 方向
  IN  VOID                           *HostAddress,   // CPU 端地址
  IN  OUT UINTN                      *NumberOfBytes,  // 传输大小
  OUT EFI_PHYSICAL_ADDRESS           *DeviceAddress,  // 输出：设备 DMA 地址
  OUT VOID                           **Mapping        // 输出：映射句柄
);
```

### 2.3 Operation 参数：DMA 方向

```c
typedef enum {
  EfiPciIoOperationBusMasterRead,       // 设备读内存（设备是 master，从内存读）
  EfiPciIoOperationBusMasterWrite,      // 设备写内存（设备是 master，向内存写）
  EfiPciIoOperationBusMasterCommonBuffer // 双向（设备和 CPU 都读写同一块内存）
} EFI_PCI_IO_PROTOCOL_OPERATION;
```

⚠️ **注意命名的视角**：这里的"Read"和"Write"是**站在设备（Bus Master）的角度说的**：

| Operation | 设备的动作 | 数据流向 | 典型场景 |
|-----------|-----------|---------|----------|
| `BusMasterRead` | 设备**读**内存 | 内存 → 设备 | 网卡发包、写数据到 SSD |
| `BusMasterWrite` | 设备**写**内存 | 设备 → 内存 | 网卡收包、从 SSD 读数据 |
| `BusMasterCommonBuffer` | 双向 | 内存 ↔ 设备 | 描述符环、命令队列 |

> 🐛 **经典踩坑**：把 Read/Write 的方向搞反了。你以为"读 SSD"应该用 `BusMasterRead`，但其实 SSD 读数据 = 设备往内存写 = `BusMasterWrite`。这个 bug 能让你调一下午。

### 2.4 完整的 Map/DMA/Unmap 流程

```c
EFI_STATUS DoDmaTransfer(
  EFI_PCI_IO_PROTOCOL  *PciIo,
  VOID                 *Buffer,
  UINTN                BufferSize
) {
  EFI_STATUS            Status;
  EFI_PHYSICAL_ADDRESS  DeviceAddress;
  VOID                  *Mapping;
  UINTN                 NumberOfBytes = BufferSize;

  // ① Map：获取设备 DMA 地址
  Status = PciIo->Map(
    PciIo,
    EfiPciIoOperationBusMasterWrite,  // 设备写内存（从设备读数据到内存）
    Buffer,
    &NumberOfBytes,
    &DeviceAddress,
    &Mapping
  );
  if (EFI_ERROR(Status)) {
    return Status;
  }

  // ⚠️ 检查：实际映射的大小可能小于请求的大小！
  if (NumberOfBytes < BufferSize) {
    // 需要分多次传输
    PciIo->Unmap(PciIo, Mapping);
    return EFI_BAD_BUFFER_SIZE;
  }

  // ② 配置设备 DMA 寄存器
  //    告诉设备：往 DeviceAddress 这个地址写数据
  WriteDeviceDmaAddress(DeviceAddress);
  WriteDeviceDmaLength(NumberOfBytes);
  TriggerDeviceDma();

  // ③ 等待 DMA 完成
  WaitForDmaComplete();

  // ④ Flush：确保所有 DMA 写入对 CPU 可见
  PciIo->Flush(PciIo);

  // ⑤ Unmap：释放映射
  Status = PciIo->Unmap(PciIo, Mapping);

  return Status;
}
```

### 2.5 Map() 内部到底做了什么？

```
PciIo->Map() 的内部逻辑：

1. 检查 HostAddress 是否在设备 DMA 可达范围内
   ├── 可达 → DeviceAddress = HostAddress（直接映射）
   └── 不可达（如地址 > 4GB 但设备只支持 32位 DMA）
       ├── 分配 Bounce Buffer（< 4GB）
       ├── 如果 Operation 是 BusMasterRead（设备读内存）
       │   └── 把数据从 HostAddress 复制到 Bounce Buffer
       └── DeviceAddress = Bounce Buffer 的地址

2. 如果平台有 IOMMU
   ├── 在 IOMMU 页表中建立映射
   └── DeviceAddress = IOMMU 映射后的设备虚拟地址

3. 如果需要缓存操作
   └── Flush/Invalidate Cache

4. 返回 DeviceAddress 和 Mapping 句柄
```

### 2.6 Unmap() 内部做了什么？

```
PciIo->Unmap() 的内部逻辑：

1. 如果用了 Bounce Buffer 且 Operation 是 BusMasterWrite（设备写内存）
   └── 把数据从 Bounce Buffer 复制回原始 HostAddress

2. 如果平台有 IOMMU
   └── 删除 IOMMU 页表中的映射

3. 释放 Bounce Buffer（如果有）

4. Invalidate Cache（如果需要）
```

---

## 三、AllocateBuffer() 和 FreeBuffer()——分配 DMA 专用内存

### 3.1 什么时候用 AllocateBuffer？

当你需要一块 CPU 和设备**共享的内存**（Common Buffer），比如：

- **描述符环（Descriptor Ring）**：CPU 写描述符，设备读描述符
- **命令队列（Command Queue）**：NVMe 的 Submission Queue / Completion Queue
- **状态寄存器区域**：设备把状态写到内存里，CPU 去读

这些内存需要 CPU 和设备**同时频繁访问**，不适合每次都 Map/Unmap。

### 3.2 函数原型

```c
typedef
EFI_STATUS
(EFIAPI *EFI_PCI_IO_PROTOCOL_ALLOCATE_BUFFER)(
  IN  EFI_PCI_IO_PROTOCOL  *This,
  IN  EFI_ALLOCATE_TYPE     Type,       // AllocateAnyPages / AllocateMaxAddress
  IN  EFI_MEMORY_TYPE       MemoryType, // EfiBootServicesData
  IN  UINTN                 Pages,      // 页数（4KB 每页）
  OUT VOID                  **HostAddress,
  IN  UINT64                Attributes  // EFI_PCI_ATTRIBUTE_MEMORY_WRITE_COMBINE 等
);
```

### 3.3 AllocateBuffer vs gBS->AllocatePages

你可能会问：直接用 `gBS->AllocatePages()` 分配内存不行吗？为什么要用 `PciIo->AllocateBuffer()`？

| | `gBS->AllocatePages()` | `PciIo->AllocateBuffer()` |
|---|---|---|
| **地址范围** | 随机分配，可能 > 4GB | 保证在设备 DMA 可达范围内 |
| **缓存属性** | 默认 Write-Back | 可指定 Write-Combine / Uncached |
| **对齐** | 4KB 对齐 | 4KB 对齐 |
| **适用场景** | 普通内存分配 | DMA 共享缓冲区 |

### 3.4 使用 AllocateBuffer + Map 的标准模式

```c
// 分配 Common Buffer 的标准写法
VOID                 *CommonBuffer;
EFI_PHYSICAL_ADDRESS  DeviceAddress;
VOID                  *Mapping;
UINTN                 NumberOfBytes = 4096;  // 1 页

// ① 分配 DMA 友好的内存
Status = PciIo->AllocateBuffer(
  PciIo,
  AllocateAnyPages,
  EfiBootServicesData,
  1,  // 1 页
  &CommonBuffer,
  0
);

// ② Map 成 Common Buffer 模式
Status = PciIo->Map(
  PciIo,
  EfiPciIoOperationBusMasterCommonBuffer,
  CommonBuffer,
  &NumberOfBytes,
  &DeviceAddress,
  &Mapping
);

// ③ 现在可以同时使用：
//    CPU 通过 CommonBuffer 指针读写
//    设备通过 DeviceAddress 做 DMA 读写

// ④ 用完后释放
PciIo->Unmap(PciIo, Mapping);
PciIo->FreeBuffer(PciIo, 1, CommonBuffer);
```

---

## 四、Flush()——确保 DMA 数据可见

### 4.1 为什么需要 Flush？

```
场景：设备通过 DMA 往内存写了数据

  设备 DMA 写入 → 数据到了内存（或者到了写缓冲区）
  CPU 读取       → 可能读到旧数据（因为 CPU Cache 里有旧值）

PciIo->Flush() 做的事：
  → 确保所有 PCI DMA 写入操作对 CPU 可见
  → 类似 Linux 的 wmb()（写内存屏障）+ Cache Invalidate
```

### 4.2 什么时候调用 Flush？

```c
// 典型流程
TriggerDeviceDma();           // 让设备开始 DMA
WaitForDmaComplete();         // 等 DMA 完成
PciIo->Flush(PciIo);         // ← 在读取 DMA 数据之前调用
ReadData(CommonBuffer);        // 现在读到的数据是最新的
```

> 🐛 **经典 Bug**：忘了调 Flush()，DMA 数据有时候对有时候错——因为有时候 Cache 里正好没有旧数据就对了，有时候有就错了。这种概率性 bug 是最难调的。

---

## 五、非 PCI 设备的 DMA：怎么办？

不是所有设备都是 PCI/PCIe。在 ARM SoC 上，很多设备（UART、I2C、SPI、UFS）直接挂在 SoC 内部总线上，没有 PciIo Protocol。

### 5.1 UEFI 的 IoMmu Protocol

UEFI 规范定义了 `EDKII_IOMMU_PROTOCOL`，专门处理非 PCI 设备的 DMA 地址映射：

```c
struct _EDKII_IOMMU_PROTOCOL {
  UINT64                       Revision;
  EDKII_IOMMU_SET_ATTRIBUTE    SetAttribute;
  EDKII_IOMMU_MAP              Map;
  EDKII_IOMMU_UNMAP            Unmap;
  EDKII_IOMMU_ALLOCATE_BUFFER  AllocateBuffer;
  EDKII_IOMMU_FREE_BUFFER      FreeBuffer;
};
```

看起来和 PciIo 的 DMA 接口几乎一样——因为逻辑是一样的，只是不依赖 PCI 总线。

### 5.2 直接操作寄存器

在很多嵌入式 / ARM BSP 场景中，如果没有 IOMMU 也没有 PciIo，驱动会直接操作设备的 DMA 寄存器：

```c
// 直接设置设备 DMA 地址（简化示例）
// ⚠️ 只在确认物理地址 = DMA 地址的平台上可以这样做
VOID *Buffer;
EFI_PHYSICAL_ADDRESS PhysicalAddress;

// 分配对齐的内存
gBS->AllocatePages(
  AllocateAnyPages,
  EfiBootServicesData,
  Pages,
  &PhysicalAddress
);

// 在没有 IOMMU 的平台上，物理地址就是 DMA 地址
Buffer = (VOID *)(UINTN)PhysicalAddress;

// 刷缓存（ARM 平台关键步骤！）
WriteBackInvalidateDataCache();   // 或者用 ArmDataSynchronizationBarrier()

// 直接把物理地址写入设备 DMA 寄存器
MmioWrite32(UFS_BASE + DMA_ADDR_REG, (UINT32)PhysicalAddress);
MmioWrite32(UFS_BASE + DMA_ADDR_HI_REG, (UINT32)(PhysicalAddress >> 32));
MmioWrite32(UFS_BASE + DMA_LEN_REG, BufferSize);
MmioWrite32(UFS_BASE + DMA_CTRL_REG, DMA_START);
```

> ⚠️ 这种"裸写"方式只在**没有 IOMMU 且物理地址等于 DMA 地址**的平台上才安全。在有 IOMMU 的平台上这样做会导致 DMA 失败。

---

## 六、DMA 编程的三大坑（附解决方案）

### 坑1：忘记 Map/Unmap

```c
// ❌ 错误
UfsHc->DmaAddress = (UINT64)Buffer;  // 直接用 CPU 地址
UfsHc->DmaStart = 1;

// ✅ 正确
PciIo->Map(PciIo, ..., Buffer, &DeviceAddress, &Mapping);
UfsHc->DmaAddress = DeviceAddress;
UfsHc->DmaStart = 1;
// ... DMA 完成后
PciIo->Unmap(PciIo, Mapping);
```

### 坑2：DMA 方向搞反

```c
// ❌ 错误：从设备读数据，但用了 BusMasterRead（设备读内存）
PciIo->Map(PciIo, EfiPciIoOperationBusMasterRead, ...);

// ✅ 正确：从设备读数据 = 设备往内存写 = BusMasterWrite
PciIo->Map(PciIo, EfiPciIoOperationBusMasterWrite, ...);
```

### 坑3：Common Buffer 忘记用 AllocateBuffer

```c
// ❌ 错误：用普通内存做 Common Buffer
gBS->AllocatePages(AllocateAnyPages, EfiBootServicesData, Pages, &Buffer);
// 这块内存可能在 4GB 以上，32 位 DMA 设备访问不了

// ✅ 正确：用 PciIo->AllocateBuffer
PciIo->AllocateBuffer(PciIo, AllocateAnyPages, EfiBootServicesData, Pages, &Buffer, 0);
// AllocateBuffer 保证内存在设备 DMA 可达范围内
```

---

## 七、实战案例：UFS 控制器的 DMA 流程

高通平台的 UFS 控制器（就在你的 RB3Gen2 开发板上）使用 UFSHCI（UFS Host Controller Interface）标准，其 DMA 流程是个很好的学习案例：

```
UFS 读数据的 DMA 流程：

1. 驱动分配 UTRD（UFS Transfer Request Descriptor）← Common Buffer
   └── PciIo->AllocateBuffer() + Map(CommonBuffer)

2. 驱动分配数据缓冲区
   └── PciIo->Map(BusMasterWrite)  // 设备写内存

3. 填写 UTRD（在 Common Buffer 中）
   ├── 命令类型：READ
   ├── 数据缓冲区的 DMA 地址（Map 返回的 DeviceAddress）
   └── 数据长度

4. 通知 UFS 控制器执行
   └── 写 UTRD 指针到 UFS 控制器寄存器

5. UFS 控制器通过 DMA 执行：
   ├── 读取 UTRD（从 Common Buffer）
   ├── 根据 UTRD 中的信息，从 UFS 设备读数据
   └── 通过 DMA 写入数据到数据缓冲区

6. 完成后 UFS 控制器触发中断

7. 驱动处理完成：
   ├── PciIo->Flush()
   ├── 检查 UTRD 中的完成状态
   ├── Unmap 数据缓冲区
   └── 数据可用
```

### 对应到 edk2-platforms 中的代码

如果你去看 `Silicon/Qualcomm/Drivers/QcomUfsHcDxe`（你 RB3Gen2 上的 UFS 驱动），里面就会看到类似的 DMA 操作。

---

## 八、DMA 操作对照表：UEFI vs Linux

如果你同时写 UEFI 驱动和 Linux 内核驱动，这张对照表能帮你快速切换：

| 操作 | UEFI (PciIo) | Linux Kernel |
|------|-------------|-------------|
| **映射 DMA 地址** | `PciIo->Map()` | `dma_map_single()` |
| **取消映射** | `PciIo->Unmap()` | `dma_unmap_single()` |
| **分配 DMA 内存** | `PciIo->AllocateBuffer()` + `Map(CommonBuffer)` | `dma_alloc_coherent()` |
| **释放 DMA 内存** | `PciIo->Unmap()` + `FreeBuffer()` | `dma_free_coherent()` |
| **刷 DMA 缓存** | `PciIo->Flush()` | `dma_sync_single_for_cpu()` |
| **Scatter-Gather** | 多次 Map 或 自己管理描述符 | `dma_map_sg()` |
| **DMA 方向** | `EfiPciIoOperationBusMasterRead/Write` | `DMA_TO_DEVICE / DMA_FROM_DEVICE` |

### 命名陷阱对比

| | UEFI | Linux | 实际含义 |
|---|---|---|---|
| **数据从内存到设备** | `BusMasterRead` | `DMA_TO_DEVICE` | 设备读内存 → 数据去设备 |
| **数据从设备到内存** | `BusMasterWrite` | `DMA_FROM_DEVICE` | 设备写内存 → 数据来自设备 |
| **双向** | `BusMasterCommonBuffer` | `DMA_BIDIRECTIONAL` | CPU 和设备都读写 |

> 💡 UEFI 用的是**设备视角**（Bus Master Read/Write），Linux 用的是**数据流向视角**（TO_DEVICE/FROM_DEVICE）。同一件事，两种说法。

---

## 九、面试题

### Q1：PciIo->Map() 什么情况下会分配 Bounce Buffer？

当设备 DMA 地址受限（如只支持 32 位 DMA）而 HostAddress 超出设备寻址范围时。具体来说：
- 设备只支持 4GB 以下的 DMA 地址
- 但 HostAddress 在 4GB 以上
- Map() 会在 4GB 以下分配 Bounce Buffer，并在 Unmap() 时复制数据回去

### Q2：Common Buffer 和 Streaming DMA 的区别？

- **Common Buffer**（`BusMasterCommonBuffer`）：CPU 和设备同时频繁访问同一块内存，Map 一次长期使用。用 `AllocateBuffer()` 分配。
- **Streaming DMA**（`BusMasterRead/Write`）：单向、一次性传输。每次传输都 Map/Unmap。

### Q3：如果忘了调 Unmap() 会怎样？

- 如果使用了 Bounce Buffer：数据不会被复制回原始 Buffer（`BusMasterWrite` 场景下丢数据）
- 如果使用了 IOMMU：IOMMU 映射不会被释放，可能导致映射表满
- 内存泄漏（Bounce Buffer 不会被释放）

---

## 十、总结

```
UEFI DMA 编程知识地图：

核心 API（PciIo Protocol）
  ├── Map()           → CPU 地址转设备 DMA 地址
  ├── Unmap()         → 释放映射 + 回写数据
  ├── AllocateBuffer()→ 分配 DMA 友好内存
  ├── FreeBuffer()    → 释放 DMA 内存
  └── Flush()         → 确保 DMA 数据对 CPU 可见

三种 DMA 操作类型
  ├── BusMasterRead      → 设备读内存（内存→设备）
  ├── BusMasterWrite     → 设备写内存（设备→内存）
  └── BusMasterCommonBuffer → 双向共享

标准编程模式
  ├── 单向传输：Map → 配置设备 → DMA → Flush → Unmap
  └── 共享缓冲区：AllocateBuffer → Map(CommonBuffer) → 长期使用 → Unmap → Free

三大坑
  ├── ❌ 不 Map 直接用 CPU 地址
  ├── ❌ DMA 方向搞反（Read vs Write）
  └── ❌ 忘记 Flush / Unmap
```

| 你以为的 UEFI DMA | 实际的 UEFI DMA |
|-------------------|----------------|
| 直接把地址给设备就行 | 必须通过 Map() 转换 |
| Read 就是读，Write 就是写 | Read/Write 是站在**设备**角度说的 |
| 分配普通内存就能做 DMA | 需要用 AllocateBuffer 确保 DMA 可达 |
| Unmap 只是释放资源 | Unmap 还负责回写 Bounce Buffer 数据 |

---

下一篇：**#022 IOMMU/SMMU：给设备装一个 MMU**——CPU 有 MMU 做地址翻译和内存保护，设备凭什么没有？IOMMU 就是给设备装的 MMU，它是 DMA 安全的守门人。我们来看看它是怎么工作的。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
