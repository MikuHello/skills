---
name: token-usage-pricing
description: 核算 AI 套餐和 API 的用量、额度、成本与倍率。适用于查询套餐用量、从截图或日志反推周/月配额、买前估算、中转站充值与扣费对账、核对缓存计费，以及决策模型、语言模型、多模态、图像按张、视频按秒、按次和积分计费。
---

# Token 用量与成本核算

## 核算流程

1. **确定对象与取数。** 确认平台、产品、模型和要算的结果。从账号数据、截图、账单、日志或文字中提取[对应字段](references/measurement.md#按目标选字段)。查公开费率和套餐规则时使用[官方来源与查询入口](references/sources.md)。
2. **统一计价口径。** 对齐币种、站内单位、账期、模型版本、服务档和计费项。按原始字段定义拆分未缓存输入、缓存读取、缓存创建、输出与模态，避免重复计数。长上下文按模型阈值与整请求或分段规则计价。
3. **计算并标注依据。** 按下方规则核算。用百分比仪表时查[额度反推](references/measurement.md#从仪表反推额度)，核对中转账单时查[充值与实际扣费](references/measurement.md#充值与实际扣费)。优先给能算出的值，证据够时给区间。来源冲突时写明各自口径。

## 计算规则

### 成本与倍率

- **额度**是特定池和窗口里的可用量，单位可以是 credits、站内金额、Token、次数或积分。**API 等值额度**是把对应模型用量按指定官方 API 价格计出的金额。
- **倍率（折扣）**是实际或折算成本除以同量官方 API 原价。`0.10×` 表示成本为原价的 10%。

```text
line_cost(qty, rate) = qty.value / rate.per.value * rate.charge.value
usage_cost          = sum(line_cost)  # 先对齐数量单位，再按 charge.unit 合计
plan_unit_cost      = payment.paid.value / pool.limit.value
plan_line_cost      = (扣除的原生额度) * plan_unit_cost
relay_cost          = deduction.amount.value * payment.paid.value / payment.credited.value
cost_per_output     = batch_cost / output.quantity.value
multiplier          = actual_or_allocated_cost / official_list_cost
```

- **计价单位。** Token 单价通常以每百万为基数，将 `rate.per` 写为 `{value: 1000000, unit: token}`。工具、时长、存储按各自单位算。
- **套餐折算。** 使用同一账期的实付和可用总额度。倍率用该账期实付作分子。满额倍率用满额的官方 API 等值作分母，实际倍率用实际用量的官方 API 等值作分母。积分套餐根据同类积分的单位成本和单批扣分算费用，再除以张数或总秒数。
- **中转扣费。** 账单已有实际货币扣款时，直接用该金额。复核站标倍率时还需站方基础价表与充值规则。
- **原价基准。** 倍率比较同量、同范围、同币种的官方原价。有当期官方促销时，同时给可核实的当期倍率和生效期。原价缺失或为零时，给可核实的单价或差异。

### 额度池与重置窗口

- **重置窗口。** 按实际存在的窗口给出额度、已用、剩余和重置时间。区分账单周期、发放周期、固定重置和滚动窗口，注明时区。
- **周与月。** 月付仅在周或短窗口重置时，优先给实际窗口。需比较月度用量时，用实际账期和重叠限制估计，或把 `周额度 × 30 ÷ 7` 标为 30 天折算量。周、月额度同时存在时，用官方月池算整期成本。
- **共享与专属。** 同一共享池内，`池用量比例 = Σ(模型用量 ÷ 该模型在同池同窗口的独占上限)`。模型专属池或附加上限单独列出。
- **跨月订阅。** 季付、半年付、年付按整期实付和实际发放规则计算。按月发放时列月配额和月均实付。整期一次发放时列整期额度，月均量标为平均值。首购与续费分别核算。
- **按量与预充值。** 列实际单价、可用余额、有效期和适用的速率限制。区分订阅、加购、赠送和超额扣费。

## 结果字段与呈现

### 结果字段

按计费单位选 Token 计价、按次与产出或混合计价。查套餐用量时使用公共字段和额度记录。`?` 表示按需填写。

```text
common = {scope, result, unit, basis, evidence, missing?}
quota[]? = {pool, window, limit?, used?, remaining?, resets_at?}

token[] = {kind, modality?, tier?, price_per_1m?, official_list_per_1m?, multiplier?, current_multiplier?}
per_unit[] = {quantity, unit_cost, billed_quantity?, specs?}
mixed = {items: [{kind, quantity, cost}], total, multiplier?, current_multiplier?}

evidence[] = {type: official | ledger | inferred | estimate, source, as_of?}
```

### 呈现顺序

**结果 → 口径 → 来源。** 数字紧跟单位。注明是官方公布、账单实测、截图反推还是公开估算。缺项只列会改变结论的字段。核中转时可并列站标、实测和差值。Tip 留给能改善计算或解读的提示。

默认在聊天中用短答、表格或明细呈现。需要机器可读时输出 JSON。需要对比页或分享图时，按[图像呈现](references/rendering.md)选择风格，将结果填入 HTML 模板，再按交付场景导出 PNG 或 SVG。
