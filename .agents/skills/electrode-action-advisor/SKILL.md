---
name: electrode-action-advisor
description: >
  基于约28000个历史电芯Package的完整设计差分Action Set，进行模糊动作匹配、
  配套动作集合补全和性能目标驱动的设计动作生成。
  用户询问阴极、阳极上层、阳极下层的主辅材、配比、涂布重量、
  压实密度、层数、基材厚度等修改时使用。
---

# Electrode Action Advisor

## 角色与核心对象

本 Skill 是动作集合检索、补全和候选设计生成的**编排层**。

`ActionSet(P_source, P_target) = {atomic_action_1, ..., atomic_action_n}`

一个 Action Set 表示从一个 Package 变成另一个 Package 所需的**全部设计字段变化**。它不是时间序列，也不自动证明真实迭代、工程依赖、补偿效果或因果关系。

严禁凭语言模型记忆编造历史 Package、共现频率、实验结果或性能预测值。大规模差分、统计、约束校验和预测必须由确定性后端工具完成。

## 请求模式

- `ACTION_COMPLETION`：用户给出部分设计动作，检索完整历史 Action Set，推荐剩余配套动作。
- `GOAL_GENERATION`：用户只给性能目标，检索历史动作集合并迁移到当前 Base。
- `HYBRID`：用户同时给出动作和性能目标；尊重明确动作、硬约束和禁止修改字段。
- `EXPLAIN`：解释历史 Action Set、来源和适用范围。

## 领域字段与动作

参考 `references/action-ontology.md`。至少覆盖阴极、阳极上层、阳极下层、整体结构的主辅材、配比、涂布重量、压实密度、层数和集流体厚度。动作类型为 `ADD / REMOVE / INCREASE / DECREASE / REPLACE / SET`。

连续参数保留旧值、新值、绝对变化、相对变化及单位；配比变化使用百分点 pp。只有明确材料映射时才使用 `REPLACE`，否则保留 `REMOVE + ADD`。

## 在线工作流

### 1. 解析用户意图

识别当前 `base_package_id`、自然语言需求、已指定动作、允许和禁止修改字段、性能目标、硬约束。未知幅度或材料保持 `unspecified`，不得自行编造确定数值。缺少必要 Base 时尝试从工具获取，否则请求用户指定。

### 2. 检索完整历史 Action Set

调用 `search_action_sets`，优先在同代系、材料族、结构类型和可修改范围内匹配。检索单位是**完整 Action Set**，不是孤立原子动作。

综合用户动作覆盖率、动作模板、层位、方向、幅度、历史源 Package 与当前 Base 的相似度。必须返回来源 `source_package_id`、`target_package_id`、`pair_provenance`、`action_set_id`、`actions`、`context`、`score_components`。

跨域候选显式标记；共享 Package 的多个差分对不得当作独立实验。

### 3. 动作集合补全

将用户动作 `A_user` 与历史集合 `A_hist` 进行结构化对齐。模糊动作不允许简单按字符串计算差集。

生成三类候选：
1. `NO_COMPLETION`：仅执行用户动作及配比闭合等必需修改；
2. `HISTORICAL_SET_ADAPTATION`：补全单个匹配历史集合中的剩余动作；
3. `COMPOSED_SET`：必要时组合多个历史集合的部分动作，并执行额外兼容性检查。

区分确定性的配比/工艺规则与统计共现的经验动作。共现不自动构成 `COMPENSATES` 或 `REQUIRES` 关系。允许返回“没有足够证据推荐额外动作”。

### 4. 应用到当前 Base

调用 `apply_action_set` 和 `validate_candidate`。必须把动作应用到当前 Base，而不是复制历史 Target Package。

检查材料比例闭合、层位、材料 ID、单位、参数边界、互斥动作、工艺限制、用户禁止修改字段及外推幅度。硬约束失败的候选不得作为可执行推荐。

### 5. 预测与排序

如可用，调用 `electrode-design-advisor` 或 `predict_performance`，评估完整候选 Package 的快充、循环寿命、DCR、能量密度等性能与不确定性。历史观测性能差异不得解释为因果增益。

预测服务不可用时仅输出“待验证候选”，不得虚构性能改善数值。

### 6. 输出

每个候选须包括：
- 用户原始动作与系统补全动作；
- 按阴极、阳极上层、阳极下层、整体结构分组的完整 Action Set；
- 历史来源 Package 对、真实迭代/构造差分标记；
- 匹配理由、共现分母、支持数、证据质量；
- 应用当前 Base 后的设计变化、约束检查结果；
- 预测性能及不确定性（如有）；
- 风险、适用范围、待验证事项。

## 预期后端工具

- `get_package(package_id)`
- `diff_packages(source_package_id, target_package_id)`
- `search_action_sets(base_package_id, partial_actions, objectives, filters, top_k)`
- `complete_action_set(base_package_id, partial_actions, objectives, candidate_action_set_ids, top_k)`
- `apply_action_set(base_package_id, action_set)`
- `validate_candidate(candidate_package, constraints)`
- `predict_performance(candidate_package, conditions)`（可选）
- `get_action_set_evidence(action_set_id)`（可选）

若工具不可用，明确说明不能执行真实检索，仅提供实现计划或请求必要数据。

## 质量与规模约束

- 28K Package 的无序设计对约 391,986,000，默认不全量物化。
- 优先真实版本对、同代系/同材料族近邻、任务相关候选对。
- 保留完整 Action Set 的多动作联合结构，不可只存两两共现边。
- 差分有方向，反向应明确计算。
- 共现统计报告筛选条件、分母、支持数、分组权重和来源。
- 评测按项目、Package 家族或时间分组，避免同源差分对跨训练与测试泄漏。
