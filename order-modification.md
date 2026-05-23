# Vendure 订单修改（Order Modification）链路分析

## 一、整体架构概览

管理员修改既有订单的链路涉及以下核心模块：

| 层级 | 主要职责 | 关键文件 |
|------|----------|----------|
| GraphQL API 层 | 定义接口契约与权限控制 | `packages/core/src/api/schema/admin-api/order.api.graphql` |
| Resolver 层 | 请求路由、事务管理、参数校验 | `packages/core/src/api/resolvers/admin/order.resolver.ts` |
| Service 层 | 业务编排、前置校验 | `packages/core/src/service/services/order.service.ts` |
| Helper 层 | 核心修改逻辑实现 | `packages/core/src/service/helpers/order-modifier/order-modifier.ts` |
| 实体层 | 数据模型定义 | `order-modification.entity.ts`, `order.entity.ts`, `surcharge.entity.ts` |
| 状态机层 | 订单状态流转与守卫 | `default-order-process.ts`, `order-state-machine/` |
| 审计层 | 变更历史记录 | `history.service.ts`, `order-history-entry.entity.ts` |

---

## 二、订单修改前置条件

### 2.1 状态流转约束

订单必须处于 `Modifying` 状态才能进行修改。状态流转路径定义在 `default-order-process.ts:174-235`：

```
PaymentAuthorized ──→ Modifying
PaymentSettled  ──→ Modifying
PartiallyShipped ──→ Modifying
Shipped         ──→ Modifying
PartiallyDelivered ─→ Modifying
```

**关键校验点** (`order-modifier.ts:390-392`)：
```typescript
if (order.state !== 'Modifying') {
    return new OrderModificationStateError();
}
```

### 2.2 退出 Modifying 状态的校验

在 `default-order-process.ts:262-278` 中定义了 `onTransitionStart` 守卫：

- `checkModificationPayments`（默认开启）：所有 `OrderModification` 必须已结算（`isSettled === true`）
- `isSettled` 判断逻辑 (`order-modification.entity.ts:56-61`)：
  - `priceChange === 0` → 已结算
  - 或存在关联的 `payment` / `refund` → 已结算

---

## 三、GraphQL API 入口

### 3.1 Mutation 定义

在 `order.api.graphql:30` 定义：
```graphql
modifyOrder(input: ModifyOrderInput!): ModifyOrderResult!
```

### 3.2 ModifyOrderInput 结构 (`order.api.graphql:185-206`)

| 字段 | 类型 | 说明 |
|------|------|------|
| `dryRun` | `Boolean!` | 是否仅预览，不实际提交 |
| `orderId` | `ID!` | 订单 ID |
| `addItems` | `[AddItemInput!]` | 新增商品行 |
| `adjustOrderLines` | `[OrderLineInput!]` | 调整既有商品行数量 |
| `surcharges` | `[SurchargeInput!]` | 附加费用（可正可负） |
| `updateShippingAddress` | `UpdateOrderAddressInput` | 更新收货地址 |
| `updateBillingAddress` | `UpdateOrderAddressInput` | 更新账单地址 |
| `note` | `String` | 修改备注 |
| `refund` / `refunds` | `AdministratorRefundInput` | 退款信息（单个/多个） |
| `options` | `ModifyOrderOptions` | 修改选项（冻结促销、重算运费） |
| `couponCodes` | `[String!]` | 优惠券码 |
| `shippingMethodIds` | `[ID!]` | 配送方式 ID |

### 3.3 Resolver 层处理 (`order.resolver.ts:197-211`)

```typescript
@Transaction('manual')
@Mutation()
@Allow(Permission.UpdateOrder)
async modifyOrder(@Ctx() ctx: RequestContext, @Args() args: MutationModifyOrderArgs) {
    await this.connection.startTransaction(ctx);
    const result = await this.orderService.modifyOrder(ctx, args.input);

    if (args.input.dryRun || isGraphQlErrorResult(result)) {
        await this.connection.rollBackTransaction(ctx);
    } else {
        await this.connection.commitOpenTransaction(ctx);
    }
    return result;
}
```

**关键机制**：
- 使用手动事务管理（`@Transaction('manual')`）
- `dryRun=true` 时回滚事务，仅返回预览结果
- 发生错误时也回滚事务

---

## 四、核心修改逻辑分析

核心逻辑集中在 `order-modifier.ts:376-697` 的 `modifyOrder` 方法。

### 4.1 初始化与前置校验

```typescript
// 1. 检查是否有变更
if (this.noChangesSpecified(input)) {
    return new NoChangesSpecifiedError();
}

// 2. 初始化 refundInputs
const refundInputs: RefundOrderInput[] = refundInputArray.map(refund => ({
    lines: [],
    adjustment: 0,
    shipping: 0,
    paymentId: refund.paymentId,
    amount: refund.amount,
    reason: refund.reason || input.note,
}));
```

`noChangesSpecified` 检查 (`order-modifier.ts:762-773`)：
- `adjustOrderLines`、`addItems`、`surcharges` 均为空
- 无地址更新、无优惠券变更、无配送方式变更、无自定义字段变更

### 4.2 商品行增删

#### 4.2.1 新增商品行 (`addItems`) (`order-modifier.ts:413-442`)

```typescript
for (const row of input.addItems ?? []) {
    // 1. 数量校验
    if (quantity < 0) return new NegativeQuantityError();
    
    // 2. 获取或创建 OrderLine
    const orderLine = await this.getOrCreateOrderLine(ctx, order, productVariantId, customFields);
    
    // 3. 库存约束校验
    const correctedQuantity = await this.constrainQuantityToSaleable(...);
    if (correctedQuantity < quantity) {
        return new InsufficientStockError(...);
    }
    
    // 4. 更新数量（含库存分配）
    await this.updateOrderLineQuantity(ctx, orderLine, initialQuantity + correctedQuantity, order);
    
    // 5. 记录修改明细
    const orderModificationLine = await this.connection
        .getRepository(ctx, OrderModificationLine)
        .save(new OrderModificationLine({ orderLine, quantity: quantity - initialQuantity }));
    modification.lines.push(orderModificationLine);
}
```

#### 4.2.2 调整商品行 (`adjustOrderLines`) (`order-modifier.ts:444-505`)

```typescript
for (const row of input.adjustOrderLines ?? []) {
    // 1. 查找 OrderLine
    const orderLine = order.lines.find(line => idsAreEqual(line.id, orderLineId));
    
    // 2. 数量增加时的库存校验
    if (initialLineQuantity < quantity) {
        const additionalQuantity = await this.constrainQuantityToSaleable(...);
        correctedQuantity = initialLineQuantity + additionalQuantity;
    }
    
    // 3. 数量减少时的处理
    if (quantity < initialLineQuantity) {
        // 取消库存分配 + 记录取消历史
        const cancelLinesInput = [{ orderLineId, quantity: initialLineQuantity - quantity }];
        await this.cancelOrderByOrderLines(ctx, { orderId: order.id }, cancelLinesInput);
        orderLine.quantity = quantity;
        
        // 将减少的数量自动加入退款明细
        refundInputs.forEach(ri => {
            ri.lines?.push({ orderLineId: orderLine.id, quantity: qtyDelta });
        });
    }
    
    // 4. 记录修改明细
    modification.lines.push(orderModificationLine);
}
```

#### 4.2.3 库存变动处理 (`updateOrderLineQuantity`) (`order-modifier.ts:214-249`)

| 场景 | 操作 |
|------|------|
| 数量增加 | 创建 `Allocation` 库存分配记录 |
| 数量减少 | 创建 `Cancellation` 取消记录 + `Release` 释放记录 |

### 4.3 金额调整（Surcharge）(`order-modifier.ts:507-539`)

```typescript
for (const surchargeInput of input.surcharges ?? []) {
    const taxLines = surchargeInput.taxRate != null
        ? [{ taxRate: surchargeInput.taxRate, description: surchargeInput.taxDescription || '' }]
        : [];
    
    const surcharge = await this.connection.getRepository(ctx, Surcharge).save(
        new Surcharge({
            sku: surchargeInput.sku || '',
            description: surchargeInput.description,
            listPrice: surchargeInput.price,
            listPriceIncludesTax: surchargeInput.priceIncludesTax,
            taxLines,
            order,
        }),
    );
    order.surcharges.push(surcharge);
    modification.surcharges.push(surcharge);
    
    // 负的 Surcharge 自动计入退款调整
    if (surcharge.priceWithTax < 0) {
        refundInputs.forEach(ri => {
            if (ri.adjustment != null) {
                ri.adjustment += Math.abs(surcharge.priceWithTax);
            }
        });
    }
}
```

**Surcharge 实体** (`surcharge.entity.ts`)：
- `price`：不含税金额
- `priceWithTax`：含税金额
- `taxRate`：税率（由 `taxLines` 求和得出）
- 可正可负，用于任意金额调整

### 4.4 地址与其他更新

- **收货地址更新** (`order-modifier.ts:541-555`)：更新 `shippingAddress`，记录到 `modification.shippingAddressChange`
- **账单地址更新** (`order-modifier.ts:557-571`)：更新 `billingAddress`，记录到 `modification.billingAddressChange`
- **优惠券更新** (`order-modifier.ts:573-608`)：校验有效性，记录 `ORDER_COUPON_APPLIED` / `ORDER_COUPON_REMOVED` 历史
- **配送方式更新** (`order-modifier.ts:613-618`)：调用 `setShippingMethods` 重新分配

### 4.5 价格重新计算

```typescript
// 1. 应用渠道价格与税费
for (const orderLine of updatedOrderLines) {
    const variant = await this.productVariantService.applyChannelPriceAndTax(...);
    const priceResult = await orderItemPriceCalculationStrategy.calculateUnitPrice(...);
    orderLine.listPrice = priceResult.price;
    orderLine.listPriceIncludesTax = priceResult.priceIncludesTax;
}

// 2. 应用促销调整
await this.orderCalculator.applyPriceAdjustments(ctx, order, promotions, updatedOrderLines, {
    recalculateShipping: input.options?.recalculateShipping,
});
```

---

## 五、退款发起机制

### 5.1 退款触发条件 (`order-modifier.ts:652-687`)

```typescript
const newTotalWithTax = order.totalWithTax;
const delta = newTotalWithTax - initialTotalWithTax;

if (delta < 0) {
    // 订单金额减少，需要退款
    if (refundInputs.length === 0) {
        return new RefundPaymentIdMissingError();
    }
    
    // 选择金额最大的退款作为 primary refund
    const primaryRefund = refundInputs.slice()
        .sort((a, b) => (b.amount || 0) - (a.amount || 0))[0];
    
    // 处理运费差额
    const shippingDelta = order.shippingWithTax - initialShippingWithTax;
    if (shippingDelta < 0) {
        primaryRefund.shipping = shippingDelta * -1;
    }
    
    // 创建退款
    for (const refundInput of refundInputs) {
        const refund = await this.paymentService.createRefund(ctx, refundInput, order, payment);
        if (!isGraphQlErrorResult(refund)) {
            if (idsAreEqual(payment.id, primaryRefund.paymentId)) {
                modification.refund = refund;
            }
        }
    }
}
```

**关键逻辑**：
- `delta < 0`（新总价 < 原总价）时必须指定退款
- `delta > 0` 时需要后续通过 `addManualPaymentToOrder` 补收差额，订单状态会流转到 `ArrangingAdditionalPayment`

### 5.2 退款创建流程 (`payment.service.ts:320-400`)

```typescript
async createRefund(ctx, input, order, selectedPayment) {
    // 1. 校验可退款金额
    const refundableAmount = paymentToRefund.amount - this.getPaymentRefundTotal(paymentToRefund);
    if (refundableAmount < input.amount) {
        return new RefundAmountError({ maximumRefundable: refundableAmount });
    }
    
    // 2. 创建 Refund 实体（状态：Pending）
    let refund = new Refund({
        payment: paymentToRefund,
        total: constrainedTotal,
        reason: input.reason,
        method: selectedPayment.method,
        state: 'Pending',
        // ...
    });
    
    // 3. 调用支付处理器的 createRefund 方法
    const createRefundResult = await handler.createRefund(...);
    
    // 4. 状态流转
    // ...
}
```

---

## 六、变更审计（History）写入与校验

### 6.1 历史记录类型

在 `history.service.ts:71-124` 中定义了 `OrderHistoryEntryData` 接口：

| 类型 | 触发时机 | 数据内容 |
|------|----------|----------|
| `ORDER_STATE_TRANSITION` | 订单状态变更 | `{ from, to }` |
| `ORDER_MODIFIED` | 订单修改完成 | `{ modificationId }` |
| `ORDER_CANCELLATION` | 订单取消 | `{ lines, shippingCancelled, reason }` |
| `ORDER_COUPON_APPLIED` | 优惠券应用 | `{ couponCode, promotionId }` |
| `ORDER_COUPON_REMOVED` | 优惠券移除 | `{ couponCode }` |
| `ORDER_REFUND_TRANSITION` | 退款状态变更 | `{ refundId, from, to, reason }` |
| `ORDER_FULFILLMENT_TRANSITION` | 履约状态变更 | `{ fulfillmentId, from, to }` |

### 6.2 审计写入点

#### 6.2.1 订单修改完成 (`order.service.ts:1399-1406`)

```typescript
await this.historyService.createHistoryEntryForOrder({
    ctx,
    orderId: input.orderId,
    type: HistoryEntryType.ORDER_MODIFIED,
    data: {
        modificationId: result.modification.id,
    },
});
```

#### 6.2.2 取消商品行 (`order-modifier.ts:362-371`)

```typescript
await this.historyService.createHistoryEntryForOrder({
    ctx,
    orderId: order.id,
    type: HistoryEntryType.ORDER_CANCELLATION,
    data: {
        lines: lineInputs,
        reason: input.reason || undefined,
        shippingCancelled: !!input.cancelShipping,
    },
});
```

#### 6.2.3 优惠券变更 (`order-modifier.ts:588-604`)

```typescript
// 新增优惠券
await this.historyService.createHistoryEntryForOrder({
    ctx,
    orderId: order.id,
    type: HistoryEntryType.ORDER_COUPON_APPLIED,
    data: { couponCode, promotionId: validationResult.id },
});

// 移除优惠券
await this.historyService.createHistoryEntryForOrder({
    ctx,
    orderId: order.id,
    type: HistoryEntryType.ORDER_COUPON_REMOVED,
    data: { couponCode: existingCouponCode },
});
```

#### 6.2.4 状态流转 (`default-order-process.ts:444-452`)

```typescript
await historyService.createHistoryEntryForOrder({
    orderId: order.id,
    type: HistoryEntryType.ORDER_STATE_TRANSITION,
    ctx,
    data: { from: fromState, to: toState },
});
```

### 6.3 HistoryService 实现 (`history.service.ts:285-301`)

```typescript
async createHistoryEntryForOrder<T extends keyof OrderHistoryEntryData>(
    args: CreateOrderHistoryEntryArgs<T>,
    isPublic = true,
): Promise<OrderHistoryEntry> {
    const { ctx, data, orderId, type } = args;
    const administrator = await this.getAdministratorFromContext(ctx);
    const entry = new OrderHistoryEntry({
        type,
        isPublic,
        data: data as any,
        order: { id: orderId },
        administrator,
    });
    const history = await this.connection.getRepository(ctx, OrderHistoryEntry).save(entry);
    await this.eventBus.publish(new HistoryEntryEvent(...));
    return history;
}
```

**关键特性**：
- 自动关联当前操作的管理员（从 `RequestContext` 获取）
- `isPublic` 标记控制是否对客户可见
- 发布 `HistoryEntryEvent` 事件供插件监听
- `data` 字段类型安全，通过 `OrderHistoryEntryData` 接口约束

### 6.4 OrderModification 实体审计 (`order-modification.entity.ts`)

```typescript
@Entity()
export class OrderModification extends VendureEntity {
    @Column() note: string;                    // 修改备注
    @ManyToOne(...) order: Order;              // 关联订单
    @OneToMany(...) lines: OrderModificationLine[];  // 修改的商品行明细
    @OneToMany(...) surcharges: Surcharge[];   // 修改的附加费用
    @Money() priceChange: number;              // 价格变动差额
    @ManyToOne(...) refund?: Refund;           // 关联退款
    @ManyToOne(...) payment?: Payment;         // 关联补收款
    @Column('simple-json') shippingAddressChange: OrderAddress;  // 地址变更记录
    @Column('simple-json') billingAddressChange: OrderAddress;
}
```

`OrderModificationLine` 实体 (`order-modification-line.entity.ts`)：
- 记录每个商品行的变更数量（可正可负）
- 关联 `OrderModification` 和 `OrderLine`

---

## 七、完整修改流程时序

```
管理员操作
    ↓
1. transitionOrderToState('Modifying')
    → 写入 ORDER_STATE_TRANSITION 历史
    ↓
2. modifyOrder(input)
    ├─ 前置校验（状态、变更内容）
    ├─ 处理 addItems
    │   ├─ getOrCreateOrderLine
    │   ├─ constrainQuantityToSaleable（库存校验）
    │   ├─ updateOrderLineQuantity（库存分配）
    │   └─ 创建 OrderModificationLine
    ├─ 处理 adjustOrderLines
    │   ├─ 数量增加：库存分配
    │   ├─ 数量减少：cancelOrderByOrderLines
    │   │   ├─ 取消库存分配
    │   │   └─ 写入 ORDER_CANCELLATION 历史
    │   └─ 创建 OrderModificationLine + 自动加入 refund.lines
    ├─ 处理 surcharges
    │   ├─ 创建 Surcharge 实体
    │   └─ 负 surcharge 自动计入 refund.adjustment
    ├─ 处理地址更新 → 记录到 modification
    ├─ 处理优惠券 → 写入 COUPON_APPLIED/REMOVED 历史
    ├─ 重新计算价格（applyPriceAdjustments）
    ├─ dryRun 判断
    │   ├─ true → 回滚事务，返回预览
    │   └─ false → 继续
    ├─ 计算 delta = newTotal - initialTotal
    │   ├─ delta < 0 → 创建 Refund
    │   │   └─ paymentService.createRefund
    │   └─ delta > 0 → 需要后续 addManualPaymentToOrder
    ├─ 保存 OrderModification
    └─ 发布 OrderEvent('updated')
    ↓
3. 写入 ORDER_MODIFIED 历史（order.service）
    ↓
4. 事务提交
    ↓
5. transitionOrderToState(目标状态)
    → onTransitionStart 校验 isSettled
    → 写入 ORDER_STATE_TRANSITION 历史
```

---

## 八、关键校验点汇总

| 校验阶段 | 校验内容 | 错误类型 |
|----------|----------|----------|
| 入口 | 订单状态必须为 `Modifying` | `OrderModificationStateError` |
| 入口 | 必须指定至少一项变更 | `NoChangesSpecifiedError` |
| 商品行 | 数量不能为负 | `NegativeQuantityError` |
| 商品行 | 库存不足 | `InsufficientStockError` |
| 商品行 | 超出订单商品数量限制 | `OrderLimitError` |
| 金额减少 | 必须指定退款 paymentId | `RefundPaymentIdMissingError` |
| 退款 | 退款金额不能超过可退金额 | `RefundAmountError` |
| 优惠券 | 优惠券有效性校验 | `CouponCodeInvalidError` 等 |
| 配送方式 | 配送方式资格校验 | `IneligibleShippingMethodError` |
| 状态退出 | Modification 必须已结算 | 状态机守卫阻止 |

---

## 九、设计特点与注意事项

### 9.1 幂等性与事务
- 整个修改操作在事务中执行
- `dryRun` 模式用于预览，不实际修改数据
- 所有错误都会导致事务回滚

### 9.2 库存处理
- 非活跃订单（已结账）修改时，库存变动通过 `Allocation`/`Cancellation`/`Release` 记录追踪
- 新增数量时创建 `Allocation`（锁定库存）
- 减少数量时创建 `Cancellation` 和 `Release`（释放库存）

### 9.3 金额变动处理
- `delta = newTotalWithTax - initialTotalWithTax`
- `delta < 0`：必须在 modifyOrder 时同步发起退款
- `delta > 0`：订单流转到 `ArrangingAdditionalPayment`，需手动调用 `addManualPaymentToOrder` 补收

### 9.4 审计完整性
- 所有状态变更、商品行变更、金额调整、优惠券变更均有历史记录
- `OrderModification` 实体记录修改的完整快照
- `OrderModificationLine` 记录每个商品行的变更数量
- 自动关联操作管理员

### 9.5 扩展性
- 可通过 `OrderProcess` 扩展状态流转守卫
- 可通过 `orderItemPriceCalculationStrategy` 自定义价格计算
- 可通过 `shippingLineAssignmentStrategy` 自定义配送分配
- 可扩展自定义 `HistoryEntryType` 记录自定义业务事件
