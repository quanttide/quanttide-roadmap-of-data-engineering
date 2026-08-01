# 数据云 CLI 多语言运行时泛化设计

> 更新日期：2026-08-02 | 来源：qtcloud-data CLI 对接 Stata 讨论（泛化为 Python / MATLAB / R / Stata 等多语言兼容）

---

## 背景

qtcloud-data CLI 现有 pipeline 执行器按文件扩展名硬编码分发（`.py` → python3、`.sh` → bash、其余按裸命令），`implement` 只支持 `--lang python`，`doctor` 硬编码命令检测列表。

目标：让系统语言无关，Python / MATLAB / R / Stata 等多种语言可以按统一机制接入，而不是每加一种语言堆一个 `else if` 分支。

## 核心思路：解耦三个正交维度

```
语言（写什么代码）  ≠  运行时（怎么执行）  ≠  数据契约（步骤间传什么）
python / r / stata    python3 / Rscript /      CSV（中立格式）
matlab / bash         stata-mp / matlab -batch
```

- **语言**：步骤代码使用的语言
- **运行时**：执行引擎与命令行形态，随平台差异（如 Stata Windows 用 `StataMP-64 /e`、Linux/macOS 用 `stata-mp -b`）
- **数据契约**：步骤间的交换格式，当前为 CSV

## 现状耦合点

| 位置 | 现状 | 问题 |
|------|------|------|
| `process.rs` `run_pipeline` | 按扩展名 `if .py / else if .sh / else` 分发 | 每加一种语言改一处分支 |
| `implement.rs` | `match lang { "python" }` + 硬编码 `extract_python_fn` | prompt、抽取器、扩展名写死 |
| `doctor.rs` | 硬编码 `check_command("python3")` 等 | 新语言要手动加检测 |
| 传参方式 | 位置参数 `script in out` | Stata 用 `args`、R 用 `commandArgs`、MATLAB 需包函数入口，语义不一致 |

## Runtime Adapter 架构

以注册表替代扩展名分支：`resource` 字段（`engine:<script>` 形态，如 `builtin:copy`、`python:<script>`）作为**显式契约**优先，扩展名只做兜底推断。

```rust
pub trait RuntimeAdapter: Send + Sync {
    fn engine(&self) -> &'static str;                       // "python" | "r" | "stata" | ...
    fn file_extensions(&self) -> &'static [&'static str];   // [".py"], [".R"], [".do"], [".m"]
    fn build_command(&self, script: &Path, input: &Path, output: &Path) -> Command;
    /// 成败判定：python/r 看退出码；stata/matlab 需解析日志（batch 模式退出码不可靠）
    fn check_success(&self, status: ExitStatus, log: &Path) -> Result<(), String>;
    fn doctor_check(&self) -> Check;                        // 可选：运行时是否安装
    fn codegen_lang(&self) -> Option<CodegenSpec>;          // 可选：implement 支持
}
```

### 各运行时差异（adapter 的核心价值）

| 引擎 | 执行命令 | 成败判定 | 平台差异 |
|------|----------|----------|----------|
| python | `python3 script in out` | 退出码 | 低 |
| r | `Rscript script in out` | 退出码 | 低 |
| stata | `stata-mp -b do script in out` | **解析 `.log`**（batch 模式恒 exit 0） | 高（Windows `StataMP-64 /e`） |
| matlab | `matlab -batch "run('script.m')"` | 解析日志 | 中 |
| bash | `bash script in out` | 退出码 | 低 |
| builtin | `builtin:copy` / `builtin:dta2csv` 等 | — | — |

### 取参方式差异

- Stata：`args` 宏（`args input output`）
- R：`commandArgs()`
- MATLAB：包一层函数入口
- Python：`sys.argv`

## 数据契约

- `contract.input.format`（Blueprint 已有该字段，如 `"CSV"`）作为**步骤间契约**，默认 CSV
  - Stata `import delimited`、R `read.csv`、MATLAB `readtable` 均可原生消费
- `.dta` 等原生格式只在链头链尾出现；需要中间转换时加 `builtin:dta2csv` / `builtin:csv2dta` 内置 adapter
- 多输入场景（manifest）建议传**环境变量或 manifest 文件路径**，而非堆位置参数

## implement / doctor 泛化

- `implement --lang X`：注册表内 `codegen_lang` 携带 prompt 模板、代码块抽取器（` ```r ` / ` ```stata ` / ` ```matlab `）、输出扩展名（`.py` / `.R` / `.do` / `.m`），`--lang` 直接查表
- `doctor`：遍历注册表执行 `doctor_check()`，新语言零改动接入

## CLI（Rust）与 Provider（Go）双端一致性

- Specification YAML 是共享事实源，`resource` 命名规范（`engine:script`）确定后双端各自实现同一注册表
- 「resource 字符串 → adapter 配置」的解析做成纯函数，双端各有独立测试保证行为一致

## 落地顺序（避免过度泛化）

1. **抽 trait + 注册表**：`run_pipeline` 扩展名分发改成 registry 查表，python / bash 为现成样例
2. **加 R 与 Stata 两个 adapter 验证设计**：R 最简路径（`Rscript`、退出码可靠），Stata 验证日志解析路径（最复杂场景）
3. **implement / doctor 跟随注册表**改造
4. MATLAB 等后续按需添加；注册表枚举足够，**不做插件动态加载**

## 结论

> 从「按扩展名猜怎么跑」泛化为「按注册表查怎么跑 + 按契约传数据 + 按 adapter 判成败」。
> 先让 trait 设计经住 R（简单）与 Stata（复杂）两个极端用例，架构即可收敛。
