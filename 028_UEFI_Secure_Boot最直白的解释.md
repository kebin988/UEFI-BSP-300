# UEFI Secure Boot 最直白的解释

> 🔥 UEFI/BSP 开发系列第 028 篇 | 难度：⭐⭐ 进阶入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们用 UEFITool 亲手拆开了一个 BIOS 镜像，看到了里面的模块和变量。

今天要聊的话题，你一定不陌生——

> "请先关闭 Secure Boot 再安装 Linux"
> "安全启动验证失败，系统无法启动"

**Secure Boot（安全启动）** 是 UEFI 最受关注、也最容易引发争议的功能之一。Linux 社区曾经因为它吵翻了天，自由软件基金会甚至一度把微软骂上热搜。

但抛开争议，Secure Boot 到底在干什么？为什么需要它？今天用最直白的话讲清楚。

---

## 一、一句话：Secure Boot 在防谁？

**Secure Boot 的目标：确保从固件到操作系统的每一个环节，都没有被恶意篡改。**

想象这个场景：

```
正常启动：
  固件 → 加载 Windows Boot Manager → 加载 Windows 内核 → 正常运行

被攻击的启动：
  固件 → 加载【被植入木马的 Boot Manager】→ 加载 Windows 内核
         ↑
         这个木马在操作系统启动之前就运行了
         杀毒软件根本看不到它（因为杀毒软件还没启动）
```

这种在操作系统启动之前就植入的恶意软件叫 **Bootkit（引导区恶意软件）**。它比普通病毒更恐怖——因为它运行在杀毒软件、甚至操作系统的**下面**，是真正的"上帝视角"。

> 打个比方：普通病毒是潜伏在公司内部的间谍，安保系统能抓到。Bootkit 是在安保系统上班之前就潜入大楼的间谍——安保系统启动时，间谍已经把监控摄像头给关了。

Secure Boot 就是为了对付这种威胁：

```
开启 Secure Boot 后：
  固件 → 验证 Boot Manager 的签名 → 签名有效？
         ├── 有效 → 继续启动
         └── 无效/被篡改 → 拒绝启动 ❌
```

**核心逻辑**：不是我信任的代码，我不运行。

---

## 二、签名验证的原理（不讲数学）

Secure Boot 的底层是**数字签名**。你不需要懂密码学的数学推导，只需要理解这个类比：

```
现实世界：
  公证处给你发一个【公证书】（签名）
  别人拿到你的文件 + 公证书
  → 去公证处验证：这个公证书是不是真的？文件有没有被改过？

Secure Boot：
  微软/OEM 用【私钥】给 Boot Manager 签名
  固件启动时拿到 Boot Manager + 签名
  → 用存在固件里的【公钥】验证：签名是不是真的？代码有没有被改过？
```

关键点：

| 概念 | 类比 | 存在哪里 | 谁持有 |
|------|------|---------|-------|
| 私钥 | 公证处的印章 | 签名方保密 | 微软/OEM |
| 公钥 | 公证处的公开验证号码 | 固件 NV 变量区 | 所有人可见 |
| 签名 | 公证书 | 附在 .efi 文件上 | 随文件分发 |

> 💡 私钥签名，公钥验证。谁有私钥谁说了算。这就是 Secure Boot 权力链的核心。

---

## 三、密钥链：PK → KEK → db/dbx

Secure Boot 的密钥不是一个，而是一整条**信任链**。一共 4 种密钥/数据库：

```
PK（Platform Key）
  ↓ PK 授权谁可以修改 KEK
KEK（Key Exchange Key）
  ↓ KEK 授权谁可以修改 db/dbx
db（Signature Database）  ← 白名单：允许启动的签名
dbx（Forbidden Database） ← 黑名单：禁止启动的签名
```

### PK — 平台密钥（老大）

```
PK = 整个 Secure Boot 的信任根

谁设置的？ → OEM（戴尔、联想、华硕...）在出厂时写入
能干什么？ → 决定谁可以修改 KEK
             修改 PK 本身需要旧 PK 的签名（或者进入 Setup Mode）
通常是？   → OEM 自己的公钥
```

> 打个比方：PK 是公司的**董事长**——他决定谁当总经理（KEK）。

### KEK — 密钥交换密钥（总经理）

```
KEK = 负责管理白名单和黑名单的中间人

谁设置的？ → OEM 出厂写入，通常同时包含 OEM 和微软的公钥
能干什么？ → 修改 db（白名单）和 dbx（黑名单）
通常有？   → 微软的 KEK + OEM 自己的 KEK
```

> KEK 是**总经理**——决定公司允许谁进门（db）和禁止谁进门（dbx）。

### db — 签名数据库（白名单）

```
db = 允许启动的签名列表

里面有什么？
  → 微软的 UEFI 签名证书（Windows Boot Manager 就是用这个签的）
  → 微软的第三方 UEFI 签名证书（shim/GRUB 等 Linux 引导器用的）
  → OEM 自己的证书（某些定制驱动）
  → 特定 .efi 文件的哈希值（精确到个体的白名单）
```

> db 是**门禁白名单**——名单上有你名字（或你老板担保了你），保安才放行。

### dbx — 禁止签名数据库（黑名单）

```
dbx = 禁止启动的签名列表

里面有什么？
  → 已知漏洞的 Boot Manager 的哈希值
  → 被撤销的证书
  → 已被攻破的签名

优先级 > db！如果一个签名同时在 db 和 dbx 中，dbx 说了算——禁止启动。
```

> dbx 是**门禁黑名单**——就算你名字在白名单上，只要上了黑名单，一律不让进。黑名单优先级最高。

### 验证流程

```
固件加载一个 .efi 文件：

Step 1: 检查 dbx（黑名单）
        └── 在黑名单里？→ 拒绝 ❌ 结束

Step 2: 检查 db（白名单）
        ├── 签名证书在 db 中？→ 验证签名 → 通过 ✅
        ├── 文件哈希在 db 中？→ 哈希匹配 → 通过 ✅
        └── 都不在？→ 拒绝 ❌
```

---

## 四、这些密钥存在哪里？

还记得第 018 篇讲的 UEFI 变量吗？**PK、KEK、db、dbx 全都是 UEFI 变量**，存在 SPI Flash 的 NV 变量区里。

```
变量名    GUID                                   类型
PK       EFI_GLOBAL_VARIABLE_GUID               认证变量（需要签名才能修改）
KEK      EFI_GLOBAL_VARIABLE_GUID               认证变量
db       EFI_IMAGE_SECURITY_DATABASE_GUID        认证变量
dbx      EFI_IMAGE_SECURITY_DATABASE_GUID        认证变量
```

上一篇用 UEFITool 打开 BIOS 镜像时，在 NVRAM 区域就能看到这些变量。

> 💡 这就是为什么 Secure Boot 密钥要用"认证变量"——如果谁都能直接修改 db（白名单），那坏人只要把自己的恶意代码签名加进去，Secure Boot 就形同虚设了。

---

## 五、Linux 和 Secure Boot：恩怨情仇

Secure Boot 从诞生之初就和 Linux 社区有过节。核心矛盾是：

```
微软的立场：
  "我们设计 Secure Boot 是为了安全。OEM 必须在出厂时开启它。"
  "如果你想启动其他操作系统，可以关掉 Secure Boot 或者用我们签名的引导器。"

Linux 社区的顾虑：
  "你控制了 PK 和 KEK，等于控制了哪些操作系统能在 PC 上运行。"
  "如果哪天微软决定不给 Linux 签名了怎么办？"
  "这是不是变相锁定了 Windows 垄断？"
```

最终的妥协方案：

### shim — Linux 的"免死金牌"

```
没有 shim 的 Linux 启动：
  固件 → 加载 GRUB → ❌ 签名不在 db 里，被拒！

有 shim 的 Linux 启动：
  固件 → 加载 shim（微软签名 ✅）
         → shim 加载 GRUB（shim 用自己的密钥验证 ✅）
           → GRUB 加载 Linux 内核
```

shim 是一个小小的引导程序，由**微软用第三方 UEFI CA 证书签名**。各大 Linux 发行版（Ubuntu、Fedora、SUSE 等）的 shim 都经过微软签名，所以它们可以在 Secure Boot 开启的状态下正常启动。

> 如果你用的是主流 Linux 发行版，**不需要关闭 Secure Boot**。Ubuntu、Fedora、openSUSE 等都自带 shim，开箱即用。
>
> 需要关闭 Secure Boot 的情况：你在用自编译内核、冷门发行版、或者自签名的 UEFI 驱动。

### MOK — Machine Owner Key

shim 还提供了一个叫 **MOK（Machine Owner Key）** 的机制——让用户可以自己注册信任的签名密钥：

```
你编译了一个自定义内核模块（比如 NVIDIA 驱动）
  → 用自己的密钥签名
  → 通过 mokutil 注册你的公钥到 MOK 列表
  → shim 会在白名单中查找 MOK 密钥
  → 签名匹配 → 加载成功 ✅
```

---

## 六、Setup Mode vs User Mode

Secure Boot 有两种运行模式：

### User Mode（用户模式）— 正常使用

```
PK 已设置 → Secure Boot 完全生效
  → 所有启动的 .efi 都必须通过签名验证
  → 修改 PK/KEK/db/dbx 需要正确的签名
```

这是出厂时的默认状态。

### Setup Mode（设置模式）— 配置阶段

```
PK 被清除 → 进入 Setup Mode
  → Secure Boot 不强制验证签名
  → 可以自由写入 PK/KEK/db/dbx（不需要签名）
  → 设置完 PK 后自动回到 User Mode
```

> 💡 **这也是"重置 Secure Boot 密钥"的原理**：清除 PK → 进入 Setup Mode → 重新写入新的密钥链 → 回到 User Mode。

如果你想完全自定义 Secure Boot（比如只允许启动你自己签名的系统），流程是：

```
1. 进 BIOS 设置 → 清除所有 Secure Boot 密钥
2. 系统进入 Setup Mode
3. 写入你自己的 PK、KEK、db
4. PK 写入后自动切换到 User Mode
5. 现在只有你签名的 .efi 才能启动
```

---

## 七、Secure Boot 在实际启动中的位置

```
电源键按下
  │
  ▼
SEC/PEI 阶段 — 不受 Secure Boot 保护（由 Boot Guard 保护，下一篇讲）
  │
  ▼
DXE 阶段 — DXE 驱动加载
  │           ├── 如果启用了 Secure Boot
  │           │   → 加载 Security Architecture Protocol
  │           │   → 后续所有 .efi 的加载都要过签名验证
  │           └── 如果关闭了 Secure Boot
  │               → 谁都能加载，不检查签名
  │
  ▼
BDS 阶段 — 选择启动设备
  │
  ▼
加载 OS Bootloader（如 bootmgfw.efi 或 shimx64.efi）
  │           ├── Secure Boot 开启 → 验证签名
  │           └── 验证失败 → "Security Violation" 错误
  │
  ▼
OS Bootloader 加载内核
  │
  ▼
操作系统启动
```

> ⚠️ **注意**：Secure Boot 保护的是**从 DXE 开始到 OS Loader 结束**这段过程。SEC 和 PEI 阶段不受 Secure Boot 保护——那段由更底层的 **Intel Boot Guard / AMD PSB / Qualcomm Secure Boot** 负责。

---

## 八、⚠️ 重要澄清：其实有"两层" Secure Boot

> 这是很多刚入门的人最容易混淆的地方。前面六章讲的内容，**只是 Secure Boot 的"上半场"**——也就是 OS 启动的最后一段。**前面还有一段更底层、更关键的"下半场"，也叫 Secure Boot，但完全是另一套机制。**

### 两层 Secure Boot 的对照

|  | **Platform / Silicon Secure Boot**（下半场）| **UEFI Secure Boot**（上半场）|
|---|--------------------------------------------|------------------------------|
| **保护范围** | SoC 上电 → BL1 → BL2 → ... → UEFI 主体加载完 | UEFI DXE → OS Loader → 内核签名验证 |
| **信任根** | **efuse 烧死的 OEM 公钥哈希**（一次性、不可改）| **NV 变量里的 PK 公钥**（可以在 Setup Mode 改）|
| **谁在跑** | SoC 内置 ROM 的 PBL（高通）/ Intel Boot Guard ACM / AMD PSB | UEFI DXE driver（DxeImageVerificationLib）|
| **验签对象** | 高通 MBN 头里的签名链（XBL/AOP/TZ/HYP/imagefv 等）| `bootmgfw.efi` / `winload.efi` / .efi 驱动 |
| **是否能关** | ❌ 无法关闭（efuse 一次性烧写，量产机器永久启用）| ✅ 能。BIOS 设置里有开关，普通用户也能关 |
| **OS 是否感知** | ❌ 完全不感知，OS 看不到这层 | ✅ Windows / Linux 都能查询状态 |
| **典型故障** | "Authentication failure" → 设备砖了，无法进 BIOS | "Security Violation" → 进得去 BIOS，能换启动项 |
| **本系列前 27 篇讲过的** | 高通 MBN 签名 / Sectools / OEM_PK_HASH | PK / KEK / db / dbx |

### 为什么这两层都叫 "Secure Boot"？

因为它们**都遵守同一个原则**：**不是我信任的代码，我不运行**。只是"我"是谁不一样：

```
下半场（Platform Secure Boot）：
  "我" = SoC 硅片本身（efuse 里烧死的根公钥）
  "信任的代码" = OEM 用根私钥签的所有底层固件
  → 验证早于 UEFI 启动，验证失败 = 设备根本起不来

上半场（UEFI Secure Boot）：
  "我" = UEFI BIOS（NV 变量里的 PK）
  "信任的代码" = 微软/OEM/用户签名的 OS 加载器和驱动
  → UEFI 已经跑起来了，只验证后续要加载的 .efi 文件
  → 验证失败 = 进得去 BIOS 设置，但启动不了 OS
```

### 在高通 ARM 平台上的具体链路

```
①(芯片上电)
   PBL 在 SoC 内 ROM 里跑     ← 信任根，从 efuse 读 OEM_PK_HASH
   │
   ▼ 用 OEM_PK_HASH 验 xbl.elf 的 MBN 签名
②  XBL（SBL）开始跑          ← Platform Secure Boot 区段
   │
   ▼ 用同样的根公钥继续验：tz.mbn / hyp.mbn / aop.mbn / imagefv.elf ...
③  UEFI 主体（imagefv.elf）开始跑 ← 这里 Platform Secure Boot 任务完成
   │
   ▼ DXE 阶段加载 SecurityArchProtocol（如果开启 UEFI SecBoot）
④  UEFI Secure Boot 接力      ← 上半场开始
   │
   ▼ 用 PK/KEK/db/dbx 验 bootmgfw.efi、第三方 driver、PCIe OPROM
⑤  Windows OS Loader 加载 OS 内核
```

**①②③ 是下半场（Platform/Silicon Secure Boot）**——本系列前面讲过的 [#021 什么是 BL1/BL2/BL31/BL33](./021_什么是BL1_BL2_BL31_BL33_ARM启动流程.md)、[#022 高通 SBL 与 EDK2 UEFI 的衔接](./022_高通SBL与EDK2_UEFI的衔接.md) 等都属于这一层。

**④⑤ 是上半场（UEFI Secure Boot）**——也就是本篇前六章讲的 PK/KEK/db/dbx 体系。

### Windows OS 里看到的"那个 Secure Boot"是哪一层？

**只是上半场。**

```powershell
# Windows PowerShell 里查 Secure Boot 状态
Confirm-SecureBootUEFI
# 返回 True/False，反映的就是 UEFI Secure Boot（上半场）
```

```cmd
# msinfo32 → 系统摘要 → "安全启动状态"
# 同样只反映上半场
```

**OS 完全看不到下半场**——因为 Platform Secure Boot 在 UEFI 跑起来之前就已经验完所有底层 firmware 了，那时候 OS 还没影。

更进一步：

| 用户场景 | 实际涉及的层 |
|---------|------------|
| BIOS 里关闭 Secure Boot 装 Linux | 只关上半场，下半场永远开着 |
| Windows 设备管理器看到"安全启动 = 已启用" | 上半场 |
| 设备开机黑屏/Authentication failure | 下半场出问题（root cert / efuse 烧错）|
| `bootmgfw.efi` 签名验证失败 | 上半场 |
| 你 OEM 厂家拿到的 `OEM_PK_HASH` 烧入 fuse | 下半场（一次性、决定整台设备命运）|

### 为什么必须有两层？

如果只有上半场（UEFI Secure Boot），**问题在于 UEFI 自己怎么被信任**？UEFI 镜像在 Flash/UFS 里——如果攻击者改了 UEFI 镜像，把 PK 替换成自己的，上半场就形同虚设。

所以需要下半场（Platform Secure Boot）从硅片层面保护 UEFI 镜像本身：

```
信任根永远从【最难篡改的地方】开始：
  efuse（烧一次永远不变） > Flash NV 变量 > 文件签名
  ─────────下半场─────────  ────上半场────
```

> 💡 **一句话总结**：Platform Secure Boot 保护"BIOS 自己"，UEFI Secure Boot 保护"BIOS 之后的 OS 启动"。两者必须都开启，安全启动链才完整。

### 028 这篇文章的定位

本文前六章讲的 PK/KEK/db/dbx，**严格说只是上半场（UEFI Secure Boot）**。这是因为：

1. 这是绝大多数普通用户和应用层开发者唯一会接触到的"Secure Boot"
2. 下半场对每个 SoC 厂商实现都不同（高通 / Intel / AMD / NXP / TI 各有专利机制），需要单独讲
3. 本系列后续会有专门章节讲：[#029 TPM](./029_什么是TPM_它和UEFI安全启动的关系.md)、Intel Boot Guard、Qualcomm Secure Boot 等下半场内容

如果你是 BSP 工程师，**真正烧脑的是下半场**——上半场基本是"配好密钥就能跑"，下半场要处理 efuse 烧写、根证书管理、签名工具链、量产防错等一系列硬件相关问题。

---

## 九、常见疑问

### Q1："Secure Boot 是微软搞的垄断工具吗？"

Secure Boot 是 UEFI Forum 制定的**行业标准**，不是微软独家技术。微软只是规定了"Windows 认证的 PC 必须默认开启 Secure Boot"。UEFI 规范本身要求 x86 平台允许用户**关闭** Secure Boot——也就是说，你有权关掉它。

不过在 ARM 平台（如 Windows RT），微软曾经要求不允许关闭 Secure Boot——这确实引发了很大争议。

### Q2："关闭 Secure Boot 有什么风险？"

关闭后，任何 .efi 文件都能被加载执行，包括恶意软件。对于日常使用的普通用户，建议保持开启。对于开发者和 Linux 用户，可以按需开关。

### Q3："为什么有时候 BIOS 更新后 Secure Boot 被重置了？"

因为 BIOS 更新可能会擦除或重置 NV 变量区（PK/KEK/db/dbx 都存在那里）。更新后 Secure Boot 可能回到 Setup Mode，需要重新设置。

### Q4："我能给自己的 UEFI 应用签名吗？"

可以。流程大概是：

1. 用 OpenSSL 生成一对密钥（私钥 + 公钥证书）
2. 用 `sbsign` 工具给你的 .efi 文件签名
3. 把公钥证书注册到 db 或 MOK 中
4. 签名后的 .efi 就能在 Secure Boot 下正常加载

```bash
# 生成密钥对
openssl req -new -x509 -newkey rsa:2048 -keyout my.key -out my.crt -days 3650 -nodes

# 给 .efi 签名
sbsign --key my.key --cert my.crt --output signed.efi unsigned.efi

# 注册到 MOK（重启后在 shim 的蓝色界面确认）
mokutil --import my.crt
```

---

## 十、总结

```
Secure Boot 核心逻辑：

目的：防止 Bootkit（引导区恶意软件）
原理：数字签名验证（私钥签名，公钥验证）

密钥链：
  PK（平台密钥）→ 信任链的根，OEM 设置
    ↓
  KEK（密钥交换密钥）→ 管理白/黑名单
    ↓
  db（白名单）→ 允许启动的签名/哈希
  dbx（黑名单）→ 禁止启动的签名/哈希（优先级最高）

验证范围：DXE 阶段开始 → OS Loader 加载完毕
         （SEC/PEI 由 Boot Guard 负责）

Linux 兼容：通过 shim + MOK 机制，主流发行版可正常启动

密钥存储：SPI Flash 上的 UEFI 认证变量
```

| 你以为的 Secure Boot | 实际的 Secure Boot |
|---------------------|-------------------|
| 微软搞出来限制 Linux 的 | UEFI Forum 的行业标准，x86 可以关闭 |
| 只有 Windows 能过 | 微软给 Linux shim 签了名，主流发行版都能过 |
| 开了就安全 | 只保护启动阶段，不替代杀毒软件 |
| 没什么用 | 能防住 Bootkit 这类操作系统都看不到的威胁 |

> ⚠️ **再次强调**：本文核心讲的是 **UEFI Secure Boot（上半场）**——也就是 OS-facing 那层、Windows 设备管理器/`Confirm-SecureBootUEFI` 看到的那层。**Platform / Silicon Secure Boot（下半场）**是另一套基于 efuse 的、SoC 厂商各自实现的机制，本文第八章做了对比，详细内容会放在后续 SoC 专题中讲。

---

下一篇：**#029 什么是 TPM？它和 UEFI 安全启动的关系**——TPM（可信平台模块）是另一块安全芯片。Secure Boot 负责"验证"，TPM 负责"记录"。Windows 11 强制要求 TPM 2.0 不是没有原因的。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
