# 工具链与 MCP 搭配（真实路径与职责）

本 skill 是**纯文档型**：不含可执行脚本，references 按需读入即可。真正执行「流程图→梯形图→写进 TIA」的是 `C:\Work\plc-tools\` 下的本地 Python 工具链 + 一个 .NET 写的 TIA MCP。下面把依赖关系写死，避免重复踩「连不上 / 用错传输层」的坑。

**适用范围**：以下路径是本机（Windows + TIA Portal V21）实测配置。换机器、换 AI 智能体只需替换路径与注册方式——方法论、s7dcl 语法、MCP 工具名都不变。

## 设计期 vs 运行期（先分清楚）

| 阶段 | 干什么 | 用什么 |
|---|---|---|
| **设计期**（本 skill 主体） | 生成 s7dcl / 写块 / 编译 / 保存 / 仿真 | 本地 Python 工具链 + `tia-mcp.exe` |
| **运行期联调**（可选，另一套 skill 才用） | 把程序下发 S7-1200 实跑 / 仿真、读写 I/O、OPC UA 监测 | `plc-ai-agent` 的 OPC UA 仿真器 |

本 skill 只覆盖**设计期**。运行期联调不是它的职责，别把两者混在一个流程里。

## MCP 注册表（本机示例：`~/.workbuddy-ai/mcp.json`，3 个，本任务用到前 2 个）

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

- **`tia-portal`**：设计期主力。底层 .NET，封装 TIA Openness，`TiaPortalV21Dir` 指向 Siemens.Engineering 程序集目录（`Siemens.Engineering.AddIn.Step7` 等 DLL）。在 WorkBuddy 连接器页「信任」后生效；其他平台按各自的 MCP 注册/信任机制加载。
- **`plc-ai-agent`**：运行期。asyncua 做 OPC UA 仿真/联调，模拟器开在 `127.0.0.1:4841`。本 skill 不依赖它。
- **`windows-mcp`**（CursorTouch）：通用 Windows MCP，**与本 skill 无关**，别误挂。

## 其他 AI 智能体怎么接（三种方式，任选其一）

1. **支持 MCP 的客户端**：按上面 `tia-portal` 的 command/args/env 注册；工具名直接用 `bind_project` / `plc_read` / `plc_write` / `compile_check` / `save_project` / `browse_project_tree`。
2. **不支持 MCP**：任何语言起子进程 `tia-mcp.exe --access-mode read-write`，按 JSON-RPC 2.0 over stdio 发 `initialize` → `notifications/initialized` → `tools/call`（`tia_session.py` 30 行可参考）。
3. **完全不用 MCP**：直接用 .NET 调 Siemens.Engineering Openness API（`D:\Siemens\Automation\Portal V21\PublicAPI\V21\net48`），把本文的 s7dcl 文本用 `ImportDocuments` 导入。

## 两种传输层（关键，容易用错）

`tia-mcp.exe` 既能当标准 MCP 被任意客户端注册，也能被**本地直驱**：

1. **本地直驱（工具链实际用法）**：`ladlib.py` / `tia_session.py` 用 `subprocess` 直接拉起
   `C:\Users\Holmes\.dotnet\tools\tia-mcp.exe`，自带 JSON-RPC 2.0 over stdio 握手
   （`Session.__init__` 发 `initialize` → `notifications/initialized` → `tools/call`）。
   好处：能在**一个持久 Python 进程里连续调一串 TIA 工具**（bind→write→compile→save），
   省掉每个调用都重新起进程的开销，也便于带 240–600s 长超时。
2. **平台 MCP 连接器**：同一 `tia-mcp.exe` 注册为 `tia-portal`，由所在平台的 MCP 客户端托管。

设计期跑 `gen*.py` / `ladfc.py` 走的是**第 1 种本地直驱**（`from ladlib import Tia`）。
两条路通同一个 exe，只是宿主不同，**不要以为平台注册了就能自动被 Python 工具链用上**。

## 本地 Python 工具链（`C:\Work\plc-tools\`，非 MCP，直接 import）

| 文件 | 职责 |
|---|---|
| `tia_session.py` | MCP 会话层：拉 `tia-mcp.exe` + JSON-RPC over stdio（`Session`） |
| `ladlib.py` | `Tia` 类：bind/open/close/read/write_block/create_block/delete/compile/save/tree + s7dcl 元件 + `split_bundle`；顶层也 re-export 元件构造器 |
| `ladfc.py` | 构件 API + 步序模板（`step/step0/step_branch`、脉冲版 `*_p`）+ 写块流水线（`write_db/write_fc/report_compile`） |
| `mlc.py` | 中文网络标题 XML 注入（`inject/write_with_titles`）+ `split_bundle` |
| `sw.py` | `switch()` 幂等切项目 |
| `gen1.py / gen3.py / gen5.py` | 各站生成器（数据驱动，把「步序数据」喂给模板） |
| `sim2.py` / `test2_sim.py` | 梯级解释器 + 六类场景仿真断言 |

**运行时**（本机示例）：`C:\Users\Holmes\.workbuddy-ai\binaries\python\envs\default\Scripts\python.exe`
（Python 3.13）。其他环境换成自己的 Python 3.10+ 即可，脚本只依赖标准库 + `tia-mcp.exe`。跑任何 `gen*.py` 都用它。

## 搭配的 skill / agent

- **`skill-creator`**：本 skill（`plc-ladder-design`）就是用它造的——加载它 → `init_skill.py` 建骨架 → 写正文与 references → `quick_validate.py` 校验。
- **`plc-ladder-design`**：本次产物本身。
- **agent**：无。本 skill 纯文档型，不派 general-purpose / Explore 之类子 agent；references 直接读入上下文即可。不引入额外 agent 是为了让流程可复现、可被别的会话原样执行。

## 一键自检（换机 / 换会话先跑）

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

读通即「设计期链路 OK」；运行期再加 `plc-ai-agent` 的 OPC UA 端点探活。
