# Rule Selection Logic

这个文档描述当前仓库里“订单到底会命中哪一套分仓规则页”的核心逻辑，主要来自 `OrderDispatchPageDOServiceImpl.getMostSuitableRules(...)`。

## 1. 输入条件

当前方法会使用这些输入来判断规则页：

- `merchantNo`
- `orderNo`
- 收货地址里的 `country`
- 订单商品 SKU 列表

它会进一步通过商品查询拿到 tags，判断是否包含 `SAMPLE`。

## 2. 先判断订单特征

### 是否跨境单

判断方式很直接：

- 如果收货国家不是 `USA` 或 `US`
- 则视为 `isCrossBoarderOrder = true`

### 是否样品单

会根据订单 SKU 去查商品信息，然后取 tags。  
只要 tags 里包含 `SAMPLE`，就视为 `isSampleOrder = true`。

## 3. 再取 merchant 的规则页

逻辑会先查询 merchant 的所有规则页：

- 如果没有规则，会先自动初始化默认页
- 然后把页面分成 `defaultRule` 和 `otherRules`

如果只有默认页，就直接返回默认页中所有 `switchOn=true` 的 ruleId。

## 4. 非默认页的命中顺序

当前代码没有看到独立的“优先级字段”。  
方法会遍历 `otherRules`，一旦命中就 `break`，也就是“遇到第一个符合条件的非默认页就停”。

已实现的命中分支只有：

### 同时命中样品单 + 跨境单

页面里同时启用了：

- `CONFIG_SAMPLE_ORDER`
- `CONFIG_CROSS_BORDER_ORDER`

且订单本身同时满足样品单和跨境单时，选这页。

### 只命中样品单

页面启用了：

- `CONFIG_SAMPLE_ORDER`

且订单是样品单时，选这页。

### 只命中跨境单

页面启用了：

- `CONFIG_CROSS_BORDER_ORDER`

且订单是跨境单时，选这页。

### 其他情况

回落到默认页。

## 5. 需要特别注意的事实

### `CONFIG_REPLACEMENT_ORDER` 当前主要出现在校验逻辑

已读代码里，它参与了页面组合唯一性校验。  
但在 `getMostSuitableRules(...)` 当前分支中，没有看到独立的 replacement 命中逻辑。

所以：

- 可以说“replacement 配置存在”
- 不能直接说“当前选页一定会按 replacement 命中”

### 代码里没有 dry-run 接口

规则页命中的解释可以基于这个方法推断，  
但当前仓库里没看到单独暴露给外部的“模拟一次分仓结果”接口。

## 6. 用这个逻辑做“重分仓建议”的方式

当用户问“这单要不要重分仓”时，先按下面思路回答：

1. 这单理论上会命中默认页、样品页还是跨境页
2. 如果预期与现状不符，是规则设计问题还是单单重跑即可
3. 如果只是单据数据修正后需要重算，可建议 re-dispatch
4. 如果根因是规则页不对，应先改规则，再决定是否补跑历史单
