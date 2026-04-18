# MyLVS 开发前置学习路线

> 目标：从零开始，系统掌握开发一个 LVS 工具所需的全部知识，并知道"在哪学、怎么学、学到什么程度"。

---

## 一、全景：要做 LVS，需要哪些知识栈

LVS 工具横跨 **半导体/IC 设计**、**几何算法**、**图算法**、**文件格式解析**、**系统工程** 五个领域。

| 层 | 内容 | 在 LVS 中的角色 |
|---|---|---|
| 领域知识 | IC 设计流程、器件原理、PDK | 明白 LVS 在验证什么 |
| 输入格式 | GDSII / OASIS、SPICE / CDL、Verilog netlist | 读得懂用户的文件 |
| 几何算法 | 多边形布尔运算、R-tree、扫描线 | Layout extraction 的地基 |
| 器件识别 | Layer derivation、connectivity extraction | 把几何变成电路 |
| 图算法 | Graph isomorphism（Gemini / Nauty / partition refinement） | netlist 比较的核心 |
| 系统工程 | 大规模数据结构、并行、内存布局 | 让工具跑得动工业级设计 |
| Rust 实战 | arena、index-based graph、nom / pest、rayon | 实现手段 |

---

## 二、分阶段学习路径

### 阶段 0：IC 设计与 LVS 的"为什么"（1–2 周）

**目标**：能用一段话说清楚"LVS 在设计流程的哪一步、为什么需要、没它会出什么问题"。

**核心概念**
- 芯片流程：RTL → Gate-level netlist → Schematic → Layout → GDSII → tape-out
- MOSFET（NMOS/PMOS）的 W/L 参数的物理含义
- Schematic（原理图）与 Layout（版图）的对应关系
- 物理验证三件套：**DRC**（几何规则）、**LVS**（版图 vs 原理图等价性）、**ERC**（电学规则）
- PDK（Process Design Kit）是什么，里面有 layer 定义、device model、LVS/DRC rule deck

**动手**
- 装 KLayout + Netgen + open_pdks (SKY130)
- 跑一遍 SKY130 里一个 inverter 的 LVS，把报告从头到尾看懂

**资料**
- 《CMOS VLSI Design》Weste & Harris（第 1–3 章）
- Coursera：Rob Rutenbar "VLSI CAD Part I: Logic"（UIUC，免费）
- SKY130 PDK 官方文档与 open_pdks README

---

### 阶段 1：Netlist 与 SPICE（约 1 周）

**目标**：徒手写出一个带层次的 SPICE netlist，并能用 Rust 解析它。

**核心概念**
- SPICE netlist 语法：`M1 d g s b nmos W=... L=...`、`X1 ... subckt_name`
- `.subckt` / `.ends` 层次
- CDL（Cadence 的裁剪版 SPICE）与纯 SPICE 的差异
- Verilog structural netlist 的 instance / port 概念（可选）
- 把 netlist 存成**二部图**（器件节点 + 电气节点）的数据结构

**动手**
1. 用 `nom` 或 `pest` 写一个能处理 subckt 层次的 mini SPICE parser
2. 把结果存成 `Vec<Device>` + `Vec<Net>` + 二部邻接表

**资料**
- ngspice user manual 第 2–3 章（netlist 语法最权威）
- Rust `nom` tutorial / `pest` book

---

### 阶段 2：Layout 与 GDSII（1–2 周）

> 如果你最终决定只做 netlist-vs-netlist，本阶段可跳过。但建议至少读到能看懂 GDS 结构。

**目标**：从零写一个 minimal GDSII reader，用 KLayout 能打开你生成的 GDS。

**核心概念**
- GDSII 是二进制 record 流：record = header + data，每种 record 有固定 tag
- 层次：Library → Structure (cell) → Element (BOUNDARY / PATH / SREF / AREF / BOX)
- Layer/datatype：PDK 决定哪层是 poly、metal1、contact……
- 坐标变换：SREF 的 translate / rotate / mirror / magnification
- OASIS：GDS 的现代替代，压缩率和结构都更好，二期再看

**动手**
1. 对照 spec 读一个小 GDS 的 hex dump
2. 用 Rust 写只支持 BOUNDARY + SREF 的 reader/writer
3. 对比开源实现：`gds21` crate（Rust）、gdstk（C++/Python）、klayout 源码

**资料**
- GDSII 原始 spec（Calma 1988，网上可搜到 PDF）
- `gds21` Rust crate 源码（代码短，适合对照学习）
- KLayout 源码 `src/db/` 目录

---

### 阶段 3：几何算法与 Layout Extraction（3–4 周，最难的一段）

**目标**：把 polygon + layer 数据转换成"器件 + 连接"清单。

**核心概念**
1. **Polygon Boolean operations**：AND / OR / NOT / XOR / SIZE(grow, shrink)
   - 经典算法：Vatti、Weiler–Atherton
   - 现代主流：扫描线 + 区间树
2. **Spatial indexing**：R-tree / quad-tree（大版图必须）
3. **Layer derivation**（逻辑层推导）
   - 例：`nmos_gate = poly AND ndiff`，`nmos_sd = ndiff NOT poly`
   - PDK 在 LVS rule deck 里写这些规则，工具要能脚本化执行
4. **Connectivity extraction**
   - 导体层（metal / poly / diff）内用 union-find 做连通区域 → 同一 net
   - via / contact 把相邻层的 net 合并
5. **Device recognition**
   - 找出每个 MOSFET 的 gate/source/drain/bulk 各自连到哪个 net
   - 提取 W（gate 沿宽度方向）和 L（沿沟道方向）——几何测量

**动手**
1. 实现一个简单的扫描线 polygon union
2. 实现 NMOS 识别：输入 poly + ndiff，输出带 W/L 的 MOSFET 列表
3. 对照学习：Magic 的 `extract` 模块、KLayout 的 LVS framework 源码

**资料**
- **必读**：Magic Tutorial #8 *"Circuit Extraction"* (Scott)
- Hon & Sequin *"A Guide to LSI Implementation"*（有 extraction 章节）
- 《Algorithms for VLSI Physical Design Automation》Sherwani，第 2–3 章
- 《Computational Geometry: Algorithms and Applications》de Berg et al.，第 2、17 章
- KLayout 官方 LVS tutorial

---

### 阶段 4：Netlist 比较 = 图同构（2–3 周，算法核心）

**目标**：理解为什么 netlist 比较本质是图同构问题，并实现一个能用的比较器。

**核心概念**
1. **建模**：netlist = 二部图。两个 netlist 等价 ⇔ 两图同构 + 对应器件参数匹配
2. **Partition refinement / 签名迭代（Ebeling 的 Gemini 算法）**
   - 每个 node 算初始 signature（类型、度数、邻居类型多重集）
   - 反复用邻居 signature 更新自己，直到稳定
   - 稳定后 signature 相同归一组；每组两侧各一个 → 匹配成功
   - 组内多于一个 → case splitting / backtrack
3. **对称处理**：差分对两根线可 swap——Gemini 用 automorphism classes 处理
4. **Pre-simplification**：串并联合并、并联管子 merge——先简化再比
5. **更通用算法**：McKay 的 nauty、bliss（canonical labeling）、Weisfeiler–Lehman

**动手**
1. 精读 Netgen 源码（~2 万行 C/Tcl，核心在 `netcmp.c`）
2. 用 Rust 实现 Gemini 式 partition refinement，先跑通小例子
3. 故意改错一个 netlist（交换 NMOS 连接 / 改 W 值），看工具能否定位到 device/net

**资料**
- **必读**：Ebeling & Zajicek 1983 *"Validating VLSI circuit layout by wirelist comparison"* (ICCAD)
- Ebeling 1988 *"GeminiII: A Second Generation Layout Validation Program"*
- Spickelmier & Newton 1983 *"WOMBAT: a new netlist comparator"*
- McKay *"Practical graph isomorphism"*（nauty 理论基础）
- Netgen 项目文档（Tim Edwards 维护）

---

### 阶段 5：层次化 LVS 与参数匹配（约 2 周）

**目标**：理解为什么工业级 LVS 必须 hierarchical，以及参数比较的容差。

**核心概念**
- Hierarchical matching：在 cell 边界匹配、递归进入 sub-cell——比 flat 快几个数量级
- Cell boundary detection 与 pin-based matching
- 参数匹配：W/L 容差、multiplier (M) 合并、fingered transistor (NF) 处理
- Mismatch reporting：必须告诉用户"哪两个 net 不对应、哪个 device 参数差多少"

**资料**
- Calibre LVS user manual（能搞到的话，业界事实标准）
- KLayout LVS 源码 `src/db/db/dbLayoutVsSchematic.cc`

---

### 阶段 6：Rust 工程化（贯穿全程）

**目标**：让工具能跑到百万器件级而不 OOM、不 panic。

**关键技巧**
- **Arena + index**：不要 `Arc<Node>`，用 `Vec<Node>` + `NodeId(u32)`。EDA 图极深，引用计数会崩
- **Zero-copy parsing**：大 GDS/SPICE 用 `&[u8]` + `nom`，避免 `String` 拷贝
- **Parallelism**：`rayon` 做 per-cell extraction 很自然
- **序列化中间结果**：`bincode` / `rkyv` 存 extraction 结果，调试比较器不用每次重跑
- **Testing**：snapshot test（`insta`）+ property-based test（`proptest`）

**资料**
- 《Programming Rust》第二版
- `petgraph` crate 源码（index-based graph 的标准写法）
- Jon Gjengset《Rust for Rustaceans》

---

## 三、必学的开源工具（从它们身上偷想法）

| 工具 | 为什么看 | 深度 |
|---|---|---|
| **Netgen** | 纯 netlist 比较的标杆，算法干净 | 精读源码 |
| **Magic** | 最成熟的 extraction，有配套教程 | 读 extract 模块 |
| **KLayout** | 现代 C++ 实现，完整 LVS framework | 参考架构 |
| **SKY130 PDK** | 免费真实 PDK，能端到端跑 LVS | 当测试数据源 |
| **open_pdks** | SKY130 + gf180 rule deck 集合 | 看 rule deck 什么样 |
| **gds21 / gdstk** | GDS 读写参考实现 | 写自己 parser 前对照 |

---

## 四、推荐节奏（每周 10–15 小时投入）

| 周 | 阶段 | 里程碑 |
|---|---|---|
| 1–2 | 阶段 0 + 1 | 能跑通 SKY130 inverter LVS，能解析 SPICE |
| 3–5 | 阶段 2 | GDS reader 能 round-trip |
| 6–9 | 阶段 3 | NMOS 提取 demo（几何 → 器件）|
| 10–12 | 阶段 4 | flat netlist 比较器 demo |
| 13–14 | 阶段 5 初探 + 阶段 6 重构 | 跑通一个最小 end-to-end LVS |

三个月做出一个 **flat、只支持 SKY130 几个基本器件的 LVS prototype** 是合理目标。别一开始就想着对标 Calibre。

---

## 五、Checkpoint：每阶段结束自测

能无障碍回答以下问题，该阶段才算学扎实：

- 阶段 0：为什么 DRC 通过不保证 LVS 通过？
- 阶段 1：SPICE `.subckt` 和 Verilog module 在数据结构上本质差异是什么？
- 阶段 2：GDS 里 AREF 相比展开成多个 SREF 有什么好处？
- 阶段 3：为什么 W 要用 gate ∩ diffusion 测量，不是 gate 自己的长度？
- 阶段 4：partition refinement 遇到差分对这类对称结构为什么会 ambiguous？怎么破？
- 阶段 5：hierarchical LVS 什么时候必须 flatten？

---

## 六、附录：关键术语速查

| 术语 | 含义 |
|---|---|
| **PDK** | Process Design Kit，工艺厂给的设计文件包，含 layer、model、rule |
| **DRC** | Design Rule Check，几何规则检查 |
| **LVS** | Layout Versus Schematic，版图 vs 原理图等价性检查 |
| **ERC** | Electrical Rule Check，电学规则（浮空 gate、短路等）|
| **Rule deck** | LVS/DRC 规则脚本，定义 layer 运算和器件识别规则 |
| **Extraction** | 从几何版图提取出器件和连接关系的过程 |
| **Flattening** | 展平层次，把 subcircuit instance 全部 inline |
| **Graph isomorphism** | 图同构，LVS 比较的数学本质 |
