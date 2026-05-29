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

### 3.3 分组在促销中的匹配——与渠道可见性的隐耦合

**核心文件** `packages/core/src/config/promotion/conditions/customer-group-condition.ts`

```ts
async check(ctx, order, args) {
    if (!order.customer) {
        return false;                                         // ← 失败路径 A
    }
    const customerId = order.customer.id;
    const groupIds = await groupIdCache.get(customerId, async () => {
        const groups = await customerService.getCustomerGroups(ctx, customerId);
        return groups.map(g => g.id);
    });

    return !!groupIds.find(id => idsAreEqual(id, args.customerGroupId));
}
```

表面上看，这段代码只关心"客户是否属于某个分组"。但 `getCustomerGroups` 内部调用了 `findOneInChannel`，这使得渠道可见性成了一个**隐式前提条件**——客户必须在当前渠道可见，才能查到分组。

#### `getCustomerGroups` 的渠道过滤分支

`CustomerService.getCustomerGroups()`（`customer.service.ts:179-197`）的实现：

```ts
async getCustomerGroups(ctx: RequestContext, customerId: ID): Promise<CustomerGroup[]> {
    const customerWithGroups = await this.connection.findOneInChannel(
        ctx,
        Customer,
        customerId,
        ctx?.channelId,       // ← 关键：用当前请求的渠道过滤
        { relations: ['groups'], where: { deletedAt: IsNull() } },
    );
    if (customerWithGroups) {
        return customerWithGroups.groups;     // ← 命中路径：客户在此渠道可见
    } else {
        return [];                             // ← 失败路径：客户不在此渠道，返回空组
    }
}
```

而 `findOneInChannel`（`transactional-connection.ts:350-379`）生成的 SQL 是：

```sql
SELECT customer.* FROM customer
  LEFT JOIN customer_channels__channel AS __channel
    ON __channel.customerId = customer.id
  WHERE customer.id = :id
    AND __channel.channelId = :channelId   -- 只匹配当前渠道
```

**如果 Customer 没有被分配到 `ctx.channelId` 对应的渠道，`findOneInChannel` 返回 `undefined`，`getCustomerGroups` 走 `else` 分支，返回 `[]`。**

#### 缓存键不含渠道 ID 的隐患

缓存键格式为 `PromotionCondition:customer_group:${customerId}`——**不包含 `channelId`**。

这意味着同一个 `customerId` 在不同渠道下首次查询后，结果会被缓存，后续渠道的请求可能命中错误渠道的缓存结果（详见第 8 节）。

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

### 6.1 CustomerGroup 不绑定渠道，但分组查询受渠道约束

CustomerGroup 实体没有 `channels` 关系，它是全局的。**然而**，读取客户分组的路径（`getCustomerGroups` → `findOneInChannel`）会在查询 Customer 实体时过滤渠道。这意味着：

- CustomerGroup 的**定义**是全局的——一个分组不专属于某个渠道。
- 但**客户→分组的关系**是通过 Customer 实体间接获取的，而 Customer 是 ChannelAware 的。
- 因此，"客户属于某分组"这个事实虽然本身与渠道无关，但**查询这个事实的代码路径**会被渠道可见性阻断。

操作分组中的客户时（`getGroupCustomers`、`getCustomersFromIds`），同样会过滤 `channel.id = :channelId`。

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

## 8. 跨渠道场景：从 order.customer 到 customerGroup.check() 的命中与失败路径

Vendure 多渠道架构下，同一个 Customer 可以属于多个 Channel（通过 `customer.channels` 多对多关系），但客户的**渠道归属并不总是对称的**。一个在 Channel A 注册的老客户，首次在 Channel B 下单时，可能尚未被分配到 Channel B。这种不对称会直接影响促销中的客户分组匹配。

### 8.1 order.customer 从何而来——不受渠道过滤

当 `OrderService.applyPriceAdjustments()` 被调用时（`order.service.ts:2312-2383`），传入的 `order` 对象是经 `OrderService.findOne()` 加载的（`order.service.ts:214-233`），其默认 relations 包含 `'customer'` 和 `'customer.user'`：

```ts
const effectiveRelations = relations ?? [
    'channels',
    'customer',          // ← 直接 LEFT JOIN customer 表
    'customer.user',
    'lines',
    ...
];
```

这是普通的 `ManyToOne` 关联加载，**不走 `findOneInChannel`，不做渠道过滤**。因此 `order.customer` 永远是有效的——只要订单上关联了客户，无论该客户是否在当前渠道可见，`order.customer` 都不为 `undefined`。

### 8.2 命中路径 vs 失败路径——并列对照

设定：客户 C 属于分组 G（VIP），且在 Channel A 注册。Channel B 有一个促销 P，条件为 `customer_group = G`。

#### 路径 A：客户在当前渠道可见 → 分组命中

```
请求进入 Channel B（ctx.channelId = B）
  │
  ├─ OrderService.findOne() 加载 order
  │    → order.customer = C（无渠道过滤，始终存在）
  │
  ├─ OrderService.applyPriceAdjustments(ctx, order, promotions)
  │    ├─ promotionService.getActivePromotionsInChannel(ctx)
  │    │    → 返回 Channel B 的促销 [P]（P 已分配到 Channel B）
  │    │
  │    └─ orderCalculator.applyPromotions(ctx, order, [P])
  │         └─ P.test(ctx, order)
  │              └─ customerGroup.check(ctx, order, { customerGroupId: G })
  │                   │
  │                   ├─ order.customer 存在 → 跳过失败路径 A
  │                   │
  │                   ├─ groupIdCache.get(C.id, fn)
  │                   │    └─ 缓存未命中 → 调用 fn()
  │                   │         └─ customerService.getCustomerGroups(ctx, C.id)
  │                   │              └─ findOneInChannel(ctx, Customer, C.id, ctx.channelId=B)
  │                   │                   → SQL: WHERE customer.id=C AND __channel.channelId=B
  │                   │                   → 客户 C 已被分配到 Channel B ✅
  │                   │                   → 返回 customerWithGroups
  │                   │                   → 返回 customerWithGroups.groups = [G]
  │                   │              → groupIds = [G.id]
  │                   │         → 缓存写入: C.id → [G.id]
  │                   │
  │                   └─ groupIds.find(id === G.id) → ✅ 匹配成功
  │                        → P.apply() → 生成 Adjustment → 折扣生效
```

**前提**：客户 C 必须已被分配到 Channel B。分配发生在：
- C 在 Channel B 注册时：`customerService.create()` → `channelService.assignToCurrentChannel()`
- C 在 Channel B 验证邮箱时：`verifyCustomerEmailAddress()` → `channelService.assignToChannels()`
- C 在 Channel B 下单时（guest checkout）：`createOrUpdate()` → `customer.channels.push(ctx.channel)`
- 管理员手动分配：`channelService.assignToChannels()`

#### 路径 B：客户不在当前渠道 → 分组静默失败

```
请求进入 Channel B（ctx.channelId = B）
  │
  ├─ OrderService.findOne() 加载 order
  │    → order.customer = C（无渠道过滤，仍然存在）
  │
  ├─ OrderService.applyPriceAdjustments(ctx, order, promotions)
  │    ├─ promotionService.getActivePromotionsInChannel(ctx)
  │    │    → 返回 Channel B 的促销 [P]
  │    │
  │    └─ orderCalculator.applyPromotions(ctx, order, [P])
  │         └─ P.test(ctx, order)
  │              └─ customerGroup.check(ctx, order, { customerGroupId: G })
  │                   │
  │                   ├─ order.customer 存在 → 跳过失败路径 A
  │                   │
  │                   ├─ groupIdCache.get(C.id, fn)
  │                   │    └─ 缓存未命中 → 调用 fn()
  │                   │         └─ customerService.getCustomerGroups(ctx, C.id)
  │                   │              └─ findOneInChannel(ctx, Customer, C.id, ctx.channelId=B)
  │                   │                   → SQL: WHERE customer.id=C AND __channel.channelId=B
  │                   │                   → 客户 C 未被分配到 Channel B ❌
  │                   │                   → LEFT JOIN 结果为空
  │                   │                   → 返回 undefined
  │                   │              → 走 else 分支 → 返回 []
  │                   │         → groupIds = []
  │                   │         → 缓存写入: C.id → []    ← 注意：空数组被缓存！
  │                   │
  │                   └─ groupIds.find(id === G.id) → ❌ 不匹配
  │                        → check() 返回 false
  │                        → 促销 P 不生效，无折扣
  │                        → 无任何日志或错误提示
```

**关键点**：这是一个**静默失败**——没有报错，没有日志，只是分组匹配结果为 `false`。客户明明属于分组 G，但在 Channel B 中查询分组时返回了空数组。

#### 失败路径的触发场景

| 场景 | 为什么客户不在渠道中 |
|------|----------------------|
| 管理员在 Channel A 创建了客户，未分配到 Channel B | 创建客户时 `assignToCurrentChannel()` 只分配当前渠道 + 默认渠道 |
| 客户在 Channel A 注册，从未在 Channel B 登录 | `registerCustomerAccount()` 只在当前渠道创建/关联客户 |
| 管理员通过 `addCustomerToOrder` 切换了客户，但新客户不属于当前渠道 | `setCustomerForOrder()` 检查了渠道归属，但旧订单数据可能残留 |
| 数据库直接操作，跳过渠道分配 | 绕过了 Service 层的渠道分配逻辑 |

### 8.3 缓存键不含渠道 ID 的跨渠道污染

缓存键 `PromotionCondition:customer_group:${customerId}` 只按客户 ID 缓存，**不区分渠道**。这导致两种问题：

#### 污染场景 1：渠道 A 先查，渠道 B 后查——B 意外命中 A 的结果

```
T1: Channel A 请求 → getCustomerGroups(ctx_A, C.id)
    → findOneInChannel(ctx_A, Customer, C.id, channelA)
    → 客户在 Channel A 可见 ✅ → 返回 [G1, G2]
    → 缓存: C.id → [G1.id, G2.id]

T2: Channel B 请求（客户不在 B）→ groupIdCache.get(C.id)
    → 缓存命中！返回 [G1.id, G2.id]
    → customerGroup.check() 返回 true
    → 促销在 Channel B 生效——但客户在 Channel B 不可见，不应命中分组
```

**结果**：客户本不应在 Channel B 被识别为分组 G1 成员，但因缓存了 Channel A 的结果，分组条件错误地命中了。

#### 污染场景 2：渠道 B 先查，渠道 A 后查——A 被误判为无分组

```
T1: Channel B 请求（客户不在 B）→ getCustomerGroups(ctx_B, C.id)
    → findOneInChannel(ctx_B, Customer, C.id, channelB)
    → 客户在 Channel B 不可见 ❌ → 返回 []
    → 缓存: C.id → []

T2: Channel A 请求 → groupIdCache.get(C.id)
    → 缓存命中！返回 []
    → customerGroup.check() 返回 false
    → 促销在 Channel A 也不生效——但客户明明在 Channel A 属于分组 G1
```

**结果**：本应在 Channel A 命中的分组促销，因为 Channel B 的查询先污染了缓存，导致失效。

缓存 TTL 为 1 周，这意味着污染一旦发生，可能持续影响很长时间。`CustomerGroupChangeEvent` 只在客户被加入/移出分组时清除缓存，**渠道分配变更不会触发缓存清除**。

### 8.4 正常情况下为什么不会频繁触发

实际运行中，上述失败路径并不容易触发，原因在于 Vendure 的客户-渠道关联机制：

1. **注册时自动关联**：`registerCustomerAccount()` 最终调用 `createOrUpdate()`，后者会 `customer.channels.push(ctx.channel)`，确保客户出现在新渠道。
2. **邮箱验证时补关联**：`verifyCustomerEmailAddress()` 显式调用 `channelService.assignToChannels(ctx, Customer, customer.id, [ctx.channelId])`。
3. **Guest checkout 时关联**：`createOrUpdate()` 同样会 push 当前渠道。
4. **订单切换客户时校验**：`setCustomerForOrder()` 检查目标客户是否属于订单所在的所有渠道，否则抛出 `UserInputError`。

因此，正常流程下，只要客户在某个渠道有活跃会话，该客户就应该已被分配到该渠道。失败路径主要发生在：
- 数据库直接操作绕过了 Service 层
- 渠道分配因并发问题丢失
- 自定义代码未正确处理渠道分配

### 8.5 路径判定速查表

```
customerGroup.check(ctx, order, args) 结果
│
├── order.customer === undefined
│    └── → false（失败路径 A：无客户）
│
├── order.customer 存在
│    ├── getCustomerGroups(ctx, customerId) 返回 []
│    │    ├── findOneInChannel 找不到客户（客户不在当前渠道）
│    │    │    └── → false（失败路径 B：渠道可见性阻断）
│    │    │
│    │    └── 客户在当前渠道，但确实不属于任何分组
│    │         └── → false（正常不匹配）
│    │
│    └── getCustomerGroups(ctx, customerId) 返回 [G1, G2, ...]
│         ├── args.customerGroupId 在 groupIds 中
│         │    └── → true（命中路径 ✅）
│         │
│         └── args.customerGroupId 不在 groupIds 中
│              └── → false（正常不匹配）
```

---

## 9. 源文件索引

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
| TransactionalConnection | `packages/core/src/connection/transactional-connection.ts` |
