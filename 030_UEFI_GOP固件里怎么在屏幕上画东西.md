# UEFI GOP：固件里怎么在屏幕上画东西？

> 🔥 UEFI/BSP 开发系列第 030 篇 | 难度：⭐⭐ 进阶入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇聊了 TPM——启动链的黑匣子。

今天换个轻松点的话题：**开机时屏幕上显示的东西是怎么画出来的？**

你有没有想过：
- 开机时的品牌 Logo（Dell/联想/华硕）是谁画的？
- BIOS 设置界面是谁渲染的？
- Windows 的启动圈圈（那个转啊转的）是谁画的？
- 如果启动出错显示的蓝屏文字是谁输出的？

答案是：**UEFI 固件通过 GOP（Graphics Output Protocol）把像素画到屏幕上的**。

在传统 BIOS 时代，这个活是 VGA BIOS 干的。现在 2025 年了，VGA BIOS 已经入土为安，GOP 是新时代的显示方案。

---

## 一、为什么固件也要显示东西？

```
固件需要显示的场景：
  ├── 开机 Logo（品牌展示，必须有）
  ├── BIOS 设置界面（Setup UI，用户交互）
  ├── 启动设备选择菜单（Boot Menu）
  ├── 安全警告信息（Secure Boot 失败等）
  ├── POST 错误信息（硬件自检失败）
  ├── 操作系统启动画面（Windows/Linux 的早期画面）
  └── UEFI Shell 的文本输出
```

**在操作系统接管 GPU 驱动之前**，屏幕上显示的一切都是固件画的。

---

## 二、GOP 是什么？

GOP = **Graphics Output Protocol**（图形输出协议）。

它是 UEFI 规范定义的一个 Protocol，作用是：**提供一个 framebuffer，让固件代码可以往屏幕上画像素**。

### 2.1 GOP vs VGA BIOS vs UGA

| | VGA BIOS（传统） | UGA（UEFI 早期） | **GOP（现行标准）** |
|---|---|---|---|
| **时代** | 传统 BIOS | UEFI 早期 | UEFI 现行 |
| **接口** | INT 10h 中断 | UGA Protocol | GOP Protocol |
| **分辨率** | 640x480（最高 1024x768） | 较高 | 最高支持 4K+ |
| **色深** | 16 色 / 256 色 | 32 位 | 32 位 |
| **CPU 模式** | 实模式 | 保护模式 | 64 位长模式 |
| **状态** | 已淘汰 | 已废弃 | ✅ 当前标准 |

> 💀 VGA BIOS 的限制有多离谱？它运行在 16 位实模式下，最高分辨率 1024x768，只有 256 色。在 4K 显示器满地走的今天，这简直是数码原始人。

### 2.2 GOP 不是 GPU 驱动！

重要区分：

```
GOP ≠ 完整的 GPU 驱动

GOP 能做的：
  ✅ 设置分辨率和色深
  ✅ 提供 framebuffer 地址
  ✅ 往 framebuffer 写像素
  ✅ BLT 操作（Block Transfer：矩形区域拷贝）

GOP 不能做的：
  ❌ 3D 渲染
  ❌ 硬件加速
  ❌ 视频解码
  ❌ 多显示器管理
  ❌ 电源管理（DPMS）

类比：
  GOP = 画布 + 画笔（能画静态图）
  GPU 驱动 = Photoshop（能做所有事）
```

---

## 三、GOP Protocol 详解

### 3.1 数据结构

```c
typedef struct _EFI_GRAPHICS_OUTPUT_PROTOCOL {
  EFI_GRAPHICS_OUTPUT_PROTOCOL_QUERY_MODE  QueryMode;   // 查询支持的模式
  EFI_GRAPHICS_OUTPUT_PROTOCOL_SET_MODE    SetMode;     // 设置显示模式
  EFI_GRAPHICS_OUTPUT_PROTOCOL_BLT         Blt;         // 块传输（画图）
  EFI_GRAPHICS_OUTPUT_PROTOCOL_MODE        *Mode;       // 当前模式信息
} EFI_GRAPHICS_OUTPUT_PROTOCOL;
```

只有 **4 个成员**——QueryMode、SetMode、Blt、Mode。简洁到令人感动。

### 3.2 Mode 信息

```c
typedef struct {
  UINT32                                 MaxMode;      // 支持的模式总数
  UINT32                                 Mode;         // 当前模式编号
  EFI_GRAPHICS_OUTPUT_MODE_INFORMATION   *Info;        // 当前模式详情
  UINTN                                  SizeOfInfo;
  EFI_PHYSICAL_ADDRESS                   FrameBufferBase; // framebuffer 地址！
  UINTN                                  FrameBufferSize; // framebuffer 大小
} EFI_GRAPHICS_OUTPUT_PROTOCOL_MODE;

typedef struct {
  UINT32                     Version;
  UINT32                     HorizontalResolution;  // 水平分辨率
  UINT32                     VerticalResolution;    // 垂直分辨率
  EFI_GRAPHICS_PIXEL_FORMAT  PixelFormat;           // 像素格式
  EFI_PIXEL_BITMASK          PixelInformation;      // 像素位掩码
  UINT32                     PixelsPerScanLine;     // 每扫描行的像素数
} EFI_GRAPHICS_OUTPUT_MODE_INFORMATION;
```

### 3.3 像素格式

```c
typedef enum {
  PixelRedGreenBlueReserved8BitPerColor,  // RGBX（每通道 8 位，R 在低位）
  PixelBlueGreenRedReserved8BitPerColor,  // BGRX（每通道 8 位，B 在低位）
  PixelBitMask,                           // 自定义位掩码
  PixelBltOnly,                           // 只能通过 Blt 操作，不能直接写 framebuffer
  PixelFormatMax
} EFI_GRAPHICS_PIXEL_FORMAT;
```

> ⚠️ **经典坑**：不同平台的像素格式不一样！有的是 RGB，有的是 BGR。如果你画出来的颜色不对（红色变蓝色），99% 是像素格式搞反了。这个 bug 是新手固件工程师的"成人礼"——几乎人人都会踩一次。

---

## 四、用 GOP 画个东西

### 4.1 获取 GOP Protocol

```c
EFI_GRAPHICS_OUTPUT_PROTOCOL *Gop;
EFI_STATUS Status;

Status = gBS->LocateProtocol(
  &gEfiGraphicsOutputProtocolGuid,
  NULL,
  (VOID **)&Gop
);
if (EFI_ERROR(Status)) {
  Print(L"No GOP found! Running headless.\n");
  return Status;
}

Print(L"Resolution: %dx%d\n",
  Gop->Mode->Info->HorizontalResolution,
  Gop->Mode->Info->VerticalResolution
);
Print(L"FrameBuffer at 0x%lx, size %d bytes\n",
  Gop->Mode->FrameBufferBase,
  Gop->Mode->FrameBufferSize
);
```

### 4.2 直接写 Framebuffer（画一个红色矩形）

```c
// 直接写 framebuffer —— 最暴力的方式
VOID DrawRedRect(
  EFI_GRAPHICS_OUTPUT_PROTOCOL *Gop,
  UINTN X, UINTN Y,
  UINTN Width, UINTN Height
) {
  UINT32 *FrameBuffer = (UINT32 *)(UINTN)Gop->Mode->FrameBufferBase;
  UINT32 PixelsPerLine = Gop->Mode->Info->PixelsPerScanLine;
  
  // 判断像素格式
  UINT32 Red;
  if (Gop->Mode->Info->PixelFormat == PixelRedGreenBlueReserved8BitPerColor) {
    Red = 0x000000FF;  // RGBX: R 在最低字节
  } else {
    Red = 0x00FF0000;  // BGRX: R 在第三字节
  }
  
  for (UINTN row = Y; row < Y + Height; row++) {
    for (UINTN col = X; col < X + Width; col++) {
      FrameBuffer[row * PixelsPerLine + col] = Red;
    }
  }
}
```

### 4.3 用 Blt() 操作（推荐方式）

`Blt()` = Block Transfer，类似 Windows GDI 的 BitBlt：

```c
// 用 Blt 画一个绿色矩形
VOID DrawGreenRectBlt(
  EFI_GRAPHICS_OUTPUT_PROTOCOL *Gop,
  UINTN X, UINTN Y,
  UINTN Width, UINTN Height
) {
  EFI_GRAPHICS_OUTPUT_BLT_PIXEL Green = {0, 255, 0, 0};  // BGR顺序！
  
  Gop->Blt(
    Gop,
    &Green,                    // 源像素
    EfiBltVideoFill,           // 操作类型：填充
    0, 0,                      // 源坐标（填充时忽略）
    X, Y,                      // 目标坐标
    Width, Height,             // 宽高
    0                          // Delta（每行字节数，0 表示紧凑排列）
  );
}
```

### 4.4 Blt 的四种操作

| 操作 | 说明 | 用途 |
|------|------|------|
| `EfiBltVideoFill` | 用单个像素填充矩形区域 | 清屏、画色块 |
| `EfiBltVideoToBltBuffer` | 从屏幕读取到内存 | 截屏、保存背景 |
| `EfiBltBufferToVideo` | 从内存画到屏幕 | 显示图片、文字 |
| `EfiBltVideoToVideo` | 屏幕内区域复制 | 滚动、移动窗口 |

---

## 五、开机 Logo 是怎么显示的？

```
UEFI 开机 Logo 显示流程：

1. DXE 阶段：GOP 驱动加载，初始化显示硬件
2. BDS 阶段：固件从 FV 中找到 Logo 图片
   └── Logo 存储为 BMP 格式，嵌入在固件镜像中
3. 调用 GOP->Blt() 把 Logo 画到屏幕中央
4. BGRT（Boot Graphics Resource Table）ACPI 表记录 Logo 信息
   └── 位置、大小、状态
5. OS 启动时读取 BGRT 表
   └── 在 OS 启动画面中继续显示同一个 Logo
   └── 实现"无缝过渡"效果

这就是为什么 Windows 启动时能显示品牌 Logo
而不是 Windows 自己的 Logo——因为它从 BGRT 读取了固件画的 Logo
```

> 🎨 **有趣的事实**：你在 Windows 电脑上看到的开机品牌 Logo，其实是 BIOS 固件画的，Windows 只是"不擦掉"它而已。所以如果你改了 BIOS 里的 Logo 图片，开机画面就变了——某些极客会把自己的头像或者二次元老婆放上去。

---

## 六、GOP 在不同平台上的实现

### [x86] Intel/AMD 平台

```
x86 平台的 GOP 来源：
  ├── GPU 厂商提供的 UEFI GOP Driver（.efi 文件）
  │   ├── Intel: IntelGopDriver.efi（集成显卡）
  │   ├── AMD: AmdGopDriver.efi
  │   └── NVIDIA: NvGopDriver.efi
  ├── 以 Option ROM（OpROM）形式嵌入 GPU 固件
  └── 或者直接编译进 BIOS 固件镜像

特点：
  → GPU 厂商提供闭源的 GOP 驱动二进制
  → BIOS 厂商（AMI/Insyde/Phoenix）集成到固件中
  → 固件工程师通常不需要自己写 GOP
```

### [ARM] 高通/飞腾/瑞芯微平台

```
ARM 平台的 GOP 挑战：
  ├── 没有独立 GPU 厂商提供 GOP 驱动
  ├── SoC 自带的 Display Controller（如高通 MDP/DPU）
  ├── 需要 BSP 工程师自己写 GOP 驱动！
  └── 参考 Linux 内核的 DRM/KMS 驱动来实现

```

### GOP 驱动实现架构（如果你要写）

```
GOP 驱动的内部结构：

┌──────────────────────────┐
│   EFI_GRAPHICS_OUTPUT_PROTOCOL  │  ← UEFI 标准接口
├──────────────────────────┤
│   GOP Driver Core                │  ← QueryMode/SetMode/Blt 实现
├──────────────────────────┤
│   Display Controller HAL         │  ← 硬件抽象层
│   ├── MDP/DPU 初始化              │
│   ├── 时钟树配置                   │
│   ├── 电源域管理                   │
│   └── Framebuffer 分配             │
├──────────────────────────┤
│   Panel/Interface Driver          │  ← 面板/接口驱动
│   ├── DSI PHY 初始化               │
│   ├── DSI 命令/数据通道配置         │
│   └── Panel 初始化序列              │
└──────────────────────────┘
         ↓
    Display Hardware（MDP → DSI → 面板/显示器）
```

---

## 七、GOP 和 Console 的关系

UEFI 中文本输出有两层：

```
应用层：Print(L"Hello World\n");
    ↓
文本协议：EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL
    ↓ (ConSplitter)
图形协议：EFI_GRAPHICS_OUTPUT_PROTOCOL
    ↓
硬件：Display Controller → 屏幕

过程：
  1. Print() 调用 ConOut（Simple Text Output）
  2. ConOut 内部使用 GOP 把文字渲染成像素
  3. GOP 把像素写入 framebuffer
  4. 显示硬件把 framebuffer 扫描到屏幕
```

### 没有 GOP 的情况

```
如果平台没有 GOP：
  ├── ConOut 退化为串口输出
  ├── Print() 的文字只出现在串口终端
  ├── BIOS Setup UI 无法显示
  ├── 开机 Logo 无法显示
  └── UEFI Shell 只能在串口里用
  
所以 GOP = 从"命令行"到"有屏幕"的关键一步
```

---

## 八、UEFI Shell 中查看 GOP 信息

如果你有一台支持 GOP 的 UEFI 系统，可以在 Shell 中查看：

```
Shell> dh -p GraphicsOutput
  Handle 3F: GraphicsOutput

Shell> mode
  Available modes for current text output device:
    Mode 0: 80 x 25
    Mode 1: 80 x 50
    Mode 2: 100 x 31
    ...
```

---

## 九、面试题

### Q1：GOP 和传统 VGA 的核心区别是什么？

- GOP 运行在 64 位长模式，VGA BIOS 运行在 16 位实模式
- GOP 提供 framebuffer 直接访问，VGA 用 INT 10h 中断
- GOP 支持高分辨率（4K+），VGA 最高 1024x768
- GOP 是 UEFI 标准 Protocol，VGA BIOS 是 x86 专有

### Q2：为什么 ARM 平台的 GOP 驱动比 x86 难做？

x86 平台的 GPU 厂商（Intel/AMD/NVIDIA）会提供预编译的 GOP 驱动二进制。ARM 平台的 Display Controller 是 SoC 自带的，需要 BSP 工程师自己写 GOP 驱动，涉及 Display Controller 寄存器操作、DSI PHY 配置、时钟树管理等底层工作。

### Q3：Blt() 的四种操作分别用在什么场景？

- `VideoFill`：清屏、画色块
- `VideoToBltBuffer`：截屏、保存屏幕内容
- `BufferToVideo`：显示图片（Logo、字体位图）
- `VideoToVideo`：屏幕内复制（滚动文本）

---

## 十、总结

```
GOP 知识地图：

是什么？
  → UEFI 的图形输出协议
  → 提供 framebuffer + Blt 操作
  → 替代了传统 VGA BIOS

四个接口：
  ├── QueryMode() → 查可用分辨率
  ├── SetMode()   → 设置分辨率
  ├── Blt()       → 画图（填充/拷贝）
  └── Mode*       → 当前模式信息（含 framebuffer 地址）

x86 vs ARM：
  x86: GPU 厂商提供闭源 GOP 驱动，集成即可
  ARM: 需要 BSP 工程师自己写！（大活）

开机 Logo 流程：
  GOP 驱动加载 → BMP 解码 → Blt 到屏幕
  → BGRT ACPI 表 → OS 继续显示
```

| 你以为的 GOP | 实际的 GOP |
|-------------|-----------|
| GPU 驱动 | 只是最基础的 framebuffer 访问 |
| 很复杂 | 接口只有 4 个（QueryMode/SetMode/Blt/Mode） |
| x86 和 ARM 一样 | x86 用现成的，ARM 要自己写 |
| 可有可无 | 没有 GOP 就没有屏幕输出 |

---

下一篇：**#031 UEFI 网络栈：固件也能上网？**——PXE 网络启动、HTTP Boot、固件更新下载……固件阶段的网络协议栈是怎么工作的？

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
