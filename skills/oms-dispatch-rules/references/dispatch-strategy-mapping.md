# Dispatch Strategy Mapping

这个文档把用户的自然语言需求映射到仓库里真实存在的分仓接口、模型和枚举。  
默认不要把这里的控制器、接口路径和模型名直接展示给客户。  
其中 `Custom Rule` 相关内容仅用于历史配置排查，不作为现行对客能力。

## 1. 控制器入口

来源：

- `oms/linker-module-oms-opc/linker-module-oms-opc-biz/src/main/java/com/unis/linker/module/opc/controller/app/NewOrderRoutingController.java`
- `oms/linker-module-oms-opc/linker-module-oms-opc-biz/src/main/java/com/unis/linker/module/opc/controller/rpc/route/OrderRoutingController.java`

### 规则页

- `GET /routing/v2/rules`：查询 merchant 的所有规则页
- `POST /routing/v2/rule`：新增或更新规则页
- `DELETE /routing/v2/rule`：删除规则页

请求模型：`DispatchRuleVO`

### Legacy: Custom Rule（已弃用，不对客展示）

- `POST /routing/v2/custom-rule`：新增 custom rule
- `PUT /routing/v2/custom-rule`：更新 custom rule
- `DELETE /routing/v2/custom-rule/{id}`：删除 custom rule
- `GET /routing/v2/custom-rule`：分页查询 custom rule
- `GET /routing/v2/customRule/shipService`：模糊查询 ship service code

请求模型：`AddCustomRuleReqVO`、`UpdateCustomRuleReqVO`

仅在排查历史配置、兼容行为或存量数据时参考；不要作为新方案推荐。

### Special Rule

- `POST /routing/v2/special-rule`

请求模型：`OrderDispatchSpecialRuleDTO`

### Reset 默认规则

- `POST /routing/reset-dispatch-rules`

请求模型：`ResetRuleVO`

## 2. 页级规则如何映射

来源：`DispatchStrategyEnum`

### 仓库过滤

- “按 ZIP / postcode 分仓” → `ZIPCODE`
- “目的国仓内优先” / “留在目的国市场” → `COUNTRY`

### item line 分配

- “必须单仓发完” / “不允许拆单” → `NO_SPLIT`
- “允许拆单” / “尽量拆单满足” → `MINIMAL_SPLIT`
- “样品不能拆” → `SAMPLE_NO_SPLIT`
- “按 accounting code 指定仓” → `SPECIFY_WAREHOUSE`
- 历史上“命中 custom rule 后指定仓” → `CUSTOM`（已弃用，不作为新方案推荐）

### item line 分配附加策略

- “从最近仓发货” → `CLOSEST_WAREHOUSE`
- “按 SKU 指定仓” → `SKU_SPECIFY_WAREHOUSE`

### 异常兜底

- “库存不足时指向最高优先级仓” → `ONE_WAREHOUSE_BACKUP`
- “库存不足时允许多仓兜底” → `MULTI_WAREHOUSE_BACKUP`
- “库存不足直接异常” → `EXCEPTION_BACKUP`

### 生单结果

- “一个仓生成一个 DO” → `ONE_WAREHOUSE_ONE_DELIVERY`
- “一件一单” → `ONE_ITEM_ONE_ORDER`
- “指定承运商行单独出 dispatchNo” → `SPECIFY_CARRIER_DELIVERY`

### 其他配置

- “产品不存在时自动建品” → `CONFIG_AUTO_CREATE_PRODUCT_IF_NOT_EXISTS`

### 订单类型标签页

- “样品单规则页” → `CONFIG_SAMPLE_ORDER`
- “跨境单规则页” → `CONFIG_CROSS_BORDER_ORDER`
- “replacement 单规则页” → `CONFIG_REPLACEMENT_ORDER`

## 3. `DispatchRuleVO` 组织方式

用于 `POST /routing/v2/rule`

```json
{
  "pageId": 123,
  "merchantNo": "M10001",
  "pageName": "Sample Cross Border Rule",
  "isDefault": false,
  "ruleItems": [
    { "ruleId": 52, "ruleName": "CONFIG_CROSS_BORDER_ORDER", "switchOn": true },
    { "ruleId": 51, "ruleName": "CONFIG_SAMPLE_ORDER", "switchOn": true },
    { "ruleId": 11, "ruleName": "NO_SPLIT", "switchOn": true },
    { "ruleId": -3, "ruleName": "EXCEPTION_BACKUP", "switchOn": true }
  ]
}
```

说明：

- `pageId` 为空可表示新增；非空表示更新
- `ruleName` 主要用于可读性，真正校验依据是 `ruleId`
- `switchOn` 表示该 rule item 是否启用
- 页级规则最终落地时，仍会通过 `DispatchStrategyEnum` 驱动运行时 dispatch 逻辑

## 4. Legacy: `AddCustomRuleReqVO` 映射方式（仅历史排查）

该能力已弃用，不适合继续作为新方案输出。  
只有在排查历史“满足条件时优先指定某仓”的存量配置时，才参考本节。

关键字段：

- 基础字段：`merchantNo`、`ruleName`、`warehouseId`、`tenantId`、`status`
- 渠道条件：`channelId`、`channelStatus`、`channelOp`
- 件数条件：`itemQty`、`itemStatus`、`itemOp`
- 承运商条件：`carrierJson`、`carrierStatus`、`carrierOp`
- ship service 条件：`shipService`、`shipServiceStatus`、`shipServiceOp`

建议映射方式：

- 用户明确说“按渠道 XXX”时，再打开 `channelStatus`
- 用户明确说“件数大于/小于/等于 N”时，再打开 `itemStatus`
- 用户明确说“承运商为 XXX”时，再打开 `carrierStatus`
- 用户明确说“ship service 为 XXX”时，再打开 `shipServiceStatus`

注意：

- 仓库里只看到了这些操作符字段，但没有在当前代码片段里看到操作符枚举定义或解释逻辑
- 因此不要擅自发明 `channelOp` / `itemOp` 的合法取值；如果用户没有给出现成约定，要明确提示“操作符需沿用前后端既有约定”

## 5. `OrderDispatchSpecialRuleDTO` 映射方式

当用户只想启停一个特殊规则时，使用：

```json
{
  "merchantNo": "M10001",
  "ruleCode": "XXX_RULE_CODE",
  "status": 1
}
```

说明：

- `status` 在模型上被限制为 `0` 或 `1`
- `ruleCode` 的备选值需要来自现有业务约定；当前片段里未展示完整枚举来源

## 6. Reset 规则的映射方式

来源：`ResetRuleVO`

字段分组：

- `warehouseFilters`
- `itemLineDispatches`
- `exceptionItemLineHandlers`
- `resultHandlers`
- `otherConfigs`

这些字段接收的都是 `DispatchStrategyEnum.name()` 字符串，例如：

```json
{
  "merchantNo": "M10001",
  "warehouseFilters": ["ZIPCODE"],
  "itemLineDispatches": ["NO_SPLIT"],
  "exceptionItemLineHandlers": ["EXCEPTION_BACKUP"],
  "resultHandlers": ["ONE_WAREHOUSE_ONE_DELIVERY"],
  "otherConfigs": ["CONFIG_AUTO_CREATE_PRODUCT_IF_NOT_EXISTS"]
}
```

说明：

- reset 会按分组重建 merchant 的整套规则组合
- 更适合“把默认规则整体替换成一套新组合”的场景，不适合只改一两个局部条件

## 7. 生成回答时的推荐模板

当用户给出自然语言需求时，优先产出：

1. 这属于哪类规则
2. 建议配置方案或明确说明“该能力已弃用”
3. 对业务效果的解释
4. 哪些信息仍缺失
5. 哪些地方可能被后端校验拒绝
6. 仅在明确要求时再附内部模型或 JSON 草案
