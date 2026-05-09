# Chapter 4 - 处理器架构与逻辑设计

**Attention: 由于ppt一直在变化，导致笔记可能会有重复的部分，需要注意一下** 
## 一、总览：处理器和微架构 (Processors & Microarchitectures)

### 1.1 计算机架构的层次

计算机架构分两层理解：

- **指令集架构 ISA (Instruction Set Architecture)**：就是"处理器能听懂什么话"，比如 Intel x86、ARM、RISC-V 等。同一个 ISA 家族里的处理器都能运行同样的程序。
- **微架构 (Microarchitecture)**：就是"处理器内部怎么干活"。同一个 ISA 可以有不同的微架构实现，性能和成本不同。

### 1.2 常见微架构实现方式

| 实现方式 | 英文名                   | 特点（傻瓜版）                     |
| ---- | --------------------- | --------------------------- |
| 单周期  | Single Cycle          | 一条指令一个时钟周期干完，但时钟要很慢（等最慢的指令） |
| 多周期  | Multi-Cycle           | 一条指令分好几步，每步一个短时钟，省资源        |
| 流水线  | Pipelined             | 像工厂流水线，多条指令重叠执行，又快又省        |
| 超标量  | Superscalar           | 一个时钟能同时发射多条指令               |
| 乱序执行 | Out-Of-Order          | 不按程序顺序，哪条能跑先跑哪条             |
| 投机执行 | Speculative Execution | 猜测分支方向，提前执行                 |


## 二、处理器的两大部分：数据通路与控制 (Datapath & Control)

### 2.1 数据通路 (Datapath) —— "肌肉/干活的"

负责**实际的数据处理**，包括：

- **ALU (Arithmetic Logic Unit)**：算术逻辑单元，做加减法、逻辑运算
- **寄存器 (Registers)**：临时存数据的小仓库
- **存储器接口 (Memory Interface)**：和内存打交道
- **总线 (Buses)**：数据传输的"马路"

### 2.2 控制单元 (Control Unit) —— "大脑/指挥的"

负责**指令的流转和信号生成**，包括：

- **PC (Program Counter)**：程序计数器，记住"下一条指令在哪"
- **取指 (Instruction Fetch)**：从内存里拿指令
- **控制信号生成 (Control Signal Generation)**：告诉数据通路"现在该干嘛"

---

## 三、指令执行周期 (Instruction Execution Cycle)

### ⭐ 核心三步曲：Fetch → Decode → Execute

**获取，译码，执行**

每条指令都经历这三个阶段：

1. **取指 (Fetch)**：用 PC 的值作为地址，从指令存储器中取出指令；然后 PC = PC + 4（因为 RISC-V 每条指令 4 字节）,当指令集扩展为 **C,(compressed)** 的时候，有可能会出现，PC = PC + 2，指令从 32位降低到16位，只占 2 个 byte **C 中，32位指令和16位指令是混合使用的，PC 动的时候会预先判断**

2. **译码 (Decode)**：分析指令是什么操作，读取需要的寄存器
3. **执行 (Execute)**：用 ALU 计算。根据指令类型不同，计算内容不同：
    - **算术指令**：计算算术结果（如 add, sub）
    - **访存指令**：计算内存地址（如 lw, sw）
    - **分支指令**：计算跳转目标地址（如 beq）
4. **访存 (Memory Access)**：load/store 指令需要访问数据存储器
5. **更新 PC**：PC ← 目标地址（分支）或 PC + 4（顺序执行）

---

## 四、⭐⭐⭐ CPU 时间公式（考试必考！）

### 4.1 核心公式
$$\frac{\text{time}}{\text{program}} = \frac{\text{instructions}}{\text{program}} \times \frac{\text{cycles}}{\text{instruction}} \times \frac{\text{time}}{\text{cycle}}$$
$$\text{CPU Time} = \text{Instruction Count} \times \text{CPI} \times \text{Clock Cycle Time}$$

| 符号                           | 英文                     | 含义（傻瓜版）       |
| ---------------------------- | ---------------------- | ------------- |
| **Instruction Count**        | 指令数                    | 程序总共有多少条指令    |
| **CPI**                      | Cycles Per Instruction | 平均每条指令要几个时钟周期 |
| **Clock** **Cycle** **Time** | 时钟周期                   | 一个时钟滴答多长时间（秒） |

> 等价写法：$$\text{CPU Time} = \frac{\text{Instruction Count} \times \text{CPI}}{\text{Clock Frequency(时钟频率)}}$$

### 4.2 三种微架构对公式的影响

| 微架构                    | CPI        | 时钟周期         | 特点        |
| ---------------------- | ---------- | ------------ | --------- |
| 单周期 (**Single Cycle**) | 1          | 很长（要迁就最慢的指令） | 简单但慢      |
| 多周期 (**Multi-Cycle**)  | m（每条指令不同）  | 很短           | 灵活但复杂     |
| 流水线 (**Pipelined**)    | 约等于 1(牛逼！) | 很短           | 又快又好，主流做法 |
|                        |            |              |           |

### 4.3 数值例子（考试可能考的那种）

假设有 4 条指令 w, x, y, z，执行周期分别为 4, 2, 3, 4。假设 1 个长周期 = 4 个短周期：

- **单周期**：每条都用最长时间 → $4 \times 4 = 16$ 个短周期
- **多周期**：各用各的时间 → $4 + 2 + 3 + 4 = 13$ 个短周期
- **流水线**：重叠执行 → 约 $4 + 3 = 7$ 个短周期（流水线填满后每周期完成一条）

### 4.4 CISC vs RISC 对公式的影响

| 指标                         | CISC        | RISC         |
| -------------------------- | ----------- | ------------ |
| **指令数**(Instruction Count) | 低           | 高            |
| **指令复杂度**                  | 高(一条指令干很多事) | 低(每条指令只干一点事) |
| **CPI**                    | 高           | 低            |
| **时钟周期**(Clock Cycle Time) | 长           | 短            |
 现代 x86 处理器的秘密：外面接收 x86 指令（CISC），内部翻译成微操作 micro-operations（RISC）来执行！

### 4.5 FMA 例子（Fused Multiply-Add）

比较 `fma x1, x2, x3, x4`（1 条指令，CPI=5）和 `fmul + fadd`（2 条指令，CPI=4）：

$$\text{FMA: } 1 \times 5 \times \text{clk} = 5 \text{ clk}$$ $$\text{分开: } 2 \times 4 \times \text{clk} = 8 \text{ clk}$$

FMA 更快！
**但是最新版本的ppt似乎没有这个了，考试要看的时候注意一下哈**

## 五、逻辑设计基础 (Logic Design Basics)
### 5.0 逻辑设计演变

**过去**：纸笔画图 + 布尔表达式
**现在**：用 **HDL(硬件描述语言)** (Verilog/SystemVerilog/VHDL) 描述
**流程**：HDL → 逻辑综合器 → 最后得到：布尔方程 + FSM(有限状态机)
### 5.1 信息编码

- **低电压(low voltage) = 0，高电压 = 1**
- 一根线传一个 bit，多根线组成**总线 (Bus)** 传多个 bit

### 5.2 ⭐ 三大组件(components)

| 组件       | 英文                  | 特点                                                        | 例子          | 功能          |
| -------- | ------------------- | --------------------------------------------------------- | ----------- | ----------- |
| **组合逻辑** | Combinational Logic | **无状态**，输出只取决于当前输入                                        | ALU、加法器、MUX | 操控数据        |
| **时序逻辑** | Sequential Logic    | **有状态**，输出取决于输入 + 内部状态 (比如寄存器会保存数据，当你把输入撤走了，寄存器依然能保存这个数据) | 寄存器、计数器、FSM | 存储数据        |
| **时钟信号** | Clock Signal        | 控制何时更新存储元素                                                | 时钟          | 调控存储单元的更新机制 |
|          |                     |                                                           |             |             |

> **考试爱考**：组合逻辑 vs 时序逻辑的区别？ 答：组合逻辑无状态 (stateless)，输出纯粹由输入决定，如 ALU；时序逻辑有状态 (stateful)，输出由输入和内部状态共同决定，如寄存器。

### 5.3 基本逻辑门 (Logic Gates)

|逻辑门|符号|布尔表达式|功能|
|---|---|---|---|
|AND 与门|AND|$Y = A \wedge B$|都是 1 才输出 1|
|OR 或门|OR|$Y = A \vee B$|有一个 1 就输出 1|
|NOT 非门|NOT|$Y = \neg A$|取反|
|XOR 异或门|XOR|$Y = A \oplus B$|不同为 1，相同为 0|
|XNOR 同或门|XNOR|$Y = \overline{A \oplus B}$|相同为 1，不同为 0|
### 5.4 组合电路核心特性

* **实时响应 (Always Active)**：输入一旦改变，输出在经过短暂延迟后会**自动更新**，无需触发信号。
* **传播延迟 (Propagation Delay)**：电信号通过逻辑门需要时间。
* **关键路径 (Critical Path)**：指从输入到输出经过的**最长逻辑门路径**。
* **性能决定**：整个电路的运行速度（最高时钟频率）由**关键路径的延迟**决定。
---

## 六、组合逻辑元件 (Combinational Elements)

### 6.1 多路选择器 MUX (Multiplexer)

$$Y = S \ ? \ I_1 : I_0$$

等价布尔表达式：$Y = (S \wedge I_1) \vee (\neg S \wedge I_0)$

用人话说：S 是开关，S=0 选 I0，S=1 选 I1。

### 6.2 加法器 (Adder)

$$Y = A + B$$

### 6.3 ALU (Arithmetic Logic Unit)

$$Y = F(A, B)$$

F 是控制信号，决定做什么运算（加、减、与、或等）。

### 6.4 比较器 (Comparator)

$$L = (A < B), \quad E = (A == B), \quad G = (A > B)$$

### 6.5 ⭐ 传播延迟 (Propagation Delay)

> 组合电路的传播延迟 = **关键路径 (Critical Path)** 的延迟，即从任何输入到任何输出的最长逻辑门路径。

### 6.6 超前进位加法器 (Carry Lookahead Adder, CLA)

普通加法器的问题：进位需要一位一位传递，很慢（进位传播链）。

CLA 的思路：**提前计算所有位的进位**，消除进位传播链，大大加速。

---

## 七、时序逻辑元件 (Sequential Elements)

### 7.1 寄存器 (Register)

- 用**时钟信号 (Clock Signal)** 决定何时更新存储值
- **边沿触发 (Edge-triggered)**：只在时钟从 0 变 1（上升沿）时更新,别的时候都保持不动

### 7.2 带写控制的寄存器 (Register with Write Control)

- 只有当 **Write Enable = 1** 且时钟上升沿到来时，才更新值(想必前面那个多了个 write = 1 的条件)
- **Write Enable = 0** 时，值不变（即使时钟上升沿来了也不更新）
- 哪些指令不需要写寄存器？**分支指令 (Branch) 和存储指令 (Store)**,因为一个是判断跳转，一个只是把寄存器的数据存到内存里面去 ！

### 7.3 ⭐ 时钟方法论 (Clocking Methodology)

* **组合逻辑在时钟边沿之间变换数据** 时钟边沿（如上升沿）触发时，寄存器“开闸”释放数据；在两个边沿之间的电平平稳期，数据在无状态的组合逻辑电路中流动，并完成物理层面的加工计算。 
* **输入来自状态元素，输出到状态元素** 构成了 CPU 内部经典的数据通路模型：“起点寄存器（释放旧值） $\rightarrow$ 组合逻辑（加工计算） $\rightarrow$ 终点寄存器（拦截并等待写入新值）”。 
* **最长延迟决定了时钟周期（核心瓶颈 / 关键路径）** 时钟周期的长度（打节拍的间隔时间）必须**严格大于**组合逻辑中最耗时的那条物理路径延迟。若时钟跳动过快，未计算完的“半成品”数据就会被提前抓取入库，导致程序产生乱码崩溃。 
* **同一个周期内可以：读寄存器 $\rightarrow$ 组合逻辑计算 $\rightarrow$ 写回同一个寄存器** 得益于寄存器的**边沿触发**机制（例如执行自增逻辑 `i = i + 1`）。在当前时钟边沿读出旧值，流经组合逻辑算出新值后，新值会被挡在寄存器输入端外等待；直到**下一个**时钟边沿劈下来时，新值才会被真正吞入，完美避免了电信号的无限死循环。
![[Pasted image 20260311164459.png]]
---

## 八、存储元素 (Storage Elements)

### 8.1 寄存器堆 (Register File)

RISC-V 有 **32 个 32 位寄存器**。

寄存器堆的接口：

- **2 个读端口 (Read Ports)**：同时读两个寄存器（busA 和 busB）
- **1 个写端口 (Write Port)**：写一个寄存器（busW）
- RA, RB：选择要读的寄存器编号（5 位，因为 $2^5 = 32$）
- RW：选择要写的寄存器编号
- Write Enable：只在 = 1 时才写入
- **读操作是组合逻辑**（地址有效 → 输出有效，不需要时钟）
- **写操作是时序逻辑**（需要时钟上升沿）

> 超标量处理器 (2-wide superscalar) 需要多少读端口？至少 4 个（每条指令最多读 2 个寄存器）。 FMA 指令需要 3 个读端口（三个源操作数）。

### 8.2 数据存储器 (Data Memory)

- 一个输入总线 (Data In)，一个输出总线 (Data Out)
- Address 选择要访问的字
- Mem Write = 1：写入；Mem Read = 1：读取
- 读操作也是组合逻辑行为

---

## 九、⭐ RISC-V 指令格式 (Instruction Formats)
### How to Design a Processor

1. **需求分析**：分析指令集（ISA），确定数据在寄存器之间如何流动。
2. **选定组件**：选择合适的硬件元件（ALU、寄存器等），并制定时钟策略。
3. **搭建通路**：连接硬件组件，构建能支持指令运行的数据通路（Datapath）。
4. **确定控制点**：分析每条指令执行时，哪些开关（Mux）和写使能信号需要被激活。
5. **实现控制**：设计并组装控制逻辑（Control Logic），自动化指挥数据流向。
### 9.1 我们关注的指令子集

|指令|类型|功能|
|---|---|---|
|`add rd, rs1, rs2`|R-type|加法|
|`sub rd, rs1, rs2`|R-type|减法|
|`and rd, rs1, rs2`|R-type|按位与|
|`or rd, rs1, rs2`|R-type|按位或|
|`addi rd, rs1, imm12`|I-type|立即数加法|
|`lw rd, imm12(rs1)`|I-type|从内存加载字|
|`sw rs2, imm12(rs1)`|S-type|存字到内存|
|`beq rs1, rs2, imm12`|B-type|相等则跳转|
|`jal rd, target`|J-type|跳转并链接|

### 9.2 四种指令格式（32 位定长）

**R-type**（寄存器-寄存器运算）：

```
| funct7 (7) | rs2 (5) | rs1 (5) | funct3 (3) | rd (5) | opcode (7) |
```

**I-type**（立即数运算、Load）：

```
| imm[11:0] (12) | rs1 (5) | funct3 (3) | rd (5) | opcode (7) |
```

**S-type / B-type**（Store / Branch）：

```
| imm[11:5] (7) | rs2 (5) | rs1 (5) | funct3 (3) | imm[4:0] (5) | opcode (7) |
```

**J-type**（Jump）：

```
| imm[20|10:1|11|19:12] (20) | rd (5) | opcode (7) |
```

### 9.3 ⭐ RISC-V 指令格式的规律性 (Regularity)

这些规律让硬件设计更简单（**考试爱考**）：

- **opcode** 总在 bits \[6:0]
- **rd**（目标寄存器）总在 bits \[11:7]
- **rs1**（第一源寄存器）总在 bits \[19:15]
- **rs2**（第二源寄存器）总在 bits \[24:20]
- 至于为什么把这些写死，而牺牲了立即数 imm 切成两半，因为在硬件电路上运行的快啊
- 指令定长：16 或 32 位
- 立即数放在指令的高位

### 9.4 (Example) B型指令机器码拆解速查模板
**例子：**
**0xFE6298E3**

**1. 转二进制并按B型格式切分**
`1111 1110 0110 0010 1001 1000 1110 0011`

| 31      | 30:25     | 24:20 | 19:15 | 14:12  | 11:8     | 7       | 6:0     |
|---------|-----------|-------|-------|--------|----------|---------|---------|
| imm[12] | imm[10:5] | rs2   | rs1   | funct3 | imm[4:1] | imm[11] | opcode  |
| 1       | 111111    | 00110 | 00101 | 001    | 1000     | 1       | 1100011 |

**2. 查表提取常规字段**
* `opcode` + `funct3` (001) = `bne`
* `rs1` = 00101 = `x5`
* `rs2` = 00110 = `x6`

**3. 缝合立即数 (注意乱序与补零)**
* 拼接顺序：`imm[12]` `imm[11]` `imm[10:5]` `imm[4:1]` `0` (最低位硬件默认补0)
* 拼接结果：`1` `1` `111111` `1000` `0` = `1111111110000`

**4. 算真值 (补码转换)**
* 13位补码 `1111111110000`，首位为1，记为负数。
* 取反加1得绝对值：`0000000010000` = 十进制 16。
* 真实立即数 = `-16`。
**结果：** `bne x5, x6, -16`

## 十、⭐⭐ 寄存器传输描述 (Logical Register Transfer)
### 寄存器与寄存器堆 (Register & Register File)

#### 1. 单个寄存器 (Register)
寄存器是 CPU 内部存储数据的基本单元，本质上是由 **D 触发器** 组成的同步时序电路。

* **核心特性**：
    * **N 位输入/输出**：对应处理器的位数（如 RV32 为 32 位）。
    * **写使能信号 (Write Enable, WE)**：控制数据写入的“开关”。
        * `WE = 0`：输出保持不变，保护原有数据。
        * `WE = 1`：在时钟边沿（Clk），`Data Out` 更新为 `Data In`。
* **时钟 (Clk)**：仅在**写入**操作时起作用，决定数据存入的时机。
![[Pasted image 20260330153936.png]]

**单个寄存器**：
当 write enable 为0的时候，无论Data in 是什么，都不会写入寄存器，Data out不变
只有当 write enable 变成1的时候，在下一个时钟周期上升沿(CLK)，寄存器会把 Data in 的值写入，这时候 Data out 就会变成这个新的值

#### 2. 寄存器堆 (Register File)
RISC-V 包含 **32 个** 32 位通用寄存器，它们被整合为一个寄存器堆。

##### 端口设计
* **读端口 (Read Ports)**：
    * **2 个 32 位输出总线**：`busA` 和 `busB`，支持同时读取两个操作数（如 $rs1$ 和 $rs2$）。
    * **选择信号**：`RA` 选择映射到 `busA` 的寄存器；`RB` 选择映射到 `busB` 的寄存器。
    * **特性**：读取操作是**组合电路行为**，不需要时钟触发，给地址即出数据。
* **写端口 (Write Port)**：
    * **1 个 32 位输入总线**：`busW`，用于回写计算结果（如 $rd$）。
    * **选择信号**：`RW` 指定要写入的寄存器编号。
    * **特性**：必须满足 `Write Enable = 1` 且 **时钟边沿触发** 才能完成写入。
    * 因为一般都是算完最后只有一个值，所以 write port 只有一个

#### 扩展思考 (Hardware Scaling)
| 场景                            | 需求分析                 | 读端口数量 |
| :---------------------------- | :------------------- | :---: |
| **标准 RISC-V 指令**              | 同时读取 $rs1, rs2$      |   2   |
| **FMA 指令 ($a \times b + c$)** | 同时读取三个源操作数           |   3   |
| **2-路超标量处理器**                 | 同时执行两条双源指令,比如 CPU 并发 |   4   |

---

这是每条指令在硬件层面"到底做了什么"的精确描述（**考试极可能让你默写**）：

| 指令         | 寄存器传输                                                                  |
| ---------- | ---------------------------------------------------------------------- |
| ADD        | `R[rd] ← R[rs1] + R[rs2]; PC ← PC + 4`                                 |
| SUB        | `R[rd] ← R[rs1] - R[rs2]; PC ← PC + 4`                                 |
| LOAD (lw)  | `R[rd] ← MEM[R[rs1] + SignExt(Imm12)]; PC ← PC + 4`                    |
| STORE (sw) | `MEM[R[rs1] + SignExt(Imm12)] ← R[rs2]; PC ← PC + 4`                   |
| ADDI       | `R[rd] ← R[rs1] + SignExt(Imm12); PC ← PC + 4`                         |
| BEQ        | `if (R[rs1] == R[rs2]) then PC ← PC + SignExt(Imm12) else PC ← PC + 4` |

> **SignExt** = 符号扩展 (Sign Extension)：把短的立即数扩展成 32 位，保持正负号不变。

### 补充L: Storage Element: Memory (内存存储元件)

- **数据接口**: 
  - `Data In` (32-bit): 单一输入总线
  - `Data Out` (32-bit): 单一输出总线
  - 内存一次只能读/写 一个数据，这和寄存器是不一样的
- **地址接口**: 
  - `Address`: 定位要读/写的特定内存字 (Word)
- **控制信号**:
  - `Mem Read`: 为 1 时，读取 `Address` 指定的数据到 `Data Out`
  - `Mem Write`: 为 1 时，将 `Data In` 的数据写入到 `Address` 指定的位置

## 十一、数据通路设计步骤 (How to Design a Processor)

### ⭐ 五步设计法（考试可能考流程）

1. **分析指令集需求** (Analyze Instruction Set)：搞清楚每条指令需要什么硬件资源
2. **选择数据通路组件和时钟方法** (Select Components & Clocking)：选好组合逻辑和时序逻辑元件
3. **组装数据通路** (Assemble Datapath)：把组件连起来
4. **分析控制点** (Analyze Control Points)：确定每条指令需要什么控制信号
5. **设计控制逻辑** (Assemble Control Logic)：用控制信号驱动数据通路

### 数据通路需要的资源

根据寄存器传输分析，数据通路需要：

- **存储器**：存指令和数据（I-cache 和 D-cache 分开）
- **寄存器堆**：32 × 32-bit，支持读 RS1、读 RS2、写 RD
- **PC 寄存器**
- **扩展器 (Extender)**：做零扩展或符号扩展
- **ALU**：加减法和逻辑运算
- **加法器**：给 PC 加 4 或加偏移量

---

## 十二、数据通路组装 (Datapath Assembly)

### 12.1 取指单元 (Instruction Fetch Unit)

- 从 `mem[PC]` 取指令
- 顺序执行：`PC ← PC + 4`
- 分支/跳转：`PC ← 其他地址`

### 12.2 分支操作 (Branch Operations)

以 `beq rs1, rs2, imm12` 为例：

1. 从内存取指令
2. 比较 `R[rs1]` 和 `R[rs2]` 是否相等
3. 如果相等：$PC \leftarrow PC + (\text{SignExt}(\text{imm12}) \times 2)$
4. 如果不等：$PC \leftarrow PC + 4$

> 为什么乘以 2？因为指令地址总是偶数 (指令占据的字节只可能是2/4字节)（最低位必为0），为了节省空间并扩大跳转范围，存入的立即数是砍掉最低位0的“压缩版”，所以计算真实字节地址时必须乘以 2（左移 1 位）进行还原。

> BEQ 为什么需要两个 adders？ 为了并行计算(parallel calculating)
> 一个计算减法的值(相减以判断是否相等)，
> 一个计算 PC +4 / PC + extend(Imm) * 2

### ⭐ 分支用减法的陷阱 (Branch with Subtraction Alert)

问题：当我们比较两个数的大小的时候，是否可以偷懒直接用 A-B 的符号来判断呢？
(比较相等的时候我们已经偷懒了 hhh)

- 直觉上，比较可以用减法实现（结果为正/零/负决定分支方向）
- **问题**：如果减法溢出 (overflow) 了怎么办？结果的符号就不对了！
- **RISC-V 的做法**：RISC-V 不支持硬件溢出陷阱，需要软件显式检查，这导致程序员可能要写一坨大便代码来处理这个东西？！
- **实际处理器**：
	1. 直接设计一个不会错的比较逻辑，100%正确，就是很贵还很复杂
	2. **(常用)** 在实际应用中，使用 V (overflow) 和 C (carry，进位) 标志位在临时状态寄存器中，CPU 比较的时候看看这些标志还有正负号再综合判断

### 12.3 R-type 操作 (Add/Sub)

`add rd, rs1, rs2` 的数据流：

1. rs1, rs2 字段 → 寄存器堆的 RA, RB 输入 → 读出两个值到 busA, busB
2. busA, busB → ALU → 计算结果
3. ALU 结果 → 寄存器堆的 busW → 写入 rd 指定的寄存器
4. 控制信号 RegWrite = 1，在时钟上升沿写回

### 12.4 Load/Store 操作

`lw rd, imm12(rs1)` 的数据流：

1. rs1 → 寄存器堆 → busA（基地址）
2. imm12 → 符号扩展 → ALU 的第二输入
3. ALU 做加法：基地址 + 偏移 = 内存地址
4. 内存地址 → 数据存储器 → 读出数据
5. 数据 → 寄存器堆 → 写入 rd

### 12.5 合并 Mem 和 R-type 指令的数据通路

需要**两个 MUX**：

- **ALU 输入 MUX**：选择第二操作数是寄存器值还是立即数
- **写回 MUX**：选择写回寄存器的是 ALU 结果还是内存读出的数据

---

## 十三、⭐⭐ 控制信号 (Control Signals)

### 主要控制信号

| 控制信号             | 英文                | 功能                            |
| ---------------- | ----------------- | ----------------------------- |
| PCsrc            | PC Source         | 选择 PC+4 还是 PC+偏移（分支目标）        |
| ALUSrc           | ALU Source        | 选择 ALU 第二输入是寄存器还是立即数          |
| ALUctr           | ALU Control       | 告诉 ALU 做什么运算（加/减/与/或）         |
| MemRd            | Memory Read       | 是否读数据存储器                      |
| MemWr            | Memory Write      | 是否写数据存储器                      |
| MemtoReg / WBsel | Write-Back Select | 选择写回寄存器的数据来源（ALU结果/内存数据/PC+4） |
| RegWr            | Register Write    | 是否写寄存器堆                       |

### 三个关键 MUX 的控制

1. **ALUSrc MUX**：选择寄存器数据 or 立即数 → R-type 选寄存器，I-type/Load/Store 选立即数
2. **MemtoReg / WBsel MUX**：选择 ALU 输出 or 内存数据 or PC+4 → R-type 选 ALU ，Load 选内存，JAL 选 PC+4
如果是ALU: 将计算的结果存进当前寄存器
如果是Load: 当前计算的结果是一个Address，去内存中把这个地址存储的数据加载到当前寄存器
如果是JAL: 存储PC + 4这个下一条指令的地址，方便(比如函数返回的时候)找地址
3. **PCsrc MUX**：选择 PC+4 or 分支目标地址 → 不分支选 PC+4，分支选目标地址

---

## 十四、⭐⭐ 单周期处理器 (Single Cycle Processor)

### 14.1 工作原理

> 每个时钟上升沿，处理器完成一条指令的所有步骤。

1. 当前状态元素的输出驱动组合逻辑的输入
2. 组合逻辑的输出在下一个时钟上升沿之前稳定
3. 下一个上升沿到来 → 所有状态元素更新 → 进入下一个周期

### 14.2 单周期的限制 (Limitations)

- **资源不能复用**：需要多个加法器（PC 更新用一个，ALU 用一个）
- **指令存储器和数据存储器必须分开**：因为一个周期内不能访问同一个存储器两次
- **寄存器堆需要 2 读端口 + 1 写端口**：同一周期内读两个寄存器又写一个
- **时钟周期由最慢的指令决定**：所有指令都要等最慢的那个（通常是 load）

### 14.3 单周期的低效 (Inefficiency)

- 乘法、除法等迭代运算在单周期中需要**复制**大量组合逻辑，成本太高
- 解决方案：多周期实现 → 用计数器让同一个硬件资源被反复使用

---

## 十五、⭐⭐ 关键路径 (Critical Path)

### 关键路径的定义

**关键路径** = 数据通路中**耗时最长的路径**，它决定了时钟周期的最小值。
(在第三个project中我们对其有了简单的了解)

### ⭐⭐ Load 指令的关键路径（最长的，考试必考！）

$$T_{cycle} = T_{clk-to-Q}^{PC} + T_{access}^{IMem} + T_{access}^{RegFile} + T_{ALU} + T_{access}^{DMem} + T_{setup}^{RegFile} + T_{skew}^{Clock}$$

用人话说，Load 指令的关键路径时间 = 以下所有时间之和：

1. **PC 的 Clk-to-Q 时间**：时钟到来后 PC 输出新值的延迟
2. **指令存储器访问时间**：根据地址取出指令
3. **寄存器堆访问时间**：读出基地址寄存器
4. **ALU 延迟**：计算内存地址（基地址 + 偏移）
5. **数据存储器访问时间**：根据地址读出数据
6. **寄存器堆写入的建立时间 (Setup Time)**：数据要在时钟上升沿前稳定
7. **时钟偏移 (Clock Skew)**：不同组件收到时钟信号的时间差

> 这是**单周期处理器时钟周期的下限**，时钟周期必须 ≥ 这个值！

---

## 十六、架构状态与上下文切换 (Architecture States & Context Switch)

### 需要保存的状态

上下文切换（线程切换 Thread Switch 或进程切换 Process Switch）时需要保存：

- **线程切换**：处理器寄存器
- **进程切换**：PCB (Process Control Block，进程控制块)，包含寄存器

### 处理器状态包括

- PC（程序计数器）
- 整数寄存器 (Integer Registers)
- 浮点寄存器 (FP Registers)
- 向量寄存器 (Vector Registers)
- 状态寄存器/标志位 (Status Registers / Flags)
- 页表指针 (Pointer to Page Table)

> 缓存、控制信号等不需要保存——它们要么保持不变，要么由 OS 维护，要么可以重新生成。 恢复程序只需恢复其状态即可。

---

## 十七、硬件描述语言 (Hardware Description Languages, HDL)

### 从前 vs 现在

- **从前**：画电路图、写布尔表达式
- **现在**：用 HDL 描述硬件结构

### 常见 HDL

- **Verilog**
- **SystemVerilog**（Verilog 的超集）
- **VHDL**
- **Chisel**（SiFive 用于 RISC-V）
- **HCL** (Hardware Control Language)：教材中定义的描述控制逻辑的语言

### 类比

|软件世界|硬件世界|
|---|---|
|高级语言 (C/C++)|硬件描述语言 (Verilog)|
|编译器 (Compiler)|逻辑综合器 (Logic Synthesizer)|
|汇编/机器码|布尔方程 + FSM|

---

## 十八、计算机仿真 (Computer Simulation)

- 计算机仿真 = 模拟**架构状态的变化**
- 每条指令执行后，寄存器、PC、内存等状态会更新
- 调试时的单步执行 (Single Stepping) 就是在观察状态变化
- 也需要考虑外部事件：I/O 和中断 (Interrupts)

---

## 考试重点速查表

### 必背公式

$$\boxed{\text{CPU Time} = \text{Instruction Count} \times \text{CPI} \times \text{Clock Cycle Time}}$$

$$\boxed{T_{cycle} \geq T_{clk-to-Q}^{PC} + T_{IMem} + T_{RegRead} + T_{ALU} + T_{DMem} + T_{setup} + T_{skew}}$$

### 必背概念

1. **Datapath vs Control**：数据通路是干活的肌肉，控制单元是指挥的大脑
2. **Combinational vs Sequential Logic**：组合逻辑无状态（ALU），时序逻辑有状态（寄存器）
3. **Fetch-Decode-Execute 周期**
4. **Single Cycle vs Multi-Cycle vs Pipelined** 的 CPI 和时钟周期对比
5. **RISC-V 指令格式** 的字段位置规律性
6. **寄存器传输描述**（每条指令做了什么）
7. **关键路径**（Load 指令最长）
8. **控制信号** 各自的作用和 MUX 选择逻辑
9. **单周期限制**：资源不能复用、最慢指令决定时钟
10. **上下文切换需要保存的状态**

### 必背对比

|组合逻辑 (Combinational)|时序逻辑 (Sequential)|
|---|---|
|无状态 (Stateless)|有状态 (Stateful)|
|输出只取决于输入|输出取决于输入 + 状态|
|例：ALU、MUX、Adder|例：Register、Counter、FSM|
|不需要时钟|需要时钟驱动更新|