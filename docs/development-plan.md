# MyLVS 开发计划（Y1 · 任务细化版）

> 配套：`learning-roadmap.md`

**本版说明**：不按周/月分配，按**任务粒度**拆。每个任务的 DoD（Definition of Done）必须包含代码 + 单元测试 + 所属 regression。**M1–M8 是逻辑 phase，不是时间框**。任务 ID 命名：`T-<phase>.<group>.<n>`（例：`T-M1.SP.02` 表示 M1 阶段 SPICE parser 组第 2 号）。

---

## Part I — 目标、策略、总览

### §0 Y1 唯一硬验收
**替换 `../myadc` SKY130 设计里的 `Magic extract + netgen`，跑通完整流程。**

- `myadc/sky130/layout/verification/run_lvs.sh` 和 `run_top_lvs.sh` 能用 mylvs 跑
- 覆盖 cell：`pmos_comp` / `cap_dac` / `sar_logic` / `sampling_sw` / `sar_adc_top`（747 devices / 361 nets）

### §1 Y1 非目标
- PTM 28nm、OASIS、CDL、Spectre、Verilog structural、FinFET、multi-patterning
- 自研 rule deck DSL（Y1 直接吃 Magic `.tech` + netgen `setup.tcl`）
- 百万器件性能优化
- 人类 debug GUI / RVE / KLayout marker

### §2 Y1 设计原则
1. **AI-native 输出**：JSON 主输出，exit code 严谨，stderr/stdout 分离
2. **增强工具，不降级输入**：遇到 myadc 跑不通就改 mylvs，不改 myadc；入 `BUGS.md`
3. **工艺无关内核 + 工艺相关 adapter**
4. **Path I → Path II 顺序**

### §3 Path I / Path II
```
Path I  (M1–M4):  extracted.spice + schematic.spice   →  mylvs compare  →  JSON
Path II (M5–M7):  X.gds + schematic.spice            →  mylvs run     →  JSON
M8: hierarchical + parallel
```

### §4 Phase 总览

| Phase | 名称 | 关键任务组 | Phase 验收（硬卡点） |
|---|---|---|---|
| M1 | 基建 + SPICE parser + IR | SETUP / CLI / IR / SP / LOW / OUT / FIX / REG | 解析全部 myadc SPICE 文件 + round-trip |
| M2 | Flat compare α | CMP / SIG / REF / MATCH / PARAM / RPT | `pmos_comp` LVS = netgen |
| M3 | Flat compare β | SIMP / SYM / FLAT / TOL | `cap_dac` / `sar_logic` / `sampling_sw` = netgen |
| M4 | Path I 合拢 | TECH / POSTPROC / HYP / AGUIDE | `sar_adc_top` 全流程 = netgen |
| M5 | GDS + 几何 | GDS / BOOL / RTREE / LAYER | `nmos_gate` layer 对拍 KLayout |
| M6 | Extraction | CONN / DEV / MEAS / EXTOUT | `pmos_comp.gds` 提取 = Magic |
| M7 | Path II 合拢 | RUN / XY | `sar_adc_top.gds` 端到端 = Magic+netgen |
| M8 | 层次化 + 性能 | HIER / PAR / BENCH | 相比 Magic+netgen 快 3× |

---

## Part II — 任务级开发计划

### §5 Phase M1 — 基建 + SPICE + IR

**Phase DoD**：
- [ ] 所有 M1 任务 DoD 勾完
- [ ] `mylvs parse` 能吃下 myadc 下列文件无错：`subcircuits.spice`、`sar_adc_top_flat.spice`、`pmos_comp_schematic.spice`、`pmos_comp_extracted.spice`、`sar_adc_top_schematic.spice`、`cap_dac_schematic.spice`、`sar_logic_schematic.spice`、`sampling_sw_schematic.spice`
- [ ] 所有上述文件 round-trip 结果 IR 同构
- [ ] CI 全绿（fmt / clippy / test）
- [ ] `BUGS.md` 至少记录 M1 中暴露的 1 个真实 bug（否则说明覆盖不足）

---

#### 组 SETUP — 基础设施

##### T-M1.SETUP.01 — Cargo workspace
- **Del**：`Cargo.toml` (workspace root) + 9 个 `crates/*/Cargo.toml`（按 §13 结构）
- **Unit**：`cargo check --workspace` 通过
- **Reg**：CI `cargo check` job
- **Deps**：—
- **DoD**：`cargo build --workspace` 成功

##### T-M1.SETUP.02 — Rust 版本固定
- **Del**：`rust-toolchain.toml`，channel = "1.84" 或发版时 stable
- **DoD**：rustup 自动切换正确版本

##### T-M1.SETUP.03 — 共享依赖
- **Del**：workspace root `Cargo.toml` 的 `[workspace.dependencies]` 列 §12 全部 crate
- **DoD**：各 crate 从 workspace 统一引用

##### T-M1.SETUP.04 — CI workflow
- **Del**：`.github/workflows/ci.yml` 跑 `fmt --check` / `clippy -- -D warnings` / `test --workspace`
- **DoD**：PR 触发全 job 通过

##### T-M1.SETUP.05 — 纪律文件骨架
- **Del**：`BUGS.md` / `CLAUDE.md` / `docs/json-schema.md` / `README.md` 空壳（标题 + sections）
- **DoD**：文件存在

---

#### 组 CLI — 命令行骨架

##### T-M1.CLI.01 — clap 命令树
- **Del**：`crates/mylvs-cli/src/cli.rs`，定义 `parse` / `compare` / `extract` / `run` 四个子命令（后三个 M1 只占位）
- **Unit**：`tests/cli_help.rs` 断言 `--help` 输出包含所有子命令
- **DoD**：`mylvs --help`、`mylvs parse --help` 输出正常

##### T-M1.CLI.02 — 全局 flag
- **Del**：`--format json|text`、`-v/--verbose` / `-vv`、`--out PATH`
- **Unit**：`tests/cli_args.rs` 覆盖各 flag 组合
- **DoD**：flag 解析正确；未知 flag 返回 exit 2

##### T-M1.CLI.03 — tracing 初始化
- **Del**：`mylvs-cli/src/logging.rs`，`tracing_subscriber::fmt().with_writer(std::io::stderr)`
- **Unit**：`tests/cli_logging.rs` 验证 `-v` 提升日志级别
- **DoD**：日志走 stderr，非 JSON 污染 stdout

##### T-M1.CLI.04 — Exit code 主循环
- **Del**：`mylvs-cli/src/main.rs`，把 `Result<i32>` 映射到退出码（0/1/2/3/4 按 §14.1）
- **Unit**：`tests/exit_codes.rs`（5 个 case 覆盖 5 个退出码）
- **DoD**：每个退出码都能从正常路径产生

---

#### 组 IR — 核心数据结构

##### T-M1.IR.01 — Id 类型
- **Del**：`crates/mylvs-core/src/ids.rs` — `CellId` / `DeviceId` / `NetId` / `ModelId`（均 `u32` newtype，derive Copy/Eq/Hash/Serialize）
- **Unit**：`ids::tests` 验证 `Copy`、`Hash`、序列化
- **DoD**：无 heap 分配、`size_of<DeviceId>() == 4`

##### T-M1.IR.02 — Netlist / Cell / Device / Net 结构
- **Del**：`crates/mylvs-core/src/netlist.rs`（按 §15 定义）
- **Unit**：手写构造一个 2 MOS + 1 R 的 Netlist，验证 arena 索引正确
- **DoD**：结构稳定；`#[derive(Debug, Serialize)]`

##### T-M1.IR.03 — DeviceKind / MosFlavor / BjtFlavor
- **Del**：`crates/mylvs-core/src/kind.rs`
- **Unit**：enum 变体 serialize 成可读字符串（`"nfet"`, `"pfet"`, ...）
- **DoD**：JSON 输出人类可读

##### T-M1.IR.04 — DeviceParams / ParamValue
- **Del**：`crates/mylvs-core/src/params.rs`，`w/l/m/nf/value: Option<f64>` + `extras: BTreeMap<String, ParamValue>`
- **Unit**：round-trip `{ w: 1e-6, extras: { "vth": Expr("..") } }`
- **DoD**：浮点 serialize 固定 6 位有效位（避免表示漂移）

##### T-M1.IR.05 — SourceLoc + Arc<str> interning
- **Del**：`crates/mylvs-core/src/source.rs` + `crates/mylvs-core/src/intern.rs`（用 `dashmap` 或自家 `HashMap<&'static str, Arc<str>>`）
- **Unit**：同一字符串 intern 两次返回同一 `Arc::ptr_eq` 实例
- **DoD**：不增加线性内存

##### T-M1.IR.06 — NetlistBuilder
- **Del**：`crates/mylvs-core/src/builder.rs`，链式 API：`.add_cell().add_net().add_device().pin()`
- **Unit**：builder 构造结果与手工构造一致
- **DoD**：所有 ID 分配单调 u32，复查无重复

##### T-M1.IR.07 — Ground net 识别
- **Del**：`intern::is_ground(name)` 认识 `0`、`gnd`、`gnd!`、`vss`（可配置列表）
- **Unit**：16 个变体（大小写、带感叹号、别名）全部识别
- **DoD**：合并同名 ground 为单一 NetId

---

#### 组 SP — SPICE parser

##### T-M1.SP.01 — Tokenizer / lexer
- **Del**：`crates/mylvs-spice/src/lex.rs`，按行分 token，支持 `*` / `$` 注释、`+` 续行、大小写无关
- **Unit**（`lex::tests`）：
  - `comment_line`：`"* hello"` → 空 token 流
  - `inline_dollar`：`"M1 d g s b n $ nmos"` → 前 5 token + 注释被剥
  - `continuation`：`"R1 a b\n+ 1k"` → 合成单行
  - `case_insensitive`：`.SUBCKT` 和 `.subckt` 同 token
- **DoD**：所有 unit 通过；fuzz 1000 个随机输入不 panic

##### T-M1.SP.02 — 数值 literal 解析
- **Del**：`crates/mylvs-spice/src/num.rs::parse_number -> Result<f64>`
- **Unit**（`num::tests`）：
  - `integer`：`"42"` → 42.0
  - `float`：`"3.14"` → 3.14
  - `scientific`：`"1.5e-6"` → 1.5e-6
  - `si_k`：`"1k"` → 1e3
  - `si_meg`：`"2meg"` → 2e6
  - `si_u`：`"2.5u"` → 2.5e-6
  - `si_f`：`"10f"` → 10e-15
  - `unit_ignored`：`"1.5um"` → 1.5e-6
  - `case`：`"1K"` → 1e3, `"1MEG"` → 1e6
  - `invalid`：`"abc"` → Err
- **DoD**：所有通过；表驱动覆盖 `{k, meg, g, u, n, p, f, mil, a, t, m}`

##### T-M1.SP.03 — 标识符 / net 名
- **Del**：`crates/mylvs-spice/src/ident.rs::parse_ident`，允许 `[A-Za-z_][A-Za-z0-9_\[\]\.#!]*`
- **Unit**：包含 `#`（Magic 风格）、`!`（global）、`.`（hier）、`[n]`（bus）的 net 名全部过
- **DoD**：myadc 里出现过的所有 net 名能 parse

##### T-M1.SP.04 — 表达式 passthrough
- **Del**：`crates/mylvs-spice/src/expr.rs`，识别 `{...}` 或 `'...'` 包裹，存原文不求值
- **Unit**：嵌套大括号、转义、未闭合 → 正确报错
- **DoD**：嵌套 3 层大括号能提取

##### T-M1.SP.05 — `key=value` 参数
- **Del**：`crates/mylvs-spice/src/kv.rs::parse_params -> BTreeMap<String, ParamValue>`
- **Unit**：
  - `w_l`：`"W=1 L=0.15"` → `{w: 1, l: 0.15}`
  - `with_expr`：`"R={R1+R2}"` → `{r: Expr("R1+R2")}`
  - `mixed`：`"W=1u L=0.15u M=4 NF=2"` → 四键全中
  - `duplicate`：`"W=1 W=2"` → Err
- **DoD**：参数键大小写无关（规范化为小写）

##### T-M1.SP.06 — M device（MOSFET）
- **Del**：`crates/mylvs-spice/src/dev/mos.rs`，识别 `M<name> D G S B MODEL [params]`
- **Unit**：
  - `basic`：`"M1 d g s b nmos W=1 L=0.15"` → 结构体字段正确
  - `no_params`：`"M2 d g s b pmos"` → W/L None
  - `with_finger`：`"M3 d g s b nmos W=1 L=0.15 M=4 NF=2"` → M/NF 填充
- **DoD**：解析结果 round-trip 成 SPICE 源文格式

##### T-M1.SP.07 — R device
- **Del**：`crates/mylvs-spice/src/dev/res.rs`
- **Unit**：
  - `value_only`：`"R1 a b 1k"` → value = 1000
  - `with_model`：`"R2 a b res_poly R=2k"` → model="res_poly", value=2000
  - `param_form`：`"R3 a b R=1k"` → value=1000

##### T-M1.SP.08 — C device
- **Del**：`dev/cap.rs`；Unit 类似 R

##### T-M1.SP.09 — L / V / I / D / Q devices
- **Del**：`dev/ind.rs` / `dev/src.rs` / `dev/diode.rs` / `dev/bjt.rs`
- **Unit**：每类 2–3 个 case（V/I 独立源最小 metadata）
- **Note**：V/I M1 只存参数，compare 不涉及

##### T-M1.SP.10 — X subckt 实例
- **Del**：`dev/subckt.rs`，识别 `X<name> n1 n2 ... SUBCKT [params]`
- **Unit**：
  - `sky130_mos`：`"Xn1 d g s b sky130_fd_pr__nfet_01v8 W=1 L=0.15"` → SubcktInstance + params
  - `plain`：`"X1 in out vdd vss my_cell"` → SubcktInstance, no params
- **Note**：M1 保留为 `DeviceKind::SubcktInstance`，**M4 tech adapter 做 MOSFET 下沉**

##### T-M1.SP.11 — `.subckt` / `.ends` directive
- **Del**：`directive/subckt.rs`
- **Unit**：
  - `basic`：`.subckt inv a b y` + body + `.ends` → 含 3 port 的 cell
  - `with_params`：`.subckt buf in out PARAMS: W=1 L=0.15` → default params 挂 cell
  - `unterminated`：缺 `.ends` → Err

##### T-M1.SP.12 — `.include` directive（递归 + cycle 检测）
- **Del**：`directive/include.rs`；递归深度 max 32
- **Unit**：
  - `linear`：A include B include C → 结构正确
  - `cycle`：A include B include A → Err(IncludeCycle)
  - `relative_path`：相对主文件路径解析

##### T-M1.SP.13 — `.lib PATH SECTION` directive
- **Del**：`directive/lib.rs`；section = `.lib NAME` 到 `.endl NAME` 之间
- **Unit**：
  - `basic`：`.lib foo tt` 命中 `tt` section
  - `section_not_found`：Err
- **Reg**：能读 `sky130.lib.spice` 的 `tt` section

##### T-M1.SP.14 — `.param` directive
- **Del**：`directive/param.rs`
- **Unit**：`.param vdd=1.8 tt_temp=25` → 两个 param 记录

##### T-M1.SP.15 — `.model` directive（passthrough）
- **Del**：`directive/model.rs`，存 name + kind + body 原文
- **Unit**：SKY130 `.model nmos_example nmos( level=57 ... )` 能存储

##### T-M1.SP.16 — `.global` / `.option` / `.end` / `.endl`
- **Del**：`directive/misc.rs`
- **Unit**：各一个 case

##### T-M1.SP.17 — 未支持 directive warning
- **Del**：`directive/fallback.rs`，`.tran`/`.dc`/`.ac`/`.print`/`.save`/`.step` 等归并 warn 不阻塞
- **Unit**：每种 directive 触发一条 `warnings[]` 条目

##### T-M1.SP.18 — 错误类型 + 行号
- **Del**：`crates/mylvs-spice/src/error.rs::SpiceError { kind, file, line, col, ctx }`
- **Unit**：错误带上精准行号；3 个故意错误 case 验证

##### T-M1.SP.19 — Parser 顶层入口
- **Del**：`crates/mylvs-spice/src/lib.rs::parse_file(path) -> Result<SpiceFile>`
- **Unit**：4 个 fixture（basic / with-subckt / with-include / with-lib）

---

#### 组 LOW — AST → IR lowering

##### T-M1.LOW.01 — AST → Netlist lowering
- **Del**：`crates/mylvs-spice/src/lower.rs::lower(ast) -> Netlist`
- **Unit**：2 MOS + 1 subckt 的 AST → Netlist，device/net 计数正确
- **DoD**：端口顺序保留

##### T-M1.LOW.02 — Net 去重 / alias
- **Del**：`lower::NetRegistry`，同名 net 合并
- **Unit**：5 个 device 引用同一 net 结果 `Net.pins.len() == 5`

##### T-M1.LOW.03 — Subckt 内外 scope
- **Del**：subckt 内 net 命名不污染顶层
- **Unit**：两 subckt 各有 `a` net，顶层互不影响

##### T-M1.LOW.04 — Ground 自动识别
- **Del**：用 `intern::is_ground` 标记 `Net.is_ground = true`
- **Unit**：`0` 和 `vss` 均被标记

##### T-M1.LOW.05 — 错误收集（不致命）
- **Del**：lower 过程累积 `warnings[]` 而非 panic
- **Unit**：未知 subckt 引用 warn 但 Netlist 可构造

---

#### 组 OUT — 输出 / round-trip

##### T-M1.OUT.01 — Netlist → SPICE dumper
- **Del**：`crates/mylvs-core/src/dump.rs::to_spice(nl) -> String`
- **Unit**：手造 Netlist → SPICE → 重新 parse → 同构（`assert_eq!` 在关键字段）
- **DoD**：输出风格稳定，字段序固定

##### T-M1.OUT.02 — Netlist → JSON dumper
- **Del**：`mylvs-core/src/dump.rs::to_json(nl) -> String`，serde 序列化
- **Unit**：JSON schema 稳定；`jq` 能读

##### T-M1.OUT.03 — CLI `parse` 输出
- **Del**：`mylvs-cli/src/cmd/parse.rs` 实现 `mylvs parse FILE --format json|text`
- **Unit**：集成测试跑 fixture → stdout JSON 非空、exit 0

---

#### 组 FIX — 测试 fixture

##### T-M1.FIX.01 — myadc 代表性 fixture 拷贝
- **Del**：`tests/fixtures/myadc/` 放入以下软链或副本：
  - `sky130/top/subcircuits.spice`
  - `sky130/layout/verification/sar_adc_top_flat.spice`
  - `sky130/layout/verification/sar_adc_top_combined.spice`
  - `sky130/comparator/pmos_comp_schematic.spice`
  - `sky130/comparator/pmos_comp_extracted.spice`
  - `sky130/dac/cap_dac_schematic.spice`
  - `sky130/sar_logic/sar_logic_schematic.spice`
  - `sky130/sampling_sw/bootstrap_sw.spice`（或对应 schematic）
- **DoD**：`ls tests/fixtures/myadc/*.spice` 有至少 8 个文件

##### T-M1.FIX.02 — 合成小 fixture
- **Del**：`tests/fixtures/synth/` 10–20 个手造 netlist 覆盖：
  - `inv.spice`（2 MOS）
  - `nand2.spice`（4 MOS）
  - `diff_pair.spice`（差分对，M3 对称测试用）
  - `rc_filter.spice`
  - `subckt_depth3.spice`（3 层嵌套）
  - `include_cycle.spice` + partner（负面测试）
  - `malformed_missing_ends.spice`（负面）
  - `x_device_sky130.spice`（SKY130 X 调用）
- **DoD**：整目录一次性导入 tests 全通过

---

#### 组 REG — 回归

##### T-M1.REG.01 — `tests/myadc_regression/parse_all.sh`
- **Del**：shell 脚本遍历 `tests/fixtures/myadc/*.spice` 跑 `mylvs parse`，任何非 0 退出 → fail
- **DoD**：脚本能被 CI 调用，全绿

##### T-M1.REG.02 — Round-trip regression
- **Del**：`tests/roundtrip.rs`（Rust 集成测试）
- **Cases**：每个 `myadc` / `synth` fixture：`parse → dump → parse` 两次 IR `assert_eq!`（忽略 SourceLoc）
- **DoD**：CI 跑通

##### T-M1.REG.03 — Snapshot harness
- **Del**：`tests/snapshots/`（`insta` 管理），每个合成 fixture 的 JSON IR 存快照
- **DoD**：首次跑 `cargo insta accept`，之后回归防护

##### T-M1.REG.04 — BUGS.md M1 条目
- **Del**：M1 开发中至少记录 1 个真实 bug（示例见 §20）
- **DoD**：Fixed 分区有对应 commit hash

---

### §6 Phase M2 — Flat compare α

**Phase DoD**：
- [ ] 所有 M2 任务完成
- [ ] `pmos_comp` LVS 结论 = netgen（PASS）
- [ ] 4 种错配注入（W 改、S/D swap、删 MOS、加 MOS）全部定位正确 device
- [ ] snapshot 入库
- [ ] proptest ≥ 1000 次随机 bipartite graph 自对比全 PASS

---

#### 组 CMP — 比较器框架

##### T-M2.CMP.01 — Bipartite graph 构建
- **Del**：`crates/mylvs-compare/src/graph.rs::Bipartite`，device 节点 + net 节点 + 邻接表
- **Unit**：
  - `empty`：Netlist 空 → 空图
  - `inv`：2 MOS 的 inv → 2 device 节点 + 4 net 节点
  - `with_ground`：ground net 单独 flag
- **DoD**：`Bipartite::stats()` 输出 device_count_by_kind

##### T-M2.CMP.02 — CompareOpts + Report 结构
- **Del**：`mylvs-compare/src/opts.rs` + `mylvs-report/src/report.rs`（按 §14.2 JSON schema）
- **Unit**：默认 opts 序列化正确
- **DoD**：schema_version = "1.0.0"

##### T-M2.CMP.03 — compare 主入口
- **Del**：`mylvs-compare/src/lib.rs::compare(a, b, opts) -> Report`
- **Note**：骨架实现，各子步骤在后续任务填
- **DoD**：未实现处 `unimplemented!("stage=X")` 明确

---

#### 组 SIG — 签名

##### T-M2.SIG.01 — Device 初始 signature
- **Del**：`mylvs-compare/src/sig.rs::device_sig_initial`
- **Formula**：`hash(kind_id, degree, sorted(pin_role_ids))`
- **Unit**：
  - 两个结构相同的 inv 的 MOS 初始 sig 相等
  - 改 kind（nmos→pmos）sig 变
  - S/D 对称性：因为 pin_role 把 S/D 归一，swap 后 sig 不变

##### T-M2.SIG.02 — Net 初始 signature
- **Del**：`sig::net_sig_initial`
- **Formula**：`hash("NET", degree, sorted(neighbor_kinds), is_port, is_ground)`
- **Unit**：
  - ground net 和普通 net 不同
  - port net 和内部 net 不同

##### T-M2.SIG.03 — Refine step
- **Del**：`sig::refine_step(graph, sigs) -> new_sigs`
- **Formula**：`new_sig[n] = hash(old_sig[n], sorted(old_sig[neighbor] for neighbor in n))`
- **Unit**：
  - 单步 refine 在 inv 上：sig 发生变化
  - 多次 refine 最终稳定
  - 两个同构图最终 sig 序列相等

##### T-M2.SIG.04 — Refine 主循环（带收敛）
- **Del**：`sig::refine_until_stable(graph, max_iter=32)`
- **Unit**：
  - 单链路网络收敛在 3 步内
  - 高对称图（2x2 grid）5 步内
  - 超 max_iter → Err(DidNotConverge)
- **DoD**：触发条件上报 `Report.stats.partition_iterations`

##### T-M2.SIG.05 — Hash seed 控制
- **Del**：`sig::SIG_HASHER_SEED` 固定常量；运行时可环境变量覆盖
- **Unit**：同输入两次运行 sig 字节一致

---

#### 组 REF — Partition

##### T-M2.REF.01 — Partition by signature
- **Del**：`compare/partition.rs::partition_by(sigs) -> Vec<(Sig, Vec<NodeId>)>`
- **Unit**：
  - 均匀 sig → 一个 partition
  - 两个 sig → 两个 partition
  - 排序稳定

##### T-M2.REF.02 — Partition 多重集对比
- **Del**：`partition::signature_multiset_eq(pa, pb)`
- **Unit**：
  - 相同多重集 → true
  - class size 差异 → false
- **DoD**：不等时返回具体差异，填入 `Mismatch::topology_partition`

---

#### 组 MATCH — 匹配

##### T-M2.MATCH.01 — Unique class 绑定
- **Del**：`compare/match.rs::bind_unique(pa, pb, &mut bindings)`
- **Unit**：2 个 1-size class → 2 个 binding

##### T-M2.MATCH.02 — Bindings 存储
- **Del**：`compare/bindings.rs::Bindings { device: BiMap, net: BiMap }`
- **Unit**：双向查询 O(1)；冲突 binding 报错

##### T-M2.MATCH.03 — Backtrack search（McGregor 风格，限深度）
- **Del**：`match::backtrack_match(ga, gb, class_a, class_b, bindings, depth)`
- **Params**：max_depth = 6, max_class_size = 8
- **Unit**：
  - 差分对（class size 2）成功
  - NAND 树（class size 4）成功
  - 故意构造歧义：class size 3 × 成功后再验证
  - 超限 → Err(AmbiguousExceedsDepth)
- **DoD**：backtrack_hits 上报到 `Report.stats`

##### T-M2.MATCH.04 — Ambiguous 无解 → mismatch 上报
- **Del**：超过 max_depth 仍未解 → `Mismatch::topology_ambiguous`
- **Unit**：构造 2 个仅差一边的同样签名图，得到精确 mismatch 位置

---

#### 组 PARAM — 参数比较

##### T-M2.PARAM.01 — 精确参数比较（tol=0）
- **Del**：`compare/params.rs::params_equal(p_a, p_b, tol=0)`
- **Unit**：W/L/M/NF 任一不等 → false

##### T-M2.PARAM.02 — 参数 mismatch 条目生成
- **Del**：`compare/params.rs::diff_params(da, db) -> Vec<Mismatch>`
- **Unit**：改 W → 一条 parameter_mismatch，字段齐全

##### T-M2.PARAM.03 — Extras 参数暂时忽略
- **Del**：`params_equal` 不看 extras（M3 才放开）
- **Unit**：extras 不同但主参数相同 → equal

---

#### 组 RPT — 报告

##### T-M2.RPT.01 — Mismatch 构造器
- **Del**：`mylvs-report/src/builder.rs::MismatchBuilder` — fluent API
- **Unit**：link 出 5 种 category 的 mismatch 全字段覆盖

##### T-M2.RPT.02 — Mismatch ID（确定性）
- **Del**：`id = "mm-" + blake3(canonical_serialize(mismatch))[..12]`
- **Unit**：同输入 ID 字节一致；不同输入 ID 不同
- **DoD**：JSON `mismatches[]` 排序按 ID，回归 diff-able

##### T-M2.RPT.03 — Summary 区构造
- **Del**：`report::build_summary(a, b, bindings, mms)` 填充 §14.2 summary 块
- **Unit**：device/net/by_kind 计数准确

##### T-M2.RPT.04 — JSON / text 双输出
- **Del**：`report::render(&Report, Format) -> String`
- **Unit**：
  - JSON 能被 `serde_json::from_str` 读回
  - text 包含 PASS/FAIL 汇总 + mismatch 摘要

##### T-M2.RPT.05 — `mylvs compare` 命令 MVP
- **Del**：`mylvs-cli/src/cmd/compare.rs`
- **Unit**：集成测试跑 2 个 fixture → exit 0 / 1，JSON 输出合法
- **DoD**：运行 `mylvs compare --layout A --schematic B` 产出 JSON

---

#### 组 REG — M2 回归

##### T-M2.REG.01 — `pmos_comp.sh` 端到端
- **Del**：`tests/myadc_regression/pmos_comp.sh`
- **Cmd**：
  ```
  mylvs compare \
    --layout  tests/fixtures/myadc/pmos_comp_extracted.spice \
    --schematic tests/fixtures/myadc/pmos_comp_schematic.spice \
    --format json > out.json
  ```
- **Assert**：`jq .status out.json` == "PASS"；对拍 `../myadc/.../comp.out`

##### T-M2.REG.02 — 错配注入测试
- **Del**：`tests/fixtures/myadc_negatives/` 4 个变种：
  - `pmos_comp_schematic_wrongW.spice`（改 W）
  - `pmos_comp_schematic_swap_sd.spice`
  - `pmos_comp_schematic_missing_M3.spice`
  - `pmos_comp_schematic_extra_M.spice`
- **Del**：`tests/negative_injection.rs`（Rust 集成测试）
- **Unit**：每个变种跑 mylvs，断言：
  - status = FAIL
  - mismatches[0].category 匹配预期
  - mismatches[0].layout_ref.instance_path 指向正确 device
- **DoD**：4 个全通过

##### T-M2.REG.03 — Snapshot 快照
- **Del**：pmos_comp 正向 + 4 错配变种共 5 份 JSON 快照（`insta`）
- **DoD**：`cargo insta test` 无 diff

##### T-M2.REG.04 — Proptest 随机同构
- **Del**：`tests/proptest_selfcompare.rs`
- **Strategy**：生成合法 Netlist（5–50 device），mylvs compare 自身必 PASS
- **Runs**：≥ 1000
- **DoD**：CI 跑通

---

### §7 Phase M3 — Flat compare β

**Phase DoD**：
- [ ] `cap_dac` / `sar_logic` / `sampling_sw` 三 cell LVS = netgen
- [ ] 并联 / 串联合并各一个专门黄金 fixture 过
- [ ] 对称 swap class 不报 mismatch 黄金 fixture 过
- [ ] simplify trace 在 JSON 中可回溯

---

#### 组 SIMP — 预简化

##### T-M3.SIMP.01 — SimplifyPass trait 与 pipeline
- **Del**：`compare/simplify/mod.rs::SimplifyPass`，`run(nl, ctx)` 返回变更统计
- **Unit**：空 pipeline 不改 Netlist

##### T-M3.SIMP.02 — 并联 MOS merge pass
- **Del**：`simplify/parallel_mos.rs`
- **Rule**：同 kind + 同 (D,G,S,B) 4 元组 (S/D 视为无序) + params 兼容 → `M += other.M`, `NF += other.NF`, 其余必须 identical
- **Unit**：
  - 差分对每腿 4 finger → 合并成 2 device
  - 参数冲突（W 不同）→ 不合并
  - Fingered + 整数倍 W → 不合并（`NF` 已表示）
- **DoD**：trace 记录 `{merged_from: [Id, Id], merged_into: Id}`

##### T-M3.SIMP.03 — 串联 MOS merge pass
- **Del**：`simplify/series_mos.rs`
- **Rule**：两 MOS 共享 degree-2 中间 net + 链式 + kind/W 相等 → stacked MOS，L 累加
- **Unit**：NAND 堆叠 2 MOS → 1 stacked
- **DoD**：stacked 能再次参与 parallel merge（组合性）

##### T-M3.SIMP.04 — 串联 R merge
- **Del**：`simplify/series_r.rs`；R 总 = R1 + R2
- **Unit**：3 个 R 链式 → 1 R
- **Edge**：中间 net 有第三条连接 → 不合并

##### T-M3.SIMP.05 — 并联 C merge
- **Del**：`simplify/parallel_c.rs`
- **Unit**：2 C 并联 → 1 C

##### T-M3.SIMP.06 — 短路合并
- **Del**：`simplify/short.rs`；R < 1e-6 视为短路，合并两端 net
- **Unit**：5 个 0Ω 电阻连 3 个 net → 1 net

##### T-M3.SIMP.07 — CLI `--simplify none|basic|aggressive`
- **Del**：`cmd/compare.rs` 接入 opts
- **Unit**：三档 trace 计数不同

##### T-M3.SIMP.08 — Simplify trace 写入 JSON
- **Del**：`Report.stats.simplify` 字段
- **Unit**：3 种 merge 各触发一次，字段准确

##### T-M3.REG.01 — Simplify 黄金 fixture
- **Del**：`tests/fixtures/simplify/` 各类合并的小电路
- **Unit**：`tests/simplify_golden.rs` 断言合并后 device 数 / trace 内容
- **DoD**：每类 simplify 一个 positive + 一个 negative

---

#### 组 SYM — 对称处理

##### T-M3.SYM.01 — Automorphism class 检测
- **Del**：`compare/sym.rs::detect_automorphism(graph, class)`
- **Unit**：差分对 class 被识别；非对称 class 不被

##### T-M3.SYM.02 — Swap group binding
- **Del**：`bindings::bind_swap_group(class_a, class_b)` — 整组绑，不要求一对一
- **Unit**：差分对两侧 PASS，不论具体怎么对
- **DoD**：`Report.stats.swap_groups` 计数 >= 1

##### T-M3.SYM.03 — 对称 + 参数匹配
- **Del**：swap group 内参数必须"成多重集等"
- **Unit**：
  - 对称 + 参数全同 → PASS
  - 对称但参数不同 → `parameter_mismatch`（组级别，不是单 device）

##### T-M3.REG.01 — 差分对黄金 fixture
- **Del**：`tests/fixtures/synth/diff_pair.spice` + 错配变种
- **Unit**：正向 PASS；故意让一边 W 不同 → FAIL with 正确归因
- **DoD**：snapshot 入库

---

#### 组 FLAT — Flatten 选项

##### T-M3.FLAT.01 — Schematic flatten pass
- **Del**：`compare/flatten.rs`，递归展开 `SubcktInstance` 到顶层，实例名用 `X1/X2/M3` 路径形式
- **Unit**：
  - 2 层嵌套 schematic → 展平后 device 数 = 两边之和
  - 循环 subckt 调用 → 之前已被 parser 禁止
- **DoD**：展平后 source_loc 保留原文件行号

##### T-M3.FLAT.02 — Layout flatten pass（M4 也用）
- **Del**：同上但跑在 extracted netlist
- **Unit**：Magic flat extract 产物本已 flat，此 pass 应是 identity；带层次的 hier extract 产物（未来）展平

##### T-M3.FLAT.03 — CLI `--flatten-layout` / `--flatten-schematic`
- **Del**：`cmd/compare.rs` 接入

---

#### 组 TOL — 参数容差

##### T-M3.TOL.01 — Tol 配置结构
- **Del**：`CompareOpts::tols { w_rel, l_rel, m_rel, nf_rel, value_rel }` 默认 1e-6（几乎精确）
- **Unit**：序列化正确

##### T-M3.TOL.02 — 相对容差比较
- **Del**：`params::params_equal_with_tol(pa, pb, tol)`
- **Unit**：
  - 1% tol + 0.5% 偏差 → equal
  - 1% tol + 2% 偏差 → not equal

##### T-M3.TOL.03 — Tech 文件覆盖 tol
- **Del**：`sky130a.toml` 写 `w_rel = 0.01`，CLI `--tech sky130a` 生效
- **Unit**：CLI 显式 `--w-tol` > tech > default

##### T-M3.TOL.04 — CLI flag `--w-tol` / `--l-tol`
- **Del**：`cmd/compare.rs` 接入
- **Unit**：覆盖正确

##### T-M3.REG.01 — 容差黄金
- **Del**：`tests/fixtures/myadc/` 变种：故意 W 差 0.5% → 不同 tol 下 PASS/FAIL 切换

---

#### 组 REG — M3 myadc 回归

##### T-M3.REG.10 — `cap_dac.sh`
- **Del**：`tests/myadc_regression/cap_dac.sh`
- **Cmd**：`mylvs compare --layout ... --schematic .../cap_dac_schematic.spice --simplify basic`
- **Assert**：status=PASS, 对拍 netgen
- **DoD**：全绿

##### T-M3.REG.11 — `sar_logic.sh`
- **Del**：`tests/myadc_regression/sar_logic.sh`；如需 flatten 先加 `--flatten-schematic`
- **Assert**：status=PASS
- **DoD**：全绿（sar_logic 是 M3 最硬的卡点，含标准单元）

##### T-M3.REG.12 — `sampling_sw.sh`
- **Del**：`tests/myadc_regression/sampling_sw.sh`
- **Assert**：对称 bootstrap 能 PASS

##### T-M3.REG.13 — Snapshot 扩展
- **Del**：三 cell 各一份黄金 JSON（`insta`）
- **DoD**：cargo insta test 无 diff

---

### §8 Phase M4 — Path I 合拢

**Phase DoD**：
- [ ] `run_top_lvs.sh` 里 netgen 替换为 mylvs 跑全绿
- [ ] `sar_adc_top` 对比结论 = netgen
- [ ] 至少 3 个 BUGS.md Fixed 条目
- [ ] `docs/ai-agent-guide.md` 发布
- [ ] 版本 tag `v0.1-path1`

---

#### 组 TECH — Tech adapter

##### T-M4.TECH.01 — netgen setup.tcl parser
- **Del**：`crates/mylvs-tech/src/netgen.rs`
- **Cmds supported**：`permute`, `property`, `equate classes`, `ignore class`, `ignore pin`
- **Unit**：`tests/fixtures/netgen_setup/` 放入 `sky130A_setup.tcl` 子集 + 4 个 toy setup
- **DoD**：可查询 "nfet 的 S/D 是否 permutable"

##### T-M4.TECH.02 — `permute` 应用到 signature
- **Del**：device 初始 signature 考虑 permutable pins（S/D 归一）
- **Unit**：有 permute 和无 permute 下差分对签名行为对比

##### T-M4.TECH.03 — `property tolerance` 应用到参数
- **Del**：tech 提供 per-kind W/L tol
- **Unit**：nmos 1% tol 对 pmos 不生效

##### T-M4.TECH.04 — `ignore class` 实现
- **Del**：compare 忽略指定 kind
- **Unit**：`.option ignore class capacitor` → 全部 cap 不影响结论

##### T-M4.TECH.05 — 未支持指令 warn + BUGS.md
- **Del**：未识别命令 → `warnings[]` + 自动记 BUGS.md 候选
- **Unit**：setup.tcl 里塞一个假命令 `foobar`，mylvs 能 warn 且不崩

##### T-M4.TECH.06 — SKY130 device 硬编码映射
- **Del**：`crates/mylvs-tech/src/sky130a.rs::SKY130A_DEVICE_MAP`（按 §18.2）
- **Unit**：myadc 用到的所有 `sky130_fd_pr__*` 命名都命中
- **DoD**：查询函数 `map_subckt_to_device_kind(name)` 覆盖率 100%

##### T-M4.TECH.07 — X device 下沉到 MOSFET
- **Del**：Netlist 后处理 pass：`SubcktInstance(sky130_fd_pr__nfet_01v8)` → `Mosfet { flavor: N }`，pin 按 [D,G,S,B] 映射
- **Unit**：Magic 的 `X Mxxx ...` 和 schematic 的 `X Xn1 ...` 下沉后 compare 对齐

---

#### 组 POSTPROC — Magic extract 后处理

##### T-M4.POSTPROC.01 — 后处理规则配置
- **Del**：`crates/mylvs-tech/src/postproc/`，TOML 格式（§18.4）
- **Unit**：解析 `sky130a-flat.toml` 正确

##### T-M4.POSTPROC.02 — 正则 net rename
- **Del**：`postproc::apply_net_rename`
- **Unit**：`pmos_comp_0.VPB` → `vdd`、`w_123_456#` → `vdd`、`SUB` → `vss`

##### T-M4.POSTPROC.03 — 实例前缀 strip
- **Del**：`postproc::strip_prefix`
- **Unit**：`top_routing_0.M1` → `M1`

##### T-M4.POSTPROC.04 — `mylvs compare --magic-postproc NAME`
- **Del**：CLI 绑定
- **Unit**：默认 `sky130a-flat`；`none` 跳过

##### T-M4.POSTPROC.05 — Regression
- **Del**：`tests/myadc_regression/postproc.sh` 专门验证 VPB/VNB/SUB/w_# 四类全变
- **Assert**：变换前后 device 数一致，ground/vdd net 合并

---

#### 组 HYP — Hypotheses

##### T-M4.HYP.01 — Hypothesis generator 框架
- **Del**：`mylvs-report/src/hypothesis.rs::HypoGenerator` trait，mismatch in → Vec<Hypothesis> out

##### T-M4.HYP.02 — W/L mismatch 假设生成
- **Del**：
  - "Check finger count" (delta > 20% 时)
  - "Bin rounding difference" (delta < 20% 时)
  - "Swapped W/L" (互换后匹配时)
- **Unit**：每条触发条件有测试

##### T-M4.HYP.03 — 缺失 device 假设
- **Del**：
  - "Extraction missed a polygon in layout"
  - "Schematic forgot subckt instantiation"
- **Unit**：删一个 device 的变种触发

##### T-M4.HYP.04 — 多余 net 假设
- **Del**：
  - "Unrouted port in layout"
  - "Floating intermediate node in schematic"

##### T-M4.HYP.05 — Confidence 定标
- **Del**：confidence 0..1 基于启发规则
- **Unit**：同 mismatch 至少 1 条 confidence >= 0.5

---

#### 组 AGUIDE — AI agent 文档

##### T-M4.AGUIDE.01 — `docs/ai-agent-guide.md`
- **Del**：文档内容：
  - 推荐调用模式
  - exit code 语义表
  - JSON schema 导航（field → 用途）
  - 典型 mismatch 处理流程（用 `hypotheses[0]` 做归因）
  - 多次运行 diff 策略
  - 错误恢复建议
- **DoD**：至少 5 个 agent 端使用示例代码片段（Python / bash 调用）

---

#### 组 REG — M4 回归

##### T-M4.REG.01 — `sar_adc_top.sh`
- **Del**：`tests/myadc_regression/sar_adc_top.sh`
- **Cmd**：仿照 myadc 的 `run_top_lvs.sh` 第 69–72 行，替换 netgen 为 mylvs
- **Assert**：status=PASS；对比 netgen 输出

##### T-M4.REG.02 — `run_all.sh` 汇总
- **Del**：遍历 5 个 cell + top，汇总 PASS/FAIL
- **Exit**：任一 FAIL → 非 0

##### T-M4.REG.03 — CI nightly 跑全量
- **Del**：`.github/workflows/nightly.yml`，每天 02:00 跑 myadc_regression 全量
- **DoD**：失败发邮件 / GitHub issue

##### T-M4.REG.04 — 压测 negative 回归
- **Del**：`tests/myadc_regression/negatives/` 放 5–10 个刻意构造错配的 sar_adc_top 变种
- **Assert**：每个都 FAIL 且 mismatch 定位正确

##### T-M4.REG.05 — Schema v1.0 冻结测试
- **Del**：`tests/schema_frozen.rs`，确保 JSON 输出符合 `docs/json-schema.md` 约定
- **DoD**：任何 schema 向后不兼容改动触发 test 失败

---

### §9 Phase M5 — GDS + 几何

**Phase DoD**：
- [ ] 读 `sar_adc_top.gds` cell / element 计数 = KLayout
- [ ] `poly AND ndiff` 面积对拍 KLayout < 1e-4 误差
- [ ] polygon boolean fuzz ≥ 10000 次全通过
- [ ] R-tree 查询正确性 unit test 全通过

---

#### 组 GDS — GDS reader

##### T-M5.GDS.01 — Record 基础 I/O
- **Del**：`crates/mylvs-gds/src/record.rs` — 读二进制 record（tag + length + data）
- **Unit**：小手写 GDS fixture 解析正确

##### T-M5.GDS.02 — Library / Structure / Element 解析
- **Del**：支持 HEADER, BGNLIB, LIBNAME, UNITS, BGNSTR, STRNAME, ENDSTR, ENDLIB
- **Unit**：空 GDS 库能读

##### T-M5.GDS.03 — BOUNDARY / BOX / PATH
- **Del**：`elements.rs`
- **Unit**：合成 fixture：含 10 个 BOUNDARY、5 个 BOX、3 个 PATH，全部字段读回

##### T-M5.GDS.04 — SREF / AREF + 坐标变换
- **Del**：STRANS / ANGLE / MAG 支持；AREF COLROW
- **Unit**：
  - 旋转 90°/180°/270° 坐标正确
  - mirror + rotate 组合
  - AREF 3x3 array 展开坐标对拍 KLayout

##### T-M5.GDS.05 — 错误恢复 / warning
- **Del**：未知 record warn 不崩；截断文件报错明确
- **Unit**：3 个 malformed fixture

##### T-M5.GDS.06 — `mylvs extract --dump-layers` 输出
- **Del**：CLI 子命令，把各 layer polygon 数 dump 成 JSON
- **Unit**：集成测试跑 myadc 小 GDS → JSON 字段齐全

##### T-M5.GDS.REG.01 — sar_adc_top.gds 读入
- **Del**：`tests/myadc_regression/gds_read.sh`
- **Assert**：cell 数 / BOUNDARY 计数 与 KLayout 脚本结果对齐

---

#### 组 BOOL — Polygon boolean

##### T-M5.BOOL.01 — 评估 / 集成 `i-overlay` 或 fork
- **Del**：选型决策文档 `docs/polygon-lib-eval.md`
- **DoD**：2–3 个候选对比 bench 后定结论

##### T-M5.BOOL.02 — PolygonSet API
- **Del**：`crates/mylvs-geom/src/polyset.rs::PolygonSet`
- **Ops**：`and / or / not / xor / size(d) / area / bbox`
- **Unit**：
  - 2 矩形相交 → 面积正确
  - L 形 union → 边界正确
  - size(+d) / size(-d) 双向
- **Unit count**：≥ 30 个

##### T-M5.BOOL.03 — 定点整数坐标
- **Del**：坐标类型 `Coord = i64`（dbu 单位，1 dbu = 1 nm）
- **Unit**：0.5 nm 偏移不丢精度

##### T-M5.BOOL.04 — Fuzz vs 参考实现
- **Del**：`tests/geom_fuzz.rs`，`proptest` 生成随机多边形对比 KLayout（通过 Python 脚本 subprocess 生成 ground truth）
- **Runs**：≥ 10000
- **DoD**：面积差 < 1e-6；边界 hash 一致率 >= 99%

##### T-M5.BOOL.REG.01 — 黄金案例
- **Del**：`tests/fixtures/geom/` 放 20 个手造多边形对、各种 bool 结果
- **DoD**：snapshot 入库

---

#### 组 RTREE — 空间索引

##### T-M5.RTREE.01 — 封装 `rstar`
- **Del**：`mylvs-geom/src/rtree.rs`，按 bbox 插入 + 查询
- **Unit**：插入 10K polygon + 1K 查询，结果与 brute force 一致

##### T-M5.RTREE.02 — 按 layer 分索引
- **Del**：`LayerIndex { layer_id -> RTree }`
- **Unit**：每层查询隔离

##### T-M5.RTREE.03 — 内存 / 性能 benchmark
- **Del**：`benches/rtree.rs`（criterion）
- **Goal**：1M polygon 查询 < 1ms
- **DoD**：report check in repo

---

#### 组 LAYER — Layer derivation

##### T-M5.LAYER.01 — Magic `.tech24` parser（子集）
- **Del**：`crates/mylvs-tech/src/magic_tech.rs`，解析 `types` / `planes` / `contact` / `styles` 段
- **Unit**：`sky130A.tech24` 里 myadc 用到的 layer 全部能列出
- **DoD**：导出 `Tech { layers: Vec<Layer>, contacts: Vec<Contact> }`

##### T-M5.LAYER.02 — Derived layer DSL 内部表示
- **Del**：`mylvs-geom/src/layer_expr.rs`，`Expr = Base(layer) | And(Box, Box) | Or(...) | Not(...) | Size(Box, f64)`
- **Unit**：parse `"poly AND ndiff"` → 正确 Expr

##### T-M5.LAYER.03 — 规则执行器
- **Del**：`layer_expr::evaluate(expr, layer_sets)` → PolygonSet
- **Unit**：
  - `nmos_gate = poly AND ndiff` 面积对拍 KLayout
  - `poly_contact = cc AND poly`

##### T-M5.LAYER.04 — 规则文件加载
- **Del**：tech 配置 TOML 支持 derived layers 声明
- **Unit**：`sky130a.toml` 声明 5 个 derived layer

##### T-M5.LAYER.REG.01 — myadc layer derivation
- **Del**：`tests/myadc_regression/layer_nmos_gate.sh`
- **Assert**：对 `pmos_comp.gds` 算 `nmos_gate` 总面积 = KLayout

---

### §10 Phase M6 — Extraction

**Phase DoD**：
- [ ] `pmos_comp.gds` 提取 device / net 数与 Magic `ext2spice` 相等
- [ ] `pmos_comp.gds` W/L 误差 < 0.5%
- [ ] `sar_adc_top.gds` flat 提取跑通（count 吻合 ±1%）

---

#### 组 CONN — Connectivity

##### T-M6.CONN.01 — Union-find 结构
- **Del**：`crates/mylvs-extract/src/unionfind.rs`（path compression + rank）
- **Unit**：10K op benchmark < 10ms

##### T-M6.CONN.02 — 导体层同层连通
- **Del**：`conn::same_layer_merge`，用 R-tree 查邻接 polygon
- **Unit**：metal1 上 L 形连通测试

##### T-M6.CONN.03 — Via/contact 跨层合并
- **Del**：tech 提供 contact 列表 (layerA, layerB, contact_layer) → 合并规则
- **Unit**：cc 连 m1/poly 测试

##### T-M6.CONN.04 — 输出 Net 结构
- **Del**：`conn::build_nets() -> Vec<Net>`，每个 net 带 polygon 列表 + layer 集合
- **Unit**：toy GDS 2 net 正确

##### T-M6.CONN.REG.01 — pmos_comp connectivity
- **Del**：对拍 Magic extract 的 net 数
- **Assert**：净数相等

---

#### 组 DEV — Device recognition

##### T-M6.DEV.01 — NMOS 识别
- **Del**：`extract/device/mos.rs::find_nmos`
- **Rule**：每个 `nmos_gate` polygon → 一个 NMOS
- **Pin 识别**：
  - gate = polygon 所在 poly 连通网
  - source/drain = gate 两侧 ndiff 连通网（按几何邻接判定）
  - body = psub 连通网（myadc 场景通常是 global vss）
- **Unit**：单 NMOS fixture → 1 NMOS + 4 pin 正确

##### T-M6.DEV.02 — PMOS 识别
- **Del**：`find_pmos`（对称逻辑）
- **Unit**：单 PMOS + nwell body

##### T-M6.DEV.03 — cap_mim 识别
- **Del**：`find_cap_mim`
- **Rule**：`capm` layer 与 m3/m4 层交叠
- **Unit**：cap_dac 的一个 cap 元单位 fixture

##### T-M6.DEV.04 — 器件 Pin 方向规范化
- **Del**：写死 NMOS=[D,G,S,B]，PMOS 同；MOS 内 S/D 对称，需输出时按几何 x 坐标排序
- **Unit**：生成 Device 后 pin 列表顺序稳定

---

#### 组 MEAS — 几何测量

##### T-M6.MEAS.01 — W / L 提取
- **Del**：`extract/measure/wl.rs`
- **Rule**：
  - L = gate polygon 在沿栅方向的宽度（poly 延伸方向）
  - W = gate 与 diff 交集在垂直方向的长度
- **Unit**：
  - 矩形 gate（W=1u L=0.15u）精确提取
  - 多段 gate → 分段累加 W（先不支持，报错）
- **DoD**：误差 < 0.5%

##### T-M6.MEAS.02 — 容值提取（mim_cap）
- **Del**：`extract/measure/cap.rs`
- **Rule**：C = overlap_area × cap_density（tech 给）
- **Unit**：1 unit cap 面积 → 值正确

##### T-M6.MEAS.REG.01 — pmos_comp W/L 回归
- **Del**：`tests/myadc_regression/wl_pmos_comp.sh`
- **Assert**：每个 MOS 的 W/L 对拍 Magic 提取值差 < 0.5%

---

#### 组 EXTOUT — SPICE 输出

##### T-M6.EXTOUT.01 — Netlist → SPICE（Magic 风格）
- **Del**：`extract/out/spice.rs::write_spice_magic_style`
- **Format**：与 `ext2spice lvs` 对齐（device 命名、参数格式、header 注释）
- **Unit**：一个 Device 的输出 line 对拍 Magic 输出

##### T-M6.EXTOUT.02 — CLI `mylvs extract`
- **Del**：`cmd/extract.rs`，跑 extract pipeline → stdout/文件
- **Unit**：集成测试跑 pmos_comp.gds → 输出文件和 Magic 对拍 device count

##### T-M6.EXTOUT.REG.01 — pmos_comp 完整对拍
- **Del**：`tests/myadc_regression/extract_pmos_comp.sh`
- **Flow**：`mylvs extract pmos_comp.gds` → `mylvs compare` 自家输出 vs Magic 输出
- **Assert**：自身对 Magic 提取 → PASS（即两者等价）

##### T-M6.EXTOUT.REG.02 — sar_adc_top flat extract
- **Del**：`tests/myadc_regression/extract_sar_adc_top.sh`
- **Assert**：device 数 吻合 Magic ±1%（允许小差异，因标准单元处理方式）

---

### §11 Phase M7 — Path II 合拢

**Phase DoD**：
- [ ] `mylvs run --gds sar_adc_top.gds --schematic sar_adc_top_combined.spice --tech sky130A` 结论 = Magic + netgen
- [ ] 至少 1 个 cell wall-clock 快于 Magic + netgen
- [ ] 版本 tag `v0.2-path2`

---

#### 组 RUN — 端到端

##### T-M7.RUN.01 — `mylvs run` 命令
- **Del**：`cmd/run.rs` 串联 extract → compare（内存直通，不落盘）
- **Unit**：集成测试跑 pmos_comp.gds + schematic

##### T-M7.RUN.02 — `--dump-extracted` flag
- **Del**：可选落盘用于调试
- **Unit**：路径存在则写入

##### T-M7.RUN.03 — Extract + compare 错误传播
- **Del**：extract 失败不走 compare；汇总到 report.warnings
- **Unit**：坏 GDS 情况下 exit 2

---

#### 组 XY — Mismatch 带坐标

##### T-M7.XY.01 — Extract 产物 device 带 gds_xy
- **Del**：`Device.source_loc_gds: Option<GdsXy>`
- **Unit**：extract 出来的每个 device 带坐标

##### T-M7.XY.02 — Mismatch report 带 gds_xy
- **Del**：JSON `layout_ref.gds_xy = [x, y]`（bbox 中心）
- **Unit**：negative 测试确认字段存在

##### T-M7.XY.REG.01 — sar_adc_top 端到端
- **Del**：`tests/myadc_regression/run_top.sh`
- **Assert**：mylvs run 结论 = Magic + netgen

##### T-M7.XY.REG.02 — 性能比较
- **Del**：`tests/myadc_regression/perf.sh` 跑 mylvs run vs Magic+netgen，记录 wall-clock
- **Assert**：pmos_comp 上 mylvs 不慢于 Magic+netgen

---

### §12 Phase M8 — 层次化 + 性能

**Phase DoD**：
- [ ] 层次化路径结论 = flat 路径
- [ ] 相对 M7 快 ≥ 2×
- [ ] 相对 Magic+netgen 快 ≥ 3×
- [ ] 版本 tag `v0.3-y1-end`

---

#### 组 HIER — 层次化

##### T-M8.HIER.01 — Per-cell extract 缓存
- **Del**：`extract/cache.rs`，按 `blake3(tech || gds_slice)` 做 key，`~/.cache/mylvs/` 存结果
- **Unit**：二次运行快 > 10×

##### T-M8.HIER.02 — Hierarchical compare
- **Del**：`compare/hier.rs`，subckt 边界先匹配，再递归进入
- **Unit**：2 层嵌套 schematic vs 展平 layout 仍能匹配
- **DoD**：结果与 flat compare 结论一致（回归）

##### T-M8.HIER.REG.01 — 层次 flat 一致性
- **Del**：`tests/myadc_regression/hier_vs_flat.sh`
- **Assert**：两种路径跑 sar_adc_top，JSON status 相同

---

#### 组 PAR — 并行

##### T-M8.PAR.01 — rayon per-cell extraction
- **Del**：`extract/parallel.rs`
- **Unit**：8 核下 1.5× 以上加速

##### T-M8.PAR.02 — rayon per-partition refinement
- **Del**：compare 阶段的 sig refine 按 class 并行
- **Unit**：≥ 1.3× 加速

---

#### 组 BENCH — Benchmark

##### T-M8.BENCH.01 — criterion 基线
- **Del**：`benches/myadc.rs`，5 cell 各一个 bench
- **DoD**：CI 存 baseline，回归检测

##### T-M8.BENCH.02 — vs Magic+netgen wall-clock 脚本
- **Del**：`tests/myadc_regression/perf_vs_golden.sh`
- **Assert**：每个 cell mylvs/golden_time 比值上报

##### T-M8.BENCH.03 — 内存 peak RSS
- **Del**：`benches/memory.sh` 用 `/usr/bin/time -v` 记录
- **Assert**：sar_adc_top < 2 GB

---

## Part III — 技术规范

### §13 项目结构

```
mylvs/
├── Cargo.toml                    # workspace
├── rust-toolchain.toml
├── crates/
│   ├── mylvs-core/               # IR (§15)
│   ├── mylvs-spice/              # SPICE parser (§16)
│   ├── mylvs-tech/               # Magic/netgen adapter (§18)
│   ├── mylvs-compare/            # Gemini + simplify (§17)
│   ├── mylvs-report/             # JSON + hypothesis
│   ├── mylvs-gds/                # GDS reader
│   ├── mylvs-geom/               # polygon boolean + R-tree + layer derivation
│   ├── mylvs-extract/            # connectivity + device recognition
│   └── mylvs-cli/                # binary
├── tests/
│   ├── myadc_regression/         # 每个 cell 一个 .sh + run_all.sh
│   ├── fixtures/
│   │   ├── synth/                # 手造小 netlist
│   │   ├── myadc/                # myadc 代表性文件
│   │   ├── myadc_negatives/      # 错配变种
│   │   ├── simplify/             # simplify pass 专用
│   │   ├── geom/                 # polygon 黄金
│   │   └── netgen_setup/         # setup.tcl 解析 fixture
│   ├── snapshots/                # insta
│   └── proptest-regressions/     # proptest 种子
├── benches/                      # criterion
├── docs/
│   ├── learning-roadmap.md
│   ├── development-plan.md
│   ├── json-schema.md
│   ├── ai-agent-guide.md         # M4 发布
│   └── polygon-lib-eval.md       # M5 写
├── .github/workflows/
│   ├── ci.yml
│   └── nightly.yml
├── BUGS.md
├── README.md
└── CLAUDE.md
```

### §14 Crate 选型

| 用途 | Crate |
|---|---|
| parser combinator | `nom` |
| CLI | `clap` (derive) |
| error | `thiserror` / `anyhow` |
| serde | `serde` / `serde_json` |
| schema | `schemars` |
| hash map | `hashbrown` + `ahash`（固定 seed） |
| small vec | `smallvec` |
| parallel | `rayon` |
| snapshot test | `insta` |
| property test | `proptest` |
| benchmark | `criterion` |
| logging | `tracing` / `tracing-subscriber` |
| GDS | fork `gds21` |
| polygon boolean | 评估 `i-overlay` |
| R-tree | `rstar` |
| hash | `blake3` |

### §15 核心 IR（M1 定稿）

（与之前一致，略；完整 Rust 定义见上一版 §15，此处保持不变）

```rust
pub struct DeviceId(pub u32);
pub struct NetId(pub u32);
pub struct CellId(pub u32);
pub struct ModelId(pub u32);

pub struct Netlist { cells: Vec<Cell>, ... }
pub struct Cell   { ports: Vec<NetId>, devices: Vec<Device>, nets: Vec<Net>, ... }
pub struct Device { kind: DeviceKind, pins: SmallVec<[NetId; 4]>, params: DeviceParams, source: Option<SourceLoc> }
pub enum DeviceKind { Mosfet { model, flavor }, Resistor { model }, ... , SubcktInstance { target: CellId } }
pub struct DeviceParams { w, l, m, nf, value: Option<f64>, extras: BTreeMap<String, ParamValue> }
pub enum ParamValue { Number(f64), Expr(String), Str(String) }
pub struct Net { name: String, is_port: bool, is_ground: bool, pins: SmallVec<[(DeviceId, u8); 4]> }
pub struct Model { name: String, kind: ModelKind, body: Arc<str> }
pub struct SourceLoc { file: Arc<str>, line: u32, col: u32 }
```

### §16 SPICE 支持范围

**支持**：`.subckt/.ends`, `.include`, `.lib NAME SECTION`, `.param`, `.model`（body 原文）, `.global`, `.option`, `.end`；M/R/C/L/V/I/D/Q/X 设备；数值 SI 后缀 (k/meg/g/u/n/p/f/mil/a/t/m)；表达式 `{}` / `''` 存原文不求值；`*` 注释、`$` inline 注释、`+` 续行、case-insensitive。

**不支持**（warn 不 fatal）：`.tran/.dc/.ac/.op/.noise/.print/.save/.plot/.step/.mc`、`B` 行为源、`T/W` 传输线。
**不支持**（fatal）：`.if/.elseif/.else/.endif`、`.func`。

**SKY130 特化**：X device 下沉（M4）按 §18.2 映射；Magic extract net 改名规则集 `sky130a-flat`（§18.4）。

### §17 Gemini 算法细节

（保持之前的伪代码规格，略——关键点：初始 sig hash(kind, degree, sorted(pin_roles))；refine step hash(old_sig, sorted(neighbor_sigs))；稳定后 partition；unique class 直绑；ambiguous → automorphism 检测 → swap group 或 backtrack 搜索。）

### §18 SKY130 tech adapter

**§18.1** 配置源：`~/sky130A/libs.tech/magic/sky130A.tech24` + `~/sky130A/libs.tech/netgen/sky130A_setup.tcl`
**§18.2** 硬编码 SKY130 subckt → DeviceKind 映射表（`sky130_fd_pr__nfet_01v8` 等，见 T-M4.TECH.06）
**§18.3** 支持的 netgen 指令：`permute`, `property`, `equate classes`, `ignore class`, `ignore pin`
**§18.4** Magic postproc 规则集 `sky130a-flat`：VPB/VNB/`w_N_N#`/SUB net 改名 + 实例前缀 strip

### §19 输出接口规范（CLI + JSON + exit code）

**CLI**：
```
mylvs parse   FILE                                      [--format] [--out]
mylvs compare --layout L --schematic S --tech T         [--format] [--out]
              [--flatten-layout] [--flatten-schematic]
              [--simplify none|basic|aggressive]
              [--w-tol F] [--l-tol F]
              [--magic-postproc RULE]
              [--netgen-setup FILE]
mylvs extract --gds G --tech T --cell NAME              [--out] [--dump-layers]
mylvs run     --gds G --schematic S --tech T            [--out] [--dump-extracted]
```

**Exit code**：0 PASS / 1 FAIL / 2 输入错 / 3 内部错 / 4 资源限

**Stderr** = 日志；**Stdout** = 结构化结果。严格分离。

**JSON schema v1.0**（冻结于 M4 Phase end）：见上一版完整规格——核心字段 `schema_version` / `status` / `summary` / `mismatches[]` / `warnings[]` / `stats`，mismatches 按 id (blake3 hash) 排序确保确定性。

---

## Part IV — 项目纪律

### §20 测试策略汇总

| 层 | 技术 | 主要位置 | 运行频率 |
|---|---|---|---|
| Unit | `#[test]` | 每 crate `src/**/*.rs::tests` | 每次 push |
| Integration | `tests/*.rs` | `tests/` 顶层 | 每次 push |
| Snapshot | `insta` | `tests/snapshots/` | 每次 push |
| Property | `proptest` | `tests/proptest_*.rs` | 每次 push（release 加大 case 数） |
| myadc regression | shell | `tests/myadc_regression/*.sh` | nightly + 手工 |
| Fuzz（geom） | `proptest` vs KLayout subprocess | `tests/geom_fuzz.rs` | nightly |
| Bench | `criterion` | `benches/*.rs` | 每周 + release |

### §21 CI 流水线

```yaml
# .github/workflows/ci.yml
jobs:
  quick:
    steps:
      - cargo fmt --check
      - cargo clippy --workspace -- -D warnings
      - cargo test --workspace
  
# .github/workflows/nightly.yml
jobs:
  full:
    steps:
      - cargo test --workspace --release (proptest cases x10)
      - bash tests/myadc_regression/run_all.sh
      - bash tests/myadc_regression/perf_vs_golden.sh
```

### §22 BUGS.md 模板

```markdown
# MyLVS Bug Tracking

## Open

### BUG-001: <一句话标题>
- **Discovered**: 2026-MM-DD
- **Phase**: M<n>
- **Context**: 跑 myadc 的 <cell>
- **Symptom**: mylvs <现象>；期望 <现象>
- **Reproducer**: `mylvs compare --layout ... --schematic ...`
- **Root cause (if known)**: ...
- **Blocks**: Phase M<n> DoD / myadc <cell>
- **Priority**: P0 / P1 / P2

## Fixed

### BUG-001: ... (commit abc1234, 2026-MM-DD)
- 根因 + 修复 + 回归路径（哪个 test 覆盖）
```

**准入**：任何 myadc 跑不通或结论异常，先入 Open。修复后必须附回归 test 路径。

### §23 风险登记

| # | 风险 | 缓解 | 监测信号 |
|---|---|---|---|
| R1 | Gemini backtrack 爆炸 | 深度限 + fallback ambiguous | `stats.backtrack_hits` > 阈值 |
| R2 | Polygon boolean 退化 | `i-overlay` + fuzz vs KLayout | 10k fuzz 失败 > 0.1% |
| R3 | 标准单元复杂度 | M6 允许先 flatten | sar_logic extract 失败 |
| R4 | netgen 指令未覆盖 | 遇一个入 BUGS.md | `warnings[]` 含 `netgen_unsupported_*` |
| R5 | Magic postproc 约定变 | TOML 配置化 | 某 cell 突然 topology mismatch |
| R6 | Schema 漂移 | M4 起冻结 v1.0 | schema_version minor bump 频率 |
| R7 | 标准单元 W/L 提取 | M6 只保矩形 gate | extract 报错 "non-rectangular gate" |

### §24 学习前置映射

| Phase | 前置学习阶段（见 `learning-roadmap.md`） |
|---|---|
| M1 | 阶段 1（SPICE）、阶段 6（Rust 工程化基础） |
| M2 | 阶段 4 前半（Gemini 基本） |
| M3 | 阶段 4 后半（对称、预简化） |
| M4 | 阶段 5 初探（层次化概念） |
| M5 | 阶段 2（GDS）、阶段 3 前半（polygon boolean） |
| M6 | 阶段 3 后半（extraction） |
| M7 | — |
| M8 | 阶段 5 深入 + 阶段 6 并行 |

---

## Part V — Kickoff

### §25 M1 第一批可并行启动的任务
- T-M1.SETUP.01 / 02 / 03 / 04 / 05（一天搞完）
- T-M1.CLI.01 / 02 / 03 / 04（workspace 就绪后）
- T-M1.IR.01 / 02 / 03 / 04 / 05 / 06 / 07（与 SP 并行）
- T-M1.FIX.01 / 02（随时）

SP 组内部有依赖链，SP.01 → 其他 SP.*。

### §26 Re-plan 节奏

每个 Phase DoD 勾完后：
- 更新本文档 `§27 Revision log`（添加条目）
- 复盘：已发生 bug 反映的前置学习漏洞 / 任务拆分粒度问题
- 下一 Phase 的任务清单按实际情况增删

### §27 Revision log

- **v2.0 · 2026-04-18**：任务级重写，去除时间分配，每任务含明确 DoD + unit + regression
- **v1.0 · 2026-04-18**：初版（按月 / 周）
