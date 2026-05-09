# CSC3060 计算机体系结构 - Chapter 4: Control（第二部分）


## 一、单周期数据通路回顾 (Single Cycle Datapath Review)

### 什么是单周期处理器？

想象你去食堂打饭：**一个周期 = 从排队到吃完一整顿饭**。不管你点的是简单的白饭（ADD 指令）还是复杂的套餐（LOAD 指令），食堂都要求你花同样长的时间才能离开。这就是单周期处理器——**每条指令都用一个时钟周期完成，CPI（Cycles Per Instruction）= 1**。

### 数据通路的五大阶段

一条指令在单周期数据通路里要走过这些步骤：

1. **取指 (Instruction Fetch, IF)**：从指令存储器 (Instruction Memory) 读出指令
2. **译码 (Instruction Decode, ID)**：读寄存器文件 (Register File)，取出 rs1、rs2 的值
3. **执行 (Execute, EX)**：ALU 做运算
4. **访存 (Memory Access, MEM)**：Load/Store 访问数据存储器 (Data Memory)
5. **写回 (Write Back, WB)**：把结果写回寄存器文件
6. ppt 上的图一要大致理解了

## 二、控制信号 (Control Signals) ⭐⭐⭐ 考试重点

### 控制信号是什么？

数据通路就像一条有很多岔路口的高速公路，**控制信号就是每个岔路口的红绿灯**，告诉数据该往哪边走。

### 控制信号从哪来？

指令的不同字段决定了控制信号：

| 指令字段       | 位域 (Bit Field) | 作用                                    |
| ---------- | -------------- | ------------------------------------- |
| **Opcode** | inst[6:0]      | 决定指令大类（R-type、I-type、S-type、B-type 等） |
| **funct3** | inst[14:12]    | 在同一大类中区分具体操作（如 ADD vs SLT）            |
| **funct7** | inst[31:25]    | 进一步区分（如 ADD vs SUB）                   |
| **Rs1**    | inst[19:15]    | 源寄存器1                                 |
| **Rs2**    | inst[24:20]    | 源寄存器2                                 |
| **Rd**     | inst[11:7]     | 目的寄存器                                 |
**注意，0位是最低位(最右边),31位是最高位(最左边)**

### 六大核心控制信号 ⭐⭐⭐ 必须默写

sel: 选择器

> 这里是考试最爱考的内容，对于每种指令类型，每个控制信号应该设成什么值。

| 控制信号               | 含义                                 | 表达式                           |
| ------------------ | ---------------------------------- | ----------------------------- |
| **PCSrc**          | PC 的下一个值从哪来？0=PC+4，1=分支/跳转目标       | `(Branch \| JAL) ? 1 : 0`     |
| **RegWr (RegWen)** | 是否写寄存器？                            | `(Branch \| Store) ? 0 : 1`   |
| **MemRd**          | 是否读数据存储器？                          | `Load ? 1 : 0`                |
| **MemWr**          | 是否写数据存储器？                          | `Store ? 1 : 0`               |
| **ALUSrc (BSel)**  | ALU 第二个输入是什么？0=寄存器(rs2)，1=立即数(imm) | `R-type ? 0 : 1`              |
| **WBSel**          | 写回寄存器的数据从哪来？                       | `Load:0 \| R-type:1 \| JAL:2` |

### 更完整的控制信号表（考试常考填表题）⭐⭐⭐

|控制信号|R-type|Load (I)|Store (S)|Branch (B)|JAL (J)|JALR|
|---|---|---|---|---|---|---|
|**PCSrc**|0|0|0|1(若条件满足)|1|1|
|**RegWr**|1|1|0|0|1|1|
|**MemRd**|0|1|0|0|0|0|
|**MemWr**|0|0|1|0|0|0|
|**ALUSrc/BSel**|0 (rs2)|1 (imm)|1 (imm)|0 (rs2)|×|×|
|**ASel**|0 (rs1)|0 (rs1)|0 (rs1)|1 (PC)|1 (PC)|0 (rs1)|
|**BSel**|0 (rs2)|1 (imm)|1 (imm)|1 (imm)|1 (imm)|1 (imm)|
|**WBSel**|1 (ALU)|0 (Mem)|×|×|2 (PC+4)|2 (PC+4)|

> **记忆口诀**：
> 
> - **谁不写寄存器？** Store 和 Branch（它们只存数据/跳转，不产生结果）
> - **谁读内存？** 只有 Load
> - **谁写内存？** 只有 Store
> - **谁用立即数做ALU输入？** 除了 R-type 和 Branch，都用立即数
> - **WBSel**：Load 从内存拿(0)，R-type 从 ALU 拿(1)，JAL/JALR 存 PC+4(2)

### ASel 和 BSel 的含义

在更完整的数据通路中，ALU 的两个输入前面各有一个多路选择器 (MUX)：

- **ASel (A Select)**：选择 ALU 的第一个输入
    - `0` → rs1 的值（大多数指令）
    - `1` → PC 的值（Branch 计算跳转地址、JAL 计算 PC+imm）
- **BSel (B Select)**：选择 ALU 的第二个输入
    - `0` → rs2 的值（R-type 指令）
    - `1` → 立即数 (Immediate)（I-type、S-type、JAL 等）

---

## 三、ALU 控制 (ALU Control) ⭐⭐⭐ 考试重点

### ALU 怎么知道该做什么运算？

ALU 控制信号由**两层**决定：

1. **第一层**：主控制器根据 opcode 产生 **ALUOp**（2-bit）
2. **第二层**：ALU 控制器根据 ALUOp + funct3 + funct7 产生具体的 **ALUControl** 信号

这就像去餐厅：主控制器说"点中餐"（ALUOp），ALU控制器再细化"要宫保鸡丁"（具体运算）。

### ALUOp 编码 ⭐

|ALUOp|含义|对应指令类型|
|---|---|---|
|`00`|做加法|Load / Store（计算地址 = base + offset）|
|`01`|做减法|Branch（比较两个寄存器）|
|`10`|看 funct3/funct7 决定|R-type / I-type 算术指令|

### ALUOp = 10 时的详细解码表 ⭐⭐⭐ 必须默写

当 ALUOp = `10` 时，需要进一步看 funct3 和 funct7[5]（即 bit30）来确定具体运算：

|funct3|funct7[5]|ALU 运算|对应指令|
|---|---|---|---|
|`000`|`0`|ADD|add, addi|
|`000`|`1`|SUB|sub|
|`001`|×|SLL (左移)|sll, slli|
|`010`|×|SLT (小于置位)|slt, slti|
|`100`|×|XOR|xor, xori|
|`101`|`0`|SRL (逻辑右移)|srl, srli|
|`101`|`1`|SRA (算术右移)|sra, srai|
|`110`|×|OR|or, ori|
|`111`|×|AND|and, andi|

> **记忆技巧**：funct7[5] 只在 funct3=`000`（ADD/SUB）和 funct3=`101`（SRL/SRA）时才有区别作用，其它情况不用管。

### Verilog 实现方式（了解即可）

控制信号可以用 HDL（Hardware Description Language，硬件描述语言）来描述，比如 Verilog 和 VHDL。写好的 HDL 代码可以通过设计编译器（如 Synopsys、Cadence 工具）自动综合成电路网表 (Netlist)。

---

## 四、指令执行实例分析 ⭐⭐ 考试画图/填信号题

### 例1：SW X2, offset(X1) — Store Word

**指令含义**：把 X2 的值存到内存地址 `X1 + offset` 处。

**控制信号设置**：

|信号|值|原因|
|---|---|---|
|RegWr|0|Store 不写寄存器|
|MemWr|1|要写内存|
|MemRd|0|不读内存|
|ALUSrc/BSel|1|ALU 第二个输入是 offset（立即数）|
|ASel|0|ALU 第一个输入是 rs1|
|ALUOp|00|ALU 做加法（计算地址）|
|PCSrc|0|PC = PC + 4，顺序执行|

**数据流**：rs1(基地址) + imm(偏移量) → ALU 算出地址 → rs2 的数据写入该地址

### 例2：BEQ X1, X2, immediate — Branch if Equal

**指令含义**：如果 X1 == X2，就跳转到 PC + immediate。

**控制信号设置**：

|信号|值|原因|
|---|---|---|
|RegWr|0|Branch 不写寄存器|
|MemWr|0|不写内存|
|MemRd|0|不读内存|
|ALUSrc/BSel|0|比较 rs1 和 rs2，所以用寄存器值|
|ALUOp|01|做减法来比较|
|PCSrc|1（若相等）|跳转到 PC + imm|

**数据流**：ALU 计算 rs1 - rs2 → 如果结果为 0（即 Equal=1），则 PC 更新为 PC + immediate

---

## 五、单周期设计的缺陷 (Drawback of Single-Cycle Design) ⭐⭐⭐ 必考概念

### 核心问题：时钟周期太长

单周期处理器的时钟周期必须适应**最慢的指令**，而最慢的指令是 **Load 指令**。

### Load 指令的关键路径 (Critical Path) ⭐⭐⭐ 必须默写

$$T_{cycle} = T_{PC(clk \to Q)} + T_{IMem} + T_{RF(read)} + T_{ALU} + T_{DMem} + T_{RF(setup)} + T_{skew}$$

各项含义：

|组件|英文|含义|
|---|---|---|
|$T_{PC(clk \to Q)}$|PC Clock-to-Q|PC 寄存器输出稳定需要的时间|
|$T_{IMem}$|Instruction Memory Access Time|从指令存储器读指令的时间|
|$T_{RF(read)}$|Register File Access Time|从寄存器文件读数据的时间|
|$T_{ALU}$|ALU Delay|ALU 计算地址的时间|
|$T_{DMem}$|Data Memory Access Time|从数据存储器读数据的时间|
|$T_{RF(setup)}$|Register File Setup Time|数据写回寄存器前的建立时间|
|$T_{skew}$|Clock Skew|时钟信号在芯片不同位置到达的时间差|

### 为什么这是个大问题？

比如各指令实际需要的时间：

|指令类型|需要经过的阶段|实际需要时间|
|---|---|---|
|**Load**|IF → ID → EX → MEM → WB|最长（全部阶段）|
|R-type|IF → ID → EX → WB|少了 MEM 阶段|
|Store|IF → ID → EX → MEM|少了 WB 阶段|
|Branch|IF → ID → EX|最短之一|

但在单周期设计中，**所有指令都必须用 Load 的时间**来执行。这意味着像 ADD 这种本来很快的指令也要"等"很久，大量时间被浪费了。

> **类比**：就像考试规定所有人都必须坐满 3 小时才能交卷，即使你 1 小时就做完了也不能走。

---

## 六、RISC-V 为什么简化了控制设计？ ⭐⭐

RISC-V 的指令集设计天然利于硬件实现：

1. **固定长度指令 (Fixed-size Instructions)**：所有指令都是 32 位，取指简单
2. **规整的格式 (Regular Format)**：rs1 和 rs2 总在相同位置（inst[19:15] 和 inst[24:20]），不用先解码 opcode 才能找到寄存器编号
3. **立即数大小统一**：要么 12-bit 要么 20-bit，立即数生成逻辑简单
4. **运算只在寄存器/立即数上进行**：不像 x86 那样可以直接对内存做运算，简化了数据通路

> 这些特点降低了设计成本和测试难度。

---

## 七、考试速查清单 ⭐⭐⭐

### 必须能默写的内容

1. **六大控制信号的表达式**（PCSrc、RegWr、MemRd、MemWr、ALUSrc、WBSel）
2. **各指令类型的控制信号值**（上面那个大表）
3. **ALUOp 编码**（00=加法、01=减法、10=看funct）
4. **ALUOp=10 时 funct3/funct7 到 ALU 运算的映射表**
5. **Load 指令的关键路径公式**（7 个延迟项）
6. **单周期设计的缺点**：时钟周期由最慢指令(Load)决定，其他指令都在"浪费时间"

### 常见考试题型

- **填控制信号表**：给你一条指令，要求你填出每个控制信号的值
- **画数据通路上的信号**：在数据通路图上标注控制信号和数据流向
- **计算时钟周期**：给出各部件延迟，计算单周期的时钟周期
- **解释单周期缺点**：说明为什么 CPI=1 不一定快，引出流水线的必要性
- **ALU 控制解码**：给 opcode/funct3/funct7，写出 ALU 应该做什么运算