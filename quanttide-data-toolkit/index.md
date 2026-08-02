# quanttide-data-toolkit — 领域模型库（现状勘察 + 提取路径）

> 更新日期：2026-08-02 | 来源：quanttide-data-toolkit monorepo 勘察 + qtcloud-data src/cli 模型梳理

## 一、toolkit 现状：多语言 monorepo

`packages/quanttide-data-toolkit` 是**多语言数据工具包单仓**（`monorepo`），统一承载数据领域模型，各语言 SDK 共享契约：

```
quanttide-data-toolkit/
├── packages/
│   ├── python/          # Python SDK（quanttide_data，Pydantic 模型）
│   ├── dart/            # Dart 工具包
│   ├── flutter/         # Flutter SDK（flutter_quanttide_data，UI 组件 + BLoC）
│   └── rust/            # Rust crate（quanttide-data v0.1.0）★ CLI 已在消费
```

## 二、Rust 包现状（CLI 提取的目标，已存在非新建）

`packages/rust/` = crate `quanttide-data`（v0.1.0，已发布 crates.io）——**qtcloud-data CLI 已依赖**（`Cargo.toml: quanttide-data = "0.1.0"`，`spec/mod.rs` 使用 `quanttide_data::Blueprint`）。

### 已有模型（toolkit 侧）

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

## 三、模型矩阵：toolkit 已有 vs CLI 待下沉

| 领域模型 | CLI 位置 | toolkit Rust 包 | 状态 |
|---------|---------|----------------|------|
| Blueprint | （消费 toolkit） | ✅ `types::blueprint` | **已下沉** |
| Contract | （消费 toolkit） | ✅ `types::contract` | **已下沉** |
| Pipeline/Step | （消费 toolkit） | ✅ `types::pipeline` | **已下沉** |
| **Specification envelope** | `src/spec/mod.rs`（api_version/kind/metadata/spec + wrap/validate） | ❌ 无 | **待下沉** |
| **Volume** | `src/implementation/catalog.rs`（registry.json 记录） | ❌ 无（datasource 相关） | **待下沉** |
| **Job** | `src/stage/process.rs`（jobs.json 记录） | ❌ 无（status 相关） | **待下沉** |
| DRD | `src/stage/clarify.rs`（自由格式文件，无强类型） | ❌ 无 | 暂不建模 |

## 四、提取路径（补齐已有 Rust 包，非新建）

1. **Specification envelope 下沉**：`spec/mod.rs` 的 Specification 模型 + wrap/validate 迁入 `packages/rust/`（新 `types/specification.rs`），CLI `spec/` 保留命令包装
2. **Volume 下沉**：catalog.rs 的 `Volume`/`RegisterVolume` 迁入（挂 datasource 或新 `types/catalog.rs`）
3. **Job 下沉**：process.rs 的 `ProcessJobRecord` 迁入（挂 status 或新 `types/job.rs`）
4. 消费点改 `use quanttide_data::...`，CLI 150+ 测试全绿，Rust 包升 v0.2.0

## 五、配套文档同步（toolkit 现状过时项）

勘察发现 toolkit 仓库内部文档滞后于实际，提取时一并修正：

| 文档 | 过时项 | 修正 |
|------|--------|------|
| `README.md` SDK 表 | 只列 python/flutter，缺 dart/rust | 补全四语言 |
| `ROADMAP.md`（主仓） | "创建 packages/rust/"未勾选，实际已存在 | 勾选/移除 |
| `packages/rust/ROADMAP.md` | "包名 quanttide-data-core \| 状态规划中"与 Cargo.toml（quanttide-data）不符 | 对齐包名与状态 |
| `STATUS.md`（quanttide-data 仓） | 标注 dart/v0.3.0-9，未体现 rust 包 | 补 rust/v0.1.0 行 |

## 六、消费方地图（下沉价值）

| 消费方 | 语言 | 模型来源 |
|--------|------|---------|
| qtcloud-data CLI | Rust | `quanttide-data` crate（Blueprint/Contract/Pipeline ✅，Specification/Volume/Job ⏳） |
| Python SDK 消费方 | Python | `packages/python`（契约测试互验） |
| Flutter/Dart 消费方 | Dart | `packages/flutter`/`packages/dart`（UI 展示模型） |
| provider Go 服务 | Go | 跨语言借鉴设计（单一事实源在 toolkit） |
