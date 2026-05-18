# Vendure 插件机制分析

## 一、插件加载顺序与时序约束

Vendure 的插件加载遵循一个严格的、分阶段的启动流程，确保所有扩展在应用启动前正确配置。其中**实体注册先于模块加载**是核心时序约束。

### 1.1 启动引导流程（bootstrap）

插件加载的核心逻辑在 `packages/core/src/bootstrap.ts` 中实现，主要分为以下阶段：

**阶段 1: 预启动配置（preBootstrapConfig）**
```
调用顺序:
1. setConfig(userConfig)          - 初始化用户配置
2. getAllEntities(userConfig)     - 从 userConfig.plugins 收集核心实体 + 插件定义的实体
3. setConfig(dbConnectionOptions) - 将实体注册到数据库连接配置
4. runPluginConfigurations()      - 按顺序执行所有插件的 configuration 函数（可修改 config.plugins）
5. setEntityIdStrategy()          - 应用实体 ID 策略
6. setMoneyStrategy()             - 应用货币策略
7. validateCustomFieldsConfig()   - 验证自定义字段配置
8. registerCustomEntityFields()   - 注册自定义实体字段
9. runEntityMetadataModifiers()   - 运行实体元数据修改器
```

**阶段 2: 插件兼容性检查（checkPluginCompatibility）**
- 在 `preBootstrapConfig` 完成之后执行
- 遍历 `config.plugins`（经过 configuration 函数可能修改后的最终列表）
- 检查每个插件的 `compatibility` 字段与当前 Vendure 版本
- 使用 semver 的 `satisfies` 函数验证版本兼容性
- 不兼容的插件会阻止启动，除非在 `ignoreCompatibilityErrorsForPlugins` 中明确豁免

**阶段 3: NestJS 模块加载**
- `PluginModule.forRoot()` 动态导入 `getConfig().plugins` 中的所有插件
- `createDynamicGraphQlModulesForPlugins()` 遍历 `getConfig().plugins` 创建 Resolver 模块
- `extendSchemaWithPluginApiExtensions()` 使用 `config.plugins` 应用 Schema 扩展
- 插件作为 NestJS Module 被加载，其 imports、providers、controllers 等被注册到 DI 容器
- ⚠️ **重要时序约束**：此时所有实体已注册完成，模块加载时可安全依赖数据库 schema

### 1.2 config.plugins 的阶段化影响

**⚠️ 核心结论**：`config.plugins` 在不同阶段有不同的来源，插件 configuration 函数对其修改的影响范围有明确边界。

| 阶段 | 使用的 plugins 来源 | configuration 函数修改 config.plugins 是否生效 |
|------|-------------------|-----------------------------------------------|
| **实体收集阶段** | `userConfig.plugins`（原始配置） | ❌ **不生效** - 实体在 runPluginConfigurations 之前收集 |
| **兼容性检查阶段** | `config.plugins`（最终配置） | ✅ **生效** - 检查的是经过修改后的最终列表 |
| **模块加载阶段** | `getConfig().plugins`（最终配置） | ✅ **生效** - 加载的是经过修改后的最终列表 |

**代码证据链**：

```typescript
// 证据 1：实体收集使用原始 userConfig.plugins
// bootstrap.ts:379-381
export function getAllEntities(userConfig: Partial<VendureConfig>): Array<Type<any>> {
    const coreEntities = Object.values(coreEntitiesMap) as Array<Type<any>>;
    const pluginEntities = getEntitiesFromPlugins(userConfig.plugins);  // ✅ 原始 userConfig
    // ...
}

// 证据 2：runPluginConfigurations 可修改 config.plugins
// bootstrap.ts:365-373
export async function runPluginConfigurations(config: RuntimeVendureConfig): Promise<RuntimeVendureConfig> {
    for (const plugin of config.plugins) {
        const configFn = getConfigurationFunction(plugin);
        if (typeof configFn === 'function') {
            const result = await configFn(config);
            Object.assign(config, result);  // ✅ 浅拷贝合并，可修改 plugins
        }
    }
    return config;
}

// 证据 3：兼容性检查使用修改后的 config.plugins
// bootstrap.ts:328-332
function checkPluginCompatibility(config: RuntimeVendureConfig, ...): void {
    for (const plugin of config.plugins) {  // ✅ 最终 config
        // ... 检查兼容性
    }
}

// 证据 4：模块加载使用最终配置
// plugin.module.ts:14-19
export class PluginModule {
    static forRoot(): DynamicModule {
        return {
            module: PluginModule,
            imports: [...getConfig().plugins],  // ✅ 最终 config
        };
    }
}

// dynamic-plugin-api.module.ts:15-17
export function createDynamicGraphQlModulesForPlugins(apiType: 'shop' | 'admin'): DynamicModule[] {
    return getConfig().plugins.map(plugin => {  // ✅ 最终 config
        // ... 创建动态模块
    });
}

// get-final-vendure-schema.ts:99, 126
schema = extendSchemaWithPluginApiExtensions(schema, config.plugins, apiType);  // ✅ 最终 config
getPluginAPIExtensions(plugins, apiType)  // ✅ 接收最终 plugins
```

**实际影响场景**：
- 如果插件 A 的 configuration 函数向 `config.plugins` 中添加了新插件 B
  - ✅ 插件 B 会被进行兼容性检查
  - ✅ 插件 B 会作为 NestJS Module 被加载
  - ❌ 插件 B 定义的实体不会被收集（因为实体收集在 configuration 之前执行）
- 如果插件 A 的 configuration 函数从 `config.plugins` 中移除了插件 B
  - ✅ 插件 B 不会被进行兼容性检查
  - ✅ 插件 B 不会作为 NestJS Module 被加载
  - ❌ 插件 B 定义的实体已经被收集，仍然会存在于数据库 schema 中

### 1.3 实体注册先于模块加载的时序约束

```
时序图:
preBootstrapConfig(userConfig)
    ↓
getAllEntities(userConfig)  →  从 userConfig.plugins 收集实体（不可再改变）
    ↓
setConfig(dbConnectionOptions.entities)  →  实体注册到 TypeORM
    ↓
runPluginConfigurations(config)  →  可修改 config.plugins（影响后续阶段）
    ↓
registerCustomEntityFields()  →  自定义字段注册
    ↓
runEntityMetadataModifiers()  →  实体元数据修改完成
    ↓
preBootstrapConfig 返回 config  →  ✅ 实体层完全就绪
    ↓
checkPluginCompatibility(config.plugins)  →  使用最终 plugins 列表
    ↓
import('./app.module.js')  →  模块加载开始
    ↓
PluginModule.forRoot()  →  从 getConfig().plugins 加载插件为 NestJS Module
    ↓
createDynamicGraphQlModulesForPlugins()  →  从 getConfig().plugins 创建 Resolver 模块
    ↓
NestFactory.create()  →  DI 容器初始化完成
```

**设计意图**：
- 确保 TypeORM 连接建立时，所有实体元数据已就绪
- 避免模块加载时因实体未注册导致的运行时错误
- 插件的 configuration 函数可以动态增删插件，但对实体集合的影响有明确边界

### 1.4 插件 configuration 函数执行顺序

`runPluginConfigurations()` 函数（bootstrap.ts:365-374）按照插件在配置数组中的顺序依次执行：

```typescript
export async function runPluginConfigurations(config: RuntimeVendureConfig): Promise<RuntimeVendureConfig> {
    for (const plugin of config.plugins) {
        const configFn = getConfigurationFunction(plugin);
        if (typeof configFn === 'function') {
            const result = await configFn(config);
            Object.assign(config, result);  // 浅拷贝合并
        }
    }
    return config;
}
```

**关键特性**：
- 顺序执行：插件 A 的配置修改会被插件 B 看到
- 可链式修改：后执行的插件可以覆盖先执行插件的配置
- 支持异步：configuration 函数可以是 async
- 执行时机：在实体注册之后、模块加载之前
- 可修改 `config.plugins`：影响后续的兼容性检查和模块加载

### 1.5 GraphQL Schema 扩展加载顺序

在 `get-final-vendure-schema.ts` 中，Schema 构建遵循以下顺序：

```
1. 加载基础 Schema（从 .graphql 文件）
2. 应用插件 API 扩展（extendSchemaWithPluginApiExtensions）
   - 遍历 config.plugins（最终列表）按顺序应用扩展
3. 生成 ListOptions 类型
4. 添加自定义字段
5. 生成认证类型
6. 生成错误码枚举
7. 生成权限枚举
```

**重要**：插件 Schema 扩展在自定义字段之前应用，这意味着自定义字段可以基于插件扩展的类型。

---

## 二、对核心服务的依赖注入

Vendure 插件的依赖注入基于 NestJS 的 DI 系统，但增加了特定的约定和辅助模块。

### 2.1 PluginCommonModule：核心服务注入桥梁

`PluginCommonModule`（plugin-common.module.ts）是插件访问核心服务的标准入口，它导出以下模块：

| 模块 | 提供的服务 |
|------|------------|
| `EventBusModule` | EventBus - 事件总线 |
| `ConfigModule` | ConfigService - 配置服务 |
| `ConnectionModule` | 数据库连接 |
| `ServiceModule` | 所有实体服务（ProductService, OrderService 等） |
| `JobQueueModule` | JobQueueService - 作业队列 |
| `HealthCheckModule` | HealthCheckRegistryService - 健康检查 |
| `CacheModule` | 缓存服务 |
| `I18nModule` | 国际化服务 |
| `ProcessContextModule` | 进程上下文 |
| `DataImportModule` | 数据导入服务 |

**插件使用示例**：
```typescript
@VendurePlugin({
    imports: [PluginCommonModule],
    // ...
})
export class MyPlugin {
    // 可以直接注入核心服务
    constructor(private productService: ProductService) {}
}
```

### 2.2 自动导出 Providers

在 `VendurePlugin` 装饰器（vendure-plugin.ts:164-192）中，有一个重要的自动机制：

```typescript
// 自动将插件的 providers 添加到 exports 数组
const exportedProviders = (nestModuleMetadata.providers || []).filter(provider => {
    // 排除全局 provider（APP_INTERCEPTOR, APP_FILTER 等）
    if (isNamedProvider(provider)) {
        if (nestGlobalProviderTokens.includes(provider.provide as any)) {
            return false;
        }
    }
    return true;
});
nestModuleMetadata.exports = [...(nestModuleMetadata.exports || []), ...exportedProviders];
```

**设计意图**：
- GraphQL Resolvers 被动态创建在单独的 Module 中
- 这些 Resolvers 需要依赖插件中定义的服务
- 自动导出确保了这些服务对动态创建的 Resolver Module 可见

### 2.3 动态 GraphQL Resolver 模块

`createDynamicGraphQlModulesForPlugins()`（dynamic-plugin-api.module.ts:15-34）为每个插件的 GraphQL 扩展动态创建模块：

```typescript
export function createDynamicGraphQlModulesForPlugins(apiType: 'shop' | 'admin'): DynamicModule[] {
    return getConfig()
        .plugins.map(plugin => {  // ✅ 使用最终 config.plugins
            const pluginModule = isDynamicModule(plugin) ? plugin.module : plugin;
            const resolvers = graphQLResolversFor(plugin, apiType) || [];

            if (resolvers.length) {
                return {
                    module: DynamicModuleClass,
                    imports: [pluginModule, ...imports],
                    providers: [...resolvers],
                };
            }
        })
        .filter(notNullOrUndefined);
}
```

**依赖注入流程**：
1. 动态模块 imports 原插件模块，继承其所有 providers
2. Resolvers 作为 providers 注册到动态模块
3. NestJS DI 系统自动解析 Resolver 的构造函数依赖

### 2.4 服务注入的生命周期

- **单例模式**：默认情况下，所有服务都是单例
- **请求范围**：可以使用 `@Injectable({ scope: Scope.REQUEST })` 创建请求范围的 provider
- **Transient 范围**：每次注入都创建新实例

**注意**：RequestContext 不是通过 DI 注入的，而是通过 `@Ctx()` 装饰器作为方法参数传递。

---

## 三、数据库扩展的统一管理

Vendure 提供了**两条独立的实体元数据扩展路径**，以及明确的配置函数影响边界。

### 3.1 实体扩展机制

插件通过 `entities` 字段定义自定义实体：

```typescript
@VendurePlugin({
    entities: [SearchIndexItem],  // 直接数组
    // 或函数形式（支持条件性加载）
    entities: () => DefaultJobQueuePlugin.options.useDatabaseForBuffer === true
        ? [JobRecord, JobRecordBuffer]
        : [JobRecord],
})
```

### 3.2 实体收集与去重

`getAllEntities()`（bootstrap.ts:379-395）统一管理所有实体：

```typescript
export function getAllEntities(userConfig: Partial<VendureConfig>): Array<Type<any>> {
    const coreEntities = Object.values(coreEntitiesMap) as Array<Type<any>>;
    const pluginEntities = getEntitiesFromPlugins(userConfig.plugins);  // ✅ 原始 userConfig

    const allEntities: Array<Type<any>> = coreEntities;

    // 检查实体名称冲突
    for (const pluginEntity of pluginEntities) {
        if (allEntities.find(e => e.name === pluginEntity.name)) {
            throw new InternalServerError('error.entity-name-conflict', { entityName: pluginEntity.name });
        } else {
            allEntities.push(pluginEntity);
        }
    }
    return allEntities;
}
```

**⚠️ 关键证据**：`getAllEntities(userConfig)` 的参数是原始的 `userConfig`，而不是经过 `runPluginConfigurations` 修改后的 `config`。

**实体收集流程**：
1. 收集核心实体（coreEntitiesMap）
2. 调用 `getEntitiesFromPlugins(userConfig.plugins)` 收集所有插件的实体
3. 按实体名称检查冲突
4. 合并后返回完整实体数组

### 3.3 实体元数据提取

`getEntitiesFromPlugins()`（plugin-metadata.ts:17-27）从插件元数据中提取实体：

```typescript
export function getEntitiesFromPlugins(plugins?: Array<Type<any> | DynamicModule>): Array<Type<any>> {
    if (!plugins) {
        return [];
    }
    return plugins
        .map(p => reflectMetadata(p, PLUGIN_METADATA.ENTITIES))
        .reduce((all, entities) => {
            const resolvedEntities = typeof entities === 'function' ? entities() : (entities ?? []);
            return [...all, ...resolvedEntities];
        }, []);
}
```

**支持两种实体定义方式**：
- 静态数组：`entities: [Entity1, Entity2]`
- 函数形式：`entities: () => [Entity1, Entity2]`（支持条件逻辑）

### 3.4 实体注册到 TypeORM

收集到的实体在 `preBootstrapConfig()` 中被注册到数据库连接配置：

```typescript
const entities = getAllEntities(userConfig);
await setConfig({
    dbConnectionOptions: {
        entities,  // 所有实体（核心 + 插件）
        subscribers: [...],
    },
});
```

### 3.5 两条实体元数据扩展路径对比

Vendure 提供了两条独立的实体元数据扩展路径，二者在调用时机、作用范围和使用方式上有本质区别：

| 特性 | 插件 init() 直接修改 | EntityMetadataModifier 统一修改器 |
|------|---------------------|----------------------------------|
| **触发时机** | 用户配置阶段调用 `Plugin.init()` 时 | `preBootstrapConfig` 后期统一调用 |
| **配置位置** | 插件静态方法内部 | `config.entityOptions.metadataModifiers` 数组 |
| **作用范围** | 通常只修改插件自己定义的实体 | 可以修改任何实体（包括核心实体） |
| **实现方式** | 直接调用 TypeORM 装饰器（`@Column` 等） | 操作 TypeORM `MetadataArgsStorage` |
| **执行顺序** | 最早（用户代码加载时） | 最晚（preBootstrapConfig 第 9 步） |
| **典型用途** | 根据插件选项条件性添加字段 | 添加索引、修改列类型等全局调整 |

#### 路径 1：插件 init() 直接修改实体元数据

**代码证据**（default-search-plugin.ts:107-116, 241-256）：
```typescript
static init(options: DefaultSearchPluginInitOptions): Type<DefaultSearchPlugin> {
    this.options = options;
    if (options.indexStockStatus === true) {
        this.addStockColumnsToEntity();  // ✅ 立即修改元数据
    }
    if (options.indexCurrencyCode) {
        this.addCurrencyCodeToEntity();  // ✅ 立即修改元数据
    }
    return DefaultSearchPlugin;
}

private static addStockColumnsToEntity() {
    const instance = new SearchIndexItem();
    // 直接调用 TypeORM 装饰器修改全局 MetadataArgsStorage
    Column({ type: 'boolean', default: true })(instance, 'inStock');
    Column({ type: 'boolean', default: true })(instance, 'productInStock');
}
```

**工作原理**：
- TypeORM 的装饰器（`@Column`, `@Index` 等）在被调用时会直接修改全局的 `getMetadataArgsStorage()`
- `DefaultSearchPlugin.init()` 在用户配置文件中被调用时（如 `plugins: [DefaultSearchPlugin.init({...})]`），立即执行字段添加
- 这个调用发生在 `preBootstrapConfig` 之前，在用户代码加载阶段

**典型应用场景**：
- 插件根据用户传入的选项条件性地扩展自己的实体
- 避免破坏向后兼容性的 schema 变更

#### 路径 2：EntityMetadataModifier 统一元数据修改器

**代码证据**（entity-metadata-modifier.ts, run-entity-metadata-modifiers.ts）：
```typescript
// 类型定义
export type EntityMetadataModifier = (metadata: MetadataArgsStorage) => void | Promise<void>;

// 执行时机
export async function runEntityMetadataModifiers(config: VendureConfig) {
    if (config.entityOptions?.metadataModifiers?.length) {
        const metadataArgsStorage = getMetadataArgsStorage();
        for (const modifier of config.entityOptions.metadataModifiers) {
            await modifier(metadataArgsStorage);  // ✅ 统一调用
        }
    }
}
```

**使用方式**（在 vendure-config.ts 中配置）：
```typescript
export const config: VendureConfig = {
    entityOptions: {
        metadataModifiers: [
            // 为 ProductVariant.sku 添加唯一索引
            (metadata) => {
                const instance = new ProductVariant();
                Index({ unique: true })(instance, 'sku');
            },
            // 修改 ProductTranslation.description 的列类型
            (metadata) => {
                const descriptionColumnIndex = metadata.columns.findIndex(
                    col => col.propertyName === 'description' && col.target === ProductTranslation,
                );
                if (-1 < descriptionColumnIndex) {
                    metadata.columns.splice(descriptionColumnIndex, 1);
                    const instance = new ProductTranslation();
                    Column({ type: 'mediumtext' })(instance, 'description');
                }
            }
        ]
    }
};
```

**工作原理**：
- 在 `preBootstrapConfig` 的最后阶段（第 9 步）统一调用
- 接收 TypeORM 的 `MetadataArgsStorage` 作为参数，可以直接操作元数据
- 可以修改任何实体的元数据，包括 Vendure 核心实体

**典型应用场景**：
- 为核心实体添加数据库索引
- 修改列的数据类型（如将 text 改为 mediumtext）
- 全局的 schema 调整

### 3.6 配置函数对实体集合的影响边界

**⚠️ 证据化结论**：插件 configuration 函数对实体集合的影响有明确边界，基于以下代码证据：

**证据 1：实体收集使用原始 userConfig**
```typescript
// bootstrap.ts:294
const entities = getAllEntities(userConfig);  // ✅ 使用原始 userConfig
```

`getAllEntities()` 的参数是传入 `preBootstrapConfig` 的原始 `userConfig`，此时 `runPluginConfigurations` 尚未执行。

**证据 2：实体注册后 configuration 才执行**
```typescript
// bootstrap.ts:294-312
const entities = getAllEntities(userConfig);       // 第 2 步：收集实体
await setConfig({ dbConnectionOptions: { entities } }); // 第 3 步：注册实体
// ...
config = await runPluginConfigurations(config);   // 第 4 步：执行配置函数（在实体注册之后！）
```

**证据 3：配置函数通过 Object.assign 浅合并**
```typescript
// bootstrap.ts:369-370
const result = await configFn(config);
Object.assign(config, result);  // ✅ 浅拷贝合并
```

**影响边界总结表**：

| 操作 | 是否可行 | 说明 |
|------|---------|------|
| 修改 `config.plugins` 增删插件 | ✅ 部分生效 | 影响兼容性检查和模块加载，但不影响已收集的实体 |
| 修改 `config.dbConnectionOptions.entities` | ✅ 有效 | 直接替换实体数组，后续 TypeORM 连接会使用修改后的列表 |
| 通过 `config.entityOptions.metadataModifiers` 添加修改器 | ✅ 有效 | 修改器在后续的 `runEntityMetadataModifiers` 中执行 |
| 添加新的实体类到 `entities` 数组 | ✅ 条件可行 | 必须确保实体类已导入且元数据已注册 |

**注意事项**：
- 虽然技术上可以修改 `dbConnectionOptions.entities`，但这不是推荐的做法
- 推荐方式是在插件的 `entities` 元数据中声明实体，让系统统一收集
- 修改 `entities` 数组可能导致实体名称冲突检查被绕过
- 在 configuration 函数中增删插件时，新增插件的实体会缺失，被删插件的实体会残留

### 3.7 迁移命令复用预启动配置管线

所有迁移命令（`runMigrations`、`generateMigration`、`revertLastMigration`）都复用同一套预启动配置管线，确保迁移时看到的实体 schema 与运行时完全一致。

**migrate.ts 中的实现**：

```typescript
// 运行迁移
export async function runMigrations(userConfig: Partial<VendureConfig>): Promise<string[]> {
    const config = await preBootstrapConfig(userConfig);  // ✅ 复用预启动配置
    const connection = await createConnection(createConnectionOptions(config));
    // ... 执行迁移
}

// 生成迁移
export async function generateMigration(
    userConfig: Partial<VendureConfig>,
    options: MigrationOptions,
): Promise<string | undefined> {
    const config = await preBootstrapConfig(userConfig);  // ✅ 复用预启动配置
    const connection = await createConnection(createConnectionOptions(config));
    // ... 生成迁移文件
}

// 回滚迁移
export async function revertLastMigration(userConfig: Partial<VendureConfig>) {
    const config = await preBootstrapConfig(userConfig);  // ✅ 复用预启动配置
    const connection = await createConnection(createConnectionOptions(config));
    // ... 回滚迁移
}
```

**复用 preBootstrapConfig 带来的一致性保证**：

| 阶段 | 迁移时执行 | 运行时执行 | 作用 |
|------|-----------|-----------|------|
| getAllEntities() | ✅ | ✅ | 从 userConfig.plugins 收集核心 + 插件实体 |
| runPluginConfigurations() | ✅ | ✅ | 应用插件配置修改（可修改 config.plugins） |
| registerCustomEntityFields() | ✅ | ✅ | 注册自定义字段 |
| runEntityMetadataModifiers() | ✅ | ✅ | 应用实体元数据修改 |

**设计优势**：
1. **Schema 一致性**：迁移生成的 schema 与运行时完全一致，避免"在我机器上能跑"问题
2. **插件友好**：插件定义的实体会被自动包含在迁移中
3. **配置感知**：插件 configuration 函数对配置的修改会影响迁移
4. **单一真相源**：只有一套实体构建逻辑，避免重复代码

**迁移工作流**：
```
1. 开发者添加插件或修改自定义字段配置
2. 运行 `vendure migration:generate -n MyChange`
   → 内部调用 preBootstrapConfig()
   → 从 userConfig.plugins 收集所有实体（含插件实体）
   → 比较数据库 schema 与实体元数据
   → 生成迁移脚本
3. 运行 `vendure migration:run`
   → 再次调用 preBootstrapConfig()
   → 确保 TypeORM 连接使用完整的实体配置
   → 执行迁移
```

### 3.8 Worker 进程的数据库同步

在 Worker 启动时，有特殊的数据库表验证逻辑（validateDbTablesForWorker）：

```typescript
// Worker 启动时检查表结构是否存在
// 防止 Server 和 Worker 并发启动时的竞态条件
async function validateDbTablesForWorker(worker: INestApplicationContext) {
    const connection: Connection = worker.get(getConnectionToken());
    // 轮询检查 Administrator 表是否存在数据
    // 最多重试 10 次，每次间隔 5 秒
}
```

---

## 四、兼容性规则

### 4.1 兼容性检查机制

`checkPluginCompatibility()`（bootstrap.ts:328-360）的执行时机和规则：

```typescript
function checkPluginCompatibility(
    config: RuntimeVendureConfig,
    ignoredPlugins: Array<DynamicModule | Type<any>> = [],
): void {
    for (const plugin of config.plugins) {  // ✅ 使用最终 config.plugins
        const compatibility = getCompatibility(plugin);
        const pluginName = (plugin as any).name as string;
        if (!compatibility) {
            Logger.info(
                `The plugin "${pluginName}" does not specify a compatibility range, so it is not guaranteed to be compatible with this version of Vendure.`,
            );
        } else {
            // 使用 semver.satisfies 验证
            // 参数: { loose: true, includePrerelease: true }
            if (!satisfies(VENDURE_VERSION, compatibility, { loose: true, includePrerelease: true })) {
                // ... 处理不兼容
            }
        }
    }
}
```

### 4.2 执行时机

**⚠️ 关键事实**：兼容性检查在 `preBootstrapConfig` **之后**执行，检查的是 `config.plugins`（经过 configuration 函数可能修改后的列表）。

**bootstrap 函数中的实际顺序**：
```typescript
export async function bootstrap(
    userConfig: Partial<VendureConfig>,
    options?: BootstrapOptions,
): Promise<INestApplication> {
    const config = await preBootstrapConfig(userConfig);  // 1. 预启动配置（内部可能修改 config.plugins）
    Logger.useLogger(config.logger);
    Logger.info(`Bootstrapping Vendure Server...`);
    checkPluginCompatibility(config, options?.ignoreCompatibilityErrorsForPlugins);  // 2. 检查最终 config.plugins
    
    // ... 后续模块加载
}
```

**设计原因**：
- `preBootstrapConfig` 中的 `runPluginConfigurations()` 可能会动态修改 plugins 数组
- 兼容性检查需要看到最终的插件列表

### 4.3 兼容性规则详解

| 场景 | 行为 |
|------|------|
| 插件未定义 `compatibility` | 记录 info 日志，继续加载 |
| 插件定义 `compatibility` 且版本兼容 | 无日志，正常加载 |
| 插件定义 `compatibility` 但版本不兼容 | 抛出错误，阻止启动 |
| 不兼容但在 `ignoreCompatibilityErrorsForPlugins` 中 | 记录 warn 日志，继续加载 |

**semver 验证参数说明**：
- `loose: true`：允许宽松的版本号解析
- `includePrerelease: true`：预发布版本（如 `3.0.0-beta.1`）可以匹配正常范围

---

## 五、架构设计总结

### 5.1 核心设计原则

1. **基于 NestJS Module 系统**：插件本质上是增强版的 NestJS Module
2. **元数据驱动**：通过 Reflect metadata 存储和提取插件配置
3. **约定优于配置**：自动导出 providers、标准化的扩展点
4. **分阶段加载**：配置 → 实体 → Schema → 服务，确保依赖顺序正确
5. **时序强约束**：实体注册先于模块加载，保证 TypeORM 连接安全
6. **多路径扩展**：提供插件 init() 和统一修改器两条实体元数据扩展路径
7. **阶段化插件列表**：`userConfig.plugins` 用于实体收集，`config.plugins` 用于兼容性检查和模块加载

### 5.2 插件能力边界

| 能力 | 实现方式 |
|------|----------|
| 配置修改 | `configuration` 函数 |
| 数据库扩展（新表） | `entities` 字段 + TypeORM 实体 |
| 数据库扩展（修改自有实体） | 插件 `init()` 方法 + TypeORM 装饰器 |
| 数据库扩展（修改任意实体） | `config.entityOptions.metadataModifiers` |
| GraphQL 扩展 | `shopApiExtensions` / `adminApiExtensions` |
| 业务逻辑 | `providers` + NestJS DI |
| REST API | `controllers` |
| 事件监听 | 注入 `EventBus` 订阅事件 |
| 后台任务 | `JobQueueService` + `JobQueueStrategy` |
| 动态增删插件 | configuration 函数中修改 `config.plugins` |

### 5.3 关键数据流

```
用户配置 (vendure-config.ts)
    ↓
Plugin.init() 被调用  →  [可选] 插件直接修改自有实体元数据
    ↓
bootstrap(userConfig)
    ↓
preBootstrapConfig(userConfig)
    ├─→ getAllEntities(userConfig) 从 userConfig.plugins 收集所有实体（快照）
    ├─→ setConfig() 注册实体到 TypeORM
    ├─→ runPluginConfigurations(config) 执行插件配置函数
    │   ├─→ 可修改 config.plugins（影响后续阶段）
    │   ├─→ 可修改 config.dbConnectionOptions.entities
    │   └─→ 可添加/修改 config.entityOptions.metadataModifiers
    ├─→ registerCustomEntityFields() 注册自定义字段
    └─→ runEntityMetadataModifiers() 应用统一元数据修改器
    ↓
checkPluginCompatibility(config.plugins) 验证最终列表的版本兼容性
    ↓
import('./app.module.js') 模块加载开始
    ├─→ PluginModule.forRoot() 从 getConfig().plugins 加载插件为 NestJS Module
    ├─→ createDynamicGraphQlModulesForPlugins() 从 getConfig().plugins 创建 Resolver 模块
    └─→ extendSchemaWithPluginApiExtensions() 从 config.plugins 应用 Schema 扩展
    ↓
NestFactory.create() 启动应用
    └─→ 触发 OnApplicationBootstrap 生命周期钩子
```

### 5.4 预启动配置管线的复用

`preBootstrapConfig()` 是 Vendure 架构的核心抽象，被多个入口复用：

| 入口 | 调用 preBootstrapConfig | 用途 |
|------|------------------------|------|
| `bootstrap()` | ✅ | 启动 Server |
| `bootstrapWorker()` | ✅ | 启动 Worker |
| `runMigrations()` | ✅ | 执行数据库迁移 |
| `generateMigration()` | ✅ | 生成迁移脚本 |
| `revertLastMigration()` | ✅ | 回滚迁移 |

**这种设计确保了**：
- 所有场景下的实体 schema 完全一致
- 插件配置在所有入口点生效
- 自定义字段和元数据修改器统一应用
- 迁移生成的 schema 与运行时无差异

### 5.5 实体元数据扩展的时序总览

```
用户代码加载阶段
    ↓
vendure-config.ts 执行
    ├─→ DefaultSearchPlugin.init({ indexStockStatus: true })
    │   └─→ addStockColumnsToEntity() 立即修改 MetadataArgsStorage
    └─→ 配置 entityOptions.metadataModifiers 数组
    ↓
preBootstrapConfig 阶段
    ├─→ getAllEntities(userConfig) 收集实体类引用（内部调用 getEntitiesFromPlugins(userConfig.plugins)
    ├─→ registerCustomEntityFields() 添加自定义字段
    └─→ runEntityMetadataModifiers() 执行统一修改器
    ↓
TypeORM 连接建立
    └─→ MetadataArgsStorage 被转换为数据库 schema
```

**关键洞察**：三条路径在不同时间点操作同一个 `MetadataArgsStorage`，最终共同决定数据库 schema。
