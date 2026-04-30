# 从 C 语言角度理解 UEFI Protocol——就是函数指针表

> 🔥 UEFI/BSP 开发系列第 016 篇 | 难度：⭐ 入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们讲了 Handle 和 Protocol 的概念，用了医院的类比。你可能理解了"Protocol 是一组服务"，但心里还是犯嘀咕：

> "说白了到底是啥？能不能用代码让我看看？"

今天我们就从**纯 C 语言**的角度，把 Protocol 的本质扒个精光。

**剧透**：Protocol = 结构体 + 函数指针。没了。就这么简单。

---

## 一、先回忆一下 C 语言的函数指针

如果你学过 C 语言，函数指针肯定不陌生：

```c
// 声明一个函数指针类型
typedef int (*MathFunc)(int a, int b);

// 实现两个函数
int Add(int a, int b) { return a + b; }
int Mul(int a, int b) { return a * b; }

// 用函数指针调用
MathFunc operation = Add;
int result = operation(3, 5);  // result = 8

operation = Mul;
result = operation(3, 5);      // result = 15
```

核心思想：**不直接调用某个具体函数，而是通过指针间接调用**，运行时可以切换实现。

> 💡 如果你理解了函数指针，你就理解了 Protocol 的 80%。

---

## 二、把多个函数指针装进结构体

一个函数指针只能指向一个函数，如果我们想把**一组相关的功能**打包呢？很自然——用结构体：

```c
// 定义一组"动物行为"
typedef struct {
    void (*Eat)(void);
    void (*Sleep)(void);
    void (*MakeSound)(void);
} ANIMAL_PROTOCOL;
```

然后让不同的"动物"提供自己的实现：

```c
// 猫的实现
void CatEat(void)       { printf("猫在吃小鱼干\n"); }
void CatSleep(void)     { printf("猫在纸箱里蜷着睡\n"); }
void CatMeow(void)      { printf("喵～\n"); }

ANIMAL_PROTOCOL gCatProtocol = {
    .Eat       = CatEat,
    .Sleep     = CatSleep,
    .MakeSound = CatMeow
};

// 狗的实现
void DogEat(void)       { printf("狗在啃骨头\n"); }
void DogSleep(void)     { printf("狗四仰八叉躺在地上\n"); }
void DogBark(void)      { printf("汪！汪汪！\n"); }

ANIMAL_PROTOCOL gDogProtocol = {
    .Eat       = DogEat,
    .Sleep     = DogSleep,
    .MakeSound = DogBark
};
```

调用者不需要知道具体是猫还是狗：

```c
void InteractWithAnimal(ANIMAL_PROTOCOL *Animal) {
    Animal->Eat();        // 多态！
    Animal->MakeSound();  // 具体执行哪个函数，取决于传入的是哪个 Protocol
}

InteractWithAnimal(&gCatProtocol);  // 猫在吃小鱼干 → 喵～
InteractWithAnimal(&gDogProtocol);  // 狗在啃骨头 → 汪！汪汪！
```

> **这就是 UEFI Protocol 的本质**——一个装满函数指针的结构体。调用者面向接口编程，不关心底层实现。

---

## 三、真实的 UEFI Protocol 长啥样

来看 UEFI 规范中最常用的 `EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL`（在屏幕上打字用的）：

```c
typedef struct _EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL {
    EFI_TEXT_RESET          Reset;          // 重置输出设备
    EFI_TEXT_STRING         OutputString;   // 输出字符串（最核心！）
    EFI_TEXT_TEST_STRING    TestString;     // 测试字符串能否显示
    EFI_TEXT_QUERY_MODE     QueryMode;      // 查询显示模式
    EFI_TEXT_SET_MODE       SetMode;        // 设置显示模式
    EFI_TEXT_SET_ATTRIBUTE  SetAttribute;   // 设置颜色属性
    EFI_TEXT_CLEAR_SCREEN   ClearScreen;    // 清屏
    EFI_TEXT_SET_CURSOR     SetCursorPosition; // 设置光标位置
    EFI_TEXT_ENABLE_CURSOR  EnableCursor;   // 显示/隐藏光标
    SIMPLE_TEXT_OUTPUT_MODE *Mode;          // 当前模式信息
} EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL;
```

其中每个成员（除了 Mode）都是函数指针。比如 `EFI_TEXT_STRING` 的定义：

```c
typedef
EFI_STATUS
(EFIAPI *EFI_TEXT_STRING)(
    IN EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL  *This,
    IN CHAR16                          *String
);
```

看到了吗？第一个参数是 `This`——是不是跟 C++ 的 `this` 指针一模一样？

### 这就是 C 语言模拟面向对象的标准手法

```
C++:    object.Method(args)     → 编译器自动传 this
UEFI:   Protocol->Method(Protocol, args)  → 手动传 This
```

你在 UEFI 中见到的几乎所有 Protocol 调用都长这样：

```c
// 在屏幕上输出 "Hello UEFI"
gST->ConOut->OutputString(gST->ConOut, L"Hello UEFI\r\n");
//                        ^^^^^^^^^^^
//                        手动传 This 指针
```

---

## 四、从零手搓一个 Protocol

为了彻底理解，我们手搓一个 `MY_CALCULATOR_PROTOCOL`：

### 第 1 步：定义 GUID

```c
// 每个 Protocol 都需要一个全球唯一的 GUID
#define MY_CALCULATOR_PROTOCOL_GUID \
  { 0x12345678, 0xabcd, 0xef01, \
    { 0x23, 0x45, 0x67, 0x89, 0xab, 0xcd, 0xef, 0x01 } }

EFI_GUID gMyCalculatorProtocolGuid = MY_CALCULATOR_PROTOCOL_GUID;
```

### 第 2 步：定义 Protocol 结构体（接口）

```c
// 前向声明
typedef struct _MY_CALCULATOR_PROTOCOL MY_CALCULATOR_PROTOCOL;

// 函数指针类型
typedef
EFI_STATUS
(EFIAPI *MY_CALC_ADD)(
    IN  MY_CALCULATOR_PROTOCOL  *This,
    IN  UINT32                  A,
    IN  UINT32                  B,
    OUT UINT32                  *Result
);

typedef
EFI_STATUS
(EFIAPI *MY_CALC_MULTIPLY)(
    IN  MY_CALCULATOR_PROTOCOL  *This,
    IN  UINT32                  A,
    IN  UINT32                  B,
    OUT UINT32                  *Result
);

// Protocol 结构体 = 函数指针表
struct _MY_CALCULATOR_PROTOCOL {
    MY_CALC_ADD       Add;
    MY_CALC_MULTIPLY  Multiply;
    UINT32            Version;  // 也可以放数据成员
};
```

### 第 3 步：实现具体功能（驱动端）

```c
// 具体的加法实现
EFI_STATUS
EFIAPI
MyCalcAdd(
    IN  MY_CALCULATOR_PROTOCOL  *This,
    IN  UINT32                  A,
    IN  UINT32                  B,
    OUT UINT32                  *Result
)
{
    *Result = A + B;
    return EFI_SUCCESS;
}

// 具体的乘法实现
EFI_STATUS
EFIAPI
MyCalcMultiply(
    IN  MY_CALCULATOR_PROTOCOL  *This,
    IN  UINT32                  A,
    IN  UINT32                  B,
    OUT UINT32                  *Result
)
{
    *Result = A * B;
    return EFI_SUCCESS;
}

// 创建 Protocol 实例
MY_CALCULATOR_PROTOCOL mMyCalcProtocol = {
    .Add      = MyCalcAdd,
    .Multiply = MyCalcMultiply,
    .Version  = 1
};
```

### 第 4 步：安装到 Handle（注册服务）

```c
EFI_STATUS
EFIAPI
MyCalcDriverEntryPoint(
    IN EFI_HANDLE        ImageHandle,
    IN EFI_SYSTEM_TABLE  *SystemTable
)
{
    EFI_STATUS  Status;
    EFI_HANDLE  Handle = NULL;

    // 安装 Protocol 到一个新 Handle
    Status = gBS->InstallProtocolInterface(
        &Handle,
        &gMyCalculatorProtocolGuid,
        EFI_NATIVE_INTERFACE,
        &mMyCalcProtocol
    );

    return Status;
}
```

### 第 5 步：其他模块使用（消费服务）

```c
EFI_STATUS UseCalculator(VOID)
{
    EFI_STATUS                Status;
    MY_CALCULATOR_PROTOCOL    *Calc;
    UINT32                    Result;

    // 在系统中找到 Calculator Protocol
    Status = gBS->LocateProtocol(
        &gMyCalculatorProtocolGuid,
        NULL,
        (VOID **)&Calc
    );
    if (EFI_ERROR(Status)) {
        Print(L"找不到计算器 Protocol！\n");
        return Status;
    }

    // 使用计算器
    Calc->Add(Calc, 10, 20, &Result);
    Print(L"10 + 20 = %d\n", Result);   // 输出：10 + 20 = 30

    Calc->Multiply(Calc, 6, 7, &Result);
    Print(L"6 * 7 = %d\n", Result);     // 输出：6 * 7 = 42

    return EFI_SUCCESS;
}
```

---

## 五、用图来总结整个流程

```
┌─────────────────────────────────────────────────┐
│                Handle Database                   │
│                                                  │
│  Handle_A ──┬── Calculator Protocol (GUID: xxx)  │
│             │     .Add      = MyCalcAdd          │
│             │     .Multiply = MyCalcMultiply      │
│             │     .Version  = 1                   │
│             └── 其他 Protocol ...                 │
│                                                  │
│  Handle_B ──┬── Block IO Protocol (GUID: yyy)    │
│             └── Device Path Protocol (GUID: zzz) │
│                                                  │
└─────────────────────────────────────────────────┘

调用者：
  gBS->LocateProtocol(&CalcGuid) → 拿到 &mMyCalcProtocol
  Calc->Add(Calc, 10, 20, &Result) → 实际执行 MyCalcAdd()
```

---

## 六、Protocol 与其他语言的对比

| UEFI Protocol（C 语言） | C++ | Java | Go |
|------------------------|-----|------|----|
| 结构体+函数指针 | class + virtual | interface | interface |
| 手动传 This | 自动传 this | 自动传 this | 自动传 receiver |
| GUID 标识 | typeid / RTTI | Class 对象 | reflect.Type |
| InstallProtocol | 构造函数 | new + register | Register() |
| LocateProtocol | 工厂模式/DI容器 | ServiceLoader | inject |

**核心区别**：C++ / Java 的"接口"靠**编译器**实现，UEFI 的 Protocol 靠**手动构建函数指针表**实现。本质一样，形式不同。

---

## 七、常见疑问

### Q1：为什么不直接用 C++ 的 virtual？

UEFI 规范要求用纯 C，原因：
- 固件环境没有 C++ runtime（没有 new、delete、exception）
- 不同编译器的 vtable 布局可能不同，ABI 不兼容
- C 的结构体布局是确定的，跨编译器安全

### Q2：This 指针能省略吗？

不能。因为一个 Protocol 类型可能有多个实例（比如系统里有 3 块硬盘，就有 3 个 Block IO Protocol 实例），This 指针告诉函数"你操作的是哪个设备"。

### Q3：Protocol 的函数指针能是 NULL 吗？

可以，但规范会标注哪些函数是**必须实现**的，哪些是**可选**的。调用前最好检查。

---

## 八、总结

```
Protocol 的本质：
┌──────────────────────────────────┐
│  typedef struct {                │
│      FuncPtr_A  FunctionA;       │  ← 函数指针
│      FuncPtr_B  FunctionB;       │  ← 函数指针  
│      DataType   SomeData;        │  ← 数据成员（可选）
│  } SOME_PROTOCOL;                │
└──────────────────────────────────┘

一句话：Protocol = C 语言手搓的 Interface / 虚函数表
```

| 你以为的 UEFI Protocol | 实际的 UEFI Protocol |
|----------------------|---------------------|
| 很复杂的高级概念 | 结构体 + 函数指针 |
| 需要特殊编译器 | 标准 C 就行 |
| 高深莫测 | 大二 C 语言课水平 |

---

下一篇：**#017 UEFI 的内存模型：为什么固件也要管理内存？**——从没有内存的 PEI 到满血的 DXE，看看固件是怎么一步步获得内存的。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
