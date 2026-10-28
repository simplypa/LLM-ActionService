---
name: electrode-action-advisor
description: >
  电芯设计动作推荐Skill：把自然语言解析成branch_id+group_id对应的Base设计、
  多目标需求和部分动作集合；先执行CRD/可行参数范围校验，再按阳极上下层、
  阴极主辅材组合、配比、涂布、压实、层数、群裕度和基材厚度检索历史相似设计状态，
  必要时按材料物性检索替代材料，随后匹配完整历史Action Set并补全配套动作，
  验证候选Package、预测多目标性能并记录决策日志。
---

# Electrode Action Advisor

## 核心原则

本 Skill 是自然语言设计意图到可追溯候选 Action Set 的**编排层**，不是数据库、规则引擎或预测模型。

主链路：**Parse → Resolve Base → Feasibility Gateway → Historical State Retrieval → Action Set Retrieval/Completion → Validation/Prediction → Decision Log**。

- Base 由 `branch_id + group_id` 在约定数据作用域内定位；立即固定 `snapshot_id / dataset_version / design_state_id`，所有工具复用同一不可变设计版本，防止调用期间版本漂移。
- 一个 `Diff(P_i,P_j)` 对应**完整动作集合**，不是时间序列。设计对的构造差分不是实际迭代或因果证明。
- 用户明确指定的动作、禁止修改字段和硬约束必须保留；替代材料必须作为显式备选，不得暗中替换。
- 材料物性相似必须由材料特征数据库或工具计算，不得凭材料名称/LLM直觉推断。
- 无真实工具时不得编造历史 Package、相似度、CRD结论、目标区间或预测改善。

## 工作流

### 1. 解析与固定 Base

解析自然语言为：
- `base_ref`：`branch_id`、`group_id`；
- `goals[]`：指标、方向、阈值、硬/软约束、测试工况、优先级；
- `user_actions[]`：层位、材料ID、操作、旧/新值、幅度、允许修改范围；
- `restrictions[]`：不可修改字段、材料替代许可、制造约束。

调用 `resolve_base` 获取固定快照。区分未知、缺失、明确为零；未知参数不得凭空补成事实。

### 2. 第一层：CRD 与目标参数可行范围

调用 `apply_partial_actions` 构造意向设计状态；执行 `check_crd_feasibility` 和 `estimate_goal_feasible_ranges`。

- CRD硬约束：材料/层位合法、配比闭合、参数边界、工艺兼容、禁止修改字段。
- 目标区间：来自规则、历史数据或模型的估计范围，须附适用域、置信程度和版本；不能混同硬边界。
- 三态：`FEASIBLE`、`INFEASIBLE`、`UNCERTAIN`。
- `INFEASIBLE`：停止可执行推荐，解释冲突，允许提出修正后重新校验。
- `UNCERTAIN`：可继续探索性检索，但必须标注风险，不能宣称已通过验证。

### 3. 第二层：历史设计状态与材料检索

调用 `search_similar_design_states`，按用户目标字段优先、完整设计上下文复核：

1. 层位精确匹配（阴极、阳极上层、阳极下层）；
2. 主材/辅材**无序集合**匹配，材料身份对齐后再比较配比；
3. 涂布重量、压实密度、层数、群裕度、基材厚度等参数归一化距离；
4. Base 与历史源设计的材料体系、代系、结构相似度；
5. 保存每个维度的相似度、来源 State ID 和数据版本。

示例：阳极上层 `{A,B,C} → {A,B,D}`。先检索历史 `{A,B,D}`；若无，调用 `search_material_substitutes(D)` 找物性相似材料 F，再验证历史 `{A,B,F}` 是否存在。输出 `C→F` 为**替代候选**，保留用户原始 `C→D`；若禁止替代，不生成可执行替代。

**历史存在 {A,B,F} 只证明存在类似状态，不证明曾执行 C→F，也不证明其他参数是该动作的补偿。**

### 4. 第三层：完整历史 Action Set 匹配与补全

调用 `retrieve_related_action_sets`：以当前 Base、用户动作模板、历史源状态上下文匹配 `Diff(P_i,P_j)`。完整 Action Set 包含所有可比较字段变化，无默认先后关系。

将匹配集合中未覆盖的动作作为补全候选，按来源区分：
- `REQUIRED_BY_RULE`：配比闭合等确定性规则；
- `CO_OCCURRENCE`：历史共现但未验证因果；
- `SUPPORTED_COMPENSATION`：有额外机理/实验支持的补偿动作。

优先单历史集合适配；跨集合组合必须检查互斥、配比、上下文与联合性能。保留 `NO_COMPLETION` 基线。不得将相似状态中的所有参数差异直接标成历史配套动作。

### 5. 完整候选验证与性能评估

调用 `complete_candidate_action_sets`，将动作应用于**当前固定 Base**，再次执行 `validate_candidate`。如预测服务可用，调用 `predict_performance` 评估快充、循环寿命、DCR等多目标及不确定性。

硬约束失败不作为可执行推荐；没有预测服务时只给待验证候选，不编造改善幅度。

### 6. 结果与日志

输出用户原始动作、替代动作、配套动作、完整候选 Package、CRD/目标区间状态、历史 State 和 Action Set 证据、相似度分解、预测结果、风险与下一步验证。

调用 `save_decision_log` 保存 request_id、自然语言、固定 Base 快照、多目标、原始动作、各层检索与校验、候选、规则/数据/模型版本、用户反馈；遵守敏感工程数据访问与留存策略。

## 预期工具（尚待实现）

- `resolve_base(branch_id, group_id, snapshot_id?)`
- `parse_design_request(text, base)`
- `apply_partial_actions(base, actions)`
- `check_crd_feasibility(candidate, constraints)`
- `estimate_goal_feasible_ranges(base, goals, conditions)`
- `search_similar_design_states(candidate, field_scope, filters, top_k)`
- `search_material_substitutes(material_id, property_scope, filters)`
- `retrieve_related_action_sets(base, partial_actions, filters, top_k)`
- `complete_candidate_action_sets(base, matched_sets, user_actions)`
- `validate_candidate(candidate, constraints)`
- `predict_performance(candidate, conditions)`（可选）
- `save_decision_log(record)`

详细契约、例子、规模策略和验收见仓库根目录 `README.md`；动作本体见 `references/action-ontology.md`。

## 规模与质量

28K Package 对应约3.92亿无序设计对，不默认全量物化。优先真实版本对、同材料族近邻、任务相关差分与按需检索；共享Package对非独立样本。按项目/Package族/时间先分组切分，再生成训练与评测差分。状态检索、动作集合检索、材料物性近邻是不同证据，必须分开标注。
