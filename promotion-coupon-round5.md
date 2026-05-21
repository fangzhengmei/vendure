# 促销与优惠券结算链路深挖（Round 5）— 统一校订版

本文档对促销与优惠券链路的事务边界做统一校订，以"是否存在活跃事务"为唯一判定主线，修正前几轮中的绝对化表述，给出自洽的调用场景—事务语义—回滚结果对照表。

---

## 一、事务判定主线：是否存在活跃事务

### 1.1 唯一判定标准

副作用执行失败时，DB 操作能否回滚，**唯一判定标准**是：抛出异常时，`RequestContext` 中是否存在活跃的事务管理器（`TRANSACTION_MANAGER_KEY`）。

```typescript
// transaction-wrapper.ts:36-44
const entityManager = (ctx as any)[TRANSACTION_MANAGER_KEY];
let queryRunner = entityManager?.queryRunner;
if (!queryRunner || queryRunner.isReleased) {
    queryRunner = connection.createQueryRunner();
}
if (mode === 'auto') {
    await this.startTransaction(queryRunner, isolationLevel);
}
(ctx as any)[TRANSACTION_MANAGER_KEY] = queryRunner.manager;
```

**推论：**
- ✅ 有活跃事务 → 异常触发回滚
- ❌ 无活跃事务 → 异常直接抛出，DB 操作不回滚（无事务可回滚）

### 1.2 活跃事务的来源

事务可以来自以下四种途径，与调用入口无关：

| 途径 | 代码位置 | 说明 |
|-----|---------|------|
| ① `@Transaction()` 装饰器 | `api/decorators/transaction.decorator.ts:81-89` | resolver 方法上的装饰器，`TransactionInterceptor` 拦截并创建事务 |
| ② `@Transaction('manual')` + 显式调用 | `api/decorators/transaction.decorator.ts:17-24` | 手动模式，需显式调用 `startTransaction` / `commitOpenTransaction` |
| ③ `connection.withTransaction()` | `connection/transactional-connection.ts:215-232` | service 层内部的事务包裹 |
| ④ 外部调用者显式 `startTransaction` | `connection/transactional-connection.ts:239-244` | 插件、Job Worker、生命周期钩子等外部代码自行启动 |

**关键：** 事务的存在取决于执行上下文中是否有活跃的 `QueryRunner`，而非调用来自 resolver 还是外部代码。

---

## 二、调用场景—事务语义—回滚结果对照表

### 2.1 `applyPriceAdjustments` 的所有调用路径

| # | 调用场景 | 事务来源 | 事务语义 | 副作用失败回滚结果 |
|---|---------|---------|---------|-------------------|
| 1 | `applyCouponCode` mutation → `orderService.applyCouponCode` → `applyPriceAdjustments` | ① `@Transaction()` (shop-order.resolver.ts:452, draft-order.resolver.ts:210) | 自动事务，resolver 返回后 commit | ✅ 全部 DB 操作回滚 |
| 2 | `addItemToOrder` mutation → `orderService.addItemToOrder` → `applyPriceAdjustments` | ① `@Transaction()` (shop-order.resolver.ts:332) | 自动事务 | ✅ 回滚 |
| 3 | `addItemsToOrder` mutation → `orderService.addItemsToOrder` → `applyPriceAdjustments` | ① `@Transaction()` (shop-order.resolver.ts:356) | 自动事务 | ✅ 回滚 |
| 4 | `adjustOrderLine` mutation → `orderService.adjustOrderLine` → `applyPriceAdjustments` | ① `@Transaction()` (shop-order.resolver.ts:395) | 自动事务 | ✅ 回滚 |
| 5 | `removeOrderLine` mutation → `orderService.removeOrderLine` → `applyPriceAdjustments` | ① `@Transaction()` (shop-order.resolver.ts:407) | 自动事务 | ✅ 回滚 |
| 6 | `removeAllOrderLines` mutation → `orderService.removeAllOrderLines` → `applyPriceAdjustments` | ① `@Transaction()` (shop-order.resolver.ts:426) | 自动事务 | ✅ 回滚 |
| 7 | `setCurrencyCodeForOrder` mutation → `orderService.updateOrderCurrency` → `applyPriceAdjustments` | ① `@Transaction()` (shop-order.resolver.ts:378) | 自动事务 | ✅ 回滚 |
| 8 | `addPaymentToOrder` mutation → `orderService.addPaymentToOrder` → `revalidateCouponCodesForOrder` → `applyPriceAdjustments` | ① `@Transaction()` (shop-order.resolver.ts:482) | 自动事务 + `assertInTransaction` 校验 | ✅ 回滚 |
| 9 | `modifyOrder` mutation → `orderService.modifyOrder` → `orderModifier.modifyOrder` → `orderCalculator.applyPriceAdjustments` | ② `@Transaction('manual')` (admin/order.resolver.ts:197) | 手动事务，resolver 显式 commit/rollback | ✅ 回滚（手动模式下由 `rollBackTransaction` 处理） |
| 10 | `setShippingMethod` → `orderService.setShippingMethod` → `applyPriceAdjustments` | ① `@Transaction()` (shop-order.resolver.ts:259) | 自动事务 | ✅ 回滚 |
| 11 | `addItemToDraftOrder` / `adjustDraftOrderLine` / `removeDraftOrderLine` → `applyPriceAdjustments` | ① `@Transaction()` (draft-order.resolver.ts:84, 100, 116) | 自动事务 | ✅ 回滚 |
| 12 | `orderService.transitionToState` → 内部 `withTransaction` 包裹 | ③ `connection.withTransaction()` (order.service.ts:1296) | 自动事务（方法内部自行包裹） | ⚠️ **不适用** — `transitionToState` 不调用 `applyPriceAdjustments` |
| 13 | 外部插件直接调用 `orderService.applyPriceAdjustments`，**未**启动事务 | 无 | 无事务上下文 | ❌ 异常抛出，DB 操作不回滚 |
| 14 | 外部插件调用 `applyPriceAdjustments`，**且**通过 ③ 或 ④ 启动事务 | ③ 或 ④ | 有活跃事务 | ✅ 回滚 |
| 15 | Job Worker 中调用，**未**启动事务 | 无 | 无事务上下文 | ❌ 不回滚 |
| 16 | `order-testing.service.ts` 测试调用 | 无 | 测试环境无事务 | ❌ 不回滚 |

### 2.2 对前几轮表述的修正

| 轮次 | 原表述 | 修正 |
|-----|--------|------|
| Round 3 | "副作用失败导致整个事务完全回滚" | 限定为"调用链存在活跃事务时，副作用失败导致事务回滚" |
| Round 4 | "仅在调用链经过 `@Transaction()` 装饰的 resolver 时成立" | 扩大为"调用链中存在活跃事务（来源包括 ①②③④ 任一途径）时成立" |
| Round 4 | "外部代码直接调用 → 异常抛出，DB 操作不回滚" | 修正为"外部代码直接调用且**未**启动事务 → 不回滚；外部代码调用且**已**启动事务 → 回滚" |
| Round 4 | "所有 GraphQL mutation resolver → 有 @Transaction" | 修正为"调用 `applyPriceAdjustments` 的 mutation resolver **确实都**有 `@Transaction`，但这只是事实巧合，而非因果关系。判定标准是活跃事务的存在，而非调用入口类型" |

---

## 三、`assertInTransaction` 的保护机制

### 3.1 哪些方法强制要求事务

`order.service.ts` 中的某些方法会在入口处主动检查事务上下文：

```typescript
// order.service.ts:1610-1623
private assertInTransaction(ctx: RequestContext, methodName: string): void {
    const entityManager = (ctx as unknown as Record<symbol, EntityManager | undefined>)[
        TRANSACTION_MANAGER_KEY
    ];
    const queryRunner = entityManager?.queryRunner;
    if (!queryRunner || queryRunner.isReleased) {
        throw new InternalServerError(
            `${methodName} must be called within a transaction. ` +
                'Wrap the call with the @Transaction() resolver decorator, or — when ' +
                'invoking from outside a resolver — start a transaction via ' +
                'TransactionalConnection.startTransaction() before calling.',
        );
    }
}
```

源码位置：`packages/core/src/service/services/order.service.ts:1610-1623`

**调用此检查的方法：**
- `addPaymentToOrder`（order.service.ts:1432）

**不调用此检查的方法（包括 `applyPriceAdjustments`）：**
- `applyPriceAdjustments` 本身没有 `assertInTransaction`
- `applyCouponCode` 没有 `assertInTransaction`
- `addItemToOrder` 没有 `assertInTransaction`

**设计意图（来自代码注释）：**
```
Resolvers reach these methods through the @Transaction() decorator,
which is the supported entry path. Callers from outside the resolver
layer (background jobs, plugin services, lifecycle hooks) must wrap
the call themselves via TransactionalConnection.startTransaction()
or the equivalent helper.
```

源码位置：`packages/core/src/service/services/order.service.ts:1604-1608`

### 3.2 对促销链路的影响

由于 `applyPriceAdjustments` 本身不做 `assertInTransaction` 检查：
- ✅ resolver 层调用：有 `@Transaction()` → 安全
- ⚠️ 外部调用：可能没有事务 → 副作用失败不回滚
- ⚠️ 外部调用者需自行确保事务上下文

---

## 四、EventBus 与事务的协作（统一校订版）

### 4.1 EventBus 的两层机制与事务的关系

| 维度 | 普通订阅（ofType / filter） | 阻塞订阅（registerBlockingEventHandler） |
|-----|---------------------------|--------------------------------------|
| **事务依赖** | ✅ 依赖事务状态，等待 COMMIT | ❌ 不依赖事务状态 |
| **事务 COMMIT** | ✅ 调用订阅者 | ✅ 已执行 |
| **事务 ROLLBACK** | ❌ 不调用 | ✅ 已执行（DB 操作随事务回滚） |
| **无事务** | ✅ 直接调用 | ✅ 直接执行 |

### 4.2 事务回滚时 EventBus 行为的精确描述

```typescript
// event-bus.ts:311-347 — awaitActiveTransactions
// 事务回滚时：throw TransactionSubscriberError → filter(notNullOrUndefined) 过滤
// 结果：普通订阅者完全不被调用
```

```typescript
// event-bus.ts:215-232 — executeBlockingEventHandlers
// 事务回滚时：已经同步执行完毕，无法撤回
// 但内部使用事件 ctx 的 DB 操作随事务回滚
// 外部 API 调用无法回滚
```

### 4.3 对促销链路中事件的具体影响

在 `applyCouponCode` 流程中：

```
@Transaction() resolver
    │
    ├─ order.couponCodes.push(couponCode)        [内存]
    │
    ├─ historyService.createHistoryEntry(...)    [DB, 在事务中]
    │
    ├─ eventBus.publish(CouponCodeEvent)         [事件发布]
    │   │
    │   ├─ 阻塞处理器：同步执行完毕
    │   │   ├─ 内 DB 操作：随事务回滚
    │   │   └─ 外部 API 调用：无法回滚
    │   │
    │   └─ 普通订阅：尚未执行（等待事务 COMMIT）
    │       └─ 事务回滚 → 不调用 ✅
    │
    └─ applyPriceAdjustments
            ├─ Order.save()                      [DB, 在事务中]
            ├─ runPromotionSideEffects()         [可能抛异常]
            │   └─ ❌ 异常 → 事务回滚
            └─ OrderLine.save()                  [未执行]
```

**对 Round 4 的修正：** Round 4 中关于 EventBus 的描述是准确的，无需修正。

---

## 五、自洽结论速查表

| 问题 | 答案 | 判定依据 |
|-----|------|---------|
| 副作用失败时 DB 操作会回滚吗？ | 如果有活跃事务 → 会；没有 → 不会 | `RequestContext` 中是否有 `TRANSACTION_MANAGER_KEY` |
| 所有 resolver mutation 都有 `@Transaction` 吗？ | 不是，但调用 `applyPriceAdjustments` 的那些都有 | 代码事实（shop-order.resolver.ts, draft-order.resolver.ts） |
| 外部调用一定会不回滚吗？ | 不是，外部调用也可以启动事务 | ③ `withTransaction` 或 ④ `startTransaction` |
| `applyPriceAdjustments` 会主动检查事务吗？ | 不会 | 无 `assertInTransaction` 调用 |
| 哪些方法会主动检查事务？ | `addPaymentToOrder` | 有 `assertInTransaction` 调用 |
| 普通订阅者在事务回滚时会被调用吗？ | 不会 | `awaitActiveTransactions` 返回 `undefined` |
| 阻塞处理器在事务回滚时已执行吗？ | 已执行，DB 操作随事务回滚 | 同步执行，不等待事务 |

---

## 六、关键代码位置汇总

| 功能 | 文件位置 |
|-----|---------|
| 事务判定（唯一标准） | `transaction-wrapper.ts:36-44` |
| `@Transaction()` 装饰器 | `api/decorators/transaction.decorator.ts:81-89` |
| `TransactionInterceptor` | `api/middleware/transaction-interceptor.ts:29-60` |
| `withTransaction()` | `connection/transactional-connection.ts:215-232` |
| `startTransaction()` | `connection/transactional-connection.ts:239-244` |
| `assertInTransaction()` | `order.service.ts:1610-1623` |
| `addPaymentToOrder`（有 assert 检查） | `order.service.ts:1432` |
| `transitionToState`（内部 withTransaction） | `order.service.ts:1296` |
| `applyPriceAdjustments`（无 assert 检查） | `order.service.ts:2312` |
| 副作用调用点 | `order.service.ts:2410` |
| `awaitActiveTransactions` | `event-bus.ts:311-347` |
| 阻塞处理器执行 | `event-bus.ts:215-232` |
| TransactionSubscriber | `connection/transaction-subscriber.ts:50-114` |
