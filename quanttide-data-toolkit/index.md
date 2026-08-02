# quanttide-data-toolkit — 领域模型库

> 更新日期：2026-08-02 | 来源：qtcloud-data src/cli 领域模型梳理

## 定位

**领域库**（`packages/toolkit` 约定）：承载量潮数据领域四层框架的**领域模型**（DRD / Specification / Catalog / Job），供 CLI、provider Go 服务、Studio 等所有消费方共享。**不承载通用机制**（registry/util/error 等非领域内容不在本库）。

## 领域模型清单

| 模型 | 来源模块 | 四层框架 | 内容 | 当前状态 |
|------|---------|---------|------|---------|
| **Specification** | `src/spec/mod.rs` | 规格层 | envelope（api_version/kind/metadata/spec）+ Contract + Blueprint（`quanttide_data::Blueprint`），wrap/validate 逻辑 | 已有完整模型 + 逻辑，CLI 仅是消费者 |
| **Volume** | `src/implementation/catalog.rs` | 实现层 | `registry.json` volume 记录（字段级定义见 catalog.md） | 类型存在，随 catalog 命令绑定 |
| **Job** | `src/stage/process.rs` | 任务层 | `jobs.json` job 记录（ProcessJobRecord） | 类型存在，随 process 命令绑定 |
| **DRD** | `src/stage/clarify.rs` | 需求层 | 需求文档——当前为自由格式文件，无强类型模型 | 暂不下沉；若建模（结构化为 YAML）再入库 |

## 不在本库范围

- **通用机制**：`registry.rs`（JSON 注册表）、`util.rs`、`error.rs`——非领域内容，不属领域库
- **存储平台**：`storage/`（StorageProvider + 6 平台）——是领域**能力**而非**模型**，提取与否另议
- **应用编排**：`stage/` Handler、StepExecutor、命令装配——保留 CLI

## 建议结构

```
qtcloud-data-toolkit/
├── src/
│   ├── lib.rs
│   ├── spec.rs        # Specification 模型 + wrap/validate
│   ├── catalog.rs     # Volume 数据格式
│   └── job.rs         # Job 记录（jobs.json）
└── tests/             # 模型序列化/校验测试随迁
```

依赖：`quanttide-data`（Blueprint）、`serde/serde_yaml`。

## 提取步骤

1. `spec.rs`：Specification 模型 + wrap/validate 下沉（逻辑已有，迁移最小）
2. `catalog.rs` + `job.rs`：Volume/Job 类型下沉，CLI 命令改依赖 toolkit
3. 消费点改 `use qtcloud_data_toolkit::...`，150+ 测试全绿后发 v0.1.0
