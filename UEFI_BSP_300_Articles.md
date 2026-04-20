# UEFI / EDK2 / BSP 开发系列文章 — 300 篇标题（由浅入深）

> 目标：从零基础到芯片原厂 BSP 开发工程师
> 发布平台：知乎、掘金、CSDN、GitHub

## 发布规则

1. **内容安全**：所有内容仅基于公开可查阅的信息（规范文档、开源代码、官方手册），严禁泄露任何本地项目源代码
2. **风格定位**：讲概念不讲实现细节，紧跟网络热梗、幽默风趣，降低硬核技术的阅读门槛
3. **多平台适配**：每篇文章同时适配知乎（专栏）、掘金（文章）、CSDN（博客）、GitHub（仓库 markdown）
4. **文件存储**：每篇文章创建在当前目录下，文件名格式 `001_UEFI与EDK2最通俗的理解.md`
5. **自动发布**：PostFlow 发掘金+CSDN，知乎手动复制粘贴，GitHub 直接 git push
6. **发布频率**：每天 1 篇，避免触发平台风控

## 自动发布工具方案

### 推荐：PostFlow（掘金/CSDN/知乎/小红书）

| 项目 | 说明 |
|------|------|
| GitHub | https://github.com/JessAI2099/PostFlow |
| 原理 | Playwright 浏览器自动化，模拟真实用户操作 |
| 优点 | 不需要 API Key，只需首次登录一次 |
| 支持平台 | 掘金、CSDN、知乎、小红书 |
| 依赖 | Python 3.8+、playwright、pillow |

```bash
# 安装
pip install playwright pillow
python -m playwright install chromium

# 发布
python publish_all.py -m metadata.json -p juejin,csdn,zhihu
```

### 备选：md-publisher（CSDN/掘金/知乎）

| 项目 | 说明 |
|------|------|
| GitHub | https://github.com/koffuxu/md-publisher |
| 原理 | Playwright + browser-cookie3 自动提取 Cookie |
| 特点 | 直接读取 Markdown 文件，支持封面图 |
| 注意 | 需要 macOS（剪贴板依赖 pyobjc），Windows 需适配 |

> GitHub 发布：直接 git push 到仓库即可，无需额外工具。

---

## 第一阶段：入门篇 — UEFI 与固件基础 (1-30)

1. UEFI与EDK2最通俗的理解
2. 为什么传统BIOS被淘汰了？UEFI的前世今生
3. 开机按下电源键后到底发生了什么？
4. UEFI vs Legacy BIOS：一张图看懂区别
5. 什么是固件？它和软件、硬件的关系
6. UEFI规范到底规定了什么？五分钟读懂核心概念
7. EDK2是什么？为什么它是UEFI开发的事实标准
8. 第一次搭建EDK2开发环境（Windows篇）
9. 第一次搭建EDK2开发环境（Linux篇）
10. 用EDK2编译你的第一个UEFI程序：HelloWorld
11. UEFI Shell是什么？它能干什么？
12. 在QEMU中运行你的第一个UEFI固件
13. UEFI的模块化思想：为什么要把BIOS拆成几百个小模块？
14. 什么是UEFI驱动模型？和操作系统驱动有什么区别？
15. UEFI中的Handle和Protocol：理解UEFI的面向对象思想
16. 从C语言角度理解UEFI Protocol——就是函数指针表
17. UEFI的内存模型：为什么固件也要管理内存？
18. 什么是UEFI变量？它和注册表有什么像？
19. UEFI事件和定时器：固件里的异步机制
20. 理解GUID：UEFI世界的身份证号
21. 什么是FV、FFS、FD？UEFI固件的打包格式
22. 用UEFITool打开一个真实的BIOS镜像看看里面有什么
23. UEFI Secure Boot最直白的解释
24. 什么是TPM？它和UEFI安全启动的关系
25. UEFI GOP：固件里怎么在屏幕上画东西？
26. UEFI网络栈：固件也能上网？
27. 固件工程师的日常工作是什么样的？
28. BSP开发工程师和BIOS工程师有什么区别？
29. 芯片原厂 vs OEM vs ODM：固件开发的三个角色
30. 学习UEFI开发的路线图和推荐资源

## 第二阶段：启动流程深入 (31-60)

31. UEFI启动的四个阶段：SEC→PEI→DXE→BDS 全景图
32. SEC阶段：CPU上电后执行的第一条指令在哪里？
33. Reset Vector揭秘：为什么x86从0xFFFFFFF0开始执行？
34. CAR（Cache As RAM）：没有内存怎么跑C代码？
35. 临时内存栈的建立：SEC阶段的核心任务
36. PEI阶段全解：在没有完整内存前我们能做什么？
37. PEIM是什么？PEI模块的加载与执行
38. PPI（PEI-to-PEI Interface）：PEI阶段的通信机制
39. 内存初始化（MRC）：PEI阶段最关键的一步
40. 什么是FSP？Intel固件支持包的前世今生
41. FSP-T/FSP-M/FSP-S三部曲：各自负责什么？
42. HOB（Hand-Off Block）：PEI如何把数据传给DXE？
43. DXE阶段全解：UEFI真正的主角
44. DXE Dispatcher：几百个驱动的加载顺序是怎么决定的？
45. DXE驱动的Depex依赖表达式：谁先加载谁后加载
46. BDS阶段：找到启动设备并引导操作系统
47. Boot Manager：UEFI如何管理多个启动项？
48. UEFI启动到Windows/Linux：Handoff发生了什么？
49. Runtime Services：OS启动后固件还在干活？
50. SMM（系统管理模式）：固件中最神秘的执行环境
51. SMM的安全隐患与防护：为什么它被称为Ring -2？
52. S3休眠恢复的UEFI流程：为什么不用重新初始化所有硬件？
53. Capsule Update：UEFI固件在线升级的原理
54. Recovery模式：BIOS损坏后怎么自动修复？
55. UEFI启动过程中的每一次握手（Handshake）
56. 为什么UEFI启动比Legacy BIOS快？技术原因分析
57. 并行化启动：现代UEFI如何缩短开机时间
58. Measured Boot与Verified Boot的区别
59. TCG测量启动：每个阶段都在TPM里留下指纹
60. 从一次完整的启动日志学习UEFI启动流程

## 第三阶段：EDK2 开发实战 (61-100)

61. EDK2源码目录结构全解析
62. 理解.DSC文件：它是EDK2项目的总管
63. 理解.DEC文件：包的对外声明
64. 理解.INF文件：模块的身份证
65. 理解.FDF文件：固件镜像的布局图
66. PCD（Platform Configuration Database）：EDK2的全局配置系统
67. FixedAtBuild / PatchableInModule / Dynamic / DynamicEx：四种PCD类型详解
68. 用PCD控制功能开关：比#ifdef更优雅的条件编译
69. Library Class机制：EDK2的依赖注入
70. 为什么同一个Library在不同平台有不同实现？
71. NULL Library Instance：不改代码就能注入功能
72. EDK2的构建系统：从.DSC到最终FD的全过程
73. BaseTools详解：build命令背后做了什么？
74. AutoGen.c和AutoGen.h：EDK2自动生成的秘密
75. 手写一个UEFI DXE驱动：从零到运行
76. 手写一个UEFI PEI模块：在内存初始化前执行
77. UEFI Application开发：写一个磁盘工具
78. 在UEFI Shell中调试你的程序
79. EDK2中的字符串处理：Unicode与ASCII
80. 理解EFI_STATUS返回值体系
81. UEFI中的链表操作：和Linux内核链表的异同
82. BootServicesTable和RuntimeServicesTable详解
83. 如何在EDK2中添加一个全新的Package
84. 如何在EDK2中添加一个Platform
85. Override机制：不修改上游代码的定制方法
86. 多板支持：MultiBoardPkg的设计模式
87. 使用Git管理EDK2代码：submodule与补丁管理
88. EDK2 CI/CD实践：自动化构建与测试
89. EDK2的Python工具链：理解build脚本
90. 为什么EDK2代码风格那么严格？编码规范解读
91. 理解UEFI的Image格式：PE32+和TE格式
92. 固件卷（FV）的内部结构：手动解析一个FV
93. FFS文件系统：UEFI固件中的"文件系统"
94. Section和Compression：固件压缩的原理
95. GUIDed Section：签名和加密的固件段
96. 从构建日志分析编译错误的技巧
97. EDK2常见编译错误排查手册
98. 如何给EDK2社区提交补丁
99. EDK2的单元测试框架：UnitTestFrameworkPkg
100. 用QEMU+GDB调试EDK2固件：断点打在固件里

## 第四阶段：ACPI 与电源管理 (101-130)

101. ACPI是什么？为什么固件工程师必须懂它？
102. ACPI规范的核心概念：一文入门
103. ACPI表的种类：DSDT、SSDT、FADT、MADT...
104. ASL（ACPI Source Language）入门：固件里的"脚本语言"
105. 用iasl编译和反编译ACPI表
106. 从一台真实电脑导出ACPI表并分析
107. ACPI设备树：操作系统如何知道有哪些硬件？
108. _HID和_CID：ACPI设备的识别码
109. ACPI方法（Method）：固件里的可执行代码
110. ACPI操作区（OperationRegion）：固件与硬件的桥梁
111. ACPI电源状态：S0/S1/S3/S4/S5 全解
112. CPU C-states详解：处理器怎么省电？
113. CPU P-states详解：动态调频的固件基础
114. ACPI Thermal Zone：温度管理的固件实现
115. DPTF（动态平台热管理框架）深入解析
116. EC（嵌入式控制器）与ACPI的交互
117. QEvent：EC事件在ACPI中的处理
118. 电池管理的ACPI实现：_BIF/_BST/_BIX
119. AC适配器检测：_PSR方法详解
120. ACPI GPE（通用事件）机制
121. 背光控制的ACPI实现
122. 风扇控制策略的ACPI实现
123. Modern Standby（S0ix）：新型低功耗待机
124. S0ix vs S3：为什么新电脑不用传统休眠了？
125. 低功耗空闲（LPI）的ACPI描述
126. 用Windows ACPI调试工具排查问题
127. Linux下的ACPI调试：/sys/firmware/acpi
128. ACPI调试神器：AcpiExec的使用
129. 写一个自定义ACPI设备驱动（Windows WDF篇）
130. ACPI表的热修复：SSDT overlay技术

## 第五阶段：硅平台 BSP 开发 (131-170)

131. 什么是BSP（Board Support Package）？
132. 芯片原厂BSP工程师的核心职责
133. Intel硅平台代码结构全解（以Panther Lake为例）
134. AMD硅平台代码结构全解（以Genoa为例）
135. 高通ARM平台的UEFI适配
136. Intel FSP的架构设计与接口规范
137. FSP UPD（Updatable Product Data）：如何配置FSP
138. FSP Integration Guide阅读指南
139. 从FSP二进制到源码：理解FSP的两种交付模式
140. Intel Reference Code的分层：IP层、硅层、平台层
141. PCH初始化流程：南桥的启动序列
142. GPIO配置详解：引脚复用的固件设置
143. GPIO Pad的电气属性配置
144. SerialIO初始化：UART/SPI/I2C的固件配置
145. PCI Express初始化：Link Training的固件实现
146. DMI链路初始化：CPU到PCH的高速通道
147. 内存初始化深入：DDR5 Training流程
148. LPDDR5 vs DDR5：内存初始化的差异
149. Intel MRC（Memory Reference Code）解析
150. SPD读取与内存拓扑识别
151. 集成显卡初始化：GOP驱动的加载流程
152. Display Port / HDMI初始化的固件实现
153. 音频子系统（HDA/SoundWire）的固件初始化
154. Intel ME（管理引擎）：固件中的固件
155. CSME初始化流程与HECI通信
156. PMC（电源管理控制器）固件详解
157. ISH（集成传感器Hub）固件与驱动
158. CNVi（集成WiFi）的固件初始化
159. Thunderbolt/USB4初始化的固件实现
160. IOM（IO管理器）固件的作用
161. VT-d（虚拟化直通）的固件配置
162. Intel TXT（可信执行技术）的BSP配置
163. Boot Guard：验证启动的硅层实现
164. 微码（Microcode）更新的固件实现
165. Power Gating与Clock Gating的固件配置
166. 硅平台的Power Delivery Network设计
167. 硅平台errata（勘误）的固件补丁方法
168. Bring-up日记：新芯片点亮的第一天
169. 硅平台ES/QS/PV各阶段的BSP工作重点
170. 从一颗新芯片到量产BIOS：BSP开发全流程

## 第六阶段：驱动与协议开发 (171-200)

171. UEFI驱动模型详解：驱动绑定协议（Driver Binding Protocol）
172. Component Name Protocol：给驱动取个好名字
173. UEFI总线驱动 vs 设备驱动 vs 混合驱动
174. PCI驱动开发入门：枚举与资源分配
175. PCI Express高级特性的固件配置（AER/ACS/ARI）
176. USB驱动开发：XHCI控制器初始化
177. USB设备驱动：键盘鼠标的固件实现
178. 存储驱动开发：NVMe初始化全流程
179. AHCI/SATA驱动的固件实现
180. UFS存储驱动开发
181. 网络驱动开发：UNDI和SNP协议
182. 串口驱动开发：调试输出的基础
183. I2C驱动框架在UEFI中的实现
184. SPI Flash驱动：固件存储的读写
185. eMMC驱动开发
186. 图形输出协议（GOP）驱动的工作原理
187. Console驱动：文本输入输出的底层
188. Block IO协议：存储抽象层
189. File System协议：固件中读文件
190. Simple Text Input/Output协议详解
191. 变量驱动（Variable DXE）：NV存储的实现
192. Flash Map与Flash Region管理
193. Watchdog Timer驱动的实现
194. RTC（实时时钟）驱动
195. HECI驱动：与Intel ME通信
196. IPMI驱动：服务器BMC通信
197. CapsuleUpdate驱动：固件在线升级的实现
198. DXE SMM驱动：系统管理模式的驱动开发
199. UEFI HII（Human Interface Infrastructure）：Setup菜单的实现
200. VFR（Visual Form Representation）：BIOS设置界面的描述语言

## 第七阶段：安全与可信计算 (201-230)

201. 固件安全威胁模型：攻击面在哪里？
202. UEFI Secure Boot深入：密钥链与签名验证
203. 自定义Secure Boot密钥：PK/KEK/db/dbx
204. Secure Boot在Linux发行版中的实现
205. Intel Boot Guard深入：OTP与ACM
206. Verified Boot vs Measured Boot vs Secure Boot
207. TPM 2.0在UEFI中的实现：Tcg2Protocol
208. PCR（Platform Configuration Register）：测量了什么？
209. 远程证明（Remote Attestation）的固件基础
210. Intel TXT（可信执行技术）固件配置详解
211. BIOS Write Protection的多层防护
212. SPI Flash保护：PRR/FLOCKDN/BIOS_CNTL
213. SMM安全：SMRAM保护与SMM通信缓冲区验证
214. 固件中的缓冲区溢出：真实漏洞案例分析
215. TOCTOU攻击在固件中的体现与防御
216. Capsule Update的安全性：签名与回滚保护
217. Intel CSME安全架构与固件工程师的关系
218. AMD PSP（Platform Security Processor）简介
219. ARM TrustZone在UEFI中的应用
220. CHIPSEC：固件安全审计工具实战
221. UEFI固件逆向工程入门
222. 从CVE学习固件安全：十个经典固件漏洞
223. 固件供应链安全：SBOM与固件透明度
224. Project Cerberus / Caliptra：平台固件完整性
225. NIST SP 800-193：平台固件弹性指南解读
226. 密码学在固件中的应用：RSA/SHA/AES
227. 安全启动链：从CPU微码到操作系统的信任传递
228. 固件隐私：哪些数据不该写入固件？
229. UEFI固件的模糊测试（Fuzzing）
230. 红蓝对抗中的固件安全评估

## 第八阶段：能耗优化与性能调优 (231-260)

231. 为什么能耗优化是芯片原厂BSP工程师的核心竞争力？
232. 现代处理器的功耗组成：动态功耗与静态功耗
233. 电压频率曲线（V-F Curve）与固件的关系
234. DVFS（动态电压频率调节）的固件实现
235. Intel Speed Shift / HWP的固件配置
236. 能效核与性能核：混合架构的固件调度
237. Package C-states（PC2/PC6/PC10）的固件使能
238. S0ix入门：现代待机的固件实现
239. S0ix深入：连接待机与断开待机的区别
240. S0ix Checker：诊断为什么进不了深度睡眠
241. Intel SLP_S0残差分析：谁在阻止低功耗？
242. 功耗墙（PL1/PL2/PL4）的固件配置
243. 散热设计功耗（TDP）与固件的关系
244. RAPL（Running Average Power Limit）固件配置
245. 内存功耗优化：Self-Refresh与Power Down模式
246. PCIe ASPM（活动状态电源管理）的固件配置
247. PCIe L1 Sub-states（L1.1/L1.2）的固件使能
248. USB选择性挂起的固件实现
249. 显示功耗优化：PSR/ALPM/Panel Self Refresh
250. 音频功耗优化：Intel Smart Sound的固件配置
251. WiFi/BT功耗优化的固件层面配置
252. 用Intel SoC Watch测量实际功耗
253. 用Windows能源报告（powercfg）分析固件问题
254. Linux PowerTop在固件调试中的应用
255. BIOS启动时间优化：从20秒到3秒
256. Fast Boot的固件实现策略
257. 固件中的延时优化：消灭不必要的等待
258. 并行初始化：缩短PEI/DXE阶段时间
259. 固件大小优化：压缩、裁剪与按需加载
260. 能耗测试方法论：从实验室到量产的测试流程

## 第九阶段：调试工具与技术 (261-285)

261. 固件调试方法概览：从printf到JTAG
262. 串口调试：固件工程师的第一个调试工具
263. 配置EDK2的调试输出级别
264. StatusCode/PostCode：没有串口时的调试手段
265. Port 80调试卡：读懂POST Code
266. Intel DCI（Direct Connect Interface）调试入门
267. 用Intel System Debugger连接DCI目标
268. ITP-XDP调试器：芯片级的JTAG调试
269. 用GDB调试UEFI固件（源码级）
270. UEFI Shell下的内存/IO/PCI读写命令
271. RU.efi：固件工程师的瑞士军刀
272. Intel FITC（Flash Image Tool）使用指南
273. Binary Configuration Tool：配置Intel ME
274. 用Python解析UEFI固件：uefi-firmware-parser
275. UEFITool的高级用法：搜索、替换、注入
276. ACPI调试：Windows内核调试器中的ACPI
277. 内存问题调试：Training Failure的排查思路
278. PCIe Link Training失败的固件调试
279. 显示不亮的系统性排查方法
280. USB设备不识别的固件层面排查
281. 死机与蓝屏的固件排查思路
282. 固件性能Profiling：找出最慢的模块
283. UEFI固件的日志系统设计
284. 用Simics仿真器调试固件
285. 固件自动化测试框架设计

## 第十阶段：前沿技术与综合实战 (286-300)

286. RISC-V上的UEFI：开源硬件的固件适配
287. ARM服务器的UEFI BSP开发
288. 统一可扩展固件的未来：Universal Scalable Firmware
289. Rust在固件开发中的应用：安全与性能
290. Project Mu：微软的模块化UEFI固件
291. coreboot vs UEFI：两种固件哲学的对比
292. LinuxBoot：用Linux替代DXE阶段
293. 固件即服务（FaaS）：云时代的固件管理
294. CXL（Compute Express Link）的固件支持
295. UCIe（Universal Chiplet Interconnect Express）与固件
296. 从零搭建一个完整的UEFI平台：综合实战（上）
297. 从零搭建一个完整的UEFI平台：综合实战（中）
298. 从零搭建一个完整的UEFI平台：综合实战（下）
299. BSP开发工程师的技术成长路径与职业规划
300. 我理解的芯片原厂能耗优化哲学：硬件思维写固件

---

## 系列文章说明

- **发布节奏**：每日一篇，约 10 个月完成
- **难度曲线**：⭐(入门) → ⭐⭐⭐⭐⭐(专家)
- **配套代码**：GitHub 仓库同步更新示例代码
- **阶段划分**：

| 阶段 | 篇数 | 难度 | 核心目标 |
|------|------|------|----------|
| 第一阶段 | 1-30 | ⭐ | 建立UEFI整体认知 |
| 第二阶段 | 31-60 | ⭐⭐ | 掌握启动流程细节 |
| 第三阶段 | 61-100 | ⭐⭐ | EDK2开发能力 |
| 第四阶段 | 101-130 | ⭐⭐⭐ | ACPI与电源管理 |
| 第五阶段 | 131-170 | ⭐⭐⭐⭐ | 硅平台BSP核心 |
| 第六阶段 | 171-200 | ⭐⭐⭐ | 驱动开发实战 |
| 第七阶段 | 201-230 | ⭐⭐⭐⭐ | 安全与可信计算 |
| 第八阶段 | 231-260 | ⭐⭐⭐⭐⭐ | 能耗优化（核心竞争力）|
| 第九阶段 | 261-285 | ⭐⭐⭐ | 调试工具与技术 |
| 第十阶段 | 286-300 | ⭐⭐⭐⭐⭐ | 前沿技术与综合 |
