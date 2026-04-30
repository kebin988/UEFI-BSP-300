# 031 - UEFI网络栈：固件也能上网？

> **UEFI/EDK2 BSP 开发系列第 31 篇**
> 前一篇：[#030 UEFI GOP：固件里怎么在屏幕上画东西？](030_UEFI_GOP固件里怎么在屏幕上画东西.md)
> 下一篇：[#032 固件工程师的日常工作是什么样的？](032_固件工程师的日常工作是什么样的.md)

---

## 一、固件还能上网？不是开玩笑吧？

能。而且这玩意儿从 UEFI 规范第一版就有了。

想想这些场景：

- **PXE 网络启动**：公司 IT 给 200 台新电脑装 Windows，不可能一台台插 U 盘。插上网线开机，系统镜像从服务器自动下载安装——这就是网络启动
- **HTTP Boot**：PXE 用的 TFTP 协议太老了，新时代直接走 HTTP/HTTPS 下载启动文件
- **iSCSI 启动**：数据中心的服务器连本地硬盘都不需要，直接从网络存储启动
- **远程固件更新**：通过网络推送 BIOS 更新，运维人员不用跑机房
- **BMC/IPMI 带外管理**：服务器关着机也能通过网络管理固件（虽然这个严格说是 BMC 的活）

所以 UEFI 规范说：**固件必须内置完整的网络协议栈**。而且这个网络栈的设计，可比你想象的讲究多了。

---

## 二、UEFI 网络栈全景图

先上一张全景图，从底到顶：

```
┌─────────────────────────────────────────────────────────┐
│                     应用层                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │ PXE Boot │ │HTTP Boot │ │  iSCSI   │ │ DNS 解析 │   │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘   │
│       │             │            │             │         │
│  ┌────┴─────────────┴────────────┴─────────────┴─────┐  │
│  │              传输层 / 应用协议                      │  │
│  │    TCP4 / TCP6    UDP4 / UDP6    HTTP / TLS        │  │
│  └──────────────────────┬────────────────────────────┘  │
│                         │                               │
│  ┌──────────────────────┴────────────────────────────┐  │
│  │              网络层                                │  │
│  │     IP4 / IP6     DHCP4 / DHCP6     ARP / ND      │  │
│  └──────────────────────┬────────────────────────────┘  │
│                         │                               │
│  ┌──────────────────────┴────────────────────────────┐  │
│  │              数据链路层                            │  │
│  │     MNP (Managed Network Protocol)                │  │
│  │     VLAN Config Protocol                          │  │
│  └──────────────────────┬────────────────────────────┘  │
│                         │                               │
│  ┌──────────────────────┴────────────────────────────┐  │
│  │              驱动层                                │  │
│  │     SNP (Simple Network Protocol)                 │  │
│  │     UNDI (Universal Network Device Interface)     │  │
│  └──────────────────────┬────────────────────────────┘  │
│                         │                               │
│  ┌──────────────────────┴────────────────────────────┐  │
│  │              硬件层                                │  │
│  │     网卡硬件 (Intel I210, Realtek RTL8111,        │  │
│  │     高通 EMAC, USB 网卡...)                       │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

是不是看起来很像 TCP/IP 四层模型？没错，UEFI 的网络栈就是一个**精简版的 TCP/IP 协议栈**，在操作系统还没启动的时候就能工作。

---

## 三、从底往上拆解每一层

### 3.1 最底层：SNP（Simple Network Protocol）

SNP 是 UEFI 网络栈的地基。它的作用很简单：**把网卡硬件抽象成统一接口**。

不管你底下是 Intel 千兆网卡、Realtek 百兆网卡、高通 SoC 内置的 EMAC 还是一个 USB 转网口的小玩意儿，上层协议栈看到的都是同一个 SNP 接口。

```c
// SNP 核心接口（简化）
struct _EFI_SIMPLE_NETWORK_PROTOCOL {
    EFI_SIMPLE_NETWORK_START        Start;        // 启动网卡
    EFI_SIMPLE_NETWORK_STOP         Stop;         // 停止网卡
    EFI_SIMPLE_NETWORK_INITIALIZE   Initialize;   // 初始化（分配缓冲区等）
    EFI_SIMPLE_NETWORK_RESET        Reset;        // 重置网卡
    EFI_SIMPLE_NETWORK_SHUTDOWN     Shutdown;     // 关闭
    EFI_SIMPLE_NETWORK_RECEIVE_FILTERS  ReceiveFilters;  // 设置接收过滤
    EFI_SIMPLE_NETWORK_TRANSMIT     Transmit;     // 发送以太网帧
    EFI_SIMPLE_NETWORK_RECEIVE      Receive;      // 接收以太网帧
    EFI_SIMPLE_NETWORK_GET_STATUS   GetStatus;    // 查询网卡状态
    // ...
    EFI_SIMPLE_NETWORK_MODE         *Mode;        // MAC 地址、链路状态等
};
```

**注意**：SNP 工作在**以太网帧级别**——它只负责收发原始的以太网帧（Ethernet Frame），不管 IP、TCP 这些上层协议。

这和 Linux 内核的 `net_device` / `ndo_start_xmit` 是一个层次的东西。

#### UNDI：SNP 的幕后英雄

很多网卡的 UEFI 驱动并不直接实现 SNP，而是通过 **UNDI（Universal Network Device Interface）** 间接提供 SNP。UNDI 是一种 **网卡固件级别的标准接口**，网卡的 Option ROM（或 UEFI Driver）实现 UNDI，然后 EDK2 的 `SnpDxe` 模块把 UNDI 包装成 SNP。

```
网卡硬件 → UNDI 驱动（网卡厂商提供）→ SnpDxe 包装 → SNP Protocol
```

这就像一个适配器模式：网卡厂商只要实现 UNDI，EDK2 自动帮你适配成 SNP。Intel、Realtek、Broadcom 的 UEFI 网卡驱动基本都走这条路。

### 3.2 数据链路层：MNP（Managed Network Protocol）

MNP 在 SNP 之上，提供**共享访问和帧类型分发**。

为什么需要它？因为一块网卡可能同时被多个上层协议使用——IPv4 要用、IPv6 也要用、ARP 也要用。SNP 只有一个收发接口，怎么分发？MNP 来做这件事：

```
MNP 的职责：
├── 允许多个上层协议共享同一个 SNP（网卡）
├── 按以太网帧类型分发：0x0800 → IP4, 0x86DD → IP6, 0x0806 → ARP
├── 管理接收队列
└── 支持 VLAN 标签处理
```

有点像 Linux 内核的 `netif_receive_skb()` + `ptype_all/ptype_base` 分发机制。

### 3.3 网络层：IP4 / IP6 / ARP / DHCP

这一层实现了完整的 IPv4/IPv6 协议栈：

| 协议模块 | 作用 | 对应 EDK2 源码 |
|---------|------|----------------|
| `Ip4Dxe` | IPv4 协议处理、路由、分片/重组 | `NetworkPkg/Ip4Dxe/` |
| `Ip6Dxe` | IPv6 协议处理、邻居发现、SLAAC | `NetworkPkg/Ip6Dxe/` |
| `ArpDxe` | ARP 地址解析 | `NetworkPkg/ArpDxe/` |
| `Dhcp4Dxe` | DHCP v4 客户端 | `NetworkPkg/Dhcp4Dxe/` |
| `Dhcp6Dxe` | DHCP v6 客户端 | `NetworkPkg/Dhcp6Dxe/` |
| `DnsDxe` | DNS 域名解析 | `NetworkPkg/DnsDxe/` |

**IP4 Protocol 的核心接口：**

```c
struct _EFI_IP4_PROTOCOL {
    EFI_IP4_GET_MODE_DATA  GetModeData;   // 获取 IP 配置信息
    EFI_IP4_CONFIGURE      Configure;      // 配置 IP 地址/子网掩码
    EFI_IP4_GROUPS         Groups;         // 加入/离开组播组
    EFI_IP4_ROUTES         Routes;         // 添加/删除路由
    EFI_IP4_TRANSMIT       Transmit;       // 发送 IP 包
    EFI_IP4_RECEIVE        Receive;        // 接收 IP 包
    EFI_IP4_CANCEL         Cancel;         // 取消待处理操作
    EFI_IP4_POLL           Poll;           // 轮询（因为 UEFI 没有中断驱动的网络）
};
```

注意最后一个 `Poll`——UEFI 网络栈是**轮询模式**的，不像 Linux 有 NAPI 中断+轮询混合机制。这意味着 UEFI 的网络性能不会太高，但对于启动阶段的需求（下载几百 MB 的 OS 镜像）完全够用。

### 3.4 传输层：TCP4 / TCP6 / UDP4 / UDP6

```c
// UDP4 Protocol 示例
struct _EFI_UDP4_PROTOCOL {
    EFI_UDP4_GET_MODE_DATA  GetModeData;
    EFI_UDP4_CONFIGURE      Configure;
    EFI_UDP4_GROUPS         Groups;
    EFI_UDP4_TRANSMIT       Transmit;
    EFI_UDP4_RECEIVE        Receive;
    EFI_UDP4_CANCEL         Cancel;
    EFI_UDP4_POLL           Poll;
};
```

TCP 和 UDP 的实现都在 `NetworkPkg/` 下。PXE 启动主要用 UDP（因为 TFTP 基于 UDP），HTTP Boot 则需要 TCP。

### 3.5 应用层：PXE Boot / HTTP Boot / iSCSI

这是最终面向用户的功能：

#### PXE Boot（经典网络启动）

PXE（Preboot eXecution Environment）是最经典的网络启动方式，流程：

```
客户端                                   DHCP 服务器 / TFTP 服务器
   │                                              │
   │── DHCP Discover ───────────────────────────→ │
   │← DHCP Offer (IP + Next-server + 文件名) ───│
   │── DHCP Request ────────────────────────────→ │
   │← DHCP Ack ────────────────────────────────│
   │                                              │
   │── TFTP Read Request (bootloader.efi) ─────→ │
   │← TFTP Data (分块传输) ──────────────────│
   │                                              │
   │   加载并执行 bootloader.efi                  │
```

这个流程 20 多年没变过——PXE 1.0 是 1999 年 Intel 提出的。服务器机房、企业批量部署，PXE 依然是主力。

#### HTTP Boot（新一代网络启动）

PXE 的问题：
- TFTP 没有加密，启动文件在网络上裸奔
- TFTP 性能差，每个包才 512 字节（RFC 2348 可以扩展到 65535，但很多实现不支持）
- 不能穿越公网（TFTP 用 UDP，NAT 不友好）

HTTP Boot 解决了这些问题：

```
客户端                                 DHCP 服务器 / HTTP 服务器
   │                                              │
   │── DHCP (Option 60: HTTPClient) ───────────→ │
   │← DHCP (Option 59: Boot URI) ───────────────│
   │                                              │
   │── HTTP GET https://server/boot.efi ────────→│
   │← HTTP 200 OK (完整文件一次传完) ───────────│
   │                                              │
   │   加载并执行 boot.efi                        │
```

HTTP Boot 从 UEFI 2.5 规范开始支持，EDK2 在 `NetworkPkg/HttpBootDxe/` 实现了完整的 HTTP/HTTPS Boot。

> 🔒 **安全提示**：生产环境一定要用 **HTTPS Boot**，不然启动文件在传输途中被篡改，Secure Boot 白做了。UEFI 的 TLS 实现在 `NetworkPkg/TlsDxe/` 和 `NetworkPkg/TlsAuthConfigDxe/` 中。

---

## 四、EDK2 中的网络模块地图

EDK2 的网络栈全部在 `NetworkPkg/` 目录下，模块非常清晰：

```
NetworkPkg/
├── SnpDxe/              # SNP 包装层（UNDI → SNP）
├── MnpDxe/              # Managed Network Protocol
├── ArpDxe/              # ARP 协议
├── Ip4Dxe/              # IPv4 协议栈
├── Ip6Dxe/              # IPv6 协议栈
├── Dhcp4Dxe/            # DHCP v4 客户端
├── Dhcp6Dxe/            # DHCP v6 客户端
├── Udp4Dxe/             # UDP v4
├── Udp6Dxe/             # UDP v6
├── Tcp4Dxe/             # TCP v4（旧版，已合并）
├── TcpDxe/              # TCP v4/v6 统一实现
├── DnsDxe/              # DNS 解析
├── HttpDxe/             # HTTP 协议
├── HttpBootDxe/         # HTTP Boot 实现
├── HttpUtilitiesDxe/    # HTTP 工具函数
├── TlsDxe/              # TLS 加密
├── TlsAuthConfigDxe/    # TLS 证书配置
├── UefiPxeBcDxe/        # PXE Boot 客户端
├── IScsiDxe/            # iSCSI 启动
├── VlanConfigDxe/       # VLAN 配置
├── WiFiConnectionManagerDxe/  # WiFi 连接管理
└── NetworkPkg.dsc       # 网络包的 DSC 构建描述
```

看到没？从最底层的 SNP 到最顶层的 HTTP Boot、PXE Boot、iSCSI，一个完整的网络协议栈全在这里。

---

## 五、ARM vs x86：网络栈有什么差异？

网络协议栈本身是**平台无关**的——TCP/IP 不管你是 ARM 还是 x86。但在底层网卡驱动这一层，差异就大了：

| 对比项 | x86 平台 | ARM 平台 |
|--------|----------|----------|
| **网卡来源** | 通常是 PCIe 独立网卡或 PCH 集成网卡 | 通常是 SoC 内置 EMAC/GMAC |
| **驱动获取** | 网卡厂商提供 UEFI Option ROM（Intel、Realtek 等） | 芯片原厂在 BSP 中提供 |
| **PXE Boot** | 开箱即用，几乎所有 x86 服务器都支持 | 需要 BSP 支持，部分嵌入式平台不带 |
| **HTTP Boot** | 主流服务器 BIOS 都支持 | 看 BSP 是否编译了 NetworkPkg |
| **WiFi** | 很少在 UEFI 层用 WiFi 启动 | ARM 设备（IoT/嵌入式）可能需要 WiFi Boot |
| **网卡驱动层** | UNDI + SNP（标准化程度高） | 很多直接实现 SNP（跳过 UNDI） |

### 高通平台的网络栈

高通 SoC 通常内置 **EMAC（Ethernet MAC）** 控制器，驱动在 BSP 中提供。以 QCS6490 为例：

```
高通网络栈路径：
  EMAC 硬件 → QcomEmacDxe（高通提供的 SNP 驱动）→ MNP → IP4/IP6 → ...

或者：
  USB 网卡 → USB Network 驱动（AsixDxe 等）→ SNP → MNP → ...
```

很多高通 ARM 开发板没有板载以太网口，这时候通常用 **USB 转以太网适配器**（比如 ASIX AX88772），EDK2 有现成的 USB 网卡驱动。

### Intel 平台的网络栈

Intel 平台通常用 **PCH 集成的 Intel I219/I225** 系列网卡，或者 PCIe 插槽的 **Intel X710/XXV710** 系列。Intel 提供完整的 UEFI 网卡驱动（包含 UNDI），PXE Boot 开箱即用。

```
Intel 网络栈路径：
  I219 硬件 → Intel UNDI 驱动（Option ROM）→ SnpDxe 包装 → SNP → MNP → ...
```

> 🔧 **实际经验**：Intel 服务器主板的 PXE Boot 几乎从来不需要额外配置，但 ARM 嵌入式平台想要 PXE Boot 通常需要自己确认 BSP 有没有编译网络栈。这是两个世界的差异——Intel 默认假设"网络是基础设施"，ARM 嵌入式默认假设"网络是可选功能"。

---

## 六、UEFI 网络 vs Linux 网络：有什么区别？

既然 UEFI 也有 TCP/IP 栈，它和 Linux 的网络栈有什么区别？

| 对比项 | UEFI 网络栈 | Linux 网络栈 |
|--------|------------|-------------|
| **代码量** | ~5 万行（NetworkPkg） | ~100 万行（net/） |
| **工作模式** | **轮询**（Poll） | **中断 + NAPI 混合** |
| **并发连接** | 几个 | 几万到几十万 |
| **性能** | 够用就行（MB/s 级别） | 极致优化（10Gbps+） |
| **功能** | TCP/UDP/IP/ARP/DHCP/DNS/HTTP/TLS | + Netfilter/iptables、TC、eBPF、XDP... |
| **Socket API** | 无（用 Protocol 接口） | 完整的 BSD Socket |
| **生命周期** | 启动阶段用完就丢 | 永久运行 |
| **安全加固** | 基础 TLS | SELinux、AppArmor、nftables... |

**一句话总结**：UEFI 网络栈是一个 **"够用就好"的精简版 TCP/IP 实现**，它的唯一使命就是让固件能联网下载启动文件、做网络配置。一旦操作系统起来了，这套网络栈就退休了。

> 🎮 **类比**：UEFI 网络栈就像游戏里的"新手武器"——能打，但伤害不高。Linux 网络栈是"满级神装"——功能拉满，但你不可能在出生点（固件阶段）就拿到它。

---

## 七、动手实验：在 UEFI Shell 里玩网络

如果你在 QEMU 里运行 OVMF（参见 [#012](012_在QEMU中运行你的第一个UEFI固件.md)），可以这样测试网络：

### 启动 QEMU 并配置网络

```bash
# 使用 QEMU 的用户模式网络（最简单）
qemu-system-x86_64 \
    -bios OVMF.fd \
    -net nic,model=e1000 \
    -net user
```

### 在 UEFI Shell 中测试

```
# 查看网络接口
Shell> ifconfig -l

# 通过 DHCP 获取 IP
Shell> ifconfig -s eth0 dhcp

# 查看 IP 配置
Shell> ifconfig -l

# 测试 HTTP 下载（如果支持）
Shell> httpboot
```

### 用 `ping` 命令（如果有的话）

UEFI Shell 标准并没有 `ping` 命令，但很多 BIOS 厂商会自己加一个。在 EDK2 的 `ShellPkg/` 里也有参考实现。

---

## 八、网络启动的安全隐患

网络启动很方便，但也很危险：

| 攻击方式 | 原理 | 防御手段 |
|---------|------|---------|
| **Rogue DHCP** | 攻击者冒充 DHCP 服务器，返回恶意启动文件地址 | DHCP Snooping、802.1X |
| **TFTP 篡改** | PXE 用的 TFTP 没有加密，中间人篡改启动文件 | 改用 HTTPS Boot |
| **Evil Maid** | 物理访问机器，修改 PXE 配置 | Secure Boot + Measured Boot |
| **DNS 劫持** | 篡改 DNS 响应，让 HTTP Boot 下载恶意文件 | HTTPS + 证书验证 |

这也是为什么 UEFI 2.5 之后大力推 **HTTPS Boot**——用 TLS 加密传输 + 证书验证服务器身份，把 PXE 时代的安全缺口补上。

---

## 九、面试热点 & 工程经验

### Q1："UEFI 网络启动的完整流程是什么？"

```
1. 用户选择网络启动（或在 Boot Order 中网络优先）
2. UEFI 加载网卡 SNP 驱动，初始化网卡硬件
3. PXE/HTTP Boot 模块启动
4. 发送 DHCP 请求，获取 IP 地址和启动服务器信息
5. 通过 TFTP/HTTP 下载启动文件（如 bootmgfw.efi 或 grubx64.efi）
6. 验证启动文件签名（如果 Secure Boot 开启）
7. 加载并执行启动文件，进入 OS Loader
```

### Q2："为什么 UEFI 网络栈用轮询不用中断？"

因为 UEFI 的执行模型是**单线程、事件驱动**的。没有操作系统级别的中断调度器，网卡中断不容易安全地处理。用 `Poll()` 定期检查网卡状态，简单可靠，而且启动阶段的网络性能要求不高。

### Q3："ARM 平台怎么做 PXE Boot？"

1. 确认 BSP 包含了网卡 SNP 驱动（板载 EMAC 或 USB 网卡）
2. 在 DSC 文件中启用 `NetworkPkg` 编译
3. 确认 BDS 阶段的 Boot Option 支持网络启动
4. 配置 DHCP 服务器返回正确的 ARM64 EFI 启动文件（`bootaa64.efi`）

---

## 十、本篇小结

```
┌─────────────────────────────────────────────────────┐
│                UEFI 网络栈核心要点                    │
├─────────────────────────────────────────────────────┤
│ SNP   = 网卡硬件抽象层（类似 Linux net_device）      │
│ UNDI  = 网卡厂商实现的标准接口 → 被包装成 SNP       │
│ MNP   = 多协议共享同一网卡（帧类型分发）             │
│ IP4/6 = 完整的 IPv4/IPv6 协议栈                     │
│ TCP/UDP = 传输层实现                                 │
│ PXE   = 经典网络启动（DHCP + TFTP）                 │
│ HTTP Boot = 新一代网络启动（DHCP + HTTP/HTTPS）     │
│ 轮询模式 = 简单可靠，性能够用就行                    │
│ ARM 和 x86 的差异主要在网卡驱动层，协议栈是通用的   │
└─────────────────────────────────────────────────────┘
```

UEFI 网络栈虽然没有 Linux 网络栈那么强大，但它在"开机还没进系统"的阶段就能联网下载启动文件、做远程部署、搞固件更新——这本身就是一件很酷的事。

> 下一篇：[#032 固件工程师的日常工作是什么样的？](032_固件工程师的日常工作是什么样的.md) —— 从具体技术切换到职业视角，看看固件工程师每天到底在干什么。

---

**参考资料：**
- [UEFI Specification 2.10, Chapter 28: Network Protocols](https://uefi.org/specifications)
- [EDK2 NetworkPkg 源码](https://github.com/tianocore/edk2/tree/master/NetworkPkg)
- [UEFI HTTP Boot 白皮书](https://tianocore-docs.github.io/NetworkPkg/NetworkPkg.html)
- [PXE Specification 2.1 (Intel)](https://www.intel.com/content/www/us/en/products/docs/io/pxe-boot-specification.html)
- [RFC 5970: DHCPv6 Options for Network Boot](https://www.rfc-editor.org/rfc/rfc5970)
