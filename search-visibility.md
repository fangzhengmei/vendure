# 商品检索索引构建与可见性规则协作模式

## 1. 整体架构

### 1.1 核心组件

| 组件 | 职责 | 关键文件 |
|------|------|----------|
| `DefaultSearchPlugin` | 插件入口，事件订阅，初始化配置 | `default-search-plugin.ts` |
| `SearchIndexService` | 索引更新任务队列管理 | `search-index.service.ts` |
| `IndexerController` | 实际执行索引的增删改操作 | `indexer.controller.ts` |
| `SearchIndexItem` | 索引数据存储实体 | `search-index-item.entity.ts` |
| `FulltextSearchService` | 搜索服务入口，调度具体策略 | `fulltext-search.service.ts` |
| `SearchStrategy` | 数据库特定的搜索实现 | `postgres-search-strategy.ts` 等 |

### 1.2 数据流向

```
实体变更 → 事件总线 → DefaultSearchPlugin → SearchIndexService → JobQueue
                                                         ↓
                                                IndexerController
                                                         ↓
                                                  SearchIndexItem
                                                         ↓
                                              搜索查询 → SearchStrategy
```

---

## 2. 索引项数据结构

### 2.1 SearchIndexItem 主键设计

`SearchIndexItem` 采用复合主键，确保多维度数据隔离：

```typescript
@Entity()
export class SearchIndexItem {
    @EntityId({ primary: true })
    productVariantId: ID;       // 变体ID

    @PrimaryColumn('varchar')
    languageCode: LanguageCode; // 语言编码

    @EntityId({ primary: true })
    channelId: ID;              // 频道ID

    // 可选主键（启用 indexCurrencyCode 时）
    @PrimaryColumn('varchar')
    currencyCode?: CurrencyCode;
}
```

**设计意图**：每个变体在每个频道、每种语言（可选每种货币）下都有独立的索引记录。

### 2.2 可见性相关字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `enabled` | `boolean` | 商品/变体是否启用 |
| `channelIds` | `string[]` | 该变体所属的所有频道ID |
| `inStock` | `boolean` | 变体是否有库存（可选索引） |
| `productInStock` | `boolean` | 商品下是否有任何变体有库存（可选索引） |

---

## 3. 频道可见性的协作模式

### 3.1 索引构建阶段

**查询过滤**（`indexer.controller.ts:342-381`）：

```typescript
private async getSearchIndexQueryBuilder(ctx: RequestContext, options?: {
    channels?: Channel[];
}) {
    const where: FindOptionsWhere<ProductVariant> = {
        deletedAt: IsNull(),
        product: { deletedAt: IsNull() },
    };
    // 只查询分配到指定频道的变体
    where.channels = { id: In(channels.map(c => c.id)) };
}
```

**多频道索引**（`indexer.controller.ts:401-515`）：

```typescript
private async saveVariants(ctx: MutableRequestContext, variants: ProductVariant[]) {
    for (const variant of variants) {
        for (const languageCode of availableLanguageCodes) {
            // 为变体所属的每个频道创建索引项
            for (const channel of variant.channels) {
                const item = new SearchIndexItem({
                    channelId: ctx.channelId,
                    channelIds: channelIds.map(x => x.toString()),
                    // ...
                });
                items.push(item);
            }
        }
    }
}
```

### 3.2 事件驱动更新

`DefaultSearchPlugin` 在启动时订阅频道变更事件（`default-search-plugin.ts:142-171`）：

```typescript
// 商品-频道关联变更
this.eventBus.ofType(ProductChannelEvent).subscribe(event => {
    if (event.type === 'assigned') {
        return this.searchIndexService.assignProductToChannel(
            event.ctx, event.product.id, event.channelId
        );
    } else {
        return this.searchIndexService.removeProductFromChannel(
            event.ctx, event.product.id, event.channelId
        );
    }
});

// 变体-频道关联变更
this.eventBus.ofType(ProductVariantChannelEvent).subscribe(event => {
    if (event.type === 'assigned') {
        return this.searchIndexService.assignVariantToChannel(
            event.ctx, event.productVariant.id, event.channelId
        );
    } else {
        return this.searchIndexService.removeVariantFromChannel(
            event.ctx, event.productVariant.id, event.channelId
        );
    }
});
```

### 3.3 搜索查询过滤

所有搜索策略在查询时强制应用频道过滤（`postgres-search-strategy.ts:301`）：

```typescript
private applyTermAndFilters(ctx: RequestContext, qb: SelectQueryBuilder<SearchIndexItem>, input: SearchInput) {
    // 强制过滤当前频道
    qb.andWhere('si.channelId = :channelId', { channelId: ctx.channelId });
    // ...
}
```

---

## 4. 客户分组可见性的协作模式

### 4.1 核心发现

**索引层不直接支持客户分组可见性**。`SearchIndexItem` 实体中没有 `customerGroupId` 字段，客户分组通过价格策略间接影响。

### 4.2 价格选择策略

默认价格选择策略只考虑频道和货币（`default-product-variant-price-selection-strategy.ts:17-22`）：

```typescript
export class DefaultProductVariantPriceSelectionStrategy implements ProductVariantPriceSelectionStrategy {
    selectPrice(ctx: RequestContext, prices: ProductVariantPrice[]) {
        const pricesInChannel = prices.filter(p => idsAreEqual(p.channelId, ctx.channelId));
        const priceInCurrency = pricesInChannel.find(p => p.currencyCode === ctx.currencyCode);
        return priceInCurrency;
    }
}
```

### 4.3 索引构建时的价格计算

在 `saveVariants` 中应用价格（`indexer.controller.ts:446`）：

```typescript
await this.productPriceApplicator.applyChannelPriceAndTax(variant, ctx);
const item = new SearchIndexItem({
    price: variant.price,
    priceWithTax: variant.priceWithTax,
    // ...
});
```

### 4.4 扩展点与限制

**可扩展**：通过自定义 `ProductVariantPriceSelectionStrategy` 可以根据客户分组选择不同价格。

**限制**：
- 索引中只会存储一个价格（基于索引构建时的上下文）
- 无法为每个客户分组存储不同价格
- 客户分组级别的可见性需要在应用层（查询后）处理，而不是在索引层
- `ProductVariantPrice` 实体本身也没有 `customerGroupId` 字段

---

## 5. 库存状态可见性的协作模式

### 5.1 可选配置启用

库存状态索引是可选的，通过插件配置启用（`default-search-plugin.ts:107-116`）：

```typescript
static init(options: DefaultSearchPluginInitOptions): Type<DefaultSearchPlugin> {
    this.options = options;
    if (options.indexStockStatus === true) {
        this.addStockColumnsToEntity();  // 动态添加 inStock 和 productInStock 列
    }
    // ...
}
```

### 5.2 索引构建阶段

**变体库存**（`indexer.controller.ts:478-480`）：

```typescript
if (this.options.indexStockStatus) {
    item.inStock = 0 < (await this.productVariantService.getSaleableStockLevel(ctx, variant));
}
```

**商品库存**（`indexer.controller.ts:481-503`）：

```typescript
const productInStock = await this.requestContextCache.get(
    ctx,
    `productVariantsStock-${variant.productId}-${ctx.channelId}`,
    () => this.connection.getRepository(ctx, ProductVariant)
        .find({ where: { productId: variant.productId, deletedAt: IsNull() } })
        .then(_variants => Promise.all(
            _variants.map(v => this.productVariantService.getSaleableStockLevel(ctx, v))
        ))
        .then(stockLevels => stockLevels.some(stockLevel => 0 < stockLevel)),
);
item.productInStock = productInStock;
```

### 5.3 事件驱动更新

库存变动时触发重新索引（`default-search-plugin.ts:173-178`）：

```typescript
this.eventBus.ofType(StockMovementEvent).subscribe(event => {
    return this.searchIndexService.updateVariants(
        event.ctx,
        event.stockMovements.map(m => m.productVariant),
    );
});
```

### 5.4 搜索查询过滤

用户可通过 `inStock` 参数过滤（`postgres-search-strategy.ts:208-214`）：

```typescript
if (input.inStock != null) {
    if (input.groupByProduct) {
        qb.andWhere('si.productInStock = :inStock', { inStock: input.inStock });
    } else {
        qb.andWhere('si.inStock = :inStock', { inStock: input.inStock });
    }
}
```

---

## 6. Enabled 状态的协作模式

### 6.1 索引构建阶段

商品的 `enabled` 优先级高于变体的 `enabled`（`indexer.controller.ts:455`）：

```typescript
const item = new SearchIndexItem({
    enabled: product.enabled === false ? false : variant.enabled,
    // ...
});
```

### 6.2 搜索查询过滤

通过 `enabledOnly` 参数控制（`postgres-search-strategy.ts:118-120`）：

```typescript
if (enabledOnly) {
    qb.andWhere('"si"."enabled" = :enabled', { enabled: true });
}
```

Shop API 调用时 `enabledOnly = true`，Admin API 调用时 `enabledOnly = false`。

---

## 7. 协作模式总结

| 维度 | 索引层处理 | 事件驱动 | 搜索过滤 | 备注 |
|------|-----------|----------|----------|------|
| **频道** | ✅ 主键隔离，多频道索引 | ✅ `ProductChannelEvent`<br>`ProductVariantChannelEvent` | ✅ 强制过滤 `channelId` | 完全在索引层处理 |
| **客户分组** | ❌ 无直接支持 | ❌ 无相关事件 | ❌ 无索引过滤 | 通过价格策略间接影响，需应用层处理 |
| **库存状态** | ✅ 可选索引 `inStock`<br>`productInStock` | ✅ `StockMovementEvent` | ✅ 可选过滤 `input.inStock` | 需显式配置启用 |
| **Enabled** | ✅ 索引存储 `enabled` 字段 | ✅ `ProductEvent`<br>`ProductVariantEvent` | ✅ `enabledOnly` 参数 | 商品优先级 > 变体优先级 |

---

## 8. 关键代码参考

### 8.1 索引更新队列任务类型

`search-index.service.ts:35-66` 定义了所有索引更新任务类型：

- `reindex` - 全量重建索引
- `update-product` - 更新商品索引
- `update-variants` - 更新变体索引
- `delete-product` - 删除商品索引
- `delete-variant` - 删除变体索引
- `update-variants-by-id` - 按ID更新变体
- `update-asset` / `delete-asset` - 资产变更
- `assign-product-to-channel` / `remove-product-from-channel` - 商品频道变更
- `assign-variant-to-channel` / `remove-variant-from-channel` - 变体频道变更

### 8.2 合成变体处理

当商品没有变体时，创建合成变体以确保商品可被搜索（`indexer.controller.ts:521-550`）：

```typescript
private async saveSyntheticVariant(ctx: RequestContext, product: Product) {
    const item = new SearchIndexItem({
        productVariantId: 0,  // 特殊标识
        price: 0,
        enabled: false,       // 合成变体默认不启用
        // ...
    });
}
```
