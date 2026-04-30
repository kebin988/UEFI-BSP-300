# CAR（Cache As RAM）：没有内存怎么跑 C 代码？

> 🔥 UEFI/BSP 开发系列第 039 篇 | 难度：⭐⭐⭐ 中级
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

在前两篇中我们提到了一个关键词：**CAR (Cache As RAM)**。

简单说了一句"用 CPU Cache 当 RAM"，但没展开。今天就把这个机制彻底讲清楚。

这是固件工程师面试的**高频考点**，也是理解 UEFI 启动早期阶段的关键。如果你能在面试中把 CAR 的原理、配置方法、局限性讲清楚，基本稳了。

---

## 一、为什么需要 CAR？

### 困境

CPU 上电后想执行 C 代码，但 C 代码需要：
- **栈（Stack）**—— 函数调用、局部变量
- **数据段**—— 全局变量、静态变量

这些都需要**可读写的内存**。但此时 DRAM 还没初始化！

```
鸡生蛋问题：

想初始化内存 → 需要执行 C 代码
执行 C 代码  → 需要有内存（栈）
有内存       → 需要先初始化内存

死循环！
```

### 解法

Intel 想了个绝妙的办法：**把 CPU 内部的 Cache 当 RAM 用**。

```
正常情况：
  CPU ←→ Cache ←→ DRAM
  Cache 是 DRAM 的透明缓存，自动回写

CAR 模式：
  CPU ←→ Cache（锁定，不回写）
  Cache 被当作独立的 SRAM 使用
  DRAM 完全不参与
```

---

## 二、CPU Cache 快速回顾

先确保你理解正常模式下 Cache 的工作方式：

```
CPU 读地址 0x1000：
  1. 查 Cache → 命中（Hit）→ 直接返回数据
  2. 查 Cache → 未命中（Miss）→ 去 DRAM 取数据 → 存一份到 Cache → 返回

CPU 写地址 0x1000：
  Write-Back 模式：
    先写到 Cache → 标记为 Dirty → 之后再回写到 DRAM
  Write-Through 模式：
    同时写 Cache 和 DRAM
```

**关键**：正常模式下 Cache Miss 会访问 DRAM。但启动初期 DRAM 不存在，Cache Miss = 灾难。

---

## 三、CAR 的核心思想

**让 Cache 永远不 Miss（不去访问 DRAM）**。怎么做到？

```
第 1 步：选一段地址范围作为 CAR 区域（比如 0xFEF00000 ~ 0xFEF80000，512KB）
第 2 步：用 MTRR 标记这段区域为 Write-Back
第 3 步：预先把 Cache Line 填满（Pre-fill）
第 4 步：设置 No-Eviction Mode（不驱逐）

结果：
  - 读这段地址 → 永远 Cache Hit（数据已在 Cache）
  - 写这段地址 → 写到 Cache（不回写 DRAM，因为 No-Eviction）
  - 这段 Cache 就变成了一块可读写的 SRAM！
```

---

## 四、具体配置步骤（Intel 平台）

### 4.1 设置 MTRR

**MTRR (Memory Type Range Registers)** 控制每段地址的缓存策略：

```nasm
; 设置 CAR 区域的 MTRR
; 把 CAR_BASE ~ CAR_BASE+CAR_SIZE 标记为 Write-Back (WB)

CAR_BASE  EQU  0xFEF00000    ; CAR 起始地址
CAR_SIZE  EQU  0x00080000    ; 512 KB

; MTRR PhysBase
mov     ecx, IA32_MTRR_PHYSBASE0   ; MSR 0x200
mov     eax, CAR_BASE | 6          ; 6 = Write-Back type
xor     edx, edx
wrmsr

; MTRR PhysMask  
mov     ecx, IA32_MTRR_PHYSMASK0   ; MSR 0x201
mov     eax, (~(CAR_SIZE - 1)) | 0x800  ; 有效位 + 掩码
mov     edx, 0x0000000F
wrmsr
```

### 4.2 开启 Cache 并填充

```nasm
; 1. 确保 Cache 开启（CR0.CD = 0）
mov     eax, cr0
and     eax, ~(1 << 30)    ; 清除 CD (Cache Disable)
and     eax, ~(1 << 29)    ; 清除 NW (Not Write-through)  
mov     cr0, eax

; 2. 失效整个 Cache
wbinvd

; 3. 读取 CAR 区域的每个 Cache Line，填满 Cache
mov     edi, CAR_BASE
mov     ecx, CAR_SIZE / 64  ; 每个 Cache Line 64 字节
.fill_loop:
    mov     eax, [edi]       ; 读一下，让 Cache Line 被填入
    add     edi, 64
    loop    .fill_loop
```

### 4.3 设置 No-Eviction Mode (NEM)

Intel 的 No-Eviction Mode 通过特定的 MSR 控制：

```nasm
; Intel 特定：设置 No-Eviction Mode
; （不同代的 Intel CPU 方法略有不同）

; 方法 1：通过 CR0.CD 控制
;   开启 CD 后新的 Cache Line 不再分配
;   已有的 Cache Line 不被驱逐

; 方法 2：Intel NEM MSR（新平台）
mov     ecx, MSR_NO_EVICT_MODE    ; 平台相关 MSR
rdmsr
or      eax, 1                     ; 开启 NEM
wrmsr
```

### 4.4 设置栈，跳转到 C

```nasm
; 设置栈指针到 CAR 区域顶部
mov     esp, CAR_BASE + CAR_SIZE

; 现在有了栈，可以 call C 函数了！
push    CAR_SIZE            ; 参数: RAM 大小
push    CAR_BASE            ; 参数: RAM 基址
call    SecStartup          ; 跳转到 C 代码！
```

---

## 五、CAR 的内存布局

在 CAR 可用之后，这块有限的空间这样划分：

```
CAR_BASE + CAR_SIZE (顶部)
┌─────────────────────────┐
│                         │
│     Stack (栈)          │  ← ESP 从这里向下增长
│     (约 32-128 KB)      │
│                         │
├─────────────────────────┤
│                         │
│     Heap (堆)           │  ← PEI Core 的内存分配池
│     (约 128-256 KB)     │
│                         │
├─────────────────────────┤
│     PEI Core 数据       │  ← HOB List、PPI 数据库
│     (约 64-128 KB)      │
├─────────────────────────┤
│     SEC 数据            │  ← 临时变量等
│     (很小)              │
└─────────────────────────┘
CAR_BASE (底部)
```

**注意**：这只有 **几百 KB**！所以 PEI 阶段的代码必须非常节省内存。

---

## 六、CAR 的局限性

| 限制 | 说明 |
|------|------|
| **容量极小** | 通常只有 256KB ~ 1MB（取决于 CPU Cache 大小） |
| **不能溢出** | 栈溢出 = 写到 Cache 外面 = 灾难（数据丢失或总线错误） |
| **CPU 相关** | 不同 CPU 代次的配置方法不同 |
| **不能 DMA** | 外设不能 DMA 到 Cache（Cache 不是真 RAM） |
| **多核问题** | 每个核有自己的 Cache，共享数据很麻烦 |
| **调试困难** | 内存工具看不到 Cache 内容 |

### 栈溢出是最常见的 CAR 问题

```c
// 这在 PEI 阶段是禁忌：
void BadFunction(void) {
    UINT8 BigBuffer[65536];  // 64KB 局部变量！
    // 如果 CAR 只有 256KB，这可能直接栈溢出
    // 导致整个系统崩溃，没有任何错误提示
}

// 正确做法：使用 PEI 堆分配
void GoodFunction(void) {
    UINT8 *Buffer;
    Buffer = AllocatePages(16);  // 从 PEI 堆分配
    // ...
}
```

---

## 七、CAR 结束：迁移到真正的 DRAM

当 PEI 阶段成功初始化 DRAM 后，需要**把 CAR 中的数据搬到 DRAM**，然后关闭 CAR：

```
CAR → DRAM 迁移过程：

1. DRAM 初始化完成（Memory Init PEIM 的成果）

2. 在 DRAM 中分配空间
   - 新栈区域
   - 数据拷贝目标

3. 把 CAR 中的所有数据复制到 DRAM
   - 栈内容 → 新栈位置
   - HOB List → DRAM
   - PPI 数据库 → DRAM

4. 切换栈指针
   ESP = 新栈在 DRAM 的地址

5. 关闭 CAR（恢复 Cache 正常模式）
   - 清除 NEM 模式
   - 恢复 MTRR
   - WBINVD（清空 Cache）
   - Cache 恢复为 DRAM 的透明缓存

6. 继续执行（现在在 DRAM 上运行）
```

在 EDK2 中，这个过程由 **MigrateTemporaryRam** 或 **CacheAsRamMigrate** 函数完成：

```c
// 简化的 CAR 迁移逻辑
VOID
EFIAPI
SecTemporaryRamDone(VOID)
{
    // 1. 复制 CAR 数据到 DRAM
    CopyMem(
        (VOID *)NewRamBase,       // DRAM 目标地址
        (VOID *)CAR_BASE,         // CAR 源地址
        CAR_SIZE
    );

    // 2. 调整栈指针
    //    (编译器会帮处理 EBP/ESP 的偏移修正)
    
    // 3. 关闭 CAR
    DisableNoEvictionMode();
    
    // 4. 恢复 MTRR 为正常配置
    RestoreMtrr();
    
    // 5. 刷新 Cache
    AsmWbinvd();
}
```

---

## 八、不同平台的 CAR 实现对比

| 平台 | CAR 实现方式 | 大小 | 说明 |
|------|------------|------|------|
| **Intel** | MTRR + NEM (No-Eviction Mode) | 256KB ~ 1MB | 最经典的实现 |
| **AMD** | MTRR + Fixed MTRR + NEM | 256KB ~ 512KB | 类似 Intel 但细节不同 |
| **ARM** | 不需要 CAR | — | SoC 通常有片上 SRAM |
| **RISC-V** | 取决于实现 | — | 可能有 SRAM 或类似机制 |

### ARM 为什么不需要 CAR？

ARM SoC 通常在芯片内部集成了 **SRAM (片上静态RAM)**：

```
ARM SoC 内部：
┌─────────────────────────┐
│  CPU Core               │
│  L1/L2 Cache            │
├─────────────────────────┤
│  On-chip SRAM           │  ← 上电即可用！不需要初始化
│  (64KB ~ 512KB)         │
├─────────────────────────┤
│  Memory Controller      │  ← 控制外部 DRAM
│  Peripheral Controllers │
└─────────────────────────┘
```

ARM 的 SRAM 天生可用，CPU 上电后直接把栈设到 SRAM 就行，不需要 Cache 的 hack。

---

## 九、调试 CAR 问题

CAR 出问题时的常见症状：

| 症状 | 可能原因 |
|------|---------|
| CPU Reset 循环 | CAR 配置错误，执行到无效代码 |
| 随机数据损坏 | Cache 被意外驱逐（NEM 没配好） |
| 栈溢出死机 | PEI 代码用了太多栈空间 |
| CAR 迁移后崩溃 | 指针没有正确修正 |
| 多核启动失败 | BSP/AP 的 CAR 区域冲突 |

调试手段：

```c
// 1. Post Code 追踪
IoWrite8(0x80, 0xC0);  // "CAR 开始配置"
// ... 配置代码 ...
IoWrite8(0x80, 0xC1);  // "CAR 配置完成"
IoWrite8(0x80, 0xC2);  // "栈已设置"
IoWrite8(0x80, 0xC3);  // "C 代码入口"

// 2. 验证 CAR 可读写
*(volatile UINT32 *)CAR_BASE = 0xDEADBEEF;
if (*(volatile UINT32 *)CAR_BASE != 0xDEADBEEF) {
    // CAR 配置失败！
    CpuDeadLoop();
}
```

---

## 十、总结

```
CAR 本质：
┌─────────────────────────────────────────┐
│  "我暂时把 Cache 从自动模式切到手动模式  │
│   当作一块 SRAM 用，撑到真内存初始化完" │
└─────────────────────────────────────────┘
```

| 要点 | 说明 |
|------|------|
| 什么是 CAR | 把 CPU Cache 锁定为可读写的 RAM |
| 为什么需要 | DRAM 未初始化前需要栈来执行 C 代码 |
| 怎么实现 | MTRR (WB) + Cache 预填充 + No-Eviction Mode |
| 容量 | 256KB ~ 1MB（很小！） |
| 生存期 | SEC 阶段建立 → DRAM 初始化后销毁 |
| 结束方式 | 数据迁移到 DRAM → 恢复 Cache 正常模式 |
| ARM 替代 | 片上 SRAM，天生可用 |

> **一句话总结**：CAR 是 x86 的"急救措施"——在内存可用之前，骗 CPU 把自己的 Cache 当 RAM 用。这个 hack 虽然丑陋，但撑起了整个 UEFI 的 PEI 阶段。

---

下一篇：**#040 PEI 阶段详解：PEIM 模块和 PPI 通信机制**——看看 PEI 阶段的模块是怎么互相协作的。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
