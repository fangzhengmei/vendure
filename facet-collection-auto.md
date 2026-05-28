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

## 四、预览链路与实际归组链路对照

### 4.1 两条路径入口

| | 预览链路 | 实际归组链路 |
|---|---|---|
| **入口** | `previewCollectionVariants()` | `applyCollectionFiltersInternal()` |
| **调用时机** | Admin UI 编辑 Collection 时的实时预览 | Job 队列异步执行 |
| **输入来源** | `PreviewCollectionVariantsInput`（前端传入） | 已持久化的 `Collection.filters` |
| **查询方式** | `listQueryBuilder.build()` 带 `channelId` 限制 | `masterConnection` 无 channel 限制 |
| **返回值** | `PaginatedList<ProductVariant>`（实体对象） | `ID[]`（仅 ID） |

### 4.2 inheritFilters 截断处理的核心差异

这是两条路径最关键的行为分歧点。

#### 预览链路：不截断，收集全部祖先 filters

```ts
// collection.service.ts:495-504
async previewCollectionVariants(ctx, input, options, relations) {
    const applicableFilters = this.getCollectionFiltersFromInput(input);
    if (input.parentId && input.inheritFilters) {
        const parentFilters = (await this.findOne(ctx, input.parentId, []))?.filters ?? [];
        const ancestorFilters = await this.getAncestors(input.parentId).then(ancestors =>
            ancestors.reduce(
                (_filters, c) => [..._filters, ...(c.filters || [])],
                [] as ConfigurableOperation[],
            ),
        );
        applicableFilters.push(...parentFilters, ...ancestorFilters);
    }
    // ... 然后 apply filters
}
```

逻辑：
1. 从 `input` 解析用户正在编辑的 filters
2. 如果 `input.parentId && input.inheritFilters` 为真，则加载：
   - 直接 parent 的 `filters`
   - **所有祖先的 `filters`，遍历时完全忽略 `inheritFilters: false` 截断**
3. 合并顺序：`[...inputFilters, ...parentFilters, ...ancestorFilters]`

#### 实际归组链路：截断，遇 inheritFilters: false 即停

```ts
// collection.service.ts:843-855
private async getAncestorFilters(collection: Collection): Promise<ConfigurableOperation[]> {
    const ancestorFilters: ConfigurableOperation[] = [];
    if (collection.inheritFilters) {
        const ancestors = await this.getAncestors(collection.id);
        for (const ancestor of ancestors) {
            ancestorFilters.push(...ancestor.filters);
            if (ancestor.inheritFilters === false) {
                return ancestorFilters;   // ← 截断！
            }
        }
    }
    return ancestorFilters;
}

// collection.service.ts:733-734
const ancestorFilters = await this.getAncestorFilters(collection);
const filters = [...ancestorFilters, ...(collection.filters || [])];
```

逻辑：
1. 从已持久化的 `collection.inheritFilters` 判断是否要收集祖先 filters
2. 遍历祖先时，先 push 当前祖先的 filters，**再检查**该祖先的 `inheritFilters`
3. 一旦遇到 `inheritFilters === false` 的祖先，立即返回已收集的 filters
4. 合并顺序：`[...ancestorFilters, ...collection.filters]`

#### 差异对照表

```
假设树结构：
  Root
   └── A (filters: [a], inheritFilters: false)
         └── B (filters: [b], inheritFilters: true)
               └── C (filters: [c], inheritFilters: true)
```

| | 预览 C 时 | 实际归组 C 时 |
|---|---|---|
| 收集的祖先 filters | `[a, b]`（A 和 B 的全部，不检查 A 的 inheritFilters） | `[a, b]`（A 的 filters 先 push，再检查 A.inheritFilters===false → 截断返回） |
| 最终 filter 列表 | `[c, b, a]` | `[a, b, c]` |

上面这个例子碰巧结果相同，但换一个结构：

```
  Root
   └── A (filters: [a], inheritFilters: true)
         └── B (filters: [b], inheritFilters: false)
               └── C (filters: [c], inheritFilters: true)
```

| | 预览 C 时（设 parentId=B, inheritFilters=true） | 实际归组 C 时 |
|---|---|---|
| 收集的祖先 filters | `[b, a]`（B 和 A 的全部，不检查 B 的 inheritFlags） | `[b, a]`（B 的 filters 先 push，再检查 B.inheritFilters===false → 截断返回 `[b, a]`） |
| 最终 filter 列表 | `[c, b, a]` | `[b, a, c]` |

再次碰巧相同。但关键区别在于：**预览路径不检查祖先的 `inheritFilters` 截断标记，而实际归组路径会**。当截断点在更高层时差异才暴露：

```
  Root
   └── A (filters: [a], inheritFilters: false)
         └── B (filters: [b], inheritFilters: true)
               └── C (filters: [c], inheritFilters: true)
```

预览 C 时（设 `parentId=B, inheritFilters=true`）：
- `getAncestors(B)` → 返回 `[A]`
- `ancestorFilters = reduce([A]) → [a]`（**不检查** A.inheritFilters）
- 最终：`[c, b, a]`

实际归组 C 时：
- `getAncestorFilters(C)` → C.inheritFilters=true，遍历 `[B, A]`
- push B.filters → `[b]`，B.inheritFilters=true → 继续
- push A.filters → `[b, a]`，A.inheritFilters=false → **截断返回**
- 最终：`[b, a, c]`

**结果一致**——但这是因为预览的 parentId 是 B 而非 C 的实际 parentId。预览走的是 `getAncestors(input.parentId)`，实际走的是 `getAncestors(collection.id)` 后向上遍历。两者遍历的起点不同：预览从 parent 开始向上，实际从自身开始向上。

**真正的差异场景**：当 Collection C 本身的 `inheritFilters` 为 `false` 时：

```
  Root
   └── A (filters: [a], inheritFilters: true)
         └── C (filters: [c], inheritFilters: false)
```

| | 预览 C 时 | 实际归组 C 时 |
|---|---|---|
| 自身 inheritFilters | `input.inheritFilters`（前端传入，用户可能传 true 或 false） | `collection.inheritFilters`（持久化值 = false） |
| 是否收集祖先 | 取决于 `input.inheritFilters` | `collection.inheritFilters === false` → **不收集** |
| 最终 filter 列表 | 若 `input.inheritFilters=true`：`[c, a]` | `[c]` |

**结论**：预览链路无法正确模拟 `inheritFilters: false` 的截断行为，因为它：
1. 收集祖先 filters 时不检查任何祖先的 `inheritFilters` 标记
2. 仅依赖前端传入的 `input.inheritFilters` 决定是否加载祖先 filters
3. 与实际归组使用的 `getAncestorFilters()` 逻辑完全不同

这意味着用户在 Admin UI 中预览一个 `inheritFilters: false` 的 Collection 时，如果前端传了 `inheritFilters: true`，预览结果会包含祖先 filters，但实际归组不会应用祖先 filters——**预览与实际结果不一致**。

---

## 五、Facet Value Filter 的核心 SQL 逻辑

`facetValueCollectionFilter`（`default-collection-filters.ts:46-128`）的 `apply()` 函数是最关键的自动分组实现：

### 5.1 查询结构

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

### 5.2 `containsAny` 参数的作用

- `containsAny = true` → `count = 1` → 满足任一 FacetValue 即命中
- `containsAny = false` → `count = facetValueIds.length` → 必须满足所有 FacetValue

### 5.3 `combineWithAnd` 参数

控制多 Filter 间的组合方式：
- `combineWithAnd !== false` → 使用 `qb.andWhere()`（AND 组合）
- `combineWithAnd === false` → 使用 `qb.orWhere()`（OR 组合）

### 5.4 OR 空 IDs 行为的校正定性

当前代码（`default-collection-filters.ts:120-125`）：

```ts
} else {
    // If no facetValueIds are specified, no ProductVariants will be matched.
    if (args.combineWithAnd !== false) {
        qb.andWhere('1 = 0');
    }
    // ← OR 模式时空 IDs：什么都不做
}
```

**定性**：这**不是 bug**，而是**有意的设计**，与 `variantIdCollectionFilter` / `productIdCollectionFilter` 行为完全一致：

| Filter | AND + 空 IDs | OR + 空 IDs |
|---|---|---|
| `variantIdCollectionFilter` | `andWhere('1 = 0')` | `return qb`（显式 no-op） |
| `productIdCollectionFilter` | `andWhere('1 = 0')` | `return qb`（显式 no-op） |
| `facetValueCollectionFilter` | `andWhere('1 = 0')` | 隐式 no-op（无显式 else） |

**OR 模式空 IDs 的 no-op 语义是合理的**：

OR 语义的本质是"满足此条件**或**其他条件之一即可"。当一个 filter 的 IDs 为空时，该 filter 不提供任何约束，等同于 `WHERE TRUE OR other_condition` → `TRUE`，即退化为对最终结果无影响。这与 AND 模式（`WHERE FALSE AND other_condition` → `FALSE`，必须全部拒绝）正好对称。

如果 OR 空 IDs 改为 `orWhere('1 = 0')`，会导致：
- 多 filter 组合时：`WHERE other_filter_clause OR 1 = 0` → 等价于 `WHERE other_filter_clause`（无害但冗余）
- 单 filter 且空 IDs：`WHERE 1 = 0` → 拒绝所有变体（过于严格，与"OR 语义=放宽约束"矛盾）

**代码质量问题**：虽然行为正确，但缺少显式 `else { return qb; }` 与其他 Filter 保持一致的代码风格，可读性不佳。建议补齐：

```ts
} else {
    if (args.combineWithAnd !== false) {
        qb.andWhere('1 = 0');
    } else {
        return qb;
    }
}
```

**唯一需要注意的边界**：在 `applyCollectionFiltersInternal` 中，`filteredQb` 从零开始构建，如果 Collection 的所有 filter 都在 OR 空 IDs 模式下返回 no-op，且无祖先 filter，则 `filteredQb` 无任何 WHERE 子句，会返回全部 ProductVariant。但 `filters.length === 0` 的守卫（`collection.service.ts:746-748`）在 filters 数组本身非空时不触发，所以确实存在这个窗口。不过此场景要求管理员故意配置空 IDs 的 OR filter，Admin UI 的 `facet-value-form-input` 组件通常会阻止这种配置。

---

## 六、Filter 继承机制

Collection 以闭包表（closure-table）组织为树形结构，支持 Filter 向下继承：

```
Collection (filters: [A])
  └── Collection (inheritFilters: true, filters: [B])
        └── Collection (inheritFilters: false, filters: [C])
```

`getAncestorFilters()`（`collection.service.ts:843-855`）的遍历逻辑：

1. 如果当前 Collection 的 `inheritFilters = true`，则向上遍历祖先
2. 收集每个祖先的 `filters`
3. 一旦遇到 `inheritFilters === false` 的祖先，**先 push 该祖先的 filters 再截断**（因为 `push` 在 `if` 检查之前）
4. 最终在 `applyCollectionFiltersInternal()` 中合并：`[...ancestorFilters, ...collection.filters]`

注意第 3 点的执行顺序：

```ts
for (const ancestor of ancestors) {
    ancestorFilters.push(...ancestor.filters);  // 先收集
    if (ancestor.inheritFilters === false) {     // 再检查
        return ancestorFilters;                  // 截断但已包含该祖先
    }
}
```

这意味着 `inheritFilters: false` 的祖先自身的 filters **仍会被收集**，只是阻止了更上层祖先的 filters 继续向下传递。

---

## 七、商品变更触发重归组

### 7.1 事件监听

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

### 7.2 触发流程

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

| 触发场景 | collectionIds | applyToChangedVariantsOnly | ctx.channelToken |
|---|---|---|---|
| 商品/变体变更事件 | `[]`（全部 Collection） | `undefined`（默认 true） | 事件来源 channel |
| 创建 Collection | `[新ID]` | `undefined`（默认 true） | 当前请求 channel |
| 更新 Collection 的 filters | `[当前ID]` | `false` | 当前请求 channel |
| 移动 Collection | `[当前ID]` | `undefined` | 当前请求 channel |

### 7.3 Job 处理逻辑

Job 处理函数（`collection.service.ts:128-190`）：

1. 从 `job.data.ctx` 恢复 RequestContext（`requestContextService.create({ channelOrToken: job.data.ctx.channelToken })`）
2. 如果 `collectionIds` 为空，查询**所有 Collection** 的 ID（`rawConnection`，**跨 channel**）
3. 逐个 Collection 调用 `applyCollectionFiltersInternal()`
4. 每个 Collection 处理完后更新进度
5. 如果有受影响变体，发布 `CollectionModificationEvent`（分块 50000）

**关键细节**：步骤 2 查询所有 Collection 时不按 channel 过滤，因此 Job 在单个 channel 的 ctx 下会处理所有 channel 的 Collection。步骤 3 的 `applyCollectionFiltersInternal` 使用 `masterConnection`，同样无 channel 过滤。这意味着**一个 channel 触发的归组 Job 会影响所有 channel 的 Collection**。

### 7.4 核心归组算法

`applyCollectionFiltersInternal()`（`collection.service.ts:728-837`）使用 CTE 差量计算：

```
_filtered_variants  := 所有匹配当前 Collection filter 规则的变体 ID（跨 channel）
_existing_variants  := 当前已属于该 Collection 的变体 ID（跨 channel）

toAdd    := _filtered_variants LEFT JOIN _existing_variants WHERE existing IS NULL
toRemove := _existing_variants LEFT JOIN _filtered_variants WHERE filtered IS NULL
```

然后事务性地执行：
- 批量移除（`chunkArray(toRemoveIds, 5000)`）
- 批量添加（`chunkArray(toAddIds, 5000)`）

### 7.5 affectedVariantIds 的两种返回模式

```ts
if (applyToChangedVariantsOnly) {
    return [...toAddIds, ...toRemoveIds];   // 仅变更部分
}
return [...existingIds, ...toRemoveIds];     // 全量（包含未变更的存量）
```

### 7.6 ⚠️ 事务失败后的事件一致性风险

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

**修正建议**：事务失败时返回空数组，不发布事件：

```ts
try {
    await this.connection.rawConnection.transaction(...);
} catch (e: any) {
    Logger.error(e);
    return [];
}
```

---

## 八、增量重建策略切换

### 8.1 全局开关

`setApplyAllFiltersOnProductUpdates()`（`collection.service.ts:674-676`）：

```ts
setApplyAllFiltersOnProductUpdates(applyAllFiltersOnProductUpdates: boolean) {
    this.applyAllFiltersOnProductUpdates = applyAllFiltersOnProductUpdates;
}
```

设计意图（注释 `collection.service.ts:661-672`）：大批量导入时，每次商品变更都触发全量归组代价太高，可以先 `setApplyAllFiltersOnProductUpdates(false)` 暂停自动归组，导入完成后手动调用 `triggerApplyFiltersJob()`。

### 8.2 JobBuffer 批量合并

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
        id: undefined,
        data: {
            collectionIds: unique(collectionIdsToUpdate),
            ctx: referenceJob.data.ctx,
            applyToChangedVariantsOnly: referenceJob.data.applyToChangedVariantsOnly,
        },
    })];
}
```

### 8.3 ⚠️ collectionIds 空数组合并时的语义丢失

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
2. **`applyToChangedVariantsOnly` 被错误覆盖**：Job B 需要 `false`（全量重算），但合并后取了 Job A 的 `true`。

**修正建议**：

方案 A：空数组在 reduce 时特殊处理——如果任一 Job 的 collectionIds 为空，合并结果也为空（保留"全部"语义）；如果任一 Job 需要 `applyToChangedVariantsOnly: false`，合并结果也用 `false`：

```ts
reduce(collectedJobs): Array<Job<any>> {
    const hasAllCollectionsJob = collectedJobs.some(job => job.data.collectionIds.length === 0);
    const allCollectionIds = collectedJobs.reduce(
        (result, job) => [...result, ...job.data.collectionIds], [] as ID[],
    );
    const anyRequiresFull = collectedJobs.some(job => job.data.applyToChangedVariantsOnly === false);
    const referenceJob = collectedJobs[0];

    return [new Job({
        ...referenceJob,
        id: undefined,
        data: {
            collectionIds: hasAllCollectionsJob ? [] : unique(allCollectionIds),
            ctx: referenceJob.data.ctx,
            applyToChangedVariantsOnly: anyRequiresFull ? false : referenceJob.data.applyToChangedVariantsOnly,
        },
    })];
}
```

### 8.4 ⚠️ reduce 固定使用 referenceJob.ctx 的跨 channel 影响

`CollectionJobBuffer.reduce()` 始终使用 `referenceJob = collectedJobs[0]` 的 `ctx`：

```ts
const referenceJob = collectedJobs[0];
// ...
ctx: referenceJob.data.ctx,
```

`ApplyCollectionFiltersJobData.ctx` 包含：
```ts
ctx: {
    channelToken: string;   // 触发此 Job 的 channel
    languageCode: LanguageCode;
}
```

**问题场景**：假设缓冲中有来自不同 channel 的 Job：

| Job | ctx.channelToken | 来源 |
|---|---|---|
| A | `channel-default` | Channel 1 的商品变更 |
| B | `channel-eu` | Channel 2 的商品变更 |

合并后，`ctx` 取 Job A 的值 → `channelToken: 'channel-default'`。

**影响分析**：

在 Job 处理函数中（`collection.service.ts:130-134`）：
```ts
const ctx = await this.requestContextService.create({
    apiType: 'admin',
    languageCode: job.data.ctx.languageCode,
    channelOrToken: job.data.ctx.channelToken,
});
```

恢复的 `ctx` 绑定到 `channel-default`。但随后的处理中：

1. **查询全部 Collection 时不按 channel 过滤**（`collection.service.ts:138-143`）：
   ```ts
   const collections = await this.connection.rawConnection
       .getRepository(Collection)
       .createQueryBuilder('collection')
       .select('collection.id', 'id')
       .getRawMany();
   ```
   跨 channel 查所有 Collection，所以 Channel 2 的 Collection 也会被处理。

2. **`getEntityOrThrow` 受 channel 限制**（`collection.service.ts:153-156`）：
   ```ts
   collection = await this.connection.getEntityOrThrow(ctx, Collection, collectionId, {
       retries: 5,
       retryDelay: 50,
   });
   ```
   如果 Collection 属于 Channel 2 但不属于 Channel 1，而 ctx 绑定 Channel 1，则此查询会**抛出异常**，该 Collection 被跳过。

3. **`applyCollectionFiltersInternal` 无 channel 限制**：使用 `masterConnection`，查询和写入都不按 channel 过滤。

**结果**：如果合并后 ctx 指向 Channel 1，那么仅属于 Channel 2 的 Collection 在步骤 2 会被跳过，其归组不执行。而属于 Channel 1 的 Collection 的 `applyCollectionFiltersInternal` 却会操作跨 channel 的 ProductVariant（步骤 3），可能错误地将其他 channel 的变体关联到 Channel 1 的 Collection。

**修正建议**：

方案 A：reduce 时收集所有不同的 channelToken，生成多个 Job（每个 channel 一个）：

```ts
reduce(collectedJobs): Array<Job<any>> {
    const channelTokens = unique(collectedJobs.map(j => j.data.ctx.channelToken));
    const hasAllCollectionsJob = collectedJobs.some(job => job.data.collectionIds.length === 0);
    const anyRequiresFull = collectedJobs.some(job => job.data.applyToChangedVariantsOnly === false);

    if (channelTokens.length === 1) {
        // 单 channel：行为与当前一致
        const referenceJob = collectedJobs[0];
        const allCollectionIds = collectedJobs.reduce(
            (r, j) => [...r, ...j.data.collectionIds], [] as ID[],
        );
        return [new Job({
            ...referenceJob, id: undefined,
            data: {
                collectionIds: hasAllCollectionsJob ? [] : unique(allCollectionIds),
                ctx: referenceJob.data.ctx,
                applyToChangedVariantsOnly: anyRequiresFull ? false : referenceJob.data.applyToChangedVariantsOnly,
            },
        })];
    }

    // 多 channel：每个 channel 生成一个 Job
    return channelTokens.map(token => {
        const jobsForChannel = collectedJobs.filter(j => j.data.ctx.channelToken === token);
        const referenceJob = jobsForChannel[0];
        const allCollectionIds = jobsForChannel.reduce(
            (r, j) => [...r, ...j.data.collectionIds], [] as ID[],
        );
        const hasAll = jobsForChannel.some(j => j.data.collectionIds.length === 0);
        return new Job({
            ...referenceJob, id: undefined,
            data: {
                collectionIds: hasAll ? [] : unique(allCollectionIds),
                ctx: referenceJob.data.ctx,
                applyToChangedVariantsOnly: anyRequiresFull ? false : referenceJob.data.applyToChangedVariantsOnly,
            },
        });
    });
}
```

方案 B：更激进的方案——让 `applyCollectionFiltersInternal` 在每个 Collection 所属的 channel 上下文中执行，而非使用 Job 的统一 ctx。但这需要更大的重构。

### 8.5 缓冲的激活时机

`SearchJobBufferService`（`search-job-buffer.service.ts:25-29`）在应用启动时根据配置激活：

```ts
onApplicationBootstrap() {
    if (this.bufferUpdates === true) {
        this.jobQueueService.addBuffer(this.searchIndexJobBuffer);
        this.jobQueueService.addBuffer(this.collectionJobBuffer);
    }
}
```

### 8.6 缓冲的 flush 流程

`JobQueueService.flush()` → `JobBufferService.flush()`：
1. 从缓冲存储中取出所有收集的 Job
2. 传入 `JobBuffer.reduce()` 进行合并
3. 将合并后的 Job 提交到实际队列执行

---

## 九、完整数据流图

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
  triggerApplyFiltersJob(ctx) ── collectionIds: [] (全量), ctx: 当前 channel
        │
        ▼
  JobQueue.add() ──► CollectionJobBuffer.collect() ── 缓冲
        │                                         │
        │                                  flush 时 reduce()
        │                                         │ ⚠️ 空数组语义丢失
        │                                         │ ⚠️ referenceJob.ctx 丢失其他 channel
        ▼                                         ▼
  apply-collection-filters Job 执行         合并后的单个/多个 Job
        │
        ▼
  requestContextService.create({ channelOrToken })  ← 从 job.data.ctx 恢复
        │
        ▼
  查询所有 Collection (跨 channel, rawConnection)
        │
        ▼
  getEntityOrThrow(ctx, Collection, id) ← 受 channel 限制！
        │                                    ⚠️ 仅属于其他 channel 的 Collection 被跳过
        ▼
  applyCollectionFiltersInternal(collection):
    1. getAncestorFilters(collection)  ← 有 inheritFilters 截断
    2. 合并 [ancestorFilters + collection.filters]
    3. 构建 filteredQb (masterConnection, 跨 channel)
    4. 构建 existingVariantsQb (masterConnection, 跨 channel)
    5. CTE 差量: toAdd / toRemove
    6. 事务执行: 批量 add/remove (chunk=5000)
        │
        ├─ 事务成功 ──► 返回 affectedVariantIds
        │                    │
        │                    ▼
        │              CollectionModificationEvent(ctx, collection, chunk)
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

  ┌─ 预览链路（对照）──────────────────────────────────────────┐
  │ previewCollectionVariants(ctx, input):                      │
  │   1. getCollectionFiltersFromInput(input)                   │
  │   2. if input.parentId && input.inheritFilters:             │
  │        parentFilters = findOne(parentId).filters             │
  │        ancestorFilters = getAncestors(parentId).reduce(...)  │
  │        ⚠️ 不检查祖先的 inheritFilters 截断                  │
  │   3. 合并 [inputFilters, parentFilters, ancestorFilters]     │
  │   4. listQueryBuilder.build() (带 channelId 限制)            │
  │   5. 返回 PaginatedList<ProductVariant>                      │
  └─────────────────────────────────────────────────────────────┘
```

---

## 十、已识别问题汇总

| # | 问题 | 位置 | 严重性 | 触发条件 |
|---|---|---|---|---|
| 1 | 预览链路不遵循 inheritFilters 截断 | `collection.service.ts:496-504` | 中 | 预览 inheritFilters=false 的 Collection，前端传 inheritFilters=true |
| 2 | 非列表 ID 编码多做 `JSON.stringify` | `configurable-operation-codec.ts:75-76` | 中 | 自定义 Filter 使用 `type:'ID', list:false` 且启用非恒等 EntityIdStrategy |
| 3 | 事务失败后仍返回预期变更 ID | `collection.service.ts:802-830` | 高 | 事务执行失败（如死锁、超时） |
| 4 | JobBuffer 合并吞噬空数组"全量"语义 | `collection-job-buffer.ts:16-18` | 高 | 缓冲中混合"全量"Job 和"指定"Job |
| 5 | JobBuffer 合并错误取 `applyToChangedVariantsOnly` | `collection-job-buffer.ts:27` | 高 | 缓冲中混合 `true` 和 `false` 的 Job |
| 6 | JobBuffer reduce 固定用 referenceJob.ctx 丢失其他 channel | `collection-job-buffer.ts:26` | 高 | 缓冲中混合来自不同 channel 的 Job |

---

## 十一、关键文件索引

| 文件 | 核心职责 |
|---|---|
| `core/src/config/catalog/collection-filter.ts` | CollectionFilter 类定义，`apply()` 签名 |
| `core/src/config/catalog/default-collection-filters.ts` | 四个内置 Filter 实现，含 facet-value-filter 的 SQL 逻辑 |
| `core/src/config/vendure-config.ts:763-770` | CatalogOptions.collectionFilters 注册入口 |
| `core/src/entity/collection/collection.entity.ts:70` | filters 以 `simple-json` 存储 |
| `core/src/entity/collection/collection.entity.ts:75` | inheritFilters 字段 |
| `core/src/service/services/collection.service.ts:104-126` | 事件监听 + 防抖触发 |
| `core/src/service/services/collection.service.ts:489-527` | 预览链路 ⚠️ inheritFilters 截断缺失 |
| `core/src/service/services/collection.service.ts:728-837` | 核心归组算法（CTE 差量） |
| `core/src/service/services/collection.service.ts:843-855` | getAncestorFilters 有截断 |
| `core/src/service/services/collection.service.ts:802-827` | 事务执行与错误处理 ⚠️ |
| `core/src/service/services/collection.service.ts:674-676` | 全局开关 setApplyAllFiltersOnProductUpdates |
| `core/src/service/services/collection.service.ts:687-702` | triggerApplyFiltersJob 入口 |
| `core/src/service/services/collection.service.ts:130-134` | Job 处理中从 ctx 恢复 RequestContext |
| `core/src/service/services/collection.service.ts:136-143` | 空 collectionIds 时跨 channel 查全部 Collection |
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
| `core/src/api/schema/admin-api/collection.api.graphql:69-73` | PreviewCollectionVariantsInput 定义 |
