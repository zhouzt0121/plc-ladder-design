# PLC Ladder Design Methodology (Sequential-Control Flowchart + IO Table → S7 LAD)

> Audience: **any AI agent** (Claude, GPT, local agent frameworks) — no dependency on a specific platform, session memory, or plugin mechanism.
> After reading this document, an agent can independently turn a sequential-control flowchart and an IO table into a compilable, runnable, well-commented Siemens S7-1200/1500 ladder program.
>
> **The flowchart is not limited to IFAE**: IFAE, GPP, SFC, hand-drawn sketches, Excel/Word tables, and plain-text step descriptions all work — as long as the source expresses "what each step does, and what condition advances to the next step". This methodology operates on **step semantics**, independent of the diagram style.
>
> Details live in `references/`; this file is the overview and index.

---

## 0. One-sentence method

**Do not copy an existing program. Use the "two-level decomposition" to turn each flow step into mechanical language, then into LAD elements, assemble them with the fixed block architecture, and verify with both compilation and simulation after writing into TIA.**

---

## 1. Directory layout

```
plc-ladder-design/
├── README.md                      # Chinese overview (primary)
├── README.en.md                   # This document (English overview)
├── SKILL.md                       # Platform entry (WorkBuddy trigger; other AIs may ignore frontmatter)
└── references/
    ├── ladder-grammar.md          # LAD/s7dcl syntax: RUNG/wire/contacts/coils, naming
    ├── step-templates.md          # Step templates: A/C, step/step0/step_branch, pulse-gate variant
    ├── tia-openness.md            # TIA write path: s7dcl/XML bundle, Chinese title injection, dry-check, simulation
    └── toolchain.md               # Toolchain & MCP: real paths, transports, design-time vs runtime
```

> The reference files are written in Chinese. That is intentional: they contain exact s7dcl syntax, tag names, and error messages from the field. This English README gives the complete method; use the references for verbatim element syntax.

---

## 2. Core: two-level decomposition

Each flow step is first decomposed into **mechanical language**, then into **program language**. Do not skip a level.

### Level 1: step action → mechanical language

Translate "step N" into 4 fields. Each field describes only what happens mechanically — no program symbols:

| Field | Question | Example (step 1: turntable pick-up) |
|---|---|---|
| Entry action | At step entry, which actuator does what | Turntable positioning motor rotates to 60° |
| Transition condition | Which physical feedback allows advancing | Turntable in position AND part detected at pick-up point |
| Actuators involved | Which actuator classes | Positioning motor / cylinder / gripper |
| Next state | Where the step advances to | Step 2 |

Mechanical language is restricted to: **position/move, extend/retract, clamp/release, raise/lower, part present/absent, in position/not in position**. It maps one-to-one to the flowchart and can be checked word by word by a human.

### Level 2: mechanical language → program language

| Mechanical language | Program language | Where it hangs |
|---|---|---|
| Entry action (S-type, e.g. "extend") | `S_Coil( output tag )` | After the run gate |
| Entry action (R-type, e.g. "retract/stop") | `R_Coil( output tag )` | After the run gate |
| Transition condition (in position / part present) | `Contact( input tag )` | Transition-evaluation branch |
| Transition condition (negated, e.g. "not in position") | `I_Contact( input tag )` | Transition-evaluation branch |
| Transition condition (delay T) | `TON( pt := T#Ns )` series gate | End of transition-evaluation branch |
| Clear current step + set next step | `R_Coil(STEP[n])` + `S_Coil(STEP[n+1])` | After transition evaluation |

**The most common deadlock**: a flowchart's "entry action" is usually a **step command** (execute immediately on entering the step), not "a consequence of a condition being true". Therefore the action must hang after the gate (`STEP[n] AND (auto-running OR start)`), and **the transition condition may only hang on the downstream branch after the action**. Chaining the action after the transition condition (Template A) causes self-locking such as "wait for the gripper to clamp before issuing the clamp command".

Decision rule:

- Action triggered by a sensor, executed on arrival → **Template A**: `STEP[n] → condition → action`
- Action is a step command, issued on entry → **Template C**: `STEP[n] → gate → action`; `gate → condition → clear/set step`

Sequential-control flowcharts default to **Template C** (unless the step's action is genuinely a consequence of arrival).

---

## 3. Front-back and up-down relationships in LAD

Three rules to internalize before reading or writing LAD:

1. **Front-back = left-to-right within one rung = series = AND.** Current flows from the left power rail (`powerrail`) to the right; each contact is an "and" condition, the coil at the end is the "result". `Contact(A) Contact(B) Coil(C)` means `C = A AND B`.
2. **Up-down = new rungs branching downward from a node = parallel = OR.** A node (`wire#wN`) can be reused by multiple downward `RUNG wire#wN` rungs, each leading to a different coil — equivalent to an OR of the branches at that node.
3. **A node is the anchor of the front-back relationship.** Each `wire#wN` carries the logic value of the AND of all contacts to its left. The whole diagram stitches horizontal series and vertical parallel together through node numbers.

Node hard constraint: **every declared node must be referenced by a later rung; two adjacent nodes must not have a gap** (otherwise an empty segment is produced and the write fails).

---

## 4. Standard block architecture (role of each block)

Fixed 4 blocks + 2 DBs, single responsibility, **call order must not change** (later blocks depend on state established by earlier ones):

| Block | Type | Role | Depends on |
|---|---|---|---|
| `Main` | OB1 | Only calls the FCs below in order, **contains no logic** | — |
| `Start/Stop` | FC | Mode switching, start/stop, emergency stop, clear all steps | Global DB |
| `Return to Initial State` | FC | Initial-position evaluation, reset flow, reset indication | State from previous FC |
| `System Auto-Run` | FC | **Full step sequence**: STEP array + transitions + actions | `AutoRunning` set by previous FCs |
| Process FC (e.g. `Height Check`) | FC | Station-specific process (analog scaling, pass/fail) | Auto-run FC |
| `System Global DB` | GlobalDB | System flags + `STEP[0..15]` array | Shared by all FCs |
| `Timer DB` | GlobalDB | `TON_TIME` members (TON instances) | Used by auto-run FC |

Key conventions:

- `STEP[0..15]` is a Bool array, one bit per step, S/R paired, no integer step numbers.
- `STEP[0]` is the initial step; once auto-run is entered it stays set and **is not cleared during the cycle** (loop-entry normally-open gate); only stop/e-stop/reset clear it.
- Clear all steps with a single `R_BitfieldCoil(operand => "DB".STEP[0], n := 16)`, not 16 `R_Coil`s.

---

## 5. Toolchain and MCP

This methodology is **documentation-only**; it contains no executable scripts. Execution relies on a "Python toolchain + TIA MCP" combination. The configuration below is the **reference setup on this machine** (Windows + TIA Portal V21). On another machine or with another agent, only paths and registration change — the methodology and s7dcl syntax stay the same.

### Design time vs runtime (distinguish first)

| Phase | What | What to use |
|---|---|---|
| Design time | Generate s7dcl / write blocks / compile / save / simulate | Local Python toolchain + `tia-mcp.exe` |
| Runtime commissioning | Download and run on S7-1200 / simulate, read/write I/O, OPC UA monitoring | `plc-ai-agent` (OPC UA simulator) |

This methodology covers **design time only**; do not mix it with runtime commissioning in one flow.

### MCP registration (reference: `~/.workbuddy-ai/mcp.json`)

The JSON below is WorkBuddy's registration format. **Other AI agents can copy command/args/env into their own MCP client format.** Without MCP, call `tia-mcp.exe` over stdio JSON-RPC directly (see next section).

```json
{
  "mcpServers": {
    "windows-mcp": {
      "command": "C:\\awesome-ai-reverse\\Windows-MCP\\.venv\\Scripts\\windows-mcp.exe",
      "args": ["serve"]
    },
    "tia-portal": {
      "command": "C:\\Users\\Holmes\\.dotnet\\tools\\tia-mcp.exe",
      "args": ["--access-mode", "read-write"],
      "env": { "TiaPortalV21Dir": "D:\\Siemens\\Automation\\Portal V21\\PublicAPI\\V21\\net48" }
    },
    "plc-ai-agent": {
      "command": "C:\\Work\\plc-tools\\plc-ai-agent\\.venv\\Scripts\\python.exe",
      "args": ["-m", "plc_ai.mcp.server"],
      "env": {
        "PYTHONPATH": "C:\\Work\\plc-tools\\plc-ai-agent\\src",
        "PLC_USE_SIMULATOR": "true",
        "PLC_ENDPOINT": "opc.tcp://127.0.0.1:4841"
      }
    }
  }
}
```

- **`tia-portal`**: design-time workhorse; .NET wrapper around TIA V21 Openness; `TiaPortalV21Dir` points to the Siemens.Engineering assembly directory.
- **`plc-ai-agent`**: runtime OPC UA simulator; this methodology does not depend on it.
- **`windows-mcp`**: generic Windows capabilities; unrelated.

**How other agents connect** (pick one):

1. MCP-capable client: register `tia-portal` with the command/args/env above; tool names are `bind_project` / `plc_read` / `plc_write` / `compile_check` / `save_project` / `browse_project_tree`.
2. No MCP: spawn `tia-mcp.exe --access-mode read-write` and speak JSON-RPC 2.0 over stdio (`initialize` → `notifications/initialized` → `tools/call`; see `tia_session.py`).
3. No MCP at all: call the Siemens.Engineering Openness API directly (.NET) and import the s7dcl text with `ImportDocuments`.

### Two transports (easy to get wrong)

`tia-mcp.exe` can be registered as a standard MCP server by any client, or driven locally:

1. **Local direct drive (what the toolchain actually does)**: `subprocess` launches `tia-mcp.exe` and performs the JSON-RPC 2.0 over stdio handshake itself. This allows a sequence of TIA calls in one persistent Python process (bind → write → compile → save) without restarting the process per call, and supports 240–600 s timeouts.
2. **Platform MCP connector**: the same exe registered as `tia-portal`, hosted by the platform's MCP client.

**Registering it does not mean the Python toolchain uses it automatically** — two paths to the same exe, different hosts.

### Local Python toolchain (`C:\Work\plc-tools\`)

| File | Responsibility |
|---|---|
| `tia_session.py` | MCP session layer: launches `tia-mcp.exe` + JSON-RPC over stdio |
| `ladlib.py` | `Tia` class: bind/open/read/write_block/create_block/compile/save/tree + s7dcl elements |
| `ladfc.py` | Element API + step templates + block-writing pipeline |
| `mlc.py` | Chinese network-title XML injection |
| `sw.py` | Idempotent project switching |
| `gen1/3/5.py` | Per-station generators (data-driven) |
| `sim2.py` / `test2_sim.py` | Rung interpreter + six-scenario simulation assertions |

Runtime (reference): `C:\Users\Holmes\.workbuddy-ai\binaries\python\envs\default\Scripts\python.exe` (Python 3.13). On another environment use your own Python 3.10+; the scripts depend only on the standard library + `tia-mcp.exe`.

---

## 6. Quick start (any AI can run)

### 6.1 Link self-check

```python
import sys; sys.path.insert(0, r"C:\Work\plc-tools")
from ladlib import Tia
t = Tia()
pl = t.portals()                       # should list TIA Portal instances
if pl:
    t.s.call("bind_project", {"projectPath": r"E:\PLC\#2\#2.ap21",
                              "portalProcessId": pl[0]["processId"]}, 240)
    c = t.read("PLC_1/开车与停车")     # readable = offline and writable
t.close()
```

### 6.2 Generation flow

1. **Extract data**: flowchart + IO table → step data (step number / entry actions / transition conditions / next step / branch list).
2. **Two-level decomposition**: write the 4 mechanical-language fields per step, then map them to program-language elements.
3. **Write LAD**: assemble step networks with the `step / step0 / step_branch` templates; write start/stop/reset networks with the elements.
4. **Write into TIA**: write s7dcl → read back XML and inject Chinese titles → compile 0 errors 0 warnings → save.
5. **Simulation verification**: run the rung interpreter against the real s7dcl, with six-scenario assertions.

---

## 7. Hard constraints (follow each)

- **`I_Contact` is a normally-closed (NC) contact, not "NOT" / rising edge.** Stop/e-stop buttons are physically NC; `I_Contact` gives "stop on broken wire". Use `P_Contact` for rising edge.
- **`P_Contact` refreshes its edge-storage bit every scan**: never share one `bit` across networks. Do edge detection once, produce a single-scan pulse, and let all step networks read that pulse.
- **Wire nodes must be "sourced and referenced"**: floating nodes or adjacent empty segments make the write fail.
- **TON instances go in GlobalDB `TON_TIME` members** (`create_block` does not support InstanceDB); invalid TON syntax can crash TIA Portal — validate syntax changes in small steps.
- **Chinese network titles/comments cannot be written via s7dcl text** (`S7_NetworkTitle` / `TITLE=` / `COMMENT=` are all rejected). Use "build structure with s7dcl → read back FlgNet XML → inject MultilingualText `<Text>` → write XML back".
- **Dual-coil cylinders (extend/retract) must actively S the return coil on reset**, never R both.
- **Revert comments when reverting functionality**, otherwise the comment describes behavior not present in the code.
- **Dry-check before writing + read-back after writing**: read back to confirm the patch is not already applied and the state is clean; after writing assert the new branch exists, the old branch is gone, and the MLC reference count is unchanged; then compile and save.
- **Compiling is not the same as correct logic**: simulation must cover reset / manual / auto-cycle / stop-e-stop / fault / lamp-language.

---

## 8. Reference index

| File | Content |
|---|---|
| `references/ladder-grammar.md` | Exact RUNG/wire/contact/coil syntax, node naming, IO tag naming |
| `references/step-templates.md` | Template A/C decision, `step/step0/step_branch`, pulse-gate variant, choice sequences, last-step loop, dual-coil reset |
| `references/tia-openness.md` | s7dcl/XML bundle, Chinese title injection, compile/save, dry-check, rung simulation |
| `references/toolchain.md` | Toolchain & MCP pairing, real paths, two transports, one-shot self-check |

---

## License

MIT (see `LICENSE`). The methodology is generic; the reference paths and tag names come from a specific Siemens TIA Portal V21 project and are examples, not requirements.
