# MyLVS 开发计划（Y1）

> 配套阅读：`learning-roadmap.md`（前置知识学习路线）

---

## 0. 文档目的

本文档定义 **MyLVS Y1（约 12 个月）** 的开发目标、分期里程碑、技术选型、项目结构、接口规范、测试策略、风险登记。Y1 结束后另起 Y2 计划。

---

## 1. 项目目标与非目标

### 1.1 最终愿景（多年）
- 用 Rust 自研的商用 LVS 工具
- 支持模拟 / 混合信号，百万级器件 full chip
- 最终对标 **Calibre LVS**

### 1.2 Y1 唯一硬验收
**能在 `../myadc` SKY130 设计上替换 `Magic extract + netgen` 组合，跑通完整流程，结果等价。**

具体到文件：
- 替换 `myadc/sky130/layout/verification/run_lvs.sh` 里的 `magic … extract` 和 `netgen` 两步
- 通过 `myadc/sky130/layout/verification/run_top_lvs.sh` 的端到端流程
- 所有子 cell（comparator / dac / sar_logic / sampling_sw）和 top cell（sar_adc_top，747 devices / 361 nets）LVS 结论与 netgen 一致

### 1.3 Y1 非目标（明确不做）
- PTM 28nm 支持
- OASIS、CDL、Spectre netlist、Verilog structural
- 自研 rule deck DSL
- 百万器件规模优化（有多 cell 复用，但不刻意打磨）
- 人类可读的 debug GUI / RVE 风格界面 / KLayout marker 插件
- FinFET、多 patterning、coloring

### 1.4 Y1 设计原则
1. **AI-native 输出**：主输出格式是结构化 JSON，schema 稳定，exit code 严谨，适合 agent 直接消费
2. **增强工具，不降级输入**：mylvs 跑不通 myadc 的任何真实文件时，改 mylvs，不改 myadc；失败 case 写进 `BUGS.md`
3. **工艺无关的内核，工艺相关的 adapter**：核心算法不写死 SKY130 任何 layer/device，通过 tech adapter 读 Magic `.tech` 和 netgen `setup.tcl`
4. **Path I → Path II 顺序**：Path I（只替换 netgen）打磨到能用，再切 Path II（替换 Magic extract）

---

## 2. Path I vs Path II 分工

### Path I — Netlist vs Netlist（M1–M4）
```
Magic extract (保留)  →  extracted.spice   ┐
                                           ├→  mylvs compare  →  JSON report
schematic.spice                            ┘
```
核心：SPICE parser + Gemini 图同构 + 参数匹配 + 报告。

### Path II — Full LVS（M5–M7）
```
sar_adc_top.gds  →  mylvs extract  →  extracted.spice   ┐
                                                         ├→  mylvs compare  →  JSON
schematic.spice                                          ┘
```
新增：GDS reader + polygon boolean + layer derivation + connectivity + device recognition。

### M8 — 层次化 + 性能收尾

---

## 3. 里程碑

每个里程碑给出：**目标 / 主要交付 / 验收 / 前置学习 / 主要风险**。

### M1 — 基础建设（月 1）

**目标**：项目骨架 + SPICE 解析 + 核心数据结构 + CLI + JSON schema v1。

**交付**：
- Cargo workspace：`mylvs-core` / `mylvs-spice` / `mylvs-cli`
- SPICE parser 支持：`.subckt/.ends`、`.include/.lib`、`.param`、`X…` 子电路实例、M/R/C/V/I 器件、`nfet_01v8`/`pfet_01v8` 等 SKY130 方言
- 内部 IR：`Device { id, kind, params, pins: [NetId] }` + `Net { id, name, pins: [DeviceId] }`，全部 arena + `u32` index
- CLI 子命令雏形：`mylvs parse` / `mylvs compare`（stub）/ `--format json`
- JSON schema v1（见 §6）

**验收**：
- 解析 `myadc/sky130/top/subcircuits.spice`（几百行，多层 subckt）无错
- 解析 Magic flat extraction 产物 `sar_adc_top_flat.spice` 无错
- Round-trip：parse → dump 成 SPICE → 再 parse，结果同构

**前置学习**：阶段 1（SPICE 细节方言）、阶段 6（Rust arena 技巧）

**风险**：
- Magic extract 产物的 body pin 约定 (VPB/VNB) 和 hierarchy 前缀处理——`run_lvs.sh` 第 34–38 行有 `sed` 后处理，mylvs 需要把这些映射做成**可配置规则**（不是 hard-code）

---

### M2 — Flat 比较 α（月 2–3）

**目标**：最小可用的 netlist comparator，能处理 100 器件量级的 flat 对比。

**交付**：
- Gemini 式 partition refinement 算法（`mylvs-compare` crate）
  - 初始 signature：device kind + degree + 邻居 kind multiset
  - 迭代 refinement 至稳定
  - Ambiguous class → backtrack 搜索（McGregor 风格，限深度）
- 参数精确匹配：W、L、M、NF（容差 = 0，后面放宽）
- JSON mismatch 报告 v1：`mismatches[]` 每条有 category + layout_ref + schema_ref

**验收**：
- `comparator/pmos_comp` cell（约 16 MOS）对比结论 = netgen
- 人为引入错配（改 W、swap source/drain、加/删 device）能定位到具体 device
- 黄金对拍：netgen `comp.out` 结论 vs mylvs JSON 结论，100% 一致

**前置学习**：阶段 4（Gemini + partition refinement）

**风险**：
- 回溯深度爆炸：先限死 depth，后面再上优化
- 对称结构（差分对）会在这期暴露，M3 处理

---

### M3 — Flat 比较 β（月 4–5）

**目标**：处理模拟设计常见的等价变换，能跑通 myadc 除 top 之外的所有 cell。

**交付**：
- Pre-simplification pass：
  - 并联 MOSFET 合并（same type + same pin set → M 累加，fingered 处理）
  - 串联 MOSFET / R / C 合并
  - 短路净合并
- 对称结构：automorphism class 检测 + 成组匹配（Gemini 原论文方案）
- `flatten` 选项：按层次边界 flatten schematic 和 layout 网表
- 参数容差配置：W/L 相对容差（默认 1%，可配）

**验收**：
- `dac/cap_dac` cell（~100 个 MOM cap）对比 = netgen
- `sar_logic` cell（含标准单元，几百个器件）flat 对比 = netgen
- `sampling_sw`（bootstrap switch，对称）对比 = netgen

**前置学习**：阶段 4 续（Gemini 对称处理）、WOMBAT 论文

---

### M4 — Path I 合拢（月 6）

**目标**：替换 `run_top_lvs.sh` 的 netgen 一步，整个 top-level flat LVS 跑通。

**交付**：
- Tech adapter 雏形：读 `~/sky130A/libs.tech/netgen/sky130A_setup.tcl` 的 `permute` / `property` / `equate` 指令
- Magic extract 产物后处理内化（把 `run_lvs.sh` 第 33–51 行的 sed 规则变成 mylvs 内建 option）
- End-to-end regression 脚本：`tests/myadc_regression/` 下定期跑全套 cell + top
- `BUGS.md` v1 模板（和 rustspice 对齐）

**验收（Path I 官宣交付）**：
- 把 `run_top_lvs.sh` 里的 `netgen -batch lvs …` 替换成 `mylvs compare …`
- `sar_adc_top` 对比结论 = netgen（PASS）
- 回归脚本全绿
- 对 myadc 至少发现 3 个 mylvs 自身 BUG（若 0 个则说明测试不够）

**前置学习**：阶段 5 初探（层次化概念）、netgen `setup.tcl` 语义（读它源码 + doc）

**风险**：
- netgen setup.tcl 指令繁多，Y1 不需要全支持，只支持 myadc 用到的（大概 10–20 条指令）

---

### M5 — GDS + 几何（月 7–8）

**目标**：GDSII reader 可用，核心 polygon 运算就位。Path II 开端。

**交付**：
- GDS reader：至少支持 BOUNDARY / PATH / BOX / SREF / AREF（可以先 fork `gds21` 再扩展）
- Polygon boolean（扫描线实现）：`AND` / `OR` / `NOT` / `XOR` / `SIZE`
- R-tree 空间索引
- Layer derivation 脚本执行器（Magic `.tech` DRC-style 规则的子集）
- CLI：`mylvs extract-geom --gds X.gds --tech sky130A --layer poly_and_diff`（debug 用）

**验收**：
- 读 `sar_adc_top.gds` 无错，cell 数 / element 数与 KLayout 一致
- 简单 layer 运算（如 `poly AND ndiff`）对拍 KLayout 脚本结果（多边形面积总和误差 < 1e-4）

**前置学习**：阶段 2（GDS 格式）、阶段 3 前半（polygon boolean、R-tree）

**风险**：
- Polygon boolean 几何退化情况（共线边、自交叉）是经典坑，实现前先读 `geo-rs`、`i-overlay` 等已有 crate 源码

---

### M6 — Extraction（月 9–10）

**目标**：从 GDS 提取出 device + connectivity，替代 Magic extract。

**交付**：
- Connectivity：union-find over 导体层，via/contact 跨层 merge
- Device recognition：SKY130 核心器件
  - `nfet_01v8` / `pfet_01v8`（MOSFET，提取 W/L）
  - `cap_mim` 或 MOM cap（myadc 的 cap_dac 用）
  - （电阻可选，myadc 未必用到——先跳过）
- W/L 几何测量：gate 沿宽方向、沟道沿长方向
- 输出 SPICE netlist，格式与 Magic `ext2spice lvs` 兼容

**验收**：
- `pmos_comp.gds` 提取 netlist 与 `magic ext2spice` 产物对比：
  - device 数 100% 一致
  - net 数 100% 一致（允许 naming 差异，靠 mylvs compare 验证等价）
  - W/L 参数差 < 0.5%（几何量化误差允许）
- `sar_adc_top.gds` flat 提取跑通（不要求参数精度，要求 device/net count 吻合 ±1）

**前置学习**：阶段 3 后半（layer derivation、device recognition、W/L 提取）

**风险**：
- SKY130 标准单元（sar_logic 用的 sky130_fd_sc_hd）内部结构复杂，M6 阶段可以要求输入先 flatten 再提，不做 hierarchical extraction——hierarchical 归 M8

---

### M7 — Path II 合拢（月 11）

**目标**：`mylvs run --gds X.gds --schematic X.spice --tech sky130A` 一条命令跑完整 LVS。

**交付**：
- 端到端 CLI：`mylvs run`
- extract 产物直接进 compare，不落盘（可选 `--dump-extracted` 调试用）
- Mismatch 报告能关联到 GDS 坐标（layout_ref 带 `gds_xy`）

**验收（Path II 官宣交付）**：
- `sar_adc_top.gds` + `sar_adc_top_combined.spice` 一键跑通
- 结论与 Magic + netgen 组合一致
- 至少 1 个 cell 跑得比 Magic + netgen 快（时间墙钟）

---

### M8 — 层次化 + 性能（月 12）

**目标**：为 Y2 打地基——层次化雏形 + rayon 并行。

**交付**：
- Hierarchical extraction：按 cell 边界单独提 + 缓存（文件指纹作 key）
- Hierarchical comparison：subckt 边界先匹配，递归进入
- `rayon` per-cell 并行 extraction
- 性能 benchmark harness：对 myadc 各 cell 计时、定期跑、存在 `benches/`

**验收**：
- 层次化路径跑通 `sar_adc_top`，结果与 flat 路径一致
- 相比 M7，wall-clock 至少快 2×
- 相比 Magic + netgen 组合，wall-clock 至少快 3×（myadc 这个规模 netgen 本来就快，目标定得保守）

---

## 4. 技术选型

### 4.1 Rust crate 清单

| 用途 | 首选 | 备注 |
|---|---|---|
| Parser combinator | `nom` v7 | SPICE / Magic `.tech` 都用它 |
| CLI | `clap` v4 (derive) | |
| 错误 | `thiserror` + `anyhow` | lib 用 thiserror，bin 用 anyhow |
| 序列化 | `serde` + `serde_json` | JSON 输出 |
| JSON schema | `schemars` | 生成 schema 给下游 agent 校验 |
| Graph 参考 | `petgraph` | 仅做参考，不直接用（我们用自家 arena） |
| 并行 | `rayon` | M8 |
| Snapshot test | `insta` | 黄金对拍主力 |
| Property test | `proptest` | 图同构算法做 fuzz |
| 日志 | `tracing` + `tracing-subscriber` | 结构化日志，agent 友好 |
| GDS | fork `gds21` | M5，起步对照学习 |
| Polygon boolean | 看 `i-overlay` / `geo-booleanop` 决定 fork 还是自写 | M5 再评估 |
| R-tree | `rstar` | M5 |

### 4.2 数据结构纪律
- **Arena + typed index**：`Vec<Device>` + `DeviceId(u32)`；禁用 `Rc<Device>` / `Arc<Node>`
- **零拷贝 parser**：输入 `&[u8]`，`nom` + `&str` 视图，`String` 只在最终 IR 里出现
- **Small-vec 优化**：Device 的 pin 列表用 `SmallVec<[NetId; 4]>`（MOS 4 脚占绝大多数）

---

## 5. 项目结构

```
mylvs/
├── Cargo.toml                  # workspace
├── crates/
│   ├── mylvs-core/             # IR: Device/Net/Netlist/Hierarchy, arena 基建
│   ├── mylvs-spice/            # SPICE / CDL parser（M1 只 SPICE）
│   ├── mylvs-tech/             # Tech adapter: 读 Magic .tech / netgen setup.tcl
│   ├── mylvs-compare/          # 图同构 + 参数匹配 + 简化 pass (M2, M3)
│   ├── mylvs-report/           # JSON schema 定义 + 渲染
│   ├── mylvs-gds/              # GDS reader (M5)
│   ├── mylvs-geom/             # polygon boolean + R-tree (M5)
│   ├── mylvs-extract/          # connectivity + device recognition (M6)
│   └── mylvs-cli/              # 二进制入口
├── tests/
│   ├── myadc_regression/       # 对 ../myadc 的每个 cell 跑回归
│   ├── fixtures/               # 小型人造测试电路
│   └── snapshots/              # insta 快照
├── benches/                    # criterion benchmark (M8)
├── docs/
│   ├── learning-roadmap.md     # 前置学习
│   ├── development-plan.md     # 本文档
│   ├── json-schema.md          # 输出格式契约
│   └── ai-agent-guide.md       # 给 AI agent 的使用说明（M4 交付）
├── BUGS.md                     # 缺陷清单（rustspice 风格）
├── README.md
└── CLAUDE.md                   # Claude/agent 工作守则
```

---

## 6. 输出接口规范

> 主消费者是 AI agent，所以接口要 **机器可靠 + LLM 可读** 兼顾。

### 6.1 CLI 合约

```
mylvs compare --layout LAYOUT.spice --schematic SCHEMATIC.spice --tech TECH [--format json|text] [--out PATH]
mylvs extract --gds X.gds --tech TECH --cell NAME [--out PATH]
mylvs run     --gds X.gds --schematic X.spice --tech TECH [--out PATH]
```

**Exit code**：
- `0` = LVS PASS（网表等价）
- `1` = LVS FAIL（有 mismatch）
- `2` = 输入错误（文件不存在 / parse 失败 / tech file 不对）
- `3` = 工具内部错误（panic / 未实现）

**Stderr** = 运行日志（tracing 输出）；**Stdout** = 结构化结果（默认 JSON）。严格分离。

### 6.2 JSON schema v1 草案

```json
{
  "schema_version": "1.0.0",
  "mylvs_version": "0.1.0",
  "status": "PASS" | "FAIL" | "ERROR",
  "summary": {
    "layout_devices": 747,
    "schema_devices": 747,
    "layout_nets": 361,
    "schema_nets": 361,
    "mismatches": 0,
    "runtime_ms": 123
  },
  "mismatches": [
    {
      "id": "mm-001",
      "category": "parameter_mismatch" | "device_mismatch" | "net_mismatch" | "port_mismatch" | "topology_mismatch",
      "severity": "error" | "warning",
      "layout_ref": {
        "cell": "sar_adc_top_flat",
        "instance_path": "pmos_comp_0/M3",
        "device_kind": "nfet_01v8",
        "params": { "W": 1.0, "L": 0.15 },
        "gds_xy": [12345, 67890],
        "source_file": "sar_adc_top_flat.spice",
        "source_line": 1024
      },
      "schema_ref": {
        "cell": "pmos_comp",
        "instance_path": "X1.M3",
        "device_kind": "nfet_01v8",
        "params": { "W": 0.84, "L": 0.15 },
        "source_file": "pmos_comp_schematic.spice",
        "source_line": 42
      },
      "message": "Parameter W mismatch: layout=1.0um, schematic=0.84um (delta 19%).",
      "hypotheses": [
        { "text": "Layout W is rounded up to a larger bin, update schematic.", "confidence": 0.6 },
        { "text": "Wrong finger count in schematic.", "confidence": 0.3 }
      ]
    }
  ],
  "stats": { "partition_iterations": 7, "backtrack_hits": 0 }
}
```

**稳定性规则**：
- 任何字段新增：minor version bump，agent 允许忽略
- 任何字段移除或重命名：major version bump，agent 必须升级
- `mismatches` 排序必须确定性（按 `id`，`id` 由内容 hash 生成），这样 diff 两次运行稳定

### 6.3 AI agent 使用指南（`docs/ai-agent-guide.md`，M4 交付）
- 推荐消费模式：`--format json` + stdout 解析
- Exit code 判断
- `hypotheses` 字段作为 agent 定位问题的起点
- 分级输出模式：`--summary-only` / `--focus <cell>`

---

## 7. 测试与回归

### 7.1 测试金字塔
- **单元测试**：每个 crate 自带，覆盖算法核心（partition refinement、union-find、polygon ops）
- **集成测试**：`tests/fixtures/` 下 20–50 个 ≤ 10 器件的手造 netlist，覆盖 pass / fail / 各类 mismatch
- **Snapshot 测试**：`insta` + 黄金 JSON 文件，回归防护
- **Property 测试**：`proptest` 造随机 netlist，验证"自己和自己 LVS 必 PASS"
- **myadc 回归**：CI 脚本周期跑全部 myadc cell，对拍 netgen

### 7.2 myadc 回归脚本雏形
```bash
# tests/myadc_regression/run.sh
for cell in pmos_comp cap_dac sar_logic sampling_sw sar_adc_top; do
    mylvs compare … > $cell.json
    diff <(jq '.status' $cell.json) <(grep -q 'Circuits match uniquely' ../myadc/.../$cell.out && echo \"PASS\" || echo \"FAIL\")
done
```

---

## 8. BUGS.md 模板

沿用 rustspice 风格：

```markdown
# MyLVS Bug Tracking

## Open

### BUG-001: <一句话标题>
- **Discovered**: 2026-MM-DD
- **Context**: 跑 myadc 的 <cell> 时 …
- **Symptom**: mylvs 报 <现象>；netgen 期望 <现象>
- **Reproducer**: `mylvs compare --layout … --schematic …`
- **Root cause (if known)**: …
- **Blocks**: M<n> 验收 / myadc <cell>
- **Priority**: P0 / P1 / P2

## Fixed

### BUG-XXX: ... (commit abc1234, 2026-MM-DD)
```

---

## 9. 风险登记

| # | 风险 | 缓解 |
|---|---|---|
| R1 | Gemini 回溯搜索在某些 corner case 指数爆炸 | 限深度，超限 fallback 到分治报 ambiguous |
| R2 | Polygon boolean 几何退化 bug 影响正确性 | 先用已有 crate（`i-overlay`），稳定后评估是否自造 |
| R3 | SKY130 标准单元（sky130_fd_sc_hd）内部复杂，M6 提取难度超预期 | M6 允许先 flatten 再提，hierarchical extraction 放 M8 |
| R4 | netgen `setup.tcl` 里有用到但 mylvs 未支持的指令 | M4 开始就把未支持指令列进 BUGS.md，逐个补 |
| R5 | Magic extract 的 body pin / hierarchy 前缀约定不稳定 | tech adapter 里做成可配置规则，参考 `run_lvs.sh` 的 sed |
| R6 | 单人开发时间压力 | 每个 M 末尾 re-plan，必要时压缩 M8（层次化/性能可 Y2） |
| R7 | JSON schema 频繁变动破坏下游 agent | 开发期 0.x 自由变，M4 Path I 交付起冻结 v1.0 |

---

## 10. 立刻可做的第一步

1. `cargo new --bin mylvs-cli` + 建 workspace
2. 把 myadc 几个代表性 SPICE 文件复制到 `tests/fixtures/myadc/` 作为永久 fixture（或直接引用 ../myadc 路径）
3. M1 起步：SPICE parser + IR
4. 新建 `BUGS.md`（现在空）、`docs/json-schema.md`（M1 末尾填充）、`CLAUDE.md`（工作守则）

---

## 附：与学习路线的映射

| 里程碑 | 必须完成的学习阶段 |
|---|---|
| M1 | 阶段 1（SPICE）、阶段 6（Rust 工程化基础） |
| M2 | 阶段 4 前半（Gemini 基本算法） |
| M3 | 阶段 4 后半（对称、预简化） |
| M4 | 阶段 5（层次化概念） |
| M5 | 阶段 2（GDS 格式）、阶段 3 前半（几何算法） |
| M6 | 阶段 3 后半（extraction、device recognition） |
| M7 | — |
| M8 | 阶段 5 深入 + 阶段 6 并行 |

> 建议：M1 开工前把阶段 0–1 学完，不用等阶段 3–4，边开发边学。
