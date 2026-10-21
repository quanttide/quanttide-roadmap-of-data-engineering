# quanttide-roadmap-of-data-engineering

数据工程领域的产品路线图、版本规划与迭代计划。

## 概述

本仓维护数据工程领域的方向文档，回答「先做什么、后做什么」：

- 阶段划分与各阶段的达成标志
- 里程碑与验证方式
- 优先级与排期
- 演进顺序与后置代价

不回答「为什么做、做成什么样」——那属意图仓（`data/intention`）。

## 文档

- [qtcloud-data/blueprint.md](qtcloud-data/blueprint.md)
- [qtcloud-data/lang.md](qtcloud-data/lang.md)
- [qtcloud-data/spec.md](qtcloud-data/spec.md)
- [quanttide-data-toolkit/index.md](quanttide-data-toolkit/index.md)
- [quanttide-data-toolkit/rust.md](quanttide-data-toolkit/rust.md)

## 约定

- 阶段三要素：达成标志、顺序依据、后置代价；缺一项该阶段不成立。
- 同一事项只写一处：实施步骤与代码落点归应用仓 `src/<端>/docs/dev-guide/roadmap.md`，本仓只留方向与判据。
- 复核按抽样：每阶段至少抽一条，条目须陌生人当场可判。

## 关联

- 意图（为什么做、做成什么样）：领域仓 `data/intention`
- 日志（发生了什么）：`data/journal`
- 档案（沉淀与叙事）：`data/profile`
- 端级实现计划：应用仓 `src/<端>/docs/dev-guide/roadmap.md`

## 许可

[CC BY 4.0](LICENSE)
