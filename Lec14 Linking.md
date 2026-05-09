# CSC3060 第七章 链接（Linking）完整复习笔记

> 课程：CSC3060 Computer Architecture - Chapter 7: Linking 用最直白的话讲清楚每个考点

---

## 一、什么是链接 Linking（最基础概念）

### 1.1 定义

**链接（Linking）**：把若干份代码和数据「拼装」成一个可以装入内存执行的目标文件的过程。

**链接器（Linker, ld）**：完成这个拼装工作的系统程序。

### 1.2 为什么需要链接器？（考试爱问 Why Linkers）

链接器的存在是为了支持 **分离编译（Separate Compilation）**。好处有两个层面：

**模块化（Modularity）**

- 大程序可以拆成很多小源文件，而不是一坨巨型代码
- 可以构建公共函数库，比如数学库 `/lib/libm`、C 标准库 `/lib/libc`
- 头文件（`.h`）声明类型，库里定义实现

**效率（Efficiency）**

- **时间效率**：改一个文件只需重新编译这一个，再链接一次即可。多个文件可以并行编译。
- **空间效率**：公共函数集中到库里。两种方式：
    - **静态链接（Static Linking）**：可执行文件中只包含实际用到的库代码
    - **动态链接（Dynamic Linking）**：可执行文件不含库代码，运行时多个进程共享同一份库代码

---

## 二、链接器到底干两件事（核心！）

链接器只做两件事，**这两件事是整章的主线**：

|步骤|名称|作用|
|---|---|---|
|Step 1|**符号解析 Symbol Resolution**|把每个符号「引用」对应到唯一的「定义」|
|Step 2|**重定位 Relocation**|合并各 `.o` 的代码/数据段，把符号的相对位置改成最终的绝对地址，并更新所有引用|

---

## 三、目标文件的三种形态（Three Kinds of Object Files）

|类型|后缀|说明|
|---|---|---|
|可重定位目标文件 Relocatable object file|`.o`|可以和其它 `.o` 拼装成可执行文件。一个 `.c` 文件 → 一个 `.o`|
|可执行目标文件 Executable object file|`a.out`|可直接拷贝到内存执行|
|共享目标文件 Shared object file|`.so`|特殊的可重定位文件，装入时或运行时被动态链接。Windows 里叫 **DLL（Dynamic Link Library）**|

---

## 四、ELF 目标文件格式（Executable and Linkable Format）⭐ 必考

ELF 是 Linux 下统一的二进制格式，三种 `.o`/`a.out`/`.so` 都用这个格式。

### 4.1 ELF 文件的整体结构（要会按顺序默写）

```
┌─────────────────────────────┐
│ ELF header                  │ ← 字大小、字节序、文件类型、机器类型
├─────────────────────────────┤
│ Segment header table        │ ← 可执行文件必须有，给 OS 加载用
├─────────────────────────────┤
│ .text       (代码)           │
├─────────────────────────────┤
│ .rodata     (只读数据)       │ ← 跳转表、字符串常量
├─────────────────────────────┤
│ .data       (已初始化全局)   │
├─────────────────────────────┤
│ .bss        (未初始化全局)   │ ← 只占符号表条目，不占文件空间！
├─────────────────────────────┤
│ .symtab     (符号表)         │
├─────────────────────────────┤
│ .rel.text   (.text 重定位)   │
├─────────────────────────────┤
│ .rel.data   (.data 重定位)   │
├─────────────────────────────┤
│ .debug      (调试信息)       │ ← gcc -g 才有
├─────────────────────────────┤
│ Section header table        │
└─────────────────────────────┘
```

### 4.2 各个 section 的含义（重点）

|Section|中文|内容|
|---|---|---|
|`.text`|代码段|编译好的机器指令|
|`.rodata`|只读数据段|跳转表、字符串字面量（read only data）|
|`.data`|数据段|**已初始化的**全局变量|
|`.bss`|BSS段|**未初始化的**全局变量；**有符号表条目但不占文件空间**|
|`.symtab`|符号表|函数名、静态变量名、各 section 的名字与位置|
|`.rel.text`|text 重定位|哪些指令要被改、怎么改|
|`.rel.data`|data 重定位|哪些指针数据要被改|
|`.debug`|调试段|用 `gcc -g` 才会生成|

> **冷知识（考点）**：`.bss` 名字两种说法 —— "Block Started by Symbol" 或 "Better Save Space"。它**只有 section header，不占文件空间**，纯粹是个「我以后要这么大空间」的占位声明。

---

## 五、符号（Symbols）和符号表 ⭐

### 5.1 三种链接器符号（必背）

|类型|英文|定义|
|---|---|---|
|**全局符号**|Global symbols|模块 m 定义的、可被其它模块引用的符号。即非 static 的函数和非 static 的全局变量|
|**外部符号**|External symbols|模块 m **引用**但由其它模块**定义**的全局符号|
|**局部符号**|Local symbols|仅在模块 m 内部定义和使用的符号。即用 `static` 修饰的函数和全局变量|

> **特别注意**：链接器的 local symbols **不是** C 程序里的局部变量！局部变量在栈上，链接器根本不管。

### 5.2 局部静态变量怎么处理？

```c
static int x = 15;       // 文件级
int f() { static int x = 17; return x++; }
int g() { static int x = 19; return x += 14; }
int h() { return x += 27; }
```

- 普通局部变量（non-static local）：放在**栈（stack）**上，链接器不管
- **局部静态变量（static local）**：放在 `.bss` 或 `.data`
- 编译器为每个 `static` 定义在 `.data` 分配空间，并在符号表里给**唯一名字**，比如 `x`、`x.1721`、`x.1724`

### 5.3 哪些名字会进 `symbols.o` 的符号表？（slide 16 经典题）

```c
int incr = 1;                    // 全局 → 在
static int foo(int a) {          // foo 是 local symbol → 在
  int b = a + incr;              // a, b 都是栈变量 → 不在
  return b;
}
int main(int argc, char** argv){ // main 全局 → 在；argc/argv 栈 → 不在
  printf("%d\n", foo(5));        // printf 是外部符号 → 在
  return 0;
}
```

**会进符号表的**：`incr`, `foo`, `main`, `printf` **不会进**：`a`, `b`, `argc`, `argv`, `"%d\n"`（字符串字面量进 `.rodata` 但不是 symbol）

可以用 `readelf -s symbols.o` 查看。

---

## 六、链接器如何处理重复定义 ⭐⭐⭐ 必考！

### 6.1 强符号 vs 弱符号

|类型|英文|包含|
|---|---|---|
|**强符号**|Strong|函数（procedures）、**已初始化**的全局变量|
|**弱符号**|Weak|**未初始化**的全局变量；用 `extern` 声明的|

### 6.2 链接器的三条规则（必默写！）

> **Rule 1**: 多个强符号不允许 → **链接错误**（Multiple strong symbols are not allowed）
> 
> **Rule 2**: 一个强符号 + 多个弱符号 → 选**强**的，所有弱引用都解析到强符号
> 
> **Rule 3**: 多个弱符号 → 任选一个（arbitrary）。可用 `gcc -fno-common` 强制报错

### 6.3 经典 Linker Puzzles（slide 21 必看）

|情况|结果|
|---|---|
|两个文件都 `p1() {}`|**链接错误**（两个强符号 p1）|
|两个文件都 `int x;`|OK，两个弱符号选一个；都引用同一个未初始化 int（**但这真是你想要的吗？**）|
|一边 `int x;`, `int y;`，另一边 `double x;`|**危险！** p2 写 x 可能覆盖 y！（链接器**不做类型检查**！）|
|一边 `int x=7; int y=5;`，另一边 `double x;`|**依然危险！** 强符号是 int x（4字节），但 p2 当作 double（8字节）写入会越界覆盖 y！|
|一边 `int x=7;`，另一边 `int x;`|OK，引用都指向初始化的那个|

### 6.4 隐蔽错误例子（Insidious Errors）

```c
// 文件1
int x;             // 弱符号
void f() { x = 15212; }

// 文件2
int x = 15213;     // 强符号
int main(){ f(); printf("%d\n", x); return 0; }
```

编译没错没警告，**打印 15212**！因为弱 x 被解析到强 x，f 改的就是同一个 x。

### 6.5 类型不匹配的危险（Type Mismatch）

```c
// 文件1
double x = 3.14;   // 强符号，8 字节

// 文件2
long int x;        // 弱符号，按 8 字节读
int main(){ printf("%ld\n", x); return 0; }
```

编译通过没警告。但读出来是 3.14 的 IEEE 754 二进制按整数解读 —— 一个奇怪的大数。**链接器不做类型检查**！

### 6.6 全局变量的最佳实践

- 能不用就不用全局变量
- 必须用就用 `static`（变成局部符号，避免冲突）
- 定义全局变量就**初始化**它（变成强符号，避免被默默合并）
- 引用别处的全局变量用 `extern`（明确表明这是外部引用，未定义时会**链接错误**）

---

## 七、Step 2: 重定位 Relocation ⭐⭐⭐

### 7.1 重定位做什么

1. 把所有 `.o` 文件中**相同 section 合并**（所有 `.text` 拼到一起，所有 `.data` 拼到一起）
2. 把符号从 `.o` 中的**相对位置**重新分配到可执行文件中的**绝对内存地址**
3. **更新所有对这些符号的引用**

### 7.2 重定位条目（Relocation Entries）

每个需要被修改的地方对应一个 relocation entry。结构（32-bit 例子）：

```c
typedef struct {
    int offset;          // 这条指令在 section 中的偏移
    int symbol:24,       // 指向符号表的索引（24 位）
        type:8;          // 重定位类型（8 位）
} elf32_rel;
```

### 7.3 x86-64 重定位例子（slide 28-29）

**重定位前**（`main.o`，用 `objdump -r -d main.o` 看）：

```
0000000000000000 <main>:
   0:  48 83 ec 08      sub    $0x8,%rsp
   4:  be 02 00 00 00   mov    $0x2,%esi
   9:  bf 00 00 00 00   mov    $0x0,%edi      # %edi = &array
                  a: R_X86_64_32 array        ← 重定位条目！告诉链接器：
                                                 偏移 a 处要填 array 的绝对地址
   e:  e8 00 00 00 00   callq  13 <main+0x13> # sum()
                  f: R_X86_64_PC32 sum-0x4    ← 偏移 f 处要填 PC 相对地址
  13:  48 83 c4 08      add    $0x8,%rsp
  17:  c3               retq
```

**重定位后**（最终可执行文件）：

```
00000000004004d0 <main>:
  4004d0:  48 83 ec 08         sub    $0x8,%rsp
  4004d4:  be 02 00 00 00      mov    $0x2,%esi
  4004d9:  bf 18 10 60 00      mov    $0x601018,%edi   ← array 的绝对地址填入
  4004de:  e8 05 00 00 00      callq  4004e8 <sum>     ← PC 相对偏移 0x5
  4004e3:  48 83 c4 08         add    $0x8,%rsp
  4004e7:  c3                  retq
```

### 7.4 PC 相对寻址公式（重要！考试可能要算）

`callq` 用 **PC-relative addressing** 调用 `sum`：

$$ \text{目标地址} = \text{下一条指令地址} + \text{偏移量} $$

具体到上面：

$$ 0x4004e8 = 0x4004e3 + 0x5 $$

其中：

- `0x4004e3` 是 callq 的下一条指令地址（callq 占 5 字节，从 0x4004de 开始 → 下一条 0x4004e3）
- `0x5` 是编码在 callq 里的 32 位偏移
- 0x4004e8 就是 sum 的真实起始地址 ✓

### 7.5 RISC-V 的重定位（slide 30）

RISC-V 用 **AUIPC + ADDI/JALR** 模式（因为指令是 32 位定长，装不下完整地址）：

```asm
# 加载 array 地址
4: 00000517   auipc a0, 0x0          ← 高 20 位
8: 00050513   addi  a0, a0, 0        ← 低 12 位（相对 PC）

# 加载立即数 2
c: 4589       li    a1, 2

# 调用 sum
e:  00000097  auipc ra, 0x0
12: 000080e7  jalr  ra
```

对应的重定位条目：

|offset|symbol|type|说明|
|---|---|---|---|
|4|array|`R_RISCV_PCREL_HI20`|顶部 20 位|
|8|array|`R_RISCV_PCREL_LO12_I`|低 12 位|
|e|sum|`R_RISCV_CALL_PLT`|函数调用|

> 「为什么要拆成 HI20 + LO12？」 因为 RISC-V 一条 32-bit 指令塞不下 32-bit 地址，所以拆成「高 20 位 + 低 12 位」分两条指令拼起来。

---

## 八、可执行目标文件的内存布局（必背图）

加载到内存后的虚拟地址空间布局（从下到上地址递增）：

```
┌──────────────────────────────────┐ 高地址
│  Kernel virtual memory           │ ← 用户不可见
├──────────────────────────────────┤
│  User stack (运行时创建)         │ ← %rsp 指向栈顶
│             ↓ 向下生长            │
│             …                     │
│             ↑ 向上生长            │
│  Memory-mapped region for        │ ← 共享库映射区
│  shared libraries                 │
│             …                     │
│             ↑                     │
│  Run-time heap (malloc 创建)     │ ← brk 指针
├──────────────────────────────────┤
│  Read/write data segment         │ ← .data, .bss
├──────────────────────────────────┤
│  Read-only code segment          │ ← .init, .text, .rodata
├──────────────────────────────────┤  0x400000
│  Unused                           │
└──────────────────────────────────┘ 低地址 0
```

**记忆口诀**：从下往上是 **代码（只读）→ 数据（读写）→ 堆（malloc）→ 共享库 → 栈（向下长）→ 内核**。

---

## 九、静态库 Static Libraries

### 9.1 为什么需要库？

如果不打包，要么把所有函数塞一个大 `.o`（空间浪费），要么每个函数一个 `.o` 让程序员手动链接（累死人）。

### 9.2 静态库（`.a` archive 文件）

- 把若干相关 `.o` **拼接**成一个文件，加上索引，叫 archive
- 链接器扫到未解析的引用时，会去 archive 里找
- 找到的话，把那个 `.o` 抽出来链进可执行文件
- **按需付费（Pay-as-you-go）**：只有用到的 `.o` 才被链入

### 9.3 创建静态库（命令）

```bash
unix> ar rs libc.a atoi.o printf.o ... random.o
```

`ar` 支持增量更新（option `ru`）：只重新编译变化的函数，替换 archive 里对应 `.o` 即可。

### 9.4 常用库

|库|内容|
|---|---|
|`libc.a`|C 标准库，4.6 MB，1496 个 `.o`：I/O、内存、信号、字符串、时间、随机数、整数运算|
|`libm.a`|C 数学库，2 MB，444 个 `.o`：`sin`/`cos`/`tan`/`log`/`exp`/`sqrt` 等浮点运算|

### 9.5 链接器解析外部引用的算法（必背！）⭐

> 链接器按命令行**从左到右**扫 `.o` 和 `.a` 文件
> 
> 维护一个**未解析引用列表（Wanted list / TO-DO list）**
> 
> 每遇到一个新文件 `obj`，尝试用 `obj` 中的定义解析 Wanted list 里的引用
> 
> 扫描结束后 Wanted list **不为空** → 报错

### 9.6 命令行顺序很重要！⭐⭐⭐ 经典考点

```bash
unix> gcc -L. libtest.o -lmine        # ✓ 正确
unix> gcc -L. -lmine libtest.o        # ✗ undefined reference to 'libfun'
```

**为什么第二条会出错？**

- Step 1: 链接器看到 `-lmine`。Wanted list **是空的**（还没扫到任何 `.o`）。库被忽略。
- Step 2: 看到 `libtest.o`，发现它调用了 `libfun()`，把 `libfun` 加入 Wanted list。
- Step 3: 命令结束，Wanted list 还有 `libfun`，但 `libmine` 已经过去了，回不去了！
- 结果：undefined reference 错误。

**口诀**：**库永远放命令行最右边！**（Moral: put libraries at the end）

### 9.7 为什么自定义 malloc 能覆盖 libc 的 malloc？（slide 40）

```bash
unix> gcc a.o b.o main.o /usr/libc.a
```

假设 `a.c` 和 `b.c` 调用 `malloc`，`main.c` 自己定义了 `malloc`：

|Step|文件|关于 malloc 的动作|
|---|---|---|
|1|a.o|看到 call malloc → 加入 Wanted list|
|2|b.o|看到 call malloc → 已在 list（无变化）|
|3|main.o|看到 malloc 定义 → **解析掉 1、2 的引用**|
|4|/usr/libc.a|malloc 已解析 → 不动了|

所以自定义版本胜出。

---

## 十、共享库 Shared Libraries（动态链接）⭐

### 10.1 静态库的缺点

- 每个可执行文件里**重复**包含 libc → 磁盘空间浪费
- 每个运行进程内存里**重复**包含 libc → 内存浪费
- libc 修个 bug，所有应用都得**重新链接**（比如著名的 CVE-2015-7547 glibc 漏洞）

### 10.2 共享库

- `.so` 文件（Windows 叫 DLL）
- 装入应用时**动态**链接，时机有两种：
    - **加载时链接（Load-time linking）**：可执行文件首次加载运行时链接。Linux 默认，由**动态链接器 ld-linux.so** 处理。`libc.so` 通常如此。
    - **运行时链接（Run-time linking）**：程序跑起来后再链接，通过 `dlopen()` 接口。用于发布软件、高性能 web 服务器、运行时库插桩。
- 一份共享库代码可被**多个进程共享**

### 10.3 ld-linux.so 加载流程（slide 43）

|类型|OS loader 看到时的行为|
|---|---|
|普通共享库（如 libc.so）|映射到内存，等待主程序调用|
|动态链接器 ld-linux.so 本身|看到可执行文件 ELF 头里有 **PT_INTERP** 字段指向 ld-linux.so：先加载 ld-linux.so，**不直接跑 main()**，而是跳到 ld-linux.so 的入口；ld-linux.so 修地址、加载依赖，最后跳到用户 main()|

> **PT_INTERP** = ELF Program Header 里的特殊条目，指明动态链接器的绝对路径。

### 10.4 LD_PRELOAD 和库插桩（Runtime Library Interpositioning）

- 设置环境变量 `LD_PRELOAD` → 强迫动态加载器**先**加载你的自定义库
- 你的库里如果有同名函数（比如 `malloc`），动态加载器就用你的版本
- 用途：拦截库调用做调试、监控、注入

注意：**不能拦截 `_start`**（在 `crt1.o` 里、静态链接），但可以拦截 `__libc_start_main`。

### 10.5 加载时动态链接（Load-time）和运行时动态链接（Run-time）的区别

**Load-time（slide 45）**：

```bash
# 编译共享库（注意 -fpic）
unix> gcc -shared -o libvector.so addvec.c multvec.c -fpic
```

链接时，链接器（`ld`）只生成「部分链接」的可执行文件（含重定位和符号表信息），真正解析共享库地址在 **execve 加载** 时由动态链接器完成。

**Run-time（slide 46-48）**：用 `dlopen` API 主动加载

```c
#include <dlfcn.h>
void *handle;
void (*addvec)(int*, int*, int*, int);
char *error;

// 1. 打开共享库
handle = dlopen("./libvector.so", RTLD_LAZY);
if (!handle) { fprintf(stderr, "%s\n", dlerror()); exit(1); }

// 2. 取函数指针
addvec = dlsym(handle, "addvec");
if ((error = dlerror()) != NULL) { ... }

// 3. 像普通函数一样调用
addvec(x, y, z, 2);

// 4. 卸载
if (dlclose(handle) < 0) { ... }
```

**`dlopen` / `dlsym` / `dlclose` / `dlerror` 这四个 API 必须记住！**

---

## 十一、位置无关代码 PIC（Position-Independent Code）⭐⭐⭐

### 11.1 为什么需要 PIC？

共享库的目的是「多进程共享同一份代码」。但每个进程虚拟地址不一样，怎么共享？

- **方案 A**：给每个共享库划一段固定的虚拟地址空间 → 太死板（库多了不够用）
- **方案 B**（采用）：编译成**可以装到任何地址都能正常运行**的代码 → **PIC**

### 11.2 PIC 的关键观察 / 核心 trick ⭐

> 「**不管把目标模块装到哪个地址，data segment 总是紧跟在 code segment 后面**」
> 
> 所以**任何指令到任何全局变量的距离是运行时常量（runtime constant）**！

利用这个事实，编译器在 data segment 起始处放一张表 —— **GOT (Global Offset Table)**，里面给每个被引用的全局数据对象一个条目。

### 11.3 GOT 和 PLT（必懂）

|表|全称|作用|位置|
|---|---|---|---|
|**GOT**|Global Offset Table|装外部**全局变量**的真实地址|data segment 开头|
|**PLT**|Procedure Linkage Table|装外部**函数**的跳转 stub（小段代码）|code segment（`.text` 或 `.plt`），只读|

**GOT 怎么填的**：程序首次加载时，动态链接器（ld-linux.so）查依赖、找到共享库被加载到的实际地址，**把真实地址写入 GOT**。

**PLT 是什么**：每个外部函数对应一段小的「跳板代码」（trampoline / stub）。

### 11.4 访问全局变量 my_var 的 PIC 代码（RISC-V）

```asm
# 1. 算出 my_var 在 GOT 中条目的 PC 相对地址
auipc a0, %got_pcrel_hi(my_var)

# 2. 从 GOT 加载 my_var 的真实地址到 a0
ld a0, %pcrel_lo(...)(a0)

# 3. 加载 my_var 的实际值（比如 32 位字）
lw a0, 0(a0)
```

**记忆**：访问外部变量要 **三跳**：`auipc` 算 GOT 条目地址 → `ld` 取出真实地址 → `lw` 取实际值。

### 11.5 调用外部函数 printf 的 PIC 代码（必考流程）⭐⭐⭐

```c
main() { printf(...); }
```

编译/汇编/链接的逐步过程：

**Step 1：编译器**生成伪指令 `call printf`

**Step 2：汇编器**把伪指令展开成两条 RISC-V 指令：

```asm
Label0: auipc ra, %pcrel_hi(printf@plt)   # 加载 PLT 偏移高 20 位
        jalr  ra, %pcrel_lo(Label0)(ra)   # 跳到 PLT stub
```

并留下重定位条目：「请把这两条指令改成指向 printf 的 PLT stub」。

**Step 3：链接器**发现 printf 是外部动态函数，自动生成 **PLT stub**（三条指令），创建 `.plt` section：

```asm
auipc t3, %hi(printf@got)   # 算出 GOT 中 printf 条目的地址
ld    t3, %lo(...)(t3)      # 从 GOT[printf] 取出真实地址到 t3
jalr  t0, t3                # 跳到 printf 真实地址
```

**Step 4：链接器**创建 `.got.plt` section，给 printf 留 4B（32位）/ 8B（64位）空间，**真实地址等动态链接器填**。

**Step 5：动态链接器**（运行时）填入 printf 的真实内存地址。

### 11.6 总结表（必背）

|阶段|谁干的|干了什么|
|---|---|---|
|Step 1|编译器（compile time）|生成伪指令 `call printf`|
|Step 2|汇编器（compile time）|展开为 AUIPC/JALR；插入重定位条目|
|Step 3|链接器（link time）|创建 `.plt` section 和 stub（AUIPC/LD/JALR 三条）|
|Step 4|链接器（link time）|在 `.got.plt` 中给 printf 留地址槽（32 位 4B / 64 位 8B）|
|Step 5|动态链接器（run time）|把 printf 真实地址写入 GOT|

### 11.7 GOT/PLT 内存布局图

```
code segment (read-only)        data segment (read-write)
┌───────────┐                   ┌───────────┐
│  .text    │                   │  .GOT     │ ← 全局变量地址
├───────────┤                   ├───────────┤
│  .plt     │ ──┐               │  .GOT.plt │ ← 函数地址（动态链接器填）
└───────────┘   └─reads──────→  ├───────────┤
                                 │  .data    │
                                 ├───────────┤
                                 │  .bss     │
                                 └───────────┘
```

`.plt` 中的 stub **读** GOT 中的地址来跳转。GOT 由**动态链接器在加载时更新**。

---

## 十二、考前自查清单（这些必须能默写）

- [ ] 链接器干哪两件事？→ Symbol Resolution + Relocation
- [ ] 三种目标文件？→ `.o` / `a.out` / `.so`（DLL）
- [ ] ELF 文件结构里至少 8 个 section（`.text`/`.rodata`/`.data`/`.bss`/`.symtab`/`.rel.text`/`.rel.data`/`.debug`）
- [ ] 三种链接器符号？→ Global / External / Local
- [ ] 强符号 vs 弱符号定义
- [ ] 链接器三条规则
- [ ] PC 相对寻址公式：`目标地址 = 下一条指令地址 + 偏移`
- [ ] 重定位条目结构（offset / symbol / type）
- [ ] 内存布局（`.text` 在低地址 → 栈在高地址向下生长）
- [ ] 静态库扫描算法（Wanted list，命令行从左到右）
- [ ] 库在命令行最右边的原因
- [ ] 静态库 vs 共享库的优劣
- [ ] dlopen / dlsym / dlclose / dlerror 四个 API
- [ ] PIC 核心 trick（code 和 data 距离运行时常数）
- [ ] GOT 和 PLT 各自作用
- [ ] 调用外部函数 5 个 step（编译器 / 汇编器 / 链接器创建.plt / 链接器创建.got.plt / 动态链接器填地址）
- [ ] LD_PRELOAD 和 PT_INTERP 是什么

---

**关键命令速查**

```bash
gcc -Og -o prog main.c sum.c           # 完整编译链接
gcc -c file.c                          # 只编译到 .o
gcc -shared -o libfoo.so a.c -fpic     # 编译共享库
gcc -fno-common                        # 强制多重定义报错
ar rs libc.a a.o b.o                   # 创建静态库
ar -t libc.a                           # 列出库内容
ar -ru libc.a a.o                      # 增量更新
readelf -s file.o                      # 查看符号表
objdump -r -d file.o                   # 看反汇编 + 重定位条目
objdump -d prog                        # 看可执行文件反汇编
```

祝考试顺利！