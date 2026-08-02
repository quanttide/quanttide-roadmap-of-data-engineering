# 数据云架构设计

> 更新日期：2026-07-23 | 来源：7月22日数据业务部门全员会议

---

## 四层框架

```
需求层 Requirement（面向客户，动词 clarify）
    ↓
规格层 Specification（面向技术，动词 design）
    ├── Contract（数据契约：输入输出规格）
    └── Blueprint（处理蓝图：工作流步骤）
    ↓
实现层 Implementation（动词 implement）
    ├── Catalog（数据目录：运行时文件注册）
    └── Pipeline（数据管道：可执行处理流程）
    ↓
任务层 Task（动词 execute）
    ├── Feature（产出）
    └── Observation（观测）
```

完整流程链：

```
Context → clarify → Requirements (DRD) → design → Specification (Contract + Blueprint)
    → implement → Catalog + Pipeline
    → execute → Task
    → report → transfer → Delivery
```

## 能力入口（Skill）

CLI 作为能力入口，组织为领域模型：

```
Skill（能力入口）
├── Blueprint（做什么）
├── Contract（输入输出规范）
├── Pipeline（怎么执行）
└── Catalog（注册和发现）
```

领域模型命名示例：`catalog discover`、`contract validate`、`blueprint design`。

## 命令映射（旧→新）

| 新命令 | 旧命令 | 层级 | 说明 |
|--------|--------|------|------|
| `clarify` | — | Requirement | 从客户上下文澄清需求，生成 DRD |
| `design` | `design` + `formalize` | Specification | 从 DRD 设计规格书（Contract + Blueprint） |
| `implement` | — | Implementation | 从 Specification 生成 Catalog + Pipeline |
| `preview` | `preview` | 跨层 | 预览 Specification / DRD |
| `report` | — | Delivery | 对客户汇报 |
| `transfer` | `transfer` | Delivery | 数据传输 |
| `version` | `version` | 跨层 | 版本管理 |
| `review` | `review` | 跨层 | 审计 Requirements 或 Specification |

---

# Blueprint 平台核心需求（历史参考）

> 来源：GHTorrent 脱敏交付复盘 | 日期：2026-07-17
> 注意：此部分为旧框架下的设计，已由上方四层框架替代。保留作为设计演进记录。

---

## 需求 1：Blueprint 显式建模

### 问题

当前 Blueprint 以三份独立文件（.md / .cue / .html）存在，文件之间没有显式的关联和约束。.cue 定义了类型，但 .md 和 .html 不一定与之同步。

### 目标

将 Blueprint 建模为平台的一等公民（first-class entity）：

- Blueprint 有唯一标识、元数据（负责人/仓库/状态/时间线）
- .cue 是唯一事实源，.md 和 .html 由平台从 .cue 自动生成
- Blueprint 中的类型定义（`#Blueprint`, `#Pipeline`, `#Step`, `#Contract` 等）是平台理解的"语言"

### 技术方向

- CUE 解析器：读取 `.cue` 中的 `#Blueprint` 实例
- 从 CUE 类型生成 Markdown 和 HTML 的渲染器
- Blueprint 的增删改查 API

---

## 需求 2：Blueprint Version 版本管理

### 问题

项目迭代（如变量拆分、管道重构）会产生多个版本的 Blueprint，当前版本信息仅存在于 .cue 文件的注释和 `updated_at` 字段中，没有结构化的版本历史。

### 目标

- 每个 Blueprint 可以有多个版本（`v1`, `v2`, ...）
- 每个版本记录：状态、时间线、变更说明
- 版本之间可以追溯

### 技术方向

- 在 `#Blueprint` 类型中显式建模 `version` 字段
- 版本号、发布时间、changelog 的结构化存储
- Blueprint 版本列表和版本切换的 API

---

## 需求 3：版本差异比较（Diff）

### 问题

当 Blueprint 从 V2 升级到 V3，需要知道"哪些变量拆分了、哪些步骤新增了、输出规格如何变化"。目前只能通过人工对比 .html 页面来发现差异。

### 目标

- 平台可以比较两个 Blueprint 版本的差异
- Diff 结果结构化：变量级、步骤级、契约级
- 差异可视化（新增/删除/修改），参考现有 .html 并排对比的交互模式

### 技术方向

- CUE 实例的结构化 diff 算法（字段级比较）
- 差异类型分类：新增变量、废弃变量、口径变更、步骤拆分/合并
- Diff 结果的 HTML/JSON 输出

---

## 三者关系

```
Blueprint 显式建模
    └→ 使得 Blueprint 可以被平台操作（而非手动编辑文件）
         └→ 版本管理成为可能
              └→ 版本 Diff 成为可能
                   └→ 版本迭代可追溯、可审查
```

显式建模是地基，版本管理是第一层，diff 是第二层。三者按顺序建设。

---

## Blueprint 命令集

五命令覆盖两条路径——新项目从零创建 + 老项目整理入库：

```
review → design → formalize → preview → version
  ↑                                                    │
  └──────────── 循环迭代，每次 review 更少问题 ──────────┘
```

| 命令 | 操作对象 | 功能 |
|------|----------|------|
| `review` | 已有 Blueprint | 审计现有 Blueprint，输出问题清单。老项目进入系统的入口 |
| `design` | `.md` | 维护人类可读的 Blueprint 文档 |
| `formalize` | `.cue` | 从 Markdown 形式化为 CUE 结构化定义 |
| `preview` | `.html` | 从 CUE 生成可视化页面 |
| `version` | 版本元数据 | 维护版本历史与变更记录 |

## Feature 愿景

**数据蓝图治理工具**：把零零散散、各个时期、各种版本的 Blueprint，逐渐规范到统一格式，让新人通过工具快速吸收历史海量项目经验。

- v0.1：五命令基础流程，能搞定几个项目（如 GHTorrent）
- v0.5+：完整治理能力，批量整理、跨项目一致


---

# 设计与实现不一致分析（2026-08-02 实测确认）

四层框架（本文档上半部分）是**设计**，src/cli 与 quanttide-data crate 是**实现**。
逐项对比发现三处不一致，均经实际运行验证（`spec wrap` / `spec validate` 实测）。

## 不一致 1：规格层结构——Contract 应平级，实现却嵌套

### 设计（四层框架）

```
规格层 Specification
    ├── Contract（数据契约：输入输出规格）
    └── Blueprint（处理蓝图：工作流步骤）
```

Contract 与 Blueprint **平级**。

### 实现（当前）

```rust
pub struct SpecificationBody { pub blueprint: quanttide_data::Blueprint }   // 只有 blueprint
// Blueprint 内含 contract 字段（quanttide-data crate 模型）
```

contract 被埋在 blueprint 内，envelope 没有独立的 contract 块。

### 后果（实测）

- `design contract` 产出顶层 `contract:`（平级直觉）
- `design blueprint` 产出 blueprint 内**空 contract**（`schema: ""`）
- 两产物形状不同 → `spec wrap` 前必须 merge（临时脚本 `merge_contract.py` 就是这个裂缝的补丁，示例已删）
- `spec validate` 报 `contract.input.schema must not be empty`（空模板被校验捕获）

## 不一致 2：pipeline 归属——应独立实现层，实现却嵌在 Blueprint

### 设计（四层框架）

```
规格层：Blueprint（工作流步骤）     ← 流程规格
实现层：Pipeline（可执行处理流程）   ← 流程实现
```

Blueprint 与 Pipeline **不同层**，应为投影关系（`blueprint.steps →(implement)→ pipeline.states`），非嵌套。

### 实现（当前）

```rust
// quanttide-data crate
pub struct Blueprint { ..., pub pipeline: Pipeline, ... }   // 跨层嵌套
pub struct Pipeline { pub name: String, pub steps: Vec<Step> }  // 列表模型
```

实现层的 Pipeline 被塞进规格层的 Blueprint；且 pipeline 建模为**列表**（隐式顺序），非**状态机**。

### 后果（实测）

- `design blueprint` 产出 Step Functions 风格 `start_at`/`states`（v0.2.0 特意加的规格）
- `spec wrap` 用 toolkit 模型反序列化 → **start_at/states 被 serde 静默丢弃**（模型无此字段），wrap 后只剩 `name + steps`
- 实测：wrap 前后 pipeline keys 从 `[name, start_at, states, steps]` 变为 `[name, steps]`——**"稳定化"操作丢规格，无任何错误提示**
- 分支（促销日/非促销日）无法在列表中表达

## 不一致 3：运行态混入设计态

### 设计

蓝图是设计态定义（步骤/契约/依赖）。

### 实现

```rust
pub struct Blueprint { ..., pub status: Status, pub timeline: Option<Vec<TimelineEntry>>, ... }
```

`status`（draft/submitted/...）、`timeline`（历史）是**实例运行态**，混入蓝图（设计定义）。cloud/deliverables（交付方案）也混入。

---

# 重构方案：规格层三分平级（Contract / Blueprint / Pipeline）

## 目标模型（四层框架代码化）

```yaml
Specification:
  api_version: qtcloud.quanttide.com/v1alpha1
  kind: Specification
  metadata: { name, generated_by, source_path }
  spec:
    contract:            # ← 数据契约（与 blueprint/pipeline 平级）
      input: { schema, format }
      output: { schema, format, rules }
    blueprint:           # ← 处理蓝图：工作流步骤 = 流程定义（有自己的定义方式）
      name
      steps: [ { name, from, to, desc, depends? } ]
    pipeline:            # ← 可执行管道：状态机 = blueprint 流程的投影（实现层）
      start_at
      states: { name: { type, from, to, desc, next|end, resource, condition? } }
```

**关系**：`blueprint.steps`（流程规格，设计态）→(implement)→ `pipeline.states`（可执行流程，实现态）。

## 各层职责

| 块 | 定义方式 | 回答的问题 | 产出命令 |
|----|---------|-----------|---------|
| contract | input/output 规格 | 数据长什么样 | `design contract` |
| blueprint | steps（工作流步骤 + depends） | 流程**做什么**（数据流语义） | `design blueprint` |
| pipeline | states（状态机：start/next/end/分支） | 流程**怎么跑**（控制流编排） | `design blueprint` 投影 / `implement` |

## 命令调整

| 命令 | 现状 | 重构后 |
|------|------|--------|
| `design contract` | 产出顶层 `contract:` | 产出 spec 的 contract 片段（形状不变） |
| `design blueprint` | 产出 blueprint（含空 contract + start_at/states 混合） | 产出 `blueprint.steps`（内容步骤）+ 投影 `pipeline.states`（状态机） |
| `spec wrap` | 输入 blueprint，merge 后组合 | **组合** contract + blueprint + pipeline 三片段 → envelope（merge 补丁退役） |
| `spec validate` | 校验 envelope 结构 | 校验三块完整性 + 投影一致性（steps ↔ states 对应） |
| `implement` | 从 blueprint 生成代码 | 从 blueprint.steps 生成代码（pipeline 已是状态机，无需再解释） |
| `process` | run_pipeline 解析 steps | 消费 pipeline.states（状态机执行，支持分支） |

## 兼容与迁移（避免 break change）

1. **读取兼容**：旧 `blueprint.contract` / `blueprint.pipeline` 反序列化时可选（`#[serde(default)]`），读取后迁移为新结构
2. **旧名兼容**：spec 域公开 API 变更随 v0.3.0（deprecated 别名，参考 storage 改名经验）
3. **迁移顺序**：
   - ① toolkit `quanttide-data` crate 模型重构（单一事实源先改）
   - ② CLI spec/design/implement/process 消费新模型
   - ③ 合并脚本退役、示例重建（无 merge 补丁）
   - ④ 契约测试（跨语言 monorepo 校验同一模型）

## 依赖关系

本重构依赖 toolkit 模型先落地（见 `../quanttide-data-toolkit/index.md` 的模型重构章节）——CLI 与 toolkit 的分裂根因是共享模型错误，先修共享模型。
