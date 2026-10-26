---
name: plc-ladder-design
description: Designing and generating Siemens S7-1200/1500 ladder logic (LAD / s7dcl) from any sequential-control step flowchart (IFAE, GPP, SFC, hand-drawn, tabular, or text-described) plus an IO table. This skill should be used when the user wants to turn a sequential-control flowchart and an IO point table into a working PLC ladder program, or to add/modify a step-sequence network, an auto-run FC, or a global STEP-DB on TIA Portal. Covers the two-level decomposition method (step action → mechanical language → program language), LAD rung/wire semantics, the standard block architecture, naming conventions, and the TIA Openness write path. Platform-agnostic: any AI agent with file access and MCP/subprocess capability can follow it.
agent_created: true
---

# PLC 梯形图设计（顺序控制流程图 → S7 LAD）

把「顺序控制系统流程图 + IO 表」变成能编译、能跑、注释齐全、可进 TIA 的梯形图。核心是一套**两级拆解法**加**标准块结构**，避免照抄某一站的旧程序——范式只是结果，方法才是可复用的部分。

流程图不限于 IFAE：IFAE / GPP / SFC / 手绘草图 / Excel 表格 / 纯文字步骤描述都适用——只要它能表达「每步做什么、什么条件进下一步」。本 skill 处理的是**步序语义**，与图的画法无关。

## 何时用

- 用户给一张顺序控制流程图（IFAE/GPP/SFC/手绘/表格/文字描述均可）+ 一份 IO 表，要「写出 PLC 梯形图 / 写成 S7 程序 / 在 TIA 里建块」。
- 要**新增或修改**某一步序网络、某个 FC、某个 STEP 转移条件。
- 要把手写的「机械动作描述」落成 LAD 文本（s7dcl）或写进 TIA Portal 工程。

## 方法总览

1. 读流程图 + IO 表，抽成**步序数据**（见「步序数据」节）。
2. 对每一步做**两级拆解**（见「两级拆解法」节）：先落到机械语言，再落到程序语言。
3. 按**标准块结构**（见「块的作用」节）分块，把程序语言写成 LAD 梯级（见 `references/ladder-grammar.md`）。
4. 用步序模板批量生成步网络（见 `references/step-templates.md`）。
5. 写进 TIA / 编译 / 仿真验证（见 `references/tia-openness.md`）。

## 两级拆解法（核心）

每个流程步先**拆成机械语言，再拆成程序语言**，中间不跳级。

### 第一级：步骤动作 → 机械语言

把「第 N 步」翻译成 4 个字段，每个字段只描述**机械上发生什么**，不碰任何程序符号：

| 字段 | 机械语言问法 | 例子（步1：转盘取料） |
|---|---|---|
| 入口动作 | 进步瞬间，哪个执行器做什么 | 「转盘定位电机转到 60°」 |
| 转移条件 | 等到哪个物理反馈才允许进步 | 「转盘到位 AND 取料点检测到料」 |
| 动作对象 | 涉及哪几类执行器 | 定位电机 / 气缸 / 气爪 |
| 下一状态 | 进步后停在哪 | 步2 |

机械语言只允许这些词：**定位/移动、伸出/缩回、夹紧/松开、抬起/落下、有料/无料、到位/未到**。它和流程图一一对应，是人能逐字核对的东西。

### 第二级：机械语言 → 程序语言

把 4 个字段映射到 LAD 元件，**动作与转移解耦**：

| 机械语言 | 程序语言 | 挂在哪 |
|---|---|---|
| 入口动作（S 类，如「伸出」） | `S_Coil( 输出标签 )` | 挂**运行闸门**之后 |
| 入口动作（R 类，如「缩回/停」） | `R_Coil( 输出标签 )` | 挂**运行闸门**之后 |
| 转移条件（到位/有料） | `Contact( 输入标签 )` | 串在**转移判定**支路 |
| 转移条件（取反，如「没到」） | `I_Contact( 输入标签 )` | 串在**转移判定**支路 |
| 转移条件（延时 T） | `TON( pt := T#Ns )` 串联闸门 | 串在**转移判定**支路末端 |
| 清当前步 + 置下一步 | `R_Coil(STEP[n])` + `S_Coil(STEP[n+1])` | 挂在转移判定之后 |

**最易死锁的一点**：流程图的「入口动作」通常是**步指令**（进本步立刻执行），不是「条件成立后的后果」。所以动作必须挂在闸门（`STEP[n] AND (自动运行中 OR 启动)`）之后，**转移条件只能挂在动作之后的下游支路**。把动作串在转移条件后面（模板 A）会造成「等气缸夹紧到位才发夹紧指令」这种自锁。判断依据见 `references/step-templates.md` 的模板 A/C 对照。

## 前后关系、上下关系（LAD 的物理语义）

这是读/写 LAD 之前必须内化的三条：

1. **前后 = 一条梯级内的左右 = 串联 = AND。** 电流从左电源轨（`powerrail`）向右流，途中每个触点是「与」条件，末端线圈是「结果」。`Contact(A) Contact(B) Coil(C)` 即 `C = A AND B`。
2. **上下 = 从某节点向下分出的新梯级 = 并联 = OR。** 一个节点（`wire#wN`）可以被多条向下的 `RUNG wire#wN` 复用，它们各自再通向不同线圈，等效于该节点对多个支路求「或」。
3. **节点是「前后关系」的锚点。** 每个 `wire#wN` 代表的逻辑值 = 该点左侧所有触点的 AND 结果。整张图靠节点编号把「横向串联」和「纵向并联」缝在一起。节点必须是**有源无悬空**：每个声明的节点都要被后续梯级引用，两个相邻节点之间不能空着（否则产生空段，写入即失败）。

详细的 RUNG / wire / 触点 / 线圈语法、节点命名、并联汇入汇出写法，见 `references/ladder-grammar.md`。

## 每个块的作用（标准结构）

顺序控制系统用固定 4 块 + 2 DB，职责单一、顺序固定，**调用顺序不能换**（后者依赖前者建立的状态）：

| 块 | 类型 | 作用 | 依赖 |
|---|---|---|---|
| `Main` | OB1 | 只顺序调用下列 4 个 FC，**不写任何逻辑** | — |
| `系统的开车和停车` | FC | 模式切换（自动/手动）、启停、急停、清全部步序 | 全局 DB |
| `系统回到初始状态` | FC | 初始位判定、复位流程、复位指示 | 前一个 FC 建的状态 |
| `系统开始自动运行` | FC | **全流程步序**：STEP 数组 + 转移 + 动作 | 前两个 FC 建的 `系统自动运行中` |
| 工艺 FC（如 `高度检测`） | FC | 本站工艺专用（模拟量标定、合格判定） | 自动运行 FC |
| `系统层全局DB块` | GlobalDB | 系统标志 + `STEP[0..15]` 数组 | 被所有 FC 共享 |
| `定时器DB` | GlobalDB | `TON_TIME` 成员（TON 实例） | 被自动运行 FC 用 |

关键约定（各站同构，直接套用）：
- **`STEP[0..15]` 是 Bool 数组**，每步一位，S/R 成对操作，不用整数步号。
- **`STEP[0]` 为初始步，进入自动运行后置位且循环期间不清除**（作为循环入口常开闸门），只有停止/急停/复位才清。
- **清步用 `R_BitfieldCoil(operand => "DB".STEP[0], n := 16)` 一条**，不写 16 个 `R_Coil`。
- 命名规范见 `references/ladder-grammar.md` 的「IO 标签命名」节。

## 工作流

1. **抽数据**：把流程图 + IO 表整理成「步序数据」（步号 / 入口动作 / 转移条件 / 下一步 / 分支列表）。IO 表提供「地址 → 标签名」，标签名以工程里已有的真实 tag 为准，不另造。
2. **两级拆解**：每步先写机械语言 4 字段，再落到程序语言元件（见上）。选择序列（合格/不合格）用互斥双支 `I_Contact(标志)` / `Contact(标志)` 汇到同一 `STEP[n+1]`。
3. **写 LAD**：用 `references/step-templates.md` 的 `step / step0 / step_branch` 模板拼步网络；用 `references/ladder-grammar.md` 的元件写启停/复位网络。
4. **写进 TIA**：`references/tia-openness.md` 给出 Openness 写块路径、中文网络标题的 XML 注入法、编译与保存、以及**写前先干检 + 写完读回复核**的流程。
5. **仿真验证**：编译 0 错 0 警只是第一关；用梯级解释器直接执行真实 s7dcl 文本，按 复位 / 手动 / 自动循环 / 停止急停 / 故障 / 灯语 六类场景跑断言（方法见 `references/tia-openness.md` 末节）。

## 依赖与工具（换机先读这一节）

本 skill 是**纯文档型**（无脚本），执行靠「Python 工具链 + TIA MCP」的组合。下面给的是**本机实测配置**（Windows + TIA Portal V21）；换机器/换智能体只需替换路径，方法不变。细节、真实路径、两种传输层、MCP 注册表关系见 `references/toolchain.md`。要点：

- 设计期（本 skill）：`tia-mcp.exe` + `ladlib/ladfc/mlc/sw` 工具链；运行期联调（OPC UA 仿真器 `plc-ai-agent`）**不属于本 skill**，别混。
- `ladlib` 走 **subprocess 直驱** `tia-mcp.exe`（JSON-RPC over stdio），不是平台 MCP 连接器通道——同一 exe 两条路，别以为注册了 Python 就自动用上。
- 跑任何生成器用本机 Python 3.10+（本机示例：`C:\Users\Holmes\.workbuddy-ai\binaries\python\envs\default\Scripts\python.exe`）。

## 常见坑（硬约束，逐条遵守）

- **`I_Contact` 是常闭触点（NC），不是「非」/上升沿。** 停止/急停按钮物理常闭，配 `I_Contact` 实现「断线即停」。上升沿用 `P_Contact`。
- **`P_Contact` 正跳变触点每个扫描周期都会刷新其边沿存储位**：严禁多个网络共用同一个 `bit`，否则第一个网络消费掉上升沿、其余网络永不动作。正确做法见 `references/step-templates.md` 的「脉冲闸门版」。
- **wire 节点必须「有源有引用」**：悬空节点或相邻空段会让写入直接失败。
- **TON 实例放 GlobalDB 的 `TON_TIME` 成员**（`create_block` 不支持 InstanceDB），写法 `"定时器DB".T1.TON(pt := T#2s, et => )`；**非法 TON 语法会让 TIA Portal 崩溃**，改语法前先小步验证。
- **中文网络标题/注释无法用 s7dcl 文本写**（`S7_NetworkTitle`/`TITLE=`/`COMMENT=` 全被拒）。只能走「s7dcl 建结构 → 读回 FlgNet XML → 注入 MultilingualText `<Text>` → XML 写回」（`references/tia-openness.md`）。
- **双线圈气缸（伸出/缩回两个线圈）的复位必须主动 S 回位线圈**，不能两个都 R（会同时失电，机械上不安全）。
- **改完功能要同步回退注释**，否则注释描述的功能在代码里找不到 = 验收硬伤。
- **写块前干检**：读回目标块确认补丁没打过、状态干净；写完读回复核 + 编译 + 保存。详见 `references/tia-openness.md`。

## 资源

- `references/ladder-grammar.md` — LAD/s7dcl 语法：RUNG/wire/触点/线圈、节点前后上下语义、IO 标签命名规范。
- `references/step-templates.md` — 步序模板：模板 A vs C 判定、`step/step0/step_branch`、脉冲闸门版 `step_p`、选择序列、末步回环、双线圈复位。
- `references/tia-openness.md` — TIA Openness 写块：MCP 工具、s7dcl/XML bundle、中文标题注入、编译保存、写前干检、梯级仿真验证法。
- `references/toolchain.md` — 工具链与 MCP 搭配：设计期/运行期分界、`mcp.json` 三个 MCP、`tia-mcp.exe` 两种传输层、本地 Python 工具链文件职责、一键自检。
