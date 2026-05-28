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

### 5.3 释放路径的查询顺序约束与时序稳定性分析

`BaseStockLocationStrategy.getLocationsBasedOnAllocations()` 是 Release、Sale、Cancellation 三个操作的共用核心逻辑，其实现对仓库选择顺序有重大影响。

**源码位置**：`packages/core/src/config/catalog/default-stock-location-strategy.ts:61-92`

```typescript
private async getLocationsBasedOnAllocations(
    ctx: RequestContext,
    stockLocations: StockLocation[],
    orderLine: OrderLine,
    quantity: number,
) {
    const allocations = await this.connection.getRepository(ctx, Allocation).find({
        where: {
            orderLine: { id: orderLine.id },
        },
    });
    let unallocated = quantity;
    const quantityByLocationId = new Map<ID, number>();
    for (const allocation of allocations) {
        if (unallocated <= 0) {
            break;
        }
        const qtyAtLocation = quantityByLocationId.get(allocation.stockLocationId);
        const qtyToAdd = Math.min(allocation.quantity, unallocated);
        if (qtyAtLocation != null) {
            quantityByLocationId.set(allocation.stockLocationId, qtyAtLocation + qtyToAdd);
        } else {
            quantityByLocationId.set(allocation.stockLocationId, qtyToAdd);
        }
        unallocated -= qtyToAdd;
    }
    return [...quantityByLocationId.entries()].map(([locationId, qty]) => ({
        location: stockLocations.find(l => idsAreEqual(l.id, locationId))!,
        quantity: qty,
    }));
}
```

#### 5.3.1 查询顺序约束的缺失

**关键问题**：代码中的 `.find()` 查询**没有指定 `order` 排序条件**。

```typescript
// ❌ 没有排序约束
const allocations = await this.connection.getRepository(ctx, Allocation).find({
    where: { orderLine: { id: orderLine.id } },
});
```

这意味着：
1. TypeORM 不会在 SQL 中添加 `ORDER BY` 子句
2. 查询结果的顺序完全由数据库的**默认排序行为**决定
3. SQL 标准规定：没有 `ORDER BY` 的查询结果顺序是**未定义**的

#### 5.3.2 不同数据库下可能出现的顺序差异

不同数据库在无 `ORDER BY` 时的默认排序行为不一致：

| 数据库 | 默认排序行为 | 顺序稳定性 |
|--------|-------------|-----------|
| **MySQL / MariaDB** | 通常按 `PRIMARY KEY`（id）升序返回 | ✅ 相对稳定（只要主键是自增） |
| **PostgreSQL** | 按物理存储顺序（`ctid`）返回 | ❌ 不稳定（VACUUM、UPDATE 后可能改变） |
| **SQLite** | 按 `rowid` 顺序返回 | ✅ 相对稳定（只要没有 DELETE） |
| **SQL Server** | 按聚集索引顺序返回 | ✅ 相对稳定 |

**实际风险场景**：
- **MySQL 自动增量主键**：通常按 id 升序 → 按 Allocation 创建时间先后 → FIFO（先分配的先释放）
- **UUID 主键策略**：完全无序 → 释放顺序不确定
- **PostgreSQL 表经历大量更新后**：物理存储碎片化 → 查询顺序随机化

#### 5.3.3 对可用量判断的影响

释放顺序的不确定性会直接影响**各仓库的库存分布**，进而影响后续的分配决策。

**示例场景**：

假设订单行在两个仓库有分配：
- 仓库 A（id=1）：Allocation 数量 5
- 仓库 B（id=2）：Allocation 数量 5
- 现在要部分释放 5 件

**场景 1：MySQL + 自增主键，Allocation A 先创建**
```
遍历顺序：A → B
释放结果：A 释放 5，B 释放 0
各仓库释放后 allocated：A=0, B=5
```

**场景 2：PostgreSQL + 碎片化存储，随机先返回 B**
```
遍历顺序：B → A
释放结果：B 释放 5，A 释放 0
各仓库释放后 allocated：A=5, B=0
```

**对后续可用量的连锁影响**：

假设仓库 A 的 `stockOnHand = 10`，仓库 B 的 `stockOnHand = 0`（B 已补货）：

| 释放结果 | A 可售量 | B 可售量 | 后续订单能否从 B 分配 |
|---------|---------|---------|-------------------|
| 场景 1（A 全释放） | `10 - 0 = 10` | `0 - 5 = -5` | ❌ B 仍被占用 5 件 |
| 场景 2（B 全释放） | `10 - 5 = 5` | `0 - 0 = 0` | ✅ B 释放了，可重新分配 |

**关键影响**：
1. **仓库间库存流转不均**：随机顺序导致某些仓库的 allocated 永远先被释放，另一些持续"积压"
2. **跨仓调拨判断失真**：如果基于 allocated 做调拨决策，顺序不稳定会导致决策反复
3. **测试不可重现**：同一测试用例在不同数据库环境下可能得到不同结果

#### 5.3.4 与 OrderModifier 中汇总逻辑的对比

有趣的是，`OrderModifier.cancelOrderByOrderLines()` 中计算 `totalAllocated` 时也有类似的无排序查询：

**源码位置**：`packages/core/src/service/helpers/order-modifier/order-modifier.ts:281-302`

```typescript
const allocationsForLine = await this.connection
    .getRepository(ctx, Allocation)
    .createQueryBuilder('allocation')
    .leftJoinAndSelect('allocation.orderLine', 'orderLine')
    .where('orderLine.id = :orderLineId', { orderLineId: lineInput.orderLineId })
    .getMany();  // 同样没有 .orderBy()
```

但这里**顺序不影响结果**，因为最后是用 `summate()` 做纯汇总：

```typescript
const totalAllocated =
    summate(allocationsForLine, 'quantity') +
    summate(salesForLine, 'quantity') -
    summate(releasesForLine, 'quantity');
```

**对比结论**：
- `OrderModifier` 的汇总逻辑：无排序 → ✅ 不影响正确性
- `getLocationsBasedOnAllocations` 的分配逻辑：无排序 → ❌ 影响仓库选择，可能产生业务偏差

#### 5.3.5 修复方向（当前代码未实现）

如果要保证释放顺序的确定性，可以：

**方案 A：按创建时间排序（FIFO）**
```typescript
const allocations = await this.connection.getRepository(ctx, Allocation).find({
    where: { orderLine: { id: orderLine.id } },
    order: { createdAt: 'ASC' },  // 先创建的先释放
});
```

**方案 B：按仓库优先级排序（LILO，Last-In-Last-Out）**
```typescript
const allocations = await this.connection.getRepository(ctx, Allocation).find({
    where: { orderLine: { id: orderLine.id } },
    order: { createdAt: 'DESC' },  // 后创建的先释放
});
```

**方案 C：按仓库优先级+创建时间混合排序**
- 结合 `StockLocationStrategy` 的优先级逻辑
- 释放时优先从优先级低的仓库释放（保持高优先级仓库的占用状态）

### 5.4 MultiChannelStockLocationStrategy 的 forAllocation

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

## 7. 多仓分配中的数量累加与库存更新口径对齐分析

本节深入分析 `forAllocation` 的累计分配逻辑，以及 Sale/Release/Cancellation 记录数量与各仓库存扣减之间的关系——这是多仓库场景下最容易产生理解偏差的部分。

### 7.1 forAllocation 的累计分配数量计算逻辑

`MultiChannelStockLocationStrategy.forAllocation()` 负责将订单行的分配需求拆解到多个仓库。

**源码位置**：`packages/core/src/config/catalog/multi-channel-stock-location-strategy.ts:97-136`

```typescript
async forAllocation(ctx, stockLocations, orderLine, quantity) {
    const stockLevels = await this.getStockLevelsForVariant(ctx, orderLine.productVariantId);
    let totalAllocated = 0;
    const locations: LocationWithQuantity[] = [];
    
    for (const stockLocation of stockLocations) {
        const stockLevel = stockLevels.find(sl => sl.stockLocationId === stockLocation.id);
        if (stockLevel && await this.stockLevelAppliesToActiveChannel(ctx, stockLevel)) {
            const quantityAvailable = inventoryNotTracked
                ? Number.MAX_SAFE_INTEGER
                : stockLevel.stockOnHand - stockLevel.stockAllocated - effectiveOutOfStockThreshold;
            
            if (quantityAvailable > 0) {
                const quantityToAllocate = Math.min(quantity, quantityAvailable);
                locations.push({
                    location: stockLocation,
                    quantity: quantityToAllocate,
                });
                totalAllocated += quantityToAllocate;
            }
        }
        if (totalAllocated >= quantity) {
            break;
        }
    }
    return locations;
}
```

#### 7.1.1 计算逻辑的关键特性

**逐仓累加算法（有 bug）**：
```
输入：需求 quantity = 10
对每个仓库（按配置顺序）：
  计算该仓可售量 = stockOnHand - stockAllocated - threshold
  如果可售量 > 0：
    分配量 = min(需求总量, 可售量)  ← ❌ 应该是 min(剩余需求, 可售量)
    累计 totalAllocated += 分配量
  如果 totalAllocated >= 需求总量：退出循环
```

**反例（需求 10 件）**：
- 仓库 A：可售 7 件 → 分配 `min(10, 7) = 7`，累计 7
- 仓库 B：可售 8 件 → 分配 `min(10, 8) = 8`，累计 15
- `totalAllocated(15) >= 需求(10)` → 退出
- **实际返回**：A=7, B=8（共 15 件，超额分配 5 件！）

**这是真实的 bug**：
- `Math.min(quantity, quantityAvailable)` 这里用的是**原始需求**而非剩余需求
- `totalAllocated >= quantity` 的 break 条件无法阻止超额分配
- 因为第二个仓库的 `min(10, 8) = 8` 直接把总量推到了 15
- `createAllocationsForOrderLines` 中会按返回的 `quantityToAllocate` **直接更新库存**，没有任何裁剪逻辑

**对比：getLocationsBasedOnAllocations 是正确的实现**：
```typescript
let unallocated = quantity;  // 跟踪剩余需求
for (const allocation of allocations) {
    const qtyToAdd = Math.min(allocation.quantity, unallocated);  // ✅ 用剩余需求
    unallocated -= qtyToAdd;  // 递减剩余需求
}
```

#### 7.1.2 对库存一致性的影响

这个算法有两个重要特性：

1. **确实会超额分配**：当需要跨多个仓库分配时，最终分配总量 `sum(quantityToAllocate)` **可能显著超过** `quantity`
2. **按仓库顺序优先分配**：排在前面的仓库优先被分配（类似于货架从左到右取货）

**超额分配的实际影响**：
- StockLevel.stockAllocated 会被多扣，导致可售量被不必要地压低
- 后续的 Sale 操作跟随 Allocation 记录，也会多扣 stockOnHand
- 这是一个"累积性"bug——分配时超额多少，后续操作都会延续这个误差

**对可售量判断的影响**：
- 分配前计算的 `quantityAvailable` 基于查询时的 StockLevel 快照
- 但分配时不检查"其他仓库已经分配了多少"，只检查累计总量
- 这意味着单个仓库的分配决策不考虑其他仓库的分配结果

---

### 7.2 StockMovement 记录数量 vs 库存更新数量：口径不一致问题

这是多仓库场景下的**核心隐蔽问题**：`Sale`/`Release`/`Cancellation` 实体记录的 `quantity` 字段，与实际更新到 `StockLevel` 的数量，使用了不同的口径。

#### 7.2.1 Allocation：口径一致（正确）

**源码位置**：`packages/core/src/service/services/stock-movement.service.ts:172-188`

```typescript
for (const allocationLocation of allocationLocations) {
    const allocation = new Allocation({
        productVariant: new ProductVariant({ id: orderLine.productVariantId }),
        stockLocation: allocationLocation.location,
        quantity: allocationLocation.quantity,  // ✅ 用仓库分配的数量
        orderLine,
    });
    allocations.push(allocation);

    if (this.trackInventoryForVariant(productVariant, globalTrackInventory)) {
        await this.stockLevelService.updateStockAllocatedForLocation(
            ctx,
            orderLine.productVariantId,
            allocationLocation.location.id,
            allocationLocation.quantity,  // ✅ 同样用仓库分配的数量
        );
    }
}
```

**口径一致**：Allocation 实体的 `quantity` = StockLevel 更新的 `change`，都是 `allocationLocation.quantity`。

---

#### 7.2.2 Sale：口径不一致（多仓场景有问题）

**源码位置**：`packages/core/src/service/services/stock-movement.service.ts:227-249`

```typescript
for (const saleLocation of saleLocations) {
    const sale = new Sale({
        productVariant,
        quantity: lineRow.quantity * -1,  // ❌ 用的是订单行总数量！
        orderLine,
        stockLocation: saleLocation.location,
    });
    sales.push(sale);

    if (this.trackInventoryForVariant(productVariant, globalTrackInventory)) {
        await this.stockLevelService.updateStockAllocatedForLocation(
            ctx,
            orderLine.productVariantId,
            saleLocation.location.id,
            -saleLocation.quantity,  // ✅ 用的是当前仓库分配的数量
        );
        await this.stockLevelService.updateStockOnHandForLocation(
            ctx,
            orderLine.productVariantId,
            saleLocation.location.id,
            -saleLocation.quantity,  // ✅ 用的是当前仓库分配的数量
        );
    }
}
```

**问题点**：
| 位置 | 使用的数量 | 含义 |
|------|-----------|------|
| `Sale.quantity` | `lineRow.quantity * -1` | 订单行**总数量**的负数 |
| `updateStockAllocatedForLocation` | `-saleLocation.quantity` | 当前仓库的销售数量 |
| `updateStockOnHandForLocation` | `-saleLocation.quantity` | 当前仓库的销售数量 |

---

#### 7.2.3 Release：口径不一致

**源码位置**：`packages/core/src/service/services/stock-movement.service.ts:338-353`

```typescript
for (const releaseLocation of releaseLocations) {
    const release = new Release({
        productVariant: orderLine.productVariant,
        quantity: lineInput.quantity,  // ❌ 用的是订单行总数量！
        orderLine,
        stockLocation: releaseLocation.location,
    });
    releases.push(release);
    
    if (this.trackInventoryForVariant(orderLine.productVariant, globalTrackInventory)) {
        await this.stockLevelService.updateStockAllocatedForLocation(
            ctx,
            orderLine.productVariantId,
            releaseLocation.location.id,
            -releaseLocation.quantity,  // ✅ 用的是当前仓库释放的数量
        );
    }
}
```

---

#### 7.2.4 Cancellation：口径不一致

**源码位置**：`packages/core/src/service/services/stock-movement.service.ts:288-304`

```typescript
for (const cancellationLocation of cancellationLocations) {
    const cancellation = new Cancellation({
        productVariant: orderLine.productVariant,
        quantity: lineInput.quantity,  // ❌ 用的是订单行总数量！
        orderLine,
        stockLocation: cancellationLocation.location,
    });
    cancellations.push(cancellation);

    if (this.trackInventoryForVariant(orderLine.productVariant, globalTrackInventory)) {
        await this.stockLevelService.updateStockOnHandForLocation(
            ctx,
            orderLine.productVariantId,
            cancellationLocation.location.id,
            cancellationLocation.quantity,  // ✅ 用的是当前仓库取消的数量
        );
    }
}
```

---

### 7.3 口径不一致的实际影响

#### 7.3.1 单仓库场景：碰巧正确

如果订单行的分配/销售/释放/取消都只涉及一个仓库：
- `allocationLocations.length = 1`
- `saleLocation.quantity = lineRow.quantity`
- 此时 `Sale.quantity = lineRow.quantity * -1` 恰好等于实际扣减数量
- **结果：StockMovement 记录与库存更新一致**

这就是为什么单仓库场景下没有暴露这个问题。

---

#### 7.3.2 多仓库场景：审计日志失真

**示例场景**：
- 订单行数量：10 件
- 仓库分配：A 仓 6 件，B 仓 4 件（通过 `getLocationsBasedOnAllocations` 跟随原始分配）

**Sale 创建过程**：
```
循环处理每个 saleLocation：

  仓库 A：
    Sale.quantity = 10 * -1 = -10  ← 记录的是总数量
    updateStockAllocatedForLocation(A, -6)  ← 实际扣减 6
    updateStockOnHandForLocation(A, -6)      ← 实际扣减 6

  仓库 B：
    Sale.quantity = 10 * -1 = -10  ← 记录的是总数量
    updateStockAllocatedForLocation(B, -4)  ← 实际扣减 4
    updateStockOnHandForLocation(B, -4)      ← 实际扣减 4
```

**结果对比**：
| 维度 | 期望值 | 实际值 | 偏差 |
|------|-------|-------|------|
| StockLevel.stockAllocated 变化 | -10 | -10 | ✅ 正确 |
| StockLevel.stockOnHand 变化 | -10 | -10 | ✅ 正确 |
| Sale 记录 quantity 总和 | -10 | -20 | ❌ 翻倍！ |

---

#### 7.3.3 对库存一致性判断的连锁影响

1. **审计日志不可信**：
   - 不能通过 `sum(Sale.quantity)` 反推总销量
   - 不能通过 `sum(Allocation.quantity) - sum(Release.quantity)` 反推当前 allocated
   - 不能通过 `sum(StockAdjustment.quantity) - sum(Sale.quantity) + sum(Cancellation.quantity)` 反推 stockOnHand

2. **报表和分析失真**：
   - 如果按 StockMovement 表做销售报表，多仓订单会被重复计算
   - 按仓库分组的销量统计完全错误（每个仓库都记录了总销量）

3. **业务对账困难**：
   - 财务系统按订单行对账：10 件销量 ✓
   - 库存系统按 StockLevel 扣减：10 件 ✓
   - 审计系统按 StockMovement 汇总：20 件 ✗
   - 三方对账不一致

4. **测试用例的局限性**：
   - 现有 E2E 测试大多使用单仓库场景（`DefaultStockLocationStrategy`）
   - 多仓库 E2E 测试（`stock-control-multi-location.e2e-spec.ts`）只验证了 StockLevel 的最终状态，没有断言 StockMovement 记录的数量

---

### 7.4 问题根源与修复方向

#### 7.4.1 问题根源

代码注释暗示了最初的设计假设：

```typescript
// Sale 实体的 quantity 字段在单仓库场景下是正确的
// 但多仓库引入后，没有同步更新这三处的 quantity 赋值
```

本质是**多仓重构时的遗漏**：
- `Allocation` 在多仓重构时被正确修改了（使用 `allocationLocation.quantity`）
- `Sale`/`Release`/`Cancellation` 三处遗漏了类似的修改

#### 7.4.2 修复方案

统一使用 `*Location.quantity` 作为 StockMovement 记录的数量：

**Sale 的修复**：
```typescript
const sale = new Sale({
    productVariant,
    quantity: saleLocation.quantity * -1,  // ✅ 改为 saleLocation.quantity
    orderLine,
    stockLocation: saleLocation.location,
});
```

**Release 的修复**：
```typescript
const release = new Release({
    productVariant: orderLine.productVariant,
    quantity: releaseLocation.quantity,  // ✅ 改为 releaseLocation.quantity
    orderLine,
    stockLocation: releaseLocation.location,
});
```

**Cancellation 的修复**：
```typescript
const cancellation = new Cancellation({
    productVariant: orderLine.productVariant,
    quantity: cancellationLocation.quantity,  // ✅ 改为 cancellationLocation.quantity
    orderLine,
    stockLocation: cancellationLocation.location,
});
```

#### 7.4.3 修复后的一致性保证

修复后，所有 StockMovement 类型都遵循同一规则：
- StockMovement 实体的 `quantity` 字段 = 该仓库实际发生的变动数量
- StockLevel 更新的 `change` 参数 = 该仓库实际发生的变动数量

这样就可以安全地通过 StockMovement 审计日志反推库存状态。

---

### 7.5 库存一致性判断：可直接复用的结论

综合以上分析，以下是可以直接复用的库存一致性判断结论：

#### 7.5.1 已知 bug 清单

| Bug 位置 | 影响范围 | 严重程度 | 触发条件 |
|---------|---------|---------|---------|
| **forAllocation 超额分配** | `MultiChannelStockLocationStrategy.forAllocation()` | ⚠️ 高 | 订单行数量需要从 ≥2 个仓库分配时 |
| **Sale 记录口径错误** | `createSalesForOrder()` | ⚠️ 高 | 订单行涉及 ≥2 个仓库时 |
| **Release 记录口径错误** | `createReleasesForOrderLines()` | ⚠️ 高 | 订单行涉及 ≥2 个仓库时 |
| **Cancellation 记录口径错误** | `createCancellationsForOrderLines()` | ⚠️ 高 | 订单行涉及 ≥2 个仓库时 |
| **释放查询无排序** | `getLocationsBasedOnAllocations()` | 🟡 中 | 订单行涉及 ≥2 个仓库且部分释放时 |

#### 7.5.2 一致性校验公式（当前代码下）

**✅ 总是成立（StockLevel 自洽）**：
```
StockLevel.stockAllocated = sum(Allocation.quantity for this location)
                          - sum(Release.quantity for this location)
                          - sum(Sale.quantity for this location)  // 因为 Sale.quantity 是负数

StockLevel.stockOnHand = 初始值
                       + sum(StockAdjustment.quantity for this location)
                       + sum(Sale.quantity for this location)    // 因为 Sale.quantity 是负数
                       + sum(Cancellation.quantity for this location)
```

注意：以上公式成立**不是因为 StockMovement 记录正确**，而是因为 StockLevel 更新直接用了 `*Location.quantity` 参数，与 StockMovement.quantity 字段无关。

**❌ 不成立（StockMovement 审计不可信）**：
```
// 多仓场景下以下公式不成立！
订单行总销量 ≠ sum(Sale.quantity for this orderLine)
订单行总释放 ≠ sum(Release.quantity for this orderLine)
订单行总取消 ≠ sum(Cancellation.quantity for this orderLine)
```

**⚠️ 谨慎使用（可能受超额分配影响）**：
```
// 如果 forAllocation 超额分配了，以下也会失真
订单行总分配 ≠ sum(Allocation.quantity for this orderLine)
```

#### 7.5.3 正确的一致性校验方式（绕过 bug）

要做正确的库存对账，**不要直接汇总 StockMovement.quantity**，而是：

**方式 A：按 StockLevel 为准（推荐）**：
```
对每个 (variant, location)：
  expected_stockAllocated = sum(Allocation.quantity where stockLocationId=X)
                          - sum(Release.quantity where stockLocationId=X)
                          - sum(Sale.quantity where stockLocationId=X)
  assert StockLevel.stockAllocated == expected_stockAllocated
```
但这仍然依赖 StockMovement.quantity，如果记录口径错误仍然不可信。

**方式 B：通过 OrderLine 反向计算（最可靠）**：
```
对每个 OrderLine：
  total_allocated = sum(Allocation.quantity for this orderLine)
  total_sold = sum(FulfillmentLine.quantity for this orderLine)
              - sum(Cancellation.quantity for this orderLine)
  total_released = ...  // 需要从 OrderModification 等表追溯
  
  // 注意：这里用 FulfillmentLine 而不是 Sale，因为 FulfillmentLine 记录的是真实数量
```

**方式 C：仅校验数量变更方向（保守）**：
```
对每个 StockMovement：
  assert Allocation.quantity > 0
  assert Release.quantity > 0
  assert Sale.quantity < 0
  assert Cancellation.quantity > 0
  assert StockAdjustment.quantity != 0
```
这样至少可以保证每条记录的正负号是正确的。

#### 7.5.4 与记录口径 bug 的关系分析

| 场景 | Allocation 记录 | Sale 记录 | Release 记录 | Cancellation 记录 | StockLevel 最终状态 |
|------|----------------|----------|-------------|------------------|--------------------|
| 单仓分配 10 件 | ✅ 10（正确） | ✅ -10（碰巧正确） | ✅ 10（碰巧正确） | ✅ 10（碰巧正确） | ✅ 正确 |
| 多仓分配 A=7, B=8（超额） | ⚠️ A=7, B=8（合计 15） | ⚠️ A=-10, B=-10（合计 -20） | ⚠️ A=10, B=10（合计 20） | ⚠️ A=10, B=10（合计 20） | ⚠️ 被超额分配污染 |
| 多仓分配 A=6, B=4（正常） | ✅ A=6, B=4（合计 10） | ⚠️ A=-10, B=-10（合计 -20） | ⚠️ A=10, B=10（合计 20） | ⚠️ A=10, B=10（合计 20） | ✅ 碰巧正确 |

**关键结论**：
1. **StockLevel 的正确性取决于 `*Location.quantity` 参数**，与 StockMovement.quantity 字段无关
2. **StockMovement 记录只影响审计和报表**，不影响实际库存扣减
3. **forAllocation 的超额分配 bug 会污染 StockLevel**——这是唯一能"击穿"到实际库存的 bug
4. **记录口径 bug 是"表面"bug**——只影响日志，不影响实际库存（但影响对账）

#### 7.5.5 优先级建议

| 修复优先级 | 问题 | 理由 |
|-----------|------|------|
| 🔴 最高 | forAllocation 超额分配 | 直接影响 StockLevel 正确性，导致实际库存扣减错误 |
| 🟡 中等 | Sale/Release/Cancellation 记录口径 | 影响审计和报表，但不影响实际库存 |
| 🟢 较低 | 释放查询无排序 | 仅影响仓库间分布，总量正确 |

---

## 8. 各 StockMovement 类型对 StockLevel 的影响汇总

| StockMovement 类型 | stockOnHand | stockAllocated | saleable 变化 |
|-------------------|-------------|----------------|--------------|
| StockAdjustment(+N) | +N | 不变 | +N |
| StockAdjustment(-N) | -N | 不变 | -N |
| Allocation(+N) | 不变 | +N | -N |
| Sale(-N) | -N | -N | 不变（两个量同减） |
| Release(+N) | 不变 | -N | +N |
| Cancellation(+N) | +N | 不变 | +N |

---

## 9. 并发安全分析

### 9.1 现有实现的并发模型

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

### 9.2 可售量检查与分配之间的时间窗口

```
时刻 T1: 订单 A 检查可售量 = 5 ✓
时刻 T2: 订单 B 检查可售量 = 5 ✓
时刻 T3: 订单 A 分配 5 → stockAllocated = 5, 可售量 = 0
时刻 T4: 订单 B 分配 5 → stockAllocated = 10, 可售量 = -5  ← 超卖
```

这是典型的 TOCTOU（Time-of-Check-to-Time-of-Use）问题。Vendure 通过 `outOfStockThreshold` 允许设为负值来"容忍"超卖（back order），而非通过数据库锁来防止。

### 9.3 对并发安全有实际保障的机制

1. **TypeORM 的 `@Transaction()`**：同一事务内的操作具有数据库隔离级别保证
2. **OrderStateMachine**：同一订单的状态转换是串行的，不会并发
3. **Allocation 的 forAllocation 查询 StockLevel 快照**：在 `MultiChannelStockLocationStrategy` 中，分配前查询最新的 `stockLevels`，计算 `quantityAvailable`
4. **`arrangingPaymentRequiresStock` 检查**：在进入支付前做最后防线检查

---

## 10. 关键源码文件索引

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
