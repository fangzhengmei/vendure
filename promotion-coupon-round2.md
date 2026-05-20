# 促销与优惠券结算链路深挖（Round 2）

本文档继续深挖促销与优惠券的结算链路，重点阐述三个核心问题：
1. 券码校验通过与真正生效之间的时序关系
2. 同 `priorityScore` 并列促销的顺序来源及可预期性
3. 重算后激活与失活副作用的执行链路

---

## 一、券码校验通过与真正生效的时序关系

### 1.1 券码生命周期的四个关键节点

优惠券从用户输入到真正产生折扣效果，经历四个关键节点：

```
用户输入券码
      ↓
【节点1】applyCouponCode 验证并加入 order.couponCodes
      ↓
【节点2】applyPriceAdjustments 触发价格重算
      ↓
【节点3】Promotion.test() 检查 order.couponCodes 包含该券码
      ↓
【节点4】Promotion.apply() 执行折扣动作
      ↓
支付时
      ↓
【节点5】revalidateCouponCodesForOrder 悲观锁二次验证
```

### 1.2 节点1：券码应用时的验证

```typescript
async applyCouponCode(ctx: RequestContext, orderId: ID, couponCode: string) {
    const order = await this.getOrderOrThrow(ctx, orderId);
    
    // 重复检查
    if (order.couponCodes.includes(couponCode)) return order;
    
    // 验证：有效性、过期、使用限制
    const validationResult = await this.promotionService.validateCouponCode(
        ctx, couponCode, order.customer?.id
    );
    
    if (isGraphQlErrorResult(validationResult)) return validationResult;
    
    // ⚠️  关键点：只是把券码加入数组，并未立即产生折扣
    order.couponCodes.push(couponCode);
    
    // 记录历史 + 发布事件
    await this.historyService.createHistoryEntryForOrder(...);
    await this.eventBus.publish(new CouponCodeEvent(...));
    
    // 触发价格重算
    return this.applyPriceAdjustments(ctx, order);
}
```

源码位置：`packages/core/src/service/services/order.service.ts:1056-1082`

**此时的状态：**
- `order.couponCodes` 包含该券码 ✓
- 订单价格尚未更新 ✗
- 折扣效果尚未体现 ✗

### 1.3 节点3：促销测试时的券码检查

券码真正发挥作用是在 `Promotion.test()` 中：

```typescript
async test(ctx: RequestContext, order: Order): Promise<PromotionTestResult> {
    // 1. 有效期检查
    if (this.endsAt && this.endsAt < new Date()) return false;
    if (this.startsAt && this.startsAt > new Date()) return false;
    
    // 2. ⚠️  核心：检查订单是否包含该券码
    if (this.couponCode && !order.couponCodes.includes(this.couponCode)) {
        return false;
    }
    
    // 3. 检查所有促销条件
    for (const condition of this.conditions) {
        const result = await promotionCondition.check(...);
        if (!result) return false;
    }
    
    return promotionState;
}
```

源码位置：`packages/core/src/entity/promotion/promotion.entity.ts:199-229`

### 1.4 节点5：支付时的二次验证（悲观锁机制）

这是最容易被忽略但至关重要的一步。在 `addPaymentToOrder` 中，支付前会重新验证所有券码：

```typescript
async addPaymentToOrder(ctx: RequestContext, orderId: ID, input: PaymentInput) {
    this.assertInTransaction(ctx, 'OrderService.addPaymentToOrder');
    const order = await this.getOrderOrThrow(ctx, orderId);
    
    const totalWithTaxBeforeRevalidation = order.totalWithTax;
    
    // ⚠️  支付前强制重新验证所有优惠券
    const couponsRemoved = await this.revalidateCouponCodesForOrder(ctx, order);
    
    // 如果券码被移除导致总价变化，拒绝支付
    const freshOrder = couponsRemoved ? await this.getOrderOrThrow(ctx, orderId) : order;
    if (couponsRemoved && totalWithTaxBeforeRevalidation < freshOrder.totalWithTax) {
        return new PaymentFailedError({
            paymentErrorMessage: 'Order total changed during checkout because a coupon is no longer available...'
        });
    }
    
    // 继续支付流程...
}
```

源码位置：`packages/core/src/service/services/order.service.ts:1427-1467`

### 1.5 revalidateCouponCodesForOrder 的悲观锁实现

```typescript
private async revalidateCouponCodesForOrder(ctx: RequestContext, order: Order): Promise<boolean> {
    let removedAny = false;
    
    for (const couponCode of [...order.couponCodes]) {
        const promotion = await this.connection.getRepository(ctx, Promotion).findOne({...});
        
        if (promotion) {
            // ⚠️  获取悲观写锁，序列化并发支付尝试
            try {
                await this.connection
                    .getRepository(ctx, Promotion)
                    .createQueryBuilder('promotion')
                    .setLock('pessimistic_write')  // SELECT ... FOR UPDATE
                    .where('promotion.id = :id', { id: promotion.id })
                    .getOne();
            } catch (e) {
                // SQLite 不支持，继续
            }
        }
        
        // 再次验证使用次数
        const validationResult = await this.promotionService.validateCouponCode(
            ctx, couponCode, order.customer?.id, order.id
        );
        
        if (isGraphQlErrorResult(validationResult)) {
            order.couponCodes = order.couponCodes.filter(c => c !== couponCode);
            removedAny = true;
        }
    }
    
    if (removedAny) {
        await this.applyPriceAdjustments(ctx, order);
    }
    return removedAny;
}
```

源码位置：`packages/core/src/service/services/order.service.ts:1541-1593`

**设计意图（来自代码注释）：**
```
Acquire a pessimistic write lock on the promotion row to
serialize concurrent payment attempts. The lock is held
until the resolver's @Transaction() commits.
```

**并发场景说明：**
- 两个用户同时使用同一张限量优惠券（usageLimit=1）
- 没有锁的情况下：两个事务都能读到 count=0，都通过验证，导致超发
- 有锁的情况下：第二个事务会等待第一个事务提交，然后读到 count=1，验证失败

### 1.6 时序总结表

| 阶段 | 券码位置 | 折扣生效？ | 验证内容 |
|-----|---------|-----------|---------|
| 用户输入后 | 内存变量 | 否 | 无 |
| `applyCouponCode` 后 | `order.couponCodes`（DB 已存） | 否 | 有效期、使用次数（乐观） |
| `applyPriceAdjustments` 后 | `order.couponCodes` | 是 | `Promotion.test()` 检查 |
| `addPaymentToOrder` 时 | `order.couponCodes` | 可能被移除 | 悲观锁 + 二次验证 |

**关键结论：** 券码加入 `order.couponCodes` 不等于折扣已锁定，每次价格重算和支付时都会重新验证。

---

## 二、同 `priorityScore` 并列促销的顺序来源及可预期性

### 2.1 priorityScore 的排序规则

促销列表的获取和排序在 `getActivePromotionsInChannel` 中定义：

```typescript
getActivePromotionsInChannel(ctx: RequestContext) {
    return this.connection
        .getRepository(ctx, Promotion)
        .createQueryBuilder('promotion')
        .leftJoin('promotion.channels', 'channel')
        .leftJoinAndSelect('promotion.translations', 'translation')
        .where('channel.id = :channelId', { channelId: ctx.channelId })
        .andWhere('promotion.deletedAt IS NULL')
        .andWhere('promotion.enabled = :enabled', { enabled: true })
        .orderBy('promotion.priorityScore', 'ASC')  // ⚠️  只有一级排序
        .getMany()
        .then(promotions => promotions.map(p => this.translator.translate(p, ctx)));
}
```

源码位置：`packages/core/src/service/services/promotion.service.ts:289-301`

**关键点：** 查询只按 `priorityScore` 升序排序，**没有二级排序字段**。

### 2.2 同分数时的顺序来源

当多个促销具有相同的 `priorityScore` 时，顺序由以下因素决定：

#### 因素1：数据库的自然排序
- **PostgreSQL/MySQL**：通常按主键（`id`）升序，即先创建的排在前面
- **SQLite**：按 `ROWID` 升序，同样是插入顺序
- **但请注意**：这是数据库的默认行为，**并非 SQL 标准保证**，数据库优化器可能在特定查询计划中改变顺序

#### 因素2：多对多关联表的顺序
`order.promotions` 是通过 `addPromotion` 方法添加的：

```typescript
private addPromotion(order: Order, promotion: Promotion) {
    if (order.promotions && !order.promotions.find(p => idsAreEqual(p.id, promotion.id))) {
        order.promotions.push(promotion);  // ⚠️  按应用顺序 push
    }
}
```

源码位置：`packages/core/src/service/helpers/order-calculator/order-calculator.ts:375-379`

所以 `order.promotions` 数组的顺序是**促销实际被应用的顺序**，而非 `priorityScore` 排序顺序。

### 2.3 同分数促销的应用顺序示例

假设我们有三个促销：

| 促销 | priorityScore | 创建时间（id 顺序） |
|-----|--------------|-------------------|
| 促销A | 10 | 2024-01-01（id=1） |
| 促销B | 10 | 2024-01-02（id=2） |
| 促销C | 5 | 2024-01-03（id=3） |

**查询返回顺序：** [促销C (5), 促销A (10), 促销B (10)]

**应用时的实际顺序：**
```
遍历促销列表 → [C, A, B]
  ↓
C.test() → 通过 → C.apply() → 加入 order.promotions
  ↓
A.test() → 通过 → A.apply() → 加入 order.promotions
  ↓
B.test() → 通过（基于A应用后的价格） → B.apply() → 加入 order.promotions
  ↓
最终 order.promotions = [C, A, B]
```

### 2.4 可预期性分析

| 场景 | 可预期性 | 说明 |
|-----|---------|------|
| 不同 priorityScore | ✅ 完全可预期 | 分数低的先应用 |
| 同 priorityScore，单实例部署 | ⚠️ 基本可预期 | 通常按 id/创建顺序 |
| 同 priorityScore，不同数据库 | ❌ 不可预期 | 不同数据库排序规则可能不同 |
| 同 priorityScore，复杂查询 | ❌ 不可预期 | 数据库优化器可能改变顺序 |
| order.promotions 数组 | ✅ 可预期 | 按实际应用顺序排列 |

### 2.5 最佳实践建议

1. **避免依赖同分数顺序**：不要假设同分数促销的应用顺序
2. **显式设置 priorityValue**：通过调整条件和动作的 `priorityValue` 来确保预期顺序
3. **示例**：
   ```typescript
   // 确保促销A先于促销B
   const promotionConditionA = new PromotionCondition({
       code: 'condition_a',
       priorityValue: 5,  // 总和 = 5 + 0 = 5
       check: ...
   });
   
   const promotionConditionB = new PromotionCondition({
       code: 'condition_b', 
       priorityValue: 10,  // 总和 = 10 + 0 = 10
       check: ...
   });
   ```

---

## 三、重算后激活与失活副作用的执行链路

### 3.1 副作用的定义与用途

促销动作可以定义 `onActivate` 和 `onDeactivate` 副作用，用于处理价格计算之外的逻辑（如添加赠品、发送通知等）：

```typescript
export interface PromotionActionConfig<T extends ConfigArgs, U extends ...> {
    // ...
    /**
     * Invoked when the promotion becomes active. 
     * Can be used for things like adding a free gift to the order
     * or other side effects unrelated to price calculations.
     */
    onActivate?: PromotionActionSideEffectFn<T>;
    
    /**
     * Used to reverse or clean up any side effects 
     * executed as part of the onActivate function.
     */
    onDeactivate?: PromotionActionSideEffectFn<T>;
}
```

源码位置：`packages/core/src/config/promotion/promotion-action.ts:178-199`

### 3.2 副作用的执行时机：`runPromotionSideEffects`

```typescript
async runPromotionSideEffects(ctx: RequestContext, order: Order, promotionsPre: Promotion[]) {
    const promotionsPost = order.promotions;
    
    // 第一步：处理失活的促销（先失活，后激活）
    for (const activePre of promotionsPre) {
        if (!promotionsPost.find(p => idsAreEqual(p.id, activePre.id))) {
            // activePre 不再活跃，调用 deactivate
            await activePre.deactivate(ctx, order);
        }
    }
    
    // 第二步：处理新激活的促销
    for (const activePost of promotionsPost) {
        if (!promotionsPre.find(p => idsAreEqual(p.id, activePost.id))) {
            // activePost 是新活跃的，调用 activate
            await activePost.activate(ctx, order);
        }
    }
}
```

源码位置：`packages/core/src/service/services/promotion.service.ts:313-327`

### 3.3 `activate` / `deactivate` 的具体实现

```typescript
// Promotion 实体中的方法
async activate(ctx: RequestContext, order: Order) {
    for (const action of this.actions) {
        const promotionAction = this.allActions[action.code];
        await promotionAction.onActivate(ctx, order, action.args, this);
    }
}

async deactivate(ctx: RequestContext, order: Order) {
    for (const action of this.actions) {
        const promotionAction = this.allActions[action.code];
        await promotionAction.onDeactivate(ctx, order, action.args, this);
    }
}
```

源码位置：`packages/core/src/entity/promotion/promotion.entity.ts:231-243`

### 3.4 完整的执行链路

副作用在 `applyPriceAdjustments` 的**最后**调用，在订单保存**之后**：

```typescript
async applyPriceAdjustments(ctx, order, updatedOrderLines?, relations?) {
    // 1. 获取促销列表
    const allPromotions = await this.promotionService.getActivePromotionsInChannel(ctx);
    const activePromotionsPre = await this.promotionService.getActivePromotionsOnOrder(ctx, order.id);
    
    // 2. 过滤已用尽的促销
    const exhaustedIds = await this.promotionService.getExhaustedPromotionIds(...);
    const promotions = allPromotions.filter(p => !exhaustedIds.has(p.id.toString()));
    
    // 3. 计算订单项价格...
    
    // 4. ⚠️  核心价格计算
    const updatedOrder = await this.orderCalculator.applyPriceAdjustments(
        ctx, order, promotions, updatedOrderLines ?? []
    );
    
    // 5. 保存订单（先保存，后执行副作用）
    await this.connection.getRepository(ctx, Order).save(omit(updatedOrder, [...]);
    
    // 6. ⚠️  执行副作用（在保存之后！）
    await this.promotionService.runPromotionSideEffects(ctx, order, activePromotionsPre);
    
    // 7. 保存订单项
    await this.connection.getRepository(ctx, OrderLine).save(updatedOrder.lines, ...);
    
    return order;
}
```

源码位置：`packages/core/src/service/services/order.service.ts:2312-2414`

### 3.5 执行顺序的设计考量

**为什么先保存订单，后执行副作用？**

1. **原子性保障**：价格计算是核心业务，必须确保持久化成功
2. **副作用隔离**：副作用失败（如添加赠品失败）不影响价格计算结果
3. **数据一致性**：副作用执行时，订单已在数据库中处于最新状态

**潜在风险：**
- 副作用执行失败不会回滚订单保存，需要自行处理失败重试
- 如果副作用修改了订单（如添加赠品），需要额外保存

### 3.6 完整的副作用触发链路图

```
订单变更（加购/改数量/用券）
        ↓
applyPriceAdjustments 入口
        ↓
┌─────────────────────────────────┐
│ 保存重算前的促销列表             │
│ activePromotionsPre             │
└─────────────────────────────────┘
        ↓
┌─────────────────────────────────┐
│ 价格计算（orderCalculator）      │
│ - 清除 order.promotions         │
│ - 逐个测试并应用促销             │
│ - 重新填充 order.promotions      │
│   （按应用顺序）                 │
└─────────────────────────────────┘
        ↓
┌─────────────────────────────────┐
│ 保存订单到数据库                 │
└─────────────────────────────────┘
        ↓
┌─────────────────────────────────┐
│ runPromotionSideEffects         │
│  ┌──────────────────────────┐   │
│  │ 遍历 activePromotionsPre │   │
│  │ 不在 post 中 → deactivate│   │
│  └──────────────────────────┘   │
│  ┌──────────────────────────┐   │
│  │ 遍历 order.promotions    │   │
│  │ 不在 pre 中 → activate   │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
        ↓
保存订单项
        ↓
返回订单
```

### 3.7 副作用调用的其他入口

除了 `applyPriceAdjustments`，订单修改时也会调用：

```typescript
// order-modifier.ts 中
await this.orderCalculator.applyPriceAdjustments(...);
await this.promotionService.runPromotionSideEffects(ctx, order, activePromotionsPre);
```

源码位置：`packages/core/src/service/helpers/order-modifier/order-modifier.ts:637-640`

### 3.8 副作用使用示例（赠品场景）

虽然核心库中没有实际使用 `onActivate`/`onDeactivate` 的例子，但代码注释说明了典型用途：

```typescript
// 概念示例：添加免费赠品
const freeGiftAction = new PromotionOrderAction({
    code: 'free_gift',
    args: { giftVariantId: 'ID' },
    onActivate: async (ctx, order, args, promotion) => {
        // 添加赠品到订单
        await orderService.addItemToOrder(ctx, order.id, args.giftVariantId, 1);
    },
    onDeactivate: async (ctx, order, args, promotion) => {
        // 移除赠品
        const giftLine = order.lines.find(l => l.productVariantId === args.giftVariantId);
        if (giftLine) {
            await orderService.removeItemFromOrder(ctx, order.id, giftLine.id);
        }
    },
    execute: () => 0  // 价格调整为0，因为赠品是通过副作用添加的
});
```

---

## 四、关键代码位置汇总

| 功能 | 文件位置 |
|-----|---------|
| 券码应用 | `packages/core/src/service/services/order.service.ts:1056-1082` |
| 支付时二次验证 | `packages/core/src/service/services/order.service.ts:1541-1593` |
| 悲观锁获取 | `packages/core/src/service/services/order.service.ts:1564-1570` |
| 促销列表查询（排序） | `packages/core/src/service/services/promotion.service.ts:289-301` |
| 添加促销到订单 | `packages/core/src/service/helpers/order-calculator/order-calculator.ts:375-379` |
| 副作用执行 | `packages/core/src/service/services/promotion.service.ts:313-327` |
| activate/deactivate | `packages/core/src/entity/promotion/promotion.entity.ts:231-243` |
| 副作用调用点1 | `packages/core/src/service/services/order.service.ts:2410` |
| 副作用调用点2 | `packages/core/src/service/helpers/order-modifier/order-modifier.ts:640` |

---

## 五、核心结论速查

### 5.1 券码时序
- ✅ 加入 `order.couponCodes` ≠ 折扣锁定
- ✅ 每次价格重算都会重新检查券码有效性
- ✅ 支付前会用**悲观锁**二次验证，防止并发超发
- ❌ 不要假设券码应用后一定会产生折扣

### 5.2 同分数排序
- ✅ 不同 `priorityScore`：分数低的先应用（可预期）
- ❌ 同 `priorityScore`：依赖数据库自然排序（不可预期）
- ✅ `order.promotions` 数组：按实际应用顺序排列（可预期）
- 💡 最佳实践：通过 `priorityValue` 显式控制顺序

### 5.3 副作用执行
- ✅ 执行时机：订单保存**之后**，价格计算**之后**
- ✅ 执行顺序：先 `deactivate` 失活促销，后 `activate` 新激活促销
- ✅ 失败隔离：副作用失败不影响价格计算结果
- ❌ 副作用修改订单需自行处理保存逻辑
