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
