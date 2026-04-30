# PPI（PEI-to-PEI Interface）：PEI 阶段的通信机制

> 🔥 UEFI/BSP 开发系列第 043 篇 | 难度：⭐⭐⭐ 中级
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

DXE 阶段有 Protocol（用来在驱动之间通信），PEI 阶段也需要类似机制——**PPI (PEIM-to-PEIM Interface)**。

PPI 是 Protocol 的"青春版"——功能类似但更简单轻量，因为 PEI 阶段资源有限，经不起 Protocol 那样的完整 Handle Database。

---

## 一、PPI vs Protocol：对比理解

| | PPI (PEI) | Protocol (DXE) |
|--|-----------|----------------|
| 全称 | PEIM-to-PEIM Interface | UEFI Protocol |
| 标识 | GUID | GUID |
| 本质 | 函数指针结构体 | 函数指针结构体 |
| 注册方式 | InstallPpi | InstallProtocolInterface |
| 查找方式 | LocatePpi | LocateProtocol |
| Handle 支持 | ❌ 没有 Handle 概念 | ✅ 挂在 Handle 上 |
| 多实例 | 支持（按索引查找） | 支持（通过 Handle） |
| 数据库 | 简单链表 | 完整 Handle Database |
| 生命周期 | PEI 阶段结束即消亡 | Boot Services 结束消亡 |

**简单理解**：PPI 就是 Protocol 的精简版——没有 Handle，直接用 GUID 查找。

---

## 二、PPI 的定义

PPI 和 Protocol 一样，本质是一个结构体（通常包含函数指针）：

```c
// 定义一个 PPI —— 温度传感器接口
typedef struct _MY_TEMP_SENSOR_PPI {
    EFI_STATUS (EFIAPI *ReadTemperature)(
        IN  MY_TEMP_SENSOR_PPI  *This,
        OUT INT32               *Temperature
    );
    EFI_STATUS (EFIAPI *GetSensorName)(
        IN  MY_TEMP_SENSOR_PPI  *This,
        OUT CHAR16              **Name
    );
} MY_TEMP_SENSOR_PPI;

// PPI 的 GUID
#define MY_TEMP_SENSOR_PPI_GUID \
  { 0x11111111, 0x2222, 0x3333, \
    { 0x44, 0x55, 0x66, 0x77, 0x88, 0x99, 0xAA, 0xBB } }

extern EFI_GUID gMyTempSensorPpiGuid;
```

不是所有 PPI 都有函数指针——有些 PPI 纯粹是"标志"，表示某件事已完成：

```c
// "内存已发现" PPI —— 没有函数，纯粹是一个信号
// 当这个 PPI 被安装时，表示 DRAM 初始化完成了
// 其他 PEIM 的 DEPEX 可以依赖它

// 定义（规范中）：
// gEfiPeiMemoryDiscoveredPpiGuid
// 无接口结构体，安装时 PPI pointer = NULL
```

---

## 三、安装 PPI

### 3.1 PPI Descriptor

安装 PPI 需要构造一个描述符：

```c
// PPI 描述符结构
typedef struct {
    UINTN    Flags;      // 标志位
    EFI_GUID *Guid;      // PPI 的 GUID
    VOID     *Ppi;       // PPI 接口指针（函数指针表）
} EFI_PEI_PPI_DESCRIPTOR;

// Flags 的值：
#define EFI_PEI_PPI_DESCRIPTOR_PPI            0x00000010
#define EFI_PEI_PPI_DESCRIPTOR_TERMINATE_LIST 0x80000000
```

### 3.2 安装示例

```c
// 温度传感器 PEIM 安装 PPI

// 实现函数
EFI_STATUS EFIAPI ReadTemp(MY_TEMP_SENSOR_PPI *This, INT32 *Temp) {
    *Temp = 42;  // 读硬件寄存器获取温度
    return EFI_SUCCESS;
}

EFI_STATUS EFIAPI GetName(MY_TEMP_SENSOR_PPI *This, CHAR16 **Name) {
    *Name = L"CPU Thermal Sensor";
    return EFI_SUCCESS;
}

// PPI 实例
MY_TEMP_SENSOR_PPI mTempSensorPpi = {
    .ReadTemperature = ReadTemp,
    .GetSensorName   = GetName
};

// PPI 描述符
EFI_PEI_PPI_DESCRIPTOR mTempSensorPpiDesc = {
    EFI_PEI_PPI_DESCRIPTOR_PPI | EFI_PEI_PPI_DESCRIPTOR_TERMINATE_LIST,
    &gMyTempSensorPpiGuid,
    &mTempSensorPpi
};

// PEIM 入口函数中安装
EFI_STATUS EFIAPI TempSensorPeimEntry(
    IN EFI_PEI_FILE_HANDLE FileHandle,
    IN CONST EFI_PEI_SERVICES **PeiServices
) {
    return (*PeiServices)->InstallPpi(PeiServices, &mTempSensorPpiDesc);
}
```

### 3.3 安装多个 PPI

```c
// 一次安装多个 PPI（数组形式）
EFI_PEI_PPI_DESCRIPTOR mPpiList[] = {
    {
        EFI_PEI_PPI_DESCRIPTOR_PPI,     // 非最后一个，不加 TERMINATE
        &gPpiAGuid,
        &mPpiA
    },
    {
        EFI_PEI_PPI_DESCRIPTOR_PPI | EFI_PEI_PPI_DESCRIPTOR_TERMINATE_LIST,
        &gPpiBGuid,                      // 最后一个，加 TERMINATE
        &mPpiB
    }
};

(*PeiServices)->InstallPpi(PeiServices, mPpiList);
```

---

## 四、查找和使用 PPI

```c
// 另一个 PEIM 想使用温度传感器
EFI_STATUS UseTemperatureSensor(CONST EFI_PEI_SERVICES **PeiServices)
{
    EFI_STATUS           Status;
    MY_TEMP_SENSOR_PPI   *TempSensor;
    INT32                Temperature;
    
    // 查找 PPI
    Status = (*PeiServices)->LocatePpi(
        PeiServices,
        &gMyTempSensorPpiGuid,   // 要找的 PPI 的 GUID
        0,                        // Instance（第几个实例，从 0 开始）
        NULL,                     // 输出：PPI Descriptor（不需要可传 NULL）
        (VOID **)&TempSensor     // 输出：PPI 指针
    );
    
    if (EFI_ERROR(Status)) {
        DEBUG((DEBUG_ERROR, "Temperature sensor PPI not found!\n"));
        return Status;
    }
    
    // 使用 PPI
    TempSensor->ReadTemperature(TempSensor, &Temperature);
    DEBUG((DEBUG_INFO, "CPU Temperature: %d C\n", Temperature));
    
    return EFI_SUCCESS;
}
```

### 查找多个实例

如果系统中有多个温度传感器 PPI（比如 CPU 和 PCH 各一个）：

```c
// 遍历所有实例
for (UINTN Index = 0; ; Index++) {
    Status = (*PeiServices)->LocatePpi(
        PeiServices,
        &gMyTempSensorPpiGuid,
        Index,          // 第 Index 个实例
        NULL,
        (VOID **)&TempSensor
    );
    
    if (EFI_ERROR(Status)) break;  // 没有更多实例了
    
    TempSensor->ReadTemperature(TempSensor, &Temp);
    DEBUG((DEBUG_INFO, "Sensor #%d: %d C\n", Index, Temp));
}
```

---

## 五、PPI 通知机制（Notify）

有时你想在某个 PPI 被安装时收到通知，而不是主动去轮询。这就是 **Notify PPI**：

```c
// 当内存初始化完成时，我想收到通知

// 通知回调函数
EFI_STATUS
EFIAPI
MemoryDiscoveredCallback(
    IN EFI_PEI_SERVICES          **PeiServices,
    IN EFI_PEI_NOTIFY_DESCRIPTOR *NotifyDescriptor,
    IN VOID                      *Ppi
)
{
    DEBUG((DEBUG_INFO, "Memory has been initialized! I can do more now.\n"));
    
    // 在这里执行需要内存才能做的操作
    // 比如分配大块内存、初始化数据结构等
    
    return EFI_SUCCESS;
}

// 通知描述符
EFI_PEI_NOTIFY_DESCRIPTOR mMemoryNotify = {
    EFI_PEI_PPI_DESCRIPTOR_NOTIFY_CALLBACK | 
    EFI_PEI_PPI_DESCRIPTOR_TERMINATE_LIST,
    &gEfiPeiMemoryDiscoveredPpiGuid,   // 关注这个 PPI
    MemoryDiscoveredCallback            // 回调函数
};

// 在 PEIM 入口中注册通知
EFI_STATUS EFIAPI MyPeimEntry(...) {
    (*PeiServices)->NotifyPpi(PeiServices, &mMemoryNotify);
    return EFI_SUCCESS;
}
```

**执行时机**：
- MyPeim 注册通知后立即返回
- 当 MemoryInit PEIM 安装 `gEfiPeiMemoryDiscoveredPpiGuid` 时
- PEI Core 检查通知列表，调用 `MemoryDiscoveredCallback`

```
时间线：
  MyPeim: NotifyPpi(MemoryDiscovered) → return
  ...
  MemoryInitPeim: InstallPpi(MemoryDiscovered)
  ...
  PEI Core: 发现通知匹配 → 调用 MemoryDiscoveredCallback
```

---

## 六、PEI 阶段的关键 PPI

| PPI GUID | 含义 | 谁安装 |
|----------|------|--------|
| `gEfiPeiMemoryDiscoveredPpiGuid` | DRAM 初始化完成 | Memory Init PEIM |
| `gEfiPeiMasterBootModePpiGuid` | Boot Mode 已确定 | Platform Init |
| `gEfiPeiResetPpiGuid` | 系统重置服务 | Platform PEIM |
| `gEfiPeiStallPpiGuid` | 微秒延时服务 | Timer PEIM |
| `gEfiPeiReadOnlyVariable2PpiGuid` | 读取 UEFI 变量 | Variable PEIM |
| `gEfiDxeIplPpiGuid` | DXE 加载器 | DXE IPL PEIM |
| `gEfiPeiCpuIoPpiGuid` | CPU IO 访问 | CPU IO PEIM |
| `gEfiPeiPciCfg2PpiGuid` | PCI 配置空间访问 | PCI PEIM |

---

## 七、PPI 的内部实现

PEI Core 用一个简单的数组/链表存储所有已安装的 PPI：

```c
// PEI Core 内部的 PPI 数据库（简化）
#define MAX_PPI_DESCRIPTORS  64

typedef struct {
    BOOLEAN                     Occupied;
    EFI_PEI_PPI_DESCRIPTOR     *PpiDescriptor;
} PPI_ENTRY;

typedef struct {
    PPI_ENTRY  PpiList[MAX_PPI_DESCRIPTORS];     // 已安装的 PPI
    UINTN      PpiListEnd;                        // 当前使用到的位置
    
    PPI_ENTRY  NotifyList[MAX_PPI_DESCRIPTORS];  // 通知列表
    UINTN      NotifyListEnd;
} PEI_PPI_DATABASE;
```

**LocatePpi 的实现就是遍历这个数组**：

```c
EFI_STATUS LocatePpi(GUID *Guid, UINTN Instance, ...) {
    UINTN Found = 0;
    for (UINTN i = 0; i < Database->PpiListEnd; i++) {
        if (CompareGuid(Database->PpiList[i].Guid, Guid)) {
            if (Found == Instance) {
                *Ppi = Database->PpiList[i].Ppi;
                return EFI_SUCCESS;
            }
            Found++;
        }
    }
    return EFI_NOT_FOUND;
}
```

简单粗暴，但在 PEI 阶段（几十个 PPI）完全够用。

---

## 八、PPI 的生命周期

```
PPI 的生命周期：

  安装 ────────────── 使用 ────────────── 消亡
  │                    │                    │
  PEIM A 调用          PEIM B/C 调用       PEI 阶段结束
  InstallPpi()         LocatePpi()          ↓
                                       跳转到 DXE
                                       PPI 数据库消失
                                       所有 PPI 不复存在
```

**关键**：PPI 只存在于 PEI 阶段。一旦跳转到 DXE，PPI 全部消失。如果需要在 PEI 和 DXE 之间传递信息，用 **HOB**。

---

## 九、ReInstallPpi：替换 PPI

有时需要替换一个已安装的 PPI（比如从默认实现升级到更完整的实现）：

```c
// 场景：Pre-Memory 阶段安装了一个简单版 PPI
// Post-Memory 后想替换成功能更全的版本

EFI_STATUS EFIAPI UpgradeAfterMemory(...)
{
    EFI_PEI_PPI_DESCRIPTOR *OldPpiDesc;
    MY_TEMP_SENSOR_PPI     *OldPpi;
    
    // 找到旧的
    (*PeiServices)->LocatePpi(
        PeiServices, &gMyTempSensorPpiGuid, 0, &OldPpiDesc, &OldPpi
    );
    
    // 替换为新的
    (*PeiServices)->ReInstallPpi(
        PeiServices,
        OldPpiDesc,           // 旧的描述符
        &mNewTempSensorDesc   // 新的描述符
    );
    
    return EFI_SUCCESS;
}
```

---

## 十、总结

```
PPI 核心要点：

  定义：GUID + 函数指针结构体（和 Protocol 一样）
  安装：InstallPpi（注册到 PEI Core 数据库）
  查找：LocatePpi（按 GUID + 实例号查找）
  通知：NotifyPpi（异步回调，PPI 安装时触发）
  生命：仅在 PEI 阶段存活
```

| 操作 | 函数 | 说明 |
|------|------|------|
| 安装 | `InstallPpi` | 注册 PPI 到数据库 |
| 查找 | `LocatePpi` | 按 GUID 查找 |
| 通知 | `NotifyPpi` | PPI 安装时回调 |
| 替换 | `ReInstallPpi` | 替换已有 PPI |

> **一句话总结**：PPI 是 PEI 阶段的轻量版 Protocol——用 GUID 标识，用函数指针表定义接口，让 PEIM 之间能互相发现和调用。

---

下一篇：**#044 内存初始化（MRC）：PEI 阶段最关键的一步**——看看初始化几 GB DRAM 到底有多复杂。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
