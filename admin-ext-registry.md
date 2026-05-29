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

## 8. 关键文件索引

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
| 权限服务 | `packages/admin-ui/src/lib/core/src/providers/permissions/permissions.service.ts` | 10-41 |
| UI 扩展类型定义 | `packages/ui-devkit/src/compiler/types.ts` | 130-264 |

---

## 9. 总结

Vendure 后台扩展注册链路的设计亮点：

1. **分层解耦**：服务端声明、编译时收集、运行时注册、路由挂载四层清晰分离
2. **虚拟模块**：利用 Vite 虚拟模块实现零配置动态扩展收集
3. **全局单例**：通过 globalThis 挂载的注册表解决多 bundle 环境下的状态共享
4. **两阶段注册**：先收集所有扩展再统一应用，支持跨插件的菜单合并
5. **灵活权限**：支持字符串、数组、自定义函数三种权限声明方式

该设计既保证了插件的独立性，又实现了扩展间的协同与整合。
