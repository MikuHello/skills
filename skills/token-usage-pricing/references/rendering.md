# 图像呈现

[卡片入口](../assets/card.html)根据语义化数据选择并生成原版布局。

| 数据 | 布局 |
| --- | --- |
| 1 个套餐、1 个模型 | 单套餐单模型 |
| 1 个套餐、2–4 个模型 | 单套餐多模型 |
| 2–4 个套餐，每套餐 1 个对比模型 | 多套餐比较 |

Agent 可按场景选择风格，用户指定时优先沿用。`white` 素白和 `night` 夜幕是 **1080 × 1350**，`field` 色域是 **1600 × 900**。省略 `theme` 或使用 `auto` 时，上述三种布局分别默认使用素白、夜幕、色域。四模型含费率时自动选素白，明确选夜幕时分卡。

## 输入

将 `assets/card.html` 复制到输出路径，用数据替换 `#card-data` 中的 `null`，浏览器打开即显示卡片。编程时调用 `window.tokenUsageCard.generate(data)` 获取独立 HTML 和画布尺寸。

```text
card  = {theme?: auto | white | night | field, title?, labels?, ratioUnit?, plans[]}
plan  = {product, tier?, fee?, models[]}
model = {model, provider?, ratio?, quota?[], prices?[]}
quota = {label, value, window?}
price = {label, plan?, official?}
```

- 倍率填写数值因子 `0.05`。默认显示 `0.5折`，`ratioUnit: "×"` 显示 `0.05×`。金额保留币种。费率标题和单位按需填 `labels.prices`、`labels.unit`。
- 按真实重置窗口从短到长填写额度。夜幕刻度用同币种、可线性比较的金额。同卡比较采用一致的倍率与费用口径。
- 多模型与多套餐超过单张容量时分卡。素白和色域的多模型卡使用实际月额度，缺月额度时留空。自定义月额度标签可加 `window: "month"`。夜幕单模型或多套餐把月费展示在刻度上，有月费时需提供正值额度。未入卡的费率或其他窗口额度随正文给出。程序写入 JSON 时把 `<` 转义为 `\u003c`。

## 导出与路径

- **HTML。** 交付填好的入口副本，或保存 `generate(data).html`。按用户指定路径保存，省略时使用当前环境的输出目录。
- **图片。** 等待 `html[data-ready="1"]`。按画布宽高截图。复杂 CSS 优先 PNG，采用 SVG 时核对字体与样式。截图工具支持路径时可直接指定。网页下载则按浏览器的下载设置保存。
- **字体。** 沿用原版 Inter 和思源黑体 CDN，离线按原字体栈回退。运行只需要浏览器。
