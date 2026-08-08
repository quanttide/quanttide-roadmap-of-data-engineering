# Specification 域设计（src/spec/）

> 更新日期：2026-08-02 | 对应代码：`src/spec/`（spec/blueprint/contract/version）+ `quanttide-data` crate（共享模型）

---

# 设计与实现不一致分析（实测确认）

四层框架（[index.md](index.md)）是**设计**，`src/cli` 与 `quanttide-data` crate 是**实现**。
逐项对比发现三处不一致，均经实际运行验证（`spec wrap` / `spec validate` 实测）。

## 不一致 1：规格层结构——Contract 应平级，实现却嵌套

### 设计（四层框架）

```
规格层 Specification
    ├── Contract（数据契约：输入输出规格）
    └── Blueprint（数据蓝图：工作流步骤）
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
    blueprint:           # ← 数据蓝图：工作流步骤 = 流程定义（有自己的定义方式）
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
