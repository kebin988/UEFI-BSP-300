# SEC 阶段：CPU 上电后执行的第一条指令在哪里？

> 🔥 UEFI/BSP 开发系列第 037 篇 | 难度：⭐⭐ 进阶
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们画了 SEC→PEI→DXE→BDS 的全景图。从这篇开始，我们把镜头对准每个阶段，像慢动作回放一样看清楚里面的每一个细节。

今天聚焦 **SEC 阶段**——整个 UEFI 固件的**第一行代码**从哪里来、怎么执行、干了什么。

> 这个阶段虽然短（通常只有几微秒到几毫秒），但它决定了系统能不能活过"第一口呼吸"。

---

## 一、CPU 上电：从物理层面说起

当你按下电源键：

```
电源按下
    │
    ▼
电源单元（PSU）开始供电
    │
    ▼
各路电压稳定（3.3V、1.8V、1.1V、0.8V...）
    │
    ▼
芯片组（PCH/南桥）产生 RESET 信号
    │
    ▼
CPU 接收到 RESET 信号解除 → CPU 开始执行第一条指令
```

**关键点**：CPU 上电后不是"从头开始"，而是从一个**硬件决定的固定地址**开始取指令。这个地址叫 **Reset Vector**。

---

## 二、Reset Vector：CPU 的"出生地"

### 2.1 x86 架构

x86 CPU 上电后处于 **Real Mode**（实模式），但有个诡异的设定：

- CS（代码段寄存器）= `0xF000`
- IP（指令指针）= `0xFFF0`
- 但 CS 的 Base 被硬件设置为 `0xFFFF0000`（而不是正常实模式下的 `0xF000 << 4 = 0xF0000`）

所以第一条指令的**物理地址** = `0xFFFF0000 + 0xFFF0` = **`0xFFFFFFF0`**

```
4 GB 地址空间：

0x00000000 ┌─────────────────────┐
           │                     │
           │    DRAM（未初始化）  │
           │                     │
           ├─────────────────────┤
           │       ...           │
           ├─────────────────────┤
0xFF000000 │                     │
           │   SPI Flash 映射区  │  ← Flash 被映射到地址空间顶部
           │                     │
0xFFFFFFF0 │ ← Reset Vector!     │  ← CPU 的第一条指令在这里
0xFFFFFFFF └─────────────────────┘
```

Flash 芯片通过 SPI 总线连接，芯片组把它映射到 4GB 空间的顶部。所以 CPU 上电后直接从 Flash 执行代码。

### 2.2 ARM AArch64 架构

ARM 简单得多：

- Reset Vector 默认在 `0x00000000`
- 也可以通过硬件引脚（RVBAR）配置到其他地址
- CPU 上电后直接进入 EL3（最高异常等级）

### 2.3 RISC-V 架构

- Reset Vector 由硬件实现决定
- 通常在 `0x80000000` 或 `0x00001000`

---

## 三、Reset Vector 处的代码长啥样

`0xFFFFFFF0` 处只有 **16 个字节**（到 `0xFFFFFFFF` 就是地址空间末尾了），所以通常是一条跳转指令：

```nasm
; x86 Reset Vector（16字节空间）
; 位于 Flash 中的最末尾 16 字节

ORG 0xFFFFFFF0

ResetVector:
    jmp     far ptr SEC_ENTRY    ; 跳转到 SEC 的真正入口
    ; 或者
    jmp     short RelativeOffset ; 短跳转到附近代码

; 剩余字节可能是填充或签名
```

在 EDK2 源码中，x86 的 Reset Vector 代码位于：

```
edk2/UefiCpuPkg/ResetVector/Vtf0/
├── Ia16/
│   ├── ResetVectorVtf0.asm    ← 16位实模式代码
│   └── Init16.asm
├── Ia32/
│   └── ...
└── X64/
    └── ...
```

实际代码（简化版）：

```nasm
; ResetVectorVtf0.asm (简化)
BITS 16

; 这是 CPU 执行的第一条指令！
; 紧接着就跳转到 SEC Core 入口
ResetHandler:
    ; 关中断
    cli
    ; 跳转到 SEC 入口（在同一个 Flash 中）
    jmp     EarlyBspInitReal16
```

---

## 四、SEC 入口做了什么

跳转到 SEC 的真正入口后，开始执行一系列**极其底层**的操作：

### 4.1 第一步：切换到保护模式

x86 上电后是 16 位实模式，只能寻址 1MB。SEC 要做的第一件事就是切到保护模式（32 位）或长模式（64 位）：

```nasm
; 1. 加载临时 GDT
    lgdt    [TempGdt]

; 2. 开启保护模式（设置 CR0.PE）
    mov     eax, cr0
    or      eax, 1          ; PE bit
    mov     cr0, eax

; 3. 跳转到 32 位代码段
    jmp     CODE32_SEL:ProtectedModeEntry

BITS 32
ProtectedModeEntry:
    ; 现在是 32 位保护模式了
    ; 设置段寄存器
    mov     ax, DATA32_SEL
    mov     ds, ax
    mov     es, ax
    mov     ss, ax
```

### 4.2 第二步：建立 Cache As RAM (CAR)

这是 SEC 最核心的工作——让 CPU 有一块可用的"内存"：

```nasm
; 设置 MTRR，把一段地址范围标记为 Write-Back
; 然后锁定 Cache，使其不回写到（不存在的）DRAM

; 1. 关闭 Cache
    mov     eax, cr0
    or      eax, (1 << 30)  ; CD (Cache Disable)
    mov     cr0, eax
    wbinvd                  ; 清空 Cache

; 2. 设置 MTRR，指定 CAR 区域
    ; 把 CAR_BASE ~ CAR_BASE+CAR_SIZE 设为 Write-Back
    mov     ecx, MTRR_PHYS_BASE_0
    mov     eax, CAR_BASE | MTRR_TYPE_WB
    wrmsr

    mov     ecx, MTRR_PHYS_MASK_0
    mov     eax, CAR_MASK | MTRR_VALID
    wrmsr

; 3. 开启 Cache（但 No-Eviction Mode）
    mov     eax, cr0
    and     eax, ~(1 << 30)  ; 清除 CD
    mov     cr0, eax

; 4. 用 Cache 预填充
    ; 读取 CAR 区域的每个 Cache Line，让 Cache 填满
    mov     edi, CAR_BASE
    mov     ecx, CAR_SIZE / 4
    rep stosd

; 现在 CAR_BASE ~ CAR_BASE+CAR_SIZE 就是可用的"RAM"了！
```

CAR 建立后，终于可以设置栈了：

```nasm
; 设置栈指针
    mov     esp, CAR_BASE + CAR_SIZE  ; 栈顶
    ; 现在可以调用 C 函数了！

    ; 跳转到 SEC 的 C 代码入口
    call    SecStartup
```

### 4.3 第三步：进入 C 代码

从汇编跳到 C 后，SEC 的 C 代码主要做：

```c
VOID
EFIAPI
SecStartup(
    IN UINT32  SizeOfRam,
    IN UINT32  TempRamBase,
    IN VOID    *BootFirmwareVolume
)
{
    EFI_SEC_PEI_HAND_OFF    SecCoreData;
    EFI_PEI_PPI_DESCRIPTOR  *PpiList;

    // 1. 填充 SecCoreData（传给 PEI 的信息）
    SecCoreData.DataSize               = sizeof(SecCoreData);
    SecCoreData.BootFirmwareVolumeBase  = BootFirmwareVolume;
    SecCoreData.BootFirmwareVolumeSize  = FV_SIZE;
    SecCoreData.TemporaryRamBase       = (VOID *)TempRamBase;
    SecCoreData.TemporaryRamSize       = SizeOfRam;
    SecCoreData.PeiTemporaryRamBase    = (VOID *)(TempRamBase + SizeOfRam/2);
    SecCoreData.PeiTemporaryRamSize    = SizeOfRam / 2;
    SecCoreData.StackBase              = (VOID *)TempRamBase;
    SecCoreData.StackSize              = SizeOfRam / 2;

    // 2. 可选：安全验证（Verified Boot 的 Root of Trust）
    //    验证 PEI Core 的签名/哈希
    
    // 3. 查找 PEI Core 入口
    PeiCoreEntry = FindPeiCore(BootFirmwareVolume);

    // 4. 跳转到 PEI Core！SEC 使命完成
    (*PeiCoreEntry)(&SecCoreData, PpiList);
    
    // 永远不会执行到这里
    CpuDeadLoop();
}
```

---

## 五、SEC 阶段的安全角色

SEC 的全称是 **Security Phase**——安全阶段。它是信任链的**起点**：

```
硬件 Root of Trust
      │
      ▼
SEC 验证 PEI Core 的完整性（哈希/签名）
      │
      ▼
PEI Core 验证 PEIM 模块
      │
      ▼
DXE Core 验证 DXE 驱动
      │
      ▼
BDS 验证 OS Boot Loader（Secure Boot）
```

如果平台支持 Intel Boot Guard 或 ARM Trusted Boot：
- 硬件在 CPU Reset 前就验证了 SEC 代码的签名
- SEC 代码被 ACM (Authenticated Code Module) 验证
- 形成了从硬件到软件的完整信任链

---

## 六、在 EDK2 中找到 SEC 代码

```
EDK2 源码中的 SEC 相关目录：

edk2/
├── UefiCpuPkg/
│   ├── ResetVector/           ← Reset Vector 代码
│   │   └── Vtf0/
│   │       ├── Ia16/          ← 16 位启动代码
│   │       ├── Ia32/          ← 32 位代码
│   │       └── X64/           ← 64 位代码
│   └── SecCore/               ← SEC Core 模块
│       ├── SecMain.c          ← SEC 的 C 入口
│       ├── SecMain.h
│       └── FindPeiCore.c      ← 查找 PEI Core
│
├── ArmPlatformPkg/
│   └── Sec/                   ← ARM 平台的 SEC
│       └── Sec.c
│
├── OvmfPkg/
│   └── Sec/                   ← QEMU 虚拟机的 SEC（简化版）
│       └── SecMain.c
```

如果你用 QEMU + OVMF 调试，OVMF 的 SEC 是最简单的入门版本，因为虚拟机不需要真正的 CAR。

---

## 七、SEC 阶段的平台差异

| 平台 | SEC 特点 |
|------|---------|
| **Intel** | Reset Vector at 4GB-16, 需配置 CAR (MTRR), 可能有 Intel ACM 预验证 |
| **AMD** | Reset Vector at 4GB-16, PSP (Platform Security Processor) 先于 CPU 执行 |
| **ARM** | EL3 入口, 通常有 ATF (Arm Trusted Firmware) 作为更底层的 SEC |
| **RISC-V** | M-mode 入口, OpenSBI 可能承担部分 SEC 角色 |
| **QEMU/OVMF** | 极度简化, 不需要真正的 CAR（内存"天生"可用） |

> 💡 Intel 和 AMD 平台上，SEC 之前其实还有一个"隐藏阶段"：
> - Intel: ME (Management Engine) 先跑
> - AMD: PSP (Platform Security Processor) 先跑
> 
> 这些是独立的微处理器在主 CPU 之前执行的代码，不属于 UEFI 规范。

---

## 八、调试 SEC 阶段

SEC 是最难调试的阶段，因为：
- 没有内存 → 不能用 printf
- 没有串口初始化 → 不能输出
- 时间极短 → 逻辑分析仪才能捕捉

常用调试手段：

| 方法 | 适用场景 |
|------|---------|
| **Port 80h** | 往 IO 端口 0x80 写值，用 POST Card 显示 |
| **LED 闪烁** | 嵌入式平台常用，GPIO 点灯 |
| **JTAG/ITP** | 硬件调试器，连 CPU 的调试接口 |
| **QEMU + GDB** | 虚拟机调试，设置断点在 Reset Vector |
| **串口（CAR后）** | CAR 建立后可以初始化串口输出 |

```c
// 最原始的调试：写 Port 80h
IoWrite8(0x80, 0x01);  // 表示"我到了第 1 步"
IoWrite8(0x80, 0x02);  // 表示"我到了第 2 步"
// 用 POST Card（主板诊断卡）看这个值
```

---

## 九、总结

```
SEC 阶段时间线：

CPU Reset ──→ Reset Vector (0xFFFFFFF0)
                    │
                    ▼
              JMP to SEC Entry
                    │
                    ▼
              切换保护模式/长模式
                    │
                    ▼
              建立 CAR (Cache As RAM)
                    │
                    ▼
              设置栈 → 可以执行 C 代码了！
                    │
                    ▼
              验证 PEI Core（可选）
                    │
                    ▼
              跳转到 PEI Core
                    │
              ══════════════════
              SEC 使命结束（微秒~毫秒）
```

| 关键点 | 说明 |
|--------|------|
| Reset Vector | x86 在 4GB-16 处，ARM 在 0x0 |
| 第一条指令 | 通常是 JMP 跳转 |
| CAR | 用 CPU Cache 当临时 RAM |
| SEC 的产出 | 临时 RAM 信息 + PEI Core 入口 |
| 安全角色 | 信任链的根 (Root of Trust) |
| 典型大小 | 几 KB（汇编+少量C） |

---

下一篇：**#038 PEI 阶段详解：在没有内存的世界里初始化内存**——看看 PEI 怎么用几十 KB 的 Cache 完成初始化几 GB DRAM 的壮举。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
