---
name: cell-cost-verification
description: 面向电芯Base Package的降本验证接口，识别模糊动作和性能需求，匹配结构化降本动作，执行CRD与软约束验证，输出成本、性能及风险验证结果。
---

# 电芯降本验证接口 Skill

## 定位
本Skill是Agent编排规则，不替代确定性的业务工具。降本动作库保存可执行JSON；CRD、成本模型、性能预测、知识检索分别由工具执行。

## 用户输入
- Base设计：唯一标识(package + branchID + group + 可选revision)或完整Package JSON。
- 意图：降本/其他性能优化、目标降本比例（可选）。
- 性能需求：指标、比较符、目标值、单位、测试工况（寿命、低温、快充、DCR等）。
- 用户模糊或明确动作（可选）；缺省时由动作库推荐。

## 执行步骤
1. **意图识别**：多轮提取Base、模糊动作、目标和多维性能要求；工况缺失时追问或标记待确认，不得猜测。
2. **Base获取**：通过Data MCP读取完整设计，确认材料体系、Mapping字段、比例和结构参数。
3. **动作召回**：从降本动作库按材料体系、Base相似度、模糊意图和历史案例检索；无用户动作时主动推荐。
4. **动作实例化**：将模板映射为明确field_path、operation、expected_old_value、new_value、联动参数，生成候选Package；缺少材料编码则返回NEEDS_CLARIFICATION。
5. **候选校验**：CRD硬约束失败则拒绝候选；统计软约束异常保留WARNING并输出风险，不将统计罕见等同于不可设计。
6. **成本评估**：可计算时使用用量×价格和已知工艺成本；不可计算时只输出方向性结论及Wiki/GO相似案例证据，禁止编造百分比。
7. **性能评估**：调用性能服务，按一致工况比较Base、候选、需求；提供原始值、预测不确定性、达标裕度及雷达图数据。
8. **风险证据**：分别输出Wiki机理/失效经验、GO经典案例观察、软约束异常和模型预测风险，标注证据来源。
9. **验证结果**：返回具体动作、候选设计、校验状态、成本估计、性能对比、风险、历史相似案例和待实验事项。
10. **实验反馈**：仅将有真实实验记录的成功/失败案例用于更新推荐统计；模型预测不算成功案例。

## 动作库冷启动（离线流程）
Wiki八类降本知识与GO元素推荐动作 → 模糊动作抽取 → Mapping字段/材料编码落地 → JSON Schema校验 → CRD可设计性验证与软约束风险记录 → 入库。
冷启动动作允许没有成功案例，必须明确evidence_level=knowledge_only。已有GO元素经典案例若包含Base/动作/实验结果，可作为独立case记录并关联动作；不得自动视为普遍适用。

## 建议工具
- get_package_design
- search_cost_actions
- instantiate_cost_action
- verify_design_constraints
- estimate_cost_delta
- predict_cell_performance
- retrieve_cost_evidence
- build_cost_verification_result
- record_cost_feedback

## 不可违反的规则
- 不伪造Mapping字段、材料编码、材料价格、真实实验结果。
- 入库前模板校验与用户Base上的候选校验是两次不同的校验。
- CRD硬约束失败不得进入常规推荐。
- 软约束WARNING须进入最终风险项。
- 预测满足需求不等于真实实验通过。
- 输出名称统一为“验证结果”，动作库统一称为“降本动作库”。
