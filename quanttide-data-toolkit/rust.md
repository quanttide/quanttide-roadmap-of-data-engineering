# Rust 包模型设计（packages/rust/，quanttide-data crate）

> 更新日期：2026-08-02 | 对应子包：`packages/rust/`（crate `quanttide-data` v0.1.0，CLI 已依赖）

---

# 一、模型现状

`packages/rust/` = crate `quanttide-data`（v0.1.0，已发布 crates.io）——qtcloud-data CLI 已依赖（`Cargo.toml: quanttide-data = "0.1.0"`）。

### 已有模型

| 类型 | 说明 |
|------|------|
| `Blueprint` | 顶层实体：metadata / requirements / pipeline / cloud / deliverables / status |
| `Pipeline` / `Step` | 有序步骤列表 / 步骤（name/from/to/desc/depends） |
| `Contract` / `PanelSpec` / `ColumnDef` | 输入输出结构约束 / 输出规格 / 列定义 |
| `SourceTable` / `UserFilter` | 数据源表 / 用户过滤 |
| `CloudServer` / `CloudPlan` / `ChunkedUpload` | 上云方案 |
| `Deliverable` | 交付物描述 |
| `Status` / `TimelineEntry` / `TimelineAction` | 状态枚举（draft/submitted/confirmed/rejected）/ 时间线 |

序列化：CUE ↔ Rust struct ↔ JSON/YAML；校验：`validate`。

---

# 二、模型与四层框架不一致分析（实测确认）

`packages/rust/` 的模型是 CLI 与平台共享的**单一事实源**——它错了，CLI 只能打补丁。
实测确认（`spec wrap` 静默丢字段）后，逐项列出不一致。

## 不一致 1：Blueprint 聚合了不该聚合的（跨层 + 运行态混入）

```rust
pub struct Blueprint {
    pub name, description,
    pub contract: ContractPair,   // ← 数据契约（框架中与 blueprint 平级）
    pub pipeline: Pipeline,       // ← 可执行管道（框架中在实现层，独立）
    pub cloud: Option<CloudPlan>,   // ← 上云方案（交付实现）
    pub deliverables: Option<Deliverables>,  // ← 交付物（交付实现）
    pub status: Status,           // ← 运行态（实例状态，非设计态）
    pub timeline: Option<Vec<TimelineEntry>>, // ← 运行态（历史）
    pub created_at, updated_at,
}
```

三个问题：
1. **contract 嵌套**：框架中 Contract 与 Blueprint 平级，模型做成子字段
2. **pipeline 跨层**：框架中 Pipeline 在实现层，模型塞进规格层 Blueprint
3. **运行态混入设计态**：status/timeline 是实例状态（draft/submitted/执行历史），不属于蓝图（设计定义）；cloud/deliverables 是交付方案

## 不一致 2：Pipeline 建模为列表，而非状态机

```rust
pub struct Pipeline { pub name: String, pub steps: Vec<Step> }   // 列表：隐式顺序
```

1. **无法表达分支/条件**（案例：促销日 vs 非促销日走不同流程）
2. **与 CLI 产出不一致**：CLI `design blueprint` 产出 Step Functions 风格 `start_at`/`states`——模型无此字段 → `spec wrap` 时 **serde 静默丢弃**（实测：wrap 后 start_at/states 消失，无错误）
3. 列表 + `depends` 双语义（顺序 + 显式依赖）重叠，语义不清

## 不一致 3：CLI 与 toolkit 各说各话（单一事实源失效）

| 模型 | pipeline 形状 | contract 位置 |
|------|--------------|--------------|
| CLI design 产出 | start_at/states/steps（状态机） | 顶层 / blueprint 空 |
| toolkit 模型 | name/steps（列表） | blueprint 内 |
| CLI spec 消费 | toolkit 模型 | blueprint 内 |

产出（状态机）→ 共享模型（列表）→ 消费（列表）：**产出到消费之间信息丢失**。这正是"知识即代码、单一事实源"原则被破坏的实证——模型层错了，上层全错。

---

# 三、重构方案：Rust 模型对齐四层框架

## 目标模型

```rust
// types/specification.rs（新）
pub struct Specification {
    pub api_version: String,
    pub kind: String,
    pub metadata: SpecificationMetadata,
    pub spec: SpecificationBody,
}
pub struct SpecificationBody {
    pub contract: Contract,      // ← 与 blueprint/pipeline 平级
    pub blueprint: Blueprint,
    pub pipeline: Pipeline,      // ← 可执行管道（独立，实现层）
}

// types/blueprint.rs（重构）
pub struct Blueprint {
    pub name: String,
    pub description: Option<String>,
    pub steps: Vec<Step>,        // ← 工作流步骤 = 蓝图自己的流程定义
    // cloud/deliverables/status/timeline 剥离
}

// types/pipeline.rs（重构为状态机）
pub struct Pipeline {
    pub start_at: String,
    pub states: BTreeMap<String, PipelineState>,
}
pub struct PipelineState {
    pub state_type: StateType,     // task / choice / parallel
    pub from: String,
    pub to: String,
    pub desc: String,
    pub resource: String,          // builtin:copy / python / r ...
    pub next: Option<String>,      // 或 end: true
    pub condition: Option<String>, // choice 分支条件
}
```

## 类型拆分对照

| 类型 | 现状 | 重构后 | 层 |
|------|------|--------|-----|
| `Blueprint` | name+contract+pipeline+cloud+deliverables+status+timeline | name+description+steps | 规格层 |
| `Pipeline` | name+steps（列表） | start_at+states（状态机） | 实现层 |
| `Specification` | 无 | envelope + contract/blueprint/pipeline 三块 | 规格层 |
| `Contract` | Blueprint 子字段 | Specification 平级块 | 规格层 |
| `Status`/`Timeline` | Blueprint 内 | **剥离**（运行态，后续独立 `Execution`/实例类型） | 运行层 |
| `CloudPlan`/`Deliverable` | Blueprint 内 | **剥离**（交付方案，独立类型） | 交付层 |

## 投影关系（steps → states）

```rust
impl Pipeline {
    /// 从 blueprint 工作流步骤投影为可执行状态机
    pub fn from_blueprint(bp: &Blueprint) -> Self;
}
// steps（数据流语义）→ states（控制流语义）
// 简单顺序：相邻步骤 next 串联；depends 表达的分支在投影时补 condition
```

## 与 CLI 的协作（单一事实源恢复）

1. CLI `design blueprint` → 产出 `blueprint.steps`（内容步骤）
2. CLI `spec wrap` → 组合 contract + blueprint + pipeline（**无 merge 补丁**）
3. CLI `implement` → 从 blueprint.steps 生成代码（pipeline 已是状态机）
4. CLI `process` → 消费 pipeline.states（状态机执行，支持分支）

---

# 四、提取路径（CLI → Rust 包下沉）

1. **Specification envelope 下沉**：`spec/mod.rs` 的 Specification 模型 + wrap/validate 迁入 `packages/rust/`（新 `types/specification.rs`，与重构合并），CLI `spec/` 保留命令包装
2. **Volume 下沉**：catalog.rs 的 `Volume`/`RegisterVolume` 迁入（挂 datasource 或新 `types/catalog.rs`）
3. **Job 下沉**：process.rs 的 `ProcessJobRecord` 迁入（挂 status 或新 `types/job.rs`）
4. 消费点改 `use quanttide_data::...`，CLI 150+ 测试全绿

## 跨语言同步（monorepo）

- `packages/rust/` 重构后，`packages/python/`（Pydantic）、`packages/flutter/`、`packages/dart/` 同步模型
- 契约测试：跨语言序列化互验（同一 Specification YAML 各语言解析一致）
- CLI v0.3.0 的模型下沉项（Specification/Volume/Job）与本重构合并执行

## 版本与迁移

- `quanttide-data` crate v0.1.0 → v0.2.0（破坏性：Blueprint 结构变更，旧字段 serde 兼容读取）
- CLI 依赖升 v0.2.0 后适配（spec 域重构随 v0.3.0）
- 兼容：旧 blueprint YAML（含 pipeline/status 字段）读取时迁移/忽略，写时新结构
