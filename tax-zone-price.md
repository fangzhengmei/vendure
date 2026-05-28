# Vendure 税区（Tax Zone）与价格计算服务协作实现分析

## 一、核心实体关系模型

### 1.1 实体关联图

```
ProductVariant (产品变体)
      │
      ├─→ taxCategoryId ──→ TaxCategory (税类)
      │                         │
      │                         └─→ taxRates (一对多)
      │
      └─→ productVariantPrices (多价格，按 Channel)

Channel (销售渠道)
      │
      ├─→ defaultTaxZone ──→ Zone (税区)
      │                         │
      │                         ├─→ members (Region 国家/地区)
      │                         └─→ taxRates (一对多)
      │
      └─→ pricesIncludeTax (布尔值: 价格是否含税)

TaxRate (税率)
      │
      ├─→ category ──→ TaxCategory
      ├─→ zone ──────→ Zone
      ├─→ customerGroup (可选: 客户组)
      └─→ value (税率值 %)
```

### 1.2 关键实体定义

#### TaxCategory (税类)
**文件**: `packages/core/src/entity/tax-category/tax-category.entity.ts`

```typescript
@Entity()
export class TaxCategory extends VendureEntity {
    @Column() name: string;                    // 税类名称 (如 "标准税率"、"零税率")
    @Column({ default: false }) isDefault: boolean;
    
    @OneToMany(type => ProductVariant, productVariant => productVariant.taxCategory)
    productVariants: ProductVariant[];         // 关联的产品变体
    
    @OneToMany(type => TaxRate, taxRate => taxRate.category)
    taxRates: TaxRate[];                       // 关联的税率
}
```

**作用**: 定义商品的税务分类，每个产品变体必须属于一个税类。

---

#### Zone (税区)
**文件**: `packages/core/src/entity/zone/zone.entity.ts`

```typescript
@Entity()
export class Zone extends VendureEntity {
    @Column() name: string;                    // 税区名称 (如 "欧盟"、"美国")
    
    @ManyToMany(type => Region)
    @JoinTable()
    members: Region[];                         // 包含的国家/地区成员
    
    @OneToMany(type => Channel, country => country.defaultTaxZone)
    defaultTaxZoneChannels: Channel[];         // 作为默认税区的 Channel
    
    @OneToMany(type => TaxRate, taxRate => taxRate.zone)
    taxRates: TaxRate[];                       // 关联的税率
}
```

**作用**: 将多个国家/地区分组为一个税务区域，用于统一税率管理。

---

#### TaxRate (税率)
**文件**: `packages/core/src/entity/tax-rate/tax-rate.entity.ts`

```typescript
@Entity()
export class TaxRate extends VendureEntity {
    @Column() name: string;                    // 税率名称
    @Column() enabled: boolean;                // 是否启用
    @Column({ type: 'decimal', precision: 5, scale: 2 }) value: number;  // 税率值 (%)
    
    @ManyToOne(type => TaxCategory, taxCategory => taxCategory.taxRates)
    category: TaxCategory;                     // 关联税类
    
    @ManyToOne(type => Zone, zone => zone.taxRates)
    zone: Zone;                               // 关联税区
    
    @ManyToOne(type => CustomerGroup, { nullable: true })
    customerGroup?: CustomerGroup;             // 可选: 特定客户组
    
    // 辅助计算方法
    taxComponentOf(grossPrice: number): number;    // 从含税价提取税额
    netPriceOf(grossPrice: number): number;        // 从含税价计算净价
    taxPayableOn(netPrice: number): number;        // 从净价计算税额
    grossPriceOf(netPrice: number): number;        // 从净价计算含税价
    
    // 匹配测试: 检查是否适用于指定税区和税类
    test(zone: Zone | ID, taxCategory: TaxCategory | ID): boolean {
        const taxCategoryId = this.isId(taxCategory) ? taxCategory : taxCategory.id;
        const zoneId = this.isId(zone) ? zone : zone.id;
        return idsAreEqual(taxCategoryId, this.categoryId) && idsAreEqual(zoneId, this.zoneId);
    }
}
```

**关键**: `test()` 方法是税率匹配的核心逻辑，通过 **税区 ID + 税类 ID** 双重匹配确定适用税率。

---

## 二、税率解析流程

### 2.1 TaxRateService: 税率服务核心

**文件**: `packages/core/src/service/services/tax-rate.service.ts`

#### 核心功能: 缓存与查找

```typescript
@Injectable()
export class TaxRateService {
    private activeTaxRates: SelfRefreshingCache<TaxRate[], [RequestContext]>;
    
    // 获取适用税率
    async getApplicableTaxRate(
        ctx: RequestContext,
        zone: Zone | ID,
        taxCategory: TaxCategory | ID,
    ): Promise<TaxRate> {
        const rate = (await this.getActiveTaxRates(ctx)).find(r => r.test(zone, taxCategory));
        return rate || this.defaultTaxRate;  // 未找到时返回 0% 税率
    }
    
    // 自刷新缓存: 缓存所有启用的税率
    private async ensureCacheExists() {
        this.activeTaxRates = await createSelfRefreshingCache({
            name: 'TaxRateService.activeTaxRates',
            ttl: this.configService.entityOptions.taxRateCacheTtl,
            refresh: { fn: ctx => this.findActiveTaxRates(ctx), defaultArgs: [RequestContext.empty()] },
        });
    }
    
    private async findActiveTaxRates(ctx: RequestContext): Promise<TaxRate[]> {
        return await this.connection.getRepository(ctx, TaxRate).find({
            relations: ['category', 'zone', 'customerGroup'],
            where: { enabled: true },
        });
    }
}
```

**设计要点**:
- **性能优化**: 使用 `SelfRefreshingCache` 缓存所有启用的税率，避免频繁查询数据库
- **降级策略**: 未找到匹配税率时返回 0% 的默认税率（`defaultTaxRate`）
- **缓存失效**: 创建/更新/删除税率时触发 `updateActiveTaxRates()` 刷新缓存

### 2.2 税率匹配逻辑

```
输入参数: Zone (税区) + TaxCategory (税类)
    │
    ▼
从缓存获取所有启用的 TaxRate
    │
    ▼
遍历每个 TaxRate，调用 test() 方法
    │
    ├─→ 匹配条件 1: taxCategoryId 相等?
    ├─→ 匹配条件 2: zoneId 相等?
    │
    ▼
返回第一个匹配的 TaxRate，无匹配返回 0% 默认税率
```

---

## 三、Channel 税策略覆盖机制

### 3.1 Channel 实体中的税相关配置

**文件**: `packages/core/src/entity/channel/channel.entity.ts`

```typescript
@Entity()
export class Channel extends VendureEntity {
    @ManyToOne(type => Zone, zone => zone.defaultTaxZoneChannels)
    defaultTaxZone: Zone;           // 默认税区
    
    @ManyToOne(type => Zone, zone => zone.defaultShippingZoneChannels)
    defaultShippingZone: Zone;      // 默认配送区
    
    @Column() pricesIncludeTax: boolean;  // 价格是否含税
}
```

### 3.2 TaxZoneStrategy: 税区确定策略

**接口定义**: `packages/core/src/config/tax/tax-zone-strategy.ts`

```typescript
export interface TaxZoneStrategy extends InjectableStrategy {
    determineTaxZone(
        ctx: RequestContext,
        zones: Zone[],
        channel: Channel,
        order?: Order,
    ): Zone | Promise<Zone> | undefined;
}
```

**使用场景**:
1. **产品展示时**: `order` 为 undefined，根据上下文确定税区
2. **订单计算时**: `order` 存在，可结合配送地址确定税区

---

#### 策略 1: DefaultTaxZoneStrategy (默认策略)

**文件**: `packages/core/src/config/tax/default-tax-zone-strategy.ts`

```typescript
export class DefaultTaxZoneStrategy implements TaxZoneStrategy {
    determineTaxZone(ctx: RequestContext, zones: Zone[], channel: Channel, order?: Order): Zone {
        return channel.defaultTaxZone;  // 直接使用 Channel 的默认税区
    }
}
```

**适用场景**: 单区域销售，或所有区域使用统一税率。

---

#### 策略 2: AddressBasedTaxZoneStrategy (基于地址)

**文件**: `packages/core/src/config/tax/address-based-tax-zone-strategy.ts`

```typescript
export class AddressBasedTaxZoneStrategy implements TaxZoneStrategy {
    determineTaxZone(ctx: RequestContext, zones: Zone[], channel: Channel, order?: Order): Zone {
        const countryCode = order?.shippingAddress?.countryCode;
        if (order && countryCode) {
            // 根据配送地址国家查找匹配的税区
            const zone = zones.find(z => z.members?.find(member => member.code === countryCode));
            if (zone) {
                return zone;
            }
        }
        return channel.defaultTaxZone;  // 降级到默认税区
    }
}
```

**适用场景**: 跨境 B2C 销售，需根据配送目的地国家适用不同税率（如欧盟 OSS 方案）。

### 3.3 税计算策略配置

**文件**: `packages/core/src/config/default-config.ts`

```typescript
taxOptions: {
    taxZoneStrategy: new DefaultTaxZoneStrategy(),           // 税区确定策略
    taxLineCalculationStrategy: new DefaultTaxLineCalculationStrategy(),  // 税行计算
    orderTaxCalculationStrategy: new DefaultOrderTaxCalculationStrategy(), // 订单总计计算
}
```

---

## 四、价格计算服务协作流程

### 4.1 场景一: 产品展示价格计算 (ProductPriceApplicator)

**文件**: `packages/core/src/service/helpers/product-price-applicator/product-price-applicator.ts`

```typescript
async applyChannelPriceAndTax(
    variant: ProductVariant,
    ctx: RequestContext,
    order?: Order,
): Promise<ProductVariant> {
    // 步骤 1: 选择 Channel 价格
    const channelPrice = await productVariantPriceSelectionStrategy.selectPrice(...);
    
    // 步骤 2: 确定适用税区 (调用 TaxZoneStrategy)
    const activeTaxZone = await this.requestCache.get(
        ctx,
        CacheKey.ActiveTaxZone_PPA(ctx.channelId),
        () => taxZoneStrategy.determineTaxZone(ctx, zones, ctx.channel, order),
    );
    
    // 步骤 3: 获取适用税率
    const applicableTaxRate = await this.requestCache.get(
        ctx,
        `applicableTaxRate-${activeTaxZone.id}-${variant.taxCategory.id}`,
        () => this.taxRateService.getApplicableTaxRate(ctx, activeTaxZone, variant.taxCategory),
    );
    
    // 步骤 4: 价格计算 (ProductVariantPriceCalculationStrategy)
    const { price, priceIncludesTax } = await productVariantPriceCalculationStrategy.calculate({
        inputPrice: channelPrice?.price ?? 0,
        activeTaxZone,
        ctx,
        taxCategory: variant.taxCategory,
        ...
    });
    
    variant.listPrice = price;
    variant.listPriceIncludesTax = priceIncludesTax;
    variant.taxRateApplied = applicableTaxRate;
    
    return variant;
}
```

**关键设计**:
- **两级缓存**: 
  - RequestContextCache: 同一请求内缓存税区和税率
  - TaxRateService 自刷新缓存: 跨请求缓存税率列表

---

### 4.2 场景二: 订单价格计算 (OrderCalculator)

**文件**: `packages/core/src/service/helpers/order-calculator/order-calculator.ts`

```typescript
async applyPriceAdjustments(
    ctx: RequestContext,
    order: Order,
    promotions: Promotion[],
): Promise<Order> {
    // 步骤 1: 确定税区
    const activeTaxZone = await this.requestContextCache.get(
        ctx,
        CacheKey.ActiveTaxZone(ctx.channelId),
        () => taxZoneStrategy.determineTaxZone(ctx, zones, ctx.channel, order),
    );
    
    // 步骤 2: 税区变更检测
    if (!order.taxZoneId || !idsAreEqual(order.taxZoneId, activeTaxZone.id)) {
        order.taxZoneId = activeTaxZone.id;
        taxZoneChanged = true;
    }
    
    // 步骤 3: 应用税费到订单行
    if (taxZoneChanged) {
        await this.applyTaxes(ctx, order, activeTaxZone);
    }
    
    // 步骤 4: 应用促销
    await this.applyPromotions(ctx, order, promotions);
    
    // 步骤 5: 促销后重新计算税费 (因促销可能改变单价)
    if (order.subTotal !== totalBeforePromotions) {
        await this.applyTaxes(ctx, order, activeTaxZone);
    }
    
    // 步骤 6: 应用配送费
    await this.applyShipping(ctx, order);
    
    // 步骤 7: 计算订单总计
    this.calculateOrderTotals(order);
    
    return order;
}
```

#### 税费应用到订单行

```typescript
private async applyTaxesToOrderLine(
    ctx: RequestContext,
    order: Order,
    line: OrderLine,
    getTaxRate: (taxCategoryId: ID) => Promise<TaxRate>,
) {
    const applicableTaxRate = await getTaxRate(line.taxCategoryId);
    
    // 调用 TaxLineCalculationStrategy 计算税行
    line.taxLines = await taxLineCalculationStrategy.calculate({
        ctx,
        applicableTaxRate,
        order,
        orderLine: line,
    });
}

// 创建带缓存的税率获取器
private createTaxRateGetter(ctx: RequestContext, activeZone: Zone): (taxCategoryId: ID) => Promise<TaxRate> {
    const taxRateCache = new Map<ID, TaxRate>();  // 单次计算内的缓存
    
    return async (taxCategoryId: ID): Promise<TaxRate> => {
        const cached = taxRateCache.get(taxCategoryId);
        if (cached) return cached;
        const rate = await this.taxRateService.getApplicableTaxRate(ctx, activeZone, taxCategoryId);
        taxRateCache.set(taxCategoryId, rate);
        return rate;
    };
}
```

---

### 4.3 税行计算策略 (TaxLineCalculationStrategy)

**接口**: `packages/core/src/config/tax/tax-line-calculation-strategy.ts`

```typescript
export interface TaxLineCalculationStrategy extends InjectableStrategy {
    calculate(args: CalculateTaxLinesArgs): TaxLine[] | Promise<TaxLine[]>;
}

export interface CalculateTaxLinesArgs {
    ctx: RequestContext;
    order: Order;
    orderLine: OrderLine;
    applicableTaxRate: TaxRate;
}
```

#### 默认实现: DefaultTaxLineCalculationStrategy

**文件**: `packages/core/src/config/tax/default-tax-line-calculation-strategy.ts`

```typescript
export class DefaultTaxLineCalculationStrategy implements TaxLineCalculationStrategy {
    calculate(args: CalculateTaxLinesArgs): TaxLine[] {
        const { orderLine, applicableTaxRate } = args;
        return [applicableTaxRate.apply(orderLine.proratedUnitPrice)];
    }
}
```

**扩展点**: 自定义策略可调用第三方税务 API 或查询自定义税率表。

---

### 4.4 订单税总计计算策略 (OrderTaxCalculationStrategy)

**接口**: `packages/core/src/config/tax/order-tax-calculation-strategy.ts`

```typescript
export interface OrderTaxCalculationStrategy extends InjectableStrategy {
    calculateOrderTotals(order: Order): OrderTotalsResult;  // 计算小计、总计
    calculateTaxSummary(order: Order): OrderTaxSummary[];    // 生成税汇总
}
```

#### 默认实现: 逐行汇总

**文件**: `packages/core/src/config/tax/default-order-tax-calculation-strategy.ts`

```typescript
export class DefaultOrderTaxCalculationStrategy implements OrderTaxCalculationStrategy {
    calculateOrderTotals(order: Order): OrderTotalsResult {
        let subTotal = 0;
        let subTotalWithTax = 0;
        for (const line of order.lines ?? []) {
            subTotal += line.proratedLinePrice;           // 每行独立取整
            subTotalWithTax += line.proratedLinePriceWithTax;
        }
        // ... 附加费、配送费同理
        return { subTotal, subTotalWithTax, shipping, shippingWithTax };
    }
}
```

**替代方案**: `OrderLevelTaxCalculationStrategy` 按税率分组汇总后取整，消除逐行取整累积误差。

---

## 五、协作流程总图

### 5.1 订单价格计算完整流程

```
订单变更触发 (添加商品/修改地址等)
        │
        ▼
OrderCalculator.applyPriceAdjustments()
        │
        ├─→ 确定税区 (TaxZoneStrategy)
        │     └─→ DefaultTaxZoneStrategy → 使用 Channel.defaultTaxZone
        │     └─→ AddressBasedTaxZoneStrategy → 配送地址国家匹配
        │
        ├─→ 税区变更检测 → taxZoneChanged
        │
        ├─→ 应用税费 (税区变更时)
        │     │
        │     ├─→ createTaxRateGetter() → 建立税类→税率缓存
        │     ├─→ 遍历订单行
        │     │     ├─→ TaxRateService.getApplicableTaxRate(zone, taxCategory)
        │     │     │     └─→ TaxRate.test() 匹配
        │     │     └─→ TaxLineCalculationStrategy.calculate()
        │     └─→ 生成 OrderLine.taxLines
        │
        ├─→ 应用促销
        │     └─→ 可能改变订单行单价
        │
        ├─→ 重新应用税费 (促销后)
        │     └─→ 因促销可能影响税基
        │
        ├─→ 应用配送费
        │     └─→ ShippingMethod 含税率
        │
        └─→ 计算订单总计 (OrderTaxCalculationStrategy)
              └─→ 汇总所有行的税费
```

### 5.2 缓存层级设计

```
┌──────────────────────────────────────────────────────────┐
│ 缓存层级 1: TaxRateService.activeTaxRates               │
│ 范围: 全局跨请求                                         │
│ TTL: 可配置 (taxRateCacheTtl)                           │
│ 内容: 所有 enabled=true 的 TaxRate (带关联)              │
└──────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────┐
│ 缓存层级 2: RequestContextCacheService                  │
│ 范围: 单次请求内                                         │
│ Key: CacheKey.ActiveTaxZone(channelId)                  │
│ 内容: 本次请求确定的 activeTaxZone                       │
└──────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────┐
│ 缓存层级 3: OrderCalculator.createTaxRateGetter         │
│ 范围: 单次 applyPriceAdjustments 调用内                  │
│ 实现: Map<taxCategoryId, TaxRate>                       │
│ 内容: 本次计算已查询过的税率                              │
└──────────────────────────────────────────────────────────┘
```

---

## 六、关键代码位置索引

| 功能模块 | 文件路径 |
|---------|---------|
| TaxRate 实体 | `packages/core/src/entity/tax-rate/tax-rate.entity.ts` |
| TaxCategory 实体 | `packages/core/src/entity/tax-category/tax-category.entity.ts` |
| Zone 实体 | `packages/core/src/entity/zone/zone.entity.ts` |
| Channel 实体 | `packages/core/src/entity/channel/channel.entity.ts` |
| TaxRateService | `packages/core/src/service/services/tax-rate.service.ts` |
| OrderCalculator | `packages/core/src/service/helpers/order-calculator/order-calculator.ts` |
| ProductPriceApplicator | `packages/core/src/service/helpers/product-price-applicator/product-price-applicator.ts` |
| TaxZoneStrategy 接口 | `packages/core/src/config/tax/tax-zone-strategy.ts` |
| DefaultTaxZoneStrategy | `packages/core/src/config/tax/default-tax-zone-strategy.ts` |
| AddressBasedTaxZoneStrategy | `packages/core/src/config/tax/address-based-tax-zone-strategy.ts` |
| TaxLineCalculationStrategy | `packages/core/src/config/tax/tax-line-calculation-strategy.ts` |
| OrderTaxCalculationStrategy | `packages/core/src/config/tax/order-tax-calculation-strategy.ts` |
| 默认配置 | `packages/core/src/config/default-config.ts` |

---

## 七、设计特点总结

1. **策略模式广泛应用**: 税区确定、税行计算、订单税总计均可自定义策略
2. **多层缓存优化**: 从全局到单次计算，三级缓存确保性能
3. **双重匹配机制**: 税率通过 **税区 + 税类** 双重条件匹配
4. **税区动态感知**: 订单计算时检测税区变更并重新计算税费
5. **促销-税费交互**: 促销后重新计算税费，确保税基正确
6. **降级设计**: 无匹配税率时返回 0%，避免系统异常
