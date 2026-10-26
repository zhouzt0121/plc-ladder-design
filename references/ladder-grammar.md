# LAD / s7dcl 语法参考

`s7dcl` 是 TIA Portal 的 LAD 文本格式。下面所有元件写法来自实战验证过的 `ladfc.py` 构件，缩进固定：`NETWORK`/`END_NETWORK` 顶格 4 空格，`RUNG`/`END_RUNG` 8 空格，元件 12 空格。

## 块骨架

```
FUNCTION "块名" : Void
    {
        S7_Language := "LAD";
        S7_NetworkComment := "MLC_xxx";   // 注释（多语言引用）
        S7_NetworkTitle := "MLC_xxx"      // 标题（多语言引用）
    }
    NETWORK
        RUNG wire#powerrail
            Contact( "标签" )
            S_Coil( "标签" )
        END_RUNG
    END_NETWORK
END_FUNCTION
```

GlobalDB 是 `DATA_BLOCK "名"`，FC 是 `FUNCTION "名" : Void`，OB1 是 `ORGANIZATION_BLOCK "Main"`。一个块内可含多个 `NETWORK`。

## 元件（直接抄这些精确写法）

| 用途 | s7dcl 写法 |
|---|---|
| 常开触点 NO | `Contact( "标签" )` |
| 常闭触点 NC | `I_Contact( "标签" )` |
| 置位线圈 S | `S_Coil( "标签" )` |
| 复位线圈 R | `R_Coil( "标签" )` |
| 普通线圈（赋值） | `Coil( "标签" )` |
| 一次清 16 个步位 | `R_BitfieldCoil( operand => "DB".STEP[0], n := 16 )` |
| 上升沿触点 | `P_Contact( operand := "标签", bit := "DB".成员 )` |
| TON 定时器 | `"定时器DB".T1.TON( pt := T#2s, et => )` |
| 整除/比较 | `EQ_Contact(in1 := …, in2 := T#4MS) {S7_Templates := "SrcType := Time"}` |
| 实数比较 | `GT_Contact( … ) {S7_Templates := "SrcType := Real"}` |
| 模拟量标定 | `Normalize(min:=0, value:=AI, max:=13824, out=>标准化值)` |
| 缩放 | `Scale(min:=-15.0, value:=标准化值, max:=15.0, out=>测量值)` |
| 偏置 | `Add(in1:=测量值, in2:=32.0, out=>实际值)` |

**引号规则**：普通标签名、`DB.成员` 形式要加双引号（`"系统层全局DB块".系统自动运行中`）。已是引号开头的不重复加。`STEP[0]` 这种位要用 `R_BitfieldCoil`/`S_Coil( "DB".STEP[0] )`。

## RUNG 与 wire（前后关系 = 串联，上下关系 = 并联）

- **一条梯级** = `RUNG wire#起点 … END_RUNG`。从 `起点` 节点一路向右，中途触点串联（AND），末端放线圈。
- **节点** `wire#wN`（`TAP`）是锚点：该点逻辑值 = 左侧所有触点的 AND。
- **横向串联**：同一 `RUNG` 内先写触点再写线圈，中间用 `wire#wN` 分叉。
- **纵向并联（汇入）**：`RUNG wire#powerrail … I_Contact(急停) … END_RUNG wire#w4`，下一条 `RUNG wire#w4 … 动作 …`。多支共下游。
- **纵向并联（汇出）**：`RUNG wire#w1 … 触点 … wire#w2 …` 把 w1 的值继续往 w2 送。
- 节点可被多条 `RUNG wire#同节点` 复用 = 该节点对多支求 OR。

### 一条带分叉的梯级示例（动作 + 转移）

```
RUNG wire#powerrail
    Contact( "系统层全局DB块".STEP[1] )
    wire#w1
    I_Contact( "系统层全局DB块".系统自动运行中 )
    Contact( "启动" )
    wire#w2
    S_Coil( "电机0停止1运行" )        // 动作（进本步即执行）
END_RUNG
RUNG wire#w1
    Contact( "系统层全局DB块".系统自动运行中 )   // 自动档直通 w2
END_RUNG wire#w2
RUNG wire#w2
    R_Coil( "系统层全局DB块".STEP[1] )          // 清当前步
END_RUNG
RUNG wire#w2
    S_Coil( "系统层全局DB块".STEP[2] )          // 置下一步
END_RUNG
```

`w2 = STEP[1] AND (系统自动运行中 OR 启动)`。自动模式走「直通支」，手动模式走「I_Contact+启动」支，末端动作/转移共用同一节点。

## 节点命名约定

模板生成器里节点名有固定套路（见 `step-templates.md`）：

- `powerrail` 电源轨起点。
- `c{步号}a / c{步号}b / c{步号}c`：步的网络内动作源 / 动作执行节点 / 转移判定节点。
- 并联支线用 `w1, w2, …`。
- **每个 `wire#wN` 必须被后续 `RUNG wire#wN` 引用**；两个相邻 `TAP` 之间不能空（否则产生空段，写入失败）。

## IO 标签命名规范（防错设计）

| 类别 | 规则 | 例 |
|---|---|---|
| 模式旋钮 | 两种位置写进名字 | `0手动1自动` |
| 单线圈执行器 | 0/1 物理含义入名 | `电机0停止1运行`、`升降气缸0抬起1落下` |
| 双线圈执行器 | 拆两个独立输出 | `推料气缸缩回` + `推料气缸伸出` |
| 传感器 | 到位信号统一「…到位」 | `搬运初始位`、`升降气缸抬起到位`、`上料点有料` |
| 停止 / 急停 | 物理常闭接线 + `I_Contact` | 断线即停 |
| 指示灯 | 灭/亮入名 | `黄灯0灭1亮`、`蜂鸣器0不响1响` |
| 系统标志 | 放 GlobalDB 成员 | `系统自动运行中`、`系统处于初始位`、`系统正在复位中` |

**标签名以工程里 `list_tag_tables` 读到的真实 tag 为准**，生成器直接照写，不另造名字。IO 表只负责给「地址 → 标签名」的对照。
