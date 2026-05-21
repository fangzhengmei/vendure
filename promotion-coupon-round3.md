# 促销与优惠券结算链路深挖（Round 3）

本文档基于代码可证实的事实，准确梳理促销与优惠券的完整结算链路，重点澄清三个核心问题：
1. 促销与优惠券完整链路的代码可证实版本
2. 副作用执行失败在事务上下文下的准确影响范围
3. 同 `priorityScore` 并列促销顺序的代码可证实结论与边界

---

## 一、促销与优惠券完整链路（代码可证实版）

### 1.1 链路全景（代码事实）

```
用户调用 applyCouponCode mutation
        ↓
【resolver 层】shop-order.resolver.ts:452-465
  @Transaction()  // ⚠️  整个方法被事务包裹
  async applyCouponCode(...) {
      return this.orderService.applyCouponCode(ctx, order.id, args.couponCode);
  }
        ↓
【service 层】order.service.ts:1056-1082
  async applyCouponCode(...) {
      // 1. 检查重复
      if (order.couponCodes.includes(couponCode)) return order;
      
      // 2. 验证券码有效性（有效期、使用次数）
      const validationResult = await this.promotionService.validateCouponCode(...);
      if (isGraphQlErrorResult(validationResult)) return validationResult;
      
      // 3. 加入 order.couponCodes 数组（内存修改，未 flush）
      order.couponCodes.push(couponCode);
      
      // 4. 记录历史 + 发布事件
      await this.historyService.createHistoryEntryForOrder(...);
      await this.eventBus.publish(new CouponCodeEvent(...));
      
      // 5. ⚠️  触发完整价格重算
      return this.applyPriceAdjustments(ctx, order);
  }
        ↓
【service 层】order.service.ts:2312-2414
  async applyPriceAdjustments(...) {
      // 1. 获取促销列表（按 priorityScore 升序）
      const allPromotions = await this.promotionService.getActivePromotionsInChannel(ctx);
      const activePromotionsPre = await this.promotionService.getActivePromotionsOnOrder(ctx, order.id);
      
      // 2. 过滤已用尽的自动促销
      const exhaustedIds = await this.promotionService.getExhaustedPromotionIds(...);
      const promotions = allPromotions.filter(p => !exhaustedIds.has(p.id.toString()));
      
      // 3. 更新订单项价格（如果有变化）
      // ...
      
      // 4. 核心价格计算（orderCalculator）
      const updatedOrder = await this.orderCalculator.applyPriceAdjustments(
          ctx, order, promotions, updatedOrderLines ?? []
      );
      
      // 5. ⚠️  保存订单（在事务中，未提交）
      await this.connection.getRepository(ctx, Order).save(omit(updatedOrder, [...]), { reload: false });
      
      // 6. ⚠️  执行副作用（没有 try-catch！）
      await this.promotionService.runPromotionSideEffects(ctx, order, activePromotionsPre);
      
      // 7. 保存订单项和运费行（在事务中，未提交）
      await this.connection.getRepository(ctx, OrderLine).save(updatedOrder.lines, { reload: false });
      await this.connection.getRepository(ctx, ShippingLine).save(order.shippingLines, { reload: false });
      
      // 8. 返回查询后的订单
      return assertFound(this.findOne(ctx, order.id, relations));
  }
        ↓
【事务提交】transaction-wrapper.ts:62-64
  // resolver 方法正常返回后，事务提交
  if (queryRunner.isTransactionActive) {
      await queryRunner.commitTransaction();
  }
```

### 1.2 价格计算核心（orderCalculator）

```typescript
// order-calculator.ts:53-110
async applyPriceAdjustments(ctx, order, promotions, updatedOrderLines) {
    order.promotions = [];  // 重置
    
    // 1. 确定税区 → 应用税金（税区变化时）
    // ...
    
    // 2. 保存促销前总价
    const totalBeforePromotions = order.subTotal;
    
    // 3. 应用促销（Item/Line → Order）
    await this.applyPromotions(ctx, order, promotions);
    
    // 4. 促销后总价变化？→ 重算税金
    if (order.subTotal !== totalBeforePromotions) {
        await this.applyTaxes(ctx, order, activeTaxZone);
    }
    
    // 5. 应用运费 → 应用运费促销
    await this.applyShipping(ctx, order);
    await this.applyShippingPromotions(ctx, order, promotions);
    
    this.calculateOrderTotals(order);
    return order;
}
```

### 1.3 支付时的二次验证（代码事实）

```typescript
// order.service.ts:1427-1593
@Transaction()
async addPaymentToOrder(...) {
    // ⚠️  支付前强制重新验证所有优惠券
    const couponsRemoved = await this.revalidateCouponCodesForOrder(ctx, order);
    
    // 券码被移除导致总价变化？→ 拒绝支付
    if (couponsRemoved && totalWithTaxBeforeRevalidation < freshOrder.totalWithTax) {
        return new PaymentFailedError(...);
    }
    
    // 继续支付流程...
}

private async revalidateCouponCodesForOrder(ctx, order) {
    for (const couponCode of [...order.couponCodes]) {
        const promotion = await this.connection.getRepository(ctx, Promotion).findOne(...);
        
        if (promotion) {
            // ⚠️  获取悲观写锁
            try {
                await this.connection
                    .getRepository(ctx, Promotion)
                    .createQueryBuilder('promotion')
                    .setLock('pessimistic_write')
                    .where('promotion.id = :id', { id: promotion.id })
                    .getOne();
            } catch (e) { /* SQLite 不支持 */ }
        }
        
        // 再次验证
        const validationResult = await this.promotionService.validateCouponCode(...);
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

---

## 二、副作用执行失败在事务上下文下的准确影响范围

### 2.1 事务上下文的范围

**代码事实：**
1. `applyCouponCode` resolver 被 `@Transaction()` 装饰（shop-order.resolver.ts:452）
2. `@Transaction()` 由 `TransactionInterceptor` 拦截，调用 `transactionWrapper.executeInTransaction()`
3. 整个 resolver 方法（包括所有 service 调用）在同一个事务中执行
4. 事务在 resolver 方法正常返回后 `commitTransaction()`，抛出异常时 `rollbackTransaction()`

源码：`transaction-wrapper.ts:26-79`

```typescript
async executeInTransaction(originalCtx, work, mode, isolationLevel, connection) {
    // ...
    try {
        const result = await lastValueFrom(from(work(ctx)).pipe(...));
        if (queryRunner.isTransactionActive) {
            await queryRunner.commitTransaction();  // ✅ 正常返回 → 提交
        }
        return result;
    } catch (error) {
        if (queryRunner.isTransactionActive) {
            await queryRunner.rollbackTransaction();  // ❌ 抛出异常 → 回滚
        }
        throw error;
    } finally {
        // ...
    }
}
```

### 2.2 副作用的执行位置与异常处理

**代码事实：**

```typescript
// order.service.ts:2388-2412
await this.connection.getRepository(ctx, Order).save(omit(updatedOrder, [...]), { reload: false });

// ⚠️  副作用执行：没有 try-catch，异常会穿透到事务层
await this.promotionService.runPromotionSideEffects(ctx, order, activePromotionsPre);

await this.connection.getRepository(ctx, OrderLine).save(updatedOrder.lines, { reload: false });
await this.connection.getRepository(ctx, ShippingLine).save(order.shippingLines, { reload: false });
```

```typescript
// promotion.service.ts:313-327
async runPromotionSideEffects(ctx: RequestContext, order: Order, promotionsPre: Promotion[]) {
    const promotionsPost = order.promotions;
    for (const activePre of promotionsPre) {
        if (!promotionsPost.find(p => idsAreEqual(p.id, activePre.id))) {
            await activePre.deactivate(ctx, order);  // ⚠️  无 try-catch
        }
    }
    for (const activePost of promotionsPost) {
        if (!promotionsPre.find(p => idsAreEqual(p.id, activePost.id))) {
            await activePost.activate(ctx, order);    // ⚠️  无 try-catch
        }
    }
}
```

### 2.3 影响范围分析（代码可证实）

| 操作 | 执行顺序 | 副作用失败时的命运 | 原因 |
|-----|---------|------------------|------|
| `Order.save()` | 副作用前 | ❌ 回滚 | 在同一事务中 |
| `runPromotionSideEffects()` 中已执行的 DB 操作 | 副作用中 | ❌ 回滚 | 在同一事务中 |
| `OrderLine.save()` | 副作用后 | ❌ 未执行 | 异常提前抛出 |
| `ShippingLine.save()` | 副作用后 | ❌ 未执行 | 异常提前抛出 |
| 券码加入 `order.couponCodes` | 副作用前 | ❌ 回滚 | 实体状态随事务回滚 |
| 价格计算结果 | 副作用前 | ❌ 回滚 | 实体状态随事务回滚 |
| 副作用中的外部 API 调用（如发通知） | 副作用中 | ⚠️ 无法回滚 | 不在 DB 事务范围内 |
| `EventBus.publish()` 事件 | 副作用前 | ⚠️ 取决于事件处理器 | 事件发布后无法撤回 |

**准确结论：**

> 由于 `runPromotionSideEffects` 没有任何 try-catch 保护，且整个调用链处于 `@Transaction()` 包裹的事务上下文中，**副作用执行失败会导致整个事务完全回滚**，包括副作用执行前已完成的 `Order.save()`。

**修正 Round 2 的不准确表述：**
- ❌ Round 2 表述："价格计算是核心业务，必须确保持久化成功"
- ✅ 代码事实：价格计算的持久化（`Order.save()`）在事务中，副作用失败会一起回滚
- ❌ Round 2 表述："副作用失败不影响价格计算结果"
- ✅ 代码事实：副作用失败导致价格计算结果一起回滚，用户看到的是错误，而非部分成功的订单

---

## 三、同 `priorityScore` 并列促销顺序的代码可证实结论与边界

### 3.1 代码可证实的结论

**结论1：查询层面只有一级排序**

```typescript
// promotion.service.ts:289-301
getActivePromotionsInChannel(ctx: RequestContext) {
    return this.connection
        .getRepository(ctx, Promotion)
        .createQueryBuilder('promotion')
        .leftJoin('promotion.channels', 'channel')
        .leftJoinAndSelect('promotion.translations', 'translation')
        .where('channel.id = :channelId', { channelId: ctx.channelId })
        .andWhere('promotion.deletedAt IS NULL')
        .andWhere('promotion.enabled = :enabled', { enabled: true })
        .orderBy('promotion.priorityScore', 'ASC')  // ⚠️  只有这一个 orderBy
        .getMany()
        .then(promotions => promotions.map(p => this.translator.translate(p, ctx)));
}
```

✅ **代码事实：** 只有 `.orderBy('promotion.priorityScore', 'ASC')`，没有 `.addOrderBy()` 二级排序。

**结论2：同分数顺序由数据库自然排序决定**

✅ **代码事实：** TypeORM 的 `getMany()` 在只有一级排序且同分时，顺序完全由数据库返回顺序决定。

**结论3：`order.promotions` 数组顺序是实际应用顺序**

```typescript
// order-calculator.ts:375-379
private addPromotion(order: Order, promotion: Promotion) {
    if (order.promotions && !order.promotions.find(p => idsAreEqual(p.id, promotion.id))) {
        order.promotions.push(promotion);  // ⚠️  按应用顺序 push
    }
}
```

✅ **代码事实：** `order.promotions` 数组的顺序与促销实际被应用的顺序一致，而非 `priorityScore` 查询顺序。

### 3.2 代码可证实的边界

| 假设 | 是否有代码支持 | 边界说明 |
|-----|--------------|---------|
| 同分数按 `id` 升序 | ❌ 无代码保证 | 数据库可能按 id，但这是数据库默认行为，非 SQL 标准 |
| 同分数按创建时间升序 | ❌ 无代码保证 | 创建时间通常与 id 正相关，但无 `orderBy('createdAt')` |
| 同分数按 `priorityValue` 排序 | ❌ 无代码保证 | `priorityValue` 只用于计算 `priorityScore`，不用于二级排序 |
| 不同数据库行为一致 | ❌ 无代码保证 | PostgreSQL/MySQL/SQLite 的排序规则可能不同 |
| 查询计划不影响顺序 | ❌ 无代码保证 | 复杂查询时数据库优化器可能改变返回顺序 |

### 3.3 确定性边界

✅ **可依赖的确定性：**
- 不同 `priorityScore`：分数低的先应用（100% 代码保证）
- `order.promotions` 数组：按实际应用顺序排列（100% 代码保证）

❌ **不可依赖的不确定性：**
- 同 `priorityScore` 的多个促销：应用顺序无代码级保证
- 同 `priorityScore` 的优惠券与自动促销：应用顺序无代码级保证

**最佳实践（代码层面的建议）：**
如果需要严格控制顺序，必须通过调整 `priorityValue` 使 `priorityScore` 不同，而非依赖同分数的隐式顺序。

---

## 四、可依赖性总结

### 4.1 可以100%依赖的代码行为

| 行为 | 代码位置 |
|-----|---------|
| 不同 `priorityScore` 的促销，分数低的先应用 | `promotion.service.ts:298` |
| 每个促销应用前都会重新调用 `test()` 检查条件 | `order-calculator.ts:192, 230` |
| 订单级折扣按比例分摊到订单项 | `order-calculator.ts:247-262` |
| 副作用在 `Order.save()` 之后、`OrderLine.save()` 之前执行 | `order.service.ts:2392-2411` |
| 副作用失败导致整个事务回滚 | `transaction-wrapper.ts:66-70` |
| 支付时会用悲观锁二次验证券码 | `order.service.ts:1561-1570` |
| `order.promotions` 按实际应用顺序排列 | `order-calculator.ts:377` |

### 4.2 不应该依赖的隐式行为

| 隐式假设 | 风险 |
|---------|------|
| 同 `priorityScore` 促销按 id/创建时间排序 | 数据库升级/数据变化可能改变顺序 |
| 优惠券一定在自动促销之后应用 | 取决于 `priorityScore`，券码本身不影响排序 |
| 副作用失败时价格计算已持久化 | 实际上会一起回滚 |
| 券码加入 `order.couponCodes` 后折扣锁定 | 每次重算和支付时都会重新验证 |

### 4.3 关键代码位置汇总

| 功能 | 文件位置 |
|-----|---------|
| 事务包裹 resolver | `shop-order.resolver.ts:452` |
| 事务提交/回滚 | `transaction-wrapper.ts:62-70` |
| 副作用执行（无 try-catch） | `promotion.service.ts:313-327` |
| 副作用调用点 | `order.service.ts:2410` |
| 促销列表排序 | `promotion.service.ts:289-301` |
| 支付时二次验证（悲观锁） | `order.service.ts:1541-1593` |
| 促销添加到 order | `order-calculator.ts:375-379` |

---

## 五、对 Round 2 的修正说明

| Round 2 表述 | Round 3 修正（代码事实） |
|-------------|------------------------|
| "价格计算是核心业务，必须确保持久化成功" | 价格计算的持久化在事务中，副作用失败会一起回滚 |
| "副作用失败不影响价格计算结果" | 副作用失败导致价格计算结果一起回滚 |
| "先保存订单，后执行副作用" | 描述正确，但补充：都在同一事务中，失败一起回滚 |
