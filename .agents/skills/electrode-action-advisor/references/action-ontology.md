# Action Ontology / 动作本体

## 1. 基本对象

- **Package**：完整设计状态，使用唯一 `package_id`。
- **Atomic Action**：单个设计字段的定向变化，记录位置、对象、操作、旧值、新值、变化量、单位。
- **Action Set**：`Diff(source_package, target_package)` 的全部原子动作，无默认时间顺序。
- **Evidence**：来源设计对、真实迭代或构造差分标记、上下文、相关实验/报告。

## 2. 组件层级

```text
package
├── cathode
│   ├── main_materials / auxiliary_materials / ratios
│   └── coating_weight / compaction_density / current_collector_thickness
├── anode_upper
│   ├── main_materials / auxiliary_materials / ratios
│   └── coating_weight / compaction_density
├── anode_lower
│   ├── main_materials / auxiliary_materials / ratios
│   └── coating_weight / compaction_density
└── cell_structure
    └── layer_count / substrate_thickness / geometry
```

后续可扩展电解液、隔膜、工艺条件和测试工况。字段实际含义以生产数据字典为准。

## 3. 操作类型

| 操作 | 含义 | 例子 |
| --- | --- | --- |
| ADD | 原设计无此材料，目标设计新增 | 添加石墨 B |
| REMOVE | 原设计有此材料，目标设计移除 | 去除添加剂 C |
| INCREASE | 数值或比例增加 | 上层 B +5pp |
| DECREASE | 数值或比例降低 | 铝箔厚度 -2μm |
| REPLACE | 有明确对象对应关系的替换 | NCM811 → NCM9 |
| SET | 非数值状态或参数设定改变 | 工艺模式变更 |

不要仅凭一个材料消失、另一个材料出现就断言它们是替换关系。配比单位是百分点（pp），与相对百分比变化不同。

## 4. 原子动作字段

`component, field_path, material_id, operation, old_value, new_value, delta_absolute, delta_relative, unit, action_template`

动作模板用于模糊匹配和统计，动作实例用于精确重建设计。

## 5. 动作关系

- `CO_MODIFIED`：两个动作在同一差分集合出现；仅统计共现。
- `COMPENSATES`：有额外模型、机理或实验支持的补偿关系。
- `REQUIRES`：由明确约束支持的必要联动。
- `CONFLICTS_WITH`：动作之间存在互斥或约束冲突。

## 6. 数据质量原则

- 区分真实历史迭代和虚拟 Package 差分。
- 保留材料 ID、层位、配比基准、原始单位和缺失值语义。
- 共享 Package 的设计对不是独立实验样本。
- 完整 Action Set 不得退化成仅有原子动作共现边。
