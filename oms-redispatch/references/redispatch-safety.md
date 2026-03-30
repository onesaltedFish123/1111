# Redispatch Safety

这个文档描述当前仓库里已找到的重分仓执行入口，以及它们真正会做什么。

主要来源：

- `OrderRoutingAdminController`
- `ReDispatchVO`
- `DTS-5740_2025-06-11.md`
- `DTS-8536_2025-11-20.md`
- `OrderTimelineNodeNameEnum`

## 1. 控制器入口

控制器路径前缀是 `/routing`。

### DO 重分仓

- `POST /routing/re-dispatch-do`

### SO 重分仓

- `POST /routing/re-dispatch-so`

请求模型都是：

```json
{
  "orderNoList": ["ORDER_NO_1", "ORDER_NO_2"]
}
```

虽然 `ReDispatchVO` 还有：

- `orderNo`
- `merchantNo`
- `tenantId`

但当前控制器实现主要使用的是 `orderNoList`。

## 2. DO 重分仓会做什么

`reDispatchDo(...)` 的实际行为：

1. 根据 `orderNoList` 查询 `DeliveryOrderDO`
2. 删除这些订单号对应的 `OrderDispatchDO`
3. 对每个 DO：
   - 取出 `merchantNo`
   - 调用 `deliveryOrderService.getReqVO(merchantNo, orderNo)`
   - 再调用 `deliveryOrderService.process(...)`

这意味着：

- 它会真的重跑 DO 侧的处理流程
- 不是只做分析
- 会修改已有分仓相关数据

## 3. SO 重分仓会做什么

`reDispatchSo(...)` 的实际行为：

1. 遍历 `orderNoList`
2. 通过 `salesOrderMapper.selectByOrderNo(orderNo)` 查 SO
3. 取 SO 的 `tenantId`，切换租户上下文
4. 删除该 SO 对应的：
   - `OrderDispatchDO`
   - `OrderDispatchItemLineDO`
5. 读取地址和订单明细，重建 `SalesOrderDispatchDTO`
6. 调用 `salesOrderDispatchService.executeOrderDispatch(...)`

重建 DTO 时会带上的关键字段包括：

- `zipcode`
- `country`
- `items`
- `carrierSCAC`
- `carrierName`
- `specifiedWarehouseAccountingCode`
- `channelId`
- `orderTypeEnum = SALES_ORDER`

这意味着：

- SO 重分仓会真的删除旧拆单结果并重新计算
- 不是幂等查询
- 如果订单上下文数据变了，结果也可能变化

## 4. 异常和返回行为

### DO 接口

- 有整体 `try/catch`
- 异常时返回 `RE_DISPATCH_FAILED`

### SO 接口

- 找不到单号时会 `continue`
- 最终仍返回原始 `orderNoList`

所以在技能回答里要提醒用户：

- “返回 success” 不一定代表每个单号都成功重分仓
- 最好补充建议用户核对实际处理结果

## 5. 技能的默认安全策略

当用户只说“给我一个重新分仓建议”时，默认不要直接进入执行模式。  
标准做法是先输出：

1. 这更像规则问题还是单单重跑问题
2. 应走 SO 还是 DO
3. 执行后会删除什么旧数据
4. 最终请求体
5. 一句明确确认语

推荐确认语：

> 当前代码路径会删除旧的 dispatch 记录并重新执行分仓，这不是 dry-run。确认后再执行。

## 6. 执行中场景的额外约束

根据 `DTS-5740_2025-06-11.md`，很多“执行中改仓”场景不能直接理解成“只调用 re-dispatch 接口”。

已确认的解除分仓前置条件：

- 订单状态必须为 `Allocated` 或 `Warehouse Processing`
- 不能存在已发货、部分发货或 short shipped 的子单
- 需要具备解除分仓操作权限

已确认的解除分仓后果：

- 主订单状态会更新为 `Deallocated` 或 `Cancelling`
- 相关 dispatch 记录会被更新、删除或进入取消流程
- 可能向 WMS 发送取消请求

所以当用户说“执行中重新分仓”时，默认要补一句：

> 这更像“解除分仓 / 取消后再重分仓”的组合流程，而不是单纯重新跑一次 dispatch。

## 7. 合单取消后重新拆单

根据 `DTS-8536_2025-11-20.md` 与 `CommonServiceImpl.excuteDispatchByDispatchDOListByMergeOrder(...)`：

- 合单取消后，系统会识别需要重新拆单的订单
- 订单状态会更新为 `DEALLOCATED`
- 再重新执行分仓生成新的 dispatch
- 失败时会记录错误日志，并可能标记为需要人工处理

这类场景回答时要优先说：

- 这是“合单取消后重新拆单”
- 不是普通的单据级 redispatch
- 需要关注取消记录、订单状态和后续人工处理

## 8. 时间轴与结果说明

`OrderTimelineNodeNameEnum` 中已确认：

- `SALES_ORDER_REALLOCATION_SUCCESSFULLY`
- `SALES_ORDER_REALLOCATION_FAILED`

因此可以明确告诉用户：

- 项目里对重分仓成功和失败都有独立时间轴节点
- 重分仓不仅是技术动作，也会形成业务轨迹

## 9. 场景建议矩阵

| 场景 | 推荐回答 |
| --- | --- |
| 已分仓、未发货，需要重新计算仓库 | 优先评估直接 `re-dispatch-so` 或 `re-dispatch-do` |
| 已进入 `Warehouse Processing` | 优先说明解除分仓或取消流程，再决定是否重分仓 |
| 合单取消后需要重新拆单 | 引导到合单取消后的重新拆单路径 |
| 已发货、部分发货、short shipped | 默认不建议直接重分仓，提示高风险 |
