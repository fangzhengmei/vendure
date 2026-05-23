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

### 5.3 refund 与 refunds 归并与优先规则

#### 5.3.1 输入归并逻辑 (`order-modifier.ts:399-403`)

```typescript
const refundInputArray = Array.isArray(input.refunds)
    ? input.refunds
    : input.refund
      ? [input.refund]
      : [];
```

**归并优先级**：
1. **`refunds` 优先**：如果 `input.refunds` 存在且是数组，**完全忽略 `refund` 字段**
2. **降级到 `refund`**：如果 `input.refunds` 不存在，使用 `input.refund` 并包装为单元素数组
3. **都不存在**：空数组（此时若 `delta < 0` 会触发 `RefundPaymentIdMissingError`

> **注意**：当 `refund` 和 `refunds` 同时存在时，`refund` 会被完全忽略，只有 `refunds` 生效。这是因为 `Array.isArray(input.refunds)` 的判断逻辑决定的。

#### 5.3.2 退款输入初始化 (`order-modifier.ts:404-411`)

```typescript
const refundInputs: RefundOrderInput[] = refundInputArray.map(refund => ({
    lines: [],
    adjustment: 0,
    shipping: 0,
    paymentId: refund.paymentId,
    amount: refund.amount,
    reason: refund.reason || input.note,
}));
```

**字段处理**：
- `lines` 初始化为空数组，后续在 `adjustOrderLines` 数量减少时自动填充
- `adjustment` 初始化为 0，后续在负 `surcharge` 时累加
- `shipping` 初始化为 0，后续在运费差额为负时累加到 `primaryRefund`
- `reason` 优先级：`refund.reason` > `input.note`

#### 5.3.3 Primary Refund 选择规则 (`order-modifier.ts:658-660`)

```typescript
const primaryRefund = refundInputs.slice().sort((a, b) => (b.amount || 0) - (a.amount || 0))[0];
```

**选择逻辑**：
- 按 `amount` 字段**降序排序**
- 选择**金额最大**的退款作为 `primaryRefund`
- 只有 `primaryRefund` 会被关联到 `OrderModification.refund` 字段

#### 5.3.4 Primary Refund 专属特权 (`order-modifier.ts:662-670`)

```typescript
// 仅 primaryRefund 会被追加以下内容：
const shippingDelta = order.shippingWithTax - initialShippingWithTax;
if (shippingDelta < 0) {
    primaryRefund.shipping = shippingDelta * -1;  // 运费差额只加给 primaryRefund
}
if (primaryRefund.adjustment != null) {
    primaryRefund.adjustment += await this.calculateRefundAdjustment(ctx, delta, primaryRefund);  // 调整额只加给 primaryRefund
}
```

**⚠️ 重要限制**：
- 运费差额（`shippingDelta < 0`）**仅**追加到 `primaryRefund.shipping`
- 退款调整额（促销等其他因素导致的差额）**仅**追加到 `primaryRefund.adjustment`
- 其他退款只包含 `lines` 和 `surcharge` 带来的调整

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

### 6.5 dryRun 回滚时的审计一致性

#### 6.5.1 事务边界与回滚机制 (`order.resolver.ts:197-211`)

```typescript
@Transaction('manual')
@Mutation()
async modifyOrder(@Ctx() ctx: RequestContext, @Args() args: MutationModifyOrderArgs) {
    await this.connection.startTransaction(ctx);
    const result = await this.orderService.modifyOrder(ctx, args.input);

    if (args.input.dryRun || isGraphQlErrorResult(result)) {
        await this.connection.rollBackTransaction(ctx);  // ← dryRun 时回滚
    } else {
        await this.connection.commitOpenTransaction(ctx);
    }
    return result;
}
```

**关键机制**：
- 使用手动事务管理（`@Transaction('manual')`）
- `dryRun === true` 或发生错误时，**整个数据库事务回滚**
- 所有数据库写入（包括历史记录）都会被撤销

#### 6.5.2 历史记录写入时序分析

`modifyOrder` 方法内的写入顺序与 dryRun 判断位置（`order-modifier.ts:647`）：

```
modifyOrder 执行流：
├─ 前置校验（无写入）
├─ addItems
│  └─ 创建 OrderLine、OrderModificationLine、Allocation（DB 写入，非历史）
├─ adjustOrderLines
│  └─ 数量减少时 → cancelOrderByOrderLines
│     └─ 写入 ORDER_CANCELLATION 历史 ← ✅ dryRun 判断之前
├─ surcharges（DB 写入，非历史）
├─ 地址更新（DB 写入，非历史）
├─ 优惠券变更
│  ├─ 写入 ORDER_COUPON_APPLIED 历史 ← ✅ dryRun 判断之前
│  └─ 写入 ORDER_COUPON_REMOVED 历史 ← ✅ dryRun 判断之前
├─ shippingMethod 更新（DB 写入，非历史）
├─ 价格重算（无写入）
├─ 👇 dryRun 判断 (line 647)
│  ├─ true → return { order, modification }
│  │      → 事务回滚 → 所有历史记录被撤销
│  │      → ⚠️ 以下历史记录**永远不会写入**：
│  │         ORDER_REFUND_TRANSITION、ORDER_MODIFIED
│  └─ false → 继续
├─ delta < 0 → 创建 Refund
│  └─ paymentService.createRefund
│     └─ refundStateMachine.transition
│        └─ onTransitionEnd → 写入 ORDER_REFUND_TRANSITION 历史 ← ❌ dryRun 判断之后（与 HistoryEntryEvent 同时）
├─ 保存 OrderModification（DB 写入，非历史）
└─ return 结果
    ↓
orderService.modifyOrder 返回后
└─ 👇 dryRun 判断 (order.service.ts:1395)
   ├─ dryRun=true → return
   └─ dryRun=false → 写入 ORDER_MODIFIED 历史 ← ❌ dryRun 判断之后
```

#### 6.5.3 dryRun 回滚时会被撤销的历史记录

| 历史类型 | 写入位置 | 与 dryRun 判断关系 | dryRun 场景是否写入 | 回滚后是否保留 |
|----------|----------|-------------------|---------------------|----------------|
| `ORDER_CANCELLATION` | `cancelOrderByOrderLines` 行 362-371 | ✅ **之前** | ✅ 是 | ❌ 撤销 |
| `ORDER_COUPON_APPLIED` | `order-modifier.ts` 行 588-593 | ✅ **之前** | ✅ 是 | ❌ 撤销 |
| `ORDER_COUPON_REMOVED` | `order-modifier.ts` 行 599-604 | ✅ **之前** | ✅ 是 | ❌ 撤销 |
| `ORDER_REFUND_TRANSITION` | `default-refund-process.ts` 行 41-51 | ❌ **之后** | ❌ 否（dryRun 提前 return，根本不会执行到） | ❌ 撤销 |
| `ORDER_MODIFIED` | `order.service.ts` 行 1399-1406 | ❌ **之后** | ❌ 否（仅非 dryRun 执行） | ❌ 撤销 |

**⚠️ 注意**：
- 虽然 `ORDER_CANCELLATION`、`ORDER_COUPON_APPLIED`、`ORDER_COUPON_REMOVED` 在 `dryRun` 判断**之前**就写入了数据库，但由于整个操作在同一数据库事务中，`dryRun=true` 时的事务回滚会**全部撤销**这些写入。
- `ORDER_REFUND_TRANSITION` 和 `ORDER_MODIFIED` 在 `dryRun` 判断**之后**，`dryRun=true` 时**根本不会执行到**这些写入操作。
- 因此 `dryRun` 模式不会产生任何残留的历史记录，审计一致性得以保证。

#### 6.5.4 事件发布与事务的真实关系（EventBus 双轨制）

##### 核心机制 (`event-bus.ts:116-347`)

EventBus 采用**双轨制**设计，不同类型的事件处理器在事务回滚时表现完全不同：

```typescript
async publish<T extends VendureEvent>(event: T): Promise<void> {
    this.eventStream.next(event);                           // 1. 同步推送到 RxJS Subject
    await this.executeBlockingEventHandlers(event);          // 2. 同步执行阻塞处理器
}
```

##### 两种事件处理器的行为差异

| 处理器类型 | 注册方式 | 执行时机 | 事务上下文 | 回滚时可见性 |
|------------|----------|----------|------------|--------------|
| **阻塞事件处理器** | `registerBlockingEventHandler()` | `publish()` 时**同步执行** | ✅ 同一事务内 | ⚠️ 会被调用（数据库操作回滚，外部副作用不回滚） |
| **RxJS 订阅者** | `eventBus.ofType(...).subscribe()` | 事务**提交后**异步执行 | ❌ 事务已结束 | ❌ **完全不可见**（事件被过滤） |

##### `awaitActiveTransactions` 事务等待机制 (`event-bus.ts:311-347`)

所有通过 `ofType()` 订阅的事件都会经过此机制：
```typescript
try {
    await this.transactionSubscriber.awaitCommit(queryRunner);
    // ✅ 事务提交成功：返回事件给订阅者
    return event;
} catch (e: any) {
    if (e instanceof TransactionSubscriberError) {
        // ❌ 事务回滚：返回 undefined，被 filter 过滤
        return;  // 订阅者完全看不到事件
    }
    throw e;
}
```

##### 回滚场景下的事件可见性（dryRun / 错误）

按事件发布时机与 dryRun 判断（`order-modifier.ts:647`）的关系：

| 事件类型 | 发布位置 | 与 dryRun 判断关系 | dryRun 场景是否发布 | 阻塞处理器可见性 | ofType 订阅者可见性 |
|----------|----------|-------------------|---------------------|------------------|---------------------|
| `OrderLineEvent`('created') | `order-modifier.ts:204` | ✅ **之前**（addItems 新增） | ✅ 是 | ⚠️ 回滚前已调用 | ❌ 不可见 |
| `OrderLineEvent`('updated') | `order-modifier.ts:247` | ✅ **之前**（addItems/adjust 更新） | ✅ 是 | ⚠️ 回滚前已调用 | ❌ 不可见 |
| `OrderLineEvent`('cancelled') | `order-modifier.ts:339` | ✅ **之前**（adjust 数量减少） | ✅ 是 | ⚠️ 回滚前已调用 | ❌ 不可见 |
| `HistoryEntryEvent`(ORDER_CANCELLATION) | `order-modifier.ts:362` | ✅ **之前** | ✅ 是 | ⚠️ 回滚前已调用 | ❌ 不可见 |
| `HistoryEntryEvent`(ORDER_COUPON_APPLIED) | `order-modifier.ts:588` | ✅ **之前** | ✅ 是 | ⚠️ 回滚前已调用 | ❌ 不可见 |
| `HistoryEntryEvent`(ORDER_COUPON_REMOVED) | `order-modifier.ts:599` | ✅ **之前** | ✅ 是 | ⚠️ 回滚前已调用 | ❌ 不可见 |
| **dryRun 判断（order-modifier.ts:647）** | --- | --- | --- | --- | --- |
| `HistoryEntryEvent`(ORDER_REFUND_TRANSITION) | `default-refund-process.ts:41` | ❌ **之后**（仅 non-dryRun + delta<0） | ❌ 否 | ❌ 不调用 | ❌ 不可见 |
| `RefundStateTransitionEvent` | `payment.service.ts:453` | ❌ **之后**（仅 non-dryRun + delta<0） | ❌ 否 | ❌ 不调用 | ❌ 不可见 |
| `OrderEvent`('updated') | `order-modifier.ts:695` | ❌ **之后**（仅 non-dryRun） | ❌ 否 | ❌ 不调用 | ❌ 不可见 |
| **dryRun 判断（order.service.ts:1395）** | --- | --- | --- | --- | --- |
| `HistoryEntryEvent`(ORDER_MODIFIED) | `order.service.ts:1399` | ❌ **之后**（仅 non-dryRun） | ❌ 否 | ❌ 不调用 | ❌ 不可见 |

**⚠️ 关键更正**：
- ❌ **之前的错误理解**："`OrderEvent`('updated') 在 dryRun 判断前发布，dryRun 场景也会发布"
- ✅ **正确理解**：
  - `OrderEvent`('updated') 发布于 **line 695**，dryRun 判断在 **line 647**，晚 48 行
  - `HistoryEntryEvent`(ORDER_REFUND_TRANSITION)、`RefundStateTransitionEvent`、`OrderEvent`('updated')、`HistoryEntryEvent`(ORDER_MODIFIED) 均在 dryRun 判断**之后**
  - **dryRun 场景下这 4 个事件完全不会发布**，连阻塞处理器也不会收到
  - 对于**绝大多数插件使用的 `ofType()` 订阅方式**，回滚场景下**完全不会收到任何事件**
  - 只有**阻塞事件处理器**（2.2.0 新增的高级 API，使用极少）才会在回滚前被调用，但仅能看到 dryRun 判断前发布的 6 种事件
  - 阻塞处理器内的**数据库操作**在同一事务内 → 会被回滚
  - 阻塞处理器内的**外部副作用**（发送邮件、调用外部 API）→ 不会回滚

##### 订单修改流程中的事件发布时序（完整清单）

```
modifyOrder 执行流：
├─ addItems 循环
│  ├─ getOrCreateOrderLine
│  │   └─ publish OrderLineEvent('created') → ✅ dryRun 判断前
│  └─ updateOrderLineQuantity
│      └─ publish OrderLineEvent('updated') → ✅ dryRun 判断前
├─ adjustOrderLines 循环
│  ├─ 数量增加 → updateOrderLineQuantity
│  │   └─ publish OrderLineEvent('updated') → ✅ dryRun 判断前
│  └─ 数量减少 → cancelOrderByOrderLines
│      ├─ publish OrderLineEvent('cancelled') → ✅ dryRun 判断前
│      └─ historyService.createHistoryEntryForOrder
│          └─ publish HistoryEntryEvent(ORDER_CANCELLATION) → ✅ dryRun 判断前
├─ surcharges 处理（无事件发布）
├─ 地址更新（无事件发布）
├─ 优惠券变更
│  ├─ 新增优惠券 → historyService.createHistoryEntryForOrder
│  │   └─ publish HistoryEntryEvent(ORDER_COUPON_APPLIED) → ✅ dryRun 判断前
│  └─ 移除优惠券 → historyService.createHistoryEntryForOrder
│      └─ publish HistoryEntryEvent(ORDER_COUPON_REMOVED) → ✅ dryRun 判断前
├─ 配送方式更新（无事件发布）
├─ 价格重算（无事件发布）
├─ 👇 dryRun 判断（order-modifier.ts:647）
│  ├─ dryRun === true → return { order, modification }
│  │      ⚠️ 以下事件均不会发布
│  └─ dryRun === false → 继续执行
├─ delta < 0 → 创建 Refund
│  └─ paymentService.createRefund
│      ├─ refundStateMachine.transition
│      │   └─ onTransitionEnd → historyService.createHistoryEntryForOrder
│      │       └─ publish HistoryEntryEvent(ORDER_REFUND_TRANSITION) → ❌ dryRun 判断后
│      └─ publish RefundStateTransitionEvent → ❌ dryRun 判断后
├─ 保存 OrderModification（无事件发布）
└─ publish OrderEvent('updated') → ❌ dryRun 判断后
↓
orderService.modifyOrder 返回
└─ 👇 dryRun 判断（order.service.ts:1395）
   ├─ dryRun === true → return result.order
   └─ dryRun === false → historyService.createHistoryEntryForOrder
       └─ publish HistoryEntryEvent(ORDER_MODIFIED) → ❌ dryRun 判断后
```

##### 插件侧可观察边界（修正版，与上表逐条对应）

**dryRun 场景（事务回滚）：**

| 插件实现方式 | 可见事件（与上表逐条对应） | 不可见事件（与上表逐条对应） | 说明 |
|--------------|------------------------|--------------------------|------|
| `ofType(...).subscribe()` | ❌ 无任何事件 | 全部 10 种事件 | 所有事件均被 `awaitActiveTransactions` 过滤，事务回滚后订阅者完全看不到 |
| `registerBlockingEventHandler(...)` | `OrderLineEvent`('created')<br/>`OrderLineEvent`('updated')<br/>`OrderLineEvent`('cancelled')<br/>`HistoryEntryEvent`(ORDER_CANCELLATION)<br/>`HistoryEntryEvent`(ORDER_COUPON_APPLIED)<br/>`HistoryEntryEvent`(ORDER_COUPON_REMOVED) | `HistoryEntryEvent`(ORDER_REFUND_TRANSITION)<br/>`RefundStateTransitionEvent`<br/>`OrderEvent`('updated')<br/>`HistoryEntryEvent`(ORDER_MODIFIED) | 仅能看到 dryRun 判断**之前**发布的事件，共 6 种；dryRun 判断后 4 种完全不会发布 |

**错误回滚场景（如退款创建失败，仅发生在 dryRun=false 时）：**

| 插件实现方式 | 可见事件 | 不可见事件 | 说明 |
|--------------|----------|------------|------|
| `ofType(...).subscribe()` | ❌ 无任何事件 | 全部 10 种事件 | 所有事件均被 `awaitActiveTransactions` 过滤，事务回滚后订阅者完全看不到 |
| `registerBlockingEventHandler(...)` | `OrderLineEvent` × 3 种<br/>`HistoryEntryEvent`(ORDER_CANCELLATION)<br/>`HistoryEntryEvent`(ORDER_COUPON_APPLIED)<br/>`HistoryEntryEvent`(ORDER_COUPON_REMOVED)<br/>（取决于错误发生位置，可能额外看到 `HistoryEntryEvent`(ORDER_REFUND_TRANSITION) 和 `RefundStateTransitionEvent`） | `OrderEvent`('updated')<br/>`HistoryEntryEvent`(ORDER_MODIFIED) | 看到错误发生前已发布的事件；`OrderEvent`('updated') 在退款之后，退款失败时永远不会发布 |

**事务提交场景（成功修改）：**

| 插件实现方式 | 可见事件 | 说明 |
|--------------|----------|------|
| `ofType(...).subscribe()` | ✅ 全部 10 种事件 | 事务提交后通过 `awaitActiveTransactions`，正常传递给订阅者 |
| `registerBlockingEventHandler(...)` | ✅ 全部 10 种事件 | 发布时同步执行，无需等待事务提交 |

**插件开发最佳实践**：
1. 优先使用 `ofType()` 订阅 → 天然保证事务一致性，回滚场景完全不会触发
2. 如需使用阻塞处理器，**不要在其中执行不可回滚的外部操作**（如发送邮件、调用外部 API）
3. 如需执行外部操作，应在 `ofType()` 订阅中执行（事务已提交，数据已持久化）
4. 如确实需要在阻塞处理器中执行操作，请通过 `input.dryRun` 参数判断，仅在 `dryRun === false` 时执行外部操作

### 6.6 前置校验对历史写入的阻断机制

#### 6.6.1 阻断历史写入的前置校验（按执行顺序）

所有以下校验都在**任何数据库写入之前**执行，失败时直接返回错误，不会产生任何历史记录：

| 校验点 | 位置 | 错误类型 | 发生时机 |
|--------|------|----------|----------|
| 订单状态必须为 `Modifying` | `order-modifier.ts:390-392` | `OrderModificationStateError` | 最优先 |
| 必须指定至少一项变更 | `order-modifier.ts:393-395` | `NoChangesSpecifiedError` | 优先 |
| addItems 数量不能为负 | `order-modifier.ts:415-416` | `NegativeQuantityError` | addItems 循环内 |
| addItems 超出数量限制 | `order-modifier.ts:426-427` | `OrderLimitError` | addItems 循环内 |
| addItems 库存不足 | `order-modifier.ts:431-432` | `InsufficientStockError` | addItems 循环内 |
| adjustOrderLines 数量不能为负 | `order-modifier.ts:446-447` | `NegativeQuantityError` | adjustOrderLines 循环内 |
| adjustOrderLines 数量超限 | `order-modifier.ts:464-465` | `OrderLimitError` | adjustOrderLines 循环内 |
| adjustOrderLines 库存不足 | `order-modifier.ts:469-470` | `InsufficientStockError` | adjustOrderLines 循环内 |
| 优惠券无效/过期/超限 | `order-modifier.ts:580-585` | `CouponCodeInvalidError` 等 | 优惠券处理循环内 |
| 配送方式不合格 | `order-modifier.ts:613-617` | `IneligibleShippingMethodError` | 配送方式更新时 |

**校验流程**：
```
开始 modifyOrder
    ↓
1. 状态校验（无写入）→ 失败 → return 错误（无历史）
    ↓
2. noChanges 校验（无写入）→ 失败 → return 错误（无历史）
    ↓
3. addItems 循环
   ├─ 数量负校验 → 失败 → return 错误（无历史）
   ├─ 数量限制校验 → 失败 → return 错误（无历史）
   └─ 库存校验 → 失败 → return 错误（无历史）
    ↓
4. adjustOrderLines 循环
   ├─ 数量负校验 → 失败 → return 错误（无历史）
   ├─ 数量限制校验 → 失败 → return 错误（无历史）
   └─ 库存校验 → 失败 → return 错误（无历史）
    ↓
5. surcharges 处理（开始有 DB 写入，但非历史）
    ↓
6. 优惠券校验 → 失败 → return 错误（无历史）
    ↓
7. 配送方式校验 → 失败 → return 错误（无历史）
    ↓
... 后续处理 ...
```

#### 6.6.2 部分写入后失败的回滚

如果在**已有部分数据库写入后**发生错误（如退款创建失败）：

```typescript
for (const refundInput of refundInputs) {
    const refund = await this.paymentService.createRefund(ctx, refundInput, order, payment);
    if (!isGraphQlErrorResult(refund)) {
        // ...
    } else {
        throw new InternalServerError(refund.message);  // ← 抛出异常
    }
}
```

- Resolver 层捕获到错误结果后会调用 `rollBackTransaction`
- 所有已写入的数据库记录（包括历史记录）都会被回滚
- 保证审计一致性：要么全部成功，要么全部撤销

#### 6.6.3 delta < 0 时的退款校验

在 `dryRun` 判断**之后**、实际创建退款**之前**还有一次校验：

```typescript
if (delta < 0) {
    if (refundInputs.length === 0) {
        return new RefundPaymentIdMissingError();  // ← 此时已有部分 DB 写入
    }
    // ... 创建退款
}
```

**注意**：此时 `surcharges`、`OrderModificationLine` 等已经写入数据库，但由于还在同一事务中，返回错误会触发回滚，所有写入都会被撤销。

---

## 七、完整修改流程时序

### 7.1 标准修改流程

```
管理员操作
    ↓
1. transitionOrderToState('Modifying')
    → 写入 ORDER_STATE_TRANSITION 历史
    ↓
2. modifyOrder(input)
    ├─ 【前置校验1】状态必须为 Modifying
    │      → 失败：return 错误（无任何写入）
    ├─ 【前置校验2】必须有变更内容
    │      → 失败：return 错误（无任何写入）
    ├─ 处理 addItems
    │   ├─ 【校验】数量≥0 → 失败：return 错误（无任何写入）
    │   ├─ 【校验】库存足够 → 失败：return 错误（无任何写入）
    │   ├─ 【校验】数量限制 → 失败：return 错误（无任何写入）
    │   ├─ getOrCreateOrderLine
    │   ├─ updateOrderLineQuantity（库存分配）
    │   └─ 创建 OrderModificationLine
    ├─ 处理 adjustOrderLines
    │   ├─ 【校验】数量≥0 → 失败：return 错误（无任何写入）
    │   ├─ 【校验】库存足够 → 失败：return 错误（无任何写入）
    │   ├─ 【校验】数量限制 → 失败：return 错误（无任何写入）
    │   ├─ 数量增加：updateOrderLineQuantity（库存分配）
    │   ├─ 数量减少：cancelOrderByOrderLines
    │   │   ├─ 取消库存分配
    │   │   └─ 写入 ORDER_CANCELLATION 历史 ← dryRun 前已写入
    │   └─ 创建 OrderModificationLine + 自动加入 refund.lines
    ├─ 处理 surcharges
    │   ├─ 创建 Surcharge 实体
    │   └─ 负 surcharge 自动计入 refund.adjustment
    ├─ 处理地址更新 → 记录到 modification
    ├─ 处理优惠券
    │   ├─ 【校验】优惠券有效性 → 失败：return 错误（无历史）
    │   ├─ 新增 → 写入 ORDER_COUPON_APPLIED 历史 ← dryRun 前已写入
    │   └─ 移除 → 写入 ORDER_COUPON_REMOVED 历史 ← dryRun 前已写入
    ├─ 处理 shippingMethodIds
    │   └─ 【校验】配送方式资格 → 失败：return 错误（无历史）
    ├─ 重新计算价格（applyPriceAdjustments）
    ├─ dryRun 判断 (line 647)
    │   ├─ true → return { order, modification }
    │   │      ↓
    │   │      Resolver 层 rollBackTransaction
    │   │      → 所有 DB 写入撤销（包括历史记录）
    │   │      → 仅 dryRun 判断前的 6 种事件触发了阻塞处理器
    │   │      → dryRun 判断后的 4 种事件完全不会发布
    │   │      → ofType 订阅者完全看不到任何事件
    │   │
    │   └─ false → 继续
    ├─ 计算 delta = newTotal - initialTotal
    │   ├─ delta < 0
    │   │   ├─ 【校验】refundInputs 非空 → 失败：return 错误（回滚）
    │   │   ├─ 选择 primaryRefund（按 amount 降序）
    │   │   ├─ 运费差额追加到 primaryRefund.shipping
    │   │   ├─ 调整额追加到 primaryRefund.adjustment
    │   │   └─ 遍历创建 Refund
    │   │       └─ paymentService.createRefund
    │   │           ├─ 写入 ORDER_REFUND_TRANSITION 历史 ← dryRun 后
    │   │           └─ 发布 RefundStateTransitionEvent ← dryRun 后
    │   └─ delta > 0 → 后续需 addManualPaymentToOrder
    ├─ 保存 OrderModification
    └─ 发布 OrderEvent('updated') ← dryRun 后！
    ↓
3. 写入 ORDER_MODIFIED 历史（order.service）← 仅非 dryRun 执行
    ↓
4. 事务 commit
    ↓
5. transitionOrderToState(目标状态)
    → onTransitionStart 校验所有 Modification.isSettled
    → 写入 ORDER_STATE_TRANSITION 历史
```

### 7.2 dryRun 模式数据流

```
modifyOrder(dryRun=true)
    ↓
┌─────────────────────────────────────────────────┐
│  数据库事务内执行                              │
│  ├─ addItems 循环                              │
│  │  ├─ 创建 OrderLine、Surcharge 等（DB 写入） │
│  │  ├─ publish OrderLineEvent('created')       │
│  │  └─ publish OrderLineEvent('updated')       │
│  ├─ adjustOrderLines 循环（数量减少）           │
│  │  ├─ 写入 ORDER_CANCELLATION 历史             │
│  │  └─ publish OrderLineEvent('cancelled')     │
│  ├─ 优惠券变更（如有）                          │
│  │  ├─ 写入 ORDER_COUPON_APPLIED/REMOVED 历史   │
│  │  └─ publish HistoryEntryEvent × 2            │
│  ├─ 👇 dryRun 判断 (line 647) → return 预览     │
│  │    ⚠️ 以下 4 种事件**全部不发布**：           │
│  │      - HistoryEntryEvent(ORDER_REFUND_TRANSITION) │
│  │      - RefundStateTransitionEvent              │
│  │      - OrderEvent('updated')                   │
│  │      - HistoryEntryEvent(ORDER_MODIFIED)       │
│  └─ 返回预览结果                                │
└─────────────────────────────────────────────────┘
    ↓
Resolver 检测到 dryRun=true
    ↓
rollBackTransaction() → 所有 DB 写入回滚
    ↓
TransactionSubscriber 监听到 rollback 事件
    ↓
ofType() 订阅者的 awaitActiveTransactions 捕获回滚
    ↓
返回 undefined → 被 filter(notNullOrUndefined) 过滤
    ↓
最终效果：
  ✅ 无任何持久化变更
  ✅ ofType 订阅者完全看不到事件
  ⚠️ 阻塞处理器仅收到 dryRun 前发布的 6 种事件
     （但 DB 操作已回滚）
```

### 7.2.1 dryRun 场景事件分类汇总（与 6.5.4 节逐条对应）
| 事件组及事件 | 发布位置 | 是否发布 | 阻塞处理器是否可见 | ofType 订阅者是否可见 |
|--------------|----------|----------|------------------|---------------------|
| **dryRun 判断前发布（6 种事件）** | --- | ✅ 是 | ✅ 是 | ❌ 否 |
| `OrderLineEvent`('created') | `order-modifier.ts:204` | ✅ 是 | ✅ 是 | ❌ 否 |
| `OrderLineEvent`('updated') | `order-modifier.ts:247` | ✅ 是 | ✅ 是 | ❌ 否 |
| `OrderLineEvent`('cancelled') | `order-modifier.ts:339` | ✅ 是 | ✅ 是 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_CANCELLATION) | `order-modifier.ts:362` | ✅ 是 | ✅ 是 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_COUPON_APPLIED) | `order-modifier.ts:588` | ✅ 是 | ✅ 是 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_COUPON_REMOVED) | `order-modifier.ts:599` | ✅ 是 | ✅ 是 | ❌ 否 |
| **dryRun 判断后发布（4 种事件）** | --- | ❌ 否 | ❌ 否 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_REFUND_TRANSITION) | `default-refund-process.ts:41` | ❌ 否 | ❌ 否 | ❌ 否 |
| `RefundStateTransitionEvent` | `payment.service.ts:453` | ❌ 否 | ❌ 否 | ❌ 否 |
| `OrderEvent`('updated') | `order-modifier.ts:695` | ❌ 否 | ❌ 否 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_MODIFIED) | `order.service.ts:1399` | ❌ 否 | ❌ 否 | ❌ 否 |

### 7.3 错误回滚数据流

```
modifyOrder 执行中发生错误（如退款创建失败）
    ↓
┌─────────────────────────────────────────────────┐
│  已执行的操作（同一事务内）                     │
│  ├─ dryRun 判断前：                             │
│  │  ├─ 部分 OrderLine 更新                     │
│  │  ├─ 部分 Surcharge 创建                     │
│  │  ├─ 部分历史记录写入（6 种）                │
│  │  └─ publish 6 种事件 → 阻塞处理器已调用      │
│  ├─ dryRun 判断后（dryRun=false 才执行到）：     │
│  │  ├─ 可能已创建 Refund                        │
│  │  ├─ 可能已写入 ORDER_REFUND_TRANSITION 历史  │
│  │  ├─ 可能已 publish HistoryEntryEvent(ORDER_REFUND_TRANSITION) │
│  │  ├─ 可能已 publish RefundStateTransitionEvent│
│  │  └─ 但还没到 OrderEvent('updated')           │
│  └─ 抛出错误                                    │
└─────────────────────────────────────────────────┘
    ↓
Resolver 检测到 ErrorResult
    ↓
rollBackTransaction() → 全部 DB 写入撤销
    ↓
TransactionSubscriber 监听到 rollback 事件
    ↓
ofType() 订阅者的 awaitActiveTransactions 捕获回滚
    ↓
返回 undefined → 被 filter 过滤
    ↓
最终效果：
  ✅ 无任何持久化变更，审计一致性保持
  ✅ ofType 订阅者完全看不到事件
  ⚠️ 阻塞处理器收到错误发生前已发布的事件
     （DB 操作已回滚，外部副作用不回滚）
```

### 7.3.1 错误回滚场景事件分类汇总（与 6.5.4 节逐条对应）

按错误发生位置分两种情况：

**情况 1：错误发生在 dryRun 判断前**
（如 addItems 库存不足、优惠券无效等前置校验失败）

| 事件组及事件 | 发布位置 | 是否发布 | 阻塞处理器可见性 | ofType 订阅者可见性 |
|--------------|----------|----------|------------------|---------------------|
| **dryRun 判断前（6 种）** | --- | ⚠️ 取决于错误发生时机 | ⚠️ 部分可见 | ❌ 否 |
| `OrderLineEvent`('created') | `order-modifier.ts:204` | ⚠️ 取决于错误时机 | ⚠️ 可能可见 | ❌ 否 |
| `OrderLineEvent`('updated') | `order-modifier.ts:247` | ⚠️ 取决于错误时机 | ⚠️ 可能可见 | ❌ 否 |
| `OrderLineEvent`('cancelled') | `order-modifier.ts:339` | ⚠️ 取决于错误时机 | ⚠️ 可能可见 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_CANCELLATION) | `order-modifier.ts:362` | ⚠️ 取决于错误时机 | ⚠️ 可能可见 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_COUPON_APPLIED) | `order-modifier.ts:588` | ⚠️ 取决于错误时机 | ⚠️ 可能可见 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_COUPON_REMOVED) | `order-modifier.ts:599` | ⚠️ 取决于错误时机 | ⚠️ 可能可见 | ❌ 否 |
| **dryRun 判断后（4 种）** | --- | ❌ 否 | ❌ 否 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_REFUND_TRANSITION) | `default-refund-process.ts:41` | ❌ 否 | ❌ 否 | ❌ 否 |
| `RefundStateTransitionEvent` | `payment.service.ts:453` | ❌ 否 | ❌ 否 | ❌ 否 |
| `OrderEvent`('updated') | `order-modifier.ts:695` | ❌ 否 | ❌ 否 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_MODIFIED) | `order.service.ts:1399` | ❌ 否 | ❌ 否 | ❌ 否 |

**情况 2：错误发生在 dryRun 判断后（仅 dryRun=false 场景）**
（如退款创建失败、退款金额超限等）

| 事件组及事件 | 发布位置 | 是否发布 | 阻塞处理器可见性 | ofType 订阅者可见性 |
|--------------|----------|----------|------------------|---------------------|
| **dryRun 判断前（6 种）** | --- | ✅ 是 | ✅ 是 | ❌ 否 |
| `OrderLineEvent`('created') | `order-modifier.ts:204` | ✅ 是 | ✅ 是 | ❌ 否 |
| `OrderLineEvent`('updated') | `order-modifier.ts:247` | ✅ 是 | ✅ 是 | ❌ 否 |
| `OrderLineEvent`('cancelled') | `order-modifier.ts:339` | ✅ 是 | ✅ 是 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_CANCELLATION) | `order-modifier.ts:362` | ✅ 是 | ✅ 是 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_COUPON_APPLIED) | `order-modifier.ts:588` | ✅ 是 | ✅ 是 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_COUPON_REMOVED) | `order-modifier.ts:599` | ✅ 是 | ✅ 是 | ❌ 否 |
| **dryRun 判断后（4 种）** | --- | ⚠️ 取决于错误位置 | ⚠️ 部分可见 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_REFUND_TRANSITION) | `default-refund-process.ts:41` | ⚠️ 取决于错误位置 | ⚠️ 可能可见 | ❌ 否 |
| `RefundStateTransitionEvent` | `payment.service.ts:453` | ⚠️ 取决于错误位置 | ⚠️ 可能可见 | ❌ 否 |
| `OrderEvent`('updated') | `order-modifier.ts:695` | ❌ 否（在退款之后） | ❌ 否 | ❌ 否 |
| `HistoryEntryEvent`(ORDER_MODIFIED) | `order.service.ts:1399` | ❌ 否 | ❌ 否 | ❌ 否 |

---

## 八、关键校验点汇总

| 校验阶段 | 校验内容 | 错误类型 | 是否阻断历史写入 |
|----------|----------|----------|------------------|
| 入口 | 订单状态必须为 `Modifying` | `OrderModificationStateError` | ✅ 是（无任何写入） |
| 入口 | 必须指定至少一项变更 | `NoChangesSpecifiedError` | ✅ 是（无任何写入） |
| 商品行 | 数量不能为负 | `NegativeQuantityError` | ✅ 是（无任何写入） |
| 商品行 | 库存不足 | `InsufficientStockError` | ✅ 是（无任何写入） |
| 商品行 | 超出订单商品数量限制 | `OrderLimitError` | ✅ 是（无任何写入） |
| 优惠券 | 优惠券有效性校验 | `CouponCodeInvalidError` 等 | ✅ 是（无历史写入） |
| 配送方式 | 配送方式资格校验 | `IneligibleShippingMethodError` | ✅ 是（无历史写入） |
| 金额减少 | 必须指定退款 paymentId | `RefundPaymentIdMissingError` | ⚠️ 部分（已有 DB 写入但无历史） |
| 退款 | 退款金额不能超过可退金额 | `RefundAmountError` | ❌ 否（事务回滚撤销） |
| 状态退出 | Modification 必须已结算 | 状态机守卫阻止 | ✅ 是（独立流程） |

---

## 九、设计特点与注意事项

### 9.1 幂等性与事务
- 整个修改操作在手动事务中执行
- `dryRun` 模式用于预览，通过事务回滚保证不遗留数据
- 所有错误都会导致事务回滚，保证数据一致性

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
- **事务保障**：所有历史记录写入在同一事务中，要么全部成功，要么全部撤销

### 9.5 dryRun 模式的审计一致性保证
- 通过数据库事务回滚机制实现"预览但不提交"
- 即使部分历史记录在 dryRun 判断前已写入数据库，事务回滚会全部撤销
- **事件发布时序的严格边界**（按 `order-modifier.ts:647` dryRun 判断划分）：
  - **dryRun 判断前**：发布 6 种事件（`OrderLineEvent` × 3、`HistoryEntryEvent` × 3）
  - **dryRun 判断后**：发布 4 种事件（`HistoryEntryEvent`(ORDER_REFUND_TRANSITION)、`RefundStateTransitionEvent`、`OrderEvent`('updated')、`HistoryEntryEvent`(ORDER_MODIFIED)）
  - **dryRun=true 时**：判断后 4 种事件**完全不会发布**，连阻塞处理器也不会收到
- **事件层面的一致性**：通过 `EventBus.awaitActiveTransactions` 机制双重保证
  - `ofType()` 订阅者（绝大多数插件）在事务回滚时**完全看不到任何事件**
  - 只有阻塞事件处理器会被调用，但仅能看到 dryRun 判断前的 6 种事件
  - 阻塞处理器内的 DB 操作也在同一事务内 → 会被回滚
- **唯一风险点**：阻塞事件处理器内的外部副作用（发送邮件、调用外部 API）不会回滚，但阻塞处理器使用极少

### 9.8 事件-历史记录-数据库写入对应关系（完整清单）

| 序号 | 事件类型 | 历史记录类型 | 数据库写入位置 | dryRun 前后 |
|------|----------|--------------|----------------|-------------|
| 1 | `OrderLineEvent`('created') | --- | `order-modifier.ts:195`（OrderLine 保存） | ✅ 之前 |
| 2 | `OrderLineEvent`('updated') | --- | `order-modifier.ts:246`（OrderLine 保存） | ✅ 之前 |
| 3 | `OrderLineEvent`('cancelled') | --- | `order-modifier.ts:334`（OrderLine 保存） | ✅ 之前 |
| 4 | `HistoryEntryEvent` | `ORDER_CANCELLATION` | `order-modifier.ts:362`（历史记录保存） | ✅ 之前 |
| 5 | `HistoryEntryEvent` | `ORDER_COUPON_APPLIED` | `order-modifier.ts:588`（历史记录保存） | ✅ 之前 |
| 6 | `HistoryEntryEvent` | `ORDER_COUPON_REMOVED` | `order-modifier.ts:599`（历史记录保存） | ✅ 之前 |
| 7 | `HistoryEntryEvent` | `ORDER_REFUND_TRANSITION` | `default-refund-process.ts:41`（历史记录保存） | ❌ 之后 |
| 8 | `RefundStateTransitionEvent` | --- | `payment.service.ts:450`（Refund 保存） | ❌ 之后 |
| 9 | `OrderEvent`('updated') | --- | `order-modifier.ts:693`（Order 保存） | ❌ 之后 |
| 10 | `HistoryEntryEvent` | `ORDER_MODIFIED` | `order.service.ts:1399`（历史记录保存） | ❌ 之后 |

> **说明**：`OrderLineEvent` 和 `OrderEvent`、`RefundStateTransitionEvent` 本身不直接对应历史记录，它们是领域事件，用于通知外部系统订单状态变更；`HistoryEntryEvent` 对应 `OrderHistoryEntry` 实体的增删改，是审计记录的直接载体。
