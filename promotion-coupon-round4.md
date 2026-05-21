# 促销与优惠券结算链路深挖（Round 4）

本文档对促销与优惠券链路做精准校对，重点澄清三个问题：
1. EventBus 在事务回滚时的可见性行为
2. 副作用失败导致回滚的结论限定到有 `@Transaction` 保护的调用路径
3. 阻塞事件处理与普通订阅分发的差异对比

---

## 一、EventBus 在事务回滚时的可见性行为

### 1.1 EventBus 的两层发布机制

`EventBus.publish()` 内部有两条路径，行为截然不同：

```typescript
// event-bus.ts:116-119
async publish<T extends VendureEvent>(event: T): Promise<void> {
    this.eventStream.next(event);                        // 路径1：普通订阅
    await this.executeBlockingEventHandlers(event);      // 路径2：阻塞订阅
}
```

**关键代码事实：** 两条路径在同一个 `publish()` 调用中执行，但行为模式完全不同。

### 1.2 路径1：普通订阅（ofType / filter）

```typescript
// event-bus.ts:130-136
ofType<T extends VendureEvent>(type: Type<T>): Observable<T> {
    return this.eventStream.asObservable().pipe(
        takeUntil(this.destroy$),
        filter(e => e.constructor === type),
        mergeMap(event => this.awaitActiveTransactions(event)),  // ⚠️  等待事务完成
        filter(notNullOrUndefined),
    ) as Observable<T>;
}
```

**`awaitActiveTransactions` 的关键逻辑：**

```typescript
// event-bus.ts:311-347
private async awaitActiveTransactions<T extends VendureEvent>(event: T): Promise<T | undefined> {
    const entry = Object.entries(event).find(([_, value]) => value instanceof RequestContext);

    if (!entry) {
        return event;  // 无 RequestContext → 直接返回（不等待事务）
    }

    const [key, ctx]: [string, RequestContext] = entry;
    const transactionManager: EntityManager | undefined = (ctx as any)[TRANSACTION_MANAGER_KEY];
    if (!transactionManager?.queryRunner) {
        return event;  // 无活跃事务 → 直接返回
    }

    try {
        // ⚠️  等待事务 COMMIT
        await this.transactionSubscriber.awaitCommit(transactionManager.queryRunner);

        // 清除事务管理器，防止使用已释放的 QueryRunner
        const newContext = ctx.copy();
        delete (newContext as any)[TRANSACTION_MANAGER_KEY];
        (event as any)[key] = newContext;

        return event;
    } catch (e: any) {
        if (e instanceof TransactionSubscriberError) {
            // ⚠️  事务回滚 → 返回 undefined，订阅者不会被调用
            return;
        }
        throw e;
    }
}
```

**`awaitCommit` 的实现（transaction-subscriber.ts）：**

```typescript
// transaction-subscriber.ts:73-75
awaitCommit(queryRunner: QueryRunner): Promise<QueryRunner> {
    return this.awaitTransactionEvent(queryRunner, 'commit');
}

// transaction-subscriber.ts:85-113
private awaitTransactionEvent(queryRunner, type?) {
    if (queryRunner.isTransactionActive) {
        return lastValueFrom(this.subject$
            .pipe(
                filter(event => !event.queryRunner.isTransactionActive && event.queryRunner === queryRunner),
                take(1),
                tap(event => {
                    if (type && event.type !== type) {
                        // ⚠️  收到的是 rollback 而非 commit → 抛出 TransactionSubscriberError
                        throw new TransactionSubscriberError(`Unexpected event type: ${event.type}. Expected ${type}.`);
                    }
                }),
                map(event => event.queryRunner),
                delay(0),
            )
        );
    }
    // 事务已结束 → 直接返回
    return Promise.resolve(queryRunner);
}
```

### 1.3 路径1 的事务回滚行为

**准确结论：**

| 场景 | 普通订阅者是否被调用 | 原因 |
|-----|---------------------|------|
| 事件包含 `RequestContext` 且事务 COMMIT | ✅ 调用 | `awaitCommit` 返回事件，传递给订阅者 |
| 事件包含 `RequestContext` 且事务 ROLLBACK | ❌ 不调用 | `awaitCommit` 抛出 `TransactionSubscriberError`，`filter(notNullOrUndefined)` 过滤掉 |
| 事件**不包含** `RequestContext` | ✅ 调用（不等待事务） | `awaitActiveTransactions` 直接返回事件 |
| 无活跃事务 | ✅ 调用（不等待事务） | `awaitActiveTransactions` 直接返回事件 |

**对促销链路的影响：**

```typescript
// order.service.ts:1080
await this.eventBus.publish(new CouponCodeEvent(ctx, couponCode, orderId, 'assigned'));
// CouponCodeEvent 包含 RequestContext
// 如果后续事务回滚，普通订阅者不会收到此事件
```

```typescript
// order.service.ts:901
const updatedOrder = await this.applyPriceAdjustments(ctx, order, updatedOrderLines, relations);
// applyPriceAdjustments 内部没有 publish 调用
// 但副作用可能触发额外的事件
```

### 1.4 路径2：阻塞事件处理（registerBlockingEventHandler）

```typescript
// event-bus.ts:215-232
private async executeBlockingEventHandlers<T extends VendureEvent>(event: T): Promise<void> {
    const blockingHandlers = this.blockingEventHandlers.get(event.constructor as Type<T>);
    for (const options of blockingHandlers || []) {
        const timeStart = new Date().getTime();
        await options.handler(event);  // ⚠️  同步执行，不等待事务
        const timeEnd = new Date().getTime();
        // 超过 100ms 警告
        if (timeTaken > 100) {
            Logger.warn(`Blocking event handler ${options.id} took ${timeTaken}ms`);
        }
    }
}
```

**关键代码事实：**
- 阻塞处理器**在 `publish()` 调用时同步执行**
- 不经过 `awaitActiveTransactions`
- 不等待事务 commit/rollback
- 阻塞处理器抛出的异常会导致 `publish()` 抛出

### 1.5 路径2 的事务回滚行为

**准确结论：**

| 场景 | 阻塞处理器是否执行 | 说明 |
|-----|------------------|------|
| 事务回滚前调用 `publish()` | ✅ 已执行 | 同步执行，无法撤回 |
| 阻塞处理器内部使用事件的 `ctx` 做 DB 操作 | ⚠️ 执行但数据随事务回滚 | 如果用 `ctx` 获取的 EntityManager 在同一事务中 |
| 阻塞处理器抛出异常 | ❌ `publish()` 抛出 | 会导致调用链中断 |

**对促销链路的影响：**

在 `applyCouponCode` 流程中：
```typescript
await this.eventBus.publish(new CouponCodeEvent(ctx, couponCode, orderId, 'assigned'));
// 阻塞处理器此时已执行
// 后续 applyPriceAdjustments 如果失败回滚，阻塞处理器无法撤回
// 但阻塞处理器内使用 ctx 的 DB 操作会随事务回滚
```

### 1.6 完整时序图（事务回滚场景）

```
@Transaction() applyCouponCode resolver
        │
        ▼
orderService.applyCouponCode
        │
        ├─→ couponCodes.push(couponCode)        [内存修改]
        │
        ├─→ historyService.createHistoryEntry() [DB 操作，在事务中]
        │
        ├─→ eventBus.publish(CouponCodeEvent)   [事务回滚时：]
        │       │                                [  ├─ 阻塞处理器：已同步执行 ✓]
        │       │                                [  └─ 普通订阅：不调用 ✗]
        │
        └─→ applyPriceAdjustments
                │
                ├─→ Order.save()                  [DB 操作，在事务中]
                │
                ├─→ runPromotionSideEffects()     [⚠️ 如果此处抛异常]
                │
                └─→ ❌ 异常 → 事务回滚
                        │
                        ├─ Order.save() 回滚
                        ├─ historyService 回滚
                        ├─ 阻塞处理器内的 DB 操作回滚
                        ├─ 普通订阅者从未被调用
                        └─ 阻塞处理器内的外部 API 调用无法回滚
```

---

## 二、副作用失败导致回滚的结论限定

### 2.1 `@Transaction` 保护的调用路径

**代码事实：** `applyPriceAdjustments()` 方法本身**没有** `@Transaction` 装饰器：

```typescript
// order.service.ts:2312 — 没有 @Transaction()
async applyPriceAdjustments(
    ctx: RequestContext,
    order: Order,
    updatedOrderLines?: OrderLine[],
    relations?: RelationPaths<Order>,
): Promise<Order> {
```

但它在以下场景中被调用，且调用链的 resolver 层有 `@Transaction()`：

| 调用场景 | Resolver 装饰 | 事务保护 |
|---------|--------------|---------|
| `applyCouponCode` → `applyPriceAdjustments` | ✅ `@Transaction()` | ✅ 有 |
| `addItemToOrder` → `applyPriceAdjustments` | ✅ `@Transaction()` | ✅ 有 |
| `addItemsToOrder` → `applyPriceAdjustments` | ✅ `@Transaction()` | ✅ 有 |
| `adjustOrderLine` → `applyPriceAdjustments` | ✅ `@Transaction()` | ✅ 有 |
| `removeOrderLine` → `applyPriceAdjustments` | ✅ `@Transaction()` | ✅ 有 |
| `setCurrencyCodeForOrder` → `applyPriceAdjustments` | ✅ `@Transaction()` | ✅ 有 |
| `applyCouponCodeToDraftOrder` → `applyPriceAdjustments` | ✅ `@Transaction()` | ✅ 有 |
| `addPaymentToOrder` → `revalidateCouponCodesForOrder` → `applyPriceAdjustments` | ✅ `@Transaction()` | ✅ 有 |
| `order-testing.service.ts` 测试调用 | ❌ 无 | ❌ 无 |
| 外部插件直接调用 `orderService.applyPriceAdjustments` | 取决于调用者 | ⚠️ 可能无 |

**准确结论：**

> 副作用失败导致事务回滚的结论，**仅在调用链经过 `@Transaction()` 装饰的 resolver 或显式事务包裹时成立**。
>
> 如果 `applyPriceAdjustments` 被外部代码在无事务上下文中直接调用，副作用失败会导致异常抛出，但不会回滚任何已完成的 DB 操作（因为没有事务）。

### 2.2 事务回滚的具体机制

```typescript
// transaction-wrapper.ts:26-79
async executeInTransaction(originalCtx, work, mode, isolationLevel, connection) {
    const ctx = originalCtx.copy();
    // ...
    try {
        const result = await lastValueFrom(from(work(ctx)).pipe(...));
        if (queryRunner.isTransactionActive) {
            await queryRunner.commitTransaction();  // 正常返回 → 提交
        }
        return result;
    } catch (error) {
        if (queryRunner.isTransactionActive) {
            await queryRunner.rollbackTransaction();  // 异常 → 回滚
        }
        throw error;
    } finally {
        // ...
    }
}
```

**回滚范围：** 所有使用同一 `QueryRunner` 的 DB 操作，包括：
- `Order.save()`（副作用前）
- `historyService.createHistoryEntry()`（副作用前）
- 阻塞事件处理器内使用事件 `ctx` 的 DB 操作
- `OrderLine.save()` / `ShippingLine.save()`（副作用后，未执行）

---

## 三、阻塞事件处理与普通订阅分发的差异对比

### 3.1 差异总览

| 维度 | 普通订阅（ofType / filter） | 阻塞订阅（registerBlockingEventHandler） |
|-----|---------------------------|--------------------------------------|
| **执行时机** | 事务 COMMIT 后（异步） | `publish()` 调用时（同步） |
| **事务回滚时** | ❌ 不调用 | ✅ 已执行（无法撤回） |
| **事务感知** | ✅ 等待事务完成后执行 | ❌ 不感知事务状态 |
| **DB 操作安全性** | ✅ 安全（事务已提交） | ⚠️ 使用事件 ctx 的操作在同一事务中 |
| **异常处理** | 不影响 `publish()` | ❌ 异常导致 `publish()` 抛出 |
| **执行顺序** | 不确定 | 通过 `before` / `after` 精确控制 |
| **性能影响** | 不阻塞主流程 | ⚠️ 阻塞主流程，>100ms 告警 |
| **注册方式** | `eventBus.ofType().subscribe()` | `eventBus.registerBlockingEventHandler()` |
| **取消注册** | `subscription.unsubscribe()` | 无取消机制（注册后永久生效） |
| **多处理器** | 多个 subscribe，顺序不确定 | 多个 handler，通过 before/after 排序 |

### 3.2 普通订阅详解

```typescript
// 注册方式
this.eventBus
    .ofType(CouponCodeEvent)
    .pipe(filter(event => event.type === 'assigned'))
    .subscribe(async (event) => {
        // 安全：此时事务已 COMMIT
        // event.ctx 已清除 TRANSACTION_MANAGER_KEY
        // 可以安全使用 event.ctx 做 DB 查询
    });
```

**特点：**
- 订阅者在事务 COMMIT 后才被调用
- 如果事务 ROLLBACK，订阅者完全不被调用
- 订阅者内部的异常不影响发布者
- 多个订阅者的执行顺序不确定

### 3.3 阻塞订阅详解

```typescript
// 注册方式
eventBus.registerBlockingEventHandler({
    event: CouponCodeEvent,
    id: 'my-coupon-handler',
    handler: async (event) => {
        // ⚠️  事务可能仍在进行中
        // 使用 event.ctx 的 DB 操作在同一事务中
        // 如果事务回滚，这些操作也回滚
        // 但外部 API 调用无法回滚
    },
    before: 'some-other-handler',  // 可选：控制执行顺序
});
```

**特点：**
- 在 `publish()` 调用时同步执行
- 阻塞主流程直到所有阻塞处理器完成
- 异常会导致 `publish()` 抛出
- 通过 `before` / `after` 精确控制执行顺序
- 超过 100ms 会记录警告

### 3.4 对促销链路的具体影响

在 `applyCouponCode` 流程中：

```typescript
// order.service.ts:1080
await this.eventBus.publish(new CouponCodeEvent(ctx, couponCode, orderId, 'assigned'));
// 此时：
// 1. 阻塞处理器已同步执行完成
// 2. 普通订阅者尚未执行（等待事务 COMMIT）
// 3. 如果后续 applyPriceAdjustments 失败回滚：
//    - 阻塞处理器内的 DB 操作随事务回滚
//    - 阻塞处理器内的外部 API 调用无法回滚
//    - 普通订阅者不被调用

return this.applyPriceAdjustments(ctx, order);
```

### 3.5 使用建议

| 场景 | 推荐方式 | 原因 |
|-----|---------|------|
| 发送通知邮件 | 普通订阅 | 不需要阻塞主流程，事务回滚不发通知 |
| 更新订单自定义字段 | 阻塞订阅 | 需要在事务内完成，随事务一起回滚 |
| 调用外部支付 API | 普通订阅 | 事务回滚不调用外部 API |
| 数据一致性校验 | 阻塞订阅 | 校验失败需要阻止整个操作 |
| 记录日志 | 普通订阅 | 不需要阻塞，事务回滚的操作不记录 |

---

## 四、对 Round 3 的修正说明

| Round 3 表述 | Round 4 修正（精准限定） |
|-------------|------------------------|
| "副作用执行失败会导致整个事务完全回滚" | 仅在调用链经过 `@Transaction()` 装饰的 resolver 或显式事务包裹时成立 |
| "外部 API 调用无法回滚" | 补充：阻塞事件处理器内的外部 API 调用同样无法回滚 |
| — | 新增：普通订阅者在事务回滚时完全不被调用 |
| — | 新增：阻塞处理器内使用事件 ctx 的 DB 操作随事务回滚 |

---

## 五、关键代码位置汇总

| 功能 | 文件位置 |
|-----|---------|
| EventBus publish | `event-bus.ts:116-119` |
| ofType / 普通订阅 | `event-bus.ts:130-136` |
| awaitActiveTransactions | `event-bus.ts:311-347` |
| 阻塞处理器执行 | `event-bus.ts:215-232` |
| 阻塞处理器注册 | `event-bus.ts:188-208` |
| TransactionSubscriber | `transaction-subscriber.ts:50-114` |
| awaitCommit | `transaction-subscriber.ts:73-75` |
| TransactionWrapper | `transaction-wrapper.ts:26-79` |
| @Transaction 装饰器 | `api/decorators/transaction.decorator.ts:81-89` |
| TransactionInterceptor | `api/middleware/transaction-interceptor.ts:29-60` |
| CouponCodeEvent | `event-bus/events/coupon-code-event.ts:15-24` |
| CouponCodeEvent 发布点 | `order.service.ts:1080` |
| 副作用调用点 | `order.service.ts:2410` |

---

## 六、精准结论速查

### EventBus 行为

| 情况 | 普通订阅 | 阻塞处理器 |
|-----|---------|-----------|
| 事务 COMMIT | ✅ 调用 | ✅ 已执行 |
| 事务 ROLLBACK | ❌ 不调用 | ✅ 已执行（DB 操作随事务回滚） |
| 无事务 | ✅ 调用 | ✅ 已执行 |
| 事件无 RequestContext | ✅ 调用（不等待事务） | ✅ 已执行 |

### 副作用失败回滚

| 调用路径 | 有 @Transaction | 回滚行为 |
|---------|----------------|---------|
| 所有 GraphQL mutation resolver | ✅ | 全部 DB 操作回滚 |
| 外部代码直接调用 | ❌ | 异常抛出，DB 操作不回滚 |
