# RISC-V 指令集架构讲座总结

(为什么有这么多补充，因为老登每节课更新ppt然后往里面加点东西，搞得神经衰弱)
(为什么逻辑比较混乱，因为他的ppt就不是按照什么逻辑顺序弄得)
## 1. 什么是指令集架构（ISA）？
- **指令集架构（Instruction Set Architecture, ISA）**：是计算机的“语言”，定义了硬件和软件之间的接口。它规定了处理器能执行的指令、寄存器、内存访问方式等。
- **设计目标**：
  - 最大化性能（Maximize performance）
  - 最小化成本（Minimize cost）
  - 缩短设计时间（Reduce design time）,易于测试和设计
  - 最小化内存占用（Minimize memory space）
  - 最小化功耗（Minimize power consumption）
  - ISA 应具有可扩展性（scalable）、灵活性（flexible）和可扩展性（extensible）。

## 2. CISC 与 RISC 对比
- **CISC（Complex Instruction Set Computer，复杂指令集计算机）**：
  - 指令复杂，长度可变。
  - 硬件复杂，软件相对简单，内存占用更优。
  - 例子：Intel x86, IBM 360。
  - 优点：软件遗产丰富（legacy software）。
- **RISC（Reduced Instruction Set Computer，精简指令集计算机）**：
  - 指令简单，长度固定。
  - 硬件简单，软件相对复杂。
  - 采用“加载/存储（Load/Store）”架构：只有 load/store 指令可以访问内存。
  - 例子：RISC-V, MIPS, ARM。
  - 优点：易于流水线实现（pipelining），编译器友好。

## 3. RISC 哲学（RISC Philosophy/Principle）
### Simplicity favors regularity（简洁有利规则）
- 固定指令长度（Fixed instruction lengths）
- 少量的指令格式(Small number of instruction formats)
- 操作码始终是前几位(Opcode always the first several bits)
### Smaller is fast
- 有限的指令集(limit instructiong set)
- 有限的寄存器数量(Limit registers)
- 加载/存储指令集（Load-store instruction sets）
- 有限的寻址模式（Limited number of addressing modes）
### Make the common cases fast
- 算术操作来自寄存器 Arithmetic operands from the registers (load-store machine)
- 允许指令包含立即操作数 Allow instructions to contain immediate operands
- 少量简单操作（Small number of simple operations）
- 易于流水线实现（Easier for pipelined implementation）
- ISA 的评价标准：编译器如何高效使用它，而不是汇编程序员。

## 4. RISC-V ISA 概述
- **RISC-V（RV）**：一种开放、免费、可扩展的 RISC ISA。
- **变体（Variants）**：
  - RV32, RV64, RV128：不同数据宽度（32位、64位、128位）。
- **扩展（Extensions）**：
  - **I**：基础整数指令（Base Integer instructions）。
  - **E**：嵌入式系统基础（Base for embedded systems，只有16个寄存器）。
  - **M**：乘法和除法（Multiply and Divide）。
  - **A**：原子内存指令（Atomic memory instructions）。
  - **C**：压缩扩展（Compressed extension，16位指令）。
  - **F 和 D**：单精度和双精度浮点（Single and Double precision floating point）。
  - **V**：向量扩展（Vector extension）。
  - RV后面的缩写字母必须遵循定义的顺序
- 扩展命名顺序：基础实现（I 或 E） → 标准扩展 → 非标准扩展。
## 补充: RISC-V的一些指令(和 x86 的语法有很多不同)
见 汇编语言部分知识.md

## 5. 寄存器（Registers）
- **通用寄存器（General Purpose Registers, GPR）**：32个，名为 x0 到 x31。
  - x0 是硬连线到零的寄存器（hard-wired to zero），常数值为 0。
- **程序计数器（Program Counter, PC）**：32位宽，存放下一条要执行的指令在在内存中的地址，每次执行完一条指令之后，改变PC (比如 PC += k(向后顺序执行),PC = x (跳转))。 PC 的值直接决定了程序现在在执行第几行
- **控制和状态寄存器（Control and Status Registers, CSR）**：
  - 用户模式（User-mode）：如 cycle（时钟周期数）、instret（指令计数）。
  - 机器模式（Machine-mode）：如 hartid（硬件线程ID）、mepc(报错时，PC在哪)、mcause（用于异常处理,mcause记录引发异常的原因）。
  - 自定义（Custom）：如 mtohost（输出到主机）。
- **架构状态（Architecture state）**：处理器在任意时刻的所有可见信息，上下文切换时必须保存。

## 6. 为什么 x0 固定为 0？
- 经常在指令中使用常数 0，例如清零寄存器、取反、比较等。
- 例子：`A = -B`，B 在 x8 中。
  - 没有 x0 时：`XOR X10, X10, X10`（清零 x10），然后 `SUB X9, X10, X8`。
  - 有 x0 时：直接 `SUB X9, X0, X8`。
- 复制寄存器：`ADD X1, X0, X2` 将 x2 复制到 x1。
- 相当于提供了一个可以直接调用常数 0 的寄存器，无需将别的寄存器清零再使用
- Some RISC processors may even use load to x0 as memory hints(利用加载x0 一定为0的性质，作为一种加载的提示)

## 7. 指令类型（按功能）
- **寄存器-寄存器算术逻辑运算（Register-to-Register ALU operations）**
- **控制指令（Control Instructions）**：改变程序流程，如分支（B）和跳转（J）。
- **内存指令（Memory Instructions）**：加载（Load）和存储（Store）数据。
- **CSR 指令（CSR Instructions）**：在 CSR 和 GPR 之间移动数据。
 - **特权指令（Privileged Instructions）**：只有操作系统内核模式能执行(好像是需要管理员权限?)。

## 8. 指令格式（Instruction Formats）
- **R-type（寄存器类型）**：用于寄存器-寄存器操作。
  - 格式：`funct7 | rs2 | rs1 | funct3 | rd | opcode`
  - 7 + 5 + 5 + 3 + 5 + 7 = 32 总共32位bit，代表机器语言的代码
  - **opcode 决定了他是哪种指令，在后面部分我们会涉及**
  - rd<-rs1(func3,func7)rs2 （func3,func7) 共同作用指向一个操作符，比如指向 add/sub/sll/srl 中的一个 (就是一个映射关系)
  - 例子：`add x1, x2, x3`
- **I-type（立即数类型）**：用于寄存器-立即数操作和加载指令(立即数范围是12位有符号整数)。
  - 格式(ALU 和 Load 格式一样)：`imm[11:0] | rs1 | funct3 | rd | opcode`
  - 12 + 5 + 3 + 5 + 7 = 32
    - **ALU I-type**
    - rd<-rs1(func3)I-imm\[11:0\], funct3 决定了 rs1 和立即数 I-imm 的计算符号是什么，比如 addi/subi/xori ... 之类的
    - 立即数：`I-imm = signExtend(inst[31:20])`（符号扩展）
    - 立即数是 **12位的带符号整数(-2048~2047)**，但是计算的时候要符号扩展32位，人话说就是非负数在高20位补0，负数在高20位补1，我们在上一章整数表示的时候讲过这种方法
    - 例子：`addi x1, x2, 100`，`lw x1, 100(x2)`
    - **特殊情况 SLLI / SRLI / SRAI 左移，算术右移，逻辑右移**
    - rd ← rs1 (funct3, inst\[30\]) I-imm\[4:0\](RV32最多只能左移/右移32位，所以只需要5位; 如果是 RV64 就需要6位 bit 来实现64位移动了)
    - (funct3, inst\[30\]) 来决定是这三种中的哪一种,int\[30\] 主要是用来区分右移类型的
    
    - **Load I-type** 
    - opcode = LOAD: rd ← mem\[rs1(base) + I-imm(offest)\], I-imm 是偏移量,funct3 是用来确定指令的，比如 lw/lb/lbu ...
    - 举个例子,`lw x5, 8(x2)` 表示 x5 存储x2 + 8地址的东西，x2->base 8-> offset

- **S-type（存储类型）**：用于存储指令。
  - 格式：`imm[11:5] | rs2 | rs1 | funct3 | imm[4:0] | opcode`
  - 7 + 5 + 5 + 3 + 5 + 7 = 32 bit
  - opcode = STORE: mem\[rs1 + S-imm\] ← rs2, 把 rs2 里的值，存到内存地址 = (rs1 的值 + 偏移 S‑imm) 这个位置
  - 把 12 位立即数 **拆成两段**：高 7 位放在 bit\[31:25\]（那块叫 `imm[11:5]`）低 5 位塞到原来 `rd` 那块位置（bit\[11:7\]，叫 `imm[4:0]`）
  - 立即数：`S-imm = signExtend({inst[31:25], inst[11:7]})` 先把指令里的这两段 bit 拼起来，**得到一个 12 位的 S‑imm**，再符号扩展成 **32 位**来用。
  - 例子：`sw x1, 100(x2)`
  - **有没有注意到和上面的 Load I-type 有点镜像的关系**

- **B-type（分支类型）**：用于分支指令（如 beq, bne）。
  - 格式：`imm[12|10:5] | rs2 | rs1 | funct3 | imm[4:1|11] | opcode`
  - 立即数：偏移量相对于 PC，以 2 字节为单位缩放。
  - 分支范围：±4KB（$±2^{10} \times 32\text{-bit instructions}$）。
  - 在后面部分有详细的补充
- **J-type（跳转类型）**：用于直接跳转指令（jal）。
  - 格式：`imm[20|10:1|11] | imm[19:12] | rd | opcode`
  - 立即数：`J-imm = signExtend({inst[31], inst[19:12], inst[20], inst[30:21], 1'b0})`
  - 跳转范围：±1MB（$±2^{20}\text{ bytes}$）。
- **U-type（高位立即数类型）**：用于 lui 和 auipc 指令。
  - 格式：`imm[31:12] | rd | opcode`
  - 立即数：`U-imm = {inst[31:12], 12'b0}`（左移 12 位）

### 一个重要问题，寄存器是否越多越好？
流水线实现需要更多寄存器，以避免或最小化寄存器依赖 pipelined implementations require more registers so as to avoid or minimizeregister dependencies.
现代编译器知道如何有效地将变量分配给寄存器。 因此，编译器确实需要更多的寄存器
Modern compilers know how to allocate variables to registers effectively. So themore registers are indeed demanded by the compilers.

**但是，更多的寄存器意味着:**  
- 过程调用/返回期间需要更多的保存/恢复
- 上下文切换(context switch)期间需要更多的保存/恢复
- 更长的指令格式（需要更多的位数用于操作数指示符) (longer instruction format) 
- 更长的访问时间
- 从内存中取出数据会成为时间瓶颈，而不是把变量存进寄存器
## 9. 关键指令详解
其实这里很多都在前面补充了更详细的，但是还是保留吧问题也不大(笑死)
### 算术逻辑指令（ALU Instructions）
- **R-type**：`opcode = 0110011`
  - `funct3` 决定操作：ADD（000）、SUB（000，但 funct7=0100000）、SLT、SLTU、AND、OR、XOR、SLL、SRL、SRA。
  - 例子：`add x1, x2, x3`：x1 = x2 + x3。
- **I-type（立即数）**：`opcode = 0010011`
  - 操作：ADDI、SLTI、SLTIU、ANDI、ORI、XORI、SLLI、SRLI、SRAI。
  - 移位指令：`shamt` 字段（RV32 为 5 位，RV64 为 6 位）。

### 加载/存储指令（Load/Store Instructions）
- **加载（I-type）**：`opcode = 0000011`
  - `funct3`：LW（字）、LB（字节）、LBU（无符号字节）、LH（半字）、LHU（无符号半字）。
  - 公式：`rd ← mem[rs1 + I-imm]`
- **存储（S-type）**：`opcode = 0100011`
  - `funct3`：SW、SB、SH。
  - 公式：`mem[rs1 + S-imm] ← rs2`

### 分支指令（Branch Instructions）
- **B-type**：`opcode = 1100011`
  - `funct3`：BEQ、BNE、BLT、BGE、BLTU、BGEU。
  - 注意：没有 BLE 和 BGT，可通过交换操作数实现。
  - 偏移量计算：`PC ← PC + (imm * 2)`，以 2 字节为单位。
  - 分支条件：比较 rs1 和 rs2。

### 跳转指令（Jump Instructions）
- **JAL（J-type）**：跳转并链接（用于函数调用）。
  - `opcode = 1101111`
  - 操作：`rd ← PC + 4`；`PC ← PC + J-imm`。
  - 大约能覆盖 -1Mb~1MB 的地址
- **JALR（I-type）**：间接跳转并链接（用于返回和函数指针）。
  - `opcode = 1100111`
  - 操作：`rd ← PC + 4`；`PC ← (rs1 + I-imm) & ~0x01`。
注意这俩玩意的指令类型可不是一样的，一个是 I 一个是 J
### JALR 的一些讲解
JALR 是一条 **I-type** 指令，属于 **间接跳转 (Indirect Branch)**。
简单来说，它的作用是：**“跳到新地址去执行，同时记下回来的路”**。
#### 1. 核心动作 (同时进行)

JALR 指令在执行时会同时完成以下两件事：
##### 动作一：留后路 (保存返回地址)
$$ rd \leftarrow pc + 4 $$

*   **含义**：将 **下一条指令的地址** (`pc + 4`) 保存到目标寄存器 `rd` 中。
*   **作用**：为了将来能从子函数跳回来继续执行（就像在地图上插个旗，标记“我从这走的”）。
##### 动作二：跳过去 (跳转到目标地址)
$$ pc \leftarrow (rs1 + \text{I-imm}) \ \& \ \sim 0x01 $$

*   **含义**：更新程序计数器 (`pc`)，让 CPU 跳转到新的位置。
    *   `rs1`：基址寄存器，存放目标地址的大头。
    *   `I-imm`：立即数偏移量，通常用于微调地址。
    *   `& ~0x01`：**强制对齐**。将计算结果的最低位设为 0，确保跳转地址是偶数（半字对齐），防止地址错误。
#### 2. 常见应用场景

##### 场景 A：函数返回 (Return Branch)
这是最常用的场景。当子函数执行完毕，需要返回调用者时使用。

*   **公式简化**：
    $$ pc \leftarrow (rs1) \ \& \ \sim 0x01 $$
*   **说明**：此时偏移量 (`I-imm`) 设为 0。`rs1` 里存的就是之前的返回地址（通常是 `ra` 寄存器）。

##### 场景 B：跳转表 (Branch Table) / 函数指针
用于实现 `switch-case` 语句或动态函数调用。

*   **公式**：
    $$ pc \leftarrow (rs1 + \text{I-imm}) \ \& \ \sim 0x01 $$
*   **说明**：通过 `rs1`（基地址）加上 `I-imm`（索引/偏移），计算出要跳转到表格中的哪一项。

### 高位立即数指令
- **LUI（U-type）**：加载高位立即数。
  - `opcode = 0110111`
  - 操作：`rd ← U-imm`（将立即数左移 12 位后放入 rd 的高位）。
- **AUIPC（U-type）**：将 PC 与高位立即数相加。
  - `opcode = 0010111`
  - 操作：`rd ← PC + U-imm`（用于远距离跳转）。
  - 当出现 0x800 (2048)的时候，可以使用 减去 -2048 来避免有符号12位整数只到 2047 的问题
当然，上面这些东西全都可以用 li x1 imm 解决，这是一个伪指令，编译器会自动转化成需要的语句(可恶)

## 10. 立即数编码公式 
- **I-type**：$I\text{-}imm = \text{signExtend}(inst[31:20])$
- **S-type**：$S\text{-}imm = \text{signExtend}(\{inst[31:25], inst[11:7]\})$
- **B-type**：$B\text{-}imm = \text{signExtend}(\{inst[31], inst[7], inst[30:25], inst[11:8]\}) \times 2$
- **J-type**：$J\text{-}imm = \text{signExtend}(\{inst[31], inst[19:12], inst[20], inst[30:21], 1'b0\})$
- **U-type**：$U\text{-}imm = \{inst[31:12], 12'b0\}$（即左移 12 位）

## 11. 数据对齐（Data Alignment）
- **自然对齐（Natural alignment）**：数据元素应存储在大小为其整数倍的地址上。
  - 例如：LW/SW 地址是 4 的倍数，LD/SD 地址是 8 的倍数。
- 优点：简化硬件，避免缓存行跨越（cache line crossing）和双重页错误（double page faults），以及更好的缓存行利用率(Better cache line utilization)
- RISC-V 不强制对齐，但其编译器通常保证对齐。
- 数据对齐能避免**边界穿越(Boundary Crossing)**

### 内存对齐 (Stack / Frame Alignments 栈/帧对齐)
**核心逻辑：** 电脑 CPU 喜欢“整存整取”，为了效率，内存地址必须“对齐”。
*   **什么是对齐？**
    *   CPU 读取数据不是一个字节一个字节读的，而是一次抓一大把（比如 16 字节）。
    *   为了不跨区读取（避免读两次再拼接），数据必须从特定的“桩”（例如 16 的倍数地址）开始存放。
#### 结构体对齐(Struct Alignments) 
- **内部数据对齐：成员的起始地址必须是自身大小的整数倍 (可以是0倍)**
- **结构体总大小对齐: 结构体总大小必须是其最大成员大小的整数倍**
- 例子：node{int x;long long y;char c}
- x 的起始地址是0，占据了\[0,3\]的位置,y 大小是8字节，所以其必须从8开始算，占据了\[8,15\]
- char 是一位的，占据了16，现在结构体的大小是 17,这并不是 8 的倍数
- 所以这个结构体的size应该是下一个8的倍数，也就是 24,只有空出的部分一般叫padding，内存里应该也是空的
     
*   **关键指针：**
    *   **SP (栈指针)**：当前工作台的边界。
    *   **FP (帧指针)**：当前函数的“门口”。
    *   现代系统（x86-64, RISC-V 等）通常要求这些指针地址必须是 **16 字节对齐**。
*   **谁来保障？**
    *   **ABI (应用程序二进制接口)**：这是“交通规则”，规定必须对齐。
    *   **CRT (C运行时库)**：这是“后勤组”，在运行 `main` 函数前，先把栈空间整理好，确保符合对齐要求。

### 编译器重排 (Compiler Reordering)
**核心逻辑：** 编译器会为了省空间或提速调整变量顺序，但有严格的底线。
*   **独立变量（Local Variables） -> 允许重排**
    *   例如：`void func() { int a; char b; int c; }`
    *   编译器可以像打包行李一样，为了塞得更紧凑、读写更快，随意调整 `a, b, c` 在内存中的顺序。因为这些变量只在函数内部使用，外部看不见。
*   **结构体字段（Struct Fields） -> 禁止重排**
    *   例如：`struct { int a; char b; }`
    *   **绝对不能动**。代码里先写的谁，内存里就必须先放谁，地址必须递增。
    *   **原因**：
        1.  **二进制兼容性**：如果把结构体传给其他模块或硬件，对方是按顺序读的，乱序会导致数据读错。
        2.  **指针规则**：C 语言标准规定，结构体的地址必须等于它第一个成员的地址。

## 补充：指令操作数格式 (Instruction Formats)
**核心逻辑：** CPU 执行一条指令涉及几个变量？
*   **0/1 操作数 (Stack 架构)**
    *   **机制**：默认操作“栈顶”的数据。
    *   **特点**：不需要指名道姓，默认拿最上面的用。
*   **2 操作数 (`A = A + B`)**
    *   **机制**：把 B 加到 A 上。
    *   **缺点**：**破坏性更新**，A 原来的值会被覆盖。
*   **3 操作数 (`A = B + C`)**
    *   **机制**：B 加 C 放入 A。
    *   **优点**：**RISC-V 的主流格式**。保留了 B 和 C 的原值，逻辑清晰。
*   **4 操作数 (`A = B * C + D`)**
    *   **机制**：专门为 FMA 这种特殊指令设计的格式。

## 补充: FMA 指令 (Fused-Multiply-Add)
**定义**：一条指令一步完成“先乘后加”操作 (`Result = A * B + C`)。
**核心优势 (Why FMA?)：**
1.  **应用广泛**：它是**矩阵乘法**（AI、深度学习、图形处理）中最基础的原子操作。
2.  **速度更快**：将原本需要的 2 条指令（先乘、后加）合并为 1 条，减少了一半的指令数。
3.  **精度更高 (重点)**：
    *   **分步算**：乘法四舍五入一次 + 加法四舍五入一次 = **两次误差**。
    *   **FMA算**：中间不截断，算完所有步骤后**只做一次四舍五入** = **误差更小**。

补充一个**逻辑右移(SRL)** 和**算术右移(SRA)** 的区别
- SRL：无论如何高位都是补0
- SRA: 非负数高位补0，负数高位补1

## 补充: 分支指令 (Branch Instructions)

### 1. 基础概念 
*   **功能**：实现程序的逻辑控制（如 `if-else`, `loop`）。
*   **指令示例**：`beq x1, x2, Label` (Branch if Equal)。
    *   **逻辑**：如果寄存器 `x1 == x2`，则跳转到 `Label`；否则继续执行下一行。
*   **寻址方式**：**PC-relative Addressing (PC 相对寻址)**。
    *   跳转目标并非绝对地址，而是相对于当前程序计数器 (PC) 的偏移量。
### 2. 偏移量计算细节 
*   **计算公式**：`Target = PC + Offset`
*   如果分支跳转了, PC = PC + imm * 2
-   没有跳转，PC = PC + 4(下一条指令在4个字节之后)
*   **存储技巧 (重点)**：
    *   RISC-V 指令长度是 2 或 4 字节，地址必定是 2 的倍数 (偶数)。
    *   **优化**：偏移量 (Immediate) 存储时**省略最低位的 0**，以 **2 字节 (Half-word)** 为单位计数。
    *   **收益**：相同的位数 (12 bits) 可以表示的跳转范围**翻倍** (±4KB)
    * (在这个地方 ppt 讲了一个 imm * 4 的例子，当所有指令都是 32b 的时候这很方便，但遗憾的还有很多 16b 的指令,所以只能 * 2 )
### 3. 远距离跳转问题 
*   **限制**：普通的 Branch 指令 (SB-type) 只能跳较短的距离 (约 ±$2^{10}$ 条指令)。
*   **解决方案**：如果目标太远 (`Target > limit`)，编译器会自动重写代码。
    *   **转换逻辑**：
        1.  **反转条件**：先用一个短跳转处理“不满足”的情况。
        2.  **插入长跳**：在满足条件的分支里，使用无条件跳转指令 **`J` (Jump)** 或 **`JAL`**，它们拥有更大的寻址范围 (±1MB)。
    *   **代码示例**：
        ```assembly
        // 原意：如果相等，跳去远方
        beq x10, x0, far_away

        // 实际编译结果：
        bne x10, x0, NEXT   // 如果不等，跳过下一行(短跳)
        j far_away          // 否则(即相等)，用 J 指令飞过去(长跳)
        NEXT: ...
        ```

## 补充： Memory Operands（内存操作数）

- **简单变量**（如单个 int）可以直接放在寄存器里，用寄存器名（x1、x2 等）访问。
- **数组 / 字符串 / 结构体** 这类复杂数据一般放在**内存**中，而不是寄存器里。
- 要用这些数据时：
  - 先把它们在内存中的**起始地址**放进某个寄存器（称为基址寄存器），
  - 再用 **load / store 指令**，通过「基址 + 偏移量」的方式访问。
### 例子：访问 A\[8\]
- 假设：
  - `A` 是一个 32 位整型数组（每个元素 4 字节），
  - `A` 的起始地址在寄存器 `x1` 里。
- 访问 `A[8]` 时，使用：
  - `lw x2, 32(x1)`
- 含义：
  - `x1`：数组 A 的起始地址；
  - `32`：**相对于起始地址的字节偏移量**；
  - `lw`：从地址 `x1 + 32` 读一个 32 位整数（4 字节）到 `x2`。
- 为什么是 32？
  - 偏移量 = 下标 × 每个元素的字节数
  - 对 `A[8]`：偏移量 = `8 × 4 = 32` 字节。
- **关键点：偏移量单位是“字节”而不是“第几个元素”。**
  - 所以访问 `A[8]` 要写 `32(x1)`，**不能写** `8(x1)`。
## 12. 应用二进制接口（ABI）
- **ABI（Application Binary Interface）**：定义函数调用时寄存器的使用规则。
- **寄存器别名**：
  - **a0-a7**：函数参数寄存器（caller-saved）。
  - **a0, a1**：函数返回值寄存器。
  - **s0-s11**：保存寄存器（callee-saved）。
  - **t0-t6**：临时寄存器（caller-saved）。
  - **ra**：返回地址寄存器（caller-saved）。
  - **sp**：栈指针寄存器（callee-saved）。
  - **gp**：全局指针，**tp**：线程指针。

## 13. 示例：GCD 算法（C 和 RISC-V 汇编）
- **C 代码**：
  ```c
  int gcd(int a, int b) 
  {
      int t;
      while (a != 0) 
      {
          if (a >= b) 
          {
              a = a - b;
          } 
          else 
          {
              t = a;
              a = b;
              b = t;
          }
      }
      return b;
  }
  ```

- **RISC-V 汇编关键点**：使用循环、分支指令（beq, bge）、算术指令（sub）。

## 14. 异常处理（Exception Handling）

- **非法指令异常**：当执行未实现的指令时（如 RV32I 机器执行 RV32IM 的 mul 指令）
- **异常处理流程**
    1. 硬件将控制权转移到通用中断处理程序（common handler）。
    2. 保存所有 GPR 到内存。
    3. 根据 mcause 和 mepc 处理特定中断。
    4. 恢复 GPR，执行 ERET 返回。

## 15. 关键公式与概念（考试重点）

- **立即数扩展公式**：如上所述，务必记住 I、S、B、J、U 类型的立即数计算。
- **分支偏移计算**：$PC \leftarrow PC + (imm \times 2)$，范围 ±4KB。
- **跳转偏移计算**：JAL 范围 ±1MB，JALR 间接跳转。
- **寄存器 x0**：始终为 0，用于简化指令。
- **ABI 寄存器约定**：caller-saved vs callee-saved，函数参数和返回值寄存器。
- **指令格式字段**：opcode、funct3、funct7、rs1、rs2、rd、imm 的位置和含义。

## 16. 思考问题（考试可能涉及）

- **为什么 RISC-V 只有 JAL 和 JALR，没有普通跳转指令？**
- 因为 JAL (Jump and Link) 可以通过将返回地址保存到 `x0` 寄存器（该寄存器恒为 0，即丢弃返回地址）来实现无条件的普通跳转，这样简化了指令集设计。
- **为什么没有 BLE 和 BGT 指令？**
- 为了减少硬件比较器的复杂度。通过交换 `BLT` (小于则跳转) 和 `BGE` (大于等于则跳转) 的两个操作数顺序，即可在逻辑上实现 BLE 和 BGT 的效果。
- **如何设置大立即数到寄存器（使用 LUI 和 ADDI）？**
- 先使用 `LUI` 指令将立即数的高 20 位加载到目标寄存器的高位，随后使用 `ADDI` 指令加上低 12 位的立即数，从而合成一个完整的 32 位常数。
- **数据对齐的重要性是什么？**
- 数据对齐能确保 CPU 能够通过单次内存访问完成读取，避免因跨**缓存行(cache line)** 或内存页访问导致的性能下降，并简化了内存控制器的硬件设计。
- **更多寄存器（如 1024 个）的利弊是什么？**
- **利：** 可以减少程序运行中因寄存器不足而产生的访存操作（Spilling），提高计算效率。
- **弊：** 会增加指令中寄存器索引字段的长度（导致指令变长或数量减少），同时增大硬件芯片面积、功耗并可能降低主频。

---

**总结**：RISC-V 是一种简单、模块化、可扩展的 RISC ISA。关键点包括固定长度指令、加载/存储架构、多种指令格式、立即数编码、ABI 约定以及异常处理。掌握指令格式和立即数计算是考试重点。