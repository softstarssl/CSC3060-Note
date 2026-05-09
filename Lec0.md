# CSC3060 - Lecture 0: Course Organization (傻瓜入门版)

**课程:** CSC3060 Computer Systems | **讲师:** Prof. Wei Chung Hsu (徐慰中)
**核心宗旨:** 这节课主要讲咱们学什么、怎么考，以及为什么要学一个新的东西叫 **RISC-V**。

---

## 1. 这门课到底是干嘛的？(CSC3060 vs CSC3050) - **⚠️ 必考区别**

你可能觉得这门课跟隔壁 CSC3050 很像，教授特意强调了区别。

| 课程 | CSC3060 (这门课) | CSC3050 (隔壁那门) |
| :--- | :--- | :--- |
| **视角** | **程序员视角 (Programmer-centric)** | **设计者视角 (Design-centric)** |
| **核心问题** | “我的 C 代码是怎么在机器上跑起来的？” | “处理器是怎么造出来的？” |
| **重点** | 调试 (Debugging)、位操作、内存行为、写底层代码 | 硬件设计、流水线、电路逻辑 |
| **比喻** | **教你怎么把赛车(电脑)开到极限** | **教你怎么造一辆赛车** |

**教材 (Textbook):**
*   **主教材:** *CS:APP* (Computer Systems: A Programmer's Perspective). 这本书是计算机系统的“圣经”，很多名校（CMU, MIT, 北大清华）都在用。
*   **缺点:** 这本书主要讲的是 **x86** 架构（Intel 那套），有点老了。

---

## 2. 为什么要学 RISC-V？(The Rise of RISC-V) - **⚠️ 核心概念**

虽然教材讲 x86，但教授说 x86 是“老年人”，ARM 是“中年人”，**RISC-V** 才是“青少年”，前途无量。

### 2.1 什么是 RISC?
*   **全称:** Reduced Instruction Set Computer (精简指令集计算机)。
*   **对手:** CISC (Complex Instruction Set Computer)，比如 Intel 的 x86。
*   **现状:** 手机（ARM）、超级计算机大多都是 RISC 的天下。

### 2.2 RISC-V 的两大特点 (Open & Free)
**注意！这里有坑，考试容易混淆！**

1.  **Open (开放):**
    *   **意思是:** 它的标准（说明书）是公开的，谁都可以拿去设计芯片。
    *   **误区:** **Open Standard $\neq$ Open Source Code**。
    *   *人话:* 菜谱（标准）是公开的，但厨师做出来的菜（芯片设计图 RTL/HDL）可以是保密的，不一定要公开给别人看。

2.  **Free (免费):**
    *   **意思是:** 用这个标准设计芯片，**不需要交“保护费” (Royalty-free)**。
    *   **误区:** **Free License $\neq$ Free Chips**。
    *   *人话:* 你用这个技术不用给 RISC-V 基金会交钱，但你买物理芯片肯定是要花钱的！

### 2.3 三大架构的“过路费”对比 (License Fees)

| 架构 (ISA) | 厂商 | 费用模式 (Cost Model) |
| :--- | :--- | :--- |
| **x86** | Intel / AMD | **私有 (Proprietary)**。以前 AMD 要给 Intel 交钱，现在互免了，但别人想用？没门。 |
| **ARM** | ARM 公司 | **收租模式**。1. 入场费 (License): \$1M - \$10M; 2. 提成 (Royalty): 卖一个芯片抽成 $1\% - 2\%$。 |
| **RISC-V** | 基金会维护 | **免费 (Royalty-free)**。随便用，不收提成。 |

---

## 3. 软件定义的未来 (Software First) - **⚠️ 趋势分析**

以前是先造硬件，再写软件。现在反过来了！

*   **趋势:** **Software-defined Everything** (软件定义一切)。
*   **逻辑:** 现在的杀手级应用（比如 AI、ChatGPT、LLM）太火了，硬件是为了伺候这些软件而专门设计的。
*   **结论:** 谁的软件生态好，谁的硬件就能赢。RISC-V 正在疯狂补这一课。

---

## 4. 考试与评分 (Assessment) - **⚠️ 关乎生死**

这门课怎么算分？看这里：

*   **平时分 (5%):** 课堂签到、随堂小测验（Pop Quizzes）。
*   **笔试小测 (25%):** 会提前两周通知。
*   **大作业 (Projects - 45%):** 重头戏！一共 5 个 Project。
    1.  Data Lab (8%): 玩转二进制位操作。
    2.  Assembly Debugging (8%): **拆弹实验 (Bomb Lab)**，非常经典，拆不掉会扣分。
    3.  Pipelining (8%): 流水线调度。
    4.  Cache Simulator (10%): 写个模拟器模拟缓存。
    5.  Optimization (11%): 代码优化比赛，看谁跑得快。
*   **期末考 (25%):** **开卷考试 (Open Book)**！可以带书和笔记，但不能用电子设备。

---

## 5. 杂项规定 (Misc Rules)

1.  **AI 工具 (ChatGPT/DeepSeek):**
    *   **允许使用！** 做作业可以用 AI 帮忙。
    *   **条件:** 你必须把**你和 AI 的聊天记录**贴在作业里提交。你得知道 AI 写了啥，不能无脑复制。
2.  **补考 (Makeup Exam):**
    *   如果生病申请补考，是在教授办公室进行**口试 (Oral Exam)**！
    *   而且分数会**打九折 (10% discount)**。所以尽量别补考，口试很恐怖的。
3.  **学术诚信:** 抄袭直接挂科 (Zero Tolerance)。

---

## 📝 必背小抄 (Summary for Exam)

1.  **CSC3060 Focus:** Programmer-centric, System Software (OS/Compiler), How code executes.
2.  **RISC-V "Open":** Open Standard (ISA), NOT necessarily Open Source implementation.
3.  **RISC-V "Free":** Royalty-free (no license fee), NOT free physical chips.
4.  **Cost Model:**
    *   ARM: License fee + Royalty ($\approx 1-2\%$).
    *   RISC-V: $\$0$ Royalty.
5.  **Driving Force:** Software defines Architecture (Software First).