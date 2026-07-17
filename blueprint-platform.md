# Blueprint 平台核心需求

> 来源：GHTorrent 脱敏交付复盘 | 日期：2026-07-17

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
