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

## 三、GraphQL 侧 ID 编解码链路

### 3.1 问题根源

`ConfigurableOperationInput` 中，所有 `args` 的 `value` 字段在 GraphQL schema 中的类型是 `String!`，不是 `ID`。因此 `IdInterceptor` 不会自动解码这些值——它只处理 GraphQL 类型系统中标记为 `ID` 的字段。

### 3.2 完整编解码路径

#### 写入方向（Client → DB）：解码

```
GraphQL mutation (encoded IDs)
        │
        ▼
  IdInterceptor ── 仅处理 type=ID 的 GraphQL 字段 ── 跳过 ConfigArg.value
        │
        ▼
  CollectionResolver.createCollection()
        │
        ▼
  ConfigurableOperationCodec.decodeConfigurableOperationIds()
        │  (configurable-operation-codec.ts:28-52)
        │  对每个 arg:
        │    查找对应 CollectionFilter 的 argDef
        │    if argDef.type === 'ID' && argDef.list === true:
        │      JSON.parse(value) → map(idCodecService.decode) → JSON.stringify
        │    if argDef.type === 'ID' && argDef.list !== true:
        │      idCodecService.decode(value) 直接替换
        ▼
  ConfigArgService.parseInput() ── 校验+重排
        │
        ▼
  Collection.filters (simple-json) ── 存储 raw ID
```

**关键文件**：
- `collection.resolver.ts:117` — createCollection 调用 `decodeConfigurableOperationIds`
- `collection.resolver.ts:130` — updateCollection 调用 `decodeConfigurableOperationIds`
- `collection.resolver.ts:105` — previewCollectionVariants 调用 `decodeConfigurableOperationIds`
- `configurable-operation-codec.ts:28-52` — 解码实现

#### 读取方向（DB → Client）：编码

```
Collection.filters (simple-json, raw IDs)
        │
        ▼
  CollectionEntityResolver.filters() ResolveField
        │  (collection-entity.resolver.ts:149-161)
        │
        ▼
  ConfigurableOperationCodec.encodeConfigurableOperationIds()
        │  (configurable-operation-codec.ts:57-82)
        │  对每个 arg:
        │    if argDef.type === 'ID' && argDef.list === true:
        │      JSON.parse(value) → map(idCodecService.encode) → JSON.stringify
        │    if argDef.type === 'ID' && argDef.list !== true:
        │      idCodecService.encode(value) → JSON.stringify  ⚠️
        ▼
  IdCodecPlugin (Apollo plugin)
        │  (id-codec-plugin.ts:14-59)
        │  对响应中 type=ID 的字段再次编码（不会触及 ConfigArg.value）
        ▼
  GraphQL response (encoded IDs)
```

### 3.3 非列表 ID 编码的潜在 Bug

`encodeConfigurableOperationIds` 中非列表 ID 分支（`configurable-operation-codec.ts:75-76`）：

```ts
const encodedId = this.idCodecService.encode(arg.value);
arg.value = JSON.stringify(encodedId);
```

问题：`JSON.stringify("Q29sbGVjdGlvbjox")` 产生 `'"Q29sbGVjdGlvbjox"'`（多一层引号），而解码端（`configurable-operation-codec.ts:47`）不做 `JSON.stringify`：

```ts
arg.value = this.idCodecService.decode(arg.value);
```

这导致非列表 ID 经历一次 encode→decode 循环后，值被多包裹一层引号。当前四个默认 CollectionFilter 的 ID 类型参数全部声明为 `list: true`，此 bug 未被触发；但自定义 Filter 若使用 `type: 'ID', list: false`，则会在二次保存时损坏 ID 值。

**修正建议**：非列表 ID 编码应与解码对称，去掉 `JSON.stringify`：

```ts
// 修正前
arg.value = JSON.stringify(encodedId);
// 修正后
arg.value = encodedId;
```

### 3.4 EntityIdStrategy 的作用

`IdCodecService` 持有 `EntityIdStrategy` 实例（`id-codec.service.ts:11-13`），所有编解码最终调用策略的 `encodeId` / `decodeId`：

- `AutoIncrementIdStrategy`（默认）：直接返回原始值，无转换
- `UuidIdStrategy`：直接返回原始值，无转换
- 自定义策略（如 `Base64IdStrategy`）：执行实际编解码

当使用 `AutoIncrementIdStrategy` 时，整条编解码链路是恒等映射，因此上面的问题不会产生可见影响。但一旦切换到自定义编码策略，问题就会暴露。

---

## 四、Facet Value Filter 的核心 SQL 逻辑

`facetValueCollectionFilter`（`default-collection-filters.ts:46-128`）的 `apply()` 函数是最关键的自动分组实现：

### 4.1 查询结构

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

### 4.2 `containsAny` 参数的作用

- `containsAny = true` → `count = 1` → 满足任一 FacetValue 即命中
- `containsAny = false` → `count = facetValueIds.length` → 必须满足所有 FacetValue

### 4.3 `combineWithAnd` 参数

控制多 Filter 间的组合方式：
- `combineWithAnd !== false` → 使用 `qb.andWhere()`（AND 组合）
- `combineWithAnd === false` → 使用 `qb.orWhere()`（OR 组合）

### 4.4 ⚠️ Bug：facetValueIds 为空时 OR 分支缺失

当前代码（`default-collection-filters.ts:120-125`）：

```ts
} else {
    // If no facetValueIds are specified, no ProductVariants will be matched.
    if (args.combineWithAnd !== false) {
        qb.andWhere('1 = 0');
    }
    // ← 缺少 else 分支：combineWithAnd === false 时什么都不做
}
```

**问题分析**：

当 `facetValueIds` 为空且 `combineWithAnd === false`（OR 模式）时，代码不添加任何子句，`qb` 原样返回。这意味着此 filter 在 OR 组合中变成 **no-op（恒真）**。

对比 AND 模式（`1 = 0`，恒假），语义不对称：
- AND 模式空 IDs → 无变体匹配（合理：要求"必须拥有这 0 个 FacetValue"无意义，应不匹配）
- OR 模式空 IDs → 对查询无约束（隐式恒真：要求"拥有这 0 个 FacetValue 之一"为空条件）

与其他 Filter 的一致性对比：

| Filter | AND + 空 IDs | OR + 空 IDs |
|---|---|---|
| `variantIdCollectionFilter` | `andWhere('1 = 0')` | `return qb`（显式 no-op） |
| `productIdCollectionFilter` | `andWhere('1 = 0')` | `return qb`（显式 no-op） |
| `facetValueCollectionFilter` | `andWhere('1 = 0')` | 隐式 no-op（**缺少显式 else**） |

**实际影响**：

在大多数场景下，OR 模式空 IDs 的 no-op 行为是合理的——此 filter 不约束结果集，其他 filter 仍正常工作。但存在一个边界场景：

若一个 Collection **仅有** 一个 facet-value-filter 且 IDs 为空、OR 模式，则 `filteredQb` 无任何 WHERE 子句，返回所有 ProductVariant，使得该 Collection 包含全部变体。这与注释声明的 "no ProductVariants will be matched" 矛盾。

**修正建议**：

```ts
} else {
    if (args.combineWithAnd !== false) {
        qb.andWhere('1 = 0');
    } else {
        return qb;
    }
}
```

显式 `return qb` 与 `variantIdCollectionFilter` / `productIdCollectionFilter` 保持一致。若需使空 IDs 在 OR 模式下也不匹配任何变体，则应改为：

```ts
} else {
    if (args.combineWithAnd !== false) {
        qb.andWhere('1 = 0');
    } else {
        qb.orWhere('1 = 0');
    }
}
```

但需注意 TypeORM `orWhere('1 = 0')` 会改变 SQL 结构：`WHERE existing_conditions OR 1 = 0` 等价于 `WHERE existing_conditions`，仅在**唯一 filter 为空的 OR 场景**下才有实际效果（此时无 existing_conditions，`1 = 0` 正确拒绝所有行）。

---

## 五、Filter 继承机制

Collection 以闭包表（closure-table）组织为树形结构，支持 Filter 向下继承：

```
Collection (filters: [A])
  └── Collection (inheritFilters: true, filters: [B])
        └── Collection (inheritFilters: false, filters: [C])
```

`getAncestorFilters()`（`collection.service.ts:843-855`）的遍历逻辑：

1. 如果当前 Collection 的 `inheritFilters = true`，则向上遍历祖先
2. 收集每个祖先的 `filters`
3. 一旦遇到 `inheritFilters === false` 的祖先，立即停止遍历并返回已收集的 filters
4. 最终在 `applyCollectionFiltersInternal()` 中合并：`[...ancestorFilters, ...collection.filters]`

---

## 六、商品变更触发重归组

### 6.1 事件监听

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

### 6.2 触发流程

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

### 6.3 Job 处理逻辑

Job 处理函数（`collection.service.ts:128-190`）：

1. 如果 `collectionIds` 为空，查询所有 Collection 的 ID
2. 逐个 Collection 调用 `applyCollectionFiltersInternal()`
3. 每个 Collection 处理完后更新进度（`job.setProgress`）
4. 如果有受影响变体，发布 `CollectionModificationEvent`（分块 50000）

### 6.4 核心归组算法

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

### 6.5 affectedVariantIds 的两种返回模式

```ts
if (applyToChangedVariantsOnly) {
    return [...toAddIds, ...toRemoveIds];   // 仅变更部分
}
return [...existingIds, ...toRemoveIds];     // 全量（包含未变更的存量）
```

当 `applyToChangedVariantsOnly = false`（如 filter 规则变更时），返回的 `affectedVariantIds` 包含所有现有成员 + 移除的，确保 `CollectionModificationEvent` 的下游（如搜索索引重建）能感知全量影响。

### 6.6 ⚠️ 事务失败后的事件一致性风险

当前 `applyCollectionFiltersInternal()` 的事务错误处理（`collection.service.ts:802-827`）：

```ts
const [toAddIds, toRemoveIds] = await Promise.all([
    addQb.getRawMany().then(results => results.map(result => result.id)),
    removeQb.getRawMany().then(results => results.map(result => result.id)),
]);

try {
    await this.connection.rawConnection.transaction(async transactionalEntityManager => {
        // ... 批量 add/remove
    });
} catch (e: any) {
    Logger.error(e);
    // ⚠️ 事务已回滚，但代码继续执行！
}

if (applyToChangedVariantsOnly) {
    return [...toAddIds, ...toRemoveIds];   // 返回了未实际生效的 ID 列表
}
```

**风险链路**：

```
1. toAddIds/toRemoveIds 基于事务前的快照计算
2. 事务执行失败 → DB 回滚 → 关系表无任何变化
3. 方法仍返回 [toAddIds, toRemoveIds]（已过时的"预期"变更集）
4. 调用方发布 CollectionModificationEvent(ctx, collection, affectedVariantIds)
5. 下游消费者（搜索索引、缓存）收到事件，以为这些变体已增删
6. 实际 DB 状态未变 → 搜索索引与 DB 不一致
```

**具体场景**：

- 搜索插件收到 `CollectionModificationEvent` 后会触发 `updateSearchIndex` Job，将相关变体重新索引。如果事件声称变体 V1 已从 Collection C 移除，搜索索引会删除 V1 在 C 下的记录，但 DB 中 V1 仍在 C 中。
- 反之，事件声称 V2 已加入 C，搜索索引会添加记录，但 DB 中 V2 实际不在 C 中。

**修正建议**：

方案 A：事务失败时返回空数组，不发布事件：
```ts
try {
    await this.connection.rawConnection.transaction(...);
} catch (e: any) {
    Logger.error(e);
    return [];
}
```

方案 B：事务失败时重新查询实际状态，计算真实的差量：
```ts
try {
    await this.connection.rawConnection.transaction(...);
} catch (e: any) {
    Logger.error(e);
    const currentIds = await existingVariantsQb.getRawMany()
        .then(results => results.map(result => result.id));
    return currentIds;
}
```

方案 A 更简洁且安全——既然归组失败，不如静默，让下一次归组 Job 纠正状态。

---

## 七、增量重建策略切换

### 7.1 全局开关

`setApplyAllFiltersOnProductUpdates()`（`collection.service.ts:674-676`）：

```ts
setApplyAllFiltersOnProductUpdates(applyAllFiltersOnProductUpdates: boolean) {
    this.applyAllFiltersOnProductUpdates = applyAllFiltersOnProductUpdates;
}
```

设计意图（注释 `collection.service.ts:661-672`）：大批量导入时，每次商品变更都触发全量归组代价太高，可以先 `setApplyAllFiltersOnProductUpdates(false)` 暂停自动归组，导入完成后手动调用 `triggerApplyFiltersJob()`。

### 7.2 JobBuffer 批量合并

`CollectionJobBuffer`（`collection-job-buffer.ts`）在搜索插件中实现了 Job 的缓冲与合并：

```ts
collect(job): boolean {
    return job.queueName === 'apply-collection-filters';
}

reduce(collectedJobs): Array<Job> {
    const collectionIdsToUpdate = collectedJobs.reduce(
        (result, job) => [...result, ...job.data.collectionIds], []
    );
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

### 7.3 ⚠️ collectionIds 空数组合并时的语义丢失

Job 的 `collectionIds` 字段存在双重语义：
- `[]`（空数组）= 更新**所有** Collection（见 `collection.service.ts:136-143`）
- `[id1, id2, ...]` = 更新**指定** Collection

`CollectionJobBuffer.reduce()` 的合并逻辑（`collection-job-buffer.ts:16-18`）：

```ts
const collectionIdsToUpdate = collectedJobs.reduce((result, job) => {
    return [...result, ...job.data.collectionIds];
}, [] as ID[]);
```

**问题场景**：假设缓冲中有两个 Job：

| Job | collectionIds | applyToChangedVariantsOnly | 来源 |
|---|---|---|---|
| A | `[]`（全部） | `true` | 商品变更事件 |
| B | `[1, 2]` | `false` | Collection filter 规则变更 |

合并后：
```
collectionIdsToUpdate = [] ++ [1, 2] = [1, 2]
unique([1, 2]) = [1, 2]
applyToChangedVariantsOnly = referenceJob.data.applyToChangedVariantsOnly (取 Job A 的值 = true)
```

**两个语义丢失**：

1. **"全部"语义被吞噬**：Job A 本意是更新所有 Collection，合并后变成只更新 `[1, 2]`。Collection 3, 4, 5... 的归组被跳过。
2. **`applyToChangedVariantsOnly` 被错误覆盖**：Job B 需要 `false`（全量重算），但合并后取了 Job A 的 `true`。结果 Collection 1 和 2 只做增量计算，filter 规则变更后应该全量重算的语义丢失。

**修正建议**：

方案 A：空数组在 reduce 时特殊处理——如果任一 Job 的 collectionIds 为空，合并结果也为空（保留"全部"语义）：

```ts
reduce(collectedJobs): Array<Job<any>> {
    const hasAllCollectionsJob = collectedJobs.some(job => job.data.collectionIds.length === 0);
    const allCollectionIds = collectedJobs.reduce(
        (result, job) => [...result, ...job.data.collectionIds], [] as ID[],
    );
    const referenceJob = collectedJobs[0];
    const shouldApplyToAll = hasAllCollectionsJob
        ? collectedJobs.some(job => job.data.applyToChangedVariantsOnly === false)
            ? false
            : referenceJob.data.applyToChangedVariantsOnly
        : referenceJob.data.applyToChangedVariantsOnly;

    return [new Job({
        ...referenceJob,
        id: undefined,
        data: {
            collectionIds: hasAllCollectionsJob ? [] : unique(allCollectionIds),
            ctx: referenceJob.data.ctx,
            applyToChangedVariantsOnly: shouldApplyToAll,
        },
    })];
}
```

方案 B：简单规则——如果任一 Job 需要 `applyToChangedVariantsOnly: false`，合并结果也用 `false`（宁可多算，不可少算）：

```ts
const anyRequiresFull = collectedJobs.some(job => job.data.applyToChangedVariantsOnly === false);
applyToChangedVariantsOnly: anyRequiresFull ? false : referenceJob.data.applyToChangedVariantsOnly,
```

### 7.4 缓冲的激活时机

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

### 7.5 缓冲的 flush 流程

`JobQueueService.flush()` → `JobBufferService.flush()`：
1. 从缓冲存储中取出所有收集的 Job
2. 传入 `JobBuffer.reduce()` 进行合并
3. 将合并后的 Job 提交到实际队列执行

---

## 八、完整数据流图

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
        │                                         │ ⚠️ 空数组语义丢失风险
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
        ├─ 事务成功 ──► 返回 affectedVariantIds
        │                    │
        │                    ▼
        │              CollectionModificationEvent
        │                    │
        │                    ▼
        │              搜索索引更新 / 其他下游处理
        │
        └─ 事务失败 ⚠️ ──► 仍返回旧 toAddIds + toRemoveIds
                               │
                               ▼
                         CollectionModificationEvent (虚假变更集)
                               │
                               ▼
                         搜索索引与 DB 不一致
```

---

## 九、已识别问题汇总

| # | 问题 | 位置 | 严重性 | 触发条件 |
|---|---|---|---|---|
| 1 | facetValueIds 空 + OR 分支缺少显式处理 | `default-collection-filters.ts:120-125` | 低 | 空 IDs + OR 模式，且为唯一 filter |
| 2 | 非列表 ID 编码多做 `JSON.stringify` | `configurable-operation-codec.ts:75-76` | 中 | 自定义 Filter 使用 `type:'ID', list:false` 且启用非恒等 EntityIdStrategy |
| 3 | 事务失败后仍返回预期变更 ID | `collection.service.ts:802-830` | 高 | 事务执行失败（如死锁、超时） |
| 4 | JobBuffer 合并吞噬空数组"全量"语义 | `collection-job-buffer.ts:16-18` | 高 | 缓冲中混合"全量"Job 和"指定"Job |
| 5 | JobBuffer 合并错误取 `applyToChangedVariantsOnly` | `collection-job-buffer.ts:27` | 高 | 缓冲中混合 `true` 和 `false` 的 Job |

---

## 十、关键文件索引

| 文件 | 核心职责 |
|---|---|
| `core/src/config/catalog/collection-filter.ts` | CollectionFilter 类定义，`apply()` 签名 |
| `core/src/config/catalog/default-collection-filters.ts` | 四个内置 Filter 实现，含 facet-value-filter 的 SQL 逻辑 |
| `core/src/config/vendure-config.ts:763-770` | CatalogOptions.collectionFilters 注册入口 |
| `core/src/entity/collection/collection.entity.ts:70` | filters 以 `simple-json` 存储 |
| `core/src/entity/collection/collection.entity.ts:75` | inheritFilters 字段 |
| `core/src/service/services/collection.service.ts:104-126` | 事件监听 + 防抖触发 |
| `core/src/service/services/collection.service.ts:728-837` | 核心归组算法（CTE 差量） |
| `core/src/service/services/collection.service.ts:802-827` | 事务执行与错误处理 ⚠️ |
| `core/src/service/services/collection.service.ts:674-676` | 全局开关 setApplyAllFiltersOnProductUpdates |
| `core/src/service/services/collection.service.ts:687-702` | triggerApplyFiltersJob 入口 |
| `core/src/api/common/configurable-operation-codec.ts:28-52` | 解码：GraphQL → DB 的 ID 转换 |
| `core/src/api/common/configurable-operation-codec.ts:57-82` | 编码：DB → GraphQL 的 ID 转换 ⚠️ |
| `core/src/api/resolvers/admin/collection.resolver.ts:117,130` | Resolver 层调用 decode |
| `core/src/api/resolvers/entity/collection-entity.resolver.ts:149-161` | ResolveField 调用 encode |
| `core/src/api/middleware/id-interceptor.ts` | GraphQL 入站 ID 解码（不触及 ConfigArg） |
| `core/src/api/middleware/id-codec-plugin.ts` | GraphQL 出站 ID 编码（不触及 ConfigArg） |
| `core/src/api/common/id-codec.ts` | IdCodec 递归变换实现 |
| `core/src/api/common/id-codec.service.ts` | IdCodecService 注入 EntityIdStrategy |
| `core/src/config/entity/entity-id-strategy.ts` | EntityIdStrategy 接口：encodeId/decodeId |
| `core/src/plugin/default-search-plugin/search-job-buffer/collection-job-buffer.ts` | Job 缓冲与合并 ⚠️ |
| `core/src/job-queue/job-buffer/job-buffer.ts` | JobBuffer 接口定义 |
| `core/src/job-queue/job-buffer/job-buffer.service.ts` | 缓冲存储、flush、reduce 调度 |
| `core/src/service/helpers/config-arg/config-arg.service.ts:72-80` | parseInput 将 API 输入转为存储格式 |
