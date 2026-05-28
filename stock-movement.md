# Vendure Stock Movement 代码理解

## 1. 核心数据模型

### 1.1 StockMovement（库存移动）

`StockMovement` 是 TypeORM 单表继承（STI）的抽象基类，`discriminator` 列区分 6 种子类型：

| 子类型 | discriminator | quantity 符号 | 关联 |
|--------|--------------|--------------|------|
| `StockAdjustment` | ADJUSTMENT | +/− | 无 OrderLine |
| `Allocation` | ALLOCATION | + | OrderLine |
| `Release` | RELEASE | + | OrderLine |
| `Sale` | SALE | − | OrderLine |
| `Cancellation` | CANCELLATION | + | OrderLine |
| `Return` | RETURN | （仅 GraphQL schema，尚无实体实现） | — |

**源码位置**：`packages/core/src/entity/stock-movement/stock-movement.entity.ts:21`

关键字段：
- `type: StockMovementType` — 只读，子类各自固定
- `productVariant: ProductVariant` — 库存变动所属 SKU
- `stockLocation: StockLocation` — 库存变动发生在哪个仓库
- `quantity: number` — 变动数量（Sale 为负数，其余通常为正数）

### 1.2 StockLevel（库存水位）

`StockLevel` 是 **每个 ProductVariant × StockLocation 的唯一行**，拥有唯一索引 `['productVariantId', 'stockLocationId']`。

**源码位置**：`packages/core/src/entity/stock-level/stock-level.entity.ts:18`

```typescript
class StockLevel {
    productVariantId: ID;
    stockLocationId: ID;
    stockOnHand: number;      // 在手库存
    stockAllocated: number;   // 已分配库存
}
```

**重要区别**：`StockMovement` 是不可变的审计日志（append-only），`StockLevel` 是可变的当前状态快照。

### 1.3 可售量公式

```
saleable = stockOnHand - stockAllocated - outOfStockThreshold
```

其中 `outOfStockThreshold` 可为负数（允许超卖/back order）。

**源码位置**：`packages/core/src/service/services/product-variant.service.ts:321` — `getSaleableStockLevel()`

---

## 2. StockMovement 创建时机

所有 StockMovement 的创建统一由 `StockMovementService` 负责。

**源码位置**：`packages/core/src/service/services/stock-movement.service.ts`

### 2.1 StockAdjustment — 手动调整

**创建方法**：`adjustProductVariantStock()`

**触发时机**：管理员通过 Dashboard 或 API 手动设置 `stockOnHand` 值。

**流程**：
1. 获取该 (variant, location) 的当前 `StockLevel.stockOnHand`
2. 计算 `delta = newStockLevel - oldStockLevel`
3. 如果 `delta === 0`，跳过（幂等）
4. 保存 `StockAdjustment` 实体（quantity = delta）
5. 调用 `StockLevelService.updateStockOnHandForLocation()` 更新 `StockLevel.stockOnHand += delta`
6. 发布 `StockMovementEvent`

**对 StockLevel 的影响**：`stockOnHand += delta`

### 2.2 Allocation — 库存分配

**创建方法**：`createAllocationsForOrder()` / `createAllocationsForOrderLines()`

**触发时机**：订单状态转换时，`StockAllocationStrategy.shouldAllocateStock()` 返回 `true`。

**默认策略**（`DefaultStockAllocationStrategy`）：
- 订单从 `ArrangingPayment` → `PaymentAuthorized` 或 `PaymentSettled` 时触发

**源码位置**：
- 分配策略：`packages/core/src/config/order/default-stock-allocation-strategy.ts`
- 调用入口：`packages/core/src/config/order/default-order-process.ts:427-434`

```typescript
const shouldAllocateStock = await stockAllocationStrategy.shouldAllocateStock(ctx, fromState, toState, order);
if (shouldAllocateStock) {
    await stockMovementService.createAllocationsForOrder(ctx, order);
}
```

**流程**：
1. 遍历每个 OrderLine
2. 调用 `StockLocationService.getAllocationLocations()` → 委托 `StockLocationStrategy.forAllocation()` 决定从哪些仓库、各分配多少
3. 对每个分配位置，创建 `Allocation` 实体（quantity > 0）
4. 如果 variant 启用了库存追踪，调用 `StockLevelService.updateStockAllocatedForLocation()` 更新 `StockLevel.stockAllocated += quantity`
5. 批量保存所有 Allocation
6. 发布 `StockMovementEvent`

**对 StockLevel 的影响**：`stockAllocated += quantity`（stockOnHand 不变）

### 2.3 Sale — 销售出库

**创建方法**：`createSalesForOrder()`

**触发时机**：Fulfillment 从 `Created` → `Pending` 状态时。

**源码位置**：`packages/core/src/config/fulfillment/default-fulfillment-process.ts:102-104`

```typescript
if (fromState === 'Created' && toState === 'Pending') {
    await stockMovementService.createSalesForOrder(ctx, fulfillment.lines);
}
```

**流程**：
1. 遍历每条 FulfillmentLine
2. 调用 `StockLocationService.getSaleLocations()` → 委托 `StockLocationStrategy.forSale()` 决定从哪些仓库扣减
3. 创建 `Sale` 实体（quantity 为负数：`-lineRow.quantity`）
4. 如果 variant 启用了库存追踪：
   - `stockAllocated -= quantity`（释放分配）
   - `stockOnHand -= quantity`（扣减在手库存）
5. 批量保存，发布事件

**对 StockLevel 的影响**：
- `stockAllocated -= quantity`（分配 → 销售，释放 allocated 占位）
- `stockOnHand -= quantity`（实际出库）

**理解要点**：Sale 同时做两件事——将 Allocation 转化为 Sale（释放 allocated），并减少 stockOnHand。这意味着 allocated 和 stockOnHand 同时减少相同的数量，可售量不变（`saleable = (stockOnHand - qty) - (stockAllocated - qty) - threshold = 原值`）。

### 2.4 Release — 释放分配

**创建方法**：`createReleasesForOrderLines()`

**触发时机**：
- **订单取消**：已分配但未发货的订单行被取消时
- **订单修改**：`OrderModifier.cancelOrderByOrderLines()` 中区分 fulfilled 和 allocated 部分，对 allocated 部分调用 release

**源码位置**：`packages/core/src/service/helpers/order-modifier/order-modifier.ts:331`

**流程**：
1. 调用 `StockLocationService.getReleaseLocations()` → 委托 `StockLocationStrategy.forRelease()` 决定从哪些仓库释放
2. 创建 `Release` 实体（quantity > 0）
3. 如果 variant 启用了库存追踪：`stockAllocated -= quantity`
4. 保存，发布事件

**对 StockLevel 的影响**：`stockAllocated -= quantity`（stockOnHand 不变）

**效果**：释放分配后，可售量回升。`saleable = stockOnHand - (stockAllocated - qty) - threshold = 原值 + qty`

### 2.5 Cancellation — 取消已发货商品

**创建方法**：`createCancellationsForOrderLines()`

**触发时机**：
- **Fulfillment 取消**：Fulfillment 转入 `Cancelled` 状态时，先创建 Cancellation（归还 stockOnHand），再创建 Allocation（重新占用 stockAllocated）
- **订单取消**：已发货的商品被取消时

**源码位置**：`packages/core/src/config/fulfillment/default-fulfillment-process.ts:94-101`

```typescript
if (toState === 'Cancelled') {
    await stockMovementService.createCancellationsForOrderLines(ctx, orderLineInput);
    await stockMovementService.createAllocationsForOrderLines(ctx, orderLineInput);
}
```

**流程**：
1. 调用 `StockLocationService.getCancellationLocations()` → 委托策略
2. 创建 `Cancellation` 实体（quantity > 0）
3. 如果 variant 启用了库存追踪：`stockOnHand += quantity`
4. 保存，发布事件

**对 StockLevel 的影响**：`stockOnHand += quantity`（stockAllocated 不变）

**Fulfillment 取消的特殊逻辑**：Cancellation 归还 stockOnHand 后立即创建新的 Allocation（stockAllocated += qty），这确保了如果订单还有效，库存仍然被"占用"——只是从"已发货"变为"已分配"。

### 2.6 订单取消中的 Release vs Cancellation 分流

`OrderModifier.cancelOrderByOrderLines()` 的逻辑（`packages/core/src/service/helpers/order-modifier/order-modifier.ts:270-331`）：

```
对每个 orderLine:
  计算 totalAllocated = sum(allocations.quantity) + sum(sales.quantity) - sum(releases.quantity)
  计算 totalFulfilled = sum(fulfillmentLines.quantity) - sum(cancellations.quantity)

  如果 totalAllocated > 0 → 该行加入 allocatedLines
  如果 totalFulfilled > 0 → 该行加入 fulfilledLines

对 fulfilledLines → createCancellationsForOrderLines()   // 归还 stockOnHand
对 allocatedLines → createReleasesForOrderLines()         // 释放 stockAllocated
```

---

## 3. 可用量（saleable）重算逻辑

可售量不存储在数据库中，而是**每次查询时实时计算**。

### 3.1 计算入口

`ProductVariantService.getSaleableStockLevel()` — `packages/core/src/service/services/product-variant.service.ts:321`

```typescript
async getSaleableStockLevel(ctx, variant): Promise<number> {
    const { stockOnHand, stockAllocated } = await this.stockLevelService.getAvailableStock(ctx, variant.id);
    const effectiveOutOfStockThreshold = variant.useGlobalOutOfStockThreshold
        ? outOfStockThreshold
        : variant.outOfStockThreshold;
    return stockOnHand - stockAllocated - effectiveOutOfStockThreshold;
}
```

### 3.2 StockLevelService.getAvailableStock()

`packages/core/src/service/services/stock-level.service.ts:72`

1. 查出该 variant 的**所有 StockLevel 行**（跨所有 StockLocation）
2. 委托 `StockLocationStrategy.getAvailableStock(ctx, variantId, stockLevels)` 做聚合

### 3.3 两种策略的聚合方式

#### DefaultStockLocationStrategy（单仓库）

```typescript
getAvailableStock(ctx, productVariantId, stockLevels) {
    let stockOnHand = 0, stockAllocated = 0;
    for (const sl of stockLevels) {
        stockOnHand += sl.stockOnHand;
        stockAllocated += sl.stockAllocated;
    }
    return { stockOnHand, stockAllocated };
}
```

简单汇总所有仓库的库存。

#### MultiChannelStockLocationStrategy（多渠道多仓库，v3.1+ 默认）

```typescript
async getAvailableStock(ctx, productVariantId, stockLevels) {
    let stockOnHand = 0, stockAllocated = 0;
    for (const sl of stockLevels) {
        const applies = await this.stockLevelAppliesToActiveChannel(ctx, sl);
        if (applies) {
            stockOnHand += sl.stockOnHand;
            stockAllocated += sl.stockAllocated;
        }
    }
    return { stockOnHand, stockAllocated };
}
```

只汇总与当前 Channel 关联的 StockLocation 的库存。

### 3.4 何时触发可售量检查

1. **添加商品到订单时**：`addItemToOrder` mutation 检查 `saleableStockLevel >= quantity`
2. **订单进入 ArrangingPayment 时**：`defaultOrderProcess.onTransitionStart()` 中 `arrangingPaymentRequiresStock` 选项遍历所有行检查可售量

---

## 4. StockLevel 的更新机制

`StockLevelService` 提供两个原子更新方法，都是**读取-修改-写入**模式：

### 4.1 updateStockOnHandForLocation()

`packages/core/src/service/services/stock-level.service.ts:86`

```typescript
async updateStockOnHandForLocation(ctx, productVariantId, stockLocationId, change) {
    const stockLevel = await repo.findOne({ where: { productVariantId, stockLocationId } });
    if (!stockLevel) {
        // 自动创建新行
        await repo.save(new StockLevel({ productVariantId, stockLocationId, stockOnHand: change, stockAllocated: 0 }));
    }
    if (stockLevel) {
        await repo.update(stockLevel.id, { stockOnHand: stockLevel.stockOnHand + change });
    }
}
```

### 4.2 updateStockAllocatedForLocation()

`packages/core/src/service/services/stock-level.service.ts:119`

```typescript
async updateStockAllocatedForLocation(ctx, productVariantId, stockLocationId, change) {
    const stockLevel = await repo.findOne({ where: { productVariantId, stockLocationId } });
    if (stockLevel) {
        await repo.update(stockLevel.id, { stockAllocated: stockLevel.stockAllocated + change });
    }
}
```

**注意**：这里使用的是 TypeORM 的 `update()` 而非 `save()`，直接生成 `UPDATE ... SET stockAllocated = ? WHERE id = ?` SQL。在并发场景下，两个请求可能读到相同的旧值，导致其中一个更新被覆盖（lost update）。

---

## 5. 多仓库场景下的 StockLocationStrategy 协作

### 5.1 策略接口

`packages/core/src/config/catalog/stock-location-strategy.ts`

```typescript
interface StockLocationStrategy {
    getAvailableStock(ctx, productVariantId, stockLevels): AvailableStock;
    forAllocation(ctx, stockLocations, orderLine, quantity): LocationWithQuantity[];
    forRelease(ctx, stockLocations, orderLine, quantity): LocationWithQuantity[];
    forSale(ctx, stockLocations, orderLine, quantity): LocationWithQuantity[];
    forCancellation(ctx, stockLocations, orderLine, quantity): LocationWithQuantity[];
}
```

每种操作都需要策略决定"从哪些仓库、各用多少数量"。

### 5.2 BaseStockLocationStrategy 的通用逻辑

`packages/core/src/config/catalog/default-stock-location-strategy.ts:14`

**forRelease / forSale / forCancellation** 共用同一个实现 `getLocationsBasedOnAllocations()`：

1. 查询该 OrderLine 的所有 Allocation 记录
2. 按 stockLocationId 汇总每个仓库的 allocated 数量
3. 按 Allocation 的创建顺序，从最早分配的仓库开始"归还"，直到满足释放/销售/取消的数量

**关键设计**：Release/Sale/Cancellation **跟随 Allocation 的仓库分布**，而不是重新分配。这确保了"从哪个仓库分配的，就还给哪个仓库"。

### 5.3 MultiChannelStockLocationStrategy 的 forAllocation

`packages/core/src/config/catalog/multi-channel-stock-location-strategy.ts:97`

```typescript
async forAllocation(ctx, stockLocations, orderLine, quantity) {
    const stockLevels = await this.getStockLevelsForVariant(ctx, orderLine.productVariantId);
    const variant = await this.connection.getEntityOrThrow(ctx, ProductVariant, orderLine.productVariantId);
    let totalAllocated = 0;
    const locations = [];

    for (const stockLocation of stockLocations) {
        const stockLevel = stockLevels.find(sl => sl.stockLocationId === stockLocation.id);
        if (stockLevel && await this.stockLevelAppliesToActiveChannel(ctx, stockLevel)) {
            const quantityAvailable = inventoryNotTracked
                ? Number.MAX_SAFE_INTEGER
                : stockLevel.stockOnHand - stockLevel.stockAllocated - effectiveOutOfStockThreshold;
            if (quantityAvailable > 0) {
                const quantityToAllocate = Math.min(quantity, quantityAvailable);
                locations.push({ location: stockLocation, quantity: quantityToAllocate });
                totalAllocated += quantityToAllocate;
            }
        }
        if (totalAllocated >= quantity) break;
    }
    return locations;
}
```

**逻辑**：
1. 只考虑属于当前 Channel 的 StockLocation
2. 对每个仓库，计算 `quantityAvailable = stockOnHand - stockAllocated - outOfStockThreshold`
3. 如果不追踪库存，可分配量视为无穷大
4. 按仓库顺序依次分配，直到满足所需数量
5. **支持部分分配**：如果所有仓库的可用量之和 < 所需数量，返回的部分分配结果仍然会被保存

---

## 6. 完整生命周期流程图

```
[添加到购物车]
    │  检查 saleable = stockOnHand - stockAllocated - threshold
    ▼
[ArrangingPayment]
    │  再次检查可售量 (arrangingPaymentRequiresStock)
    ▼
[PaymentAuthorized / PaymentSettled]
    │  ★ Allocation 创建
    │  StockLevel.stockAllocated += qty
    ▼
[创建 Fulfillment (Created → Pending)]
    │  ★ Sale 创建
    │  StockLevel.stockAllocated -= qty   (释放分配)
    │  StockLevel.stockOnHand -= qty      (实际出库)
    ▼
[Shipped → Delivered]
    （无库存操作）
```

### 取消路径

```
路径 A: 已分配但未发货 → 取消
    ★ Release 创建
    StockLevel.stockAllocated -= qty
    可售量回升

路径 B: 已发货 → 取消 (Fulfillment Cancelled)
    ★ Cancellation 创建 → stockOnHand += qty
    ★ Allocation 创建   → stockAllocated += qty
    效果：stockOnHand 归还，但库存仍被新 Allocation 占用

路径 C: 已发货 → 订单取消 (orderModifier)
    先按 OrderLine 计算 totalFulfilled 和 totalAllocated
    ★ fulfilled 部分 → Cancellation → stockOnHand += qty
    ★ allocated 部分 → Release → stockAllocated -= qty
```

### 手动调整路径

```
[管理员修改 stockOnHand]
    ★ StockAdjustment 创建
    StockLevel.stockOnHand += delta
    （stockAllocated 不受影响）
```

---

## 7. 各 StockMovement 类型对 StockLevel 的影响汇总

| StockMovement 类型 | stockOnHand | stockAllocated | saleable 变化 |
|-------------------|-------------|----------------|--------------|
| StockAdjustment(+N) | +N | 不变 | +N |
| StockAdjustment(-N) | -N | 不变 | -N |
| Allocation(+N) | 不变 | +N | -N |
| Sale(-N) | -N | -N | 不变（两个量同减） |
| Release(+N) | 不变 | -N | +N |
| Cancellation(+N) | +N | 不变 | +N |

---

## 8. 并发安全分析

### 8.1 现有实现的并发模型

`StockLevelService` 的更新使用"先读后写"模式：

```typescript
const stockLevel = await repo.findOne(...);
await repo.update(stockLevel.id, { stockOnHand: stockLevel.stockOnHand + change });
```

TypeORM 的 `update()` 生成 `UPDATE stock_level SET stockOnHand = ? WHERE id = ?`，是**非原子性的 read-modify-write**。在两个并发请求同时分配同一 variant 的库存时，可能出现：

1. 请求 A 读取 `stockAllocated = 5`
2. 请求 B 读取 `stockAllocated = 5`
3. 请求 A 写入 `stockAllocated = 7`（+2）
4. 请求 B 写入 `stockAllocated = 7`（+2，应该是 9）

**这是 Vendure 当前已知的并发风险**。不过在实际运行中，因为：

- Vendure 使用数据库事务（`@Transaction()` 装饰器）
- 订单状态机的转换是串行的
- 同一订单的 Allocation 不会并发执行

所以对于**同一订单**的操作不存在并发问题。但**不同订单同时分配同一 SKU** 的场景下确实存在竞态条件。

### 8.2 可售量检查与分配之间的时间窗口

```
时刻 T1: 订单 A 检查可售量 = 5 ✓
时刻 T2: 订单 B 检查可售量 = 5 ✓
时刻 T3: 订单 A 分配 5 → stockAllocated = 5, 可售量 = 0
时刻 T4: 订单 B 分配 5 → stockAllocated = 10, 可售量 = -5  ← 超卖
```

这是典型的 TOCTOU（Time-of-Check-to-Time-of-Use）问题。Vendure 通过 `outOfStockThreshold` 允许设为负值来"容忍"超卖（back order），而非通过数据库锁来防止。

### 8.3 对并发安全有实际保障的机制

1. **TypeORM 的 `@Transaction()`**：同一事务内的操作具有数据库隔离级别保证
2. **OrderStateMachine**：同一订单的状态转换是串行的，不会并发
3. **Allocation 的 forAllocation 查询 StockLevel 快照**：在 `MultiChannelStockLocationStrategy` 中，分配前查询最新的 `stockLevels`，计算 `quantityAvailable`
4. **`arrangingPaymentRequiresStock` 检查**：在进入支付前做最后防线检查

---

## 9. 关键源码文件索引

| 文件 | 职责 |
|------|------|
| `core/src/entity/stock-movement/stock-movement.entity.ts` | StockMovement 抽象基类 |
| `core/src/entity/stock-movement/allocation.entity.ts` | Allocation 子类 |
| `core/src/entity/stock-movement/release.entity.ts` | Release 子类 |
| `core/src/entity/stock-movement/sale.entity.ts` | Sale 子类 |
| `core/src/entity/stock-movement/cancellation.entity.ts` | Cancellation 子类 |
| `core/src/entity/stock-movement/stock-adjustment.entity.ts` | StockAdjustment 子类 |
| `core/src/entity/stock-level/stock-level.entity.ts` | StockLevel 实体（variant × location 唯一行） |
| `core/src/entity/stock-location/stock-location.entity.ts` | StockLocation 实体 |
| `core/src/service/services/stock-movement.service.ts` | 所有 StockMovement 的创建逻辑 |
| `core/src/service/services/stock-level.service.ts` | StockLevel 的读写和聚合 |
| `core/src/service/services/stock-location.service.ts` | 获取分配/释放/销售/取消的位置策略委托 |
| `core/src/service/services/product-variant.service.ts:321` | `getSaleableStockLevel()` 可售量计算 |
| `core/src/config/order/default-order-process.ts:427-434` | 订单状态转换时触发 Allocation |
| `core/src/config/fulfillment/default-fulfillment-process.ts:93-104` | Fulfillment 状态转换时触发 Sale/Cancellation |
| `core/src/config/order/default-stock-allocation-strategy.ts` | 默认分配时机策略 |
| `core/src/config/catalog/stock-location-strategy.ts` | 仓库位置策略接口 |
| `core/src/config/catalog/default-stock-location-strategy.ts` | 默认/基础仓库位置策略 |
| `core/src/config/catalog/multi-channel-stock-location-strategy.ts` | 多渠道仓库位置策略（v3.1 默认） |
| `core/src/service/helpers/order-modifier/order-modifier.ts:270-331` | 订单取消中的 Cancellation/Release 分流 |
