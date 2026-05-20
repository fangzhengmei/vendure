# 促销规则与优惠券叠加判定顺序分析

本文档基于 Vendure 源码分析促销规则与优惠券在购物车结算时的判定顺序，包括条件匹配评估、动作应用次序与价格调整的协作关系。

---

## 一、核心实体与概念

### 1.1 Promotion 实体

`Promotion` 是促销的核心实体，每个促销可以包含：
- **条件（Conditions）**：`PromotionCondition[]`，满足所有条件才会应用促销
- **动作（Actions）**：`PromotionAction[]`，条件满足后执行的折扣动作
- **优惠券码（couponCode）**：可选，若设置则需要订单包含对应券码
- **优先级分数（priorityScore）**：决定促销应用顺序

源码位置：`packages/core/src/entity/promotion/promotion.entity.ts:61-274`

### 1.2 促销动作类型

| 动作类型 | 作用范围 | 执行函数 |
|---------|---------|---------|
| `PromotionItemAction` | 订单项（按数量计算） | `ExecutePromotionItemActionFn` |
| `PromotionLineAction` | 订单行（整行计算） | `ExecutePromotionLineActionFn` |
| `PromotionOrderAction` | 整个订单 | `ExecutePromotionOrderActionFn` |
| `PromotionShippingAction` | 运费 | `ExecutePromotionShippingActionFn` |

源码位置：`packages/core/src/config/promotion/promotion-action.ts:354-540`

---

## 二、条件匹配评估机制

### 2.1 促销条件检查入口：`Promotion.test()`

```typescript
async test(ctx: RequestContext, order: Order): Promise<PromotionTestResult> {
    // 1. 有效性检查
    if (this.endsAt && this.endsAt < new Date()) return false;
    if (this.startsAt && this.startsAt > new Date()) return false;
    
    // 2. 优惠券检查（关键！）
    if (this.couponCode && !order.couponCodes.includes(this.couponCode)) {
        return false;
    }
    
    // 3. 遍历所有条件，全部满足才通过
    const promotionState: PromotionState = {};
    for (const condition of this.conditions) {
        const promotionCondition = this.allConditions[condition.code];
        const result = await promotionCondition.check(ctx, order, condition.args, this);
        
        if (!result) return false;
        
        if (typeof result === 'object') {
            promotionState[condition.code] = result;
        }
    }
    return promotionState;
}
```

源码位置：`packages/core/src/entity/promotion/promotion.entity.ts:199-229`

**关键设计：**
- 优惠券促销需要 `order.couponCodes` 包含对应券码
- 条件检查返回 `true` 或状态对象，状态对象会传递给动作执行函数
- **全部条件满足才通过**，短路逻辑

### 2.2 优惠券的特殊处理

优惠券本质上是一种特殊的促销（设置了 `couponCode` 字段），其验证分为两个阶段：

**阶段一：券码应用时验证（`applyCouponCode`）**
```typescript
async applyCouponCode(ctx: RequestContext, orderId: ID, couponCode: string) {
    const order = await this.getOrderOrThrow(ctx, orderId);
    if (order.couponCodes.includes(couponCode)) return order;
    
    // 验证：有效性、过期、使用限制
    const validationResult = await this.promotionService.validateCouponCode(
        ctx, couponCode, order.customer?.id
    );
    
    if (isGraphQlErrorResult(validationResult)) return validationResult;
    
    // 加入订单的 couponCodes 数组
    order.couponCodes.push(couponCode);
    return this.applyPriceAdjustments(ctx, order);
}
```

源码位置：`packages/core/src/service/services/order.service.ts:1056-1082`

**阶段二：促销测试时验证（`Promotion.test()`）**
```typescript
if (this.couponCode && !order.couponCodes.includes(this.couponCode)) {
    return false;
}
```

### 2.3 促销列表获取与过滤

```typescript
// 获取所有启用的促销，按 priorityScore 升序排列
const allPromotions = await this.promotionService.getActivePromotionsInChannel(ctx);

// 过滤掉已用尽使用次数的自动促销（非优惠券类）
const exhaustedIds = await this.promotionService.getExhaustedPromotionIds(
    ctx, allPromotions, customerId
);
const promotions = allPromotions.filter(p => !exhaustedIds.has(p.id.toString()));
```

源码位置：`packages/core/src/service/services/order.service.ts:2318-2330`

---

## 三、动作应用次序

### 3.1 优先级排序：`priorityScore`

促销的应用顺序由 `priorityScore` 决定，**分数越低越先应用**：

```typescript
// 计算方式：所有条件和动作的 priorityValue 之和
private calculatePriorityScore(input: CreatePromotionInput): number {
    const conditions = input.conditions?.map(...) || [];
    const actions = input.actions?.map(...) || [];
    return [...conditions, ...actions].reduce((score, op) => score + op.priorityValue, 0);
}
```

源码位置：`packages/core/src/service/services/promotion.service.ts:507-515`

**示例：**
- 最小订单金额条件：`priorityValue: 10`
- 订单百分比折扣动作：默认 `0`
- 组合后 `priorityScore = 10 + 0 = 10`

### 3.2 促销应用总体顺序

```typescript
private async applyPromotions(ctx: RequestContext, order: Order, promotions: Promotion[]) {
    // 第一步：应用订单项级促销（Item/Line Action）
    await this.applyOrderItemPromotions(ctx, order, promotions);
    
    // 第二步：应用订单级促销（Order Action）
    await this.applyOrderPromotions(ctx, order, promotions);
}
```

源码位置：`packages/core/src/service/helpers/order-calculator/order-calculator.ts:170-174`

### 3.3 订单项级促销应用：`applyOrderItemPromotions()`

```typescript
private async applyOrderItemPromotions(ctx, order, promotions) {
    for (const line of order.lines) {
        line.clearAdjustments();  // 清除之前的调整
        
        for (const promotion of promotions) {
            // ⚠️  关键：每次应用前重新测试！
            // 因为前一个促销可能改变了订单总价，导致后续促销不再满足条件
            const applicableOrState = await promotion.test(ctx, order);
            
            if (applicableOrState) {
                const state = typeof applicableOrState === 'object' ? applicableOrState : undefined;
                const adjustment = await promotion.apply(ctx, { orderLine: line }, state);
                
                if (adjustment) {
                    line.addAdjustment(adjustment);
                    this.calculateOrderTotals(order);  // 立即重新计算订单总价
                }
                this.addPromotion(order, promotion);
            }
        }
        this.calculateOrderTotals(order);
    }
}
```

源码位置：`packages/core/src/service/helpers/order-calculator/order-calculator.ts:184-212`

**关键点：**
1. **逐行处理**：遍历每个订单行
2. **逐促销测试**：每行都遍历所有促销
3. **每次重新测试**：应用每个促销前都重新调用 `promotion.test()`
4. **实时重算总价**：应用调整后立即 `calculateOrderTotals()`

### 3.4 订单级促销应用：`applyOrderPromotions()`

```typescript
private async applyOrderPromotions(ctx, order, promotions) {
    // 清除之前的分布式订单促销调整
    order.lines.forEach(line => {
        line.clearAdjustments(AdjustmentType.DISTRIBUTED_ORDER_PROMOTION);
    });
    
    // 先筛选出满足条件的促销
    const applicableOrderPromotions = await filterAsync(promotions, p =>
        p.test(ctx, order).then(Boolean)
    );
    
    for (const promotion of applicableOrderPromotions) {
        // ⚠️  同样：每次应用前重新测试
        const applicableOrState = await promotion.test(ctx, order);
        
        if (applicableOrState) {
            const state = typeof applicableOrState === 'object' ? applicableOrState : undefined;
            const adjustment = await promotion.apply(ctx, { order }, state);
            
            if (adjustment && adjustment.amount !== 0) {
                // 按比例分摊到各订单行
                const weights = order.lines.map(l => l.proratedLinePriceWithTax);
                const distribution = prorate(weights, adjustment.amount);
                
                order.lines.forEach((line, i) => {
                    const shareOfAmount = distribution[i];
                    // 继续分摊到订单项
                    const itemWeights = Array.from({ length: line.quantity }).map(() => line.unitPrice);
                    const itemDistribution = prorate(itemWeights, shareOfAmount);
                    
                    line.addAdjustment({
                        amount: shareOfAmount,
                        type: AdjustmentType.DISTRIBUTED_ORDER_PROMOTION,
                        data: { itemDistribution },
                        // ...
                    });
                });
                this.calculateOrderTotals(order);
            }
            this.addPromotion(order, promotion);
        }
    }
}
```

源码位置：`packages/core/src/service/helpers/order-calculator/order-calculator.ts:214-272`

**关键点：**
1. 订单级折扣需要**按比例分摊**到各订单行和订单项
2. 使用 `prorate()` 算法进行精确分摊（处理四舍五入问题）
3. 同样采用**每次重新测试**策略

### 3.5 运费促销应用：`applyShippingPromotions()`

```typescript
private async applyShippingPromotions(ctx, order, promotions) {
    const applicable = await filterAsync(promotions, p => p.test(ctx, order).then(Boolean));
    
    if (applicable.length) {
        order.shippingLines.forEach(line => line.clearAdjustments());
        
        for (const promotion of applicable) {
            const applicableOrState = await promotion.test(ctx, order);
            if (applicableOrState) {
                const state = typeof applicableOrState === 'object' ? applicableOrState : undefined;
                for (const shippingLine of order.shippingLines) {
                    const adjustment = await promotion.apply(ctx, { shippingLine, order }, state);
                    if (adjustment?.amount !== 0) {
                        shippingLine.addAdjustment(adjustment);
                    }
                }
                this.addPromotion(order, promotion);
            }
        }
    }
}
```

源码位置：`packages/core/src/service/helpers/order-calculator/order-calculator.ts:274-302`

---

## 四、价格调整的协作关系

### 4.1 总体价格调整流程：`applyPriceAdjustments()`

```typescript
async applyPriceAdjustments(ctx, order, promotions, updatedOrderLines = []) {
    order.promotions = [];  // 重置，因为所有促销都需要重新验证
    
    // 1. 确定税区
    const activeTaxZone = await taxZoneStrategy.determineTaxZone(...);
    order.taxZoneId = activeTaxZone.id;
    
    // 2. 先应用税到更新的订单行
    for (const updatedOrderLine of updatedOrderLines) {
        await this.applyTaxesToOrderLine(ctx, order, updatedOrderLine, getTaxRate);
    }
    this.calculateOrderTotals(order);
    
    // 3. 如果税区变化，先应用税到所有非折扣价格
    if (taxZoneChanged) {
        await this.applyTaxes(ctx, order, activeTaxZone);
    }
    
    // 4. ⚠️  核心：测试并应用促销
    const totalBeforePromotions = order.subTotal;
    await this.applyPromotions(ctx, order, promotions);
    
    // 5. 如果促销改变了价格，重新计算税金
    if (order.subTotal !== totalBeforePromotions) {
        await this.applyTaxes(ctx, order, activeTaxZone);
    }
    
    // 6. 应用运费和运费促销
    if (options?.recalculateShipping !== false) {
        await this.applyShipping(ctx, order);
        await this.applyShippingPromotions(ctx, order, promotions);
    }
    
    this.calculateOrderTotals(order);
    return order;
}
```

源码位置：`packages/core/src/service/helpers/order-calculator/order-calculator.ts:53-110`

### 4.2 税金与促销的协作时序

```
初始价格
   ↓
[税区变化？] → 是 → 应用税金（基于原价）
   ↓
保存促销前总价: totalBeforePromotions
   ↓
应用促销（订单项级 → 订单级）
   ↓
[总价变化？] → 是 → 重新应用税金（基于折扣后价格）
   ↓
应用运费
   ↓
应用运费促销
   ↓
计算最终总价
```

### 4.3 分摊算法：`prorate()`

订单级折扣需要按比例分摊到各订单行，使用精确分摊算法：

```typescript
export function prorate(weights: number[], amount: number): number[] {
    const totalWeight = weights.reduce((total, val) => total + val, 0);
    
    // 计算各部分的理论值和取整后的值
    for (const w of weights) {
        actual[i] = totalWeight === 0 ? amount / weights.length : amount * (w / totalWeight);
        rounded[i] = Math.floor(actual[i]);
        error[i] = actual[i] - rounded[i];
        added += rounded[i];
    }
    
    // 将剩余的一分钱按误差大小分配
    while (added < amount) {
        // 找误差最大的那一项加1
        const maxErrorIndex = ...;
        rounded[maxErrorIndex] += 1;
        error[maxErrorIndex] -= 1;
        added += 1;
    }
    
    return rounded;
}
```

源码位置：`packages/core/src/service/helpers/order-calculator/prorate.ts:11-47`

---

## 五、完整结算流程图

```
订单变更（加购/改数量/用券）
         ↓
applyPriceAdjustments() 入口
         ↓
┌─────────────────────────────────┐
│ 1. 获取促销列表                   │
│   - getActivePromotionsInChannel │
│   - 按 priorityScore 升序排序     │
│   - 过滤已用尽的自动促销          │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ 2. 应用税金（税区变化时）          │
│   - 基于原价计算                  │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ 3. applyPromotions()             │
│    ┌──────────────────────────┐  │
│    │ 3.1 订单项级促销           │  │
│    │   遍历每行 →              │  │
│    │     遍历每个促销 →        │  │
│    │       promotion.test()    │  │
│    │       promotion.apply()   │  │
│    │       calculateOrderTotals│  │
│    └──────────────────────────┘  │
│    ┌──────────────────────────┐  │
│    │ 3.2 订单级促销            │  │
│    │   筛选满足条件的促销 →    │  │
│    │     遍历每个促销 →        │  │
│    │       promotion.test()    │  │
│    │       promotion.apply()   │  │
│    │       prorate()分摊       │  │
│    │       calculateOrderTotals│  │
│    └──────────────────────────┘  │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ 4. 促销后总价变化？→ 重算税金     │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ 5. 应用运费                       │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ 6. 应用运费促销                   │
└─────────────────────────────────┘
         ↓
计算最终总价
         ↓
返回订单
```

---

## 六、关键设计考量

### 6.1 为什么每次应用前都要重新测试？

```typescript
// applyOrderItemPromotions 中的注释
// We need to test the promotion *again*, even though we've tested them for the line.
// This is because the previous Promotions may have adjusted the Order in such a way
// as to render later promotions no longer applicable.
```

**场景示例：**
- 促销A：满$100减$20
- 促销B：满$90减$10
- 订单原价$100

如果先应用A再应用B：
1. 原价$100 → A测试通过 → 减$20 → 现价$80
2. 测试B：$80 < $90 → 不通过 ✅

如果不重新测试直接应用：
1. 原价$100 → A测试通过，B测试通过
2. 应用A → $80
3. 应用B → $70（错误！因为此时已不满足B的条件）❌

### 6.2 为什么需要 priorityScore？

**场景示例：**
- 促销1：买一送一（`priorityScore = 5`）
- 促销2：满$50打9折（`priorityScore = 10`）
- 订单原价$60

如果先应用促销2：
1. 原价$60 → 打9折 → $54
2. 买一送一 → 可能最终低于$50
3. 但促销2已应用，造成错误

如果先应用促销1（priorityScore 小）：
1. 原价$60 → 买一送一 → 调整后价格
2. 测试促销2是否满足 → 正确判断

### 6.3 优惠券与自动促销的关系

| 维度 | 自动促销 | 优惠券促销 |
|-----|---------|-----------|
| 触发方式 | 自动应用 | 需要输入券码，加入 `order.couponCodes` |
| 条件检查 | `test()` 中只检查条件 | `test()` 中先检查券码是否存在，再检查条件 |
| 使用次数检查 | `getExhaustedPromotionIds()` 中批量检查 | `validateCouponCode()` 中单独检查 |
| priorityScore | 参与排序 | 参与排序（与自动促销混合） |

**关键点：** 优惠券本质上就是带 `couponCode` 字段的促销，应用顺序同样由 `priorityScore` 决定。

---

## 七、常见问题与注意事项

### 7.1 促销叠加规则

- **可以叠加**：多个促销可以同时应用，按 `priorityScore` 顺序依次应用
- **动态判定**：每个促销应用后，后续促销基于最新价格重新判定条件
- **全部通过**：促销内所有条件都满足才应用动作

### 7.2 订单级折扣的分摊

订单级折扣会被分摊到各订单行和订单项，标记为 `DISTRIBUTED_ORDER_PROMOTION` 类型。这样做的好处是：
- 退货时可以精确计算应退金额
- 税金计算基于分摊后的价格

### 7.3 税金计算时机

税金会计算两次（如果促销改变了价格）：
1. 促销前：基于原价（税区变化时）
2. 促销后：基于折扣后价格（如果总价变化）

这是因为税金通常是基于最终售价计算的。

---

## 八、核心代码位置汇总

| 功能 | 文件位置 |
|-----|---------|
| 促销实体 | `packages/core/src/entity/promotion/promotion.entity.ts` |
| 促销条件定义 | `packages/core/src/config/promotion/promotion-condition.ts` |
| 促销动作定义 | `packages/core/src/config/promotion/promotion-action.ts` |
| 订单计算器（核心） | `packages/core/src/service/helpers/order-calculator/order-calculator.ts` |
| 促销服务 | `packages/core/src/service/services/promotion.service.ts` |
| 订单服务（用券） | `packages/core/src/service/services/order.service.ts:1056-1082` |
| 分摊算法 | `packages/core/src/service/helpers/order-calculator/prorate.ts` |
