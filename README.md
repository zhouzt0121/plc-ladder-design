# PLC 梯形图设计方法论（顺序控制流程图 + IO 表 → S7 LAD）

> 本文档面向**任何 AI 智能体**（Claude、GPT、本地 agent 框架等），不依赖特定平台、会话记忆或插件机制。
> 读完这一份即可独立执行：把一张顺序控制流程图和一份 IO 表，变成能编译、能跑、注释齐全的西门子 S7-1200/1500 梯形图程序。
>
> **流程图不限于 IFAE**：IFAE、GPP、SFC、手绘草图、Excel/Word 表格、纯文字步骤描述均可——只要它能表达「每一步做什么、什么条件进下一步」。本方法论处理的是**步序语义**，与图的画法无关。
>
> 配套细节在 `references/` 目录，本文是总纲与索引。

---

## 0. 一句话方法

**不照抄任何现成程序。用「两级拆解」把流程步变成机械语言，再把机械语言变成 LAD 元件，按固定块结构拼装，写进 TIA 后编译 + 仿真双重验证。**

---

## 1. 目录结构

```
plc-ladder-design/
├── README.md                      # 本文档（中文总纲，主文档）
├── README.en.md                   # 英文版总纲
├── SKILL.md                       # 平台入口（WorkBuddy 触发用；其他 AI 可忽略 frontmatter，整个目录可直接拷贝使用）
└── references/
    ├── ladder-grammar.md          # LAD/s7dcl 语法：RUNG/wire/触点/线圈、命名规范
    ├── step-templates.md          # 步序模板：模板 A/C、step/step0/step_branch、脉冲闸门版
    ├── tia-openness.md            # TIA 写块：s7dcl/XML bundle、中文标题注入、干检、仿真
    └── toolchain.md               # 工具链与 MCP：真实路径、传输层、设计期/运行期分界
```

---

## 2. 核心：两级拆解法

每个流程步先拆成**机械语言**，再拆成**程序语言**，中间不跳级。

### 第一级：步骤动作 → 机械语言

把「第 N 步」翻译成 4 个字段，只描述机械上发生什么，不碰程序符号：

| 字段 | 问法 | 例子（步1：转盘取料） |
|---|---|---|
| 入口动作 | 进步瞬间，哪个执行器做什么 | 转盘定位电机转到 60° |
| 转移条件 | 等到哪个物理反馈才允许进步 | 转盘到位 AND 取料点检测到料 |
| 动作对象 | 涉及哪几类执行器 | 定位电机 / 气缸 / 气爪 |
| 下一状态 | 进步后停在哪 | 步2 |

机械语言只允许这些词：**定位/移动、伸出/缩回、夹紧/松开、抬起/落下、有料/无料、到位/未到**。它和流程图一一对应，人能逐字核对。

### 第二级：机械语言 → 程序语言

| 机械语言 | 程序语言 | 挂在哪 |
|---|---|---|
| 入口动作（S 类，如「伸出」） | `S_Coil( 输出标签 )` | 运行闸门之后 |
| 入口动作（R 类，如「缩回/停」） | `R_Coil( 输出标签 )` | 运行闸门之后 |
| 转移条件（到位/有料） | `Contact( 输入标签 )` | 转移判定支路 |
| 转移条件（取反，如「没到」） | `I_Contact( 输入标签 )` | 转移判定支路 |
| 转移条件（延时 T） | `TON( pt := T#Ns )` 串联闸门 | 转移判定支路末端 |
| 清当前步 + 置下一步 | `R_Coil(STEP[n])` + `S_Coil(STEP[n+1])` | 转移判定之后 |

**最易死锁的一点**：流程图的「入口动作」通常是**步指令**（进本步立刻执行），不是「条件成立后的后果」。动作必须挂在闸门（`STEP[n] AND (自动运行中 OR 启动)`）之后，**转移条件只能挂在动作之后的下游支路**。把动作串在转移条件后面（模板 A）会造成「等气缸夹紧到位才发夹紧指令」的自锁。

判断规则：
- 动作由传感器触发、到位即执行 → **模板 A**：`STEP[n] → 条件 → 动作`
- 动作是步指令、进步即发出 → **模板 C**：`STEP[n] → 闸门 → 动作`；`闸门 → 条件 → 清步置步`

顺序控制流程图默认用**模板 C**（除非该步动作确实是「到位后才执行」的后果）。

---

## 3. LAD 的前后关系与上下关系

读/写 LAD 前必须内化的三条：

1. **前后 = 一条梯级内的左右 = 串联 = AND。** 电流从左电源轨（`powerrail`）向右流，途中每个触点是「与」条件，末端线圈是「结果」。`Contact(A) Contact(B) Coil(C)` 即 `C = A AND B`。
2. **上下 = 从某节点向下分出的新梯级 = 并联 = OR。** 一个节点（`wire#wN`）可被多条向下的 `RUNG wire#wN` 复用，各自通向不同线圈，等效于该节点对多支求「或」。
3. **节点是前后关系的锚点。** 每个 `wire#wN` 的逻辑值 = 该点左侧所有触点的 AND 结果。整张图靠节点编号把「横向串联」和「纵向并联」缝在一起。

节点硬约束：**每个声明的节点都要被后续梯级引用；两个相邻节点之间不能空着**（否则产生空段，写入直接失败）。

---

## 4. 标准块结构（每个块的作用）

固定 4 块 + 2 DB，职责单一、**调用顺序不可换**（后者依赖前者建立的状态）：

| 块 | 类型 | 作用 | 依赖 |
|---|---|---|---|
| `Main` | OB1 | 只顺序调用下列 FC，**不写任何逻辑** | — |
| `系统的开车和停车` | FC | 模式切换、启停、急停、清全部步序 | 全局 DB |
| `系统回到初始状态` | FC | 初始位判定、复位流程、复位指示 | 前一个 FC 建的状态 |
| `系统开始自动运行` | FC | **全流程步序**：STEP 数组 + 转移 + 动作 | 前两个 FC 建的 `系统自动运行中` |
| 工艺 FC（如 `高度检测`） | FC | 本站工艺专用（模拟量标定、合格判定） | 自动运行 FC |
| `系统层全局DB块` | GlobalDB | 系统标志 + `STEP[0..15]` 数组 | 被所有 FC 共享 |
| `定时器DB` | GlobalDB | `TON_TIME` 成员（TON 实例） | 被自动运行 FC 用 |

关键约定：
- `STEP[0..15]` 是 Bool 数组，每步一位，S/R 成对操作，不用整数步号。
- `STEP[0]` 为初始步，进入自动运行后置位且**循环期间不清除**（循环入口常开闸门），只有停止/急停/复位才清。
- 清步用 `R_BitfieldCoil(operand => "DB".STEP[0], n := 16)` 一条，不写 16 个 `R_Coil`。

---

## 5. 工具链与 MCP

本方法论是**纯文档型**，不含可执行脚本。执行依赖一套「Python 工具链 + TIA MCP」的组合。下面给的是**本机实测配置**（Windows + TIA Portal V21）；换机器/换智能体只需替换路径，方法论与实现细节不变。

### 设计期 vs 运行期（先分清楚）

| 阶段 | 干什么 | 用什么 |
|---|---|---|
| 设计期 | 生成 s7dcl / 写块 / 编译 / 保存 / 仿真 | 本地 Python 工具链 + `tia-mcp.exe` |
| 运行期联调 | 下发实跑、读写 I/O、OPC UA 监测 | `plc-ai-agent`（OPC UA 仿真器） |

本方法论只覆盖**设计期**，不要与运行期混在一个流程里。

### MCP 注册（本机示例：`~/.workbuddy-ai/mcp.json`）

以下 JSON 是 WorkBuddy 的注册格式，**其他 AI 智能体按其 MCP 客户端格式照抄 command/args/env 即可**；不支持 MCP 的，直接调用 `tia-mcp.exe` 的 stdio JSON-RPC（见下节）。

```json
{
  "mcpServers": {
    "windows-mcp": { "command": "C:\\awesome-ai-reverse\\Windows-MCP\\.venv\\Scripts\\windows-mcp.exe", "args": ["serve"] },
    "tia-portal":  { "command": "C:\\Users\\Holmes\\.dotnet\\tools\\tia-mcp.exe",
                     "args": ["--access-mode", "read-write"],
                     "env": { "TiaPortalV21Dir": "D:\\Siemens\\Automation\\Portal V21\\PublicAPI\\V21\\net48" } },
    "plc-ai-agent":{ "command": "C:\\Work\\plc-tools\\plc-ai-agent\\.venv\\Scripts\\python.exe",
                     "args": ["-m", "plc_ai.mcp.server"],
                     "env": { "PYTHONPATH": "C:\\Work\\plc-tools\\plc-ai-agent\\src",
                              "PLC_USE_SIMULATOR": "true", "PLC_ENDPOINT": "opc.tcp://127.0.0.1:4841" } }
  }
}
```

- **`tia-portal`**：设计期主力，.NET 封装 TIA V21 Openness，指向 `Siemens.Engineering` DLL。
- **`plc-ai-agent`**：运行期 OPC UA 仿真器，本方法论不依赖。
- **`windows-mcp`**：通用 Windows 能力，与本方法论无关。

**非 WorkBuddy 智能体的接法**（任选其一）：

1. 支持 MCP 的客户端：按上面 `tia-portal` 的 command/args/env 注册，工具名直接用 `bind_project` / `plc_read` / `plc_write` / `compile_check` / `save_project`。
2. 不支持 MCP：用任何语言起子进程 `tia-mcp.exe --access-mode read-write`，按 JSON-RPC 2.0 over stdio 发 `initialize` → `notifications/initialized` → `tools/call`（参考 `tia_session.py` 的 30 行实现）。
3. 完全不用 MCP：直接用 .NET 调 Siemens.Engineering Openness API（`D:\Siemens\Automation\Portal V21\PublicAPI\V21\net48`），把本文的 s7dcl 文本用 `ImportDocuments` 导入即可。

### 两种传输层（关键，容易用错）

`tia-mcp.exe` 既能被任意 MCP 客户端注册为标准 MCP，也能被本地直驱：

1. **本地直驱（工具链实际用法）**：`subprocess` 直接拉起 `tia-mcp.exe`，自带 JSON-RPC 2.0 over stdio 握手（`initialize` → `notifications/initialized` → `tools/call`）。可在同一 Python 进程里连续调一串 TIA 工具。
2. **平台 MCP 连接器**：同一 exe 注册为 `tia-portal`，由所在平台的 MCP 客户端托管。

**注册了 ≠ Python 工具链自动用上**，两条路通同一个 exe，只是宿主不同。

### 本地 Python 工具链（`C:\Work\plc-tools\`）

| 文件 | 职责 |
|---|---|
| `tia_session.py` | MCP 会话层：拉 `tia-mcp.exe` + JSON-RPC over stdio |
| `ladlib.py` | `Tia` 类：bind/open/read/write_block/create_block/compile/save/tree + s7dcl 元件 |
| `ladfc.py` | 构件 API + 步序模板 + 写块流水线 |
| `mlc.py` | 中文网络标题 XML 注入 |
| `sw.py` | 幂等切项目 |
| `gen1/3/5.py` | 各站生成器（数据驱动） |
| `sim2.py` / `test2_sim.py` | 梯级解释器 + 六类场景仿真断言 |

运行时（本机示例）：`C:\Users\Holmes\.workbuddy-ai\binaries\python\envs\default\Scripts\python.exe`（Python 3.13）。其他环境换成自己的 Python 3.10+ 即可，脚本只依赖标准库 + `tia-mcp.exe`。

---

## 6. 快速开始（任何 AI 都能跑）

### 6.1 链路自检

```python
import sys; sys.path.insert(0, r"C:\Work\plc-tools")
from ladlib import Tia
t = Tia()
pl = t.portals()                       # 应列出 TIA Portal 实例
if pl:
    t.s.call("bind_project", {"projectPath": r"E:\PLC\#2\#2.ap21",
                              "portalProcessId": pl[0]["processId"]}, 240)
    c = t.read("PLC_1/开车与停车")     # 读得通 = 离线可写
t.close()
```

### 6.2 生成流程

1. **抽数据**：流程图 + IO 表 → 步序数据（步号 / 入口动作 / 转移条件 / 下一步 / 分支列表）。
2. **两级拆解**：每步写机械语言 4 字段，再落到程序语言元件。
3. **写 LAD**：用 `step / step0 / step_branch` 模板拼步网络；用元件写启停/复位网络。
4. **写进 TIA**：s7dcl 写块 → 读回 XML 注入中文标题 → 编译 0 错 0 警 → 保存。
5. **仿真验证**：梯级解释器执行真实 s7dcl，跑六类场景断言。

---

## 7. 硬约束清单（逐条遵守）

- **`I_Contact` 是常闭触点（NC），不是「非」/上升沿。** 停止/急停按钮物理常闭，配 `I_Contact` 实现「断线即停」。上升沿用 `P_Contact`。
- **`P_Contact` 正跳变触点每周期刷新边沿存储位**：严禁多网络共用同一 `bit`。正确做法是只做一次边沿检测，产生单周期脉冲位，全部步序网络读该脉冲位。
- **wire 节点必须「有源有引用」**：悬空节点或相邻空段会让写入直接失败。
- **TON 实例放 GlobalDB 的 `TON_TIME` 成员**（`create_block` 不支持 InstanceDB）；非法 TON 语法会让 TIA Portal 崩溃，改语法前先小步验证。
- **中文网络标题/注释无法用 s7dcl 文本写**（`S7_NetworkTitle`/`TITLE=`/`COMMENT=` 全被拒）。只能走「s7dcl 建结构 → 读回 FlgNet XML → 注入 MultilingualText `<Text>` → XML 写回」。
- **双线圈气缸（伸出/缩回两个线圈）复位必须主动 S 回位线圈**，不能两个都 R。
- **改完功能要同步回退注释**，否则注释描述的功能在代码里找不到 = 验收硬伤。
- **写块前干检 + 写完读回复核**：读回确认补丁没打过、状态干净；写完断言新支路存在、旧支路消失、MLC 引用数不变，再编译保存。
- **编译过 ≠ 逻辑对**：必须仿真验证，覆盖 复位 / 手动 / 自动循环 / 停止急停 / 故障 / 灯语 六类场景。

---

## 8. 参考文件索引

| 文件 | 内容 |
|---|---|
| `references/ladder-grammar.md` | RUNG/wire/触点/线圈精确写法、节点命名、IO 标签命名规范 |
| `references/step-templates.md` | 模板 A/C 判定、`step/step0/step_branch`、脉冲闸门版、选择序列、末步回环、双线圈复位 |
| `references/tia-openness.md` | s7dcl/XML bundle、中文标题注入、编译保存、写前干检、梯级仿真验证法 |
| `references/toolchain.md` | 工具链与 MCP 搭配、真实路径、两种传输层、一键自检 |
