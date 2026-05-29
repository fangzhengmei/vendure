# Vendure 后台扩展注册链路分析

本文档深入分析 Vendure 后台管理系统（Admin UI / Dashboard）的扩展注册机制，包括插件声明、路由挂载与权限映射之间的衔接关系。

## 目录

- [1. 整体架构概览](#1-整体架构概览)
- [2. 插件声明层（服务端）](#2-插件声明层服务端)
  - [2.1 VendurePlugin 装饰器](#21-vendureplugin-装饰器)
  - [2.2 两种 UI 扩展声明方式](#22-两种-ui-扩展声明方式)
- [3. 编译时扩展收集](#3-编译时扩展收集)
  - [3.1 Vite 插件与虚拟模块](#31-vite-插件与虚拟模块)
  - [3.2 Angular Admin UI 编译流程](#32-angular-admin-ui-编译流程)
- [4. 运行时注册层](#4-运行时注册层)
  - [4.1 GlobalRegistry 全局注册表](#41-globalregistry-全局注册表)
  - [4.2 defineDashboardExtension 入口函数](#42-definedashboardextension-入口函数)
  - [4.3 各类型扩展注册逻辑](#43-各类型扩展注册逻辑)
- [5. 路由挂载流程](#5-路由挂载流程)
  - [5.1 React Dashboard 路由系统](#51-react-dashboard-路由系统)
  - [5.2 Angular Admin UI 路由系统](#52-angular-admin-ui-路由系统)
- [6. 权限映射机制](#6-权限映射机制)
  - [6.1 导航菜单权限控制](#61-导航菜单权限控制)
  - [6.2 路由级权限控制](#62-路由级权限控制)
  - [6.3 PermissionsService 权限服务](#63-permissionsservice-权限服务)
- [7. 完整链路时序图](#7-完整链路时序图)
- [8. 权限约束链路深度追踪](#8-权限约束链路深度追踪)
  - [8.1 全链路总览](#81-全链路总览)
  - [8.2 阶段 1：服务端声明 — 权限的诞生](#82-阶段-1服务端声明--权限的诞生)
  - [8.3 阶段 2：前端权限获取 — 用户有什么权限](#83-阶段-2前端权限获取--用户有什么权限)
  - [8.4 阶段 3：扩展声明中的权限字段 — 传递给注册表](#84-阶段-3扩展声明中的权限字段--传递给注册表)
  - [8.5 阶段 4：判定点 — 权限在哪生效](#85-阶段-4判定点--权限在哪生效)
  - [8.6 权限字段流转汇总表](#86-权限字段流转汇总表)
  - [8.7 最小排障清单](#87-最小排障清单)
- [10. 关键文件索引](#10-关键文件索引)
- [11. 总结](#11-总结)

---

## 1. 整体架构概览

Vendure 后台扩展系统采用"**声明 → 收集 → 注册 → 挂载**"的四层架构：

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  插件声明层     │────▶│  编译时收集     │────▶│  运行时注册     │────▶│   路由挂载层    │
│  (服务端)       │     │  (Vite/Webpack) │     │  (浏览器)       │     │  (路由系统)     │
└─────────────────┘     └─────────────────┘     └─────────────────┘     └─────────────────┘
       │                        │                        │                        │
       ▼                        ▼                        ▼                        ▼
┌──────────────┐      ┌──────────────────┐     ┌────────────────────┐    ┌──────────────┐
│ VendurePlugin│      │ dashboardMetadata│     │ defineDashboardExt │    │ TanStack     │
│ 装饰器        │      │ Plugin (Vite)    │     │ globalRegistry     │    │ Router /     │
│ uiExtensions │      │ 虚拟模块生成     │     │ 各类型注册函数     │    │ Angular Router│
└──────────────┘      └──────────────────┘     └────────────────────┘    └──────────────┘
```

**关键技术点**：
- 服务端与客户端通过静态属性或元数据解耦
- 编译时通过 Vite 插件动态收集扩展
- 运行时通过单例注册表统一管理
- 路由系统动态挂载扩展路由

---

## 2. 插件声明层（服务端）

### 2.1 VendurePlugin 装饰器

**文件位置**：`packages/core/src/plugin/vendure-plugin.ts:164-192`

VendurePlugin 是扩展系统的入口装饰器，基于 NestJS Module 扩展，支持以下 UI 相关元数据：

```typescript
export interface VendurePluginMetadata extends ModuleMetadata {
    // Dashboard 扩展（React 新版）
    dashboard?: DashboardExtension;
    
    // 其他元数据...
    configuration?: PluginConfigurationFn;
    adminApiExtensions?: APIExtensionDefinition;
    entities?: Array<Type<any>>;
    compatibility?: string;
}

// DashboardExtension 类型定义
export type DashboardExtension =
    | string                     // 直接指定入口文件路径
    | { location: string };      // 指定入口文件位置
```

**工作原理**：
1. 使用 `Reflect.defineMetadata` 将元数据绑定到类
2. 自动导出 providers 供 GraphQL 解析器使用
3. 服务端启动时扫描所有插件元数据

### 2.2 两种 UI 扩展声明方式

#### 方式一：Angular Admin UI - 静态属性声明

**文件位置**：`packages/dev-server/example-plugins/product-bundles/product-bundles.plugin.ts:42-47`

```typescript
@VendurePlugin({ /* ... 服务端配置 ... */ })
export class ProductBundlesPlugin {
    // 静态属性方式声明 UI 扩展
    static uiExtensions: AdminUiExtension = {
        id: 'product-bundles',
        extensionPath: path.join(__dirname, 'ui'),
        routes: [{ route: 'product-bundles', filePath: 'routes.ts' }],
        providers: ['providers.ts'],
    };
}
```

**AdminUiExtension 核心字段**（`packages/ui-devkit/src/compiler/types.ts:130-264`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `string` | 扩展唯一标识，未指定时生成 hash |
| `extensionPath` | `string` | UI 扩展代码目录路径 |
| `routes` | `UiExtensionRouteDefinition[]` | 路由定义，指定路径和路由文件 |
| `providers` | `string[]` | 共享 providers 文件路径 |
| `ngModules` | `AdminUiExtensionModule[]` | **已废弃**，Angular 模块声明 |
| `translations` | `object` | 多语言翻译文件 glob |

#### 方式二：React Dashboard - dashboard 属性声明

在插件装饰器中直接指定 Dashboard 扩展入口：

```typescript
@VendurePlugin({
    dashboard: './ui/dashboard.tsx',  // 相对路径
    // 或
    dashboard: { location: './ui/dashboard.tsx' },
})
export class MyPlugin {}
```

---

## 3. 编译时扩展收集

### 3.1 Vite 插件与虚拟模块

**文件位置**：`packages/dashboard/vite/vite-plugin-dashboard-metadata.ts:16-73`

React Dashboard 使用 Vite 插件在编译时动态收集扩展，核心是**虚拟模块**技术。

#### 工作流程

```
1. 插件初始化
   │
   ▼
2. configResolved 阶段
   │  └─ 获取配置加载器 API
   │
   ▼
3. resolveId 钩子拦截
   │  └─ virtual:dashboard-extensions → \0virtual:dashboard-extensions
   │
   ▼
4. load 钩子生成代码
   │  ├─ 调用 getVendureConfig() 获取服务端配置
   │  ├─ 扫描所有插件的 dashboard 属性
   │  └─ 生成动态 import 代码
   │
   ▼
5. 输出虚拟模块
```

#### 核心实现代码

```typescript
export function dashboardMetadataPlugin(): Plugin {
    return {
        name: 'vendure:dashboard-extensions-metadata',
        
        async load(id) {
            if (id === resolvedVirtualModuleId) {
                // 1. 获取服务端配置和插件信息
                const { pluginInfo } = await configLoaderApi.getVendureConfig();
                
                // 2. 解析所有 Dashboard 扩展路径
                const pluginsWithExtensions = pluginInfo
                    ?.map(({ dashboardEntryPath, pluginPath, sourcePluginPath }) => {
                        if (!dashboardEntryPath) return null;
                        const basePath = sourcePluginPath 
                            ? path.dirname(sourcePluginPath) 
                            : path.dirname(pluginPath);
                        return path.resolve(basePath, dashboardEntryPath);
                    })
                    .filter(x => x != null) ?? [];

                // 3. 生成动态导入代码
                return `
                    export async function runDashboardExtensions() {
                        ${pluginsWithExtensions
                            .map(ext => `await import(\`${pathToFileURL(ext)}\`);`)
                            .join('\n')}
                    }`;
            }
        },
    };
}
```

#### 虚拟模块输出示例

当有两个插件声明了 Dashboard 扩展时，生成的虚拟模块代码：

```javascript
export async function runDashboardExtensions() {
    await import(`file:///path/to/plugin1/ui/dashboard.js`);
    await import(`file:///path/to/plugin2/ui/dashboard.js`);
}
```

### 3.2 Angular Admin UI 编译流程

**文件位置**：`packages/ui-devkit/src/compiler/compile.ts:36-150`

Angular 版本使用独立的编译流程，通过 `compileUiExtensions` 函数：

1. **脚手架搭建** (`setupScaffold`)：复制基础模板
2. **扩展合并**：将各插件的 UI 代码复制到目标目录
3. **路由生成**：动态生成扩展路由配置
4. **Angular CLI 编译**：调用 `ng build` 或 `ng serve`

---

## 4. 运行时注册层

### 4.1 GlobalRegistry 全局注册表

**文件位置**：`packages/dashboard/src/lib/framework/registry/global-registry.ts:11-50`

由于 Vite 多 bundle 环境下闭包变量不共享，Vendure 使用**单例全局注册表**管理所有扩展。

#### 设计原理

```typescript
class GlobalRegistry {
    private static instance: GlobalRegistry;
    private registry: Map<string, any> = new Map();

    constructor() {
        if (!GlobalRegistry.instance) {
            GlobalRegistry.instance = this;
        }
        return GlobalRegistry.instance;
    }

    public register<T>(key: T, value: GlobalRegistryContents[T]) {
        if (!this.registry.has(key)) {
            this.registry.set(key, value);
        }
    }

    public get<T>(key: T): GlobalRegistryContents[T] {
        return this.registry.get(key);
    }
}

// 挂载到 globalThis 确保跨 bundle 共享
const _globalRegistry: GlobalRegistry = (globalThis as any).globalRegistry ?? new GlobalRegistry();
(globalThis as any).globalRegistry = _globalRegistry;
```

#### 注册表内容定义

**文件位置**：`packages/dashboard/src/lib/framework/registry/registry-types.ts:18-34`

```typescript
export interface GlobalRegistryContents {
    // 回调集合
    extensionSourceChangeCallbacks: Set<() => void>;
    registerDashboardExtensionCallbacks: Set<() => void>;
    
    // 导航菜单
    navMenuConfig: NavMenuConfig;
    navMenuModifiers: Array<(config: NavMenuConfig) => NavMenuConfig>;
    
    // 布局扩展
    dashboardActionBarItemRegistry: Map<string, DashboardActionBarItem[]>;
    dashboardPageBlockRegistry: Map<string, DashboardPageBlockDefinition[]>;
    dashboardWidgetRegistry: Map<string, DashboardWidgetDefinition>;
    
    // 表单与数据
    inputComponents: Map<string, DashboardFormComponent>;
    displayComponents: Map<string, DataDisplayComponent>;
    bulkActionsRegistry: Map<string, BulkAction[]>;
    
    // 其他扩展...
    dashboardAlertRegistry: Map<string, DashboardAlertDefinition>;
    loginExtensions: DashboardLoginExtensions;
    historyEntries: Map<string, ComponentType>;
    dashboardToolbarItemRegistry: Map<string, DashboardToolbarItemDefinition>;
}
```

### 4.2 defineDashboardExtension 入口函数

**文件位置**：`packages/dashboard/src/lib/framework/extension-api/define-dashboard-extension.ts:89-132`

这是 Dashboard 扩展的统一入口函数，采用**两阶段注册**机制。

#### 注册流程

```
调用 defineDashboardExtension(ext)
         │
         ▼
   推入回调队列 (不立即执行)
   registerDashboardExtensionCallbacks.add(callback)
         │
         ▼
   等待 executeDashboardExtensionCallbacks() 触发
         │
         ▼
   ┌── 阶段 1: 基础注册 ──┐
   │  registerNavigationExtensions  │
   │  registerLayoutExtensions      │
   │  registerWidgetExtensions      │
   │  registerFormComponentExtensions │
   │  ...                          │
   └────────────────────────────────┘
         │
         ▼
   ┌── 阶段 2: 菜单修饰 ──┐
   │  执行 navMenuModifiers    │
   │  合并导航菜单配置         │
   └────────────────────────────────┘
```

#### 核心代码

```typescript
export function defineDashboardExtension(extension: DashboardExtension) {
    // 将注册逻辑推入队列，延迟执行
    globalRegistry.get('registerDashboardExtensionCallbacks').add(() => {
        // 1. 注册导航扩展（可能产生 navMenuModifiers）
        const navMenuModifier = registerNavigationExtensions(
            extension.navSections, 
            extension.routes
        );
        if (navMenuModifier) {
            globalRegistry.get('navMenuModifiers').push(navMenuModifier);
        }

        // 2. 注册布局扩展
        registerLayoutExtensions(extension.actionBarItems, extension.pageBlocks);
        
        // 3. 注册组件扩展
        registerWidgetExtensions(extension.widgets);
        registerFormComponentExtensions(extension.customFormComponents);
        registerDataTableExtensions(extension.dataTables);
        registerDetailFormExtensions(extension.detailForms);
        
        // 4. 注册其他扩展
        registerAlertExtensions(extension.alerts);
        registerLoginExtensions(extension.login);
        registerHistoryEntryComponents(extension.historyEntries);
        registerToolbarExtensions(extension.toolbarItems);

        // 5. 触发 HMR 回调（开发模式）
        const callbacks = globalRegistry.get('extensionSourceChangeCallbacks');
        for (const callback of callbacks) callback();
    });
}
```

#### 执行触发点

**文件位置**：`packages/dashboard/src/lib/framework/extension-api/define-dashboard-extension.ts:26-53`

```typescript
export function executeDashboardExtensionCallbacks() {
    // 阶段 1: 执行所有注册回调
    for (const callback of globalRegistry.get('registerDashboardExtensionCallbacks') ?? []) {
        callback();
    }

    // 阶段 2: 应用导航菜单修饰器
    const modifiers = globalRegistry.get('navMenuModifiers');
    if (modifiers?.length) {
        let config = getNavMenuConfig();
        for (const modifier of modifiers) {
            const result = modifier(config);
            if (result?.sections) {
                config = result;
            }
        }
        setNavMenuConfig(config);
    }
}
```

### 4.3 各类型扩展注册逻辑

#### 导航与路由注册

**文件位置**：`packages/dashboard/src/lib/framework/extension-api/logic/navigation.ts:10-59`

```typescript
export function registerNavigationExtensions(
    navSections?: DashboardNavSectionDefinition[] | ((config: NavMenuConfig) => NavMenuConfig),
    routes?: DashboardRouteDefinition[],
) {
    const navMenuModifier = registerNavSections(navSections);
    registerRoutes(routes);
    return navMenuModifier;
}

function registerRoutes(routes?: DashboardRouteDefinition[]) {
    if (!routes) return;
    
    for (const route of routes) {
        // 1. 同时注册导航菜单项
        if (route.navMenuItem) {
            addNavMenuItem({
                url: route.navMenuItem.url ?? route.path,
                id: route.navMenuItem.id ?? route.path,
                title: route.navMenuItem.title ?? route.path,
                requiresPermission: route.navMenuItem.requiresPermission,
                // ...
            }, route.navMenuItem.sectionId);
        }
        
        // 2. 注册到路由注册表
        if (route.path) {
            registerRoute(route);  // 存入 extensionRoutes Map
        }
    }
}
```

**路由注册表**（`packages/dashboard/src/lib/framework/page/page-api.ts:3-8`）：

```typescript
export const extensionRoutes = new Map<string, DashboardRouteDefinition>();

export function registerRoute(config: DashboardRouteDefinition) {
    if (config.path) {
        extensionRoutes.set(config.path, config);
    }
}
```

---

## 5. 路由挂载流程

### 5.1 React Dashboard 路由系统

#### 路由扩展 Hook

**文件位置**：`packages/dashboard/src/lib/framework/page/use-extended-router.tsx:12-132`

`useExtendedRouter` 是 Dashboard 路由挂载的核心，基于 TanStack Router。

#### 挂载流程

```
组件调用 useExtendedRouter(baseRouteTree, options)
         │
         ▼
   检查 extensionsLoaded 状态
         │
         ├─ 未加载 → 返回基础路由
         │
         ▼
   找到 AUTHENTICATED_ROUTE (/_authenticated)
         │
         ▼
   遍历 extensionRoutes Map
         │
         ├─ authenticated: true  → 挂载到 /_authenticated 下
         │  └─ 路径: /_authenticated/{extension-path}
         │
         └─ authenticated: false → 挂载到根路由
            └─ 路径: /{extension-path}
         │
         ▼
   调用 createRouter() 生成最终路由
```

#### 核心代码

```typescript
export const useExtendedRouter = (baseRouteTree: AnyRoute, routerOptions) => {
    const { extensionsLoaded } = useDashboardExtensions();

    return useMemo(() => {
        if (!extensionsLoaded) {
            return createExtendedRouter(routerOptions, baseRouteTree);
        }

        // 找到认证路由节点
        const authenticatedRouteIndex = baseRouteTree.children.findIndex(
            r => r.id === AUTHENTICATED_ROUTE_PREFIX
        );
        let authenticatedRoute = baseRouteTree.children[authenticatedRouteIndex];

        const newAuthenticatedRoutes: AnyRoute[] = [];
        const newRootRoutes: AnyRoute[] = [];

        // 遍历所有扩展路由
        for (const [path, config] of extensionRoutes.entries()) {
            const pathWithoutLeadingSlash = path.startsWith('/') ? path.slice(1) : path;
            const isAuthenticated = config.authenticated !== false;

            if (isAuthenticated) {
                // 挂载到认证路由下
                const newRoute: AnyRoute = createRoute({
                    path: `/${pathWithoutLeadingSlash}`,
                    getParentRoute: () => authenticatedRoute,
                    loader: config.loader,
                    validateSearch: config.validateSearch,
                    component: () => config.component(newRoute),
                });
                newAuthenticatedRoutes.push(newRoute);
            } else {
                // 挂载到根路由
                const newRoute: AnyRoute = createRoute({
                    path: `/${pathWithoutLeadingSlash}`,
                    getParentRoute: () => baseRouteTree,
                    // ...
                });
                newRootRoutes.push(newRoute);
            }
        }

        // 合并生成最终路由树
        const updatedAuthenticatedRoute = authenticatedRoute.addChildren([
            ...authenticatedRoute.children,
            ...newAuthenticatedRoutes,
        ]);

        const extendedRouteTree = baseRouteTree.addChildren([
            ...childrenWithoutAuthenticated,
            updatedAuthenticatedRoute,
            ...newRootRoutes,
        ]);

        return createExtendedRouter(routerOptions, extendedRouteTree);
    }, [baseRouteTree, routerOptions, extensionsLoaded]);
};
```

### 5.2 Angular Admin UI 路由系统

#### 路由注册函数

**文件位置**：`packages/admin-ui/src/lib/core/src/extension/register-route-component.ts:79-148`

```typescript
export function registerRouteComponent<Component, Entity, T, Field, R>(
    options: RegisterRouteComponentOptions<Component, Entity, T, Field, R>
) {
    const { query, entityKey, variables, getBreadcrumbs } = options;
    
    // 创建数据流
    const breadcrumbSubject$ = new BehaviorSubject<BreadcrumbValue>(options.breadcrumb ?? '');
    const titleSubject$ = new BehaviorSubject<string | undefined>(options.title);

    // 详情页数据解析器
    const resolveFn = query && entityKey
        ? createBaseDetailResolveFn({ query, entityKey, variables })
        : undefined;

    return {
        path: options.path ?? '',
        providers: [
            {
                provide: ROUTE_COMPONENT_OPTIONS,
                useValue: {
                    component: options.component,
                    title$: titleSubject$,
                    breadcrumb$: breadcrumbSubject$,
                } satisfies RouteComponentOptions,
            },
        ],
        resolve: resolveFn ? { detail: resolveFn } : {},
        data: {
            locationId: options.locationId,
            breadcrumb: breadcrumbSubject$,
            // 动态面包屑生成
            ...(getBreadcrumbs && query && entityKey
                ? {
                    breadcrumb: data =>
                        data.detail.entity.pipe(map(entity => getBreadcrumbs(entity))),
                }
                : {}),
        },
        component: AngularRouteComponent,  // 统一包装组件
    } satisfies Route;
}
```

#### 使用示例

**文件位置**：`packages/dev-server/test-plugins/with-ui-extension/ui/routes.ts:27-33`

```typescript
export default [
    registerRouteComponent({
        component: GreeterComponent,
        path: 'greet',
        title: 'Greeter Page',
    }),
];
```

---

## 6. 权限映射机制

### 6.1 导航菜单权限控制

#### React Dashboard 版本

**文件位置**：`packages/dashboard/src/lib/framework/nav-menu/nav-menu-extensions.ts:56`

```typescript
export interface NavMenuItem {
    id: string;
    title: string;
    url: string;
    // 权限控制：单个权限或权限数组（或逻辑）
    requiresPermission?: string | string[];
    // ...
}
```

#### Angular Admin UI 版本

**文件位置**：`packages/admin-ui/src/lib/core/src/providers/nav-builder/nav-builder-types.ts:46,78`

```typescript
export interface NavMenuItem {
    // 支持字符串或自定义函数
    requiresPermission?: string | ((userPermissions: string[]) => boolean);
}

export interface NavMenuSection {
    requiresPermission?: string | ((userPermissions: string[]) => boolean);
}
```

**默认权限**：`packages/admin-ui/src/lib/core/src/providers/nav-builder/nav-builder.service.ts:126-128`

```typescript
// 未指定权限时默认要求 Authenticated 权限
if (!config.requiresPermission) {
    config.requiresPermission = Permission.Authenticated;
}
```

### 6.2 路由级权限控制

#### Dashboard 路由认证配置

**文件位置**：`packages/dashboard/src/lib/framework/extension-api/types/navigation.ts:58`

```typescript
export interface DashboardRouteDefinition {
    // 是否需要认证，默认为 true
    authenticated?: boolean;
}
```

挂载时根据此属性决定路由父节点：
- `authenticated: true` → 父节点为 `/_authenticated`（受 AuthGuard 保护）
- `authenticated: false` → 父节点为根路由（公开访问）

### 6.3 PermissionsService 权限服务

**文件位置**：`packages/admin-ui/src/lib/core/src/providers/permissions/permissions.service.ts:10-41`

Angular 版本的权限检查服务：

```typescript
@Injectable({ providedIn: 'root' })
export class PermissionsService {
    private currentUserPermissions: string[] = [];
    private _currentUserPermissions$ = new BehaviorSubject<string[]>([]);
    currentUserPermissions$ = this._currentUserPermissions$.asObservable();

    // 登录或切换频道时更新权限
    setCurrentUserPermissions(permissions: string[]) {
        this.currentUserPermissions = permissions;
        this._currentUserPermissions$.next(permissions);
    }

    // 检查权限（或逻辑：任一满足即可）
    userHasPermissions(requiredPermissions: Array<string | Permission>): boolean {
        for (const perm of requiredPermissions) {
            if (this.currentUserPermissions.includes(perm)) {
                return true;
            }
        }
        return false;
    }
}
```

---

## 7. 完整链路时序图

### 7.1 React Dashboard 扩展链路

```
┌─────────┐  1. @VendurePlugin({ dashboard: './ui/ext.tsx' })  ┌────────────┐
│ 插件代码 │───────────────────────────────────────────────────▶│ 服务端启动 │
└─────────┘                                                    └──────┬─────┘
                                                                       │
                                                                       ▼
                                                         ┌───────────────────────┐
                                                         │ 收集所有插件 dashboard │
                                                         │ 路径写入临时配置       │
                                                         └───────────┬───────────┘
                                                                     │
┌────────────┐  2. Vite 启动                                         │
│ 浏览器     │◀───────────────────────────────────────────────────────┘
└─────┬──────┘
      │  3. 访问 Dashboard
      ▼
┌─────────────────────────────────────────────────────────────────┐
│ useDashboardExtensions()                                        │
│  ├─ 导入 virtual:dashboard-extensions                           │
│  └─ 调用 runDashboardExtensions()                               │
└─────────────────────┬───────────────────────────────────────────┘
                      │  4. 动态 import 各插件扩展文件
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│ defineDashboardExtension({                                      │
│   navSections: [...], routes: [...], widgets: [...]             │
│ })                                                               │
│  └─ 将注册回调推入 registerDashboardExtensionCallbacks 队列      │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│ executeDashboardExtensionCallbacks()                             │
│  ├─ 阶段1: 执行所有注册回调                                      │
│  │   ├─ registerNavigationExtensions → navMenuConfig, extensionRoutes │
│  │   ├─ registerLayoutExtensions → actionBarItemRegistry        │
│  │   └─ ... 其他类型注册                                        │
│  └─ 阶段2: 应用 navMenuModifiers → 合并菜单配置                 │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│ useExtendedRouter()                                             │
│  ├─ 读取 extensionRoutes Map                                    │
│  ├─ 根据 authenticated 决定挂载位置                              │
│  └─ 动态创建 TanStack Router 实例                               │
└─────────────────────┬───────────────────────────────────────────┘
                      │  5. 路由可用，页面渲染
                      ▼
              ┌────────────────┐
              │ 扩展页面展示    │
              └────────────────┘
```

### 7.2 Angular Admin UI 扩展链路

```
┌─────────┐  1. 静态属性 uiExtensions 声明  ┌───────────────┐
│ 插件代码 │───────────────────────────────▶│ compileUiExt() │
└─────────┘                                └───────┬───────┘
                                                   │
                                                   ▼
                                         ┌──────────────────┐
                                         │ setupScaffold()  │
                                         │ 复制扩展代码      │
                                         │ 生成路由配置      │
                                         └────────┬─────────┘
                                                  │
                                                  ▼
                                         ┌──────────────────┐
                                         │ ng build/serve   │
                                         │ 编译 Angular 应用 │
                                         └────────┬─────────┘
                                                  │
┌────────────┐  2. 浏览器加载                     │
│ 浏览器     │◀───────────────────────────────────┘
└─────┬──────┘
      │  3. APP_INITIALIZER 执行
      ▼
┌─────────────────────────────────────────────────┐
│ addNavMenuSection() / addNavMenuItem()         │
│  └─ 调用 NavBuilderService 注册菜单            │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│ Router 配置中包含扩展路由（编译时确定）         │
│  └─ /extensions/{route} 懒加载模块             │
└───────────────────┬─────────────────────────────┘
                    │  4. 导航到扩展路由
                    ▼
          ┌──────────────────────┐
          │ 扩展组件渲染          │
          └──────────────────────┘
```

---

## 8. 权限约束链路深度追踪

本节沿代码逐层追踪权限字段从插件声明到前端判定的完整数据流，明确每一站的来源字段、中间结构、判定点与失败症状。

### 8.1 全链路总览

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ 阶段 1 · 服务端声明                                                            │
│                                                                                 │
│  CrudPermissionDefinition('ProductBundle')                                      │
│  └─ .getMetadata() → [{name:'CreateProductBundle',...}, {name:'ReadProductBundle',...}, ...] │
│  └─ 推入 config.authOptions.customPermissions[]                                  │
│  └─ generatePermissionEnum() → 合并入 GraphQL Permission enum                   │
│  └─ GlobalSettingsResolver.serverConfig → {permissions:[{name,description,assignable}]} │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                                 │  GraphQL 响应
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ 阶段 2 · 前端权限获取                                                          │
│                                                                                 │
│  ┌── Dashboard (React) ──────────────────────────────────┐                      │
│  │  AuthProvider → CurrentUserQuery → me.channels[].permissions[]  │              │
│  │  usePermissions() → hasPermissions(permissions[]) → boolean     │              │
│  └────────────────────────────────────────────────────────────────┘              │
│                                                                                 │
│  ┌── Admin UI (Angular) ─────────────────────────────────┐                      │
│  │  AuthService.logIn() → activeChannel.permissions                 │              │
│  │  PermissionsService.setCurrentUserPermissions(permissions[])     │              │
│  │  PermissionsService.userHasPermissions(required) → boolean      │              │
│  └────────────────────────────────────────────────────────────────┘              │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ 阶段 3 · 扩展声明中的权限字段                                                  │
│                                                                                 │
│  defineDashboardExtension({                                                     │
│    routes: [{ navMenuItem: { requiresPermission: 'ReadProductBundle' } }]        │
│  })                                                                             │
│  └─ registerNavigationExtensions() → addNavMenuItem({requiresPermission})        │
│     └─ globalRegistry → navMenuConfig.sections[].items[].requiresPermission     │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ 阶段 4 · 判定点                                                                │
│                                                                                 │
│  判定点 A: 导航可见性 ── NavMain.isItemAllowed() / BaseNavComponent.shouldDisplayLink() │
│  判定点 B: 路由拦截 ── _authenticated.tsx beforeLoad / LoginGuard               │
│  判定点 C: 组件级 ── PermissionGuard / vdrIfPermissions / hasPermission pipe    │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 阶段 1：服务端声明 — 权限的诞生

#### 8.2.1 来源字段

**文件**：`packages/core/src/common/permission-definition.ts`

| 类 | 产出权限字符串 | 示例 |
|----|---------------|------|
| `PermissionDefinition` | `name` 原样 | `SyncInventory` |
| `CrudPermissionDefinition` | `{Create,Read,Update,Delete}{name}` | `CreateProductBundle`, `ReadProductBundle` |
| `RwPermissionDefinition` | `{Read,Write}{name}` | `ReadDashboardSavedViews` |

```typescript
// packages/dev-server/example-plugins/product-bundles/constants.ts
export const productBundlePermission = new CrudPermissionDefinition('ProductBundle');
// 产出: CreateProductBundle, ReadProductBundle, UpdateProductBundle, DeleteProductBundle
```

#### 8.2.2 注册到配置

**文件**：`packages/dev-server/example-plugins/product-bundles/product-bundles.plugin.ts:18-29`

```typescript
@VendurePlugin({
    configuration: config => {
        // 关键: 必须推入 customPermissions，否则 GraphQL enum 里没有这个权限
        config.authOptions.customPermissions.push(productBundlePermission);
        return config;
    },
})
```

**⚠️ 易漏点**：如果只声明了 `PermissionDefinition` 但忘记推入 `customPermissions`，权限字符串不会出现在 GraphQL `Permission` enum 中，前端拿不到也无法检查。

#### 8.2.3 合并进 GraphQL Schema

**文件**：`packages/core/src/api/config/generate-permissions.ts:42-67`

```typescript
export function generatePermissionEnum(schema, customPermissions) {
    // DEFAULT_PERMISSIONS + customPermissions → 合并去重
    const allPermissionsMetadata = getAllPermissionsMetadata(customPermissions);
    // 生成 GraphQL enum: enum Permission { Authenticated, SuperAdmin, CreateProductBundle, ... }
}
```

#### 8.2.4 通过 API 暴露给前端

**文件**：`packages/core/src/api/resolvers/admin/global-settings.resolver.ts:63-75`

```typescript
@ResolveField()
serverConfig(@Info() info): ServerConfig {
    const permissions = getAllPermissionsMetadata(
        this.configService.authOptions.customPermissions
    ).filter(p => !p.internal);  // 过滤掉 internal 权限
    return { permissions, ... };
}
```

**中间结构**：`ServerConfig.permissions = [{ name: 'CreateProductBundle', description: '...', assignable: true }, ...]`

### 8.3 阶段 2：前端权限获取 — 用户有什么权限

#### 8.3.1 Dashboard (React) 权限流

**文件**：`packages/dashboard/src/lib/providers/auth.tsx:84-103`

```graphql
query CurrentUserInformation {
    me {
        channels {
            id
            token
            code
            permissions   # ← 这里是用户在此 channel 下拥有的全部权限字符串
        }
    }
}
```

**中间结构**：

```typescript
// AuthContext.channels
[
    { id: '1', token: 'xxx', code: 'default', permissions: ['Authenticated', 'ReadCatalog', 'ReadProductBundle', ...] },
]
```

**判定函数**：`packages/dashboard/src/lib/hooks/use-permissions.ts:23-46`

```typescript
export function usePermissions() {
    const { channels } = useAuth();
    const { activeChannel } = useChannel();

    const hasPermissions = useCallback((permissions: string[]) => {
        if (permissions.length === 0) return true;
        const selectedChannel = channels?.find(c => c.id === activeChannel?.id);
        if (!selectedChannel) return false;
        // 关键: some() → OR 逻辑，任一匹配即通过
        return permissions.some(p => selectedChannel.permissions.includes(p as any));
    }, [channels, activeChannel?.id]);

    return { hasPermissions };
}
```

#### 8.3.2 Admin UI (Angular) 权限流

**文件**：`packages/admin-ui/src/lib/core/src/providers/auth/auth.service.ts:44-47`

```typescript
// 登录成功后立即设置权限
const activeChannel = this.getActiveChannel(login.channels);
this.permissionsService.setCurrentUserPermissions(activeChannel.permissions);
```

**文件**：`packages/admin-ui/src/lib/core/src/providers/permissions/permissions.service.ts:33-40`

```typescript
userHasPermissions(requiredPermissions: Array<string | Permission>): boolean {
    // 关键: some() → OR 逻辑，任一匹配即通过
    for (const perm of requiredPermissions) {
        if (this.currentUserPermissions.includes(perm)) return true;
    }
    return false;
}
```

**⚠️ 注意差异**：

| 前端 | OR/AND 逻辑 | 权限来源 |
|------|------------|---------|
| Dashboard `usePermissions` | **OR** (`some`) | `channel.permissions` |
| Admin UI `PermissionsService` | **OR** (`some`) | `activeChannel.permissions` |
| Admin UI `vdrIfPermissions` 指令 | **AND** (all must match) | 同上 |

### 8.4 阶段 3：扩展声明中的权限字段 — 传递给注册表

#### 8.4.1 Dashboard 路由声明的权限字段

**文件**：`packages/dashboard/src/lib/framework/extension-api/types/navigation.ts:35`

```typescript
export interface DashboardRouteDefinition {
    navMenuItem?: Partial<NavMenuItem> & { sectionId: string };
    authenticated?: boolean;  // 默认 true
}

// NavMenuItem.requiresPermission
// 文件: packages/dashboard/src/lib/framework/nav-menu/nav-menu-extensions.ts:56
export interface NavMenuItem {
    requiresPermission?: string | string[];
}
```

#### 8.4.2 注册流转

**文件**：`packages/dashboard/src/lib/framework/extension-api/logic/navigation.ts:38-58`

```typescript
function registerRoutes(routes?: DashboardRouteDefinition[]) {
    for (const route of routes) {
        if (route.navMenuItem) {
            // requiresPermission 从 route.navMenuItem 传入 navMenuConfig
            addNavMenuItem({
                requiresPermission: route.navMenuItem.requiresPermission,  // ← 传递
                ...
            }, route.navMenuItem.sectionId);
        }
        registerRoute(route);  // route.authenticated 存入 extensionRoutes Map
    }
}
```

**最终存储位置**：
- `navMenuConfig.sections[].items[].requiresPermission` → 导航可见性判定用
- `extensionRoutes Map<path, { authenticated: boolean }>` → 路由挂载位置判定用

### 8.5 阶段 4：判定点 — 权限在哪生效

#### 判定点 A：导航可见性（侧栏菜单项是否显示）

**Dashboard (React)**

**文件**：`packages/dashboard/src/lib/components/layout/nav-main.tsx:164-175`

```typescript
const isItemAllowed = useCallback((item: NavMenuItem) => {
    if (!item.requiresPermission) return true;  // 无权限要求 → 始终显示
    const permissions = Array.isArray(item.requiresPermission)
        ? item.requiresPermission
        : [item.requiresPermission];
    return hasPermissions(permissions);  // OR 逻辑
}, [hasPermissions]);
```

同文件 `getSortedSections()`（:178-202）进一步过滤：权限不通过时整节被丢弃。

**Admin UI (Angular)**

**文件**：`packages/admin-ui/src/lib/core/src/components/base-nav/base-nav.component.ts:35-48`

```typescript
shouldDisplayLink(menuItem: Pick<NavMenuItem, 'requiresPermission'>) {
    if (!this.userPermissions) return false;              // 权限未加载 → 不显示
    if (!menuItem.requiresPermission) return true;         // 无要求 → 显示
    if (typeof menuItem.requiresPermission === 'string')   // 字符串 → includes
        return this.userPermissions.includes(menuItem.requiresPermission);
    if (typeof menuItem.requiresPermission === 'function') // 函数 → 委托调用
        return menuItem.requiresPermission(this.userPermissions);
}
```

**判定逻辑总结**：

| 场景 | Dashboard | Admin UI |
|------|-----------|----------|
| `requiresPermission` 未设 | 显示 | 显示 |
| 字符串 `'ReadProductBundle'` | `channel.permissions.some(...)` | `userPermissions.includes(...)` |
| 数组 `['ReadA', 'ReadB']` | some → OR | 直接 includes 单个（Angular 版只接受 string 或函数） |
| 函数 `(perms) => ...` | 不支持 | 委托调用 |
| 用户权限未加载 | 视 channel 有无而定 | `userPermissions` 为 falsy → **不显示** |

#### 判定点 B：路由拦截（未认证用户被重定向）

**Dashboard (React)**

**文件**：`packages/dashboard/src/app/routes/_authenticated.tsx:7-17`

```typescript
export const Route = createFileRoute('/_authenticated')({
    beforeLoad: ({ context, location }) => {
        if (!context.auth.isAuthenticated) {
            throw redirect({ to: '/login', search: { redirect: location.href } });
        }
    },
});
```

扩展路由通过 `useExtendedRouter` 根据 `config.authenticated` 决定挂载在 `/_authenticated` 下还是根路由：
- `authenticated: true`（默认）→ 父节点有 `beforeLoad` 守卫 → 未登录重定向到 `/login`
- `authenticated: false` → 父节点为根路由 → 无守卫

**Admin UI (Angular)**

**文件**：`packages/admin-ui/src/lib/login/src/providers/login.guard.ts:13-26`

```typescript
@Injectable({ providedIn: 'root' })
export class LoginGuard {
    canActivate(route): Observable<boolean> {
        return this.authService.checkAuthenticatedStatus().pipe(
            map(authenticated => {
                if (authenticated) { this.router.navigate(['/']); }
                return !authenticated;
            }),
        );
    }
}
```

**⚠️ 注意**：Angular Admin UI 扩展路由的权限控制是**编译时**决定的（路由配置静态写入），不是运行时 `beforeLoad` 动态检查。路由级权限只区分"是否需要登录"，不区分具体权限字符串。细粒度权限靠导航菜单隐藏 + 模板指令控制。

#### 判定点 C：组件级权限（页面内的按钮/区域）

**Dashboard — `<PermissionGuard>`**

**文件**：`packages/dashboard/src/lib/components/shared/permission-guard.tsx:44-50`

```typescript
export function PermissionGuard({ requires, children }) {
    const { hasPermissions } = usePermissions();
    const permissions = Array.isArray(requires) ? requires : [requires];
    if (!hasPermissions(permissions)) return null;  // OR 逻辑，无权限不渲染
    return children;
}
```

**Admin UI — `*vdrIfPermissions` 指令**

**文件**：`packages/admin-ui/src/lib/core/src/shared/directives/if-permissions.directive.ts:39-49`

```typescript
// 注意: 数组形式是 AND 逻辑（所有权限都必须满足）
return this.permissionsService.currentUserPermissions$.pipe(
    map(() => this.permissionsService.userHasPermissions(permissions)),
);
```

**Admin UI — `hasPermission` 管道**

**文件**：`packages/admin-ui/src/lib/core/src/shared/pipes/has-permission.pipe.ts:31-44`

```typescript
transform(input: string | string[]): any {
    const requiredPermissions = Array.isArray(input) ? input : [input];
    this.subscription = this.permissionsService.currentUserPermissions$.subscribe(() => {
        this.hasPermission = this.permissionsService.userHasPermissions(requiredPermissions);
    });
    return this.hasPermission;  // OR 逻辑
}
```

**⚠️ AND/OR 不一致**：

| 判定点 | 逻辑 |
|--------|------|
| `vdrIfPermissions` 指令 | **AND** — 数组中所有权限都要满足 |
| `hasPermission` 管道 | **OR** — `PermissionsService.userHasPermissions()` |
| `PermissionGuard` (React) | **OR** — `hasPermissions(permissions)` |
| 导航可见性 (React) | **OR** — `isItemAllowed()` |
| 导航可见性 (Angular) | 字符串→includes，函数→委托 |

### 8.6 权限字段流转汇总表

| 站点 | 字段/结构 | 类型 | 说明 |
|------|----------|------|------|
| 服务端 `CrudPermissionDefinition` | `.config.name` | `string` | 原始名称如 `ProductBundle` |
| 服务端 `.getMetadata()` | `PermissionMetadata[]` | `{name,description,assignable,internal}[]` | CRUD 展开 |
| 服务端 `customPermissions` | `PermissionDefinition[]` | 类实例数组 | 配置中收集 |
| GraphQL `Permission` enum | enum value | 枚举值 | `CreateProductBundle` 等 |
| GraphQL `serverConfig.permissions` | `{name,description,assignable}[]` | 对象数组 | 暴露给前端 |
| GraphQL `me.channels[].permissions` | `string[]` | 权限字符串数组 | 用户实际拥有的权限 |
| 扩展声明 `navMenuItem.requiresPermission` | `string \| string[]` | 权限字符串 | 声明时硬编码 |
| 注册表 `navMenuConfig.sections[].items[].requiresPermission` | `string \| string[]` | 同上 | 运行时存储 |
| 判定函数输入 | `string[]` | 标准化后的数组 | 判定时统一为数组 |

### 8.7 最小排障清单

当扩展页面的导航项不可见或路由无法访问时，按以下顺序排查：

#### 症状 1：导航菜单项不显示

| # | 检查项 | 验证方法 | 常见原因 |
|---|--------|---------|---------|
| 1 | 权限是否注册到服务端 | 在 Admin API 中执行 `{ globalSettings { serverConfig { permissions { name } } } }`，搜索你的权限名 | 忘记 `config.authOptions.customPermissions.push()` |
| 2 | 角色是否分配了该权限 | 在 Dashboard → Settings → Roles 检查当前用户角色 | 新权限未分配给任何角色 |
| 3 | 用户 channel 中是否包含该权限 | 浏览器 DevTools → Network → `CurrentUserInformation` query → `me.channels[N].permissions` | 角色权限未同步到 channel |
| 4 | `requiresPermission` 拼写是否正确 | 对比 GraphQL enum 值与扩展声明中的字符串 | 拼写错误或缺少 CRUD 前缀（如写了 `ProductBundle` 而非 `ReadProductBundle`） |
| 5 | `sectionId` 是否指向了存在的 section | 浏览器控制台搜索 `Could not add menu item` 错误 | 菜单项注册到了不存在的 section |
| 6 | Angular 版：权限是否已加载 | 在 `BaseNavComponent.shouldDisplayLink` 断点，检查 `this.userPermissions` | `userStatus` query 未完成或 channel token 不匹配 |
| 7 | Dashboard 版：activeChannel 是否正确 | 检查 `usePermissions` hook 中 `activeChannel.id` 与 `channels` 是否匹配 | localStorage 中的 channel token 过期 |

#### 症状 2：路由被重定向到登录页

| # | 检查项 | 验证方法 | 常见原因 |
|---|--------|---------|---------|
| 1 | `authenticated` 是否为预期值 | 检查 `DashboardRouteDefinition.authenticated` 字段 | 默认为 `true`，公开页面需显式设为 `false` |
| 2 | Auth context 是否正确初始化 | 在 `AuthProvider` 检查 `status` 和 `isAuthenticated` | 登录 token 过期或 CORS 问题 |
| 3 | Angular 版路由守卫 | 检查 `LoginGuard` 是否拦截了扩展路由 | 扩展路由配置在错误的路由层级 |

#### 症状 3：页面内按钮/区域不可见

| # | 检查项 | 验证方法 | 常见原因 |
|---|--------|---------|---------|
| 1 | 确认 AND/OR 语义 | Dashboard `PermissionGuard` = **OR**，Admin UI `*vdrIfPermissions` = **AND** | 传了多个权限但语义理解错误 |
| 2 | `PermissionGuard` 的 `requires` 字段 | 确认类型为 `Permission \| string \| string[] \| Permission[]` | 传入了 PermissionDefinition 对象而非字符串 |
| 3 | `vdrIfPermissions` 数组参数 | 传入数组 = 所有权限都必须满足 | 应该用 OR 语义却传了数组 |

#### 通用调试技巧

1. **GraphQL 验证**：在 Admin API playground 执行以下查询，确认权限链完整：
   ```graphql
   query CheckPermissions {
     me { channels { id code permissions } }
     globalSettings { serverConfig { permissions { name assignable } } }
   }
   ```

2. **Dashboard 断点**：在以下位置设置断点或 `console.log`：
   - `packages/dashboard/src/lib/hooks/use-permissions.ts:37` — 查看 `selectedChannel.permissions` 与传入的 `permissions`
   - `packages/dashboard/src/lib/components/layout/nav-main.tsx:166` — 查看 `isItemAllowed` 结果
   - `packages/dashboard/src/lib/framework/extension-api/logic/navigation.ts:43` — 查看 `addNavMenuItem` 的参数

3. **Admin UI 断点**：
   - `packages/admin-ui/src/lib/core/src/providers/permissions/permissions.service.ts:33` — 查看 `userHasPermissions` 输入输出
   - `packages/admin-ui/src/lib/core/src/components/base-nav/base-nav.component.ts:35` — 查看 `shouldDisplayLink`

4. **注册表快照**：在浏览器控制台执行：
   ```javascript
   // Dashboard
   globalThis.globalRegistry.get('navMenuConfig')
   globalThis.globalRegistry.get('registerDashboardExtensionCallbacks').size
   ```

---

## 10. 关键文件索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 插件装饰器 | `packages/core/src/plugin/vendure-plugin.ts` | 164-192 |
| Dashboard 元数据插件 | `packages/dashboard/vite/vite-plugin-dashboard-metadata.ts` | 16-73 |
| 全局注册表 | `packages/dashboard/src/lib/framework/registry/global-registry.ts` | 11-50 |
| Dashboard 扩展入口 | `packages/dashboard/src/lib/framework/extension-api/define-dashboard-extension.ts` | 89-132 |
| 导航注册逻辑 | `packages/dashboard/src/lib/framework/extension-api/logic/navigation.ts` | 10-59 |
| 扩展路由挂载 | `packages/dashboard/src/lib/framework/page/use-extended-router.tsx` | 12-132 |
| Angular 路由注册 | `packages/admin-ui/src/lib/core/src/extension/register-route-component.ts` | 79-148 |
| 导航构建服务 | `packages/admin-ui/src/lib/core/src/providers/nav-builder/nav-builder.service.ts` | 25-207 |
| 权限服务 (Angular) | `packages/admin-ui/src/lib/core/src/providers/permissions/permissions.service.ts` | 10-41 |
| 权限 Hook (React) | `packages/dashboard/src/lib/hooks/use-permissions.ts` | 23-46 |
| 权限守卫 (React) | `packages/dashboard/src/lib/components/shared/permission-guard.tsx` | 44-50 |
| 权限定义 | `packages/core/src/common/permission-definition.ts` | 86-110 |
| 权限枚举生成 | `packages/core/src/api/config/generate-permissions.ts` | 42-67 |
| 默认权限 + 合并 | `packages/core/src/common/constants.ts` | 14-80 |
| 全局配置暴露权限 | `packages/core/src/api/resolvers/admin/global-settings.resolver.ts` | 63-75 |
| 认证路由守卫 | `packages/dashboard/src/app/routes/_authenticated.tsx` | 7-17 |
| 侧栏导航渲染 | `packages/dashboard/src/lib/components/layout/nav-main.tsx` | 94-350 |
| Angular 导航权限 | `packages/admin-ui/src/lib/core/src/components/base-nav/base-nav.component.ts` | 35-48 |
| Angular 权限指令 | `packages/admin-ui/src/lib/core/src/shared/directives/if-permissions.directive.ts` | 30-70 |
| Angular 权限管道 | `packages/admin-ui/src/lib/core/src/shared/pipes/has-permission.pipe.ts` | 31-44 |
| UI 扩展类型定义 | `packages/ui-devkit/src/compiler/types.ts` | 130-264 |

---

## 11. 总结

Vendure 后台扩展注册链路的设计亮点：

1. **分层解耦**：服务端声明、编译时收集、运行时注册、路由挂载四层清晰分离
2. **虚拟模块**：利用 Vite 虚拟模块实现零配置动态扩展收集
3. **全局单例**：通过 globalThis 挂载的注册表解决多 bundle 环境下的状态共享
4. **两阶段注册**：先收集所有扩展再统一应用，支持跨插件的菜单合并
5. **灵活权限**：支持字符串、数组、自定义函数三种权限声明方式

权限约束链路的关键发现：

1. **权限必须推入 `customPermissions`**：仅声明 `PermissionDefinition` 不够，必须推入配置才能进入 GraphQL enum
2. **前端权限来自 channel**：权限按 channel 隔离，切换 channel 后权限列表会变
3. **AND/OR 语义不统一**：Dashboard 统一 OR，Admin UI 的 `vdrIfPermissions` 指令是 AND，管道是 OR
4. **路由拦截只管认证不管授权**：`_authenticated` 路由守卫只检查是否登录，细粒度权限靠导航隐藏 + 组件级控制
