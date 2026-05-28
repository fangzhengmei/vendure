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

### 4.3 混合 AND/OR 条件下的过滤器应用顺序差异

两条路径的过滤器应用逻辑结构完全相同：

```ts
const { collectionFilters } = this.configService.catalogOptions;
for (const filterType of collectionFilters) {               // 外层：按 config 顺序遍历 filterType
    const filtersOfType = filters.filter(f => f.code === filterType.code);
    if (filtersOfType.length) {
        for (const collectionFilter of filtersOfType) {     // 内层：按数组顺序应用该类型的所有 filter
            qb = filterType.apply(qb, collectionFilter.args);
        }
    }
}
```

**差异仅在于 `filters` 数组的顺序**：

| 路径 | filters 数组顺序 |
|---|---|
| 预览 | `[...inputFilters, ...parentFilters, ...ancestorFilters]` |
| 实际归组 | `[...ancestorFilters, ...collection.filters]` |

**对 SQL 语义的影响**：

SQL 中 `AND` 的优先级高于 `OR`，因此过滤器的应用顺序会直接影响最终的 WHERE 子句语义。

例如，假设 Collection 配置如下：
- 祖先 A: `facet-value-filter` (combineWithAnd = true, facetValueIds = [1]) → `AND productVariant.id IN (...)`
- 自身 C: `facet-value-filter` (combineWithAnd = false, facetValueIds = [2]) → `OR productVariant.id IN (...)`

| 路径 | 应用顺序 | 生成的 SQL WHERE | 实际语义 |
|---|---|---|---|
| 预览 | `[C, A]` | `cond_C OR cond_A` | 任一 filter 满足即可 |
| 实际归组 | `[A, C]` | `cond_A AND cond_C` | 两个 filter 必须同时满足 |

两者语义完全相反！

**实际场景验证**：

如果祖先 filter 是 `AND facetValueIds=[1]`，自身 filter 是 `OR facetValueIds=[2]`：
- 预览：`cond2 OR cond1` → 变体有 facetValue 2 或 1 都会匹配
- 实际归组：`cond1 AND cond2` → 变体必须同时有 facetValue 1 和 2

**问题根源**：

`combineWithAnd` 参数控制的是**这个 filter 相对于已有条件的组合方式**，而不是 filter 内部的逻辑。因此，当同一 filterType 存在多个实例且 `combineWithAnd` 混合时，应用顺序直接决定了 SQL 的组合逻辑。

**修正建议**：

在 `getCollectionFiltersFromInput` 中调整预览的 filter 合并顺序，使其与实际归组一致：

```ts
// 修正前
applicableFilters.push(...parentFilters, ...ancestorFilters);
// 修正后
applicableFilters.unshift(...ancestorFilters, ...parentFilters);
```

这样预览的 filter 顺序变为 `[...ancestorFilters, ...parentFilters, ...inputFilters]`，与实际归组的 `[...ancestorFilters, ...collection.filters]` 一致。

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

## 六、混合 AND/OR 条件下的 SQL 语义深度分析

### 6.1 TypeORM andWhere/orWhere 的基本语义

TypeORM 的 `andWhere`/`orWhere` 每次都会用括号包裹新条件，然后用 AND/OR 连接到已有 WHERE 子句。例如：

```ts
qb.where('A').andWhere('B').orWhere('C').andWhere('D');
// 生成: WHERE (A) AND (B) OR (C) AND (D)
// 由于 AND 优先级高于 OR，等价于: WHERE ((A) AND (B)) OR ((C) AND (D))
```

**关键点**：每个新条件都被独立括号包裹，逻辑运算优先级由 SQL 标准决定（AND 优先于 OR）。

### 6.2 场景设定

考虑以下 Collection 层级（所有 `inheritFilters = true`）：

```
Root (facetValueIds=[1], combineWithAnd=true → AND)
  └── Grandparent (facetValueIds=[2], combineWithAnd=false → OR)
        └── Parent (facetValueIds=[3], combineWithAnd=true → AND)
              └── Collection (facetValueIds=[4], combineWithAnd=false → OR)
```

每个 Collection 都有一个 `facet-value-filter`，但 `combineWithAnd` 配置不同。

### 6.3 预览链路的 filter 顺序

`previewCollectionVariants`（`collection.service.ts:495-504`）：

```ts
const applicableFilters = this.getCollectionFiltersFromInput(input);  // [input.filters]
if (input.parentId && input.inheritFilters) {
    const parentFilters = (await this.findOne(ctx, input.parentId, []))?.filters ?? [];
    const ancestorFilters = await this.getAncestors(input.parentId).then(ancestors =>
        ancestors.reduce((_filters, c) => [..._filters, ...(c.filters || [])], [])
    );
    applicableFilters.push(...parentFilters, ...ancestorFilters);  // 追加到末尾
}
```

假设 `input.parentId = Parent.id`（预览时选择继承 Parent 的 filters），则最终顺序为：

```
applicableFilters = [
  { facetValueIds: [4], combineWithAnd: false },  // Collection (input) → OR
  { facetValueIds: [3], combineWithAnd: true  },  // Parent            → AND
  { facetValueIds: [2], combineWithAnd: false },  // Grandparent       → OR
  { facetValueIds: [1], combineWithAnd: true  },  // Root              → AND
]
```

**顺序特征**：`[自身, 父, 祖父, ..., 根]`

### 6.4 实际归组链路的 filter 顺序

`applyCollectionFiltersInternal`（`collection.service.ts:733-734`）：

```ts
const ancestorFilters = await this.getAncestorFilters(collection);
const filters = [...ancestorFilters, ...(collection.filters || [])];
```

`getAncestorFilters`（`collection.service.ts:843-855`）从下往上遍历祖先（Parent → Grandparent → Root），所以：

```
ancestorFilters = [
  { facetValueIds: [3], combineWithAnd: true  },  // Parent      → AND
  { facetValueIds: [2], combineWithAnd: false },  // Grandparent → OR
  { facetValueIds: [1], combineWithAnd: true  },  // Root        → AND
]

filters = [...ancestorFilters, ...collection.filters] = [
  { facetValueIds: [3], combineWithAnd: true  },  // Parent            → AND
  { facetValueIds: [2], combineWithAnd: false },  // Grandparent       → OR
  { facetValueIds: [1], combineWithAnd: true  },  // Root              → AND
  { facetValueIds: [4], combineWithAnd: false },  // Collection        → OR
]
```

**顺序特征**：`[父, 祖父, ..., 根, 自身]`

### 6.5 预览链路的 SQL 语义推导

预览链路使用 `listQueryBuilder.build()`，初始 WHERE 包含：
```sql
WHERE productVariant.deletedAt IS NULL
```
记为 `T`（通常为 true，因为已软删除的变体不会被包含）。

定义简写：
- `A` = `id IN (子查询 [4])`（Collection 的 OR filter）
- `B` = `id IN (子查询 [3])`（Parent 的 AND filter）
- `C` = `id IN (子查询 [2])`（Grandparent 的 OR filter）
- `D` = `id IN (子查询 [1])`（Root 的 AND filter）

逐步应用：

1. 应用 Collection filter (OR A):
   ```sql
   WHERE T OR (A)
   ```

2. 应用 Parent filter (AND B):
   ```sql
   WHERE (T OR (A)) AND (B)
   ```

3. 应用 Grandparent filter (OR C):
   ```sql
   WHERE ((T OR (A)) AND (B)) OR (C)
   ```

4. 应用 Root filter (AND D):
   ```sql
   WHERE (((T OR (A)) AND (B)) OR (C)) AND (D)
   ```

由于 `T = TRUE`，逐步简化：
```
(((TRUE OR A) AND B) OR C) AND D
  = ((TRUE AND B) OR C) AND D
  = (B OR C) AND D
```

**预览最终表达式**：`(B OR C) AND D`

### 6.6 实际归组链路的 SQL 语义推导

实际归组链路从零构建 QB，初始无 WHERE 子句：
```sql
SELECT productVariant.id FROM product_variant
```

逐步应用：

1. 应用 Parent filter (AND B):
   ```sql
   WHERE (B)
   ```

2. 应用 Grandparent filter (OR C):
   ```sql
   WHERE (B) OR (C)
   ```

3. 应用 Root filter (AND D):
   ```sql
   WHERE ((B) OR (C)) AND (D)
   ```

4. 应用 Collection filter (OR A):
   ```sql
   WHERE (((B) OR (C)) AND (D)) OR (A)
   ```

**实际归组最终表达式**：`((B OR C) AND D) OR A`

### 6.7 语义差异对比（真值表）

对比 `(B OR C) AND D` 与 `((B OR C) AND D) OR A`：

| A | B | C | D | 预览结果 | 实际归组结果 | 是否一致 |
|---|---|---|---|----------|--------------|----------|
| T | F | F | T | (F∨F)∧T = F | ((F∨F)∧T)∨T = T | ❌ 不一致 |
| T | T | F | F | (T∨F)∧F = F | ((T∨F)∧F)∨T = T | ❌ 不一致 |
| T | T | T | T | (T∨T)∧T = T | ((T∨T)∧T)∨T = T | ✅ 一致 |
| F | T | F | T | (T∨F)∧T = T | ((T∨F)∧T)∨F = T | ✅ 一致 |
| F | F | T | T | (F∨T)∧T = T | ((F∨T)∧T)∨F = T | ✅ 一致 |
| F | F | F | T | (F∨F)∧T = F | ((F∨F)∧T)∨F = F | ✅ 一致 |
| T | F | T | F | (F∨T)∧F = F | ((F∨T)∧F)∨T = T | ❌ 不一致 |

**结论**：当 Collection 自身的 filter 使用 `combineWithAnd=false`（OR），且存在祖先 filter 使用 `combineWithAnd=true`（AND）时，预览和实际归组的结果可能完全不同！

**风险场景**：管理员在预览时看到某些变体被包含（或排除），但保存后实际归组的结果却相反。

### 6.8 修正建议

将预览链路的 `push` 改为 `unshift`，保持与实际归组相同的顺序（祖先在前，自身在后）：

```ts
// collection.service.ts:504
// 原代码:
applicableFilters.push(...parentFilters, ...ancestorFilters);
// 修改为:
applicableFilters.unshift(...ancestorFilters, ...parentFilters);
```

修改后预览的顺序变为 `[Root, Grandparent, Parent, Collection]`，与实际归组的 `[Parent, Grandparent, Root, Collection]` 在逻辑上等价（因为同一方向的遍历只是顺序不同，且 `getAncestors` 返回的是从父到根的顺序）。

**注意**：还需要同步修复预览链路不检查 `inheritFilters` 截断的问题（见 6.3 节）。

---

## 七、Filter 继承机制

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

### 7.1 预览与实际归组在 inheritFilters 截断处理上的差异

**实际归组链路**（`getAncestorFilters`，`collection.service.ts:843-855`）会检查 `inheritFilters` 截断：

```ts
for (const ancestor of ancestors) {
    ancestorFilters.push(...ancestor.filters);
    if (ancestor.inheritFilters === false) {
        return ancestorFilters;  // 遇到 inheritFilters=false 则截断
    }
}
```

**预览链路**（`previewCollectionVariants`，`collection.service.ts:496-504`）不检查 `inheritFilters` 截断：

```ts
const ancestorFilters = await this.getAncestors(input.parentId).then(ancestors =>
    ancestors.reduce(
        (_filters, c) => [..._filters, ...(c.filters || [])],  // 直接 reduce，不检查 inheritFilters
        [] as ConfigurableOperation[],
    ),
);
applicableFilters.push(...parentFilters, ...ancestorFilters);
```

**差异总结**：

| 处理点 | 实际归组 | 预览 |
|---|---|---|
| `inheritFilters` 截断 | `getAncestorFilters` 中检查，遇到则停止向上遍历 | 直接 `reduce` 所有祖先，不检查 |
| 父级 filters 获取 | 来自 `ancestors[0]`（即 parent） | 单独 `findOne` 获取 `parentFilters` |
| 祖先顺序 | Parent → Grandparent → ... → Root | Parent → Grandparent → ... → Root（顺序相同，但不截断） |

**风险**：如果某个祖先设置了 `inheritFilters=false`，预览时会错误地包含更上层祖先的 filters，导致预览结果与实际归组结果不一致。

---

## 八、商品变更触发重归组

### 8.1 事件监听

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

### 8.2 触发流程

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

### 8.3 Job 处理逻辑

Job 处理函数（`collection.service.ts:128-190`）：

1. 从 `job.data.ctx` 恢复 RequestContext（`requestContextService.create({ channelOrToken: job.data.ctx.channelToken })`）
2. 如果 `collectionIds` 为空，查询**所有 Collection** 的 ID（`rawConnection`，**跨 channel**）
3. 逐个 Collection 调用 `applyCollectionFiltersInternal()`
4. 每个 Collection 处理完后更新进度
5. 如果有受影响变体，发布 `CollectionModificationEvent`（分块 50000）

**关键细节**：步骤 2 查询所有 Collection 时不按 channel 过滤，因此 Job 在单个 channel 的 ctx 下会处理所有 channel 的 Collection。步骤 3 的 `applyCollectionFiltersInternal` 使用 `masterConnection`，同样无 channel 过滤。这意味着**一个 channel 触发的归组 Job 会影响所有 channel 的 Collection**。

### 8.4 getEntityOrThrow 未传 channelId 的查询意图分析

在 Job 处理函数中，加载 Collection 时调用（`collection.service.ts:153-156`）：

```ts
collection = await this.connection.getEntityOrThrow(ctx, Collection, collectionId, {
    retries: 5,
    retryDelay: 50,
});
```

**未传入 `options.channelId`**。

#### getEntityOrThrow 的分支逻辑

`getEntityOrThrowInternal`（`transactional-connection.ts:313-333`）的实现：

```ts
if (options.channelId != null) {
    // 分支 A：传入了 channelId → 走 findOneInChannel
    entity = await this.findOneInChannel(ctx, entityType, id, options.channelId, ...);
} else {
    // 分支 B：未传入 channelId → 直接 findOne，无 channel 过滤
    const optionsWithId = { ...options, where: { ...(options.where || {}), id } };
    entity = await this.getRepository(ctx, entityType)
        .findOne(optionsWithId)
        .then(result => result ?? undefined);
}
```

`findOneInChannel` 的实现（`transactional-connection.ts:372-374`）：
```ts
qb.leftJoin('entity.channels', '__channel')
    .andWhere('entity.id = :id', { id })
    .andWhere('__channel.id = :channelId', { channelId });
```

#### 设计意图

**不传入 `channelId` 是有意的设计**，理由：

1. **全局归组操作**：`apply-collection-filters` Job 是全局操作——当 `collectionIds` 为空时（商品变更事件触发），需要处理所有 Collection，无论它们属于哪个 channel。

2. **Collection 的全局特性**：Collection 实体本身是全局的，channel 关联通过 `collection.channels` 多对多关系表管理。`getEntityOrThrow` 不按 channel 过滤才能加载到所有 Collection。

3. **与前置查询一致**：步骤 2 查询所有 Collection ID 时用的是 `rawConnection`（无 channel 过滤），如果步骤 3 突然按 channel 过滤，会导致步骤 2 查到的 Collection 在步骤 3 加载不到。

#### 对之前分析的修正

**之前的分析有误**：我之前认为 `getEntityOrThrow` 会按 `ctx.channelId` 过滤其他 channel 的 Collection，实际上由于没有传 `channelId`，它**不会**按 channel 过滤。跨 channel Job 的真正问题在于：
- 恢复的 `ctx` 绑定到某个 channel，但 Collection 查询本身不受影响
- 后续发布的 `CollectionModificationEvent(ctx, ...)` 携带的 ctx 可能与 Collection 实际所属的 channel 不匹配

### 8.5 核心归组算法

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

### 8.6 affectedVariantIds 的两种返回模式

```ts
if (applyToChangedVariantsOnly) {
    return [...toAddIds, ...toRemoveIds];   // 仅变更部分
}
return [...existingIds, ...toRemoveIds];     // 全量（包含未变更的存量）
```

### 8.7 ⚠️ 事务失败后的事件一致性风险

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

## 九、增量重建策略切换

### 9.1 全局开关

`setApplyAllFiltersOnProductUpdates()`（`collection.service.ts:674-676`）：

```ts
setApplyAllFiltersOnProductUpdates(applyAllFiltersOnProductUpdates: boolean) {
    this.applyAllFiltersOnProductUpdates = applyAllFiltersOnProductUpdates;
}
```

设计意图（注释 `collection.service.ts:661-672`）：大批量导入时，每次商品变更都触发全量归组代价太高，可以先 `setApplyAllFiltersOnProductUpdates(false)` 暂停自动归组，导入完成后手动调用 `triggerApplyFiltersJob()`。

### 9.2 JobBuffer 批量合并

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

### 9.3 ⚠️ collectionIds 空数组合并时的语义丢失

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

### 9.4 ⚠️ reduce 固定使用 referenceJob.ctx 的影响

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

#### 9.4.1 跨 channel 影响（修正之前的分析）

**问题场景**：假设缓冲中有来自不同 channel 的 Job：

| Job | ctx.channelToken | 来源 |
|---|---|---|
| A | `channel-default` | Channel 1 的商品变更 |
| B | `channel-eu` | Channel 2 的商品变更 |

合并后，`ctx` 取 Job A 的值 → `channelToken: 'channel-default'`。

**影响分析（已修正）**：

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

2. **`getEntityOrThrow` 也不按 channel 过滤**（`collection.service.ts:153-156`）：
   ```ts
   collection = await this.connection.getEntityOrThrow(ctx, Collection, collectionId, {
       retries: 5,
       retryDelay: 50,
   });
   ```
   **之前的分析有误**：由于没有传 `channelId` 选项，`getEntityOrThrow` 走的是直接 `findOne({ where: { id } })` 分支，**不会**按 channel 过滤。所有 Collection 都能正常加载。

3. **`applyCollectionFiltersInternal` 无 channel 限制**：使用 `masterConnection`，查询和写入都不按 channel 过滤。

**真正的跨 channel 问题**：
- 恢复的 `ctx` 绑定到某个 channel，但后续处理完全忽略这个 channel 限制
- 发布 `CollectionModificationEvent(ctx, ...)` 时携带的 `ctx` 可能与 Collection 实际所属的 channel 不匹配
- 这可能导致下游消费者（如搜索插件）使用错误的 channel 上下文处理事件

#### 9.4.2 搜索索引多语言覆盖范围的影响（校正冲突结论）

`CollectionModificationEvent` 的下游是搜索插件的 `updateVariantsById` 流程：

```
CollectionModificationEvent
        │
        ▼
default-search-plugin.ts:181-195 缓冲（debounceTime(50)）
        │
        ▼
ctx = events[0].ctx  ← 取第一个事件的 ctx
        │
        ▼
searchIndexService.updateVariantsById(ctx, ids)
        │
        ▼
Job 队列：type: 'update-variants-by-id'
        │
        ▼
indexer.controller.ts:123-161 updateVariantsById
        │
        ├─ ctx = MutableRequestContext.deserialize(rawContext)
        ├─ getSearchIndexQueryBuilder(ctx, { channels: await this.getAllChannels(ctx), ... })
        └─ saveVariants(ctx, batch)
```

**完整链路的多语言处理分析**：

`getAllChannels(ctx)`（`indexer.controller.ts:333-340`）返回**所有** channel：
```ts
private async getAllChannels(ctx: RequestContext, options?: FindManyOptions<Channel>): Promise<Channel[]> {
    return await this.connection.getRepository(ctx, Channel).find({ ...options, relationLoadStrategy: 'query' });
}
```

`getSearchIndexQueryBuilder`（`indexer.controller.ts:342-381`）查询所有 channel 的 variant：
```ts
where.channels = { id: In(channels.map(c => c.id)) };  // channels = 所有 channel
```

但 `saveVariants`（`indexer.controller.ts:401-515`）的语言处理有问题：

```ts
private async saveVariants(ctx: MutableRequestContext, variants: ProductVariant[]) {
    const originalChannel = ctx.channel;  // Job 恢复时绑定的 channel
    for (const variant of variants) {
        ctx.setChannel(originalChannel);
        
        const availableLanguageCodes = unique(ctx.channel.availableLanguageCodes);  
        // ↑ 关键：用 originalChannel 的 availableLanguageCodes，不是 variant 所属 channel 的！
        
        for (const languageCode of availableLanguageCodes) {  // 遍历 originalChannel 的语言
            for (const channel of variant.channels) {         // 遍历 variant 所属的所有 channel
                for (const currencyCode of availableCurrencyCodes) {
                    const ch = new Channel({ ...channel, defaultCurrencyCode: currencyCode });
                    ctx.setChannel(ch);
                    
                    const item = new SearchIndexItem({
                        channelId: ctx.channelId,      // variant 所属的 channel
                        languageCode,                  // originalChannel 的语言！
                        currencyCode,
                        // ...
                    });
                    items.push(item);
                }
            }
        }
    }
}
```

**问题场景**：

| 配置项 | channel-default（referenceJob 的 ctx） | channel-eu（variant 所属） |
|---|---|---|
| `availableLanguageCodes` | `[en, de]` | `[en, fr]` |

**结果**：
- 为 channel-eu 创建的 `SearchIndexItem` 只有 `en` 和 `de` 两种语言
- `fr` 语言的记录**缺失**！
- 用户在 channel-eu 用法语搜索时，找不到这些变体

**更深层的问题**：`getProductInChannelQueryBuilder` 也只传入了 `ctx.channel`：
```ts
product = await this.getProductInChannelQueryBuilder(ctx, variant.productId, ctx.channel);
```

这意味着加载 product 时也只考虑了 originalChannel，可能导致 product 在其他 channel 的关联信息加载不完整。

**冲突结论校正**：

| 之前的结论（错误） | 校正后的结论（正确） |
|---|---|
| "ctx.languageCode 不影响搜索索引更新的语言覆盖范围" | ✓ 正确，`ctx.languageCode` 本身不直接影响 |
| "referenceJob.ctx.languageCode 固定取值对搜索索引语言上下文更新没有实质性影响" | ✗ 错误，`ctx.channel.availableLanguageCodes` 会严重影响 |
| "搜索索引总是为 channel.availableLanguageCodes 中的所有语言更新记录" | ✗ 错误，只为 `ctx.channel`（referenceJob 的 channel）的语言更新记录 |

**最终结论**：
- `ctx.languageCode` 本身不影响 ✓
- 但是 `ctx.channel.availableLanguageCodes` **会严重影响**——如果 Job 合并时使用的 referenceJob.ctx 指向的 channel 的语言支持不完整，会导致其他 channel 的 variant 缺失部分语言的搜索索引记录
- 这是一个**真实的 bug**，不是"没有实质性影响"

**唯一例外**：`saveSyntheticVariant`（仅当产品没有变体时调用）确实使用 `ctx.languageCode`：
```ts
private async saveSyntheticVariant(ctx: RequestContext, product: Product) {
    const productTranslation = this.getTranslation(product, ctx.languageCode);
    const item = new SearchIndexItem({
        // ...
        languageCode: ctx.languageCode,
    });
}
```

但在 `updateVariantsById` 流程中，产品有变体（我们正在更新变体），所以 `saveSyntheticVariant` 不会被调用——它只在 `updateProductInChannel` 等产品更新流程中调用。

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

方案 C（补充）：修复 `saveVariants` 中的语言遍历逻辑，使用 variant 所属 channel 的 `availableLanguageCodes`：

```ts
// 在 saveVariants 中，内层循环时改用 channel 自己的语言
for (const channel of variant.channels) {
    const channelLanguageCodes = unique(channel.availableLanguageCodes);
    for (const languageCode of channelLanguageCodes) {  // 用 channel 自己的语言
        // ...
    }
}
```

### 9.5 缓冲的激活时机

`SearchJobBufferService`（`search-job-buffer.service.ts:25-29`）在应用启动时根据配置激活：

```ts
onApplicationBootstrap() {
    if (this.bufferUpdates === true) {
        this.jobQueueService.addBuffer(this.searchIndexJobBuffer);
        this.jobQueueService.addBuffer(this.collectionJobBuffer);
    }
}
```

### 9.6 缓冲的 flush 流程

`JobQueueService.flush()` → `JobBufferService.flush()`：
1. 从缓冲存储中取出所有收集的 Job
2. 传入 `JobBuffer.reduce()` 进行合并
3. 将合并后的 Job 提交到实际队列执行

---

## 十、完整数据流图

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
  getEntityOrThrow(ctx, Collection, id) ← 不按 channel 过滤！
        │                                    ⚠️ 但 ctx 与 Collection 所属 channel 可能不匹配
        ▼
  applyCollectionFiltersInternal(collection):
    1. getAncestorFilters(collection)  ← 有 inheritFilters 截断
    2. 合并 [ancestorFilters + collection.filters]  ⚠️ 顺序与预览相反
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
        │                                  ⚠️ ctx.channel.availableLanguageCodes 可能不完整
        │                                  ⚠️ 导致其他 channel 的语言记录缺失
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
  │      ⚠️ 顺序与实际归组相反 → 混合 AND/OR 时语义不同         │
  │   4. listQueryBuilder.build() (带 channelId 限制)            │
  │   5. 返回 PaginatedList<ProductVariant>                      │
  └─────────────────────────────────────────────────────────────┘
```

---

## 十一、已识别问题汇总

| # | 问题 | 位置 | 严重性 | 触发条件 |
|---|---|---|---|---|
| 1 | 预览链路不遵循 inheritFilters 截断 | `collection.service.ts:496-504` | 中 | 预览 inheritFilters=false 的 Collection，前端传 inheritFilters=true |
| 2 | 预览与实际归组的 filter 顺序相反，混合 AND/OR 时语义不同 | `collection.service.ts:495-522` vs `733-758` | 高 | 同一 filterType 既有祖先 filter 又有自身 filter，且 `combineWithAnd` 混合 |
| 3 | 非列表 ID 编码多做 `JSON.stringify` | `configurable-operation-codec.ts:75-76` | 中 | 自定义 Filter 使用 `type:'ID', list:false` 且启用非恒等 EntityIdStrategy |
| 4 | 事务失败后仍返回预期变更 ID | `collection.service.ts:802-830` | 高 | 事务执行失败（如死锁、超时） |
| 5 | JobBuffer 合并吞噬空数组"全量"语义 | `collection-job-buffer.ts:16-18` | 高 | 缓冲中混合"全量"Job 和"指定"Job |
| 6 | JobBuffer 合并错误取 `applyToChangedVariantsOnly` | `collection-job-buffer.ts:27` | 高 | 缓冲中混合 `true` 和 `false` 的 Job |
| 7 | JobBuffer reduce 固定用 referenceJob.ctx 丢失其他 channel 的语言覆盖 | `collection-job-buffer.ts:26` + `indexer.controller.ts:439` | 高 | 不同 channel 的 `availableLanguageCodes` 不同，参考 Job 的 channel 语言支持不完整 |
| 8 | saveVariants 使用 ctx.channel 的语言而非 variant 所属 channel 的语言 | `indexer.controller.ts:439` | 高 | variant 所属 channel 的 `availableLanguageCodes` 超出 ctx.channel 的范围 |

---

## 十二、关键文件索引

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
