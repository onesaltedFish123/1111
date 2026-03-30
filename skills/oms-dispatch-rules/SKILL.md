---
name: oms-dispatch-rules
description: 用于根据当前仓库中的 OMS 分仓代码，把自然语言需求转换成可执行的规则配置方案。适用于新增/修改/删除分仓规则页、创建 custom rule、开关 special rule、重置默认规则，或需要判断某个分仓需求应该落到哪个接口与 payload 的场景。
---

# OMS Dispatch Rules

## Overview

这个 skill 不是发明新规则，而是严格按仓库现有实现，把用户需求映射到 OMS 规则配置接口、运行时规则枚举和对应模型。  
它默认先做“意图识别 + 约束校验 + payload 生成”，只有在用户明确确认后，才建议执行真实配置变更。

## When To Use

在以下场景优先使用这个 skill：

- 用户说“帮我新增一个分仓规则页 / 默认页 / 样品单规则 / 跨境规则”
- 用户说“按渠道、承运商、ship service、件数建一个 custom rule”
- 用户说“打开/关闭某个 special rule”
- 用户需要把口头需求翻译成接口、字段和 JSON
- 用户想先知道当前规则配置是否会违反后端校验

## Supported Tasks

### 1. 分仓规则页 CRUD

对应控制器 `NewOrderRoutingController`：

- `GET /routing/v2/rules`
- `POST /routing/v2/rule`
- `DELETE /routing/v2/rule`

当用户描述的是“默认页 / 样品单页 / 跨境页 / 替换单页 / 某套 page rule 组合”时，优先落到 `DispatchRuleVO`。

### 2. Custom Rule CRUD

对应控制器：

- `POST /routing/v2/custom-rule`
- `PUT /routing/v2/custom-rule`
- `DELETE /routing/v2/custom-rule/{id}`
- `GET /routing/v2/custom-rule`

当用户描述的是“满足某些条件时指定仓库”时，优先落到 `AddCustomRuleReqVO` / `UpdateCustomRuleReqVO`。

### 3. Special Rule 开关

对应控制器：

- `POST /routing/v2/special-rule`

当用户只是在切换某个 `ruleCode` 的启停状态时，落到 `OrderDispatchSpecialRuleDTO`。

### 4. Reset 默认规则

对应控制器 `OrderRoutingController`：

- `POST /routing/reset-dispatch-rules`

如果用户表达的是“把 merchant 的整套默认规则重置成一份新组合”，优先落到 `ResetRuleVO`。

### 5. 运行时规则映射与客户解释

如果用户不是要直接改配置，而是想知道：

- “这个自然语言诉求应该落到哪组 `DispatchStrategyEnum`”
- “这页规则最终会怎么影响 dispatch”
- “客户说的最近仓、指定仓、不拆单，在项目里分别对应什么”

就优先结合：

- `references/dispatch-strategy-mapping.md`
- `references/dispatch-rule-constraints.md`

把自然语言映射成页级规则、custom rule、special rule 和运行时分仓能力的组合说明。

## Workflow

### Step 1. 识别需求属于哪类规则

先把自然语言归类为以下四类之一：

- `page rule`：规则页级别配置，使用 `DispatchRuleVO`
- `custom rule`：条件命中后指定仓，使用 `AddCustomRuleReqVO` / `UpdateCustomRuleReqVO`
- `special rule`：按 `ruleCode` 启停，使用 `OrderDispatchSpecialRuleDTO`
- `reset rules`：整套默认规则重置，使用 `ResetRuleVO`

不要把“页级策略”和“custom rule 条件仓”混在一个 payload 里。

### Step 2. 提取必要字段

按类型最少要补齐这些字段：

- `page rule`：`merchantNo`、`pageName`、`isDefault`、`ruleItems`
- `custom rule add`：`merchantNo`、`ruleName`、`warehouseId`
- `custom rule update`：`ruleId`
- `special rule`：`merchantNo`、`ruleCode`、`status`
- `reset rules`：`merchantNo` 和各规则枚举分组

如果用户只给了“仓库名”“渠道名”，不要臆造 ID。先说明仓库里的仓库/渠道查询实现目前是空返回，再要求用户补 `warehouseId`、`channelId`。

### Step 3. 映射到代码中的规则枚举和字段

先加载：

- `references/dispatch-strategy-mapping.md`
- `references/dispatch-rule-constraints.md`

映射原则：

- 页级规则用 `DispatchStrategyEnum`
- 条件路由用 `AddCustomRuleReqVO` / `UpdateCustomRuleReqVO` 的布尔开关、操作符和值
- 特殊规则只改 `ruleCode + status`

如果某个自然语言诉求在仓库代码里找不到对应字段或枚举，要直接说明“当前代码里未看到支持”，不要自造字段。

### Step 4. 做执行前校验

在给出 payload 之前，至少检查：

- backup 策略是否恰好选中 1 个
- delivery generate 策略是否超过 1 个
- page name 是否重复
- default page 是否超过 1 个
- 订单类型配置组合是否与已有页面冲突

如果用户没有提供现有页面列表，先给“静态校验结果 + 仍需线上现状确认的项”。

### Step 5. 输出可执行方案

默认按下面结构回答：

1. 需求理解
2. 推荐接口
3. 推荐 payload
4. 命中的代码约束 / 风险
5. 执行前需要用户确认的点

如果用户直接要求“帮我写请求体”，也仍然要附上至少一句风险提示。

## Output Rules

- 优先输出中文，字段名保持代码原样
- JSON 只使用仓库里真实存在的字段
- 不要捏造枚举值、操作符集合或额外接口
- 如果缺字段，先提最少问题，再生成 payload
- 如果执行存在破坏性，先确认再继续

## Important Limits

- `queryMerchantWarehouses(...)` 和 `queryMerchantChannels(...)` 当前实现返回空列表，不能依赖它们自动补齐仓库/渠道 ID
- `queryShipService(...)` 可用于模糊查 ship service code
- `CONFIG_REPLACEMENT_ORDER` 参与页面组合校验，但在当前选页逻辑里没有看到与 sample/cross-border 同级的命中分支，解释规则效果时要区分“可配置”与“当前选页是否实际使用”
- reset 默认规则已有明确入口 `POST /routing/reset-dispatch-rules`，不要再说“项目里没看到公开入口”

## References

- `references/dispatch-strategy-mapping.md`
- `references/dispatch-rule-constraints.md`
