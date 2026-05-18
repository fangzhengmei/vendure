# Vendure 插件机制分析

## 一、插件加载顺序

Vendure 的插件加载遵循一个严格的、分阶段的启动流程，确保所有扩展在应用启动前正确配置。

### 1.1 启动引导流程（bootstrap）

插件加载的核心逻辑在 `packages/core/src/bootstrap.ts` 中实现，主要分为以下阶段：

**阶段 1: 预启动配置（preBootstrapConfig）**
```
调用顺序:
1. setConfig(userConfig)          - 初始化用户配置
2. getAllEntities()               - 收集核心实体 + 插件定义的实体
3. setConfig(dbConnectionOptions) - 将实体注册到数据库连接配置
4. runPluginConfigurations()      - 按顺序执行所有插件的 configuration 函数
5. setEntityIdStrategy()          - 应用实体 ID 策略
6. setMoneyStrategy()             - 应用货币策略
7. validateCustomFieldsConfig()   - 验证自定义字段配置
8. registerCustomEntityFields()   - 注册自定义实体字段
9. runEntityMetadataModifiers()   - 运行实体元数据修改器
```

**阶段 2: 插件兼容性检查（checkPluginCompatibility）**
- 遍历所有插件，检查其 `compatibility` 字段与当前 Vendure 版本
- 使用 semver 的 `satisfies` 函数验证版本兼容性
- 不兼容的插件会阻止启动，除非在 `ignoreCompatibilityErrorsForPlugins` 中明确豁免

**阶段 3: NestJS 模块加载**
- `PluginModule.forRoot()` 动态导入所有配置的插件
- 插件作为 NestJS Module 被加载，其 imports、providers、controllers 等被注册到 DI 容器

### 1.2 插件 configuration 函数执行顺序

`runPluginConfigurations()` 函数（bootstrap.ts:365-374）按照插件在配置数组中的顺序依次执行：

```typescript
export async function runPluginConfigurations(config: RuntimeVendureConfig): Promise<RuntimeVendureConfig> {
    for (const plugin of config.plugins) {
        const configFn = getConfigurationFunction(plugin);
        if (typeof configFn === 'function') {
            const result = await configFn(config);
            Object.assign(config, result);
        }
    }
    return config;
}
```

**关键特性：**
- 顺序执行：插件 A 的配置修改会被插件 B 看到
- 可链式修改：后执行的插件可以覆盖先执行插件的配置
- 支持异步：configuration 函数可以是 async

### 1.3 GraphQL Schema 扩展加载顺序

在 `get-final-vendure-schema.ts` 中，Schema 构建遵循以下顺序：

```
1. 加载基础 Schema（从 .graphql 文件）
2. 应用插件 API 扩展（extendSchemaWithPluginApiExtensions）
   - 按插件配置顺序应用 shopApiExtensions / adminApiExtensions
3. 生成 ListOptions 类型
4. 添加自定义字段
5. 生成认证类型
6. 生成错误码枚举
7. 生成权限枚举
```

**重要：** 插件 Schema 扩展在自定义字段之前应用，这意味着自定义字段可以基于插件扩展的类型。

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

**插件使用示例：**
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

**设计意图：**
- GraphQL Resolvers 被动态创建在单独的 Module 中
- 这些 Resolvers 需要依赖插件中定义的服务
- 自动导出确保了这些服务对动态创建的 Resolver Module 可见

### 2.3 动态 GraphQL Resolver 模块

`createDynamicGraphQlModulesForPlugins()`（dynamic-plugin-api.module.ts:15-34）为每个插件的 GraphQL 扩展动态创建模块：

```typescript
export function createDynamicGraphQlModulesForPlugins(apiType: 'shop' | 'admin'): DynamicModule[] {
    return getConfig()
        .plugins.map(plugin => {
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

**依赖注入流程：**
1. 动态模块 imports 原插件模块，继承其所有 providers
2. Resolvers 作为 providers 注册到动态模块
3. NestJS DI 系统自动解析 Resolver 的构造函数依赖

### 2.4 服务注入的生命周期

- **单例模式**：默认情况下，所有服务都是单例
- **请求范围**：可以使用 `@Injectable({ scope: Scope.REQUEST })` 创建请求范围的 provider
- **Transient 范围**：每次注入都创建新实例

**注意：** RequestContext 不是通过 DI 注入的，而是通过 `@Ctx()` 装饰器作为方法参数传递。

---

## 三、数据库扩展的统一管理

Vendure 提供了一套完整的机制来管理插件对数据库 schema 的扩展。

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
    const pluginEntities = getEntitiesFromPlugins(userConfig.plugins);

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

**实体收集流程：**
1. 收集核心实体（coreEntitiesMap）
2. 调用 `getEntitiesFromPlugins()` 收集所有插件的实体
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

**支持两种实体定义方式：**
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

### 3.5 动态实体字段扩展

除了完整的自定义实体，插件还可以通过 `EntityMetadataModifier` 动态修改现有实体：

```typescript
// 在 DefaultSearchPlugin 中动态添加字段
private static addStockColumnsToEntity() {
    const instance = new SearchIndexItem();
    Column({ type: 'boolean', default: true })(instance, 'inStock');
    Column({ type: 'boolean', default: true })(instance, 'productInStock');
}
```

这通过 `runEntityMetadataModifiers()` 在启动时执行。

### 3.6 迁移管理

- 插件定义的实体需要通过 TypeORM 迁移来创建表
- 开发者需要运行 `migration:generate` 来生成包含插件实体的迁移
- 建议插件文档中明确说明需要的迁移步骤

### 3.7 Worker 进程的数据库同步

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

## 四、架构设计总结

### 4.1 核心设计原则

1. **基于 NestJS Module 系统**：插件本质上是增强版的 NestJS Module
2. **元数据驱动**：通过 Reflect metadata 存储和提取插件配置
3. **约定优于配置**：自动导出 providers、标准化的扩展点
4. **分阶段加载**：配置 → 实体 → Schema → 服务，确保依赖顺序正确

### 4.2 插件能力边界

| 能力 | 实现方式 |
|------|----------|
| 配置修改 | `configuration` 函数 |
| 数据库扩展 | `entities` 字段 + TypeORM 实体 |
| GraphQL 扩展 | `shopApiExtensions` / `adminApiExtensions` |
| 业务逻辑 | `providers` + NestJS DI |
| REST API | `controllers` |
| 事件监听 | 注入 `EventBus` 订阅事件 |
| 后台任务 | `JobQueueService` + `JobQueueStrategy` |

### 4.3 关键数据流

```
用户配置 (vendure-config.ts)
    ↓
preBootstrapConfig()
    ├─→ 收集所有实体（核心 + 插件）
    ├─→ 执行插件 configuration 函数
    ├─→ 应用自定义字段和元数据修改器
    ↓
AppModule 加载
    ├─→ PluginModule 导入所有插件
    ├─→ ApiModule 构建 GraphQL Schema（含插件扩展）
    └─→ 动态创建 Resolver 模块
    ↓
NestFactory.create() 启动应用
    └─→ 触发 OnApplicationBootstrap 生命周期钩子
```

### 4.4 版本兼容性管理

- 插件必须声明 `compatibility` 字段（semver 范围）
- 启动时自动验证版本兼容性
- 提供 `ignoreCompatibilityErrorsForPlugins` 作为逃生门
- 未声明兼容性的插件会记录警告日志
