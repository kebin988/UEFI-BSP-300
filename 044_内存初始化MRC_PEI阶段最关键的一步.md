# 内存初始化（MRC）：PEI 阶段最关键的一步

> 🔥 UEFI/BSP 开发系列第 044 篇 | 难度：⭐⭐⭐⭐ 高级
>
> 作者：BSP 开发工程师
>
> 系列目标：300 篇由浅入深，构建完整的 UEFI 固件知识体系

---

## 写在前面

如果让一个 BIOS/BSP 工程师选出 PEI 阶段最重要的一件事，答案只有一个：**内存初始化**。

这件事复杂到什么程度？Intel 的 MRC (Memory Reference Code) 闭源代码量据传有**几十万行 C 代码**。AMD 的也不遑多让。这些代码只对 OEM/ODM 合作伙伴开放，普通人看不到。

今天我们来理解：为什么初始化几条内存条这么复杂？它到底在做什么？

---

## 一、为什么初始化内存这么复杂？

### 你以为的内存初始化：
```
1. 给内存上电 ✓
2. 搞定，可以用了 ✓
```

### 实际的内存初始化：
```
1. 检测有几个通道，每个通道几条 DIMM
2. 读取每条 DIMM 的 SPD 信息
3. 确定 DIMM 的类型、速度、容量、时序参数
4. 配置内存控制器的几百个寄存器
5. 设置电压（VDDQ、VPP 等）
6. 发送 JEDEC 标准初始化序列给 DRAM 芯片
7. 设置 Mode Registers（模式寄存器）
8. 执行 ZQ Calibration（阻抗校准）
9. 执行 Write Leveling（写时序校准）
10. 执行 Read Training（读训练）
11. 执行 Write Training（写训练）
12. 执行 Command/Address Training
13. 执行 Read/Write Eye 测试
14. 执行 RMT (Rank Margin Tool) 验证
15. 处理 ECC 配置
16. 处理 Memory Interleaving（交错）
17. 建立地址映射表（物理地址如何映射到 DIMM/Rank/Bank/Row/Col）
18. 设置 Memory Protection（地址范围保护）
19. 报告可用内存给 PEI Core
20. 保存训练结果（供 S3 恢复使用）
```

> 💡 这还是简化版！真实的 MRC 还要处理各种容错、降频、坏内存颗粒标记等。

---

## 二、关键术语快速了解

| 术语 | 含义 |
|------|------|
| **MRC** | Memory Reference Code（Intel 的内存初始化代码） |
| **SPD** | Serial Presence Detect（DIMM 上的信息 EEPROM） |
| **JEDEC** | 制定 DDR 标准的组织 |
| **DIMM** | 内存条 |
| **Rank** | DIMM 上的一组芯片（一面或两面） |
| **Channel** | 独立的内存通道 |
| **Training** | 时序校准过程 |
| **Eye Diagram** | 信号质量可视化（越大越好） |

---

## 三、SPD 读取：认识你的内存条

每条 DIMM 上都有一个小小的 EEPROM 芯片（SPD），通过 I2C/SMBus 读取：

```
SPD 中的关键信息：
├── 内存类型（DDR4 / DDR5 / LPDDR5）
├── 容量（4GB / 8GB / 16GB / 32GB）
├── 速度等级（DDR4-3200 / DDR5-4800 / DDR5-6400）
├── CAS Latency（CL 值，如 CL22）
├── tRCD / tRP / tRAS 等时序参数
├── 电压要求
├── 组织结构（x4 / x8 / x16）
├── Rank 数量（1R / 2R / 4R）
├── 厂商信息、序列号
└── XMP/EXPO Profile（超频配置）
```

MRC 首先通过 SMBus 读取所有 DIMM 的 SPD：

```c
// 简化的 SPD 读取流程
for (Channel = 0; Channel < MAX_CHANNELS; Channel++) {
    for (Dimm = 0; Dimm < MAX_DIMMS_PER_CHANNEL; Dimm++) {
        Status = SmbusRead(
            SPD_ADDRESS(Channel, Dimm),
            SpdBuffer,
            SPD_SIZE   // DDR4: 512B, DDR5: 1024B
        );
        
        if (Status == SUCCESS) {
            ParseSpd(SpdBuffer, &DimmInfo[Channel][Dimm]);
            // DimmInfo 包含时序、容量、速度等
        } else {
            // 这个槽位没有内存条
            DimmInfo[Channel][Dimm].Present = FALSE;
        }
    }
}
```

---

## 四、内存控制器配置

读完 SPD 后，MRC 需要配置**内存控制器**（Memory Controller，集成在 CPU/SoC 中）：

```
内存控制器需要知道：
├── 有多少通道是活跃的
├── 每个通道的 DIMM 配置
├── 运行频率（3200MT/s? 4800MT/s?）
├── 时序参数（从所有 DIMM 的 SPD 中取最保守的公共值）
├── 地址映射模式
├── 交错 (Interleaving) 策略
└── ECC 开启还是关闭
```

举个例子：如果你插了两条不同速度的内存条：
```
DIMM 0: DDR5-6400, CL32
DIMM 1: DDR5-4800, CL40

MRC 决策：
  频率 = min(6400, 4800) = 4800 MT/s（木桶效应）
  CL = max(32, 40) = CL40 at 4800（取最慢的时序）
```

---

## 五、Memory Training：最复杂也最核心的步骤

### 5.1 为什么需要 Training？

DDR 内存跑在极高频率下（DDR5-6400 = 3200MHz 时钟，数据传输率 6400 MT/s）。在这个速度下：

```
一个时钟周期 = 1/3200MHz ≈ 0.3 纳秒
光在 0.3 纳秒内传播 ≈ 9 厘米

但 CPU 到 DIMM 的走线长度 ≈ 10-15 厘米！
信号传播延迟已经超过一个时钟周期！
```

加上：
- 每条走线长度不同（信号到达时间不同）
- 温度影响信号传播速度
- PCB 制造公差
- 阻抗不匹配导致信号反射

**结论**：必须针对每块板子、每条内存条做**实时校准**。这就是 Training。

### 5.2 Training 的主要步骤

```
┌────────────────────────────────────────────────────┐
│         Memory Training 流程                        │
├────────────────────────────────────────────────────┤
│                                                    │
│  1. Write Leveling                                 │
│     校准 DQS（数据选通信号）与 CK（时钟）的关系    │
│     确保写数据时序正确                              │
│                                                    │
│  2. Read Leveling (Read Training)                  │
│     校准读数据时 DQS 的采样点                       │
│     找到最佳采样窗口中心                            │
│                                                    │
│  3. Write Training                                 │
│     校准写数据时 DQ 与 DQS 的对齐                   │
│     逐个 byte lane 调整                            │
│                                                    │
│  4. Command/Address Training                       │
│     校准命令和地址信号的时序                        │
│                                                    │
│  5. Read/Write Eye Training                        │
│     全面扫描时序和电压的可用范围                    │
│     找到最大的 "Eye" 并对准中心                     │
│                                                    │
│  6. Margin Test (RMT)                              │
│     验证训练结果在各种压力下的余量                  │
│                                                    │
└────────────────────────────────────────────────────┘
```

### 5.3 什么是 Eye Diagram（眼图）？

```
        电压
         ^
    VH   |    ╱╲        ╱╲        ╱╲
         |   ╱  ╲      ╱  ╲      ╱  ╲
         |  ╱    ╲    ╱    ╲    ╱    ╲
         | ╱  ┌───┐  ╱      ╲  ╱      ╲
    Vref |╱───┤Eye├─╱────────╲╱────────╲──── 
         |╲───┤   ├─╲────────╱╲────────╱
         | ╲  └───┘  ╲      ╱  ╲      ╱
         |  ╲    ╱    ╲    ╱    ╲    ╱
    VL   |   ╲  ╱      ╲  ╱      ╲  ╱
         |    ╲╱        ╲╱        ╲╱
         └──────────────────────────────→ 时间
              ↑
         "Eye"（眼睛开口）越大 = 信号越好

Training 的目的：让采样点落在 Eye 的正中心
```

Training 通过不断调整延时（delay）值，扫描出信号的可用范围，然后选择中心点：

```
扫描结果示例（简化）：

Delay:  0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15
Pass?:  F  F  F  P  P  P  P  P  P  P  P  F  F  F  F  F
                  ↑                       ↑
              左边界=3                  右边界=10

最佳 Delay = (3 + 10) / 2 = 6 或 7
Margin（余量）= (10 - 3) / 2 = 3.5 个 delay step

如果 Margin 太小 → 降频或报错
```

---

## 六、Training 数据的保存和恢复

完整的 Memory Training 很耗时（几百毫秒到几秒）。为了加快 S3 唤醒和快速启动，训练结果会被保存：

```
正常冷启动：
  PEI → 完整 Training（耗时）→ 保存结果到 NVRAM/Flash

S3 唤醒：
  PEI → 跳过 Training → 从 NVRAM 恢复保存的参数 → 直接配置
  （快几十倍！）

快速启动（无硬件变更）：
  PEI → 检查 DIMM 是否变化（SPD checksum）
      → 没变 → 恢复保存的参数（快）
      → 变了 → 重新 Training（慢）
```

---

## 七、MRC 的代码架构

```
MRC 典型代码结构（Intel 闭源，此为推测架构）：

MRC/
├── SpdReader/           ← SPD 读取和解析
├── MemoryDetect/        ← DIMM 探测和兼容性检查
├── ControllerConfig/    ← 内存控制器寄存器配置
├── JedecInit/           ← JEDEC 标准初始化序列
├── Training/
│   ├── WriteLeveling/   ← 写均衡
│   ├── ReadTraining/    ← 读训练  
│   ├── WriteTraining/   ← 写训练
│   ├── CmdTraining/     ← 命令地址训练
│   └── MarginTest/      ← 边际测试
├── MemoryMap/           ← 地址映射和交错
├── RAS/                 ← 可靠性（ECC、Patrol Scrub）
├── PowerManagement/     ← 内存节能模式
├── SaveRestore/         ← Training 数据保存恢复
└── MemoryTest/          ← 基本内存自检
```

代码量：Intel MRC 通常 **10-50 万行 C 代码**（因芯片代而异）。

---

## 八、为什么 MRC 是闭源的？

| 原因 | 说明 |
|------|------|
| **竞争核心** | 内存性能直接影响芯片竞争力 |
| **NDA 保护** | 涉及内存控制器内部寄存器细节 |
| **专利** | Training 算法含大量专利 |
| **安全** | 暴露实现细节可能被用于攻击 |
| **复杂性** | 即使开源也很难被第三方正确修改 |

开源替代：
- **coreboot 的 raminit**：部分 Intel 平台有社区逆向的开源 MRC（如 Sandy Bridge、Haswell）
- **AMD 的 AGESA**：部分开源（通过 coreboot），但最新平台仍闭源
- **Intel FSP**：把 MRC 封装为二进制 blob 提供给 OEM

---

## 九、从 BSP 工程师角度看 MRC

作为芯片原厂 BSP 工程师，你可能需要：

```
日常工作：
├── 调试内存兼容性问题（某个厂商的 DIMM 训练不过）
├── 调整 Training 算法参数（提升某个场景的稳定性）
├── 支持新 DIMM 类型（新厂商、新规格）
├── 优化 Training 速度（减少启动时间）
├── 调试 S3 恢复问题（为什么唤醒后内存不稳定？）
├── 处理 RAS 特性（Memory Correctable/Uncorrectable Error）
├── 适配新平台（新 CPU 的内存控制器改了什么？）
└── 与 DRAM 厂商联合调试（三星/海力士/美光）
```

这是 BSP 工程师中**最核心也最难**的方向之一。

---

## 十、总结

```
内存初始化为什么这么复杂？

  物理层面：高频信号（6400 MT/s）+ 走线延迟 + 阻抗匹配
  → 必须做 Training 校准

  软件层面：支持各种 DIMM 组合 + 容错 + 省电 + 快速恢复
  → 几十万行代码

  工程层面：每代 CPU 控制器不同 + 每种内存不同 + 每块板子不同
  → 必须闭源按平台定制
```

| 关键概念 | 说明 |
|---------|------|
| MRC | Intel 的内存初始化代码 |
| SPD | 内存条上的参数 EEPROM |
| Training | 时序校准（Write/Read Leveling） |
| Eye Diagram | 信号质量可视化 |
| S3 Save/Restore | 保存训练结果加速唤醒 |
| 闭源原因 | 竞争核心 + 专利 + 安全 |

> **一句话总结**：MRC 用几十万行代码，把几条看似简单的内存条，在纳秒级精度下训练成能稳定工作的高速存储系统。这是 UEFI 固件中技术含量最高的部分之一。

---

下一篇：**#045 什么是 FSP？Intel 固件支持包的前世今生**——Intel 怎么把 MRC 等复杂初始化代码打包给 OEM 使用。

> 💬 如果这篇文章对你有帮助，欢迎关注本系列。300 篇 UEFI/BSP 系列文章持续更新中。
