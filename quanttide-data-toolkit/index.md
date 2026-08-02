# quanttide-data-toolkit — 从 CLI 提取共享能力

> 更新日期：2026-08-02 | 来源：qtcloud-data src/cli 模块梳理（stage/spec/storage/implementation 分组后）

---

## 定位与提取原则

**定位**：领域共享库（`packages/toolkit` 约定），服务 qtcloud-data CLI 之外的更多消费方（provider Go 服务、Studio、未来工具），是"领域能力/数据契约"层，不是 CLI 应用层。

**提取原则**（从 CLI 当前结构反推）：

```
下沉到 toolkit（领域能力 + 数据契约 + 通用机制）
     ↑
CLI 保留（应用编排：命令装配、LLM 编排、StepExecutor、入口）
```

判据：**模块是否绑定 CLI 命令装配**。绑定 → 保留；不绑定（纯机制/纯模型/纯平台适配）→ 下沉。

---

## 提取候选清单

### A. 通用机制层（与领域无关，纯工具）

| 模块 | 当前位置 | 提取理由 | 优先级 |
|------|---------|---------|--------|
| **registry** | `src/registry.rs` | 通用 JSON 注册表（key→record、读整表/写原子落盘）。无任何 CLI 依赖，任意需要本地持久化登记的消费方可用（Go 服务同理） | ★★★ |
| **util 命名/路径** | `src/util.rs` | `to_camel`/`to_snake` 命名转换、目录路径解析（spec_dir/blueprint_dir/data_root）——参数化后与 CLI 解耦 | ★★ |
| **error** | `src/error.rs` | 通用错误模型（Display/From io/字符串）——可作 toolkit 基础 error，或保持 CLI 内 | ★ |

### B. 领域数据契约层（Specification 域）

| 模块 | 当前位置 | 提取理由 | 优先级 |
|------|---------|---------|--------|
| **Specification 模型 + wrap/validate** | `src/spec/mod.rs` | 四层框架"规格层"核心：Specification envelope（api_version/kind/metadata/spec）结构、YAML 解析、wrap 包装、validate 校验。CLI 只是消费者；`quanttide_data::Blueprint` 已证明模型下沉的先例 | ★★★ |
| **Volume 数据格式** | `src/implementation/catalog.rs` | "实现层"数据契约：`registry.json`/`jobs.json`/`delivery-links.json` 字段级定义（catalog.md 为事实源）。提取为共享类型 + registry 操作 | ★★★ |

### C. 领域能力层（存储平台）

| 模块 | 当前位置 | 提取理由 | 优先级 |
|------|---------|---------|--------|
| **storage 平台适配** | `src/storage/` | StorageProvider trait + 6 平台实现（dropbox/s3/google_drive/onedrive/sftp/baidu_drive）。HTTP 客户端无 CLI 依赖（`*_with_base` 注入约定已解耦），provider Go 服务等可复用 | ★★★ |

---

## 保留在 CLI（应用编排层）

| 模块 | 保留理由 |
|------|---------|
| `stage/`（clarify/design/implement/process/transfer） | LLM 编排、StepExecutor 状态机、命令装配——应用层，依赖 CLI 命令结构 |
| `review.rs`/`doctor.rs` | 命令包装，绑定 CLI 装配 |
| `spec/blueprint.rs`/`contract.rs`/`version.rs` | 查看命令，消费 B 类模型但属于 CLI 入口 |
| `main.rs`/`lib.rs` | 入口 |

> 注：stage 的**领域模型**（DRD、Job、DeliveryLink）有下沉潜力，但编排逻辑保留 CLI——提取时只搬模型类型，不搬 Handler/命令。

---

## 建议的 toolkit crate 结构

```
qtcloud-data-toolkit/
├── src/
│   ├── lib.rs
│   ├── error.rs            # A3 通用错误（可选）
│   ├── registry.rs         # A1 JSON 注册表
│   ├── util.rs             # A2 命名转换 + 路径解析（参数化）
│   ├── spec.rs             # B1 Specification 模型 + wrap/validate
│   ├── catalog.rs          # B2 Volume 数据格式
│   └── storage/            # C1 StorageProvider trait + 平台实现
│       ├── mod.rs          # trait + from_name/detect 工厂
│       └── {dropbox,s3,google_drive,onedrive,sftp,baidu_drive}.rs
└── tests/                  # wiremock 集成测试随模块迁移
```

依赖：`quanttide-data`（Blueprint 模型）、`serde/serde_yaml`、`reqwest`（storage）、`wiremock`（dev-dep 测试）。

---

## 提取步骤（建议顺序）

1. **registry + util**（A1/A2）：零依赖迁移，先建立 toolkit crate 骨架
2. **storage**（C1）：trait + 平台 + wiremock 测试整目录搬移，CLI `transfer.rs` 改依赖 toolkit
3. **spec 模型**（B1）：Specification/wrap/validate 下沉，CLI `spec/` 保留命令包装
4. **Volume 格式**（B2）：catalog.md 字段定义与类型同步下沉，CLI catalog 命令消费共享类型
5. CLI 各消费点改 `use qtcloud_data_toolkit::...`，验证 150+ 测试全绿后发布 v0.1.0

## 风险与决策点

- **错误模型归属**：error.rs 提取则 CLI 错误需适配 toolkit 错误（`From` 转换）——低优先，可缓
- **storage 与 Go 服务共用**：需确认 Go provider 是否直接消费 Rust crate（跨语言）还是仅借鉴设计——若跨语言，toolkit 的 storage 只作"参考实现"，仍值得提取（单一事实源）
- **版本管理**：toolkit 独立发版（v0.1.0 起），CLI 按 semver 依赖，`CHANGELOG.md` 双轨记录
