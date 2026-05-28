# Facet 与 Collection 自动分组规则运转机制

## 一、核心实体关系

```
Facet ──1:N──> FacetValue <──M:N──> Product
                     │
                     └────M:N──> ProductVariant
                                    │
                     Collection <──M:N──┘
                         │
                     filters: ConfigurableOperation[]  (simple-json 列)
                     inheritFilters: boolean
```

- `FacetValue` 同时与 `Product` 和 `ProductVariant` 建立多对多关系
  - `product.entity.ts:67-69`：Product ↔ FacetValue
  - `product-variant.entity.ts:167-169`：ProductVariant ↔ FacetValue
- `Collection` 与 `ProductVariant` 通过 `productVariants` 多对多关系关联
  - `collection.entity.ts:77-79`

---

## 二、Filter 规则的存储与解析

### 2.1 存储格式

Collection 的 `filters` 字段以 `simple-json` 列存储在数据库中（`collection.entity.ts:70`）：

```ts
@Column('simple-json') filters: ConfigurableOperation[];
```

每个 `ConfigurableOperation` 的结构为：

```ts
{
  code: string;          // 如 'facet-value-filter'
  args: ConfigArg[];     // [{ name: 'facetValueIds', type: 'ID', value: '["1","2"]' }, ...]
}
```

### 2.2 写入流程

当通过 Admin API 创建或更新 Collection 时，`CollectionService` 调用 `getCollectionFiltersFromInput()`（`collection.service.ts:704-713`）：

```ts
private getCollectionFiltersFromInput(input): ConfigurableOperation[] {
    const filters: ConfigurableOperation[] = [];
    if (input.filters) {
        for (const filterInput of input.filters) {
            filters.push(this.configArgService.parseInput('CollectionFilter', filterInput));
        }
    }
    return filters;
}
```

`ConfigArgService.parseInput()`（`config-arg.service.ts:72-80`）负责：
1. 根据 `input.code` 查找匹配的 `CollectionFilter` 定义
2. 校验必填字段（`validateRequiredFields`）
3. 按 `CollectionFilter.args` 定义的顺序重排参数（`orderArgsToMatchDef`）
4. 返回规范化的 `ConfigurableOperation` 对象

### 2.3 可用 Filter 定义注册

通过 `CatalogOptions.collectionFilters` 配置注入（`vendure-config.ts:770`）：

```ts
export interface CatalogOptions {
    collectionFilters?: Array<CollectionFilter<any>>;
}
```

默认注册了四个 Filter（`default-collection-filters.ts:267-271`）：

| code | 类 | 说明 |
|---|---|---|
| `facet-value-filter` | `facetValueCollectionFilter` | 按 FacetValue 筛选 |
| `variant-name-filter` | `variantNameCollectionFilter` | 按变体名称匹配 |
| `variant-id-filter` | `variantIdCollectionFilter` | 手动选择变体 |
| `product-id-filter` | `productIdCollectionFilter` | 手动选择商品 |

---

## 三、Facet Value Filter 的核心 SQL 逻辑

`facetValueCollectionFilter`（`default-collection-filters.ts:46-128`）的 `apply()` 函数是最关键的自动分组实现：

### 3.1 查询结构

```sql
-- 子查询1: 变体自身的 facetValue
SELECT product_variant.id AS variant_id, facet_value.id AS facet_value_id
FROM product_variant
LEFT JOIN facet_value ON ... WHERE facet_value.id IN (:...ids)

UNION

-- 子查询2: 变体所属 Product 的 facetValue
SELECT product_variant.id AS variant_id, facet_value.id AS facet_value_id
FROM product_variant
LEFT JOIN product ON ...
LEFT JOIN facet_value ON ... WHERE facet_value.id IN (:...ids)

-- 外层: 按 variant_id 分组, HAVING COUNT >= :count
SELECT variant_id FROM (UNION结果)
GROUP BY variant_id
HAVING COUNT(*) >= :count
```

### 3.2 `containsAny` 参数的作用

- `containsAny = true` → `count = 1` → 满足任一 FacetValue 即命中
- `containsAny = false` → `count = facetValueIds.length` → 必须满足所有 FacetValue

### 3.3 `combineWithAnd` 参数

控制多 Filter 间的组合方式：
- `combineWithAnd !== false` → 使用 `qb.andWhere()`（AND 组合）
- `combineWithAnd === false` → 使用 `qb.orWhere()`（OR 组合）

---

## 四、Filter 继承机制

Collection 以闭包表（closure-table）组织为树形结构，支持 Filter 向下继承：

```
Collection (filters: [A])
  └── Collection (inheritFilters: true, filters: [B])
        └── Collection (inheritFilters: false, filters: [C])
```

`getAncestorFilters()`（`collection.service.ts:843-855`）的遍历逻辑：

1. 如果当前 Collection 的 `inheritFilters = true`，则向上遍历祖先
2. 收集每个祖先的 `filters`
3. 一旦遇到 `inheritFilters = false` 的祖先，立即停止遍历并返回已收集的 filters
4. 最终在 `applyCollectionFiltersInternal()` 中合并：`[...ancestorFilters, ...collection.filters]`

---

## 五、商品变更触发重归组

### 5.1 事件监听

`CollectionService.onModuleInit()`（`collection.service.ts:104-126`）订阅事件：

```ts
merge(productEvents$, variantEvents$)
    .pipe(
        filter(() => this.applyAllFiltersOnProductUpdates),
        debounceTime(50),
    )
    .subscribe(async event => {
        await this.triggerApplyFiltersJob(event.ctx);
    });
```

关键行为：
- **合并监听** `ProductEvent` 和 `ProductVariantEvent`
- **防抖 50ms**：批量更新商品时，多个事件在 50ms 内只会触发一次归组
- **门控开关**：`applyAllFiltersOnProductUpdates` 默认为 `true`

### 5.2 触发流程

`triggerApplyFiltersJob()`（`collection.service.ts:687-702`）向 `apply-collection-filters` 队列添加 Job：

```ts
async triggerApplyFiltersJob(ctx, options?) {
    await this.applyFiltersQueue.add({
        ctx: { languageCode, channelToken },
        applyToChangedVariantsOnly: options?.applyToChangedVariantsOnly,
        collectionIds: options?.collectionIds ?? [],
    }, { ctx });
}
```

不同场景的触发参数差异：

| 触发场景 | collectionIds | applyToChangedVariantsOnly |
|---|---|---|
| 商品/变体变更事件 | `[]`（全部 Collection） | `undefined`（默认 true） |
| 创建 Collection | `[新ID]` | `undefined`（默认 true） |
| 更新 Collection 的 filters | `[当前ID]` | `false` |
| 移动 Collection | `[当前ID]` | `undefined` |

### 5.3 Job 处理逻辑

Job 处理函数（`collection.service.ts:128-190`）：

1. 如果 `collectionIds` 为空，查询所有 Collection 的 ID
2. 逐个 Collection 调用 `applyCollectionFiltersInternal()`
3. 每个 Collection 处理完后更新进度（`job.setProgress`）
4. 如果有受影响变体，发布 `CollectionModificationEvent`（分块 50000）

### 5.4 核心归组算法

`applyCollectionFiltersInternal()`（`collection.service.ts:728-837`）使用 CTE 差量计算：

```
_filtered_variants  := 所有匹配当前 Collection filter 规则的变体 ID
_existing_variants  := 当前已属于该 Collection 的变体 ID

toAdd    := _filtered_variants LEFT JOIN _existing_variants WHERE existing IS NULL
toRemove := _existing_variants LEFT JOIN _filtered_variants WHERE filtered IS NULL
```

然后事务性地执行：
- 批量移除（`chunkArray(toRemoveIds, 5000)`）
- 批量添加（`chunkArray(toAddIds, 5000)`）

### 5.5 affectedVariantIds 的两种返回模式

```ts
if (applyToChangedVariantsOnly) {
    return [...toAddIds, ...toRemoveIds];   // 仅变更部分
}
return [...existingIds, ...toRemoveIds];     // 全量（包含未变更的存量）
```

当 `applyToChangedVariantsOnly = false`（如 filter 规则变更时），返回的 `affectedVariantIds` 包含所有现有成员 + 移除的，确保 `CollectionModificationEvent` 的下游（如搜索索引重建）能感知全量影响。

---

## 六、增量重建策略切换

### 6.1 全局开关

`setApplyAllFiltersOnProductUpdates()`（`collection.service.ts:674-676`）：

```ts
setApplyAllFiltersOnProductUpdates(applyAllFiltersOnProductUpdates: boolean) {
    this.applyAllFiltersOnProductUpdates = applyAllFiltersOnProductUpdates;
}
```

设计意图（注释 `collection.service.ts:661-672`）：大批量导入时，每次商品变更都触发全量归组代价太高，可以先 `setApplyAllFiltersOnProductUpdates(false)` 暂停自动归组，导入完成后手动调用 `triggerApplyFiltersJob()`。

### 6.2 JobBuffer 批量合并

`CollectionJobBuffer`（`collection-job-buffer.ts`）在搜索插件中实现了 Job 的缓冲与合并：

```ts
collect(job): boolean {
    return job.queueName === 'apply-collection-filters';
}

reduce(collectedJobs): Array<Job> {
    const collectionIdsToUpdate = collectedJobs.reduce(
        (result, job) => [...result, ...job.data.collectionIds], []
    );
    // 合并所有 collectionIds，去重后生成单个 Job
    return [new Job({
        ...referenceJob,
        data: {
            collectionIds: unique(collectionIdsToUpdate),
            ctx: referenceJob.data.ctx,
            applyToChangedVariantsOnly: referenceJob.data.applyToChangedVariantsOnly,
        },
    })];
}
```

关键行为：
- **collect**：拦截所有 `apply-collection-filters` 队列的 Job
- **reduce**：将多个 Job 的 `collectionIds` 合并去重，生成单个批处理 Job

### 6.3 缓冲的激活时机

`SearchJobBufferService`（`search-job-buffer.service.ts:25-29`）在应用启动时根据配置激活：

```ts
onApplicationBootstrap() {
    if (this.bufferUpdates === true) {
        this.jobQueueService.addBuffer(this.searchIndexJobBuffer);
        this.jobQueueService.addBuffer(this.collectionJobBuffer);
    }
}
```

`bufferUpdates` 由 `DefaultSearchPlugin` 的 `BUFFER_SEARCH_INDEX_UPDATES` 注入标记控制。

### 6.4 缓冲的 flush 流程

`JobQueueService.flush()` → `JobBufferService.flush()`：
1. 从缓冲存储中取出所有收集的 Job
2. 传入 `JobBuffer.reduce()` 进行合并
3. 将合并后的 Job 提交到实际队列执行

---

## 七、完整数据流图

```
Product/Variant 变更
        │
        ▼
  ProductEvent / ProductVariantEvent
        │
        ▼
  debounceTime(50ms) ──── applyAllFiltersOnProductUpdates = false? ── 跳过
        │
        ▼
  triggerApplyFiltersJob(ctx) ── collectionIds: [] (全量)
        │
        ▼
  JobQueue.add() ──► CollectionJobBuffer.collect() ── 缓冲
        │                                         │
        │                                  flush 时 reduce()
        │                                         │
        ▼                                         ▼
  apply-collection-filters Job 执行         合并后的单个 Job
        │
        ▼
  遍历所有 Collection
        │
        ▼
  applyCollectionFiltersInternal(collection):
    1. getAncestorFilters() 收集继承的 filters
    2. 合并 [ancestorFilters + collection.filters]
    3. 构建 filteredQb → 匹配规则的变体 ID
    4. 构建 existingVariantsQb → 已在 Collection 中的变体 ID
    5. CTE 差量: toAdd / toRemove
    6. 事务执行: 批量 add/remove (chunk=5000)
        │
        ▼
  CollectionModificationEvent(ctx, collection, affectedVariantIds)
        │
        ▼
  搜索索引更新 / 其他下游处理
```

---

## 八、关键文件索引

| 文件 | 核心职责 |
|---|---|
| `core/src/config/catalog/collection-filter.ts` | CollectionFilter 类定义，`apply()` 签名 |
| `core/src/config/catalog/default-collection-filters.ts` | 四个内置 Filter 实现，含 facet-value-filter 的 SQL 逻辑 |
| `core/src/config/vendure-config.ts:763-770` | CatalogOptions.collectionFilters 注册入口 |
| `core/src/entity/collection/collection.entity.ts:70` | filters 以 `simple-json` 存储 |
| `core/src/entity/collection/collection.entity.ts:75` | inheritFilters 字段 |
| `core/src/service/services/collection.service.ts:104-126` | 事件监听 + 防抖触发 |
| `core/src/service/services/collection.service.ts:728-837` | 核心归组算法（CTE 差量） |
| `core/src/service/services/collection.service.ts:674-676` | 全局开关 setApplyAllFiltersOnProductUpdates |
| `core/src/service/services/collection.service.ts:687-702` | triggerApplyFiltersJob 入口 |
| `core/src/plugin/default-search-plugin/search-job-buffer/collection-job-buffer.ts` | Job 缓冲与合并 |
| `core/src/job-queue/job-buffer/job-buffer.ts` | JobBuffer 接口定义 |
| `core/src/service/helpers/config-arg/config-arg.service.ts:72-80` | parseInput 将 API 输入转为存储格式 |
