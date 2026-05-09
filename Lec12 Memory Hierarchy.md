# CSC3060 Chapter 6: 存储器层次结构 (Memory Hierarchy)


## 一、为什么需要存储器层次结构？

理想情况下我们希望内存又大、又快、又便宜——但现实中这三者不可能同时满足。

**核心矛盾：CPU-Memory Gap（处理器-存储器速度鸿沟）**

CPU 性能每年提升约 60%（每 1.5 年翻倍），而 DRAM 延迟每年只改善约 9%（10 年才翻倍）。这个差距在持续扩大，形成所谓的 **Memory Wall（存储墙）**。

**解决思路**：把存储器做成多层，靠近 CPU 的小而快，远离 CPU 的大而慢。利用 **局部性原理（Principle of Locality）** ，让大部分访问都命中在快的层级上，营造出一种"又大又快又便宜"的幻觉。

---

## 二、两种局部性 (Locality)

原理：多数程序倾向于使用地址与近期调用数据或指令相近或相等的数据

| 类型    | 英文                | 大白话解释                             |
| ----- | ----------------- | --------------------------------- |
| 时间局部性 | Temporal Locality | 你刚用过的数据，过一会大概率还会再用（比如循环变量 `i`）    |
| 空间局部性 | Spatial Locality  | 你用了某个地址的数据，它旁边的数据大概率马上也要用（比如遍历数组） |

**考试常考**：给一段代码，判断它体现了哪种局部性。

---

## 三、存储层次结构总览

从上到下：速度越来越慢，容量越来越大，价格越来越便宜。

```
寄存器 (Registers)          ← 最快，最小，编译器管理
    ↕ 指令操作数
Cache / 本地存储 (Local Memory)  ← 硬件自动管理（或编译器管理）
    ↕ 块/行 (Blocks/Lines)
主存 (Main Memory / DRAM)    ← OS 管理（虚拟内存）
    ↕ 页 (Pages)
磁盘 (Disk / SSD)           ← 最慢，最大
    ↕ 文件 (Files)
磁带 (Tape)
```

**管理方式总结**（考试爱问）：

- 寄存器 ↔ 内存：编译器 / 程序员
- Cache ↔ 内存：**硬件**自动（对程序员透明）
- 内存 ↔ 磁盘：**操作系统**（虚拟内存）/ 程序员（文件 I/O）

---

## 四、RAM 技术：SRAM vs DRAM

### 4.1 对比表（⭐ 必背）

|特性|SRAM（静态RAM）|DRAM（动态RAM）|
|---|---|---|
|每个 bit 的晶体管数|6 或 8 个|**1 个晶体管 + 1 个电容**|
|速度|快（0.5~2.5 ns）|慢（50~70 ns）|
|需要刷新吗？|**不需要**（通电就一直保持）|**需要**（电容会漏电，要定期刷新）|
|密度|低（体积大）|**高**（体积小，能装很多）|
|价格|贵（$1000~$5000/GB）|**便宜**（$3~$20/GB）|
|用途|**Cache**（高速缓存）|**Main Memory**（主存）|

**一句话记忆**：SRAM 快但贵用来做 Cache，DRAM 慢但便宜用来做主存。

### 4.2 DRAM 组织结构

DRAM 芯片内部是一个 $d\times w$ 的 **二维矩阵**，同时引入了 supecell 的概念。
这里的 $d$ 代表芯片内super cell的总个数，$w$ 代表每一个超级单元所包含的比特位数（通常是一个字节，即8位）

地址被分成两半发送：

1. **RAS（Row Access Strobe，行选通）**：先发行地址，选中一整行，复制到 **Row Buffer（行缓冲区）**
2. **CAS（Column Access Strobe，列选通）**：再发列地址，从行缓冲区中选出具体的那个 supercell

**读 DRAM 的过程**（考试可能让你画/描述）：

1. 发 RAS，选中行 → 整行复制到 row buffer
2. 发 CAS，从 row buffer 中选出目标 supercell → 数据送上数据总线
3. 数据写回该行（顺便完成刷新）

### 4.3 Memory Module（内存条）

要组成 64 位数据通路，需要多个 DRAM 芯片并联。例如 8 个 8Mx8 的芯片拼起来 → 64MB 内存条，每次读出 64 位。

---

## 五、总线与内存读写过程

**Bus（总线）**：连接 CPU 和内存的一组并行线路，传输地址、数据和控制信号，常由多个硬件共享。

**读过程**（Load 指令，如 `LW X1, 4(X2)`）：

1. CPU 把地址 A 放到总线上
2. 主存读取地址 A，取出数据 x，放回总线
3. CPU 从总线读取 x，写入寄存器

**写过程**（Store 指令，如 `SW X1, 4(X2)`）：

1. CPU 把地址 A 放到总线上
2. CPU 把数据 y 放到总线上
3. 主存从总线读取 y，写入地址 A

注意，这是理想模型，现实比这个复杂的多

ppt 还讲了一些神秘的先进的 DRAM，但是似乎在 CSAPP 能找到，有点懒得搬运过来

## 补充：Memory Wall (内存墙)

**定义**: 计算机体系结构中，处理器运算速度与主存（DRAM）访问速度之间日益扩大的性能差距。

### 核心应对策略

#### 1. 应对延迟 (Latency)
* **减少绝对延迟 (Reduction)**: 通过增加存储层级或改变计算位置来缩短物理距离。
    * **技术**: Cache, Local Memory, NUMA, PIM (存内处理), CIM (存内计算)
* **隐藏延迟 (Hiding)**: 通过快速切换任务，用计算掩盖访存的等待时间。
    * **技术**: 多线程/超线程 (Multi-/Hyper-threading), WARP 交错执行 (WARP interleaving), 芯片级多线程 (chip-multithreading)

#### 2. 提升带宽 (Bandwidth)
* **内存端带宽 (Memory Bandwidth)**: 增加单位时间内可并行读取的数据量。
    * **技术**: 多 Bank 架构 (Multi-banks), 交叉内存 (Interleaved memory), SDRAM, HBM (高带宽内存)
* **通信端带宽 (Communication Bandwidth)**: 拓宽数据传输的“马路”。
    * **技术**: 加宽总线 (Wider bus), 高速互连网络 (interconnection network)

### 现代应用挑战
* 在 **HPC (高性能计算)** 和 **ML/LLM (机器学习/大语言模型)** 领域，内存性能通常是首要瓶颈。
* GPU 虽然擅长并行处理且拥有高带宽的 HBM，但受限于 **HBM 容量较小**，内外数据传输依然是核心挑战。
* 
## 六、Memory Hierarchy 核心概念（⭐ 重中之重）

**Memory Hierarchy:** 存储器层次结构：在计算机架构中，存储系统(memory system) 按照一定的顺序排列。每一层相对于下一层具有更高的速度和更低的延迟，但容量较小。
### 6.1 一些基本的概念和知识

| 术语    | 英文                   | 含义                                                        |
| ----- | -------------------- | --------------------------------------------------------- |
| 块 / 行 | Block / Line         | Cache 和内存之间数据传输的最小单位（通常 32B 或 64B）比如你在c++中一次直接拿arr\[1-15] |
| 命中    | Hit                  | 要找的数据在上层(Cache) 里，直接用                                     |
| 未命中   | Miss                 | 数据不在 Cache 里，要去下一级取                                       |
| 命中率   | Hit Rate / Hit Ratio | hits / 总访问次数                                              |
| 未命中率  | Miss Rate            | misses / 总访问次数 = 1 − Hit Rate                             |
| 命中时间  | Hit Time             | 访问 Cache 并判断是否命中所需的时间                                     |
| 未命中惩罚 | Miss Penalty         | Miss 时从下一级存储取数据的额外时间                                      |
| 标签    | Tag                  | 地址的高位，用来标识"这个 Cache 行存的是哪块内存的数据"                          |
| 有效位   | Valid Bit            | 1 = 这行里有有效数据，0 = 空的或无效的                                   |
| 脏位    | Dirty Bit            | 1 = 这行数据被修改过但还没写回内存（Write-Back 策略用）                       |

**核心关系：Hit Time << Miss Penalty**,因为 Cache 太快了

#### 1. 结构金字塔 (自顶向下)
* **寄存器 (Registers)**: 速度最快、极小、极贵。由**编译器**管理，搬运单位为**字 (Words)**。
    * *扩展*: 加入 **向量寄存器 (V Registers)** 用于并行处理 (SIMD)，提升吞吐量,执行大规模计算。
* **高速缓存 (Cache)**: 由**硬件控制器**透明管理，搬运单位为**块/行 (Blocks/Lines)**。
* **主存 (Memory)**: 速度适中、容量较大。由**操作系统**管理 (通过虚拟内存)，搬运单位为**页 (Pages)**。
* **磁盘/磁带 (Disk/Tape)**: 速度最慢、极大、极便宜。由**OS/用户**管理，搬运单位为**文件 (Files)**。

#### 2. 核心运作逻辑
* **底层兜底**: 程序的完整代码和海量数据实际储存在**大、慢、便宜**的底层。
* **顶层加速**: 得益于**局部性原理 (Locality)**，CPU 绝大多数的读写操作都命中了小、快、昂贵的顶层。

#### 3. 最终目的 (Why it Works)
* 利用架构的“障眼法”，打造出一个既有底层的超大容量和极低成本，又有顶层极速访问能力的完美存储系统。

### 补充: 存储层级管理机制 (Who Manages What)

* **寄存器 ↔ 主存**: 由 编译器 / 程序员 管理（通过明确的指令，如 `lw`, `sw`）。
* **高速缓存 (Cache) ↔ 主存**: 由 硬件 自动管理（对程序完全透明）。
* **本地内存 (Local Memory) ↔ 主存**: 由 编译器 / 程序员 手动分配与管理。
* **主存 ↔ 磁盘**: 
    * **隐式机制**: 硬件 + 操作系统 联合管理（虚拟内存与缺页调度）。
    * **显式机制**: 程序员手动管理（文件系统的读写操作）。
* **磁盘 ↔ 磁带**: 由 硬件 / 运维人员 / 程序员 管理（通常用于冷数据自动或手动归档备份）。

### Summary

#### 1. 痛点与方案
* **痛点 (内存墙)**: 访存延迟高，数据带宽不足。
* **方案**: 靠近 CPU 部署小容量、高速度的内存。
* **管理流派**:
    * **透明管理**: Caches（硬件自动控制，对程序员不可见）。
    * **显式管理**: Local Memory（如 NVIDIA 共享内存，需写代码手动分配）。

#### 2. 物理介质对比
* **DRAM**: 慢、便宜、高密度 —— 负责提供**大容量 (BIG)**。
* **SRAM**: 快、昂贵、低密度 —— 负责提供**高速度 (FAST)**。

#### 3. 核心原理与终极幻觉
* **支撑点**: 局部性原理（时间局部性 + 空间局部性）。
* **终极目标**: 利用局部性，向用户提供一个**容量和成本媲美 DRAM**，且**访问速度媲美 SRAM** 的完美存储系统。


### 6.2 Cache 设计的四大问题（⭐ 考试框架）

| #   | 问题       | 英文                | 解决方案                       |
| --- | -------- | ----------------- | -------------------------- |
| Q1  | 数据放在哪？   | Block Placement   | 直接映射 / 组相联 / 全相联           |
| Q2  | 怎么找到数据？  | Block Finding     | Tag + Index + Valid Bit    |
| Q3  | 满了替换谁？   | Block Replacement | LRU / Random / FIFO        |
| Q4  | 写的时候怎么办？ | Write Strategy    | Write-Through / Write-Back |

### 6.3 Cache 里面有什么

Cache 里面通常鄂弼划分成更小的单元，称为 **Cache line(缓存行)**
每个 cache line 里面包含
1. **Data Block** 从主存或者下一级缓存中获取的数据
2. **Tag** 包含部分内存地址的唯一标识，用于指示该行当前存储的是主存中的哪一块。
3. **Valid Bit** 一个位，指示该缓存行是有效还是无效。


## 七、Cache 映射方式（⭐ 必考）

### 7.1 三种映射方式

| 方式     | 英文                        | 大白话                                     | 特点                                  |
| ------ | ------------------------- | --------------------------------------- | ----------------------------------- |
| 直接映射   | Direct Mapped             | 每个内存块只能放在 Cache 的固定位置（取模决定）             | 简单快速，但容易冲突，一冲突你就 Cache miss 了，那不就炸了 |
| 全相联    | Fully Associative         | 内存块可以放在 Cache 的任意位置                     | 冲突最少，但查找慢（要比较所有 Tag）                |
| N 路组相联 | N-way **Set Associative** | Cache 分成若干组，每组有 N 行，内存块映射到某组后可放在该组的任意一行 | 折中方案,既有索引，又有任意一行(这不就是分块吗)           |
|        |                           |                                         |                                     |

**重要关系**：

- 直接映射 = **1路**组相联（每组只有 1 行）
- 全相联 = **1组**的组相联（所有行在同一组）
- 一个 2-way 比 4-way 有**更多的组数**（总行数不变，每组行数少了，组就多了）


### Cache 查找核心机制 (Block Finding)

#### 1. 物理地址切片 (Address Slicing)

CPU 访问内存时，硬件会将物理地址直接“切分”为三段：
* **Tag (标记位 - 高位)**：数据块的“唯一身份证”。
* **Set Selection (组索引 - 中位)**：决定数据放在 Cache 的哪一个具体槽位。
* **Byte Offset (字节偏移 - 低位)**：定位到该 Cache 行（Block）内部的具体某一个字节。

#### 2. 身份校验 (Tag)
* **目的**：多个主存块可能会映射到同一个 Cache 槽位（冲突）。通过硬件比对 Tag，才能确认槽位里当前的块是不是 CPU 正在找的那个。

#### 3. 有效位 (Valid Bit)
* **定义**：每个 Cache 槽位额外附加的 1 bit 状态位，用来防止 CPU 读到随机的垃圾电信号。
* **Valid = 1**：槽位内有合法的真实数据。
* **Valid = 0**：槽位为空或数据无效（场景：刚开机冷启动、数据还没搬完、或缓存被强制清空/Flushed）。

> **终极 Hit 条件**：必须同时满足 **Valid Bit == 1** 且 **Cache Tag == Address Tag**，才算真正命中。


### 7.2 地址划分（⭐ 必考计算题）

一个 32 位地址被分成三部分：

```
| Tag (标签) | Set Index (组索引) | Byte Offset (字节偏移) |
```

**计算方法**：

设 Cache 大小 = S，块大小 = B，相联度 = N（N-way）


**组数** (Number of Sets) = S / (B × N)

**Byte Offset** 位数 = log₂(B)

**Set Index** 位数 = log₂(组数) = log₂(S / (B × N))

**target** 位数 = 32 − Set Index 位数 − Byte Offset 位数


### 7.3 地址划分练习（⭐ 考试原题类型）

**例题：16KB Cache，块大小 64B，32 位地址**

| 映射方式                  | 组数           | Offset 位     | Index 位       | Tag 位           |
| --------------------- | ------------ | ------------ | ------------- | --------------- |
| Direct Mapped (1-way) | 16K/64 = 256 | log₂(64) = 6 | log₂(256) = 8 | 32−8−6 = **18** |
| 2-way Set Associative | 256/2 = 128  | 6            | log₂(128) = 7 | 32−7−6 = **19** |
| 4-way Set Associative | 256/4 = 64   | 6            | log₂(64) = 6  | 32−6−6 = **20** |
| Fully Associative     | 1            | 6            | 0             | 32−0−6 = **26** |

**规律：相联度越高 → 组数越少 → Index 位越少 → Tag 位越多 → Tag 开销越大**

### 7.4 Cache 总位数计算（考试常考）

**例题：Direct Mapped，16KB 数据，块大小 64B（16 words），32 位地址**

每行包含：数据 + Tag + Valid Bit（Write-Back 还要加 Dirty Bit）

```
行数 = 16KB / 64B = 256 行
每行的 bit 数 = 64×8 (数据) + 18 (Tag) + 1 (Valid) = 531 bits
总位数 = 256 × 531 = 135,936 bits ≈ 136 Kb
```

如果是 Write-Back，每行再加 1 个 Dirty Bit：

```
每行 = 64×8 + 18 + 1(valid) + 1(dirty) = 532 bits
```

如果有 ECC（纠错码），通常 8 bits ECC per 64 bits data。


> [!summary] Cache Block Size (缓存块大小) 设计权衡
> 在 Cache 总容量固定的前提下，Block Size 的设定需平衡局部性与缺失代价。

- **调大 Block Size 的收益**
  - 利用**空间局部性**，初期可有效降低 Miss Rate。
  - 减少 Cache Line 总数，从而降低 Tag 的存储开销。

- **过大 Block Size 的负面效应**
  - **冲突加剧**：块数减少导致竞争加剧，破坏时间局部性，Miss Rate 反弹。
  - **Miss Penalty 飙升**：带来极高的总线传输延迟，且易引发**缓存污染**（调入大量无用数据）。

- **长延迟的硬件缓解策略**
  - **Early restart (早启动)**：所需数据一到即恢复 CPU 执行。
  - **Critical-word-first (关键字优先)**：优先请求并传输 CPU 当前缺失的具体字（Word）。

- **容量与策略的关联**
  - **Small Caches (如 L1)**：冲突极敏感，Miss Rate 呈 U 型曲线，必须寻找最佳平衡点。
  - **Large Caches (如 L2/L3)**：容量充裕，更能从 larger lines 中获益。

## 八、Cache 读写策略

### 8.1 读操作

- **Read Hit**：直接从 Cache 取数据，正常执行
- **Read Miss**：暂停流水线（Blocking Cache）→ 从下一级取块 → 放入 Cache → 重新执行

### 8.2 写命中策略（⭐ 必背对比）

| 策略  | 英文            | 操作                                 | 优点                  | 缺点                                                     |
| --- | ------------- | ---------------------------------- | ------------------- | ------------------------------------------------------ |
| 写直达 | Write-Through | 同时写 Cache **和**内存                  | 主存和Cache中的数据一致，实现简单 | 慢！每次写都要访问内存                                            |
| 写回  | Write-Back    | 只写 Cache，标记 Dirty Bit；被替换时才从缓存写回内存 | 快，减少内存流量            | 数据不一致风险，需要 Dirty Bit.因为缓存可能在数据持久化到后端存储之前发生故障（从而导致数据丢失） |

**Write-Through 的性能问题和解决**：

如果 base CPI = 1，10% 指令是 store，写内存需要 100 cycles：

```
Effective CPI = 1 + 0.1 × 100 = 11   ← 太慢了！
```

**解决方案：Write Buffer（写缓冲）**。

这是一个队列结构，遵循 FIFO。
CPU 写完 Cache 和 Write Buffer 就继续执行，Write Buffer 在**后台**慢慢写到内存。只有 Write Buffer 满了才 stall。

### 8.3 写未命中策略

| 策略   | 英文                | 操作                                            | 常搭配           |
| ---- | ----------------- | --------------------------------------------- | ------------- |
| 写分配  | Write Allocate    | 先data block从内存调入 Cache，再写<br>赌时间局部性，赌 CPU 还会用 | Write-Back    |
| 不写分配 | No-Write Allocate | 直接写内存，不放进 Cache<br>比如写日志什么的，CPU 大概率不会再用上了就用这个 | Write-Through |
|      |                   |                                               |               |

memset strcpy 这些单向读一大堆数据的，短时间不会用重复数据的，用No-write-allocate

局部变量更新、普通数组元素更新要用 Write-allocate,因为更新了很可能马上要用

**现代 Cache 大多使用 Write-Back + Write Allocate**。

### 8.4 Store Buffer vs Write Buffer

| 组件           | 位置               | 功能                                          |
| ------------ | ---------------- | ------------------------------------------- |
| Store Buffer | CPU 与 D-Cache 之间 | 让 store 指令快速 commit，支持 store-load bypassing |
| Write Buffer | D-Cache 与下一级存储之间 | 异步写回脏行，隐藏写延迟                                |

### 8.5 Blocking Cache vs Non-Blocking Cache

 **Non-Blocking Cache 非阻塞缓存** 常用来解决 Cache miss
 
- **Blocking Cache**：Cache miss 时**整个流水线暂停**（Stall-on-Miss）
- **Non-Blocking Cache（也叫 Lockup-Free Cache）**：Cache miss 时 CPU 继续执行其他无关指令，**只有真正需要那个数据时才暂停**（Stall-on-Use）
    - 用 **MSHR（Miss Status Holding Register，未命中状态保持寄存器）** 跟踪正在处理的 miss
    - 支持 **MLP（Memory Level Parallelism，存储级并行）**
    - 统称 **Stall-on-Use** 两种模式：Hit-under-miss / Miss-under-miss

---

## 九、Cache Miss 的分类：3C / 4C（⭐ 必背）

| 类型    | 英文                        | 原因                             | 例子                           |
| ----- | ------------------------- | ------------------------------ | ---------------------------- |
| 强制缺失  | Compulsory (Cold) Miss    | 数据第一次被访问，Cache 里肯定没有           | 程序刚启动时                       |
| 冲突缺失  | Conflict (Collision) Miss | 多个块映射到同一个组，组不够大，互相挤掉           | 直接映射 Cache 中访问 0, 8, 0, 8... |
| 容量缺失  | Capacity Miss             | Cache 太小，装不下工作集                | 遍历超大数组                       |
| 一致性缺失 | Coherency Miss            | 多处理器中，其他核的写操作导致本核的 Cache 行被无效化 | 多核共享变量                       |

**冲突缺失的产生条件**：两个地址之间的距离是 Cache 大小的整数倍时，它们会映射到同一个组。
常出现在**直接映射**缓存中，可以通过组相联来缓解

---

## 十、替换策略（Block Replacement）

直接映射没得选（只有一个位置）。组相联和全相联需要选择替换谁：

|策略|英文|原理|特点|
|---|---|---|---|
|最近最少使用|LRU|替换最久没被访问的块|最常用，但硬件开销大|
|随机替换|Random|随机选一个替换|简单，有时表现不差|
|先进先出|FIFO|替换最先进入的块|简单，但不一定好|
|最优替换|Belady's / OPT|替换未来最久不会被用到的块|理论最优，**不可实现**（需要未来信息）|

**LRU 实现**：对于 N-way 组，每行维护一个 log₂(N) bit 计数器。被访问时清零，同组其他行计数器+1。替换时选计数器最大的。

**考试陷阱案例**：一个循环有 5 个块但 Cache 只有 4 行时，LRU 会导致 **0% 命中率**（不断驱逐下一轮要用的块），而直接映射反而有 60% 命中率！说明 LRU 不总是最好的。

---

## 十一、Victim Cache（牺牲者缓存）

一个小型的**全相联** Cache（通常 4~8 行），放在直接映射 L1 旁边，专门保存被驱逐出去的行 (L1 cache 和 L2 cache 之间),这个东西的存在就是相当于帮直接映射擦屁股，极大地保留了直接映射的速度

**好处**：结合了直接映射的速度和全相联的低冲突，特别适合处理集中的冲突缺失（如反复访问映射到同一组的几个地址）。

---

## 十二、块大小的权衡 (Block Size Considerations)

- 块越大 → 空间局部性越好 → miss rate 降低 ✅
- 块越大 → Tag 开销越小 ✅
- 但 Cache 大小固定时，块越大 → 块的数量越少 → 冲突增加 ❌
- 块越大 → miss penalty 越大（传输时间长） ❌
- 块越大 → Cache 污染（Pollution），把不需要的数据也搬进来 ❌

**结论**：小 Cache 需要小块，大 Cache 可以用较大的块。需要平衡。

优化手段：**Critical Word First（关键字优先）** 和 **Early Restart（提前重启）** 可以减少传输延迟。

---

## 十三、Cache 性能计算（⭐ 公式必背）

### 13.1 内存停顿周期数

$$\text{Memory Stall Cycles} = \text{I-cache miss cycles} + \text{D-cache miss cycles}$$

$$\text{I-cache miss cycles} = \text{I-cache miss rate} \times \text{Miss Penalty}$$

$$\text{D-cache miss cycles} = \%\text{load/store} \times \text{D-cache miss rate} \times \text{Miss Penalty}$$

### 13.2 实际 CPI

$$\boxed{\text{Actual CPI} = \text{Base CPI} + \text{Memory Stall Cycles per Instruction}}$$

**例题**：Base CPI = 2, I-cache miss rate = 2%, D-cache miss rate = 4%, Miss Penalty = 100 cycles, 36% 指令是 load/store。

```
I-cache stall = 0.02 × 100 = 2
D-cache stall = 0.36 × 0.04 × 100 = 1.44
Actual CPI = 2 + 2 + 1.44 = 5.44
完美 Cache 快多少倍 = 5.44 / 2 = 2.72 倍
```

### 13.3 AMAT（平均存储访问时间）（⭐ 最核心公式）

$$\boxed{\text{AMAT} = \text{Hit Time} + \text{Miss Rate} \times \text{Miss Penalty}}$$
$$
\boxed{\text{Overall AMAT}
= \% \text{instr} \times (\text{Instruction cache AMAT})
+ \% \text{data} \times (\text{Data cache AMAT})
}$$

**例题**：1ns 时钟，Hit Time = 1 cycle, Miss Penalty = 20 cycles, Miss Rate = 5%

```
AMAT = 1 + 0.05 × 20 = 2 ns = 2 cycles per access
```

**注意**：当 Cache 访问可以被重叠时（Non-Blocking Cache、OOO 处理器、Prefetching），AMAT 公式**不再准确**。

### 补充：多级缓存机制 (Multilevel Caches)

* **L1 Cache (一级缓存 / Primary Cache)**
    * **位置**：紧贴 CPU 核心。
    * **特点**：容量极小，速度极快 (Small and fast)。
    * 其更看重空间局部性 spatial locality, 常采用 write through策略
* **L2 Cache (二级缓存)**
    * **作用**：专门接收并处理 L1 中未命中 (Miss) 的数据请求。
    * **特点**：容量比 L1 大，速度比 L1 慢，但依然远快于主存 (Main Memory)。
    * 其更在意时间局部性 temporal locality，常采用 write back 策略
    * 
* **L3 Cache (三级缓存)**
    * **场景**：通常配备于高端计算机系统中。
    * **特点**：通常在同一芯片上由**多个 CPU 核心共享 (Shared by multiple cores)**。
* **⚠️ 核心设计挑战**
    * 在多级且多核共享的架构下，必须时刻维护不同缓存层级之间的**数据一致性 (Consistency / Coherence)**。
### 13.4 多级 Cache 的 AMAT（⭐ 必考）

$$\boxed{\text{AMAT} = \text{L1 Hit Time} + \text{L1 Miss Rate} \times (\text{L2 Hit Time} + \text{L2 Miss Rate} \times \text{L2 Miss Penalty})}$$

**注意**：这里的 L2 Miss Rate 是**相对 miss rate**（relative miss rate），即 L1 miss 中又 miss L2 的比例。

**例题**：CPU 4GHz（1 cycle = 0.25ns），Base CPI = 1

| 配置                                         | CPI 计算                                   |
| ------------------------------------------ | ---------------------------------------- |
| 只有 L1（miss rate 2%, penalty 400 cycles）    | CPI = 1 + 0.02×400 = **9**               |
| 加 L2（L2 hit rate 80%, L2 access 20 cycles） | CPI = 1 + 0.02×20 + 0.02×0.2×400 = **3** |

加了 L2 快了 9/3 = **3 倍**！

**典型命中率**：L1 80~95%, L2 50~80%, L3 20~40%

---

## 十四、提高 Cache 性能的三条路

$$\text{AMAT} = \underbrace{\text{Hit Time}}_{\text{降低它}} + \underbrace{\text{Miss Rate}}_{\text{降低它}} \times \underbrace{\text{Miss Penalty}}_{\text{降低它}}$$

### 降低 Hit Time

- Cache 做小做简单（小=快）
- 用直接映射或低相联度,直接不用搜索了好吧 
- **Prediction (路预测):** 如果 Cache 每个坑位可以放好几个备选数据（组相联），硬件会“盲猜”数据最可能在哪个备选位置，优先比较那个位置的 Tag，猜中了就能省下一两个周期的比较时间。
- 虚拟地址索引（避免地址翻译延迟）
- 靠近 CPU，减少电信号传递延迟

### 降低 Miss Rate

- 增大 Cache 容量
- 提高相联度（但边际递减！8-way 和 4-way 差别很小）
- 使用 Victim Cache
- 优化代码（减少冲突缺失）

### 降低 Miss Penalty

- 多级 Cache（L1 → L2 → L3）
- Critical Word First + Early Restart
- Non-Blocking Cache
- Write Buffer
- Cache Prefetching（预取）

---

## 十五、内存组织方式（提高带宽）

| 方式   | 英文                 | 思路                     |
| ---- | ------------------ | ---------------------- |
| 宽总线  | Wide Memory        | 一次传更多位（但有引脚限制、信号完整性问题） |
| 交错存储 | Interleaved Memory | 多个 bank 并行工作，轮流提供数据    |

---

## 十六、Cache 一致性（Coherence）（⭐ 考试常考概念）

### 16.1 问题

多核系统中，每个核有自己的 Cache。如果核 A 修改了变量 u，核 B 的 Cache 里还是旧值——数据不一致！即使用 Write-Through 也解决不了（因为核 B 读的是自己的 Cache，不会去看内存）。

### 16.2 Snoopy 协议（监听协议）

所有 Cache 控制器都连在总线上，**监听（Snoop）** 其他核的操作：

- 如果核 A 要写某行 → 广播 invalidate → 其他核把自己的那行作废
- 如果核 B 要读某行，而核 A 有已经修改过的脏副本 → 核 A 提供数据

**MESI 协议**（四种状态）：

- **M (Modified)**：被修改过，只有这个 Cache 有，和内存不一致
- **E (Exclusive)**：数据是干净的，只有这个 Cache 有，和内存一致。并且别的核没有
- **S (Shared)**：多个 Cache 都有，和内存一致，就是多核共享
- **I (Invalid)**：无效 (空的或者数据过时了)

### 16.3 Directory-Based 协议（目录协议）

Snoopy 依赖总线广播，**不可扩展**到大规模系统。目录协议用一个集中/分布式的**目录**记录每个 Cache 行的状态和位置，避免广播。

- 优点：可扩展到上百/上千核
- 缺点：实现复杂，可能延迟更高

---

## 十七、Inclusive vs Exclusive Cache

| 类型  | 英文        | 规则                  | 优点               | 缺点                     |
| --- | --------- | ------------------- | ---------------- | ---------------------- |
| 包含式 | Inclusive | L1 的内容一定也在 L2 中     | 一致性检查方便（查 L2 即可） | L2 有 25% 空间被 L1 内容重复占用 |
| 排斥式 | Exclusive | L1 有的行 L2 一定没有,二者互斥 | 无空间浪费            | 一致性管理更复杂               |

**Inclusive 的操作**：

- L2 驱逐某行 → 必须同时 invalidate L1 中对应行（Back-Invalidation）

**Exclusive 的操作**：

- L1 驱逐某行 → 该行移到 L2
- L1 和 L2 都 miss → 数据从内存直接放入 L1（不放 L2）
- L1 miss 但 L2 hit → L2 行迁移到 L1,然后 L2 删除掉这一行

现代处理器**大多使用 Inclusive**。

一个问题：
Cache misses are likely to be correlated – imagine an AI-based cache prefetcher! Is it possible to use ML to detect such correlations?

---

## 十八、软件与 Cache 的交互

**重要结论**：算法操作数更少不代表运行更快！如果算法的 Cache 局部性差（如大规模 Radix Sort），大量 Cache Miss 会拖垮性能，使其反而比 Quicksort 慢。

**Locality matters!!**
quickSort 的空间局部性极佳

---

## 十九、考试速查公式卡

|公式|用途|
|---|---|
|`组数 = Cache大小 / (块大小 × 相联度)`|地址划分|
|`Offset位 = log₂(块大小)`|地址划分|
|`Index位 = log₂(组数)`|地址划分|
|`Tag位 = 32 − Index位 − Offset位`|地址划分|
|`总位数 = 行数 × (数据bits + Tag bits + 1valid [+ 1dirty] [+ ECC])`|Cache 总存储开销|
|`AMAT = Hit Time + Miss Rate × Miss Penalty`|⭐ 平均访问时间|
|`CPI = Base CPI + I-stall + D-stall`|实际 CPI|
|`I-stall = I-miss-rate × penalty`|指令缓存停顿|
|`D-stall = %ld/st × D-miss-rate × penalty`|数据缓存停顿|
|`多级 AMAT = L1_HT + L1_MR × (L2_HT + L2_MR × L2_MP)`|⭐ 多级 Cache|

---

## 二十、Pop Quiz 答案速记

**"以下哪些说法正确？"** 类型题：

- ✅ 2-way 比 4-way 有更多的组数（总行数相同）
- ✅ 直接映射的 Tag 开销最小（Tag bits 最少）
- ❌ 全相联是 1-way（错！全相联是 **1-set**）
- ❌ 直接映射是 1-set（错！直接映射是 **1-way**）
- ❌ 全相联比直接映射有更多 Cache 行（错！行数由 Cache 大小和块大小决定，跟映射方式无关）

**减少冲突缺失的方法**：增大 Cache、提高相联度、重新设计算法。增大块大小和 Tag 大小不一定有效。