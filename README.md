# LLM-ActionService

面向约 **28,000 个历史电芯 Package** 的自然语言驱动设计动作推荐服务。核心链路是：

**自然语言 → Base 设计状态 + 多目标需求 + 用户动作集 → CRD/可行域校验 → 历史设计状态及材料组合检索 → 历史 Action Set 匹配与补全 → 候选 Package 验证/预测 → 决策日志。**

> 当前仓库是 Skill 和技术方案，尚未接入真实设计表、CRD、材料物性库或预测服务。所有示例数据均为示意，不得当作真实推荐。

## 1. 对象与唯一标识

- **Base State**：由 `branch_id + group_id` 定位历史设计；该组合仅在数据服务的约定作用域内唯一，必须同时绑定 `dataset_version / snapshot_id`，避免版本漂移。若数据服务保证唯一性，仍应返回不可变 `design_state_id` 供后续调用复用。
- **Goal Set**：多个性能目标及方向、阈值、优先级、硬/软约束，例如快充时间降低、循环寿命不下降、DCR 上限。
- **User Action Set**：自然语言解析的部分或完整动作；允许未指定幅度、材料别名、允许修改范围。
- **Candidate State**：将用户动作应用于 Base 得到的意向设计状态，可能包含未知字段或待补全的配比。
- **Historical State**：28K Package 设计表中的完整历史状态。
- **Historical Action Set**：`Diff(P_source,P_target)` 得到的**完整、有方向、无默认时序**的动作集合。
- **Recommendation**：保留原始动作、修正/替代动作、配套补全动作、目标验证和证据来源。

**两条检索路径必须分开**：State Retrieval 查历史上有没有类似目标设计；Action Set Retrieval 查历史上是否存在类似变换及配套动作。状态相似不能证明动作补偿或真实修改路径。

## 2. 数据字段

| 部位 | 设计维度 |
| --- | --- |
| 阳极上层 | 最多约3种主材及配比、少于7种辅材及配比、涂布重量、压实密度等 |
| 阳极下层 | 主材及配比、辅材及配比、涂布重量、压实密度等；依实际表头 |
| 阴极（主要单层涂布） | 主材及配比、辅材及配比、涂布重量、压实密度等 |
| 电芯结构 | 层数、群裕度、基材/集流体厚度、其他几何/工艺参数 |
| 上下文 | branch_id、group_id、设计版本、项目/代系、材料体系、性能测试工况 |

材料槽位属于**无序材料集合**，应按材料 ID 对齐，不可将槽位1/2/3的顺序误当材料身份。比例需保留归一化基准；主辅材是否共用分母由数据字典决定。未知、未提供和明确为零必须区分。

## 3. 总体架构

```text
用户自然语言
   │
   ▼
[Intent Parser]
   ├── resolve_base(branch_id, group_id, snapshot)
   ├── goals: 多目标 / 硬软约束 / 目标工况
   └── partial_actions: 修改对象、操作、幅度、允许范围
   │
   ▼
[Layer 1: Feasibility Gateway]
   ├── apply_partial_actions(base) → 意向 Candidate State
   ├── CRD/规则检查：材料、配比、层位、制造边界
   ├── goal_feasible_ranges：目标对应的参数可行区间（估计）
   └── FEASIBLE / INFEASIBLE / UNCERTAIN
         ├── INFEASIBLE → 解释冲突 / 提出修正，不进入可执行推荐
         └── FEASIBLE 或受控 UNCERTAIN
   │
   ▼
[Layer 2: Historical State & Material Retrieval]
   ├── exact/near match：层位、材料组合、配比、结构参数
   ├── material property neighbors：D → 物性相似的 F
   ├── 保留源历史 State、相似度各维贡献和证据
   └── 候选设计状态（不自动视为历史动作）
   │
   ▼
[Layer 3: Historical Action Set Retrieval]
   ├── 基于 Base + 用户动作检索历史完整 Diff(Pi,Pj)
   ├── 匹配用户动作 → 剩余动作补全
   ├── 必要配比闭合 vs 统计共现配套动作分别标注
   └── 单集合适配优先，跨集合组合需额外检查
   │
   ▼
[Candidate Validation & Performance]
   ├── 适配 Base → 完整候选 Package
   ├── 再次执行 CRD / 工艺硬约束
   ├── Transformer/代理模型评估多目标（可用时）
   └── Top-K 推荐 + 证据 + 不确定性
   │
   ▼
[Decision Log]
   └── 原始需求、Base快照、解析动作、每层校验/检索、
       推荐、用户反馈、数据/规则/模型版本
```

## 4. Layer 1：校验与目标可行范围

先解析用户动作并构造意向设计状态，再执行：

1. **CRD 硬校验**：合法材料、层位、配比闭合、上下限、制造兼容、禁止修改字段。
2. **目标范围估计**：基于工程知识、历史统计或预测模型，给出满足快充/循环/DCR等多目标的候选参数区间和置信程度；区间是**估计**，不能混同 CRD 硬边界。
3. **三态决策**：`FEASIBLE`（已知硬约束满足）、`INFEASIBLE`（明确违反）、`UNCERTAIN`（信息不足或模型外推）。UNCERTAIN 可进入探索性检索，但不能标记为可执行推荐；INFEASIBLE 应说明冲突并建议修正，修正后重验。

校验结果包含违反规则、所用 CRD 版本、目标区间来源、是否可继续及原因。

## 5. Layer 2：参数级历史状态检索

**示例**：Base 阳极上层主材 `{A:50%, B:30%, C:20%}`；用户要求 `C→D`，意向状态 `{A:50%, B:30%, D:20%}`。

检索顺序：

1. **同部位精确组合**：历史是否存在 `{A,B,D}`；若存在，继续比较配比、辅材、涂布/压实、结构等。
2. **物性近邻替代**：若无 `{A,B,D}`，检索 D 的物性近邻 F，并确认历史存在 `{A,B,F}`。F 只是**替代建议**，不能悄悄覆盖用户明确指定的 D；原始 C→D 保留为待验证候选。
3. **组合级排序**：材料集合相似度、材料物性距离、配比距离、极片/结构参数距离、Base 上下文相似度；各维归一化并保留分数分解。
4. **约束优先**：用户禁止替代材料时，不输出 F 作为可执行方案；物性距离需真实材料特征及适用域支持。

状态检索的结果是历史设计参考，不意味着“历史上曾执行 C→F”。

## 6. Layer 3：历史动作集合匹配与补全

`A_ij = Diff(P_i, P_j) = {a_1, ..., a_k}`，每个集合包含所有已定义可比较字段的变化，**无动作先后关系**。

- 匹配用户的部分动作模板与幅度、当前 Base 和历史源状态相似度。
- 将历史集合中未被用户动作覆盖的部分作为配套动作**候选**。
- 先输出单个历史集合的适配结果；跨集合拼接时做冲突和联合性能验证。
- 区分 `REQUIRED_BY_RULE`（配比闭合等）、`CO_OCCURRENCE`（共现）、`SUPPORTED_COMPENSATION`（另有机理/实验支持）。
- 历史独立 Package 的差分是构造对比，非真实迭代或因果证据。

28K Package 有约 **391,986,000 个无序对**；优先真实版本对、同代系/材料族、近邻候选对、按需差分，**不默认全量物化**。同一 Package 衍生的多个对高度相关，统计及评测按项目/设计族分组。

## 7. 工具接口（拟实现）

| 接口 | 职责 |
| --- | --- |
| `resolve_base(branch_id, group_id, snapshot_id?)` | 返回唯一且版本固定的完整设计状态 |
| `parse_design_request(text, base)` | 结构化多目标、动作、约束和缺失项 |
| `apply_partial_actions(base, actions)` | 构造意向状态，保留未定参数 |
| `check_crd_feasibility(candidate, constraints)` | CRD硬规则检查与三态结果 |
| `estimate_goal_feasible_ranges(base, goals, conditions)` | 多目标参数可达范围估计及证据 |
| `search_similar_design_states(candidate, field_scope, filters, top_k)` | 参数级历史状态/组合检索 |
| `search_material_substitutes(material_id, property_scope, filters)` | 物性近邻材料检索 |
| `retrieve_related_action_sets(base, partial_actions, filters, top_k)` | 完整历史差分集合检索 |
| `complete_candidate_action_sets(base, matched_sets, user_actions)` | 候选补全、替代和规则闭合 |
| `validate_candidate(candidate, constraints)` | 完整候选再校验 |
| `predict_performance(candidate, conditions)` | 多目标性能及不确定性（可选） |
| `save_decision_log(record)` | 记录输入、过程、输出、版本和反馈 |

接口为契约草案；Skill 不会自动使这些工具可用。

## 8. 日志与可追溯性

每次请求创建 `request_id`，保存：

```json
{
  "request_id": "REQ_001",
  "base": {"branch_id":"BR001","group_id":"G02","snapshot_id":"SNAP_001"},
  "user_request": {
    "text":"阳极上层C换成D，改善快充，循环不能下降",
    "goals":[
      {"metric":"fast_charge_time","direction":"minimize"},
      {"metric":"cycle_life","constraint":"no_degradation"}
    ],
    "actions":[{"component":"anode_upper","operation":"REPLACE","from":"C","to":"D"}]
  },
  "feasibility":{"status":"UNCERTAIN","rule_version":null},
  "state_matches":[],
  "material_substitutes":[],
  "matched_action_sets":[],
  "candidate_recommendations":[],
  "versions":{"dataset":null,"model":null,"skill":null}
}
```

记录每个阶段的耗时、证据、失败原因、用户是否接受以及后续实验反馈。设计/材料商业敏感数据需遵守内部访问控制、脱敏和日志留存要求。

## 9. 评测与实施

- **状态检索**：材料组合 Exact Match、Top-K Recall、物性近邻有效性、分维度排序解释性。
- **动作补全**：Masked Action Set Recall@K、Exact Set Match、配套动作专家认可率。
- **约束**：硬约束违反率、三态校准、参数区间覆盖率。
- **性能**：独立留出 Package/实验上的目标达成率、模型误差与不确定性。
- **数据切分**：先按项目/Package族/时间切分，再生成训练/评测差分，避免共享Package泄漏。

里程碑：M0字段/CRD/物性字典确认 → M1 Base固定与动作解析 → M2 CRD校验 → M3状态与材料检索 → M4动作集合索引及补全 → M5模型评估与日志闭环。

## 10. Skill 安装

Skill 位于 `.agents/skills/electrode-action-advisor/SKILL.md`；字段本体见 `references/action-ontology.md`。Skill 负责 LLM 编排，确定性计算必须由内部脚本、API 或 MCP 工具实现。上线前需确认 CRD 含义、真实字段映射、材料物性特征、版本化数据服务和性能模型接口。
