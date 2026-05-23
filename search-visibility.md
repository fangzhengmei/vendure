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
| `SearchJobBufferService` | 缓冲更新服务管理 | `search-job-buffer.service.ts` |
| `SearchIndexJobBuffer` | 索引更新任务缓冲逻辑 | `search-index-job-buffer.ts` |

### 1.2 数据流向

**无缓冲模式**：
```
实体变更 → 事件总线 → DefaultSearchPlugin → SearchIndexService → JobQueue
                                                         ↓
                                                IndexerController
                                                         ↓
                                                  SearchIndexItem
                                                         ↓
                                              搜索查询 → SearchStrategy
```

**有缓冲模式**：
```
实体变更 → 事件总线 → DefaultSearchPlugin → SearchIndexService → JobBufferService
                                                                         ↓
                                                               JobBufferStorage
                                                                         ↓
                                                    runPendingSearchIndexUpdates mutation
                                                                         ↓
                                                                  flush() → reduce()
                                                                         ↓
                                                                      JobQueue
                                                                         ↓
                                                                IndexerController
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

## 3. 缓冲更新时序与可见性一致性

### 3.1 启用条件

缓冲更新通过插件配置启用（`default-search-plugin.ts:79-80`）：

```typescript
DefaultSearchPlugin.init({
  bufferUpdates: true,
})
```

启动时注册缓冲区（`search-job-buffer.service.ts:25-30`）：

```typescript
onApplicationBootstrap(): any {
    if (this.bufferUpdates === true) {
        this.jobQueueService.addBuffer(this.searchIndexJobBuffer);
        this.jobQueueService.addBuffer(this.collectionJobBuffer);
    }
}
```

### 3.2 事件触发阶段

**可缓冲的任务类型**（`search-index-job-buffer.ts:16-21`）：

```typescript
collect(job: Job<UpdateIndexQueueJobData>): boolean | Promise<boolean> {
    return (
        job.queueName === 'update-search-index' &&
        ['update-product', 'update-variants', 'update-variants-by-id'].includes(job.data.type)
    );
}
```

**不可缓冲的任务类型**（直接执行）：
- `reindex` - 全量重建
- `delete-product` / `delete-variant` - 删除操作
- `update-asset` / `delete-asset` - 资产变更
- `assign-product-to-channel` / `remove-product-from-channel` - 频道分配
- `assign-variant-to-channel` / `remove-variant-from-channel` - 频道分配

**触发流程**（`job-queue.ts:90-109`）：

```typescript
async add(data: Data, options?: JobOptions<Data>): Promise<SubscribableJob<Data>> {
    const job = new Job<any>({ data, queueName: this.options.name, ... });

    // 检查是否有缓冲区收集此任务
    const isBuffered = await this.jobBufferService.add(job);
    if (!isBuffered) {
        // 无缓冲：直接加入队列执行
        const addedJob = await this.jobQueueStrategy.add(job, options);
        return new SubscribableJob(addedJob, this.jobQueueStrategy);
    } else {
        // 有缓冲：返回虚拟 job，实际执行延迟到 flush
        const bufferedJob = new Job({ ...job, id: 'buffered' });
        return new SubscribableJob(bufferedJob, this.jobQueueStrategy);
    }
}
```

### 3.3 缓存队列阶段

**收集逻辑**（`job-buffer.service.ts:39-49`）：

```typescript
async add(job: Job): Promise<boolean> {
    let collected = false;
    for (const buffer of this.buffers) {
        const shouldCollect = await buffer.collect(job);
        if (shouldCollect) {
            collected = true;
            await this.storageStrategy.add(buffer.id, job);  // 存储到缓冲存储
        }
    }
    return collected;
}
```

**存储策略**：
- 默认 `InMemoryJobBufferStorageStrategy` - 内存存储
- 可配置持久化存储策略

### 3.4 刷新执行阶段

**手动触发**（`fulltext-search.resolver.ts:104-110`）：

```typescript
@Mutation()
@Allow(Permission.UpdateCatalog, Permission.UpdateProduct)
async runPendingSearchIndexUpdates(...args: any[]): Promise<any> {
    void this.searchJobBufferService.runPendingSearchUpdates();
    return { success: true };
}
```

**Dashboard 提醒**（`search-index-buffer-alert.ts:21-41`）：
- 每分钟轮询 `pendingSearchIndexUpdates` 查询
- 挂起数 > 0 时显示警告
- 提供 "Run pending updates" 操作按钮

**执行顺序**（`search-job-buffer.service.ts:45-67`）：

```typescript
async runPendingSearchUpdates(): Promise<void> {
    // 1. 先刷新 collection buffer（确保集合过滤器先应用）
    const collectionFilterJobs = await this.jobQueueService.flush(this.collectionJobBuffer);
    
    // 2. 等待 collection jobs 完成（最长 15 分钟）
    if (collectionFilterJobs.length && isInspectableJobQueueStrategy(jobQueueStrategy)) {
        await forkJoin(...subscribableCollectionJobs.map(sj => 
            sj.updates({ pollInterval: 500, timeoutMs: 15 * 60 * 1000 })
        )).toPromise();
    }
    
    // 3. 再刷新 search index buffer
    await this.jobQueueService.flush(this.searchIndexJobBuffer);
}
```

**合并优化**（`search-index-job-buffer.ts:23-67`）：

```typescript
reduce(collectedJobs: Array<Job<UpdateIndexQueueJobData>>): Array<Job<any>> {
    // 1. 分离 variants jobs 和 products jobs
    const variantsJobs = this.removeBy(collectedJobs, 
        item => item.data.type === 'update-variants-by-id' || item.data.type === 'update-variants');
    const productsJobs = this.removeBy(collectedJobs, 
        item => item.data.type === 'update-product');
    
    const jobsToAdd = [...collectedJobs];
    
    // 2. 合并所有 variants jobs 为一个 update-variants-by-id job
    if (variantsJobs.length) {
        const variantIdsToUpdate: ID[] = [];
        for (const job of variantsJobs) {
            const ids = job.data.type === 'update-variants-by-id' ? job.data.ids : job.data.variantIds;
            variantIdsToUpdate.push(...ids);
        }
        // 去重后创建合并任务
        const batchedVariantJob = new Job<UpdateVariantsByIdJobData>({
            ...referenceJob,
            data: {
                type: 'update-variants-by-id',
                ids: unique(variantIdsToUpdate),  // 关键优化：去重
                ctx: referenceJob.data.ctx,
            },
        });
        jobsToAdd.push(batchedVariantJob as Job);
    }
    
    // 3. 合并 products jobs（按 productId 去重）
    if (productsJobs.length) {
        const seenIds = new Set<ID>();
        const uniqueProductJobs: Array<Job<UpdateProductJobData>> = [];
        for (const job of productsJobs) {
            if (!seenIds.has(job.data.productId)) {
                uniqueProductJobs.push(job);
                seenIds.add(job.data.productId);
            }
        }
        jobsToAdd.push(...(uniqueProductJobs as Job[]));
    }
    
    return jobsToAdd;
}
```

### 3.5 对可见性一致性的影响

| 影响维度 | 说明 |
|---------|------|
| **最终一致性** | 缓冲期间索引与实际数据不一致，用户搜索可能看到过期信息 |
| **延迟窗口** | 延迟取决于何时调用 `runPendingSearchIndexUpdates`，可能是分钟级到小时级 |
| **删除操作例外** | 删除操作不缓冲，立即执行，避免已删除商品出现在搜索结果中 |
| **频道分配例外** | 频道分配变更不缓冲，立即执行，确保多渠道可见性及时更新 |
| **批量优化代价** | 合并优化减少了任务数量，但可能导致单个任务执行时间变长 |
| **脏读风险** | 缓冲期间多次更新同一实体，最后一次 flush 时只执行一次最新的索引更新 |

---

## 4. 频道可见性的协作模式

### 4.1 索引构建阶段

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

### 4.2 频道分配时的语言联动

**触发场景**：商品/变体分配到新频道时

**语言联动逻辑**（`indexer.controller.ts:252-291`）：

```typescript
private async updateProductInChannel(
    ctx: MutableRequestContext,
    productId: ID,
    channelId: ID,
): Promise<boolean> {
    const channel = await this.loadChannel(ctx, channelId);
    ctx.setChannel(channel);
    const product = await this.getProductInChannelQueryBuilder(ctx, productId, channel);

    if (product) {
        // 关键：查询支持该商品翻译语言的所有频道
        const affectedChannels = await this.getAllChannels(ctx, {
            where: {
                availableLanguageCodes: In(product.translations.map(t => t.languageCode)),
            },
        });
        
        // 为所有相关频道更新索引
        const { variants: updatedVariants } = await this.getSearchIndexQueryBuilder(ctx, {
            channels: unique(affectedChannels.concat(channel)),
            productId,
        });
        
        // 无变体兜底路径
        if (updatedVariants.length === 0) {
            const clone = new Product({ id: product.id });
            await this.entityHydrator.hydrate(ctx, clone, { relations: ['translations' as never] });
            product.translations = clone.translations;
            await this.saveSyntheticVariant(ctx, product);
        }
        // ...
    }
}
```

**语言降级策略**（`indexer.controller.ts:568-575`）：

```typescript
private getTranslation<T extends Translatable>(
    translatable: T,
    languageCode: LanguageCode,
): Translation<T> {
    return (translatable.translations.find(t => t.languageCode === languageCode) ||
        translatable.translations.find(t => t.languageCode === this.configService.defaultLanguageCode) ||
        translatable.translations[0]) as unknown as Translation<T>;
}
```

**channelIds 字段的联动**（`indexer.controller.ts:425-435`）：

```typescript
// 收集该变体在所有支持当前语言的频道中的 ID
let channelIds = variant.channels.map(x => x.id);
const clone = new ProductVariant({ id: variant.id });
await this.entityHydrator.hydrate(ctx, clone, {
    relations: ['channels', 'channels.defaultTaxZone'],
});
channelIds.push(
    ...clone.channels
        .filter(x => x.availableLanguageCodes.includes(languageCode))
        .map(x => x.id),
);
channelIds = unique(channelIds);
```

### 4.3 无变体兜底路径

**触发条件**（`indexer.controller.ts:271-275`）：

```typescript
if (updatedVariants.length === 0) {
    // 商品已分配到频道，但还没有任何变体
    const clone = new Product({ id: product.id });
    await this.entityHydrator.hydrate(ctx, clone, { relations: ['translations' as never] });
    product.translations = clone.translations;
    await this.saveSyntheticVariant(ctx, product);
}
```

**合成变体特征**（`indexer.controller.ts:521-550`）：

```typescript
private async saveSyntheticVariant(ctx: RequestContext, product: Product) {
    const productTranslation = this.getTranslation(product, ctx.languageCode);
    const item = new SearchIndexItem({
        channelId: ctx.channelId,
        currencyCode: ctx.currencyCode,
        languageCode: ctx.languageCode,
        productVariantId: 0,  // 特殊标识：productVariantId = 0
        price: 0,
        priceWithTax: 0,
        sku: '',
        enabled: false,       // 关键：合成变体默认不启用
        slug: productTranslation.slug,
        productId: product.id,
        productName: productTranslation.name,
        description: this.constrainDescription(productTranslation.description),
        productVariantName: productTranslation.name,
        // ...
    });
    await this.queue.push(() => this.connection.getRepository(ctx, SearchIndexItem).save(item));
}
```

**合成变体清理**（`indexer.controller.ts:404, 555-566`）：

```typescript
// 保存真实变体前先清理合成变体
private async saveVariants(ctx: MutableRequestContext, variants: ProductVariant[]) {
    const items: SearchIndexItem[] = [];
    await this.removeSyntheticVariants(ctx, variants);
    // ...
}

private async removeSyntheticVariants(ctx: RequestContext, variants: ProductVariant[]) {
    const prodIds = unique(variants.map(v => v.productId));
    for (const productId of prodIds) {
        await this.queue.push(() =>
            this.connection.getRepository(ctx, SearchIndexItem).delete({
                productId,
                sku: '',    // 通过 sku = '' 和 price = 0 识别合成变体
                price: 0,
            }),
        );
    }
}
```

**兜底路径完整生命周期**：
```
1. 商品分配到频道 → updateProductInChannel
2. 检查是否有变体 → updatedVariants.length === 0
3. 创建合成变体 → productVariantId = 0, enabled = false
4. 后续添加变体 → updateVariants
5. saveVariants 执行前 → removeSyntheticVariants
6. 删除合成变体，保存真实变体
```

### 4.4 事件驱动更新

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

**重要**：频道分配变更**不经过缓冲区**，立即执行。

### 4.5 搜索查询过滤

所有搜索策略在查询时强制应用频道过滤（`postgres-search-strategy.ts:301`）：

```typescript
private applyTermAndFilters(ctx: RequestContext, qb: SelectQueryBuilder<SearchIndexItem>, input: SearchInput) {
    // 强制过滤当前频道
    qb.andWhere('si.channelId = :channelId', { channelId: ctx.channelId });
    // ...
}
```

---

## 5. 客户分组可见性的协作模式

### 5.1 核心发现

**索引层不直接支持客户分组可见性**。`SearchIndexItem` 实体中没有 `customerGroupId` 字段，客户分组需要在查询阶段通过扩展实现。

### 5.2 RequestContext 实际可用字段（已有实现）

`RequestContext` 类（`request-context.ts:179-484`）中与客户相关的可用字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `ctx.activeUserId` | `ID \| undefined` | **已有实现**。返回 `session?.user?.id`，是获取当前用户的唯一入口 |
| `ctx.session` | `CachedSession \| undefined` | **已有实现**。会话对象，包含用户信息 |
| `ctx.session?.user` | `CachedSessionUser \| undefined` | **已有实现**。只包含 `id`, `identifier`, `verified`, `channelPermissions` |
| `ctx.activeCustomer` | - | **不存在！** 之前的理解有误，RequestContext 中没有此字段 |

**正确获取客户分组的链路（已有实现）**：
```
ctx.activeUserId → userId
    ↓
customerService.findOneByUserId(ctx, userId) → Customer
    ↓
customerService.getCustomerGroups(ctx, customerId) → CustomerGroup[]
```

### 5.3 已有服务方法（仓库现有实现）

#### 5.3.1 CustomerService.findOneByUserId

**已有实现**（`customer.service.ts:141-153`）：

```typescript
/**
 * Returns the Customer entity associated with the given userId, if one exists.
 * Setting `filterOnChannel` to `true` will limit the results to Customers which are assigned
 * to the current active Channel only.
 */
findOneByUserId(
    ctx: RequestContext, 
    userId: ID, 
    filterOnChannel = true
): Promise<Customer | undefined> {
    let query = this.connection
        .getRepository(ctx, Customer)
        .createQueryBuilder('customer')
        .leftJoin('customer.channels', 'channel')
        .leftJoinAndSelect('customer.user', 'user')
        .where('user.id = :userId', { userId })
        .andWhere('customer.deletedAt is null');
    if (filterOnChannel) {
        query = query.andWhere('channel.id = :channelId', { channelId: ctx.channelId });
    }
    return query.getOne().then(result => result ?? undefined);
}
```

#### 5.3.2 CustomerService.getCustomerGroups

**已有实现**（`customer.service.ts:179-196`）：

```typescript
/**
 * Returns a list of all {@link CustomerGroup} entities.
 */
async getCustomerGroups(ctx: RequestContext, customerId: ID): Promise<CustomerGroup[]> {
    const customerWithGroups = await this.connection.findOneInChannel(
        ctx,
        Customer,
        customerId,
        ctx?.channelId,
        {
            relations: ['groups'],
            where: { deletedAt: IsNull() },
        },
    );
    if (customerWithGroups) {
        return customerWithGroups.groups;
    } else {
        return [];
    }
}
```

#### 5.3.3 CustomerGroupChangeEvent

**已有实现**（`customer-group.service.ts:168, 194`）：

```typescript
// 客户添加到分组时触发
this.eventBus.publish(new CustomerGroupChangeEvent(ctx, customers, group, 'assigned'));

// 客户移出分组时触发
this.eventBus.publish(new CustomerGroupChangeEvent(ctx, customers, group, 'removed'));
```

### 5.4 价格选择策略（已有实现）

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

### 5.5 索引构建时的价格计算（已有实现）

在 `saveVariants` 中应用价格（`indexer.controller.ts:446`）：

```typescript
await this.productPriceApplicator.applyChannelPriceAndTax(variant, ctx);
const item = new SearchIndexItem({
    price: variant.price,
    priceWithTax: variant.priceWithTax,
    // ...
});
```

`applyChannelPriceAndTax` 内部调用价格选择策略（`product-price-applicator.ts:58-108`）：

```typescript
async applyChannelPriceAndTax(variant: ProductVariant, ctx: RequestContext, order?: Order) {
    const { productVariantPriceSelectionStrategy, productVariantPriceCalculationStrategy } =
        this.configService.catalogOptions;
    
    // 调用价格选择策略
    const channelPrice = await productVariantPriceSelectionStrategy.selectPrice(
        ctx,
        variant.productVariantPrices,
    );
    
    // ... 计算税额
    
    variant.listPrice = price;
    variant.listPriceIncludesTax = priceIncludesTax;
    variant.taxRateApplied = applicableTaxRate;
    variant.currencyCode = channelPrice?.currencyCode ?? ctx.currencyCode;
    return variant;
}
```

### 5.6 能力边界

**索引层限制**：
- `SearchIndexItem` 无 `customerGroupId` 字段
- `ProductVariantPrice` 实体也没有 `customerGroupId` 字段
- 索引中只会存储一个价格（基于索引构建时的上下文）
- 无法为每个客户分组存储不同价格
- 无客户分组变更事件触发索引更新（`CustomerGroupChangeEvent` 存在但未被搜索插件订阅）

**查询层限制**：
- `SearchInput` 无客户分组过滤参数
- 搜索策略不支持按客户分组过滤

### 5.7 查询阶段的可扩展策略（均为扩展示例）

#### 扩展点 1：自定义 SearchStrategy（可选扩展）

通过自定义 `SearchStrategy` 在查询时添加客户分组逻辑。需要注入 `CustomerService`：

```typescript
// 【可选扩展】自定义搜索策略
@Injectable()
export class CustomerGroupSearchStrategy extends PostgresSearchStrategy {
    constructor(private customerService: CustomerService) {
        super();
    }

    async getSearchResults(ctx: RequestContext, input: SearchInput, enabledOnly: boolean) {
        const results = await super.getSearchResults(ctx, input, enabledOnly);
        
        // 查询后过滤：根据当前用户的客户分组过滤结果
        const customerGroups = await this.getCurrentCustomerGroups(ctx);
        if (customerGroups.length > 0) {
            // 应用客户分组级别的可见性规则
            return this.applyCustomerGroupVisibility(results, customerGroups);
        }
        
        return results;
    }

    // 【已有实现调用】获取当前客户的分组
    private async getCurrentCustomerGroups(ctx: RequestContext): Promise<CustomerGroup[]> {
        const userId = ctx.activeUserId;
        if (!userId) return [];
        
        const customer = await this.customerService.findOneByUserId(ctx, userId);
        if (!customer) return [];
        
        return this.customerService.getCustomerGroups(ctx, customer.id);
    }

    private applyCustomerGroupVisibility(
        results: SearchResult[], 
        customerGroups: CustomerGroup[]
    ): SearchResult[] {
        // 自定义可见性逻辑
        return results.filter(item => 
            this.isVisibleToGroups(item, customerGroups)
        );
    }

    private isVisibleToGroups(item: SearchResult, groups: CustomerGroup[]): boolean {
        // 示例：检查商品 customFields 中的客户分组白名单
        const allowedGroupIds = item.product?.customFields?.allowedCustomerGroupIds || [];
        if (allowedGroupIds.length === 0) return true;
        return groups.some(g => allowedGroupIds.includes(g.id.toString()));
    }
}
```

#### 扩展点 2：自定义 ProductVariantPriceSelectionStrategy（可选扩展）

通过价格选择策略实现客户分组差异化定价。需要注入 `CustomerService`：

```typescript
// 【可选扩展】自定义价格选择策略
@Injectable()
export class CustomerGroupPriceSelectionStrategy implements ProductVariantPriceSelectionStrategy {
    constructor(private customerService: CustomerService) {}

    async selectPrice(ctx: RequestContext, prices: ProductVariantPrice[]) {
        const pricesInChannel = prices.filter(p => idsAreEqual(p.channelId, ctx.channelId));
        const priceInCurrency = pricesInChannel.find(p => p.currencyCode === ctx.currencyCode);
        
        // 【已有实现调用】获取当前客户的分组
        const customerGroups = await this.getCurrentCustomerGroups(ctx);
        if (customerGroups.length > 0) {
            // 从 customFields 或其他来源获取客户分组特定价格
            const customerGroupPrice = this.getCustomerGroupPrice(
                pricesInChannel, 
                customerGroups[0]
            );
            return customerGroupPrice ?? priceInCurrency;
        }
        
        return priceInCurrency;
    }

    private async getCurrentCustomerGroups(ctx: RequestContext): Promise<CustomerGroup[]> {
        const userId = ctx.activeUserId;
        if (!userId) return [];
        
        const customer = await this.customerService.findOneByUserId(ctx, userId);
        if (!customer) return [];
        
        return this.customerService.getCustomerGroups(ctx, customer.id);
    }

    private getCustomerGroupPrice(
        prices: ProductVariantPrice[], 
        group: CustomerGroup
    ): ProductVariantPrice | undefined {
        // 示例：从 customFields 中查找客户分组特定价格
        return prices.find(p => 
            p.customFields?.customerGroupId === group.id.toString()
        );
    }
}
```

#### 扩展点 3：Resolver 层后处理（可选扩展）

在 GraphQL Resolver 层对搜索结果进行客户分组过滤。需要注入 `CustomerService`：

```typescript
// 【可选扩展】自定义解析器
@Resolver('SearchResponse')
export class CustomShopSearchResolver {
    constructor(
        private fulltextSearchService: FulltextSearchService,
        private customerService: CustomerService
    ) {}

    @Query()
    @Allow(Permission.Public)
    async search(
        @Ctx() ctx: RequestContext,
        @Args() args: QuerySearchArgs,
    ): Promise<Omit<SearchResponse, 'facetValues' | 'collections'>> {
        const result = await this.fulltextSearchService.search(ctx, args.input, true);
        
        // 【已有实现调用】获取当前客户的分组
        const customerGroups = await this.getCurrentCustomerGroups(ctx);
        if (customerGroups.length > 0) {
            // 应用客户分组可见性过滤
            result.items = result.items.filter(item => 
                this.isVisibleToCustomerGroup(item, customerGroups)
            );
            result.totalItems = result.items.length;
        }
        
        (result as any).input = args.input;
        return result;
    }

    private async getCurrentCustomerGroups(ctx: RequestContext): Promise<CustomerGroup[]> {
        const userId = ctx.activeUserId;
        if (!userId) return [];
        
        const customer = await this.customerService.findOneByUserId(ctx, userId);
        if (!customer) return [];
        
        return this.customerService.getCustomerGroups(ctx, customer.id);
    }

    private isVisibleToCustomerGroup(
        item: SearchResult, 
        groups: CustomerGroup[]
    ): boolean {
        // 自定义可见性逻辑
        return true;
    }
}
```

#### 扩展点 4：订阅 CustomerGroupChangeEvent（可选扩展）

如果客户分组变更需要触发重新索引，可以在插件中订阅该事件：

```typescript
// 【可选扩展】在自定义插件中订阅客户分组变更事件
export class MySearchPlugin {
    constructor(
        private eventBus: EventBus,
        private searchIndexService: SearchIndexService,
        private customerService: CustomerService
    ) {}

    onModuleInit() {
        // 订阅客户分组变更事件
        this.eventBus.ofType(CustomerGroupChangeEvent).subscribe(async event => {
            // 获取受影响的客户
            const customerIds = event.customers.map(c => c.id);
            
            // 查询这些客户可能影响的商品（需要额外逻辑）
            // ...
            
            // 触发索引更新
            // await this.searchIndexService.updateVariants(ctx, affectedVariants);
        });
    }
}
```

#### 扩展点 5：扩展 SearchInput（可选扩展）

通过插件扩展 GraphQL schema，添加客户分组过滤参数：

```graphql
# 【可选扩展】GraphQL schema 扩展
extend input SearchInput {
    customerGroupId: ID
}
```

然后在自定义 SearchStrategy 中处理该参数。

### 5.8 完整调用链路（已有实现）

搜索请求的完整解析链路：

```
ShopFulltextSearchResolver.search(ctx, input)  ← ctx 中只有 activeUserId，无客户信息
    ↓
FulltextSearchService.search(ctx, input, enabledOnly)
    ↓
SearchStrategy.getSearchResults(ctx, input, enabledOnly)  ← 可注入 CustomerService 查询客户信息
    ↓
PostgresSearchStrategy.applyTermAndFilters(...)  ← 无客户分组过滤
    ↓
数据库查询 SearchIndexItem
```

**关键点**：各层均只能访问 `RequestContext`，客户分组信息需要通过 `CustomerService` 主动查询获取。

---

## 6. 库存状态可见性的协作模式

### 6.1 可选配置启用

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

### 6.2 索引构建阶段

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

### 6.3 事件驱动更新

库存变动时触发重新索引（`default-search-plugin.ts:173-178`）：

```typescript
this.eventBus.ofType(StockMovementEvent).subscribe(event => {
    return this.searchIndexService.updateVariants(
        event.ctx,
        event.stockMovements.map(m => m.productVariant),
    );
});
```

### 6.4 搜索查询过滤

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

## 7. Enabled 状态的协作模式

### 7.1 索引构建阶段

商品的 `enabled` 优先级高于变体的 `enabled`（`indexer.controller.ts:455`）：

```typescript
const item = new SearchIndexItem({
    enabled: product.enabled === false ? false : variant.enabled,
    // ...
});
```

### 7.2 搜索查询过滤

通过 `enabledOnly` 参数控制（`postgres-search-strategy.ts:118-120`）：

```typescript
if (enabledOnly) {
    qb.andWhere('"si"."enabled" = :enabled', { enabled: true });
}
```

Shop API 调用时 `enabledOnly = true`，Admin API 调用时 `enabledOnly = false`。

---

## 8. 协作模式总结

| 维度 | 索引层处理 | 事件驱动 | 搜索过滤 | 缓冲更新 | 备注 |
|------|-----------|----------|----------|----------|------|
| **频道** | ✅ 主键隔离，多频道索引 | ✅ `ProductChannelEvent`<br>`ProductVariantChannelEvent` | ✅ 强制过滤 `channelId` | ❌ 不缓冲 | 完全在索引层处理，语言联动 |
| **客户分组** | ❌ 无直接支持 | ⚠️ `CustomerGroupChangeEvent`<br>存在但未被搜索插件订阅 | ❌ 无索引过滤 | ❌ 不涉及 | 需通过 `CustomerService` 查询客户信息，在查询阶段扩展实现 |
| **库存状态** | ✅ 可选索引 `inStock`<br>`productInStock` | ✅ `StockMovementEvent` | ✅ 可选过滤 `input.inStock` | ✅ 可缓冲 | 需显式配置启用 |
| **Enabled** | ✅ 索引存储 `enabled` 字段 | ✅ `ProductEvent`<br>`ProductVariantEvent` | ✅ `enabledOnly` 参数 | ✅ 可缓冲 | 商品优先级 > 变体优先级 |

---

## 9. 关键代码参考

### 9.1 索引更新队列任务类型

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

### 9.2 可缓冲 vs 不可缓冲任务

**可缓冲**（经过 `SearchIndexJobBuffer`）：
- `update-product`
- `update-variants`
- `update-variants-by-id`

**不可缓冲**（立即执行）：
- `reindex`
- `delete-product` / `delete-variant`
- `update-asset` / `delete-asset`
- `assign-product-to-channel` / `remove-product-from-channel`
- `assign-variant-to-channel` / `remove-variant-from-channel`

### 9.3 合成变体处理

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

**识别合成变体的条件**：
- `productVariantId = 0`
- `sku = ''`
- `price = 0`

### 9.4 语言降级策略

翻译查找优先级（`indexer.controller.ts:568-575`）：
1. 指定语言
2. 系统默认语言
3. 第一个可用翻译

### 9.5 客户分组相关已有方法

#### 获取客户分组信息的标准链路（已有实现）：

```
ctx.activeUserId → userId
    ↓
customerService.findOneByUserId(ctx, userId) → Customer
    ↓
customerService.getCustomerGroups(ctx, customerId) → CustomerGroup[]
```

**关键方法**（已有实现）：
- `CustomerService.findOneByUserId(ctx, userId, filterOnChannel?)` - `customer.service.ts:141-153`
- `CustomerService.getCustomerGroups(ctx, customerId)` - `customer.service.ts:179-196`
- `CustomerGroupChangeEvent` - 客户分组变更事件 - `customer-group.service.ts:168, 194`

#### RequestContext 可用字段（已有实现）：

| 字段 | 可用性 |
|------|--------|
| `ctx.activeUserId` | ✅ 已有 |
| `ctx.session?.user` | ✅ 已有（只有 `id`, `identifier`, `verified`, `channelPermissions`） |
| `ctx.activeCustomer` | ❌ 不存在 |

#### 完整搜索调用链路（已有实现）：

```
ShopFulltextSearchResolver.search(ctx, input)
    ↓
FulltextSearchService.search(ctx, input, enabledOnly)
    ↓
SearchStrategy.getSearchResults(ctx, input, enabledOnly)
    ↓
PostgresSearchStrategy.applyTermAndFilters(...)
    ↓
数据库查询 SearchIndexItem
```

**关键点**：各层只能访问 `RequestContext`，客户分组信息需要通过 `CustomerService` 主动查询获取。

### 9.6 缓冲刷新顺序

`search-job-buffer.service.ts:45-67`：
1. 刷新 CollectionJobBuffer（应用集合过滤器）
2. 等待集合任务完成
3. 刷新 SearchIndexJobBuffer（更新搜索索引）
