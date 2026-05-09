# CSC3060 Chapter 5：程序性能优化 I（Optimizing Program Performance）

> 这份笔记用"傻子也能看懂"的方式，把整章的核心知识点、公式和考点一一讲清楚。每个概念都配上中英文对照，方便默写。

---

## 一、为什么要做程序优化（Program Optimization）

一句话理解：**写出能跑对的程序只是第一步，写出跑得快的程序才是真本事。**

关键观念：

- 软件层面的优化，有时候能带来"几十倍甚至几百倍"的性能提升（orders-of-magnitude gains），比换硬件划算得多。
- 在性能敏感的系统（HPC、搜索引擎、高频交易等）里，哪怕 1% 的提升都价值巨大。

### DeepSeek 的例子（必记亮点）

DeepSeek 用 **2K GPU** 就做到了别人 **16K GPU** 才能做的事。大家以为是模型架构（MoE、GQA）带来的提升，但**真正的大头来自系统级性能优化**：

|优化类型|贡献占比|关键技术|
|---|---|---|
|**PTX & 系统级**|**60-70%**|直接写 PTX、线程优化、cache 管理|
|模型架构（Model Architecture）|20-30%|MoE、GQA|
|训练技术（Training Techniques）|10%|多步调度、强化学习|

**考点**：系统级优化贡献最大（60-70%），不是模型架构。

### 现实中优化有多重要（记几个数字就够了）

- **Google**：搜索页多 0.5 秒 → 流量掉 20%
- **Amazon**：每多 100 ms 延迟 → 销售额减少 1%
- **高频交易（HFT）**：微秒到毫秒级的差距就是生死
- **Netflix**：视频编码提升一点点 → 每年省几百万美元 CDN 费用

---

## 二、编译器优化选项（Compiler Optimization Flags）⭐考试高频

### 1. `-On` 分级优化

|选项|含义|
|---|---|
|`-O0`|**不优化**，编译超快，生成的代码用于调试（debugging）|
|`-O1`（等同 `-O`）|基础优化（Basic optimizations）|
|`-O2`|**推荐默认值**，高度优化、安全稳定、代码膨胀不大|
|`-O3`|**激进优化（Aggressive）**，包含循环展开（loop unrolling）和 SIMDilization|
|`-O4`|以前指 LTO；现在 `-O4 = -O3 + -flto`|

**`-O3` 的代价（考点）**：

- 代码更大
- 编译时间更长
- 调试更难
- **有时候运行性能反而更差**（激进变换不保证更快）

### 2. 其他重要 Flags

|Flag|作用|
|---|---|
|`-O3 -flto`|**Link-Time Optimization（链接期优化，LTO）**，全程序优化|
|`-march=native`|针对当前 CPU 架构生成专属指令（**Target CPU-specific instructions**）|
|`-mtune=native`|为当前 CPU 优化指令调度，但**不破坏与旧 CPU 的兼容性**|
|`-fprofile-generate`|插桩编译，生成 profile 数据|
|`-fprofile-use`|**Profile-Guided Optimization（PGO，剖析引导优化）**，用 profile 指导优化|
|`-Ofast`|等价于 `-O3 -ffast-math`，允许编译器**重排浮点运算**|

**记忆点**：`-march` 生成特定指令（可能不兼容旧 CPU），`-mtune` 只调度指令（保持兼容）。

---

## 三、编译器做不了什么（Limits of Compiler Optimization）⭐必考

这是考试经常问的"为什么编译器不能帮你优化掉某段代码？"

### 核心原则

编译器必须**严格保持程序语义（semantics）**——优化后的程序行为必须和原程序"一模一样"。

### 编译器办不到的五大场景

1. **不能改算法（Algorithm）** —— 冒泡排序它就给你冒泡排序，不会自动换成快排
2. **必须遵守语言规则（语义保持）**，下面这些情况是拦路虎：
    - **内存别名（Memory Aliasing）** ← 最重要的考点
    - **浮点结合律（FP Associativity）**：浮点加法不满足结合律，编译器不能随便换顺序
    - **函数副作用（Function Side Effects）**
    - **Volatile 变量**
    - **内存一致性（Memory Consistency）**
3. **缺少运行时和领域知识**
4. **很多优化问题是 NP-hard**：最优寄存器分配、最优代码调度、内存布局、phase ordering
5. **边界跨越问题（Boundary Crossing）**：通常一次只分析一个函数（separate compilation）；LTO 可以解决但贵

---

## 四、编译器优化的总体目标（General Goals）⭐核心考点

三大方向，每个方向下有具体手段：

### 1. 最小化指令数（Minimize Instructions）

|优化|全称|中文|核心思想|
|---|---|---|---|
|**CSE**|Common Subexpression Elimination|公共子表达式消除|重复计算只算一次|
|**DCE**|Dead Code Elimination|死代码消除|不执行的代码就别生成|
|**SR**|Strength Reduction|强度削减|慢指令换快指令（如乘法换加法）|

### 2. 最小化执行周期（Minimize Execution Cycles）

| 优化                                   | 全称                  | 中文         | 核心思想        |
| ------------------------------------ | ------------------- | ---------- | ----------- |
| **RA**                               | Register Allocation | 寄存器分配      | 尽量把数据放寄存器里  |
| **CS**                               | Code Scheduling     | 代码调度       | 利用 ILP 掩盖延迟 |
| Locality Improvement                 |                     | 局部性改进      | Cache 友好访问  |
| Preload & Redundant Load Elimination |                     | 预加载和冗余加载消除 | 提前加载、只加载一次  |

### 3. 避免分支（Avoid Branching）

- 用 **条件移动（conditional move，x86）** 或 **条件执行（conditional execution，ARM）**
- **循环展开（loop unrolling）**、**过程内联（procedure inlining）**、**unswitching** 可以减少分支

---

## 五、两大图：CFG 和 DFG ⭐重要概念

|图|全称|建模什么|
|---|---|---|
|**CFG**|Control Flow Graph（控制流图）|程序的**执行路径**（往哪走）|
|**DFG**|Data Flow Graph（数据流图）|信息的**流动**（携带什么值）|
|**DDG**|Data Dependency Graph（数据依赖图）|指令**执行顺序约束**，用于代码调度|

**考试要点**：

- CFG = 程序走到哪里
- DFG = 程序带着什么值走
- 两个图一起看 → 编译器能做 `-O1/-O2/-O3` 几乎所有优化的数学基础
- DFG 支持：CSE、DCE、到达定值（Reaching Definitions）、寄存器分配

### 本地 vs 全局优化

|类型|范围|例子|
|---|---|---|
|**本地优化（Local）**|单个基本块（basic block）内|常量折叠、强度削减、死代码消除、local CSE|
|**全局优化（Global）**|整个函数的 CFG|循环变换、代码移动、global CSE|

---

## 六、具体优化方法逐个击破 ⭐⭐⭐重点默写区

### 1. 常量折叠（Constant Folding）

**思想**：编译器在编译时就把常量算出来。

```c
long mask = 0xFF << 8;        →    long mask = 0xFF00;
size_t namelen = strlen("Harry Bovik");   →    size_t namelen = 11;
```

**任何输入都是常量的表达式都能折叠**，甚至能消除库函数调用。
也就是说编译的时候这个值直接被常量替换了

### 2. 死代码消除（Dead Code Elimination, DCE）

**两种情况**：

**① 永远不会执行的代码**：

```c
if (0) { puts("Kilroy was here"); }      // 删掉
if (1) { puts("Only bozos on this bus"); }   // 去掉 if
```

**② 结果被覆盖的代码**：

```c
x = 23;       // 删掉
x = 42;
```

**为什么看起来很傻但真有用？**

- 一般来说程序员不会主动干这种事情（除非是没注意）
- 其他优化可能产生这种代码
- 条件编译可能产生
- 两个赋值可能离得很远（你肉眼看不出来）

### 3. 公共子表达式消除（Common Subexpression Elimination, CSE）⭐

**思想**：重复的计算只做一次。

```c
norm[i] = v[i].x*v[i].x + v[i].y*v[i].y;
// 优化后：
elt = &v[i];
x = elt->x;
y = elt->y;
norm[i] = x*x + y*y;
```

**重要提醒**：**编译器做 CSE 非常厉害，程序员不要自己手动做**（否则代码难读还可能帮倒忙）。
说你呢 CSC3200

**更多例子（考试可能让你识别公共子表达式）**：

```c
// 例1: (x1-x2) 和 (y1-y2) 各是一个
float dist2 = (x1 - x2) * (x1 - x2) + (y1 - y2) * (y1 - y2);

// 例2: A+B 是公共子表达式
A = (A+B+C) * (A+B+D) - (A+D+B);

// 例3: 三维数组索引 i*(M*N) + j*N + k 是公共子表达式
A = B[i][j][k] + C[i][j][k];

// 例4: A->b->c 是公共子表达式
A = A->b->c->e + A->b->c->d;
```

### 4. 循环不变代码外提（Loop Invariant Code Motion, LICM）⭐

**思想**：如果某个表达式每次循环结果都一样，就把它挪到循环外面。

```c
for (j = 0; j < n; j++)
    a[n*i+j] = b[j];   // n*i 每次循环都算，但值不变
    
// ↓ 优化为

int ni = n*i;           // 挪出来
for (j = 0; j < n; j++)
    a[ni+j] = b[j];
```

**条件**：只有在"每次迭代都产生同样结果"时才有效。

### 5. 过程内联（Procedure Inlining）⭐

**思想**：把函数体直接复制到调用处。

**好处**：

- 消除函数调用开销
- 为其他优化**打开新机会**（常量传播、CSE、DCE 等）

**坏处**：

- 代码膨胀（code bloat）
- 可能变慢（因为 i-cache miss 变多）

**经典例子**：

```c
int pred(int x) {
    if (x == 0) return 0;
    else return x - 1;
}
int func(int y) {
    return pred(y) + pred(0) + pred(y+1);
}
```

内联后，编译器能发现：

- `pred(0)` 的 `0 == 0` 永远为真 → 做常量折叠 → 结果就是 0
- `pred(y+1)` 里 `y+1 == 0` 和 `(y+1)-1` 可以简化
- 最终可以简化成很短的代码

**是否内联的权衡（To Inline or Not）**：

- 函数大小（Function size）
- 执行频率（Execution frequency）：循环内的调用值得内联；错误处理的不值得
- 调用点数量（Number of call sites）
- 常量参数（Constant Arguments）：常量参数能触发二次优化
- 递归和深度限制（Recursion and Depth Limits）
- **GPU 内核倾向于全部内联**

---

## 七、内存别名（Memory Aliasing）⭐⭐⭐超高频考点

这是**全章最核心的概念之一**。

### 什么是 Aliasing？

**两个指针指向同一块内存**，编译器不敢确定它们指向的不是同一个地方，就**不敢做优化**。

### 经典例子：矩阵行求和

```c
void sum_rows1(double *a, double *b, long n) {
    long i, j;
    for (i = 0; i < n; i++) {
        b[i] = 0;
        for (j = 0; j < n; j++)
            b[i] += a[i*n + j];   // 每次都 load b[i]、store b[i]
    }
}
```

**为什么编译器不把 `b[i]` 放进寄存器？**

因为编译器不知道 `a` 和 `b` 有没有重叠！比如如果 `b = &a[i*n]`，修改 `b[i]` 就会影响 `a[i*n + j]` 的值，所以**必须每次都读写内存**。

**没优化的汇编（每次都 LOAD/STORE b[i]）**：

```asm
.L_inner_loop:
    fld     fa5, 0(a3)          # 1. LOAD b[i] 到寄存器
    fld     fa4, 0(a4)          # 2. LOAD a[...] 到寄存器
    fadd.d  fa5, fa5, fa4       # 3. 相加
    fsd     fa5, 0(a3)          # 4. STORE 回 b[i]（每次都写！）
    addi    a4, a4, 8           # 5. 指针前进 8 字节
    bne     a4, a5, .L_inner_loop  # 6. 循环
```

**优化后（b[i] 放寄存器 fa5）**：

```asm
.L_inner_loop:   // b[i] 常驻寄存器 fa5
    fld     fa4, 0(a4)
    fadd.d  fa5, fa5, fa4
    addi    a4, a4, 8
    bne     a4, a5, .L_inner_loop
```

少了两条内存访问指令，性能大幅提升。

### 怎么知道是 aliasing 在捣乱？（工具使用）

- **GCC**：
    
    ```bash
    gcc -O3 -fopt-info-missed=missed_opts.txt my_program.c
    ```
    
    看到 **"data dependence"** 或 **"dependent memory operations"** 就是 aliasing 的锅。
    
- **Clang/LLVM**：
    
    ```bash
    clang -O3 -Rpass-missed=.* -Rpass-analysis=.* my_program.c
    ```
    

### 如何避免 Aliasing 惩罚（两招）⭐

**招 1：用局部变量累加中间结果**

```c
void sum_rows2(double *a, double *b, long n) {
    long i, j;
    for (i = 0; i < n; i++) {
        double val = 0;          // 局部变量，不涉及 b[i]
        for (j = 0; j < n; j++)
            val += a[i*n + j];
        b[i] = val;              // 循环结束才写一次
    }
}
```

**招 2：用 `restrict` 关键字**

```c
void sum_rows1(double *restrict a, double *restrict b, long n)
```

`restrict` 的意思是**"这个指针是访问该内存的唯一指针，保证不 alias"**，编译器就敢优化了。

### 函数调用不能外提的情况

```c
// 坏例子：O(n²)，因为 strlen 每次都算
void lower_quadratic(char *s) {
    for (i = 0; i < strlen(s); i++) { ... }
}

// 好例子：O(n)
void lower_linear(char *s) {
    size_t n = strlen(s);    // 先算一次
    for (i = 0; i < n; i++) { ... }
}
```

**为什么编译器自己不做？**因为 `s[i]` 的赋值可能改变字符串长度（如果 `s` 和内部某全局状态 alias），编译器不敢假设 `strlen(s)` 不变。

---

## 八、现代 CPU 设计（简要了解）

现代 CPU 用**超标量（superscalar）+ 乱序执行（out-of-order）**，核心部件：

- **指令控制（Instruction Control）**：Fetch、Decode、分支预测
- **执行单元（Functional Units）**：多个 Arith（算术）、Load、Store 并行
- **Retirement Unit（退休单元）**：按顺序提交结果，更新寄存器

**关键概念**：

- **ILP（Instruction-Level Parallelism，指令级并行）**
- **Reorder Buffer（ROB，重排缓冲区）**
- **调度策略**：ROB 空了 → 优先 latency hiding（掩盖延迟）；ROB 满了 → 优先 retirement

---

## 九、循环展开（Loop Unrolling）⭐

**思想**：把循环体复制多份，减少循环次数。

**原始**：

```c
for (size_t i = 0; i < nelts; i++) {
    A[i] = B[i]*k + C[i];
}
```

**4 倍展开**：

```c
for (size_t i = 0; i < nelts - 4; i += 4) {
    A[i  ] = B[i  ]*k + C[i  ];
    A[i+1] = B[i+1]*k + C[i+1];
    A[i+2] = B[i+2]*k + C[i+2];
    A[i+3] = B[i+3]*k + C[i+3];
}
```

**好处**：

- 摊销循环条件判断的成本（amortize loop condition cost）
- 创造 CSE、代码移动、调度机会
- **为 SIMDilization 做准备**

**坏处**：

- 代码变大
- 编译时间变长

**正确性条件**：每次迭代之间没有依赖（否则展开后会错）。

---

## 十、指令调度（Instruction Scheduling）⭐

**思想**：重排指令，让 CPU 的各个执行单元都忙起来。

**经典做法**：展开后，**把所有 Load 放最前面**，让内存访问和计算重叠。

```c
for (size_t i = 0; i < nelts - 4; i += 4) {
    B0 = B[i]; B1 = B[i+1]; B2 = B[i+2]; B3 = B[i+3];   // 先全部 Load
    C0 = C[i]; C1 = C[i+1]; C2 = C[i+2]; C3 = C[i+3];
    A[i  ] = B0*k + C0;                                  // 再算
    A[i+1] = B1*k + C1;
    A[i+2] = B2*k + C2;
    A[i+3] = B3*k + C3;
}
```

**这也说明了为什么现代 CPU 需要很多寄存器**（Load 的中间结果要放地方）。

**考点**：展开和调度是**编译器该做的事**。现代 GCC/Clang/MSVC 比人类做得好，**你只需要加 `-march=native -mtune=native`**。

---

## 十一、性能测量：CPE（Cycles Per Element）⭐⭐必考公式

### 定义

**CPE = Cycles Per Element（每元素周期数）**，衡量对向量/列表操作的程序性能。

### 核心公式

$$ \boxed{\text{Cycles} = \text{CPE} \times n + \text{Overhead}} $$

其中：

- $n$ = 元素个数（向量长度）
- **CPE = 这条直线的斜率（slope）**
- **Overhead = 截距（与 $n$ 无关的固定开销）**

### 使用方式

跑不同 $n$，测总 cycles，画图：

- x 轴：$n$
- y 轴：Cycles
- **直线斜率就是 CPE**

CPE 越小 → 性能越好。

---

## 十二、Combine 系列性能对比（经典考题背景）⭐

### Benchmark 数据结构

```c
typedef struct {
    size_t len;
    data_t *data;
} vec;

int get_vec_element(vec *v, size_t idx, data_t *val) {
    if (idx >= v->len)
        return 0;
    *val = v->data[idx];
    return 1;
}
```

- `data_t` 可以是 `int`、`long`、`float`、`double`
- 操作 `OP` 可以是 `+`（单位元 0）或 `*`（单位元 1）

### Combine1（原始版）

```c
void combine1(vec_ptr v, data_t *dest) {
    long i;
    *dest = IDENT;
    for (i = 0; i < vec_length(v); i++) {    // 每次都调用 vec_length
        data_t val;
        get_vec_element(v, i, &val);          // 每次函数调用 + 边界检查
        *dest = *dest OP val;                  // 每次写内存
    }
}
```

**问题**：

1. `vec_length(v)` 每次循环都调用
2. 每次都做边界检查
3. `*dest` 每次都写内存（aliasing 问题）

### Combine4（基础优化版）⭐

```c
void combine4(vec_ptr v, data_t *dest) {
    long i;
    long length = vec_length(v);         // 1. 循环不变代码外提
    data_t *d = get_vec_start(v);        // 2. 直接拿数据指针
    data_t t = IDENT;                    // 3. 用局部变量累加（避免 aliasing）
    for (i = 0; i < length; i++)
        t = t OP d[i];                    // 只访问局部变量
    *dest = t;                            // 循环结束才写内存
}
```

**三个核心优化**：

1. 把 `vec_length` 移出循环
2. 避免每次迭代的边界检查
3. 用**局部临时变量累加**（破解 aliasing 问题）

### 性能对比表（CPE）⭐⭐⭐必背

|方法|Int Add|Int Mult|Double Add|Double Mult|
|---|---|---|---|---|
|Combine1 unoptimized|22.68|20.02|19.98|20.18|
|Combine1 `-O1`|10.12|10.12|10.17|11.14|
|Combine1 `-O3`|4.5|4.5|6|7.8|
|**Combine4**|**1.27**|**3.01**|**3.01**|**5.01**|
|**Unroll（2路展开）**|**0.81**|**1.51**|**1.51**|**2.51**|

**观察**：

- 从 `-O0` 到 `-O3`，性能提升 5 倍左右
- Combine4 的手动优化（移出 vec_length + 局部累加）又提升一大截
- 循环展开再提升一倍

### 2 路循环展开版本

```c
void unroll2a_combine(vec_ptr v, data_t *dest) {
    long length = vec_length(v);
    long limit = length - 1;
    data_t *d = get_vec_start(v);
    data_t x0 = IDENT;
    data_t x1 = IDENT;
    long i;
    /* 每次处理 2 个元素 */
    for (i = 0; i < limit; i += 2) {
        x0 = x0 OP d[i];
        x1 = x1 OP d[i+1];
    }
    /* 处理尾部残留元素 */
    for (; i < length; i++) {
        x0 = x0 OP d[i];
    }
    *dest = x0 OP x1;
}
```

**关键点**：

- 用两个累加变量 `x0`, `x1` → **打破了依赖链**（breaking dependency chain）
- 最后要处理边界残留元素
- 最后把两个累加器合并

---

## 十三、规约（Reduction）⭐

**什么是规约**：把数组所有元素通过某个二元运算合成一个结果。

**常见规约**：

- Sum reduction（求和）
- Product reduction（求积）
- Parity（奇偶）
- AND / OR / XOR
- Max / Min

### 标量规约 vs 向量规约

**标量规约（Scalar Reduction）**：顺序累加，依赖链长。

```c
int sum = 0;
for (i = 0; i < n; i++) sum += a[i];
```

依赖链：`sum → sum → sum → ...`（每步等上一步完成）

**向量规约（Vector Reduction）**：

- 把数组分成多段，每段并行累加到 `sum0, sum1, sum2, sum3`
- 最后把 `sum0+sum1+sum2+sum3` 合并为最终结果（**Final step**）
- 利用 SIMD（如 128-bit 向量）可以 4 路并行

**关键考点**：**向量规约是 `-O2` 或 `-O3` 的一部分**。

---

## 🎯 考试默写重点清单

### 一定要背的缩写对照表

|缩写|全称|中文|
|---|---|---|
|**CSE**|Common Subexpression Elimination|公共子表达式消除|
|**DCE**|Dead Code Elimination|死代码消除|
|**SR**|Strength Reduction|强度削减|
|**RA**|Register Allocation|寄存器分配|
|**CS**|Code Scheduling|代码调度|
|**LICM**|Loop Invariant Code Motion|循环不变代码外提|
|**LTO**|Link-Time Optimization|链接期优化|
|**PGO**|Profile-Guided Optimization|剖析引导优化|
|**ILP**|Instruction-Level Parallelism|指令级并行|
|**CFG**|Control Flow Graph|控制流图|
|**DFG**|Data Flow Graph|数据流图|
|**DDG**|Data Dependency Graph|数据依赖图|
|**CPE**|Cycles Per Element|每元素周期数|
|**SIMD**|Single Instruction Multiple Data|单指令多数据|

### 一定要背的公式

$$ \text{Cycles} = \text{CPE} \times n + \text{Overhead} $$

CPE 是曲线斜率，Overhead 是与 $n$ 无关的固定成本。

### 一定要能答的问题

1. **为什么编译器不能优化掉 `b[i]`？** → 因为 memory aliasing，编译器不知道 `a` 和 `b` 是否重叠
2. **怎么破解 aliasing？** → ① 用局部变量累加 ② 加 `restrict` 关键字
3. **`-march=native` 和 `-mtune=native` 区别？** → 前者生成特定指令（不兼容旧 CPU），后者只调度（保持兼容）
4. **`-O3` 为什么不一定最快？** → 代码膨胀、编译慢、调试难、有时反而更慢
5. **CSE 要不要程序员自己做？** → 不要，编译器做得比你好，手动做反而影响可读性
6. **循环展开的好处和坏处？** → 好：摊销循环开销、创造调度机会、为 SIMD 铺路。坏：代码膨胀、编译变慢
7. **Combine1 → Combine4 的三个优化是什么？** → ① vec_length 外提 ② 跳过边界检查 ③ 局部变量累加
8. **编译器办不到的五件事？** → 改算法、违反语义（aliasing/FP结合律/副作用/volatile/内存一致性）、缺运行时信息、NP-hard问题、跨编译单元