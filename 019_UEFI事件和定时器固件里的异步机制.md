# UEFI 事件和定时器：固件里的异步机制

> 🔥 UEFI/BSP 开发系列第 019 篇 | 难度：⭐⭐ 进阶入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们搞明白了 UEFI 变量——固件的"持久化记忆"。

今天聊一个你可能没想到的话题：

> "固件里有没有异步机制？比如定时器、回调函数之类的？"

你可能会想：固件又不是操作系统，没有线程、没有进程，哪来的异步？

但实际上，UEFI 确实有一套事件（Event）系统——虽然它不像操作系统那样有抢占式多任务，但它能做到"等某个条件满足时再通知我"，甚至能设定时器定期触发回调函数。

今天就把这个"固件里的异步机制"讲清楚。

---

## 一、为什么固件需要事件机制？

先想几个场景：

```
场景 1：等用户按键
  BIOS 设置界面等你按方向键选择启动项。
  总不能让 CPU 在那空转死等吧？

场景 2：超时机制
  "5 秒内按任意键进入 BIOS 设置，否则自动启动 Windows"
  这个倒计时谁来计时？

场景 3：网络启动
  PXE 启动时，固件发出 DHCP 请求，然后等待服务器响应。
  等多久？超时了怎么办？

场景 4：周期性任务
  某些硬件需要固件定期去查询状态（轮询）。
  谁来管理这个"定期"？
```

所有这些场景都需要一个核心能力：**"等待某件事发生，发生后通知我"**。

这就是 UEFI 事件系统要解决的问题。

---

## 二、UEFI 事件的基本概念

### 什么是 Event？

在 UEFI 中，Event 是一个**不透明的句柄**（`EFI_EVENT` 类型），代表一个"可等待的条件"。

```c
// Event 本质上就是一个指针/句柄
typedef VOID *EFI_EVENT;
```

Event 有两种状态：
- **未触发**（Not Signaled）：条件还没满足
- **已触发**（Signaled）：条件满足了，可以处理了

```
Event 状态机：

  ┌───────────┐    SignalEvent()    ┌───────────┐
  │ Not       │ ─────────────────→ │ Signaled  │
  │ Signaled  │                    │           │
  └───────────┘ ←───────────────── └───────────┘
                  WaitForEvent() /
                  CheckEvent() 消费后重置
```

### Event 的三种类型

| 类型 | 说明 | 典型用途 |
|------|------|----------|
| `EVT_TIMER` | 定时器事件 | 超时控制、周期性轮询 |
| `EVT_NOTIFY_SIGNAL` | 被触发时自动调用回调函数 | 异步通知 |
| `EVT_NOTIFY_WAIT` | 被检查（Wait/Check）时调用回调函数 | 轮询检查 |

> 💡 你可以简单理解为：UEFI Event ≈ 简化版的 Linux `select/poll` + 定时器 + 回调函数。

---

## 三、创建和使用 Event

### 1. 创建事件：CreateEvent

```c
EFI_EVENT  MyEvent;

EFI_STATUS Status = gBS->CreateEvent(
    EVT_TIMER | EVT_NOTIFY_SIGNAL,  // 事件类型：定时器 + 触发时回调
    TPL_CALLBACK,                    // 回调函数的优先级
    MyTimerCallback,                 // 回调函数
    &SomeContext,                    // 传给回调函数的上下文数据
    &MyEvent                         // 输出：创建的事件句柄
);
```

### 2. 回调函数长什么样？

```c
VOID
EFIAPI
MyTimerCallback(
    IN EFI_EVENT  Event,
    IN VOID       *Context
)
{
    // 定时器到期了！在这里做你想做的事
    UINT32 *Counter = (UINT32 *)Context;
    (*Counter)++;
    Print(L"定时器触发了！计数: %d\n", *Counter);
}
```

> 💡 注意：回调函数的签名是固定的——两个参数：Event 句柄和用户上下文指针。跟 C 语言的 `signal handler` 或者 JavaScript 的回调函数是同一个思路。

### 3. 设置定时器：SetTimer

```c
// 一次性定时器：3 秒后触发
gBS->SetTimer(
    MyEvent,
    TimerRelative,        // 相对时间，一次性
    30000000              // 时间单位：100 纳秒。3 秒 = 3 * 10^7 * 100ns
);

// 周期性定时器：每 1 秒触发一次
gBS->SetTimer(
    MyEvent,
    TimerPeriodic,        // 周期性
    10000000              // 1 秒 = 10^7 * 100ns
);

// 取消定时器
gBS->SetTimer(MyEvent, TimerCancel, 0);
```

> ⚠️ **时间单位是 100 纳秒**（不是毫秒也不是微秒）。这是 UEFI 的坑之一——很多人第一次用会搞错单位。
>
> 换算公式：`秒 × 10,000,000 = UEFI 定时器单位`

### 4. 等待事件：WaitForEvent

```c
UINTN      Index;
EFI_EVENT  WaitList[2] = { KeyEvent, TimerEvent };

// 阻塞等待：直到其中一个事件被触发
Status = gBS->WaitForEvent(
    2,              // 等待的事件数量
    WaitList,       // 事件数组
    &Index          // 输出：哪个事件被触发了（数组下标）
);

if (Index == 0) {
    Print(L"用户按了键！\n");
} else {
    Print(L"超时了！\n");
}
```

### 5. 非阻塞检查：CheckEvent

```c
Status = gBS->CheckEvent(MyEvent);
if (Status == EFI_SUCCESS) {
    // 事件已触发
} else if (Status == EFI_NOT_READY) {
    // 事件还没触发，继续干别的
}
```

### 6. 关闭事件：CloseEvent

```c
gBS->CloseEvent(MyEvent);
```

---

## 四、实战：等待按键或超时

这是 UEFI 开发中最经典的事件使用场景——"按任意键继续，5 秒后自动跳过"：

```c
EFI_STATUS
WaitForKeyOrTimeout(
    IN UINTN  TimeoutSeconds
)
{
    EFI_STATUS  Status;
    EFI_EVENT   TimerEvent;
    EFI_EVENT   WaitList[2];
    UINTN       Index;

    // 创建一个一次性定时器事件
    Status = gBS->CreateEvent(
        EVT_TIMER,
        TPL_APPLICATION,
        NULL,
        NULL,
        &TimerEvent
    );
    if (EFI_ERROR(Status)) return Status;

    // 设置定时器：TimeoutSeconds 秒后触发
    gBS->SetTimer(
        TimerEvent,
        TimerRelative,
        TimeoutSeconds * 10000000ULL  // 转换为 100ns 单位
    );

    // 等待两个事件中的任意一个
    WaitList[0] = gST->ConIn->WaitForKey;  // 键盘输入事件
    WaitList[1] = TimerEvent;               // 定时器事件

    Status = gBS->WaitForEvent(2, WaitList, &Index);

    if (Index == 0) {
        // 用户按了键
        EFI_INPUT_KEY Key;
        gST->ConIn->ReadKeyStroke(gST->ConIn, &Key);
        Print(L"\n你按了: %c\n", Key.UnicodeChar);
    } else {
        // 超时了
        Print(L"\n超时，自动继续...\n");
    }

    // 清理
    gBS->CloseEvent(TimerEvent);
    return EFI_SUCCESS;
}

// 使用
Print(L"按任意键进入 BIOS 设置（5 秒超时）...");
WaitForKeyOrTimeout(5);
```

> 💡 `gST->ConIn->WaitForKey` 是系统自带的一个事件——当键盘有输入时会被触发。这是 `EFI_SIMPLE_TEXT_INPUT_PROTOCOL` 里预定义的。

---

## 五、任务优先级（TPL）

UEFI 没有线程，但它有**任务优先级（Task Priority Level, TPL）**——用来控制回调函数的执行顺序和中断嵌套。

```c
// 四个优先级，从低到高
#define TPL_APPLICATION   4   // 最低，普通应用代码
#define TPL_CALLBACK      8   // 回调函数
#define TPL_NOTIFY       16   // 通知级，多数 Protocol 通知
#define TPL_HIGH_LEVEL   31   // 最高，相当于关中断
```

### TPL 的规则

```
低优先级的代码可以被高优先级的回调打断：

  TPL_APPLICATION 代码正在执行
      ↓
  定时器中断触发
      ↓
  TPL_CALLBACK 级别的回调函数执行
      ↓
  回调完成，回到 TPL_APPLICATION 继续
```

### 提升和恢复优先级

```c
// 临界区：提升优先级来防止被打断（类似自旋锁）
EFI_TPL OldTpl = gBS->RaiseTPL(TPL_HIGH_LEVEL);  // 提升到最高

// ... 在这里做不能被打断的操作 ...

gBS->RestoreTPL(OldTpl);  // 恢复原来的优先级
```

> 💡 `RaiseTPL(TPL_HIGH_LEVEL)` 相当于操作系统中的"关中断"——保证当前代码不会被任何回调打断。用完一定要 `RestoreTPL`，否则整个系统都"卡住"了。

### TPL 使用的常见陷阱

| 错误用法 | 后果 |
|---------|------|
| 在 `TPL_HIGH_LEVEL` 下调 `WaitForEvent` | 死锁！高优先级下不允许等待 |
| 忘记 `RestoreTPL` | 所有低优先级回调永远不会执行 |
| 回调函数里再 `RaiseTPL(TPL_HIGH_LEVEL)` | 可以，但要确保能恢复 |
| 在 `TPL_NOTIFY` 回调里调用 `AllocatePool` | 有些平台实现中会有问题 |

---

## 六、Signal 事件和 Notify 事件的区别

这是初学者最容易混淆的地方：

### EVT_NOTIFY_SIGNAL

"事件被触发（Signal）的**那一刻**，自动调用回调函数。"

```c
// 场景：当某个 Protocol 被安装时，通知我
gBS->CreateEvent(
    EVT_NOTIFY_SIGNAL,
    TPL_CALLBACK,
    OnProtocolInstalled,   // 当事件被 Signal 时调用
    NULL,
    &NotifyEvent
);

gBS->RegisterProtocolNotify(
    &gSomeProtocolGuid,
    NotifyEvent,
    &Registration
);

// 当有人安装了 SomeProtocol 时，OnProtocolInstalled 自动被调用
```

### EVT_NOTIFY_WAIT

"当有人**检查**（Wait/Check）这个事件时，先调用回调函数。"

```c
// 场景：键盘驱动 - 当有人等待按键时，去查一下键盘有没有数据
gBS->CreateEvent(
    EVT_NOTIFY_WAIT,
    TPL_NOTIFY,
    CheckKeyboardBuffer,   // 被 WaitForEvent/CheckEvent 时调用
    &KeyboardContext,
    &WaitForKeyEvent
);

// 这个 WaitForKeyEvent 放进 Protocol 里
// 当用户调用 WaitForEvent(WaitForKeyEvent) 时
// 系统会先调用 CheckKeyboardBuffer 去查有没有按键
// 如果有 → Signal 这个事件 → WaitForEvent 返回
// 如果没有 → 事件保持 Not Signaled → 继续等
```

> 💡 简单记忆：
> - `NOTIFY_SIGNAL`：**被触发时**回调 → 用于异步通知
> - `NOTIFY_WAIT`：**被检查时**回调 → 用于轮询检查

---

## 七、UEFI 事件 vs 操作系统中的类似机制

| UEFI 事件 | Linux | Windows | 概念类比 |
|-----------|-------|---------|----------|
| `CreateEvent` | `eventfd()` / `timerfd_create()` | `CreateEvent()` | 创建事件对象 |
| `SignalEvent` | `write(eventfd)` | `SetEvent()` | 触发事件 |
| `WaitForEvent` | `select()` / `poll()` / `epoll_wait()` | `WaitForMultipleObjects()` | 等待事件 |
| `CheckEvent` | `poll()` (非阻塞) | `WaitForSingleObject(0)` | 非阻塞检查 |
| `SetTimer` | `timer_settime()` | `SetWaitableTimer()` | 设置定时器 |
| `RaiseTPL` | `local_irq_disable()` | `KeRaiseIrql()` | 提升优先级/关中断 |
| 回调函数 | 信号处理器 / 中断处理 | APC / DPC | 异步回调 |

### 核心区别

```
操作系统的异步：
  多线程 / 多进程 → 真正的并发
  抢占式调度 → 时间片轮转
  
UEFI 的异步：
  单线程 → 没有真正的并发
  协作式 → 依赖定时器中断触发回调
  本质上是"中断驱动的事件循环"
```

> 打个比方：操作系统是一个大公司，多个员工（线程）同时干活。UEFI 是一个人的小作坊——只有一个人，但他有个闹钟（定时器），闹钟响了就暂停手头的活去处理另一件事，处理完再回来。

---

## 八、Protocol 通知：最实用的事件场景

在实际 UEFI 开发中，最常用的事件场景不是定时器，而是 **Protocol 安装通知**：

```c
// "当系统中有人安装了 XXX Protocol 时，通知我"
EFI_EVENT   Event;
VOID        *Registration;

gBS->CreateEvent(
    EVT_NOTIFY_SIGNAL,
    TPL_CALLBACK,
    OnBlockIoInstalled,
    NULL,
    &Event
);

gBS->RegisterProtocolNotify(
    &gEfiBlockIoProtocolGuid,  // 我想监听这个 Protocol
    Event,
    &Registration
);

// 回调函数
VOID EFIAPI OnBlockIoInstalled(IN EFI_EVENT Event, IN VOID *Context)
{
    // 有新的存储设备出现了！
    // 可以在这里枚举所有 BlockIO 设备
    Print(L"检测到新的存储设备！\n");
}
```

这个机制在 DXE 阶段非常重要——因为驱动的加载顺序不确定，你的模块可能在目标 Protocol 安装之前就运行了。通过注册通知，你可以"等它来了再处理"，而不是傻等。

> 💡 这就是典型的**观察者模式**——"别来问我好了没，好了我通知你。"

---

## 九、ExitBootServices 事件

另一个关键事件是 **ExitBootServices 通知**——当操作系统即将接管硬件时，固件的某些模块需要做"收尾工作"：

```c
EFI_EVENT  ExitBsEvent;

gBS->CreateEventEx(
    EVT_NOTIFY_SIGNAL,
    TPL_CALLBACK,
    OnExitBootServices,
    NULL,
    &gEfiEventExitBootServicesGuid,  // 特殊 GUID
    &ExitBsEvent
);

VOID EFIAPI OnExitBootServices(IN EFI_EVENT Event, IN VOID *Context)
{
    // OS 要接管了！最后的清理工作
    // - 停止 DMA 设备
    // - 保存必要状态
    // - 关闭不再需要的硬件
    DisableMyDevice();
}
```

> 💡 为什么这很重要？如果你的驱动正在用 DMA 往内存写数据，而 OS 回收了那块内存……砰！数据腐坏、系统崩溃。所以在 ExitBootServices 时必须停掉所有正在进行的 DMA 操作。

---

## 十、常见疑问

### Q1：UEFI 的事件是中断驱动的吗？

是，也不完全是。定时器事件靠**硬件定时器中断**驱动（通常是 8254/HPET/Local APIC Timer）。但回调函数的执行不是在中断上下文——中断处理程序只是标记事件为 Signaled，真正的回调在中断返回后、按 TPL 优先级执行。

### Q2：UEFI 有多线程吗？

标准 UEFI 规范是**单线程**的。所有代码运行在 BSP（Bootstrap Processor，主 CPU 核心）上。其他 CPU 核心在 UEFI 阶段通常处于等待状态（通过 `MP Protocol` 可以向它们派发任务，但那不是通用的多线程）。

### Q3：回调函数里能做什么？不能做什么？

能做的事取决于 TPL 级别：
- `TPL_CALLBACK`：可以使用大部分 Boot Services
- `TPL_NOTIFY`：小心使用，不能调用可能阻塞的函数
- `TPL_HIGH_LEVEL`：几乎什么都不能做，赶紧干完赶紧还

经验法则：**回调函数要短小精悍**，不要在里面做耗时操作。

### Q4：WaitForEvent 能等多个事件吗？

可以，最多可以等一组事件。返回值告诉你哪个事件先被触发了。但注意：**不能在 `TPL_APPLICATION` 以上的级别调用 `WaitForEvent`**——否则会死锁。

---

## 十一、总结

```
UEFI 事件系统：

创建 → CreateEvent / CreateEventEx
         ├── EVT_TIMER            → 定时器
         ├── EVT_NOTIFY_SIGNAL    → 被触发时回调
         └── EVT_NOTIFY_WAIT      → 被检查时回调

控制 → SetTimer (定时) / SignalEvent (触发) / CloseEvent (销毁)

等待 → WaitForEvent (阻塞) / CheckEvent (非阻塞)

优先级 → TPL_APPLICATION < TPL_CALLBACK < TPL_NOTIFY < TPL_HIGH_LEVEL

典型场景：
  ├── 等待按键或超时
  ├── Protocol 安装通知
  ├── ExitBootServices 清理
  └── 周期性硬件轮询
```

| 你以为的固件异步 | 实际的固件异步 |
|----------------|--------------|
| 固件不需要异步 | 等键盘、超时、通知都需要 |
| 固件有多线程 | 单线程 + 中断驱动的事件循环 |
| 很复杂 | CreateEvent + WaitForEvent，跟 select/poll 类似 |
| 只有 OS 才有回调 | UEFI 的回调函数一样好使 |

---

下一篇：**#020 理解 GUID：UEFI 世界的身份证号**——Protocol 有 GUID，变量有 GUID，事件有 GUID……UEFI 里到处都是 GUID，它到底是什么？为什么要用它？怎么生成？

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
