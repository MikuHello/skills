# 用量数据与账单核对

用本页从已有截图、日志或账单取数，反推额度，核对实际扣费。

## 取数与字段

### 按目标选字段

```text
查用量   -> scope + pool
反推额度 -> scope + meter + (deduction 或 usage 的官方价)
算套餐价 -> scope + payment.paid + pool.limit + 平台扣额或费率
算中转价 -> deduction.amount（实际货币），或 deduction.amount + payment.paid + payment.credited
算倍率   -> 实际或折算成本 + 同量官方原价
算产出成本 -> 已算单批成本 + output.quantity
```

用时间或请求标识对齐同一批请求、账单和仪表变化。按需核对共享产品、附加任务、重试、超额扣费、重置和退款。归因不清时，标明差值可能包含的项目。

### 字段定义与单位

`?` 表示按需取值。

```text
Amount = {value: decimal, unit: string}
Rate   = {charge: Amount, per: Amount}

scope   = {platform, plan?, model_id?, billing_period?, region?, mode?}
payment = {paid: Amount, credited?: Amount, price_phase?: first_purchase | renewal | regular}
pool[]  = {id, window, reset_kind?, resets_at?, limit?: Amount, used?: Amount, remaining?: Amount}
meter   = {pool_id, before: decimal, after: decimal, unit, shows: used | remaining, step?, from_at, to_at}
usage[] = {kind, quantity: Amount, modality?, tier?, platform_rate?: Rate, official_list?: Rate, official_current?: Rate}
deduction = {amount: Amount, pool_id?, at?, request_id?}
relay?  = {advertised_multiplier?, advertised_basis?}
output  = {quantity: Amount, specs?}
source[] = {fields, reference, as_of?}
fx?     = {from, to, rate, as_of, source}
```

- **计费项。** `kind` 可取未缓存输入、缓存读取、缓存创建、输出、存储、工具调用或运行时长。
- **金额与数量。** 实际货币用 `USD`、`CNY` 等。站内额度按平台定义记录，包括站内显示的 `$`。数量用 `token`、`request`、`image`、`second` 等。积分标明所属平台和适用范围。
- **仪表。** 百分比用 0–100，`step` 是显示步长。

## 从仪表反推额度

对**同一窗口、同一额度池**内可归因的百分比变化计算。`deduction.amount` 和 `official_cost_of_usage` 使用同一批请求。

```text
pct_used(p) = p                     if meter.shows == used
            = 100 - p               if meter.shows == remaining
used_delta  = (pct_used(meter.after) - pct_used(meter.before)) / 100
native_cap  = deduction.amount.value / used_delta
api_equiv   = official_cost_of_usage / used_delta
```

用正的增量估算。**原生额度**用实际扣额计算。**API 等值额度**随样本的模型、模式与用量结构变化。要估单模型独占上限，使用单模型样本或已知模型权重。

> **示例**　官方 API 等值 $20 的请求让周表增加 20 个百分点，周额度约 $100。若两次读数都四舍五入到整百分比，增量约 19–21 个百分点，对应 $95–$105。若同批扣了 400 credits，原生周额度约 2,000 credits。

增量下界接近零时，额度上界难以确定。

## 核对中转账单与缓存

### 充值与实际扣费

先换算扣额单位，再用站方声明的价表、分项倍率和充值规则复算站标差异。仅有单笔合计时可以算该批的整体倍率。要反推输入、输出和缓存费率，使用分别改变各计费项的多组可比账单。

> **示例**　实付 ¥100 到账 1,000 站内单位，扣 1 单位对应 ¥0.10。同量官方原价是 ¥7 时，实测倍率约 `0.0143×`。

### 原价与促销价

假设同量实付 $0.04，厂商原价 $0.50、当期价 $0.40，则对原价和当期价的倍率分别为 `0.08×` 和 `0.10×`。有证据时注明站方实际采用的价表。

### 缓存费用与命中差异

- **对齐计费项。** 确认输入总量是否包含缓存读取或创建，输出是否已含推理 Token。按文档记录缓存门槛、读取、创建、存储费用、TTL 和共享范围，标注未知项。
- **对照已有样本。** 冷样本用新前缀，热样本复用同一足够长的前缀并改末尾，破坏前缀的样本修改前段。对齐模型、路由、时间和输入输出量，再比较缓存读取与实际扣款。
- **判断差异。** 有限且稳定的命中可能来自固定系统前缀。账单反复与公开规则不符，可报告可复现的计费差异。判断上游实际命中或转发需要对应的上游用量或可核对记录。

## 图像与视频产出成本

用同一平台、同一适用范围的积分和实际扣分计算。分开列订阅积分、赠额、充值包和媒体专用积分，并注明失败或重试扣费是否纳入。

> **示例**　¥1,200 年付套餐每月发 1,000 点，折合 ¥100/月。5 秒视频扣 30 点，满额折算为 ¥3/条、¥0.60/秒。
