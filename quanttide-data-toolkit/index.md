# quanttide-data-toolkit — 领域模型库总览

> 更新日期：2026-08-02 | 来源：quanttide-data-toolkit monorepo 勘察 + qtcloud-data src/cli 模型梳理

## 文档索引

| 文档 | 对应子包 | 内容 |
|------|---------|------|
| [index.md](index.md) | monorepo | 定位、SDK 列表、模型矩阵、消费方地图、文档同步 |
| [rust.md](rust.md) | `packages/rust/` | Rust 包（quanttide-data crate）：模型现状、**不一致分析与重构方案**、提取路径 |

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

## 二、模型矩阵：toolkit 已有 vs CLI 待下沉

| 领域模型 | CLI 位置 | toolkit Rust 包 | 状态 |
|---------|---------|----------------|------|
| Blueprint | （消费 toolkit） | ✅ \`types::blueprint\` | **已下沉** |
| Contract | （消费 toolkit） | ✅ \`types::contract\` | **已下沉** |
| Pipeline/Step | （消费 toolkit） | ✅ \`types::pipeline\` | **已下沉** |
| **Specification envelope** | \`src/spec/mod.rs\`（api_version/kind/metadata/spec + wrap/validate） | ❌ 无 | **待下沉** |
| **Volume** | \`src/implementation/catalog.rs\`（registry.json 记录） | ❌ 无（datasource 相关） | **待下沉** |
| **Job** | \`src/stage/process.rs\`（jobs.json 记录） | ❌ 无（status 相关） | **待下沉** |
| DRD | \`src/stage/clarify.rs\`（自由格式文件，无强类型） | ❌ 无 | 暂不建模 |

## 三、消费方地图（下沉价值）

| 消费方 | 语言 | 模型来源 |
|--------|------|---------|
| qtcloud-data CLI | Rust | \`quanttide-data\` crate（Blueprint/Contract/Pipeline ✅，Specification/Volume/Job ⏳） |
| Python SDK 消费方 | Python | \`packages/python\`（契约测试互验） |
| Flutter/Dart 消费方 | Dart | \`packages/flutter\`/\`packages/dart\`（UI 展示模型） |
| provider Go 服务 | Go | 跨语言借鉴设计（单一事实源在 toolkit） |

## 四、配套文档同步（toolkit 现状过时项）

勘察发现 toolkit 仓库内部文档滞后于实际，重构时一并修正：

| 文档 | 过时项 | 修正 |
|------|--------|------|
| \`README.md\` SDK 表 | 只列 python/flutter，缺 dart/rust | 补全四语言 |
| \`ROADMAP.md\`（主仓） | "创建 packages/rust/"未勾选，实际已存在 | 勾选/移除 |
| \`packages/rust/ROADMAP.md\` | "包名 quanttide-data-core \| 状态规划中"与 Cargo.toml（quanttide-data）不符 | 对齐包名与状态 |
| \`STATUS.md\`（quanttide-data 仓） | 标注 dart/v0.3.0-9，未体现 rust 包 | 补 rust/v0.1.0 行 |

---

> 模型不一致分析与重构方案见 [rust.md](rust.md)（Rust 包是单一事实源，先修它）。
