# [TDD] 游戏状态与 Buff/Debuff 系统技术设计文档

## 系统概述与设计目标

- 目标：建立基于 UE GAS 的低耦合、易扩展的状态控制与 Buff 管理系统。
- 核心哲学：完全通过 GE 承载数据与效果，其作为buff/debuff，GA 仅作为状态载体(不同的状态就是不同buff/debuff的组合)，Tag 作为全局解耦纽带。

## 核心架构规范

### 职责划分

- GA：作为状态控制器（State Container）。
- GE：作为具体效果载体（Effect/Modifier）。
- Tag：状态标识与解耦媒介。

### GE 拆分判定准则

任何复合状态，只要触发以下条件之一，必须拆分为不同 GE 由 GA 组合挂载：
1. 生命周期（Duration）不一致；
2. 驱散逻辑（Dispel）需独立；
3. 叠层逻辑（Stacking）不一致。

### 驱散与清除规范

- 统一使用 Instant 类型的 GE 配合 `Remove Gameplay Effects with Tags` 驱动。
- 严禁在 GA 蓝图中直接调用 Remove Active Gameplay Effects with Granted Tags 节点手动拔除 GE，必须保持“效果推拉均由 GE 决定”的纯粹管线。

## 数据流与结构定义

- Gameplay Tag 命名规范：Granted Tag:`State.Debuff.Poison` / Asset Tag:`Effect.Cure.Poison`。

## 边缘情况与异常处理 (Edge Cases)

- 死亡时清理：玩家死亡激活 `GA_Death` 时，清理身上的所有Active GE。
- 存档恢复：所有状态恢复到存档时的具体情况，即读档后保留角色所有buff/debuff。

## 叠加策略

1. 可叠加状态皆采用共享计时，即所有叠加层的持续时间相同，当时间一到整个状态消失，而不是每层独立计时，即一层一层掉。
2. 叠满层数后，再受到状态施加时，不会增加新的叠加层，而是刷新当前叠加层的持续时间。
3. 属性修改是按层数线性叠加

## Tag 状态矩阵与免疫逻辑

1. Granted Tags：当前 Buff 赋予角色的状态 Tag（如 State.Frozen）。
2. Blocked Tags：阻断哪些 Tag 的挂载（如带有 State.SpellShield 时，阻断任何 Debuff.* 挂载并消耗 Shield）。
3. Remove Tags：挂载时自动清理的 Tag（如应用 GE_Cure 移除 State.Poisoned）。

### 免疫机制（Immunity Standard）规范

本系统统一采用【方案 A：直接拦截丢弃（Hard Immunity）】作为默认规则，特定的数值减免采用【方案 B：动态计算（Soft Resistance）】。

#### 规则 1：完全免疫（丢弃 GE）

- **定义**：当目标拥有某种免疫状态时，同类型的 GE 拒绝挂载，并向 UI 广播 `Event.UI.FloatingText.Immune` 事件，弹字“免疫”。
- **配置标准**：统一在 GE 的 `Application Tag Requirements -> Application Blocked Tags` 中配置对应的免疫 Tag（如 `State.Immune.Poison`）。
- **适用场景**：硬控免疫（如免疫眩晕、免疫冰冻）、完全毒素免疫、驱散后的短暂无敌。

#### 规则 2：动态抗性/减免（正常挂载/ Modifier 计算）

- **定义**：GE 正常挂载以维护其生命周期与 Granted Tag，但数值修改器根据目标的属性（Attribute）动态计算。
- **配置标准**：使用 MMC (Modifier Magnitude Calculation) 动态读取目标的 `Attribute.PoisonResistance` 属性进行乘算。若抗性为 100%，计算结果修正为 0，但不阻断 GE 挂载。
- **适用场景**：百分比护甲/抗性减伤（鸭科夫的抗毒机制是削弱玩家角色受到的毒伤害，而不是直接免疫）、动态霸体（只挡减速数值，不拔 Debuff）。

两者最大的区别是：
- **方案 A**：直接拦截丢弃，不计算数值（完全免疫毒素buff存在时，中毒debuff挂不上玩家角色身上）。
- **方案 B**：正常挂载，根据属性动态计算数值（抗毒状态结束前后中毒debuff都可以挂上玩家角色身上，只是受到的中毒debuff的伤害不同）。

