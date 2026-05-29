# 客户分组 × 渠道可见性 × 价目拼接——全链路解读

Vendure 的"客户分组 → 渠道过滤 → 价目拼接"并不在一条线性管道上，而是分散在三个独立阶段、由 `RequestContext` 贯穿串联。下面按请求生命周期逐层拆解。

---

## 1. 请求入口：RequestContext 确定渠道与身份

**核心文件** `packages/core/src/api/common/request-context.ts`

每个请求在 `AuthGuard.canActivate()` 阶段就会创建 `RequestContext`，其中包含：

| 属性 | 来源 | 用途 |
|------|------|------|
| `ctx.channel` | 请求头 `vendure-token` → `ChannelService.getChannelFromToken()` | 决定当前渠道 |
| `ctx.channelId` | `ctx.channel.id` | 所有后续查询的渠道过滤依据 |
| `ctx.session` | Session 缓存 | 携带当前用户信息（含 `customerId`） |
| `ctx.currencyCode` | 渠道默认货币 / 查询参数覆写 | 价格选择与计算 |

> **关键点**：渠道在请求最早期就已确定，之后所有数据访问都围绕 `ctx.channelId` 展开。

---

## 2. 渠道过滤：ChannelAware 实体的可见性墙

### 2.1 实体层面

Channel 实体（`packages/core/src/entity/channel/channel.entity.ts`）通过 `ManyToMany` 关系持有对多种实体的引用：

```
Channel ──┬── products          (Product)
           ├── productVariants   (ProductVariant)
           ├── facetValues       (FacetValue)
           ├── collections       (Collection)
           ├── promotions        (Promotion)
           ├── paymentMethods    (PaymentMethod)
           ├── shippingMethods   (ShippingMethod)
           ├── customers         (Customer)   ← 客户也属于渠道
           ├── roles             (Role)
           └── stockLocations    (StockLocation)
```

反过来，这些实体都实现 `ChannelAware` 接口，持有 `channels: Channel[]`。

### 2.2 查询层面——ListQueryBuilder 的渠道内连接

**核心文件** `packages/core/src/service/helpers/list-query-builder/list-query-builder.ts:344-348`

```ts
if (extendedOptions.channelId) {
    qb.innerJoin(`${qb.alias}.channels`, 'lqb__channel',
                 'lqb__channel.id = :channelId', {
        channelId: extendedOptions.channelId,
    });
}
```

当 Service 调用 `listQueryBuilder.build(Entity, options, { channelId: ctx.channelId, ... })` 时，会自动 `INNER JOIN` 渠道关联表，**只返回当前渠道可见的实体**。

### 2.3 单条查询——findOneInChannel

`TransactionalConnection.findOneInChannel(ctx, Entity, id, ctx.channelId, ...)` 在查询时额外加 `WHERE channel.id = :channelId` 条件，确保只返回当前渠道可见的实体。

### 2.4 渠道分配

- **创建时**：`ChannelService.assignToCurrentChannel()` 自动将实体分配到当前渠道 + 默认渠道。
- **手动分配**：`ChannelService.assignToChannels()` 将实体加入指定渠道。
- **移除**：`ChannelService.removeFromChannels()` 从渠道中移除实体，同时删除对应的 `ProductVariantPrice` 记录。

### 2.5 客户的渠道归属

Customer 实体同样是 `ChannelAware`（`packages/core/src/entity/customer/customer.entity.ts:61-63`）：

```ts
@ManyToMany(type => Channel, channel => channel.customers)
@JoinTable()
channels: Channel[];
```

- 创建客户时自动分配到当前渠道。
- 验证邮箱时也会 `assignToChannels(ctx, Customer, customerId, [ctx.channelId])`。
- 查询时 `CustomerService.findOneByUserId()` 可选 `filterOnChannel=true`，只返回当前渠道下的客户。
- `CustomerService.getCustomerGroups()` 内部也通过 `findOneInChannel` 确保只查当前渠道内的客户。

---

## 3. 客户分组匹配

### 3.1 实体关系

**核心文件** `packages/core/src/entity/customer-group/customer-group.entity.ts`

```
CustomerGroup ←──ManyToMany──→ Customer
                       (customer.groups / group.customers)
```

CustomerGroup **本身不是 ChannelAware**，不绑定渠道。它是全局概念。

### 3.2 分组分配

**核心文件** `packages/core/src/service/services/customer-group.service.ts`

- `addCustomersToGroup()`：把客户加入分组，发布 `CustomerGroupChangeEvent`。
- `removeCustomersFromGroup()`：把客户移出分组，同样发布事件。
- `getCustomersFromIds()` 查询时仍然过滤 `channel.id = :channelId`，确保只操作当前渠道内可见的客户。

### 3.3 分组在促销中的匹配

**核心文件** `packages/core/src/config/promotion/conditions/customer-group-condition.ts`

```ts
async check(ctx, order, args) {
    if (!order.customer) return false;
    const groupIds = await groupIdCache.get(customerId, async () => {
        const groups = await customerService.getCustomerGroups(ctx, customerId);
        return groups.map(g => g.id);
    });
    return !!groupIds.find(id => idsAreEqual(id, args.customerGroupId));
}
```

匹配逻辑：
1. 从订单取 `order.customer.id`。
2. 调用 `customerService.getCustomerGroups()` 获取该客户所有分组 ID（带缓存，TTL 1 周）。
3. 检查目标分组 ID 是否在其中。
4. 当 `CustomerGroupChangeEvent` 触发时，缓存被主动清除。

> **注意**：分组匹配不受渠道限制——分组是全局的，只要客户属于该分组，无论在哪个渠道的促销中都能匹配。但**促销本身**受渠道约束（见下一节）。

---

## 4. 价目拼接：从 ProductVariantPrice 到最终价格

### 4.1 价格存储模型

**核心文件** `packages/core/src/entity/product-variant/product-variant-price.entity.ts`

```
ProductVariantPrice {
    price: number;           // 税前价格（分）
    channelId: ID;           // 所属渠道
    currencyCode: CurrencyCode;  // 币种
    variant: ProductVariant;     // 所属变体
}
```

一个 ProductVariant 可以有**多个** ProductVariantPrice，每个 (channelId, currencyCode) 组合一条。

### 4.2 价格选择——ProductVariantPriceSelectionStrategy

**核心文件** `packages/core/src/config/catalog/default-product-variant-price-selection-strategy.ts`

```ts
selectPrice(ctx: RequestContext, prices: ProductVariantPrice[]) {
    const pricesInChannel = prices.filter(p => idsAreEqual(p.channelId, ctx.channelId));
    const priceInCurrency = pricesInChannel.find(p => p.currencyCode === ctx.currencyCode);
    return priceInCurrency;
}
```

两步过滤：
1. **按渠道过滤**：只保留 `channelId === ctx.channelId` 的价格。
2. **按币种匹配**：在渠道内的价格中找 `currencyCode === ctx.currencyCode` 的那一条。

### 4.3 价格应用——ProductPriceApplicator

**核心文件** `packages/core/src/service/helpers/product-price-applicator/product-price-applicator.ts`

```
applyChannelPriceAndTax(variant, ctx, order?) 流程：

  ① productVariantPriceSelectionStrategy.selectPrice(ctx, variant.productVariantPrices)
     → 选出当前渠道+币种的 ProductVariantPrice（channelPrice）

  ② 确定 activeTaxZone
     taxZoneStrategy.determineTaxZone(ctx, zones, ctx.channel, order)

  ③ 获取 applicableTaxRate
     taxRateService.getApplicableTaxRate(ctx, activeTaxZone, variant.taxCategory)

  ④ productVariantPriceCalculationStrategy.calculate({
       inputPrice: channelPrice?.price ?? 0,
       productVariantPrice: channelPrice,
       ...
     })
     → 根据 channel.pricesIncludeTax 设置决定含税/不含税

  ⑤ 填充 variant 运行时字段：
     variant.listPrice = price
     variant.listPriceIncludesTax = priceIncludesTax
     variant.taxRateApplied = applicableTaxRate
     variant.currencyCode = channelPrice?.currencyCode ?? ctx.currencyCode
```

### 4.4 变体价格 getter

**核心文件** `packages/core/src/entity/product-variant/product-variant.entity.ts:76-99`

```ts
get price(): number {
    return roundMoney(
        this.listPriceIncludesTax
            ? this.taxRateApplied.netPriceOf(this.listPrice)  // 含税→去税
            : this.listPrice                                   // 不含税→直接返回
    );
}

get priceWithTax(): number {
    return roundMoney(
        this.listPriceIncludesTax
            ? this.listPrice                                   // 含税→直接返回
            : this.taxRateApplied.grossPriceOf(this.listPrice) // 不含税→加税
    );
}
```

---

## 5. 促销价叠加——OrderCalculator 中的分组→渠道→价目衔接

**核心文件** `packages/core/src/service/helpers/order-calculator/order-calculator.ts`

当订单发生变更时，`OrderCalculator.applyPriceAdjustments()` 的执行顺序：

```
applyPriceAdjustments(ctx, order, promotions)
│
├── 1. 确定税区 (activeTaxZone)
│
├── 2. 对新增/变更的 OrderLine 应用税
│      applyTaxesToOrderLine()
│
├── 3. 计算订单小计
│
├── 4. 如果税区变化 → 全量重算税
│
├── 5. 应用促销（这是分组介入的环节）
│      ├── applyOrderItemPromotions()
│      │     → 对每个 line，遍历 promotions：
│      │       promotion.test(ctx, order)  ← 这里触发 customerGroup.check()
│      │       如果匹配 → promotion.apply() 生成 Adjustment
│      │
│      └── applyOrderPromotions()
│            → 订单级促销（含客户分组条件的）
│
├── 6. 促销改变了价格 → 重算税
│
└── 7. 计算运费 + 运费促销
```

### 促销的渠道约束

**核心文件** `packages/core/src/service/services/promotion.service.ts:289-301`

```ts
getActivePromotionsInChannel(ctx) {
    // WHERE channel.id = :channelId AND enabled = true
}
```

只有**分配到当前渠道且已启用**的促销才会被加载。

### 衔接全景

```
客户请求 → RequestContext(ctx.channelId, ctx.session)
            │
            ▼
    ┌─ 渠道过滤 ─────────────────────────────────┐
    │  ListQueryBuilder: INNER JOIN channels      │
    │  findOneInChannel: WHERE channel.id = ?     │
    │  → 只返回当前渠道可见的商品/变体/促销/客户  │
    └─────────────────────────────────────────────┘
            │
            ▼
    ┌─ 价目选择 ─────────────────────────────────┐
    │  ProductVariantPriceSelectionStrategy       │
    │  ① 过滤 channelId === ctx.channelId        │
    │  ② 匹配 currencyCode === ctx.currencyCode  │
    │  → 得到渠道级基础价格                        │
    └─────────────────────────────────────────────┘
            │
            ▼
    ┌─ 税费计算 ─────────────────────────────────┐
    │  activeTaxZone ← ctx.channel + order 地址   │
    │  applicableTaxRate ← zone + taxCategory     │
    │  PriceCalculationStrategy ← 含税标志        │
    │  → 得到 listPrice / priceWithTax            │
    └─────────────────────────────────────────────┘
            │
            ▼
    ┌─ 促销叠加 ─────────────────────────────────┐
    │  promotions ← 当前渠道 + enabled            │
    │  对每个 promotion.test(ctx, order):         │
    │    └ customerGroup.check():                │
    │        客户的分组 ID ∩ 促销要求的分组 ID?   │
    │    如果匹配 → promotion.apply() → Adjustment│
    │  → 价格被 Adjustment 修改                   │
    └─────────────────────────────────────────────┘
            │
            ▼
        最终价格
```

---

## 6. 关键细节与易混淆点

### 6.1 CustomerGroup 不绑定渠道

CustomerGroup 实体没有 `channels` 关系，它是全局的。但操作分组中的客户时（如 `getGroupCustomers`、`getCustomersFromIds`），仍然会过滤 `channel.id = ctx.channelId`，确保只操作当前渠道内可见的客户。

### 6.2 客户分组匹配在促销环节，不在价目选择环节

Vendure **没有** "按客户分组选价格" 的内置机制。`DefaultProductVariantPriceSelectionStrategy` 只看渠道+币种。如果需要实现"VIP 客户看到不同价格"，需要：

- **方案 A**：自定义 `ProductVariantPriceSelectionStrategy`，在其中读取客户的分组信息来选择不同价格行。
- **方案 B**：通过促销（Promotion）实现，用 `customer_group` 条件 + 折扣 Action 来间接实现分组定价。

### 6.3 ProductVariantPrice 是渠道级而非分组级

`ProductVariantPrice` 的唯一键是 `(variantId, channelId, currencyCode)`，没有 `customerGroupId` 字段。分组与价格的关联只能通过促销间接实现。

### 6.4 分配变体到渠道时自动创建价格

`assignProductVariantsToChannel()` 会：
1. 先 `applyChannelPriceAndTax` 算出源渠道的含税/不含税价。
2. 根据目标渠道的 `pricesIncludeTax` 选择基数。
3. 乘以 `priceFactor`（默认 1）。
4. 创建目标渠道的 `ProductVariantPrice`。

### 6.5 删除渠道级联删除价格

`ChannelService.delete()` 会级联删除该渠道下所有 `ProductVariantPrice`：
```ts
await this.connection.getRepository(ctx, ProductVariantPrice).delete({ channelId: id });
```

### 6.6 缓存与失效

- 客户分组 ID 缓存在 `PromotionCondition:customer_group:{customerId}`，TTL 1 周。
- `CustomerGroupChangeEvent`（客户加入/移出分组）触发缓存清除。
- `RequestContextCacheService` 在请求级别缓存税区、税率等，请求结束即失效。

---

## 7. 源文件索引

| 概念 | 文件路径 |
|------|----------|
| RequestContext | `packages/core/src/api/common/request-context.ts` |
| Channel 实体 | `packages/core/src/entity/channel/channel.entity.ts` |
| ChannelService | `packages/core/src/service/services/channel.service.ts` |
| Customer 实体 | `packages/core/src/entity/customer/customer.entity.ts` |
| CustomerGroup 实体 | `packages/core/src/entity/customer-group/customer-group.entity.ts` |
| CustomerGroupService | `packages/core/src/service/services/customer-group.service.ts` |
| CustomerService | `packages/core/src/service/services/customer.service.ts` |
| CustomerGroup 促销条件 | `packages/core/src/config/promotion/conditions/customer-group-condition.ts` |
| ProductVariantPrice 实体 | `packages/core/src/entity/product-variant/product-variant-price.entity.ts` |
| ProductVariant 实体 | `packages/core/src/entity/product-variant/product-variant.entity.ts` |
| PriceSelectionStrategy 接口 | `packages/core/src/config/catalog/product-variant-price-selection-strategy.ts` |
| DefaultPriceSelectionStrategy | `packages/core/src/config/catalog/default-product-variant-price-selection-strategy.ts` |
| PriceCalculationStrategy 接口 | `packages/core/src/config/catalog/product-variant-price-calculation-strategy.ts` |
| DefaultPriceCalculationStrategy | `packages/core/src/config/catalog/default-product-variant-price-calculation-strategy.ts` |
| ProductPriceApplicator | `packages/core/src/service/helpers/product-price-applicator/product-price-applicator.ts` |
| ProductVariantService | `packages/core/src/service/services/product-variant.service.ts` |
| OrderCalculator | `packages/core/src/service/helpers/order-calculator/order-calculator.ts` |
| PromotionService | `packages/core/src/service/services/promotion.service.ts` |
| ListQueryBuilder | `packages/core/src/service/helpers/list-query-builder/list-query-builder.ts` |
