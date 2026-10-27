# LLM-ActionService

基于约 **28,000 个历史电芯 Package**，构建以完整 **Action Set（动作集合）** 为基本单位的设计差分知识库，实现用户模糊动作匹配、工程配套动作补全与性能目标驱动的候选设计生成。

> 当前为 Skill 配置和技术设计草案；尚未连接真实 Package 数据、生产数据库或性能预测服务。

## 1. 核心定义

**Package A → Package B = Action Set {a1, a2, …, ak}**

- **Package**：一个完整电芯设计状态。
- **Atomic Action**：某个材料或参数的增加、减少、添加、去除、替换或设定，记录旧值、新值、幅度、单位、层位。
- **Action Set**：从源 Package 到目标 Package 的**全部设计字段变化**，不是动作执行时间序列。
- **Action Set Evidence**：源/目标 Package、设计上下文、差分来源、相关文档和实验结果。
- **Action Completion**：用户给出部分动作，匹配历史完整动作集合并推荐缺失的配套动作。
- **Goal-conditioned Generation**：用户给出性能目标，从历史动作集合中召回候选，应用到当前 Base 并评估。

两个独立 Package 的差分只是**构造的虚拟变换**，不证明真实历史迭代；动作共现也不证明补偿关系、依赖关系或因果效应。

## 2. 设计字段范围

| 组件 | 材料字段 | 设计字段 |
| --- | --- | --- |
| 阴极 cathode | 主材、辅材、材料 ID、配比 | 涂布重量、压实密度、集流体/基材厚度 |
| 阳极上层 anode_upper | 主材、辅材、材料 ID、配比 | 涂布重量、压实密度 |
| 阳极下层 anode_lower | 主材、辅材、材料 ID、配比 | 涂布重量、压实密度 |
| 整体结构 cell_structure | 可扩展 | 层数、尺寸、基材厚度 |
| 后续扩展 | 电解液、隔膜 | 工艺条件、测试工况 |

操作类型：`ADD`、`REMOVE`、`INCREASE`、`DECREASE`、`REPLACE`、`SET`。配比差异使用百分点（pp），并保留真实单位与原始值。

## 3. 系统架构

```text
[离线] 28K Packages + 字段字典 + 版本信息 + 性能表
  ├── 字段标准化 / 材料别名归一 / 单位转换
  ├── 候选 Package 对筛选（代系、材料族、近邻、真实版本）
  ├── Diff(Pi, Pj) → 完整 Action Set
  ├── Action Set 模板化、索引、去重、来源标注
  └── 条件共现 / 频繁项集 / 集合检索索引
                           │
[在线] 用户自然语言 + 当前 Base + 目标/约束
  ├── LLM 意图解析
  ├── 相似完整 Action Set 检索
  ├── 用户已指定动作对齐 → 剩余动作集合补全
  ├── 历史动作幅度适配当前 Base
  ├── 配比闭合 / 工艺约束 / 冲突检查
  ├── electrode-design-advisor 性能评估（可选）
  └── Top-K 动作集合、候选 Package、历史证据
```

## 4. 28K Package 的设计对策略

28,000 个 Package 对应 `28,000 × 27,999 / 2 = 391,986,000` 个无序对，约 **3.92 亿**。不应默认全量物化所有差分。

1. **真实版本边优先**：如果有同项目真实迭代关系，直接保留。
2. **按上下文分组**：代系、材料族、结构类型、允许修改的字段。
3. **近邻候选召回**：在组内按混合类别/连续特征距离检索 Top-K 设计对。
4. **定向差分**：计算 `Diff(Pi,Pj)`，保存方向，必要时动态生成反向。
5. **查询时扩展**：罕见材料替换或跨体系动作可扩大召回范围。
6. **来源与去偏**：多个设计对共享同一 Package，不可当作独立实验；统计按项目、Package 族或时间分组。

## 5. 数据模型

### packages
`package_id, generation, material_family, project_id, version, timestamp, normalized_design_json, dataset_version`

### action_sets
`action_set_id, source_package_id, target_package_id, pair_provenance, context_json, action_count, observed_performance_delta_json, evidence_ids, dataset_version`

### atomic_actions
`action_set_id, action_index, component, field_path, material_id, operation, old_value, new_value, delta_absolute, delta_relative, unit, action_template`

### action_set_statistics
`context_key, antecedent_templates, consequent_templates, support_count, weighted_support, confidence, lift, sampling_policy, dataset_version`

### recommendation_runs
`request_id, base_package_id, user_intent_json, matched_action_set_ids, candidate_action_sets_json, validation_results_json, prediction_results_json, created_at`

## 6. 核心工具接口

| 工具 | 输入 | 输出 |
| --- | --- | --- |
| `get_package` | package_id | 标准化 Package |
| `diff_packages` | source_id, target_id | 完整定向 Action Set |
| `search_action_sets` | Base, 部分动作, 目标, filters, top_k | 历史集合、分数、来源 |
| `complete_action_set` | Base, 用户动作, 匹配集合 | 补全动作集合 |
| `apply_action_set` | Base, Action Set | 候选 Package |
| `validate_candidate` | 候选 Package, 约束 | 可行性、冲突原因 |
| `predict_performance` | 候选 Package, 工况 | 预测结果、不确定性、模型版本 |
| `get_action_set_evidence` | action_set_id | 来源设计对与文档 |

### 示例 Action Set（虚构数据）

```json
{
  "action_set_id": "AS_P001_P008",
  "source_package_id": "P001",
  "target_package_id": "P008",
  "pair_provenance": "constructed_difference",
  "actions": [
    {"component":"cathode","field_path":"active_material","operation":"REPLACE","old_value":"NCM811","new_value":"NCM9"},
    {"component":"cathode","field_path":"coating_weight","operation":"INCREASE","old_value":200,"new_value":205,"delta_absolute":5,"unit":"example_unit"},
    {"component":"anode_upper","field_path":"active_materials[graphite_A].ratio","operation":"DECREASE","old_value":90,"new_value":85,"delta_absolute":-5,"unit":"percentage_point"},
    {"component":"anode_upper","field_path":"active_materials[graphite_B].ratio","operation":"INCREASE","old_value":10,"new_value":15,"delta_absolute":5,"unit":"percentage_point"},
    {"component":"cathode","field_path":"current_collector_thickness","operation":"DECREASE","old_value":12,"new_value":10,"delta_absolute":-2,"unit":"um"}
  ]
}
```

## 7. 两种在线工作流

### A. 用户提出部分动作：Action Completion

用户：“提高阳极上层快充石墨比例，其他材料可以调整，但不能影响循环寿命。”

1. 解析用户动作与约束，未知幅度保持未指定。
2. 检索上下文相似且包含匹配动作的**完整历史 Action Set**。
3. 对齐用户动作，计算每个历史集合中尚未指定的剩余动作。
4. 同时生成仅用户动作（加必要配比闭合）的基线候选。
5. 把补全动作适配当前 Base，检查配比、参数边界和冲突。
6. 如模型可用，评估完整候选 Package 并排序，展示历史来源。

### B. 用户提出性能目标：Goal-conditioned Generation

用户：“能量密度提高 5 Wh/kg，快充不能变差。”

1. 解析目标与可行动作空间。
2. 在可比工况下召回历史性能变化方向相符的 Action Set。
3. 观测差分仅作召回线索，不视为因果增益。
4. 把候选动作应用于当前 Base，校验并预测。
5. 输出 Top-K 动作集合及完整候选 Package。

## 8. 推荐策略 V1

检索排序结合：用户动作覆盖率、历史源 Package 与当前 Base 的上下文相似度、动作目标/方向/幅度相似度、集合复杂度、证据质量、外推风险。最终推荐还需考虑约束满足与预测性能。

关联规则、Support、Confidence、Lift 可以辅助发现共现动作，但必须保留完整 Action Set，不能仅保存两两共现边。多集合拼接需重新检查兼容性。允许 `NO_COMPLETION`（无需额外经验动作）。

## 9. 验收与评测

- **差分正确性**：`Apply(Pi, Diff(Pi,Pj))` 应复原标准化的 `Pj`（允许数值容差）。
- **集合补全**：隐藏真实集合的部分动作，评估 Recall@K、Precision@K、Exact Set Match。
- **候选合法性**：配比闭合率、参数边界满足率、冲突率。
- **跨域泛化**：按项目、材料族、Package 家族和时间划分训练/测试，避免同源差分泄漏。
- **优化价值**：目标达成率、实验预算下的 HV、可行率；优先用独立留出实验验证。
- **消融实验**：仅用户动作 / 单历史集合补全 / 共现规则 / 多集合组合 / 性能模型。

## 10. 开发里程碑

- **M0 数据体检**：确认真实字段、单位、材料 ID、缺失值、配比基准、版本关系和工况可比性。
- **M1 差分引擎**：字段标准化、完整 Action Set 提取、往返复原测试。
- **M2 离线索引**：候选对筛选、检索索引、集合统计与证据追踪。
- **M3 在线 Skill**：模糊动作解析、完整集合匹配、补全与约束校验。
- **M4 性能闭环**：对接现有 Transformer、目标排序、留出评测。

## 11. Skill 安装与运行

将 `.agents/skills/electrode-action-advisor/` 保留在本仓库。该目录中的 `SKILL.md` 定义大模型工作流，`references/action-ontology.md` 定义动作本体。

Skill 本身**不能自动访问真实 Package 数据库**。需要将第 6 节接口实现为可调用脚本、内部 API 或 MCP 工具，并授予模型访问权限。未接入工具时只能进行架构讨论，不能假装检索了真实历史。

## 12. 待确认的真实数据

1. 28K Package 的实际表头、宽表/长表/JSON 存储形式和单位。
2. 阴极、阳极上层、阳极下层的材料 ID 与别名。
3. 代系、材料体系、项目与版本字段是否存在。
4. 性能表与 Package 的关联及测试工况一致性。
5. 可修改字段、制造硬约束和材料兼容规则。
