# Vendure 多渠道价格与库存协作机制分析

## 概述

Vendure 通过三层架构实现多渠道下同一商品的不同价格和库存管理：

1. **渠道上下文注入**：请求入口处解析并注入渠道信息
2. **价格计算消费**：根据当前渠道计算商品价格
3. **库存读取裁剪**：根据当前渠道过滤可用库存

## 一、渠道上下文注入（请求入口）

### 1.1 核心组件

**AuthGuard 中间件** (`packages/core/src/api/middleware/auth-guard.ts`)

- 全局守卫，在每个请求处理前执行
- 负责创建 RequestContext 并绑定到请求对象

```typescript
// auth-guard.ts:64-77
if (targetIsFieldResolver) {
    requestContext = internal_getRequestContext(req);
} else {
    const session = await this.getSession(req, res, hasOwnerPermission, info);
    requestContext = await this.requestContextService.fromRequest(req, info, permissions, session);
    // ...
    internal_setRequestContext(req, requestContext, context);
}
```

**RequestContextService** (`packages/core/src/service/helpers/request-context/request-context.service.ts`)

- 从请求中提取 channel token（支持 query 参数或 header）
- 通过 ChannelService 获取对应的 Channel 实体
- 创建包含 channel 信息的 RequestContext

```typescript
// request-context.service.ts:94-122
async fromRequest(req: Request, info?: GraphQLResolveInfo, ...): Promise<RequestContext> {
    const channelToken = this.getChannelToken(req);
    const channel = await this.channelService.getChannelFromToken(channelToken);
    // ...
    return new RequestContext({
        req,
        apiType,
        channel,  // 注入的渠道信息
        languageCode,
        currencyCode,
        session,
        // ...
    });
}

private getChannelToken(req: Request): string {
    const tokenKey = this.configService.apiOptions.channelTokenKey;
    if (req?.query?.[tokenKey]) {
        return req.query[tokenKey];
    } else if (req?.headers?.[tokenKey]) {
        return req.headers[tokenKey] as string;
    }
    return '';
}
```

**RequestContext** (`packages/core/src/api/common/request-context.ts`)

- 贯穿整个请求生命周期的上下文对象
- 包含 `channel`、`channelId`、`currencyCode` 等关键信息
- 通过 `@Ctx()` 装饰器在 resolver 中访问

### 1.2 注入流程

```
HTTP Request
    ↓
AuthGuard.canActivate()
    ↓
RequestContextService.fromRequest()
    ├─ 提取 channelToken (query/header)
    ├─ ChannelService.getChannelFromToken()
    └─ 创建 RequestContext { channel, ... }
    ↓
internal_setRequestContext(req, ctx)
    ↓
Resolver / Controller (@Ctx() ctx)
```

## 二、价格计算（渠道消费）

### 2.1 数据模型

**ProductVariantPrice** (`packages/core/src/entity/product-variant/product-variant-price.entity.ts`)

- 每个 ProductVariant 在每个 Channel 中对应一条价格记录
- 字段：`price`、`channelId`、`currencyCode`

```typescript
@Entity()
export class ProductVariantPrice extends VendureEntity {
    @Money() price: number;
    @EntityId() channelId: ID;
    @Column('varchar') currencyCode: CurrencyCode;
    @ManyToOne(type => ProductVariant, variant => variant.productVariantPrices)
    variant: ProductVariant;
}
```

### 2.2 价格计算流程

**ProductPriceApplicator** (`packages/core/src/service/helpers/product-price-applicator/product-price-applicator.ts`)

价格计算的核心入口，`applyChannelPriceAndTax()` 方法：

```typescript
async applyChannelPriceAndTax(variant: ProductVariant, ctx: RequestContext, order?: Order): Promise<ProductVariant> {
    const { productVariantPriceSelectionStrategy, productVariantPriceCalculationStrategy } =
        this.configService.catalogOptions;
    
    // 步骤1: 选择当前渠道的价格
    const channelPrice = await productVariantPriceSelectionStrategy.selectPrice(
        ctx,
        variant.productVariantPrices,
    );
    
    // 步骤2: 确定税率区域
    const activeTaxZone = await taxZoneStrategy.determineTaxZone(ctx, zones, ctx.channel, order);
    
    // 步骤3: 计算最终价格
    const { price, priceIncludesTax } = await productVariantPriceCalculationStrategy.calculate({
        inputPrice: channelPrice?.price ?? 0,
        productVariantPrice: channelPrice,
        taxCategory: variant.taxCategory,
        productVariant: variant,
        activeTaxZone,
        ctx,  // 传递上下文
    });
    
    variant.listPrice = price;
    variant.listPriceIncludesTax = priceIncludesTax;
    return variant;
}
```

### 2.3 价格选择策略

**DefaultProductVariantPriceSelectionStrategy** (`packages/core/src/config/catalog/default-product-variant-price-selection-strategy.ts`)

```typescript
selectPrice(ctx: RequestContext, prices: ProductVariantPrice[]) {
    // 过滤出当前渠道的价格
    const pricesInChannel = prices.filter(p => idsAreEqual(p.channelId, ctx.channelId));
    // 匹配当前货币
    const priceInCurrency = pricesInChannel.find(p => p.currencyCode === ctx.currencyCode);
    return priceInCurrency;
}
```

### 2.4 价格计算策略

**DefaultProductVariantPriceCalculationStrategy** (`packages/core/src/config/catalog/default-product-variant-price-calculation-strategy.ts`)

```typescript
async calculate(args: ProductVariantPriceCalculationArgs): Promise<PriceCalculationResult> {
    const { inputPrice, activeTaxZone, ctx, taxCategory } = args;
    let price = inputPrice;
    let priceIncludesTax = false;

    // 根据渠道配置决定是否含税
    if (ctx.channel.pricesIncludeTax) {
        const isDefaultZone = idsAreEqual(activeTaxZone.id, ctx.channel.defaultTaxZone.id);
        if (isDefaultZone) {
            priceIncludesTax = true;
        } else {
            // 非默认税区需要计算净价
            const taxRateForDefaultZone = await this.taxRateService.getApplicableTaxRate(...);
            price = roundMoney(taxRateForDefaultZone.netPriceOf(inputPrice));
        }
    }

    return { price, priceIncludesTax };
}
```

### 2.5 价格计算流程总结

```
ProductVariant.productVariantPrices (所有渠道价格)
    ↓
ProductVariantPriceSelectionStrategy.selectPrice(ctx, prices)
    ├─ 过滤: p.channelId === ctx.channelId
    └─ 匹配: p.currencyCode === ctx.currencyCode
    ↓
ProductVariantPrice { price, channelId, currencyCode }
    ↓
ProductVariantPriceCalculationStrategy.calculate({ ctx, inputPrice, ... })
    ├─ 检查 ctx.channel.pricesIncludeTax
    ├─ 计算税额/净价
    └─ 返回 { price, priceIncludesTax }
    ↓
设置 variant.listPrice / variant.listPriceIncludesTax
```

## 三、库存读取（渠道裁剪）

### 3.1 核心策略

**MultiChannelStockLocationStrategy** (`packages/core/src/config/catalog/multi-channel-stock-location-strategy.ts`)

- Vendure 3.1.0+ 的默认库存策略
- 核心思想：根据当前渠道过滤关联的库存位置

### 3.2 库存位置与渠道关联

StockLocation 与 Channel 是多对多关系：

```typescript
// Channel 实体中
@ManyToMany(type => StockLocation, stockLocation => stockLocation.channels)
stockLocations: StockLocation[];
```

### 3.3 库存裁剪实现

**getAvailableStock() - 获取可用库存**

```typescript
async getAvailableStock(ctx: RequestContext, productVariantId: ID, stockLevels: StockLevel[]): Promise<AvailableStock> {
    let stockOnHand = 0;
    let stockAllocated = 0;
    for (const stockLevel of stockLevels) {
        // 关键：检查库存位置是否适用于当前渠道
        const applies = await this.stockLevelAppliesToActiveChannel(ctx, stockLevel);
        if (applies) {
            stockOnHand += stockLevel.stockOnHand;
            stockAllocated += stockLevel.stockAllocated;
        }
    }
    return { stockOnHand, stockAllocated };
}
```

**stockLevelAppliesToActiveChannel() - 渠道过滤**

```typescript
private async stockLevelAppliesToActiveChannel(ctx: RequestContext, stockLevel: StockLevel): Promise<boolean> {
    // 缓存 stockLocation 关联的 channelIds
    const channelIds = await this.channelIdCache.get(stockLevel.stockLocationId, async () => {
        const stockLocation = await this.connection.getEntityOrThrow(ctx, StockLocation, stockLevel.stockLocationId, {
            relations: { channels: true },
        });
        return stockLocation.channels.map(c => c.id);
    });
    // 检查当前 channelId 是否在关联列表中
    return channelIds.includes(ctx.channelId);
}
```

**forAllocation() - 库存分配时的渠道过滤**

```typescript
async forAllocation(ctx: RequestContext, stockLocations: StockLocation[], orderLine: OrderLine, quantity: number): Promise<LocationWithQuantity[]> {
    // ...
    for (const stockLocation of stockLocations) {
        const stockLevel = stockLevels.find(sl => sl.stockLocationId === stockLocation.id);
        // 同样检查渠道关联
        if (stockLevel && (await this.stockLevelAppliesToActiveChannel(ctx, stockLevel))) {
            const quantityAvailable = inventoryNotTracked ? Number.MAX_SAFE_INTEGER : 
                stockLevel.stockOnHand - stockLevel.stockAllocated - effectiveOutOfStockThreshold;
            if (quantityAvailable > 0) {
                locations.push({ location: stockLocation, quantity: Math.min(quantity, quantityAvailable) });
            }
        }
    }
    return locations;
}
```

### 3.4 库存显示策略

**StockDisplayStrategy** (`packages/core/src/config/catalog/stock-display-strategy.ts`)

控制对外暴露的库存信息：

```typescript
export interface StockDisplayStrategy {
    getStockLevel(ctx: RequestContext, productVariant: ProductVariant, saleableStockLevel: number): string;
}

// 默认实现：只返回 IN_STOCK / OUT_OF_STOCK / LOW_STOCK
export class DefaultStockDisplayStrategy implements StockDisplayStrategy {
    getStockLevel(ctx: RequestContext, productVariant: ProductVariant, saleableStockLevel: number): string {
        return saleableStockLevel < 1
            ? 'OUT_OF_STOCK'
            : saleableStockLevel <= this.lowStockLevel
            ? 'LOW_STOCK'
            : 'IN_STOCK';
    }
}
```

### 3.5 库存裁剪流程总结

```
StockLevel[] (所有库存位置的库存记录)
    ↓
MultiChannelStockLocationStrategy.getAvailableStock(ctx, stockLevels)
    ↓
循环每个 StockLevel:
    ├─ stockLevelAppliesToActiveChannel(ctx, stockLevel)
    │   └─ 检查 stockLocation.channels 是否包含 ctx.channelId
    └─ 仅累加适用渠道的 stockOnHand 和 stockAllocated
    ↓
AvailableStock { stockOnHand, stockAllocated }
    ↓
saleableStockLevel = stockOnHand - stockAllocated
    ↓
StockDisplayStrategy.getStockLevel(ctx, variant, saleableStockLevel)
    ↓
对外显示: IN_STOCK / LOW_STOCK / OUT_OF_STOCK
```

## 四、三者协作关系

### 4.1 完整数据流

```
HTTP Request (带 vendure-token header/query)
    ↓
┌─────────────────────────────────────────┐
│ 1. 渠道上下文注入                        │
│   AuthGuard → RequestContextService     │
│   → RequestContext { channel, ... }      │
└─────────────────────────────────────────┘
    ↓
Resolver / Service 方法调用
    ↓
┌─────────────────────────────────────────┐
│ 2. 价格计算消费                          │
│   ProductPriceApplicator                │
│   → PriceSelectionStrategy (按channel过滤)│
│   → PriceCalculationStrategy (含税计算)  │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│ 3. 库存读取裁剪                          │
│   MultiChannelStockLocationStrategy     │
│   → getAvailableStock (按channel裁剪)    │
│   → forAllocation (按channel分配)        │
└─────────────────────────────────────────┘
    ↓
Response (价格 + 库存状态)
```

### 4.2 关键协作点

| 阶段 | 核心类/方法 | 如何使用 RequestContext |
|------|------------|------------------------|
| 注入 | `AuthGuard.canActivate()` | 创建并存储 RequestContext |
| 注入 | `RequestContextService.fromRequest()` | 提取 channelToken，查询 Channel |
| 价格 | `ProductPriceApplicator.applyChannelPriceAndTax()` | 使用 `ctx.channelId`、`ctx.currencyCode`、`ctx.channel.pricesIncludeTax` |
| 价格 | `DefaultProductVariantPriceSelectionStrategy.selectPrice()` | 过滤 `p.channelId === ctx.channelId` |
| 价格 | `DefaultProductVariantPriceCalculationStrategy.calculate()` | 使用 `ctx.channel.pricesIncludeTax` |
| 库存 | `MultiChannelStockLocationStrategy.getAvailableStock()` | 使用 `ctx.channelId` 过滤库存位置 |
| 库存 | `MultiChannelStockLocationStrategy.forAllocation()` | 使用 `ctx.channelId` 过滤可分配库存 |
| 显示 | `DefaultStockDisplayStrategy.getStockLevel()` | 接收 ctx（可扩展渠道特定显示逻辑） |

## 五、扩展点

### 5.1 自定义价格策略

实现 `ProductVariantPriceCalculationStrategy` 接口，在 `VendureConfig` 中配置：

```typescript
export class MyPriceStrategy implements ProductVariantPriceCalculationStrategy {
    async calculate(args: ProductVariantPriceCalculationArgs): Promise<PriceCalculationResult> {
        const { ctx, inputPrice } = args;
        // 根据渠道特性自定义价格逻辑
        if (ctx.channel.code === 'b2b') {
            // B2B 渠道特殊定价
        }
        return { price, priceIncludesTax };
    }
}
```

### 5.2 自定义库存策略

实现 `StockLocationStrategy` 接口，支持更复杂的渠道库存规则：

```typescript
export class MyStockStrategy implements StockLocationStrategy {
    async getAvailableStock(ctx: RequestContext, productVariantId: ID, stockLevels: StockLevel[]) {
        // 自定义渠道库存裁剪逻辑
    }
}
```

### 5.3 自定义库存显示

实现 `StockDisplayStrategy` 接口，控制不同渠道的库存显示粒度：

```typescript
export class MyStockDisplayStrategy implements StockDisplayStrategy {
    getStockLevel(ctx: RequestContext, productVariant: ProductVariant, saleableStockLevel: number): string {
        if (ctx.channel.code === 'admin') {
            return `库存: ${saleableStockLevel}`; // 管理后台显示精确数量
        }
        return saleableStockLevel > 0 ? '有货' : '缺货';
    }
}
```

## 六、关键文件索引

| 功能 | 文件路径 |
|------|---------|
| 渠道上下文注入 | `packages/core/src/api/middleware/auth-guard.ts` |
| RequestContext 创建 | `packages/core/src/service/helpers/request-context/request-context.service.ts` |
| RequestContext 定义 | `packages/core/src/api/common/request-context.ts` |
| 价格应用器 | `packages/core/src/service/helpers/product-price-applicator/product-price-applicator.ts` |
| 价格选择策略 | `packages/core/src/config/catalog/default-product-variant-price-selection-strategy.ts` |
| 价格计算策略 | `packages/core/src/config/catalog/default-product-variant-price-calculation-strategy.ts` |
| 价格实体 | `packages/core/src/entity/product-variant/product-variant-price.entity.ts` |
| 多渠道库存策略 | `packages/core/src/config/catalog/multi-channel-stock-location-strategy.ts` |
| 库存显示策略 | `packages/core/src/config/catalog/default-stock-display-strategy.ts` |
| 渠道实体 | `packages/core/src/entity/channel/channel.entity.ts` |
