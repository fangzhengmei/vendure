# Vendure 多渠道价格与库存协作机制分析

## 概述

Vendure 通过三层架构实现多渠道下同一商品的不同价格和库存管理：

1. **渠道上下文注入**：请求入口处解析并注入渠道信息
2. **价格计算消费**：根据当前渠道计算商品价格
3. **库存读取裁剪**：根据当前渠道过滤可用库存

---

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

    const requestContextShouldBeReinitialized = await this.setActiveChannel(requestContext, session);
    if (requestContextShouldBeReinitialized) {
        requestContext = await this.requestContextService.fromRequest(req, info, permissions, session);
    }
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
    const apiType = getApiType(info);
    const languageCode = this.getLanguageCode(req, channel);
    const currencyCode = this.getCurrencyCode(req, channel);
    // ...
    return new RequestContext({
        req,
        apiType,
        channel,
        languageCode,
        currencyCode,
        session,
        isAuthorized,
        authorizedAsOwnerOnly,
        translationFn,
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
- 包含 `_channel`、`channelId`、`_currencyCode` 等关键信息
- 通过 `@Ctx()` 装饰器在 resolver 中访问

```typescript
// request-context.ts:179-215
export class RequestContext {
    private readonly _languageCode: LanguageCode;
    private readonly _currencyCode: CurrencyCode;
    private readonly _channel: Channel;
    private readonly _session?: CachedSession;
    // ...

    constructor(options: {
        req?: Request;
        apiType: ApiType;
        channel: Channel;
        session?: CachedSession;
        languageCode?: LanguageCode;
        currencyCode?: CurrencyCode;
        // ...
    }) {
        this._channel = channel;
        this._languageCode = languageCode || (channel && channel.defaultLanguageCode);
        this._currencyCode = currencyCode || (channel && channel.defaultCurrencyCode);
        // ...
    }

    get channelId(): ID {
        return this._channel.id;
    }

    get channel(): Channel {
        return this._channel;
    }

    get currencyCode(): CurrencyCode {
        return this._currencyCode;
    }
}
```

### 1.2 渠道写入会话与上下文重建

**setActiveChannel 方法** (`auth-guard.ts:92-136`)

在创建 RequestContext 后，AuthGuard 会尝试将当前渠道写入会话：

```typescript
private async setActiveChannel(
    requestContext: RequestContext,
    session?: CachedSession,
): Promise<boolean> {
    if (!session) {
        return false;
    }
    // 检查会话是否需要更新 activeChannel
    const activeChannelShouldBeSet =
        !session.activeChannelId || session.activeChannelId !== requestContext.channelId;
    
    if (activeChannelShouldBeSet) {
        // 写入会话的 activeChannel
        await this.sessionService.setActiveChannel(session, requestContext.channel);
        
        // 如果有用户，同时将用户关联到当前渠道
        if (requestContext.activeUserId) {
            const customer = await this.customerService.findOneByUserId(...);
            if (customer) {
                await this.channelService.assignToChannels(requestContext, Customer, customer.id, [
                    requestContext.channelId,
                ]);
            }
        }
        return true; // 返回 true 触发上下文重建
    }
    return false;
}
```

**SessionService.setActiveChannel** (`session.service.ts:294-307`)

```typescript
async setActiveChannel(serializedSession: CachedSession, channel: Channel): Promise<CachedSession> {
    const session = await this.connection.rawConnection.getRepository(Session).findOne({
        where: { id: serializedSession.id },
        relations: ['user', 'user.roles', 'user.roles.channels'],
    });
    if (session) {
        session.activeChannel = channel;
        await this.connection.rawConnection.getRepository(Session).save(session, { reload: false });
        const updatedSerializedSession = this.serializeSession(session);
        await this.sessionCacheStrategy.set(updatedSerializedSession);
        return updatedSerializedSession;
    }
    return serializedSession;
}
```

**上下文重建触发条件**：

当 `setActiveChannel()` 返回 `true` 时（即会话的 activeChannel 被更新），AuthGuard 会重新调用 `requestContextService.fromRequest()` 重建 RequestContext，确保上下文与会话状态一致。

### 1.3 完整注入流程

```
HTTP Request (带 vendure-token header/query)
    ↓
AuthGuard.canActivate()
    ├─ 解析请求，判断是否为 field resolver
    │
    ├─ [非 field resolver]
    │   ├─ getSession() → 获取或创建会话
    │   ├─ requestContextService.fromRequest()
    │   │   ├─ getChannelToken(req) → 从 query/header 提取
    │   │   ├─ channelService.getChannelFromToken(token)
    │   │   ├─ getLanguageCode(req, channel)
    │   │   └─ getCurrencyCode(req, channel)
    │   │       └─ 验证: currencyCode 是否在 channel.availableCurrencyCodes 中
    │   │
    │   ├─ setActiveChannel(ctx, session)
    │   │   ├─ 检查: !session.activeChannelId || session.activeChannelId !== ctx.channelId
    │   │   ├─ [是] sessionService.setActiveChannel(session, ctx.channel)
    │   │   │   ├─ 更新数据库 Session.activeChannel
    │   │   │   └─ 更新缓存
    │   │   ├─ [有用户] channelService.assignToChannels(Customer, userId, [ctx.channelId])
    │   │   └─ 返回 true/false
    │   │
    │   ├─ [返回 true] 重新调用 requestContextService.fromRequest()
    │   └─ internal_setRequestContext(req, ctx, context)
    │
    └─ [field resolver] internal_getRequestContext(req)
        ↓
Resolver / Controller (@Ctx() ctx)
```

---

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

### 2.2 真实查询链路：价格字段解析

**GraphQL Resolver 入口** (`product-variant-entity.resolver.ts:49-75`)

```typescript
@ResolveField()
async price(@Ctx() ctx: RequestContext, @Parent() productVariant: ProductVariant): Promise<number> {
    return this.productVariantService.hydratePriceFields(ctx, productVariant, 'price');
}

@ResolveField()
async priceWithTax(@Ctx() ctx: RequestContext, @Parent() productVariant: ProductVariant): Promise<number> {
    return this.productVariantService.hydratePriceFields(ctx, productVariant, 'priceWithTax');
}
```

**hydratePriceFields - 请求级缓存** (`product-variant.service.ts:727-766`)

```typescript
async hydratePriceFields<F extends 'currencyCode' | 'price' | 'priceWithTax' | 'taxRateApplied'>(
    ctx: RequestContext,
    variant: ProductVariant,
    priceField: F,
): Promise<ProductVariant[F]> {
    const cacheKey = `hydrate-variant-price-fields-${variant.id}`;
    let populatePricesPromise = this.requestCache.get<Promise<ProductVariant>>(ctx, cacheKey);

    if (!populatePricesPromise) {
        populatePricesPromise = new Promise(async (resolve, reject) => {
            try {
                // [分支1] 价格数据未加载
                if (!variant.productVariantPrices?.length) {
                    const variantWithPrices = await this.connection.getEntityOrThrow(
                        ctx, ProductVariant, variant.id,
                        { relations: ['productVariantPrices'], includeSoftDeleted: true },
                    );
                    variant.productVariantPrices = variantWithPrices.productVariantPrices;
                }
                // [分支2] 税率分类未加载
                if (!variant.taxCategory) {
                    const variantWithTaxCategory = await this.connection.getEntityOrThrow(
                        ctx, ProductVariant, variant.id,
                        { relations: ['taxCategory'], includeSoftDeleted: true },
                    );
                    variant.taxCategory = variantWithTaxCategory.taxCategory;
                }
                resolve(await this.applyChannelPriceAndTax(variant, ctx, undefined, true));
            } catch (e) { reject(e); }
        });
        this.requestCache.set(ctx, cacheKey, populatePricesPromise);
    }
    const hydratedVariant = await populatePricesPromise;
    return hydratedVariant[priceField];
}
```

**缓存机制**：
- 使用 `RequestContextCacheService` 基于 `WeakMap<RequestContext, Map<any, any>>` 实现
- 缓存 key: `hydrate-variant-price-fields-${variant.id}`
- 缓存生命周期：与 RequestContext 绑定，请求结束自动回收
- 同一请求中多次查询 price/priceWithTax/currencyCode 只执行一次价格计算

**ProductPriceApplicator** (`packages/core/src/service/helpers/product-price-applicator/product-price-applicator.ts`)

价格计算的核心入口，`applyChannelPriceAndTax()` 方法：

```typescript
async applyChannelPriceAndTax(
    variant: ProductVariant,
    ctx: RequestContext,
    order?: Order,
    throwIfNoPriceFound = false,
): Promise<ProductVariant> {
    const { productVariantPriceSelectionStrategy, productVariantPriceCalculationStrategy } =
        this.configService.catalogOptions;
    
    // 步骤1: 选择当前渠道的价格
    const channelPrice = await productVariantPriceSelectionStrategy.selectPrice(
        ctx,
        variant.productVariantPrices,
    );
    
    if (!channelPrice && throwIfNoPriceFound) {
        throw new InternalServerError('error.no-price-found-for-channel', {
            variantId: variant.id,
            channel: ctx.channel.code,
        });
    }
    
    // 步骤2: 确定税率区域（使用请求级缓存）
    const { taxZoneStrategy } = this.configService.taxOptions;
    const zones = await this.requestCache.get(ctx, CacheKey.AllZones, () =>
        this.zoneService.getAllWithMembers(ctx),
    );
    const activeTaxZone = await this.requestCache.get(
        ctx,
        CacheKey.ActiveTaxZone_PPA(ctx.channelId),
        () => taxZoneStrategy.determineTaxZone(ctx, zones, ctx.channel, order),
    );
    if (!activeTaxZone) {
        throw new InternalServerError('error.no-active-tax-zone');
    }
    
    // 步骤3: 获取适用税率（使用请求级缓存）
    const applicableTaxRate = await this.requestCache.get(
        ctx,
        `applicableTaxRate-${activeTaxZone.id}-${variant.taxCategory.id}`,
        () => this.taxRateService.getApplicableTaxRate(ctx, activeTaxZone, variant.taxCategory),
    );
    
    // 步骤4: 计算最终价格
    const { price, priceIncludesTax } = await productVariantPriceCalculationStrategy.calculate({
        inputPrice: channelPrice?.price ?? 0,
        productVariantPrice: channelPrice,
        taxCategory: variant.taxCategory,
        productVariant: variant,
        activeTaxZone,
        ctx,
    });
    
    variant.listPrice = price;
    variant.listPriceIncludesTax = priceIncludesTax;
    variant.taxRateApplied = applicableTaxRate;
    variant.currencyCode = channelPrice?.currencyCode ?? ctx.currencyCode;
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

    // [分支] 渠道配置价格是否含税
    if (ctx.channel.pricesIncludeTax) {
        // [子分支] 是否为默认税区
        const isDefaultZone = idsAreEqual(activeTaxZone.id, ctx.channel.defaultTaxZone.id);
        if (isDefaultZone) {
            priceIncludesTax = true;
        } else {
            // 非默认税区：计算净价
            const taxRateForDefaultZone = await this.taxRateService.getApplicableTaxRate(
                ctx, ctx.channel.defaultTaxZone, taxCategory,
            );
            price = roundMoney(taxRateForDefaultZone.netPriceOf(inputPrice));
        }
    }
    return { price, priceIncludesTax };
}
```

### 2.5 批量查询场景：列表页价格应用

**applyPricesAndTranslateVariants** (`product-variant.service.ts:773-787`)

```typescript
private async applyPricesAndTranslateVariants(
    ctx: RequestContext,
    variants: ProductVariant[],
): Promise<Array<Translated<ProductVariant>>> {
    return await Promise.all(
        variants.map(async variant => {
            const variantWithPrices = await this.applyChannelPriceAndTax(variant, ctx);
            return this.translator.translate(variantWithPrices, ctx, [
                'options', 'facetValues', ['facetValues', 'facet'],
            ]);
        }),
    );
}
```

### 2.6 价格计算完整链路

```
GraphQL Query { product { variants { price priceWithTax } } }
    ↓
ProductVariantEntityResolver.price() / priceWithTax()
    ↓
ProductVariantService.hydratePriceFields(ctx, variant, field)
    ├─ 检查请求缓存: `hydrate-variant-price-fields-${variant.id}`
    │   ├─ [缓存命中] 返回缓存的 Promise
    │   └─ [缓存未命中]
    │       ├─ [分支] 加载 productVariantPrices (DB 查询)
    │       ├─ [分支] 加载 taxCategory (DB 查询)
    │       └─ 调用 applyChannelPriceAndTax()
    ↓
ProductPriceApplicator.applyChannelPriceAndTax(variant, ctx)
    ├─ 步骤1: PriceSelectionStrategy.selectPrice(ctx, prices)
    │   ├─ 过滤: p.channelId === ctx.channelId
    │   └─ 匹配: p.currencyCode === ctx.currencyCode
    │
    ├─ 步骤2: 确定 activeTaxZone
    │   ├─ 缓存: CacheKey.AllZones
    │   └─ 缓存: CacheKey.ActiveTaxZone_PPA(ctx.channelId)
    │
    ├─ 步骤3: 获取 applicableTaxRate
    │   └─ 缓存: `applicableTaxRate-${activeTaxZone.id}-${taxCategory.id}`
    │
    └─ 步骤4: PriceCalculationStrategy.calculate({ ctx, ... })
        ├─ [分支] ctx.channel.pricesIncludeTax ?
        │   ├─ [是]
        │   │   ├─ [分支] activeTaxZone === channel.defaultTaxZone ?
        │   │   │   ├─ [是] priceIncludesTax = true
        │   │   │   └─ [否] 计算净价
        │   │   └─ 返回 { price, priceIncludesTax }
        │   └─ [否] 返回 { price, priceIncludesTax: false }
        └─ 设置 variant.listPrice / listPriceIncludesTax
    ↓
返回价格字段值
```

---

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

### 3.3 真实查询链路：库存字段解析

**GraphQL Resolver 入口** (`product-variant-entity.resolver.ts:150-153, 178-191`)

```typescript
// Shop API: 库存显示
@ResolveField()
async stockLevel(@Ctx() ctx: RequestContext, @Parent() productVariant: ProductVariant): Promise<string> {
    return this.productVariantService.getDisplayStockLevel(ctx, productVariant);
}

// Admin API: 精确库存数量
@ResolveField()
async stockOnHand(@Ctx() ctx: RequestContext, @Parent() productVariant: ProductVariant): Promise<number> {
    const { stockOnHand } = await this.stockLevelService.getAvailableStock(ctx, productVariant.id);
    return stockOnHand;
}

@ResolveField()
async stockAllocated(@Ctx() ctx: RequestContext, @Parent() productVariant: ProductVariant): Promise<number> {
    const { stockAllocated } = await this.stockLevelService.getAvailableStock(ctx, productVariant.id);
    return stockAllocated;
}
```

**getDisplayStockLevel** (`product-variant.service.ts:359-363`)

```typescript
async getDisplayStockLevel(ctx: RequestContext, variant: ProductVariant): Promise<string> {
    const { stockDisplayStrategy } = this.configService.catalogOptions;
    const saleableStockLevel = await this.getSaleableStockLevel(ctx, variant);
    return stockDisplayStrategy.getStockLevel(ctx, variant, saleableStockLevel);
}
```

**getSaleableStockLevel** (`product-variant.service.ts:321-339`)

```typescript
async getSaleableStockLevel(ctx: RequestContext, variant: ProductVariant): Promise<number> {
    const { outOfStockThreshold, trackInventory } = await this.globalSettingsService.getSettings(ctx);

    // [分支1] 库存跟踪是否关闭
    const inventoryNotTracked =
        variant.trackInventory === GlobalFlag.FALSE ||
        (variant.trackInventory === GlobalFlag.INHERIT && trackInventory === false);
    
    if (inventoryNotTracked) {
        return Number.MAX_SAFE_INTEGER;
    }
    
    // 获取渠道裁剪后的可用库存
    const { stockOnHand, stockAllocated } = await this.stockLevelService.getAvailableStock(
        ctx, variant.id,
    );
    
    // [分支2] 使用全局还是本地库存阈值
    const effectiveOutOfStockThreshold = variant.useGlobalOutOfStockThreshold
        ? outOfStockThreshold
        : variant.outOfStockThreshold;

    return stockOnHand - stockAllocated - effectiveOutOfStockThreshold;
}
```

**StockLevelService.getAvailableStock** (`stock-level.service.ts:72-80`)

```typescript
async getAvailableStock(ctx: RequestContext, productVariantId: ID): Promise<AvailableStock> {
    const { stockLocationStrategy } = this.configService.catalogOptions;
    // 查询该 variant 的所有库存记录（所有库存位置）
    const stockLevels = await this.connection.getRepository(ctx, StockLevel).find({
        where: { productVariantId },
    });
    // 委托给策略进行渠道裁剪
    return stockLocationStrategy.getAvailableStock(ctx, productVariantId, stockLevels);
}
```

### 3.4 库存裁剪实现

**getAvailableStock() - 获取可用库存** (`multi-channel-stock-location-strategy.ts:73-88`)

```typescript
async getAvailableStock(
    ctx: RequestContext,
    productVariantId: ID,
    stockLevels: StockLevel[],
): Promise<AvailableStock> {
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

**stockLevelAppliesToActiveChannel() - 渠道过滤与缓存** (`multi-channel-stock-location-strategy.ts:143-161`)

```typescript
private async stockLevelAppliesToActiveChannel(
    ctx: RequestContext,
    stockLevel: StockLevel,
): Promise<boolean> {
    // 使用全局缓存（7天 TTL），按 stockLocationId 缓存其关联的 channelIds
    const channelIds = await this.channelIdCache.get(stockLevel.stockLocationId, async () => {
        const stockLocation = await this.connection.getEntityOrThrow(
            ctx, StockLocation, stockLevel.stockLocationId,
            { relations: { channels: true } },
        );
        return stockLocation.channels.map(c => c.id);
    });
    // 检查当前 channelId 是否在关联列表中
    return channelIds.includes(ctx.channelId);
}
```

**缓存机制**：
- 使用 `CacheService` 创建全局缓存，TTL: 7天
- 缓存 key: `MultiChannelStockLocationStrategy:StockLocationChannelIds:${stockLocationId}`
- 缓存失效：StockLocation 更新时触发（通过 StockLocationEvent）

```typescript
// multi-channel-stock-location-strategy.ts:54-66
this.channelIdCache = this.cacheService.createCache({
    options: { ttl: ms('7 days'), tags: ['StockLocation'] },
    getKey: id => this.getCacheKey(id),
});

// 监听 StockLocation 更新事件，失效缓存
this.eventBus
    .ofType(StockLocationEvent)
    .pipe(filter(event => event.type !== 'created'))
    .subscribe(({ entity }) => this.channelIdCache.delete(this.getCacheKey(entity.id)));
```

**forAllocation() - 库存分配时的渠道过滤** (`multi-channel-stock-location-strategy.ts:97-136`)

```typescript
async forAllocation(
    ctx: RequestContext,
    stockLocations: StockLocation[],
    orderLine: OrderLine,
    quantity: number,
): Promise<LocationWithQuantity[]> {
    // 使用请求级缓存获取该 variant 的所有库存记录
    const stockLevels = await this.getStockLevelsForVariant(ctx, orderLine.productVariantId);
    const variant = await this.connection.getEntityOrThrow(...);
    
    let totalAllocated = 0;
    const locations: LocationWithQuantity[] = [];
    const { inventoryNotTracked, effectiveOutOfStockThreshold } = await this.getVariantStockSettings(ctx, variant);
    
    for (const stockLocation of stockLocations) {
        const stockLevel = stockLevels.find(sl => sl.stockLocationId === stockLocation.id);
        // 检查渠道关联
        if (stockLevel && (await this.stockLevelAppliesToActiveChannel(ctx, stockLevel))) {
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

**getStockLevelsForVariant - 请求级缓存** (`multi-channel-stock-location-strategy.ts:167-179`)

```typescript
private getStockLevelsForVariant(ctx: RequestContext, productVariantId: ID): Promise<StockLevel[]> {
    return this.requestContextCache.get(
        ctx,
        `MultiChannelStockLocationStrategy.stockLevels.${productVariantId}`,
        () => this.connection.getRepository(ctx, StockLevel).find({
            where: { productVariantId },
            loadEagerRelations: false,
        }),
    );
}
```

### 3.5 库存显示策略

**DefaultStockDisplayStrategy** (`packages/core/src/config/catalog/default-stock-display-strategy.ts`)

```typescript
export class DefaultStockDisplayStrategy implements StockDisplayStrategy {
    constructor(private lowStockLevel: number = 2) {}
    getStockLevel(ctx: RequestContext, productVariant: ProductVariant, saleableStockLevel: number): string {
        return saleableStockLevel < 1
            ? 'OUT_OF_STOCK'
            : saleableStockLevel <= this.lowStockLevel
            ? 'LOW_STOCK'
            : 'IN_STOCK';
    }
}
```

### 3.6 库存读取完整链路

```
GraphQL Query { product { variants { stockLevel } } }
    ↓
ProductVariantEntityResolver.stockLevel()
    ↓
ProductVariantService.getDisplayStockLevel(ctx, variant)
    ↓
ProductVariantService.getSaleableStockLevel(ctx, variant)
    ├─ 获取全局设置: outOfStockThreshold, trackInventory
    ├─ [分支] inventoryNotTracked ?
    │   ├─ [是] 返回 Number.MAX_SAFE_INTEGER
    │   └─ [否]
    │       └─ StockLevelService.getAvailableStock(ctx, variantId)
    │           ↓
    │           MultiChannelStockLocationStrategy.getAvailableStock(ctx, variantId, stockLevels)
    │               ├─ 循环所有 StockLevel
    │               │   ├─ stockLevelAppliesToActiveChannel(ctx, stockLevel)
    │               │   │   ├─ 全局缓存: StockLocationChannelIds:${stockLocationId}
    │               │   │   ├─ [缓存未命中] DB 查询 StockLocation.channels
    │               │   │   └─ 返回 channelIds.includes(ctx.channelId)
    │               │   └─ [适用] 累加 stockOnHand / stockAllocated
    │               └─ 返回 { stockOnHand, stockAllocated }
    │
    ├─ [分支] useGlobalOutOfStockThreshold ?
    │   ├─ [是] 使用全局 outOfStockThreshold
    │   └─ [否] 使用 variant.outOfStockThreshold
    └─ 计算: stockOnHand - stockAllocated - effectiveOutOfStockThreshold
    ↓
StockDisplayStrategy.getStockLevel(ctx, variant, saleableStockLevel)
    ├─ [分支] saleableStockLevel < 1 → OUT_OF_STOCK
    ├─ [分支] saleableStockLevel <= lowStockLevel → LOW_STOCK
    └─ [其他] → IN_STOCK
    ↓
返回库存状态字符串
```

---

## 四、三者协作关系

### 4.1 完整数据流

```
HTTP Request (带 vendure-token header/query)
    ↓
┌─────────────────────────────────────────────────────────┐
│ 1. 渠道上下文注入                                        │
│                                                          │
│ AuthGuard.canActivate()                                  │
│   ├─ getSession()                                        │
│   ├─ RequestContextService.fromRequest()                 │
│   │   ├─ getChannelToken(req)                            │
│   │   ├─ getChannelFromToken(token) → Channel            │
│   │   └─ new RequestContext({ channel, ... })            │
│   ├─ setActiveChannel(ctx, session)                      │
│   │   └─ [更新] Session.activeChannel = channel          │
│   └─ internal_setRequestContext(req, ctx)                │
│                                                          │
│ 输出: RequestContext {                                   │
│   _channel: Channel,                                     │
│   _currencyCode: CurrencyCode,                           │
│   channelId: ID,                                         │
│   ...                                                    │
│ }                                                        │
└─────────────────────────────────────────────────────────┘
    ↓
Resolver / Service 方法调用 (@Ctx() ctx)
    ↓
┌─────────────────────────────────────────────────────────┐
│ 2. 价格计算消费                                          │
│                                                          │
│ ProductVariantService.hydratePriceFields(ctx, variant)   │
│   ├─ 请求缓存: `hydrate-variant-price-fields-${id}`       │
│   └─ ProductPriceApplicator.applyChannelPriceAndTax()    │
│       ├─ PriceSelectionStrategy.selectPrice(ctx, prices) │
│       │   └─ 过滤: p.channelId === ctx.channelId         │
│       ├─ 缓存: AllZones, ActiveTaxZone                   │
│       └─ PriceCalculationStrategy.calculate({ ctx })     │
│           └─ 使用: ctx.channel.pricesIncludeTax          │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ 3. 库存读取裁剪                                          │
│                                                          │
│ ProductVariantService.getDisplayStockLevel(ctx, variant) │
│   └─ ProductVariantService.getSaleableStockLevel()       │
│       └─ StockLevelService.getAvailableStock(ctx, id)    │
│           └─ MultiChannelStockLocationStrategy           │
│              .getAvailableStock(ctx, id, stockLevels)    │
│               ├─ stockLevelAppliesToActiveChannel()      │
│               │   ├─ 全局缓存: StockLocationChannelIds   │
│               │   └─ 检查: channelIds.includes(ctx.channelId)
│               └─ 仅累加适用渠道的库存                     │
└─────────────────────────────────────────────────────────┘
    ↓
Response (价格 + 库存状态)
```

### 4.2 RequestContext 字段传递

| 字段 | 注入点 | 价格计算使用点 | 库存读取使用点 |
|------|--------|----------------|----------------|
| `ctx.channel` | RequestContextService.fromRequest() | PriceCalculationStrategy: `ctx.channel.pricesIncludeTax` | - |
| `ctx.channelId` | RequestContext 构造函数 | PriceSelectionStrategy: `p.channelId === ctx.channelId` | MultiChannelStockLocationStrategy: `channelIds.includes(ctx.channelId)` |
| `ctx.currencyCode` | RequestContext 构造函数 | PriceSelectionStrategy: `p.currencyCode === ctx.currencyCode` | - |
| `ctx` (整个对象) | AuthGuard | 作为参数传递给所有策略方法 | 作为参数传递给所有策略方法 |

### 4.3 缓存命中点汇总

| 缓存层级 | 缓存位置 | Key | 用途 |
|---------|---------|-----|------|
| 请求级 | RequestContextCacheService | `hydrate-variant-price-fields-${variantId}` | 同一请求中价格字段只计算一次 |
| 请求级 | RequestContextCacheService | `CacheKey.AllZones` | 同一请求中所有区域只查询一次 |
| 请求级 | RequestContextCacheService | `CacheKey.ActiveTaxZone_PPA(${channelId})` | 同一请求中激活税区只计算一次 |
| 请求级 | RequestContextCacheService | `applicableTaxRate-${zoneId}-${taxCategoryId}` | 同一请求中同一税区+分类只查询一次 |
| 请求级 | RequestContextCacheService | `MultiChannelStockLocationStrategy.stockLevels.${variantId}` | 同一请求中库存记录只查询一次 |
| 全局 | CacheService (7天TTL) | `MultiChannelStockLocationStrategy:StockLocationChannelIds:${stockLocationId}` | 库存位置关联的渠道列表 |
| 会话级 | SessionCacheStrategy | Session token | 包含 activeChannelId |

### 4.4 关键分支条件

| 位置 | 分支条件 | 影响 |
|------|---------|------|
| AuthGuard | `!session.activeChannelId \|\| session.activeChannelId !== ctx.channelId` | 触发会话更新和上下文重建 |
| PriceSelectionStrategy | `p.channelId === ctx.channelId` | 过滤出当前渠道的价格 |
| PriceSelectionStrategy | `p.currencyCode === ctx.currencyCode` | 匹配当前货币的价格 |
| PriceCalculationStrategy | `ctx.channel.pricesIncludeTax` | 决定是否进行含税计算 |
| PriceCalculationStrategy | `activeTaxZone.id === ctx.channel.defaultTaxZone.id` | 决定是否直接使用含税价还是计算净价 |
| getSaleableStockLevel | `variant.trackInventory === FALSE \|\| (INHERIT && !trackInventory)` | 不跟踪库存时返回无限库存 |
| getSaleableStockLevel | `variant.useGlobalOutOfStockThreshold` | 决定使用全局还是本地库存阈值 |
| stockLevelAppliesToActiveChannel | `channelIds.includes(ctx.channelId)` | 决定库存位置是否计入当前渠道 |

---

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

---

## 六、关键文件索引

| 功能 | 文件路径 |
|------|---------|
| 渠道上下文注入 | `packages/core/src/api/middleware/auth-guard.ts` |
| RequestContext 创建 | `packages/core/src/service/helpers/request-context/request-context.service.ts` |
| RequestContext 定义 | `packages/core/src/api/common/request-context.ts` |
| 会话渠道设置 | `packages/core/src/service/services/session.service.ts` |
| 价格字段解析 | `packages/core/src/api/resolvers/entity/product-variant-entity.resolver.ts` |
| 价格应用器 | `packages/core/src/service/helpers/product-price-applicator/product-price-applicator.ts` |
| 价格服务方法 | `packages/core/src/service/services/product-variant.service.ts` |
| 价格选择策略 | `packages/core/src/config/catalog/default-product-variant-price-selection-strategy.ts` |
| 价格计算策略 | `packages/core/src/config/catalog/default-product-variant-price-calculation-strategy.ts` |
| 价格实体 | `packages/core/src/entity/product-variant/product-variant-price.entity.ts` |
| 多渠道库存策略 | `packages/core/src/config/catalog/multi-channel-stock-location-strategy.ts` |
| 库存服务 | `packages/core/src/service/services/stock-level.service.ts` |
| 库存显示策略 | `packages/core/src/config/catalog/default-stock-display-strategy.ts` |
| 请求级缓存 | `packages/core/src/cache/request-context-cache.service.ts` |
| 渠道实体 | `packages/core/src/entity/channel/channel.entity.ts` |
