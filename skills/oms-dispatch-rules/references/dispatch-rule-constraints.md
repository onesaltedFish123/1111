# Dispatch Rule Constraints

这个文档汇总了当前仓库里已经写死的分仓规则约束，生成 payload 前要先过一遍。

来源主要是：

- `OrderDispatchPageDOServiceImpl`
- `OrderDispatchCustomRuleServiceImpl`

## 1. 默认初始化行为

如果 merchant 没有任何规则，`queryDispatchRule(...)` 会先调用 `firstInitDispatchRules(...)` 自动创建一页默认规则。

默认页初始启用的规则只有：

- `EXCEPTION_BACKUP`
- `CONFIG_AUTO_CREATE_PRODUCT_IF_NOT_EXISTS`

这意味着：

- 新 merchant 不一定一开始就有完整的页级分仓策略
- 解释线上现状时，要考虑“第一次查询自动初始化”的副作用

## 2. Reset 规则的硬约束

`resetRules(...)` 会先做单选校验，再删除 merchant 现有全部页面和规则，再按新组合重建。

其中这三组不能为空且只能有一个值：

- `itemLineDispatches`
- `exceptionItemLineHandlers`
- `resultHandlers`

如果为空，会抛 `RULES_IS_EMPTY`。  
如果多于一个，会抛 `RULES_MORE_THAN_ONE`。

## 3. 页面保存时的硬约束

`checkRepeatedRulePage(...)` 里至少有这些规则：

### backup 必须且只能启用一个

启用的 backup 项数量 `!= 1` 会报错。

### delivery generate 不能多于一个

启用的 delivery generate 项数量 `> 1` 会报错。

### pageName 不能重复

同 merchant 已存在的页面名如果包含当前 `pageName`，会报重复。

### default page 只能有一个

如果已有默认页，再新增/更新一个 `isDefault=true` 的页面，会报错。

### 订单类型配置组合不能重复

代码会检查以下三个配置是否与已有页面重复：

- `CONFIG_SAMPLE_ORDER`
- `CONFIG_CROSS_BORDER_ORDER`
- `CONFIG_REPLACEMENT_ORDER`

如果与已有页面形成重复组合，会抛 `DUPLICATE_ORDER_TYPE_PLEASE_MODIFY`。

注意：当前实现是按位比较这三个布尔位，只要某一位与已有页冲突就可能触发重复判定，因此生成建议时要尽量先看已有页面。

## 4. Custom Rule 的现实限制

### 仓库查询当前不可直接依赖

`queryMerchantWarehouses(...)` 当前实现直接返回空列表，原本的远程调用逻辑被注释掉了。

### 渠道查询当前不可直接依赖

`queryMerchantChannels(...)` 当前实现也直接返回空列表，原本的远程调用逻辑被注释掉了。

### ship service 可以辅助补码

`queryShipService(...)` 会通过 `shipServiceService.vagueQuery(...)` 返回 `shipServiceCode` 列表，可以作为用户输入 ship service 时的补全手段。

### 操作符字段虽存在，但仓库里未在当前片段看到统一枚举

这些字段都存在长度限制 `max=2`：

- `channelOp`
- `itemOp`
- `carrierOp`
- `shipServiceOp`

但当前已读代码里未看到它们的统一枚举或翻译逻辑，所以如果用户没有提供现成约定，不要自作主张写入某个操作符。

## 5. Special Rule 的限制

`OrderDispatchSpecialRuleDTO.status` 被约束为：

- 必填
- 取值只能是 `0` 或 `1`

`ruleCode` 本身需要来自既有业务约定，当前片段里没有完整枚举。

## 6. 推荐回答策略

当信息不全时：

- 先输出“按当前代码能确认的 payload 骨架”
- 明确标出缺失项
- 明确标出“需要真实线上数据校验”的部分

当信息明显违反约束时：

- 不直接给最终 payload
- 先指出冲突原因
- 再给一个满足约束的修正版建议
