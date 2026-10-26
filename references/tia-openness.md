# TIA Openness 写块与验证

把生成好的 LAD 写进 TIA Portal 工程并验证。本机已有工具链（`C:\Work\plc-tools\`）：`ladlib.py`（Tia 类，MCP 会话 + s7dcl 读回 + `write_block`）、`ladfc.py`（构件 + 写块流水线 + `report_compile`）、`mlc.py`（中文标题注入 + `split_bundle` + `write_with_titles`）、`sw.py`（`switch` 切项目）。无工具链时按下面语义用 MCP 工具直接做。

## 前置：TIA 实例与项目

- TIA Portal 实例须由 **后台启动**（否则父进程退出即被回收）。
- 切项目必须 **`close_project` 再 `open_project`**，同一 TIA 实例不能直接换项目；`open_project`/`bind_project` 都带 `portalProcessId`。
- `sw.switch(t, proj)` 是幂等的：当前已是目标项目时只 `bind_project` 校验，不关开。

## 写块（s7dcl）

`write_block(path, s7dcl, s7res=None)` 底层：

1. 打包 `--- FILE: 名.s7dcl ---\n{s7dcl}`，有标题再追加 `--- FILE: 名.s7res ---\n{s7res}`。
2. `plc_write` `operation=update_block_logic` `format=xml` `content=bundle` + `expectedContentHash`（读回块时拿到的，乐观锁）。

要点：
- **bundle 只允许含 `.s7dcl`（+ 可选 `.s7res`），绝不能带 `.xml`**，否则 `Block import did not complete`。
- 新建 GlobalDB / FC 用 `create_block(path, kind, "LAD")`；`create_block` **不支持 InstanceDB**（TON 实例放 GlobalDB 的 `TON_TIME` 成员）。
- 写新 FC 的流水线：`create_block` → `write_fc`（内部走 `mlc.write_with_titles`）→ `report_compile` → `save`。

## 中文网络标题（XML 注入法，唯一可行）

**s7dcl 文本的 `S7_NetworkTitle` / `TITLE=` / `COMMENT=` 四种写法全被拒。** 可行路径：

1. s7dcl 建好结构并 `write_block` 成功。
2. 读回该块 → `mlc.split_bundle(content)` → 取 `{名}.xml`（FlgNet）。
3. 按 `SW.Blocks.CompileUnit` 文档顺序 `mlc.inject(xml, [(标题,注释), …])`，把每个网络的 `MultilingualText CompositionName="Title"/"Comment"` 下的 `<Text>` 替换为中文。
4. 把注入后的 XML 重新打包 `--- FILE: 名.xml ---` 写回（`mlc.write_with_titles` 自动完成 2–4）。

注入后该块的 s7dcl 导出会带 `MLC_` 引用；**再单独用 s7dcl 回写必失败**，如需改结构先 `mlc.strip_mlc()` 去掉 `S7_BlockTitle`/多行属性块。标题建议命名 `MLC_<站号><网络号>`（如 `MLC_0305` = 3# 站第 5 网络）。

## 写前干检 + 读回复核（硬流程）

写块前后各一次读回，避免重复打补丁 / 静默失败：

```
读回目标块 → 确认要打的补丁未应用、旧支路仍在、状态干净 → 写
写 → read → 断言新支路存在、旧支路消失、MLC 引用数不变 → compile → save
```

编译目标 **0 错误 0 警告**；`save` 返回 `(True, '')`。

## 梯级仿真验证（编译过 ≠ 逻辑对）

用「梯级解释器」直接执行真实 s7dcl 文本（不靠 TIA 内核 / PLCSIM），覆盖 6 类场景，跑完全过才交付：

- 复位 / 手动逐步 / 自动循环（合格+不合格）/ 停止急停 / 故障判定 / 灯语矩阵。
- 实现要点：两遍求值（Pass1 节点不动点，Pass2 应用线圈，tap 取节点终值）；`I_Contact` = 常闭；`TON` 从梯级取 `pt`；物理模型气缸二次动作要复位计时器。
- 仿真查出的真 bug 案例：手动冷启动建不了 `STEP[0]`、停止后绿灯仍闪（闪烁支路要串 `I_Contact(已停止)`）。

## 硬约束清单（写块前过一遍）

- wire 节点有源有引用、相邻 TAP 不空。
- `I_Contact` = 常闭；上升沿只用 `P_Contact`，且不与多网络共享 `bit`。
- TON 实例在 GlobalDB `TON_TIME`；改 TON 语法前先小步验证（非法语法会让 TIA 崩溃）。
- 双线圈气缸复位主动 S 回位。
- 改功能同步回退注释。
