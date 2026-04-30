# 什么是 TPM？它和 UEFI 安全启动的关系

> 🔥 UEFI/BSP 开发系列第 029 篇 | 难度：⭐⭐ 进阶入门
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

上一篇我们聊了 UEFI Secure Boot——用数字签名验证启动文件的合法性。

Secure Boot 回答了一个问题：**"这个文件是不是合法的？"**

但它没回答另一个问题：**"这台电脑从开机到现在，中间有没有被篡改过？"**

举个例子：假设有个黑客在你离开的时候，用 U 盘刷了一个修改过的 BIOS，然后把 U 盘拔了。Secure Boot 密钥也被换了。你回来开机——一切看起来正常，但其实整个启动链已经被掉包了。

谁来发现这种"狸猫换太子"？答案就是——**TPM**。

---

## 一、TPM 是什么？

TPM 的全称是 **Trusted Platform Module**（可信平台模块）。

**最简单的理解**：TPM 是一块**焊在主板上的安全芯片**，专门干三件事：

```
TPM 的三大能力：
  ① 度量（Measure）：记录"开机到现在都加载了什么"
  ② 存储（Store）：安全存储密钥、密码等敏感数据
  ③ 证明（Attest）：向外部证明"我这台电脑是干净的"
```

> 💡 TPM 就像一个**不可篡改的黑匣子**——飞机（电脑）每次做了什么操作都记录在黑匣子里，事后可以检查"飞行过程中有没有出问题"。

### TPM 的物理形态

| 形态 | 说明 | 典型场景 |
|------|------|----------|
| **离散 TPM** | 独立芯片，焊在主板上 | 服务器、企业级笔记本 |
| **固件 TPM（fTPM）** | 跑在 CPU 安全区域的软件 TPM | Intel PTT、AMD fTPM、ARM TrustZone |
| **虚拟 TPM（vTPM）** | 虚拟机里的模拟 TPM | 云服务器 |
| **软件 TPM** | 纯软件模拟，不安全 | 开发测试用 |

> 🎮 **Windows 11 事件**：微软要求 Windows 11 必须有 TPM 2.0。当时一大波老电脑用户发现自己没有 TPM，引发了全网"怎么开 TPM"的搜索热潮。其实很多电脑有 fTPM，只是 BIOS 里默认没开。那段时间 BIOS 工程师的存在感突然爆表了——"哥们你帮我看看 BIOS 里 TPM 在哪开？"

---

## 二、度量启动（Measured Boot）：TPM 的核心能力

### 2.1 什么是"度量"？

"度量"（Measure）= **对启动过程中加载的每个组件计算哈希值，并记录到 TPM 中**。

```
度量过程（简化）：
  1. BIOS 固件加载 → 计算 BIOS 的 SHA-256 哈希 → 记录到 TPM
  2. 引导加载器加载 → 计算哈希 → 记录到 TPM
  3. 操作系统内核加载 → 计算哈希 → 记录到 TPM
  4. 驱动加载 → 计算哈希 → 记录到 TPM

TPM 里就有了一份"启动日记"：
  "先加载了 BIOS v1.2，然后是 GRUB 2.06，然后是 Linux 5.15..."
```

### 2.2 PCR 寄存器：TPM 的"记事本"

TPM 内部有一组特殊的寄存器叫 **PCR（Platform Configuration Register）**，用来记录度量值：

```
PCR 寄存器（TPM 2.0 至少有 24 个）：

PCR[0]  = 固件代码（BIOS/UEFI）的度量值
PCR[1]  = 固件配置（BIOS 设置）的度量值
PCR[2]  = Option ROM 的度量值
PCR[3]  = Option ROM 配置的度量值
PCR[4]  = MBR/引导加载器的度量值
PCR[5]  = 引导加载器配置的度量值
PCR[6]  = 状态转换和唤醒事件
PCR[7]  = Secure Boot 策略的度量值
PCR[8-15] = 由操作系统使用
PCR[16] = 调试用
PCR[17-22] = 由 Intel TXT/AMD SEV 使用
PCR[23] = 应用支持
```

### 2.3 PCR 扩展（Extend）操作

PCR 的更新不是简单的覆盖，而是 **扩展（Extend）**：

```
PCR 扩展公式：
  PCR_new = Hash(PCR_old || 新度量值)

解释：
  PCR 的新值 = 旧值和新度量值拼接后再哈希

为什么要这样？
  → 保证 PCR 值是所有度量的"累积哈希"
  → 任何一个中间步骤不同，最终 PCR 值就不同
  → 无法伪造——你不能"跳过"某个度量步骤来凑出想要的 PCR 值
```

```
具体例子：

开机 → PCR[0] = 0000...0000（初始值全零）

度量 BIOS：
  BIOS 哈希 = SHA256(BIOS) = AAAA...
  PCR[0] = SHA256(0000...0000 || AAAA...) = BBBB...

度量 Option ROM：
  ROM 哈希 = SHA256(OptionROM) = CCCC...
  PCR[2] = SHA256(0000...0000 || CCCC...) = DDDD...

最终，PCR[0] = BBBB...，PCR[2] = DDDD...

如果有人篡改了 BIOS：
  BIOS' 哈希 = SHA256(修改后的BIOS) = EEEE...（不等于 AAAA）
  PCR[0] = SHA256(0000...0000 || EEEE...) = FFFF...（不等于 BBBB）
  → PCR[0] 的值变了 → 被发现了！
```

> 💡 PCR 扩展就像滚雪球——你没法把雪球拆开回到之前的状态，也没法跳过某一圈雪来伪造最终大小。

---

## 三、Secure Boot vs Measured Boot

很多人把 Secure Boot 和 Measured Boot 搞混了。它们是**互补的两套机制**：

| | Secure Boot | Measured Boot |
|---|---|---|
| **问这个问题** | "这个文件合法吗？" | "从开机到现在过程正常吗？" |
| **做法** | 验证数字签名 | 计算哈希并记录到 TPM |
| **时机** | 加载前验证 | 加载时记录 |
| **不合法时** | 拒绝加载（主动拦截） | 记录下来，由后续流程判断（被动记录） |
| **依赖硬件** | UEFI 固件中的密钥数据库 | TPM 芯片 |
| **能防什么** | 未签名的恶意代码 | 已签名但被替换的合法代码 |

```
类比：
  Secure Boot = 小区门口保安查证件（没有门禁卡不让进）
  Measured Boot = 小区里的监控摄像头（进来的人都拍下来，事后可以查）

最安全的方案：Secure Boot + Measured Boot 一起用！
  → 保安拦截明显的坏人
  → 摄像头记录所有人的行踪
  → 双重保险
```

---

## 四、TPM 在 UEFI 中的接口

### 4.1 TCG2 Protocol

UEFI 规范中定义了 **EFI_TCG2_PROTOCOL**，用于和 TPM 2.0 交互：

```c
typedef struct _EFI_TCG2_PROTOCOL {
  EFI_TCG2_GET_CAPABILITY            GetCapability;     // 查询 TPM 能力
  EFI_TCG2_GET_EVENT_LOG             GetEventLog;       // 获取事件日志
  EFI_TCG2_HASH_LOG_EXTEND_EVENT     HashLogExtendEvent; // 度量并记录
  EFI_TCG2_SUBMIT_COMMAND            SubmitCommand;     // 发送 TPM 命令
  EFI_TCG2_GET_ACTIVE_PCR_BANKS      GetActivePcrBanks; // 获取活跃的 PCR 算法
  EFI_TCG2_SET_ACTIVE_PCR_BANKS      SetActivePcrBanks; // 设置活跃的 PCR 算法
  EFI_TCG2_GET_RESULT_OF_SET_ACTIVE_PCR_BANKS GetResultOfSetActivePcrBanks;
} EFI_TCG2_PROTOCOL;
```

### 4.2 度量一个文件的代码示例

```c
// 在 UEFI 中度量一个文件到 TPM
EFI_STATUS MeasureFile(VOID *FileBuffer, UINTN FileSize) {
  EFI_TCG2_PROTOCOL *Tcg2;
  EFI_STATUS        Status;
  
  // 获取 TCG2 Protocol
  Status = gBS->LocateProtocol(
    &gEfiTcg2ProtocolGuid,
    NULL,
    (VOID **)&Tcg2
  );
  if (EFI_ERROR(Status)) {
    // 没有 TPM，跳过度量
    return Status;
  }

  // 构造事件
  EFI_TCG2_EVENT *Event;
  UINTN EventSize = sizeof(EFI_TCG2_EVENT) + sizeof("MyBootFile");
  Event = AllocateZeroPool(EventSize);
  Event->Size = EventSize;
  Event->Header.HeaderSize = sizeof(EFI_TCG2_EVENT_HEADER);
  Event->Header.HeaderVersion = EFI_TCG2_EVENT_HEADER_VERSION;
  Event->Header.PCRIndex = 4;  // 记录到 PCR[4]（引导加载器）
  Event->Header.EventType = EV_EFI_BOOT_SERVICES_APPLICATION;
  CopyMem(Event->Event, "MyBootFile", sizeof("MyBootFile"));

  // 度量：计算哈希 + 扩展 PCR + 记录事件日志
  Status = Tcg2->HashLogExtendEvent(
    Tcg2,
    0,                         // Flags
    (EFI_PHYSICAL_ADDRESS)FileBuffer,  // 要度量的数据
    FileSize,                  // 数据大小
    Event                      // 事件描述
  );

  FreePool(Event);
  return Status;
}
```

### 4.3 TCG Event Log（事件日志）

TPM 的 PCR 只是最终哈希值，具体"度量了什么"记录在 **TCG Event Log** 中：

```
Event Log 内容（类似审计日志）：
  ├── Event 1: PCR[0] += Hash(SEC Core), Type=EV_S_CRTM_VERSION
  ├── Event 2: PCR[0] += Hash(PEI Core), Type=EV_EFI_PLATFORM_FIRMWARE_BLOB
  ├── Event 3: PCR[0] += Hash(DXE Core), Type=EV_EFI_PLATFORM_FIRMWARE_BLOB
  ├── Event 4: PCR[4] += Hash(bootmgfw.efi), Type=EV_EFI_BOOT_SERVICES_APPLICATION
  ├── Event 5: PCR[7] += Hash(Secure Boot Policy), Type=EV_EFI_VARIABLE_DRIVER_CONFIG
  └── ...

查看工具：
  Windows: tpm.msc → PCR 值
  Linux:   tpm2_eventlog /sys/kernel/security/tpm0/binary_bios_measurements
```

---

## 五、TPM 的其他能力

### 5.1 密钥存储

TPM 可以生成和存储加密密钥，**私钥永远不离开 TPM 芯片**：

```
典型用途：
  ├── BitLocker 密钥密封（Sealing）
  │   → BitLocker 的加密密钥被 TPM "封印"
  │   → 只有 PCR 值匹配时才解封
  │   → 如果启动链被篡改，PCR 值变化，密钥解封失败
  │   → 磁盘无法解密 → 攻击者什么也拿不到
  │
  ├── Windows Hello 密钥
  │   → 指纹/人脸识别的密钥存在 TPM 里
  │
  └── SSH/TLS 密钥
      → 可以把 SSH 私钥放在 TPM 里
```

### 5.2 密封（Sealing）和解封（Unsealing）

```
密封过程：
  1. 告诉 TPM："把这个数据（如 BitLocker 密钥）密封起来"
  2. TPM 用当前的 PCR 值作为"开锁密码"加密这个数据
  3. 密封后的数据存储在磁盘上

解封过程：
  1. 开机后 TPM 重新度量启动链 → 产生新的 PCR 值
  2. 请求 TPM 解封数据
  3. TPM 对比当前 PCR 值和密封时的 PCR 值
     ├── 相同 → 解封成功 → BitLocker 密钥可用 → 磁盘正常解密
     └── 不同 → 解封失败 → 需要恢复密钥 → 说明启动链被改过
```

> 🔐 这就是为什么你更新 BIOS 后 BitLocker 会要求输入恢复密钥——因为 BIOS 变了，PCR 值变了，TPM 拒绝自动解封。这不是 bug，是安全特性！

### 5.3 远程证明（Remote Attestation）

```
场景：企业服务器要向管理平台证明"我没有被篡改"

流程：
  1. 管理平台发送一个随机数（Nonce）给服务器
  2. 服务器的 TPM 用自己的 AIK（证明身份密钥）对 PCR 值 + Nonce 签名
  3. 签名和 Event Log 一起发送给管理平台
  4. 管理平台验证签名 + 检查 PCR 值是否和预期一致
  5. 一致 → 服务器可信
     不一致 → 服务器可能被篡改 → 发出告警
```

---

## 六、ARM 平台的 TPM

### 6.1 fTPM（固件 TPM）

ARM 平台通常不会焊一个独立的 TPM 芯片，而是用 **fTPM**——在 ARM TrustZone 安全世界里运行的软件 TPM：

```
ARM fTPM 架构：
  
  Normal World（非安全世界）         Secure World（安全世界）
  ┌──────────────────────┐        ┌──────────────────────┐
  │  UEFI / Linux / App  │        │   OP-TEE / QTEE      │
  │                      │        │                      │
  │  TPM 客户端驱动      │──SMC──→│  fTPM TA（Trusted App）│
  │  (发送 TPM 命令)     │        │  (执行 TPM 功能)      │
  └──────────────────────┘        └──────────────────────┘

SMC = Secure Monitor Call（ARM 安全世界调用）
TA = Trusted Application
```

### 6.2 高通平台的 TPM

```
高通 SoC 上的 TPM 方案：
  
  方案一：QTEE 内的 fTPM
    └── 高通自己的 TEE 里运行 fTPM 服务
    └── 通过 QSEE/QTEE API 和 Normal World 通信
  
  方案二：OP-TEE 内的 fTPM
    └── 开源 OP-TEE 里运行 Microsoft 的 fTPM TA
    └── 适用于使用开源 TEE 的方案
  
  方案三：外挂 SPI TPM 芯片
    └── 独立的 TPM 2.0 芯片（如 Infineon SLB 9670）
    └── 通过 SPI 总线连接
```

---

## 七、[x86] vs [ARM] TPM 对比

| 对比维度 | x86 (Intel/AMD) | ARM (高通等) |
|---------|----------------|-------------|
| **主流方案** | Intel PTT / AMD fTPM + 离散 TPM | TrustZone fTPM |
| **硬件基础** | Intel ME / AMD PSP 安全子系统 | ARM TrustZone |
| **UEFI 接口** | EFI_TCG2_PROTOCOL | EFI_TCG2_PROTOCOL（相同） |
| **通信方式** | MMIO / CRB (Command Response Buffer) | SMC 调用到安全世界 |
| **Event Log** | ACPI 表 + 内存中 | 相同 |
| **Secure Boot 配合** | 成熟 | 正在完善 |

---

## 八、面试题

### Q1：Measured Boot 能防止什么攻击？

Measured Boot **不能主动防止**攻击，但能**检测**启动链是否被篡改。具体来说：
- 记录每个启动组件的哈希到 TPM PCR
- 如果攻击者替换了任何启动组件，PCR 值会变化
- 结合密钥密封（Sealing），可以让被篡改的系统无法解密磁盘

### Q2：TPM 的 PCR 扩展为什么要用 Hash(old || new) 而不是直接覆盖？

如果直接覆盖，攻击者可以在最后一步 Extend 一个精心构造的值来"覆盖"之前的篡改。使用 Hash(old || new) 保证了：
- 每次 Extend 都依赖之前的值（链式哈希）
- 无法跳过中间步骤
- 无法回退到之前的值
- 最终 PCR 值唯一对应整个度量序列

### Q3：更新 BIOS 后为什么 BitLocker 要求恢复密钥？

因为 BIOS 更新改变了固件代码，导致 PCR[0]（固件度量值）发生变化。BitLocker 的密钥被 TPM 密封（Seal）时绑定了特定的 PCR 值。PCR 值变了，TPM 拒绝解封密钥，所以需要用恢复密钥手动解密。

---

## 九、总结

```
TPM 知识地图：

是什么？
  → 主板上的安全芯片（或 TrustZone 里的 fTPM）
  → 三大能力：度量、存储、证明

Measured Boot：
  ├── 计算每个启动组件的哈希
  ├── Extend 到 TPM 的 PCR 寄存器
  ├── 记录 Event Log（审计日志）
  └── 和 Secure Boot 互补（不是替代）

PCR 扩展：
  → PCR_new = Hash(PCR_old || 新度量值)
  → 链式哈希，无法伪造

密钥密封（Sealing）：
  ├── TPM 用 PCR 值"封印"密钥
  ├── 启动链不变 → 自动解封 → 正常使用
  └── 启动链被改 → 解封失败 → 攻击者拿不到密钥

平台差异：
  x86: Intel PTT / AMD fTPM / 离散 TPM
  ARM: TrustZone fTPM（OP-TEE TA 或 QTEE）

UEFI 接口：
  → EFI_TCG2_PROTOCOL
  → HashLogExtendEvent()（度量 + 记录 + 扩展一步到位）
```

| 你以为的 TPM | 实际的 TPM |
|-------------|-----------|
| 一个没用的芯片 | 整个安全启动链的信任根 |
| 只有企业才用 | Windows 11 强制要求 |
| 和固件无关 | 固件负责度量启动链并记录到 TPM |
| 开了就安全了 | 需要和 Secure Boot + BitLocker + 远程证明配合 |

---

下一篇：**#030 UEFI GOP：固件里怎么在屏幕上画东西？**——开机时看到的品牌 Logo、BIOS 设置界面、甚至 Windows 的启动动画，都是通过 UEFI GOP 画出来的。这个 Protocol 到底怎么用？

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
