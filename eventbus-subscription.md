# Vendure EventBus 与领域事件订阅：源码级梳理

## 1. 整体架构概览

Vendure 的事件系统建立在 NestJS 依赖注入之上，核心组件关系如下：

```
┌────────────────────────────────────────────────────────────────────────┐
│                        GraphQL Resolver                                │
│  @Transaction() → TransactionInterceptor → TransactionWrapper          │
│       │                                                                │
│       │  ctx (携带 TRANSACTION_MANAGER_KEY)                             │
│       ▼                                                                │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐          │
│  │  Service 层  │  │ OrderProcess │  │ 辅助组件              │          │
│  │  CRUD 发布   │  │ 回调内发布    │  │ OrderModifier        │          │
│  └──────┬──────┘  └──────┬───────┘  │ FulltextSearchService │          │
│         │                │           └──────────┬───────────┘          │
│         └────────────────┼──────────────────────┘                      │
│                          ▼                                             │
│                  ┌──────────────┐                                      │
│                  │   EventBus   │ (单例 Subject<VendureEvent>)          │
│                  └──────┬───────┘                                      │
│            ┌────────────┼────────────┐                                 │
│            ▼            ▼            ▼                                 │
│   ofType() 订阅   filter() 订阅  BlockingHandler                      │
│   (异步,等事务)    (异步,等事务)   (同步,阻塞publish)                    │
│            │            │            │                                 │
│            └───── awaitActiveTransactions ────┘                        │
│                   (TransactionSubscriber)                              │
│                          │                                             │
│                TypeORM afterTransactionCommit                          │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────┐           │
│  │  非事务发布路径                                           │           │
│  │  bootstrap.ts → BootstrappedEvent (无 ctx,立即投递)       │           │
│  │  InitializerService → InitializerEvent (无 ctx,立即投递)  │           │
│  │  EmailProcessor → EmailSendEvent (Worker JobQueue,无事务)  │           │
│  └─────────────────────────────────────────────────────────┘           │
└────────────────────────────────────────────────────────────────────────┘
```

**关键源文件**：

| 文件 | 职责 |
|------|------|
| `core/src/event-bus/event-bus.ts` | EventBus 核心：publish / ofType / filter / registerBlockingEventHandler |
| `core/src/event-bus/vendure-event.ts` | `VendureEvent` 抽象基类 |
| `core/src/event-bus/vendure-entity-event.ts` | `VendureEntityEvent<Entity, Input>` — CRUD 事件基类 |
| `core/src/event-bus/event-bus.module.ts` | NestJS Module，导入 ConnectionModule，提供并导出 EventBus |
| `core/src/connection/transaction-subscriber.ts` | 监听 TypeORM 事务 commit/rollback，提供 `awaitCommit()` |
| `core/src/connection/transaction-wrapper.ts` | `executeInTransaction()` — 事务开始、提交、回滚、释放 |
| `core/src/api/decorators/transaction.decorator.ts` | `@Transaction()` 装饰器定义 |
| `core/src/api/middleware/transaction-interceptor.ts` | 拦截器，调用 TransactionWrapper 包装 resolver |
| `core/src/common/constants.ts` | `TRANSACTION_MANAGER_KEY = Symbol('TRANSACTION_MANAGER')` |
| `core/src/api/common/request-context.ts` | RequestContext，携带事务 EntityManager |
| `core/src/bootstrap.ts` | Server/Worker 启动入口，发布 `BootstrappedEvent` |
| `core/src/service/initializer.service.ts` | 服务初始化编排，发布 `InitializerEvent` |
| `core/src/config/order/default-order-process.ts` | 内置 OrderProcess，`onTransitionEnd` 中发布 `OrderPlacedEvent` |
| `core/src/service/helpers/order-state-machine/order-state-machine.ts` | OrderStateMachine — 回调分发，不直接发布事件 |
| `core/src/service/helpers/order-modifier/order-modifier.ts` | OrderModifier — 订单修改辅助器，发布 OrderLineEvent/OrderEvent |
| `email-plugin/src/email-processor.ts` | EmailProcessor — Worker 内邮件发送，发布 EmailSendEvent |

---

## 2. 事件类型体系

### 2.1 基类

```
VendureEvent                          ← 所有事件的根
├── createdAt: Date
│
├── VendureEntityEvent<Entity, Input> ← 实体 CRUD 事件
│   ├── entity: Entity
│   ├── type: 'created' | 'updated' | 'deleted'
│   ├── ctx: RequestContext
│   └── input?: Input
│
├── XxxStateTransitionEvent           ← 状态机转换事件
│   ├── fromState / toState
│   ├── ctx: RequestContext
│   └── 实体引用 (order, payment, fulfillment, refund)
│
└── 其他独立事件                       ← 不走 CRUD 模式
    ├── LoginEvent(ctx, user)
    ├── AccountRegistrationEvent(ctx, user)
    ├── BootstrappedEvent()
    └── ...
```

### 2.2 完整事件清单（按领域分组）

#### Catalog 领域
| 事件类 | 继承自 | 事件触发场景 |
|--------|--------|-------------|
| `ProductEvent` | `VendureEntityEvent<Product>` | 产品 created / updated / deleted |
| `ProductChannelEvent` | `VendureEntityEvent<Product>` | 产品分配到/移出 Channel (assigned / removed) |
| `ProductOptionGroupEvent` | `VendureEntityEvent<ProductOptionGroup>` | 选项组 CRUD |
| `ProductOptionGroupChangeEvent` | `VendureEvent` | 选项组变更 |
| `ProductOptionEvent` | `VendureEntityEvent<ProductOption>` | 选项 CRUD |
| `ProductVariantEvent` | `VendureEntityEvent<ProductVariant[]>` | 变体 CRUD |
| `ProductVariantChannelEvent` | `VendureEntityEvent<ProductVariant>` | 变体分配到/移出 Channel |
| `ProductVariantPriceEvent` | `VendureEntityEvent<ProductVariantPrice>` | 变体价格 CRUD |
| `FacetEvent` | `VendureEntityEvent<Facet>` | Facet CRUD |
| `FacetValueEvent` | `VendureEntityEvent<FacetValue>` | FacetValue CRUD |
| `CollectionEvent` | `VendureEntityEvent<Collection>` | 集合 CRUD |
| `CollectionModificationEvent` | `VendureEvent` | 集合内容变更（受影响的变体 ID 列表） |
| `AssetEvent` | `VendureEntityEvent<Asset>` | 资产 CRUD |
| `AssetChannelEvent` | `VendureEntityEvent<Asset>` | 资产分配到/移出 Channel |

#### Order 领域
| 事件类 | 继承自 | 事件触发场景 |
|--------|--------|-------------|
| `OrderEvent` | `VendureEntityEvent<Order>` | 订单 created / updated |
| `OrderStateTransitionEvent` | `VendureEvent` | 订单状态转换 |
| `OrderPlacedEvent` | `VendureEvent` | 订单下单（由 default-order-process 发布） |
| `OrderLineEvent` | `VendureEntityEvent<OrderLine>` | 订单行 created / updated / cancelled |
| `CouponCodeEvent` | `VendureEntityEvent<CouponCode>` | 优惠券 CRUD |

#### Payment & Fulfillment
| 事件类 | 继承自 | 事件触发场景 |
|--------|--------|-------------|
| `PaymentMethodEvent` | `VendureEntityEvent<PaymentMethod>` | 支付方式 CRUD |
| `PaymentStateTransitionEvent` | `VendureEvent` | 支付状态转换 |
| `FulfillmentEvent` | `VendureEntityEvent<Fulfillment>` | 履约 CRUD |
| `FulfillmentStateTransitionEvent` | `VendureEvent` | 履约状态转换 |
| `RefundEvent` | `VendureEntityEvent<Refund>` | 退款 CRUD |
| `RefundStateTransitionEvent` | `VendureEvent` | 退款状态转换 |
| `StockMovementEvent` | `VendureEvent` | 库存调整/分配/出售/取消/释放 |
| `StockLocationEvent` | `VendureEntityEvent<StockLocation>` | 库位 CRUD |

#### Customer & Auth
| 事件类 | 继承自 | 事件触发场景 |
|--------|--------|-------------|
| `CustomerEvent` | `VendureEntityEvent<Customer>` | 客户 CRUD |
| `CustomerAddressEvent` | `VendureEntityEvent<Address>` | 客户地址 CRUD |
| `CustomerGroupEvent` | `VendureEntityEvent<CustomerGroup>` | 客户组 CRUD |
| `CustomerGroupChangeEvent` | `VendureEvent` | 客户组成员变更 (assigned / removed) |
| `AccountRegistrationEvent` | `VendureEvent` | 新用户注册 |
| `AccountVerifiedEvent` | `VendureEvent` | 用户验证 |
| `AttemptedLoginEvent` | `VendureEvent` | 登录尝试（无论成功失败） |
| `LoginEvent` | `VendureEvent` | 登录成功 |
| `LogoutEvent` | `VendureEvent` | 登出 |
| `PasswordResetEvent` | `VendureEvent` | 密码重置请求 |
| `PasswordResetVerifiedEvent` | `VendureEvent` | 密码重置验证完成 |
| `IdentifierChangeRequestEvent` | `VendureEvent` | 标识符变更请求 |
| `IdentifierChangeEvent` | `VendureEvent` | 标识符变更完成 |

#### Settings & Admin
| 事件类 | 继承自 | 事件触发场景 |
|--------|--------|-------------|
| `ChannelEvent` | `VendureEntityEvent<Channel>` | Channel CRUD |
| `ChangeChannelEvent` | `VendureEntityEvent` | 实体分配到 Channel |
| `AdministratorEvent` | `VendureEntityEvent<Administrator>` | 管理员 CRUD |
| `RoleEvent` | `VendureEntityEvent<Role>` | 角色 CRUD |
| `RoleChangeEvent` | `VendureEvent` | 角色成员变更 |
| `ApiKeyEvent` | `VendureEntityEvent<ApiKey>` | API Key CRUD |
| `GlobalSettingsEvent` | `VendureEntityEvent<GlobalSettings>` | 全局设置更新 |
| `ShippingMethodEvent` | `VendureEntityEvent<ShippingMethod>` | 配送方式 CRUD |
| `PromotionEvent` | `VendureEntityEvent<Promotion>` | 促销 CRUD |
| `TaxCategoryEvent` | `VendureEntityEvent<TaxCategory>` | 税种 CRUD |
| `TaxRateEvent` | `VendureEntityEvent<TaxRate>` | 税率 CRUD |
| `TaxRateModificationEvent` | `VendureEvent` | 税率修改（与 TaxRateEvent 同时发布） |
| `CountryEvent` | `VendureEntityEvent<Country>` | 国家 CRUD |
| `ProvinceEvent` | `VendureEntityEvent<Province>` | 省份 CRUD |
| `ZoneEvent` | `VendureEntityEvent<Zone>` | 区域 CRUD |
| `ZoneMembersEvent` | `VendureEvent` | 区域成员分配/移除 |
| `SellerEvent` | `VendureEntityEvent<Seller>` | 卖家 CRUD |

#### Other
| 事件类 | 继承自 | 事件触发场景 |
|--------|--------|-------------|
| `HistoryEntryEvent` | `VendureEntityEvent<HistoryEntry>` | 历史记录 CRUD（order / customer 两种上下文） |
| `SearchEvent` | `VendureEvent` | 搜索查询执行 |
| `InitializerEvent` | `VendureEvent` | 数据初始化完成 |
| `BootstrappedEvent` | `VendureEvent` | 服务器/Worker 启动完成 |

#### Email 插件（`@vendure/email-plugin`）
| 事件类 | 继承自 | 事件触发场景 |
|--------|--------|-------------|
| `EmailSendEvent` | `VendureEvent` | 邮件发送成功或失败 |

---

## 3. EventBus 核心实现详解

源文件：`core/src/event-bus/event-bus.ts`

### 3.1 数据结构

```typescript
@Injectable()
@Instrument()
export class EventBus implements OnModuleDestroy {
    private eventStream = new Subject<VendureEvent>();           // RxJS Subject，所有事件的中心管道
    private destroy$ = new Subject<void>();                       // 模块销毁信号
    private blockingEventHandlers = new Map<Type<VendureEvent>,   // 阻塞处理器注册表
        Array<BlockingEventHandlerOptions<any>>>();

    constructor(private transactionSubscriber: TransactionSubscriber) {}
}
```

### 3.2 publish() 方法

```typescript
async publish<T extends VendureEvent>(event: T): Promise<void> {
    this.eventStream.next(event);                    // 1. 推入 Subject → 所有 ofType/filter 订阅者异步收到
    await this.executeBlockingEventHandlers(event);   // 2. 同步等待阻塞处理器执行完毕
}
```

**执行顺序**：
1. 先将事件推入 `eventStream`，触发异步订阅流程（但 ofType/filter 订阅者会先等待事务提交）
2. 然后顺序执行所有阻塞处理器，`await` 等待每个完成
3. 阻塞处理器执行时间超过 100ms 会打印警告

### 3.3 ofType() 方法 — 异步订阅

```typescript
ofType<T extends VendureEvent>(type: Type<T>): Observable<T> {
    return this.eventStream.asObservable().pipe(
        takeUntil(this.destroy$),
        filter(e => e.constructor === type),           // 严格类型匹配（非 instanceof）
        mergeMap(event => this.awaitActiveTransactions(event)),  // 等待事务提交
        filter(notNullOrUndefined),                     // 事务回滚时返回 undefined，过滤掉
    ) as Observable<T>;
}
```

**关键点**：
- `e.constructor === type` — 精确匹配，**子类不会匹配父类**（与 `filter()` 不同）
- `awaitActiveTransactions` — 事件中包含 `RequestContext` 时，等事务 commit 后再向下传递

### 3.4 filter() 方法 — 自定义谓词订阅

```typescript
filter<T extends VendureEvent>(predicate: (event: VendureEvent) => boolean): Observable<T> {
    return this.eventStream.asObservable().pipe(
        takeUntil(this.destroy$),
        filter(e => predicate(e)),
        mergeMap(event => this.awaitActiveTransactions(event)),
        filter(notNullOrUndefined),
    ) as Observable<T>;
}
```

与 `ofType` 的区别：`filter` 支持自定义谓词，可以使用 `instanceof` 来匹配子类。

### 3.5 registerBlockingEventHandler() — 阻塞处理器

```typescript
registerBlockingEventHandler<T extends VendureEvent>(handlerOptions: BlockingEventHandlerOptions<T>) {
    // handlerOptions 包含:
    //   event: Type<T> | Array<Type<T>>   — 监听的事件类型
    //   handler: (event: T) => void | Promise<void>  — 处理函数
    //   id: string                         — 唯一标识，用于 before/after 排序
    //   before?: string                    — 在指定 id 的处理器之前执行
    //   after?: string                     — 在指定 id 的处理器之后执行
}
```

**执行语义**：
- 与 publish() 在同一个调用栈中同步执行
- 处理器抛异常 → publish() 也会抛异常 → 事务回滚
- 处理器在**同一事务**中执行（只要将 event.ctx 传给 TransactionalConnection）
- 支持 `before` / `after` 排序，支持循环依赖检测

---

## 4. 事件发布位置汇总

`eventBus.publish()` 调用并非全部在 Service 层。按**发布者角色**分为以下五类：

---

### 4.1 启动流程 — 无事务上下文的事件

这两处发布发生在 Nest 应用初始化阶段，**不携带 `RequestContext`**，因此不存在事务等待问题，订阅者会立即收到事件。

| 文件 | 事件 | 行号 | 触发时机 |
|------|------|------|----------|
| `core/src/bootstrap.ts` | `BootstrappedEvent` | :228 | Server 进程 `app.listen()` 完成后 |
| `core/src/bootstrap.ts` | `BootstrappedEvent` | :280 | Worker 进程 `NestFactory.createApplicationContext()` 完成后 |
| `core/src/service/initializer.service.ts` | `InitializerEvent` | :59 | `onModuleInit()` 中所有服务初始化完成后 |

**启动时序**：

```
bootstrapServer()
  ├── NestFactory.create(AppModule)
  ├── InitializerService.onModuleInit()
  │     ├── zoneService.initZones()
  │     ├── globalSettingsService.initGlobalSettings()
  │     ├── sellerService.initSellers()
  │     ├── channelService.initChannels()
  │     ├── roleService.initRoles()
  │     ├── administratorService.initAdministrators()
  │     ├── shippingMethodService.initShippingMethods()
  │     ├── taxRateService.initTaxRates()
  │     ├── stockLocationService.initStockLocations()
  │     └── eventBus.publish(new InitializerEvent())      ← ① InitializerEvent
  ├── app.listen(port)
  └── eventBus.publish(new BootstrappedEvent())           ← ② BootstrappedEvent

bootstrapWorker()
  ├── NestFactory.createApplicationContext(WorkerModule)
  ├── validateDbTablesForWorker()
  └── eventBus.publish(new BootstrappedEvent())           ← ③ BootstrappedEvent (Worker)
```

---

### 4.2 Service 层 CRUD 事件 — 在 create/update/delete 方法末尾

这是数量最多的发布点。典型模式（以 `ProductService` 为例）：

```typescript
// core/src/service/services/product.service.ts:257
async create(ctx: RequestContext, input: CreateProductInput): Promise<Product> {
    const product = await this.translatableSaver.create(...);
    await this.eventBus.publish(new ProductEvent(ctx, product, 'created', input));
    return product;
}
```

**事务上下文**：这些调用发生在 Service 方法内，而 Service 方法通常由 `@Transaction()` 装饰的 Resolver 调用。此时 `ctx` 携带 `TRANSACTION_MANAGER_KEY`，因此 ofType/filter 订阅者会**等事务提交后才收到事件**。

| Service 文件 | 事件 | 操作与行号 |
|-------------|------|-----------|
| `product.service.ts` | ProductEvent | created:257, updated:288, deleted:304 |
| `product.service.ts` | ProductChannelEvent | assigned:369, removed:431 |
| `product.service.ts` | ProductOptionGroupChangeEvent | :461, :504 |
| `product-variant.service.ts` | ProductVariantEvent | created:392, updated:407, deleted:713 |
| `product-variant.service.ts` | ProductVariantPriceEvent | created:617, updated:641, deleted:681 |
| `product-variant.service.ts` | ProductVariantChannelEvent | assigned:659, removed:697 |
| `product-option.service.ts` | ProductOptionEvent | created:118, updated:130, deleted:169 |
| `product-option-group.service.ts` | ProductOptionGroupEvent | created:168, updated:183, deleted:241,302 |
| `customer.service.ts` | CustomerEvent | created:295,692, updated:368, deleted:807 |
| `customer.service.ts` | CustomerAddressEvent | created:725, updated:762, deleted:792 |
| `customer-group.service.ts` | CustomerGroupEvent | created:112, updated:126, deleted:135 |
| `customer-group.service.ts` | CustomerGroupChangeEvent | assigned:168, removed:194 |
| `facet.service.ts` | FacetEvent | created:187, updated:204, deleted:233,237 |
| `channel.service.ts` | ChannelEvent | created:368, updated:464, deleted:482 |
| `channel.service.ts` | ChangeChannelEvent | assigned:125 |
| `administrator.service.ts` | AdministratorEvent | created:148, updated:214, deleted:274 |
| `role.service.ts` | RoleEvent | created:280, updated:315, deleted:329 |
| `administrator.service.ts` | RoleChangeEvent | assigned:205, removed:206 |
| `asset.service.ts` | AssetEvent | created:324, updated:366, deleted:546 |
| `asset.service.ts` | AssetChannelEvent | removed:418,434 |
| `collection.service.ts` | CollectionEvent | created:557, updated:586, deleted:614 |
| `collection.service.ts` | CollectionModificationEvent | :584, :1028 |
| `country.service.ts` | CountryEvent | created:106, updated:117, deleted:137 |
| `promotion.service.ts` | PromotionEvent | created:152, updated:191, deleted:200 |
| `shipping-method.service.ts` | ShippingMethodEvent | created:138, updated:182, deleted:193 |
| `tax-category.service.ts` | TaxCategoryEvent | created:66, updated:82, deleted:106 |
| `tax-rate.service.ts` | TaxRateEvent | created:129, updated:165, deleted:175 |
| `tax-rate.service.ts` | TaxRateModificationEvent | created:128, updated:164 |
| `zone.service.ts` | ZoneEvent | created:128, updated:138, deleted:177 |
| `zone.service.ts` | ZoneMembersEvent | assigned:197, removed:211 |
| `province.service.ts` | ProvinceEvent | created:78, updated:89, deleted:98 |
| `seller.service.ts` | SellerEvent | created:65, updated:79, deleted:87 |
| `stock-location.service.ts` | StockLocationEvent | created:94, updated:108, deleted:170 |
| `payment-method.service.ts` | PaymentMethodEvent | created:114, updated:146, deleted:173,190 |
| `global-settings.service.ts` | GlobalSettingsEvent | updated:80 |
| `history.service.ts` | HistoryEntryEvent | order-created:299,340, order-updated:365, order-deleted:373, customer-updated:392, customer-deleted:400 |
| `order.service.ts` | OrderEvent | created:469,483, updated:518,552,615, deleted:2099 |
| `order.service.ts` | OrderLineEvent | deleted:885,973 |
| `order.service.ts` | CouponCodeEvent | assigned:1080, removed:1098 |
| `order.service.ts` | RefundEvent | created:1963 |
| `fulfillment.service.ts` | FulfillmentEvent | created:114 |
| `stock-movement.service.ts` | StockMovementEvent | adjust:126, allocate:193, sale:254, cancel:309, release:358 |

---

### 4.3 订单流程编排 — OrderProcess 回调内发布

这是最复杂的发布路径。事件不在 Service 方法中直接发布，而是在 **状态机转换回调** 中发布，由 `OrderStateMachine.transition()` → `FSM.transitionTo()` → `onTransitionEnd` 链路触发。

#### 4.3.1 OrderStateMachine 的回调分发机制

`OrderStateMachine`（`core/src/service/helpers/order-state-machine/order-state-machine.ts`）本身**不发布任何事件**，也不持有 `eventBus` 引用。它的职责是：

1. 合并所有 `OrderProcess` 的 transitions 定义
2. 在 `onTransitionStart` / `onTransitionEnd` / `onTransitionError` 中依次调用每个 `OrderProcess` 的同名回调

```typescript
// order-state-machine.ts:76-81
onTransitionEnd: async (fromState, toState, data) => {
    for (const process of orderProcesses) {
        if (typeof process.onTransitionEnd === 'function') {
            await awaitPromiseOrObservable(process.onTransitionEnd(fromState, toState, data));
        }
    }
},
```

关键点：`OrderStateMachine.transition()` 在 `OrderService.transitionToState()` 的 **`withTransaction` 回调内** 被调用，因此 `onTransitionEnd` 的执行也在同一事务中。

#### 4.3.2 defaultOrderProcess — OrderPlacedEvent 的发布

`defaultOrderProcess`（`core/src/config/order/default-order-process.ts`）是 Vendure 内置的 `OrderProcess` 实现。它在 `onTransitionEnd` 中发布 `OrderPlacedEvent`：

```typescript
// default-order-process.ts:402-426
async onTransitionEnd(fromState, toState, data) {
    const { ctx, order } = data;
    const { orderPlacedStrategy } = configService.orderOptions;
    if (order.active) {
        const shouldSetAsPlaced = orderPlacedStrategy.shouldSetAsPlaced(ctx, fromState, toState, order);
        if (shouldSetAsPlaced) {
            order.active = false;
            order.orderPlacedAt = new Date();
            // ... 更新 OrderLine.orderPlacedQuantity ...
            await eventBus.publish(new OrderPlacedEvent(fromState, toState, ctx, order));  // ← :423
            await orderSplitter.createSellerOrders(ctx, order);
        }
    }
    // ... 库存分配、历史记录 ...
}
```

**发布链路**：
```
OrderService.transitionToState(ctx, orderId, 'PaymentSettled')
  → connection.withTransaction(ctx, async txCtx => {
      OrderStateMachine.transition(txCtx, order, 'PaymentSettled')
        → FSM.transitionTo()
          → onTransitionEnd(fromState, toState, { ctx: txCtx, order })
            → defaultOrderProcess.onTransitionEnd()
              → eventBus.publish(new OrderPlacedEvent(...))
      // 回到 OrderService：
      await this.eventBus.publish(new OrderStateTransitionEvent(fromState, state, txCtx, order));
  })
```

**事务边界**：`OrderPlacedEvent` 和 `OrderStateTransitionEvent` 都在**同一个 `withTransaction` 回调**中发布。`OrderPlacedEvent` 先于 `OrderStateTransitionEvent` 进入 `eventStream`，订阅者按相同顺序接收。

#### 4.3.3 defaultOrderProcess 的 init() — 懒注入 EventBus

`defaultOrderProcess` 不是 NestJS Provider，不能通过构造函数注入。它通过 `OrderProcess.init(injector)` 钩子延迟获取依赖：

```typescript
// default-order-process.ts:237-258
async init(injector) {
    const EventBus = await import('../../event-bus/index.js').then(m => m.EventBus);
    const StockMovementService = await import('../../service/index.js').then(m => m.StockMovementService);
    // ... 其他懒导入 ...
    connection = injector.get(TransactionalConnection);
    eventBus = injector.get(EventBus);
    stockMovementService = injector.get(StockMovementService);
    // ...
}
```

使用动态 `import()` 避免循环依赖——`defaultOrderProcess` 是 `DefaultConfig` 的一部分，在模块加载时就会求值，如果静态导入 EventBus 会形成循环。

#### 4.3.4 自定义 OrderProcess 的扩展点

用户通过 `configureDefaultOrderProcess()` 或自定义 `OrderProcess` 可以：
- 在 `onTransitionEnd` 中发布自己的事件（通过 `init()` 注入 EventBus）
- 在 `onTransitionStart` 中阻止状态转换（返回 `false` 或错误消息字符串）
- 在 `onTransitionError` 中记录日志

#### 4.3.5 其他状态转换事件的发布

除 `OrderPlacedEvent` 在 OrderProcess 回调中发布外，其他状态转换事件（`OrderStateTransitionEvent`、`PaymentStateTransitionEvent`、`FulfillmentStateTransitionEvent`、`RefundStateTransitionEvent`）均在 Service 层的 `withTransaction` 回调内发布：

| Service 文件 | 事件 | 行号 | 事务上下文 |
|-------------|------|------|-----------|
| `order.service.ts` | OrderStateTransitionEvent | :1309 | `withTransaction(ctx, txCtx => { ... })` |
| `order.service.ts` | RefundStateTransitionEvent | :1362, :1990 | `withTransaction(ctx, txCtx => { ... })` |
| `payment.service.ts` | PaymentStateTransitionEvent | :157, :256, :302 | `withTransaction(ctx, txCtx => { ... })` |
| `payment.service.ts` | RefundStateTransitionEvent | :452 | `withTransaction(ctx, txCtx => { ... })` |
| `fulfillment.service.ts` | FulfillmentStateTransitionEvent | :199 | `withTransaction(ctx, txCtx => { ... })` |

**注意**：这些事件的 `ctx` 参数使用的是 `txCtx`（事务上下文副本），携带活跃的 `TRANSACTION_MANAGER_KEY`。

---

### 4.4 辅助组件 — Helper 与 Plugin 内部的发布

这些发布点不在标准 Service 的 CRUD 方法中，而在辅助逻辑或插件内部。

#### 4.4.1 OrderModifier — 订单修改辅助器

`OrderModifier`（`core/src/service/helpers/order-modifier/order-modifier.ts`）处理订单修改（管理员修改已下单订单），独立发布事件：

| 事件 | 行号 | 操作 |
|------|------|------|
| `OrderLineEvent` | :204 | created — 修改中新增订单行 |
| `OrderLineEvent` | :247 | updated — 修改中更新订单行数量 |
| `OrderLineEvent` | :339 | cancelled — 修改中取消订单行 |
| `OrderEvent` | :695 | updated — 修改完成后更新整个订单 |

**调用链路**：
```
OrderService.modifyOrder(ctx, input)
  → connection.withTransaction(ctx, async txCtx => {
      OrderModifier.modifyOrder(txCtx, input, order)
        → OrderModifier.addOrderLine(ctx, ...)
          → eventBus.publish(new OrderLineEvent(ctx, order, orderLine, 'created'))
        → eventBus.modifyOrderLines(ctx, ...)
          → eventBus.publish(new OrderLineEvent(ctx, order, orderLine, 'cancelled'))
        → eventBus.publish(new OrderEvent(ctx, order, 'updated', input))
    })
```

#### 4.4.2 FulltextSearchService — 搜索服务

`FulltextSearchService`（`core/src/plugin/default-search-plugin/fulltext-search.service.ts`）在搜索查询执行时发布：

| 事件 | 行号 |
|------|------|
| `SearchEvent` | :58 |

```typescript
async search(ctx: RequestContext, input: SearchInput): Promise<SearchResult> {
    await this.eventBus.publish(new SearchEvent(ctx, input));
    // ... 执行搜索 ...
}
```

#### 4.4.3 EmailProcessor — 邮件插件

`EmailProcessor`（`email-plugin/src/email-processor.ts`）在邮件发送后发布：

| 事件 | 行号 | 场景 |
|------|------|------|
| `EmailSendEvent` | :83 | 邮件发送成功 |
| `EmailSendEvent` | :94 | 邮件发送失败 |

```typescript
async process(data: EmailJobData) {
    try {
        await this.emailSender.send(emailDetails, transportSettings);
        await this.eventBus.publish(new EmailSendEvent(ctx, emailDetails, true, undefined, data.metadata));
    } catch (err) {
        await this.eventBus.publish(new EmailSendEvent(ctx, emailDetails, false, err, data.metadata));
        throw err;
    }
}
```

**特别注意**：`EmailProcessor` 运行在 Worker 的 JobQueue 中，不在 HTTP 请求的事务上下文中。`EmailSendEvent` 携带的 `ctx` 可能已不含活跃事务，订阅者会立即收到。

---

### 4.5 认证/账户事件 — 全事务包装的发布

**核心修正**：这些方法的 Resolver **全部**都有 `@Transaction()` 装饰器！之前的"无显式事务包装"结论是错误的。

| 事件 | Service 文件 | 行号 | Resolver 入口 | Resolver 文件 | @Transaction() |
|------|-------------|------|--------------|--------------|---------------|
| `AttemptedLoginEvent` | `auth.service.ts` | :57 | `login()` / `authenticate()` | `shop-auth.resolver.ts` / `auth.resolver.ts` | 有 (Shop:65, Admin:38) |
| `LoginEvent` | `auth.service.ts` | :109 | `login()` / `authenticate()` | `shop-auth.resolver.ts` / `auth.resolver.ts` | 有 |
| `LogoutEvent` | `auth.service.ts` | :152 | `logout()` | `shop-auth.resolver.ts` / `auth.resolver.ts` | 有 (Shop:93, Admin:66) |
| `AccountRegistrationEvent` | `customer.service.ts` | :272, :453, :476 | `create()` / `registerCustomerAccount()` / `refreshVerificationToken()` | `customer.resolver.ts` / `shop-auth.resolver.ts` | 有 |
| `AccountVerifiedEvent` | `customer.service.ts` | :510 | `verifyCustomerAccount()` | `shop-auth.resolver.ts` | 有 (:133) |
| `PasswordResetEvent` | `customer.service.ts` | :522 | `requestPasswordReset()` | `shop-auth.resolver.ts` | 有 (:196) |
| `PasswordResetVerifiedEvent` | `customer.service.ts` | :562 | `resetPassword()` | `shop-auth.resolver.ts` | 有 (:211) |
| `IdentifierChangeRequestEvent` | `customer.service.ts` | :606 | `requestUpdateCustomerEmailAddress()` | `shop-auth.resolver.ts` | 有 (:282) |
| `IdentifierChangeEvent` | `customer.service.ts` | :614, :649 | `requestUpdateEmailAddress()` / `updateEmailAddress()` | `shop-auth.resolver.ts` | 有 |
| `CustomerEvent` (created) | `customer.service.ts` | :295, :692 | `create()` | `customer.resolver.ts` | 有 |
| `CustomerEvent` (updated) | `customer.service.ts` | :368 | `update()` / `updateCustomer()` | `customer.resolver.ts` / `shop-customer.resolver.ts` | 有 (Shop:35) |

#### 4.5.1 事务传播链路（以 login 为例）

```
ShopAuthResolver.login(@Ctx ctx)      ← ctx: 原始 RequestContext
  @Transaction() decorator
    ↓
TransactionInterceptor.intercept()
    ↓
TransactionWrapper.executeInTransaction(ctx, work, ...)
    ├── ctx = originalCtx.copy()      ← 浅拷贝
    ├── queryRunner = createQueryRunner()
    ├── queryRunner.startTransaction()
    ├── ctx[TRANSACTION_MANAGER_KEY] = queryRunner.manager  ← 挂事务管理器
    ├── work(ctx)
    │   ├── internal_setRequestContext(req, ctx, context)  ← 将新 ctx 绑到请求
    │   └── super.baseLogin(...)
    │       └── authService.authenticate(ctx, ...)
    │           ├── eventBus.publish(new AttemptedLoginEvent(ctx, ...))  ← ① 事务内发布
    │           ├── authenticationStrategy.authenticate()
    │           └── createAuthenticatedSessionForUser(ctx, user)
    │               ├── ...
    │               └── eventBus.publish(new LoginEvent(ctx, user))  ← ② 事务内发布
    ├── await queryRunner.commitTransaction()  ← ③ 事务提交
    └── await queryRunner.release()
```

---

### 4.5.2 事件发布/跳过条件的完整分支分析

#### 4.5.2.1 `AttemptedLoginEvent` — auth.service.ts:57

**发布条件**：**无条件发布**（方法开头第一行）

```typescript
async authenticate(ctx, apiType, authenticationMethod, authenticationData) {
    await this.eventBus.publish(new AttemptedLoginEvent(ctx, ...));  // ← 总是执行
    // ... 后续验证逻辑
}
```

**跳过情况**：无。只要方法被调用就会发布。

**事务行为**：
- 认证成功 → 正常流程 → 事务 commit → 订阅者收到
- 认证失败 → `return InvalidCredentialsError`（不抛异常）→ 事务 commit → **订阅者也收到**

这是有意设计的：用于登录审计和暴力破解检测，无论成功失败都应该记录。

---

#### 4.5.2.2 `LoginEvent` — auth.service.ts:109

**发布条件**：在 `createAuthenticatedSessionForUser()` 中发布，需要全部满足：

| 条件 | 代码位置 | 说明 |
|------|---------|------|
| 认证策略返回 User 对象 | authenticate():74 | 不是 string 也不是 null/undefined |
| 用户已验证 OR 不需要验证 | createAuthenticatedSessionForUser():96 | `!extAuths.length && requireVerification && !user.verified` 为 false |

**跳过情况**（不发布 LoginEvent，但 AttemptedLoginEvent 已发布）：

1. **认证策略返回错误字符串** → `return InvalidCredentialsError` → 提前返回，不走到 createAuthenticatedSessionForUser
2. **认证策略返回 null/undefined** → `return InvalidCredentialsError` → 提前返回
3. **需要验证但用户未验证** → `return NotVerifiedError` → 提前返回，不执行到 :109

**代码路径**：
```typescript
async createAuthenticatedSessionForUser(ctx, user, ...) {
    // ... 加载 roles ...
    if (!extAuths.length && requireVerification && !user.verified) {
        return new NotVerifiedError();  // ← 跳过 publish，直接返回
    }
    // ... 更新 lastLogin、创建 session ...
    await this.eventBus.publish(new LoginEvent(ctx, user));  // ← 只有全部通过才执行
    return session;
}
```

---

#### 4.5.2.3 `LogoutEvent` — auth.service.ts:152

**发布条件**：`session` 存在

```typescript
async destroyAuthenticatedSession(ctx, sessionToken) {
    const session = await this.connection.getRepository(...).findOne(...);
    if (session) {                      // ← 关键条件
        // ... 调用 onLogOut 回调 ...
        await this.eventBus.publish(new LogoutEvent(ctx));  // ← 找到 session 才发布
        return this.sessionService.deleteSessionsByUser(...);
    }
    // 没找到 session → 直接返回，不发布
}
```

**跳过情况**：
- token 无效 → 找不到 session → 不发布
- 使用 API Key 登录 → ShopAuthResolver.logout() 直接 `return { success: false }`，不调用 `destroyAuthenticatedSession`

---

#### 4.5.2.4 `AccountRegistrationEvent` — 三个发布点

##### 发布点 1：Admin API `create()` — customer.service.ts:272

**发布条件**：**无条件发布**（只要前面的检查都通过）

```typescript
async create(ctx, input, password?) {
    // ... 一系列提前返回检查 ...
    //    - existingCustomerInChannel → return EmailAddressConflictAdminError
    //    - existingCustomer || existingUser → return EmailAddressConflictAdminError
    //    - createCustomerUser 失败 → throw customerUser (异常，回滚)
    
    await this.eventBus.publish(new AccountRegistrationEvent(ctx, customer.user));  // ← 检查都通过后执行
    // ...
}
```

**跳过情况**：
1. 邮箱已在当前 Channel 存在 → `return EmailAddressConflictAdminError` → 不发布
2. Customer/User 不一致状态 → `return EmailAddressConflictAdminError` → 不发布
3. 创建 User 失败 → `throw customerUser` → 事务回滚 → 事件被过滤

##### 发布点 2：Shop API `registerCustomerAccount()` — customer.service.ts:453

**发布条件**：`!user.verified`（用户未验证）

```typescript
async registerCustomerAccount(ctx, input) {
    // ... 一系列提前返回检查 ...
    //    - 不需要验证但没密码 → return MissingPasswordError
    //    - 用户已验证且有本地认证方法 → return { success: true }
    //    - createOrUpdate 失败 → return customer
    //    - createCustomerUser 失败 → return customerUser
    //    - addNativeAuthenticationMethod 失败 → return addAuthenticationResult
    
    if (!user.verified) {                                // ← 关键条件
        await this.eventBus.publish(new AccountRegistrationEvent(ctx, user));
    } else {
        // 已验证用户 → 只写历史记录，不发布事件
    }
}
```

**跳过情况**：
1. 不需要验证但没提供密码 → `return MissingPasswordError` → 不发布（方法开头就返回）
2. 用户已验证且有本地认证方法 → `return { success: true }` → 不发布
3. 创建 Customer 失败 → `return customer`（ErrorResult）→ 不发布（提前返回）
4. 创建 User 失败 → `return customerUser`（ErrorResult）→ 不发布（提前返回）
5. 添加认证方法失败 → `return addAuthenticationResult`（ErrorResult）→ 不发布（提前返回）
6. **用户已验证** → 进入 else 分支 → 只写历史记录，**不发布事件**

##### 发布点 3：`refreshVerificationToken()` — customer.service.ts:476

**发布条件**：`user && !user.verified`（用户存在且未验证）

```typescript
async refreshVerificationToken(ctx, emailAddress) {
    const user = await this.userService.getUserByEmailAddress(...);
    if (user && !user.verified) {                  // ← 关键条件
        await this.userService.setVerificationToken(ctx, user);
        await this.eventBus.publish(new AccountRegistrationEvent(ctx, user));
    }
    // 用户不存在 OR 已验证 → 直接返回，不发布
}
```

**跳过情况**：
- 用户不存在 → 不发布
- 用户已验证 → 不发布

---

#### 4.5.2.5 `AccountVerifiedEvent` — customer.service.ts:510

**发布条件**：`userService.verifyUserByToken()` 成功返回 User

```typescript
async verifyCustomerEmailAddress(ctx, verificationToken, password?) {
    const result = await this.userService.verifyUserByToken(...);
    if (isGraphQlErrorResult(result)) {
        return result;  // ← 验证失败 → 提前返回，不发布
    }
    const customer = await this.findOneByUserId(...);
    if (!customer) {
        throw new InternalServerError(...);  // ← 抛异常 → 事务回滚 → 事件被过滤
    }
    // ... 分配到 Channel、写历史记录 ...
    await this.eventBus.publish(new AccountVerifiedEvent(ctx, customer));  // ← 全部通过才发布
    return user;
}
```

**跳过情况**：
1. token 无效 → `return VerificationTokenInvalidError` → 不发布
2. token 过期 → `return VerificationTokenExpiredError` → 不发布
3. 缺少密码 → `return MissingPasswordError` → 不发布
4. 密码已设置 → `return PasswordAlreadySetError` → 不发布
5. 密码验证失败 → `return PasswordValidationError` → 不发布
6. 找不到 Customer → `throw InternalServerError` → 事务回滚 → 事件被过滤

---

#### 4.5.2.6 `PasswordResetEvent` — customer.service.ts:522

**发布条件**：`user` 存在（`setPasswordResetToken` 返回非 undefined）

```typescript
async requestPasswordReset(ctx, emailAddress) {
    const user = await this.userService.setPasswordResetToken(ctx, emailAddress);
    if (user) {                             // ← 关键条件
        await this.eventBus.publish(new PasswordResetEvent(ctx, user));
        // ... 写历史记录 ...
    }
    // 用户不存在 → 直接返回，不发布
}
```

**userService.setPasswordResetToken() 返回 undefined 的情况**：
1. 找不到该邮箱的 User
2. User 没有 NativeAuthenticationMethod

**设计意图**：用户不存在时**不发布事件也不报错**，防止账户枚举攻击（攻击者不能通过"是否收到邮件"判断邮箱是否已注册）。

---

#### 4.5.2.7 `PasswordResetVerifiedEvent` — customer.service.ts:562

**发布条件**：`userService.resetPasswordByToken()` 成功返回 User

```typescript
async resetPassword(ctx, passwordResetToken, password) {
    const result = await this.userService.resetPasswordByToken(...);
    if (isGraphQlErrorResult(result)) {
        return result;  // ← 失败 → 提前返回，不发布
    }
    const customer = await this.findOneByUserId(...);
    if (!customer) {
        throw new InternalServerError(...);  // ← 抛异常 → 事务回滚
    }
    // ... 写历史记录 ...
    await this.eventBus.publish(new PasswordResetVerifiedEvent(ctx, result));  // ← 全部通过才发布
    return result;
}
```

**跳过情况**：
1. token 无效 → `return PasswordResetTokenInvalidError` → 不发布
2. token 过期 → `return PasswordResetTokenExpiredError` → 不发布
3. 密码验证失败 → `return PasswordValidationError` → 不发布
4. 找不到 Customer → `throw InternalServerError` → 事务回滚 → 事件被过滤

---

#### 4.5.2.8 `IdentifierChangeRequestEvent` — customer.service.ts:606

**发布条件**：全部满足：
1. 新邮箱没有冲突
2. User 存在
3. Customer 存在
4. `requireVerification === true`

```typescript
async requestUpdateEmailAddress(ctx, userId, newEmailAddress) {
    if (userWithConflictingIdentifier) {
        return new EmailAddressConflictError();  // ← 邮箱冲突 → 提前返回
    }
    const user = await this.userService.getUserById(...);
    if (!user) return false;                     // ← 找不到 User → 提前返回
    const customer = await this.findOneByUserId(...);
    if (!customer) return false;                 // ← 找不到 Customer → 提前返回
    
    if (this.configService.authOptions.requireVerification) {  // ← 关键条件
        // ... 设置 token ...
        await this.eventBus.publish(new IdentifierChangeRequestEvent(ctx, user));
        return true;
    } else {
        // 不需要验证 → 直接更新，发布 IdentifierChangeEvent
    }
}
```

**跳过情况**：
1. 新邮箱已被使用 → `return EmailAddressConflictError` → 不发布
2. 找不到 User → `return false` → 不发布
3. 找不到 Customer → `return false` → 不发布
4. **不需要验证** → 进入 else 分支 → 发布 `IdentifierChangeEvent` 而不是 `IdentifierChangeRequestEvent`

---

#### 4.5.2.9 `IdentifierChangeEvent` — 两个发布点

##### 发布点 1：不需要验证时 — customer.service.ts:614

```typescript
if (this.configService.authOptions.requireVerification) {
    // ... 发布 IdentifierChangeRequestEvent
} else {
    // ... 直接更新 User 和 Customer ...
    await this.eventBus.publish(new IdentifierChangeEvent(ctx, user, oldIdentifier));
    // ... 写历史记录 ...
}
```

##### 发布点 2：验证通过后 — customer.service.ts:649

```typescript
async updateEmailAddress(ctx, token) {
    const result = await this.userService.changeIdentifierByToken(...);
    if (isGraphQlErrorResult(result)) {
        return result;  // ← 验证失败 → 提前返回，不发布
    }
    const { user, oldIdentifier } = result;
    if (!user) return false;                     // ← 找不到 User → 不发布
    const customer = await this.findOneByUserId(...);
    if (!customer) return false;                 // ← 找不到 Customer → 不发布
    
    await this.eventBus.publish(new IdentifierChangeEvent(ctx, user, oldIdentifier));  // ← 全部通过才发布
    // ... 更新 Customer email、写历史记录 ...
}
```

**跳过情况**（updateEmailAddress 路径）：
1. token 无效 → `return IdentifierChangeTokenInvalidError` → 不发布
2. token 过期 → `return IdentifierChangeTokenExpiredError` → 不发布
3. 找不到 User → `return false` → 不发布
4. 找不到 Customer → `return false` → 不发布

---

### 4.5.3 统一口径：三类分支下的发布/跳过规则

按**冲突分支**、**校验失败分支**、**成功分支**三类，统一每个事件的行为：

| 事件 | 冲突分支 | 校验失败分支 | 成功分支 | 最终结论 |
|------|---------|-------------|---------|---------|
| **`AttemptedLoginEvent`** | 无冲突场景 | 密码错误：<br>✅ 已发布<br>⚠️ return `InvalidCredentialsError`<br>✅ 事务 commit<br>✅ **订阅者收到** | 认证成功：<br>✅ 已发布<br>✅ 事务 commit<br>✅ **订阅者收到** | 🔴 **总是收到**<br>（方法开头无条件发布） |
| **`LoginEvent`** | 无冲突场景 | 密码错误：<br>❌ 提前 return<br>❌ 不走到 publish<br>❌ 订阅者收不到<br><br>未验证用户：<br>❌ return `NotVerifiedError`<br>❌ 不走到 publish<br>❌ 订阅者收不到 | 认证成功+已验证：<br>✅ 走到 publish<br>✅ 事务 commit<br>✅ **订阅者收到** | 🟡 **仅成功时收到** |
| **`LogoutEvent`** | 无冲突场景 | token 无效：<br>❌ 不进入 `if (session)` 块<br>❌ 不发布<br>❌ 订阅者收不到 | 找到有效 session：<br>✅ 走到 publish<br>✅ 事务 commit<br>✅ **订阅者收到** | 🟡 **仅成功时收到** |
| **`AccountRegistrationEvent`**<br>(Shop register) | 邮箱已验证且有本地认证：<br>❌ return `{ success: true }`<br>❌ 不走到 publish<br>❌ 订阅者收不到<br><br>邮箱不一致状态：<br>❌ 上游 `createOrUpdate` 返回错误<br>❌ 不走到 publish<br>❌ 订阅者收不到 | 不需要验证但没密码：<br>❌ return `MissingPasswordError`<br>❌ 不走到 publish<br>❌ 订阅者收不到<br><br>密码验证失败：<br>❌ return `PasswordValidationError`<br>❌ 不走到 publish<br>❌ 订阅者收不到 | 成功**且用户未验证**：<br>✅ 走到 publish<br>✅ 事务 commit<br>✅ **订阅者收到**<br><br>成功**但用户已验证**：<br>❌ 进入 else 分支<br>❌ 不发布（只写历史）<br>❌ 订阅者收不到 | 🟡 **仅无冲突、校验通过、用户未验证时收到** |
| **`AccountRegistrationEvent`**<br>(Admin create) | 邮箱已存在：<br>❌ return `EmailAddressConflictAdminError`<br>❌ 不走到 publish<br>❌ 订阅者收不到 | 无校验失败场景 | 无冲突+全部检查通过：<br>✅ 走到 publish<br>✅ 事务 commit<br>✅ **订阅者收到** | 🟡 **仅无冲突且成功时收到** |
| **`AccountRegistrationEvent`**<br>(refresh) | 无冲突场景 | 用户不存在/已验证：<br>❌ 不进入 `if (user && !user.verified)`<br>❌ 不发布<br>❌ 订阅者收不到 | 用户存在且未验证：<br>✅ 走到 publish<br>✅ 事务 commit<br>✅ **订阅者收到** | 🟡 **仅用户存在且未验证时收到** |
| **`AccountVerifiedEvent`** | 无冲突场景 | token 无效/过期：<br>❌ return `VerificationTokenInvalidError`<br>❌ 不走到 publish<br>❌ 订阅者收不到<br><br>缺密码/密码已设/密码无效：<br>❌ return 对应 ErrorResult<br>❌ 不走到 publish<br>❌ 订阅者收不到 | token 验证成功：<br>✅ 走到 publish<br>✅ 事务 commit<br>✅ **订阅者收到** | 🟡 **仅验证成功时收到** |
| **`PasswordResetEvent`** | 无冲突场景 | 用户不存在/无本地认证：<br>❌ `setPasswordResetToken` 返回 undefined<br>❌ 不进入 `if (user)`<br>❌ 不发布<br>❌ 订阅者收不到 | 用户存在且有本地认证：<br>✅ 走到 publish<br>✅ 事务 commit<br>✅ **订阅者收到** | 🟡 **仅用户存在时有收到**<br>（安全设计：不存在时静默） |
| **`PasswordResetVerifiedEvent`** | 无冲突场景 | token 无效/过期：<br>❌ return `PasswordResetTokenInvalidError`<br>❌ 不走到 publish<br>❌ 订阅者收不到<br><br>密码无效：<br>❌ return `PasswordValidationError`<br>❌ 不走到 publish<br>❌ 订阅者收不到 | token 验证成功：<br>✅ 走到 publish<br>✅ 事务 commit<br>✅ **订阅者收到** | 🟡 **仅验证成功时收到** |
| **`IdentifierChangeRequestEvent`** | 新邮箱已存在：<br>❌ return `EmailAddressConflictError`<br>❌ 不走到 publish<br>❌ 订阅者收不到 | 找不到 User/Customer：<br>❌ return `false`<br>❌ 不走到 publish<br>❌ 订阅者收不到<br><br>不需要验证：<br>❌ 进入 else 分支<br>❌ 发布 `IdentifierChangeEvent` 替代 | 邮箱不冲突+找到 User/Customer+需要验证：<br>✅ 走到 publish<br>✅ 事务 commit<br>✅ **订阅者收到** | 🟡 **仅无冲突、找到、需要验证时收到** |
| **`IdentifierChangeEvent`**<br>(不需要验证路径) | 新邮箱已存在：<br>❌ return `EmailAddressConflictError`<br>❌ 不走到 publish<br>❌ 订阅者收不到 | 找不到 User/Customer：<br>❌ return `false`<br>❌ 不走到 publish<br>❌ 订阅者收不到<br><br>需要验证：<br>❌ 进入 if 分支<br>❌ 发布 `IdentifierChangeRequestEvent` 替代 | 邮箱不冲突+找到 User/Customer+不需要验证：<br>✅ 走到 publish<br>✅ 事务 commit<br>✅ **订阅者收到** | 🟡 **仅无冲突、找到、不需要验证时收到** |
| **`IdentifierChangeEvent`**<br>(token 验证路径) | 无冲突场景 | token 无效/过期：<br>❌ return `IdentifierChangeTokenInvalidError`<br>❌ 不走到 publish<br>❌ 订阅者收不到<br><br>找不到 User/Customer：<br>❌ return `false`<br>❌ 不走到 publish<br>❌ 订阅者收不到 | token 验证成功+找到 User/Customer：<br>✅ 走到 publish<br>✅ 事务 commit<br>✅ **订阅者收到** | 🟡 **仅验证成功且找到时收到** |

---

### 4.5.4 关键设计模式总结（与上表对应）

#### 🔴 模式 A："无条件前置发布" — 仅 `AttemptedLoginEvent`

```typescript
// auth.service.ts:57
async authenticate(ctx, ...) {
    await this.eventBus.publish(new AttemptedLoginEvent(ctx, ...));  // ← 方法开头第一行
    // ... 后续验证 ...
    if (失败) return new InvalidCredentialsError();  // ← 已发布了，不影响
}
```

**对应上表**：`AttemptedLoginEvent` 的所有分支都是✅收到

**效果**：无论成功失败，订阅者**总能收到**。用于安全审计（暴力破解检测、登录频率限制）。

---

#### 🟡 模式 B："提前 return ErrorResult" — 不发布也不回滚

大部分事件的冲突/校验失败用这种模式：

```typescript
if (邮箱冲突) return new EmailAddressConflictError();  // ← 提前 return
if (token无效) return new VerificationTokenInvalidError();  // ← 提前 return

// ↓ 永远走不到
await this.eventBus.publish(new AccountRegistrationEvent(ctx, user));
```

**对应上表**：冲突/校验失败列的❌收不到，成功列的✅收到

**效果**：
- 事务 commit（不回滚）
- 但 publish() 调用永远走不到
- 订阅者收不到事件

这是 Vendure 最常用的模式。

---

#### 🟢 模式 C："if 条件包裹 publish()" — 条件发布

用 if 条件包裹 publish，条件不满足时静默跳过：

```typescript
if (user && !user.verified) {  // ← 条件检查
    await this.eventBus.publish(new AccountRegistrationEvent(ctx, user));
}
// 条件不满足时，既不发布也不报错，静默继续
```

**对应上表**：`AccountRegistrationEvent` (refresh)、`PasswordResetEvent`、`LogoutEvent`

**效果**：条件不满足时，既不发布事件，也不报错。常用于可选功能（如用户不存在时不发重置邮件）。

---

#### 🔴 模式 D："throw Exception" — 回滚+过滤

只有在真正意外的情况下才用这种模式：

```typescript
const customer = await this.findOneByUserId(ctx, result.id, false);
if (!customer) {
    throw new InternalServerError('error.cannot-locate-customer-for-user');
}
```

**对应上表**：所有事件的"找不到 Customer"边缘情况（极少发生）

**效果**：
- 事务 rollback
- `awaitActiveTransactions()` 返回 undefined
- `filter(notNullOrUndefined)` 过滤掉事件
- 订阅者收不到事件

---

### 4.5.5 全局结论（与上表一致）

| 事件组 | 订阅者能否收到的关键 |
|--------|---------------------|
| **登录事件** | `AttemptedLoginEvent` 总能收到；`LoginEvent` 仅认证成功+已验证时收到 |
| **注册事件** | 仅"无冲突、校验通过、满足发布条件（如用户未验证）"三者同时满足时收到 |
| **token 验证事件** | 仅 token 有效且未过期时收到 |
| **密码重置事件** | 用户存在且有本地认证方法时收到（不存在时静默，防止枚举） |
| **邮箱变更事件** | 需要验证时发 `IdentifierChangeRequestEvent`，不需要时发 `IdentifierChangeEvent`，二选一 |

**最容易混淆的一点**：业务失败时（返回 ErrorResult）事务仍然 commit，不是 rollback，所以**如果已经发布了事件**（如 AttemptedLoginEvent），订阅者会收到。事件不被收到的原因不是回滚，而是**publish() 调用根本没执行到**。

---

### 4.6 发布路径总览图

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                            EventBus.publish()                                       │
│                                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  ┌────────────┐
│  │  启动流程      │  │  Service CRUD │  │  订单编排      │  │ 辅助组件    │  │ 认证/账户    │
│  │  (无事务)     │  │  (事务)      │  │  (事务)        │  │ (混合事务)  │  │  (事务)      │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤  ├────────────┤  ├────────────┤
│  │ bootstrap.ts │  │ *.service.ts │  │ order.service│  │ OrderModifier│ │ auth.service│
│  │  Bootstrapped│  │  EntityEvent │  │  StateTrans. │  │  OrderLine  │  │ AttemptedLogin ✅│
│  │              │  │              │  │              │  │  OrderEvent │  │ Login      ✴│
│  │ initializer  │  │              │  │ default-     │  │             │  │ Logout     ✴│
│  │  Initializer │  │              │  │  order-      │  │ Fulltext    │  │ customer.s.│
│  │              │  │              │  │  process     │  │  SearchEvent│  │ AccountReg ✴│
│  │              │  │              │  │  OrderPlaced │  │             │  │ Password   ✴│
│  │              │  │              │  │              │  │ EmailProc.  │  │ Identifier ✴│
│  │              │  │              │  │              │  │  EmailSend  │  │             │
│  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘  └────────────┘
│        ↓ 立即          ↓ 等事务提交      ↓ 等事务提交      ↓ 视调用方而定      ↓ 等事务提交    │
│                                                                                      │
│  图例：                                                                              │
│    ✅ = 无条件前置发布（总能收到，如 AttemptedLoginEvent）                           │
│    ✴ = 成功路径末尾发布（冲突/校验失败时提前 return，不走到 publish）                │
│                                                                                      │
│  认证/账户事件的统一口径（见第 4.5.3 节）：                                           │
│    AttemptedLoginEvent: 方法开头无条件发布 → 无论成功失败都能收到                     │
│    其他事件: 冲突/校验失败时提前 return → 不走到 publish → 收不到                     │
│    关键: 事务总是 commit（不 throw）→ 已发布的事件不会被过滤，但大多根本没发布        │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 插件订阅注册机制

### 5.1 ofType() 订阅 — 在 OnApplicationBootstrap 中注册

这是最常用的模式。插件实现 `OnApplicationBootstrap`，在 `onApplicationBootstrap()` 中通过 `eventBus.ofType(XxxEvent).subscribe(handler)` 注册。

**核心内置订阅者**：

#### DefaultSearchPlugin（`core/src/plugin/default-search-plugin/default-search-plugin.ts`）

```typescript
async onApplicationBootstrap() {
    this.eventBus.ofType(ProductEvent).subscribe(event => {
        if (event.type === 'deleted') return this.searchIndexService.deleteProduct(event.ctx, event.product);
        else return this.searchIndexService.updateProduct(event.ctx, event.product);
    });
    this.eventBus.ofType(ProductVariantEvent).subscribe(event => { ... });
    this.eventBus.ofType(AssetEvent).subscribe(event => { ... });
    this.eventBus.ofType(ProductChannelEvent).subscribe(event => { ... });
    this.eventBus.ofType(ProductVariantChannelEvent).subscribe(event => { ... });
    this.eventBus.ofType(StockMovementEvent).subscribe(event => { ... });

    // 批量缓冲：CollectionModificationEvent 用 debounceTime + buffer 去重
    const collectionModification$ = this.eventBus.ofType(CollectionModificationEvent);
    const closingNotifier$ = collectionModification$.pipe(debounceTime(50));
    collectionModification$.pipe(
        buffer(closingNotifier$),
        filter(events => 0 < events.length),
        map(events => ({ ctx: events[0].ctx, ids: events.reduce(...) })),
    ).subscribe(events => { ... });

    // TaxRateModificationEvent 加 delay(1) 防止 SQLite TransactionNotStartedError
    this.eventBus.ofType(TaxRateModificationEvent).pipe(delay(1)).subscribe(event => { ... });
}
```

#### EmailPlugin（`email-plugin/src/plugin.ts`）

EmailPlugin 使用声明式 `EmailEventHandler` 配置，在 `onApplicationBootstrap` 中统一订阅：

```typescript
// plugin.ts:397-402
private async setupEventSubscribers() {
    for (const handler of EmailPlugin.options.handlers) {
        this.eventBus.ofType(handler.event).subscribe(event => {
            return this.handleEvent(handler, event);
        });
    }
}
```

每个 `EmailEventHandler` 声明监听一种事件类型（如 `OrderStateTransitionEvent`），在收到事件后：
1. 匹配过滤条件（如 `toState === 'PaymentSettled'`）
2. 生成邮件内容
3. 将邮件任务推入 `JobQueue`（异步发送）或同步发送（测试模式）
4. `EmailProcessor` 发送完成后发布 `EmailSendEvent`

**关键点**：EmailPlugin 订阅者收到事件时已在事务提交之后，但邮件发送通过 JobQueue 解耦，即使发送失败也不会影响原始事务。

#### CollectionService（`core/src/service/services/collection.service.ts`）

```typescript
onModuleInit() {
    const productEvents$ = this.eventBus.ofType(ProductEvent);
    const variantEvents$ = this.eventBus.ofType(ProductVariantEvent);
    // 合并流 → debounce → 重新计算受影响的集合
    merge(productEvents$, variantEvents$).pipe(
        filter(event => event.type !== 'deleted'),
        debounceTime(50),
        // ...
    ).subscribe(async event => { ... });
}
```

#### RoleService（`core/src/service/services/role.service.ts:76`）

```typescript
onModuleInit() {
    this.eventBus.ofType(RoleEvent).subscribe(event => {
        // 清理权限缓存
    });
}
```

#### FacetValueChecker（`core/src/service/helpers/facet-value-checker/facet-value-checker.ts`）

```typescript
onModuleInit() {
    this.eventBus?.ofType(ProductEvent).subscribe(() => { this.cachedFacetValues = []; });
    this.eventBus?.ofType(ProductVariantEvent).subscribe(() => { this.cachedFacetValues = []; });
}
```

#### CustomerGroupCondition（`core/src/config/promotion/conditions/customer-group-condition.ts`）

```typescript
init(injector: Injector) {
    injector.get(EventBus).ofType(CustomerGroupChangeEvent).subscribe(() => {
        // 清除 ExhaustedPromotions 缓存
    });
}
```

#### MultiChannelStockLocationStrategy（`core/src/config/catalog/multi-channel-stock-location-strategy.ts`）

```typescript
init(injector: Injector) {
    injector.get(EventBus).ofType(StockLocationEvent).subscribe(() => {
        // 清除库存位置缓存
    });
}
```

### 5.2 registerBlockingEventHandler() — 阻塞订阅

目前核心代码中阻塞处理器仅在测试中使用（`event-bus.spec.ts`），生产代码未使用。但该 API 对外公开，插件可以使用。

### 5.3 第三方插件订阅模式

```typescript
@VendurePlugin({ imports: [PluginCommonModule] })
export class MyPlugin implements OnApplicationBootstrap {
    constructor(private eventBus: EventBus) {}

    async onApplicationBootstrap() {
        this.eventBus.ofType(OrderStateTransitionEvent)
            .pipe(filter(event => event.toState === 'PaymentSettled'))
            .subscribe(event => { /* 处理逻辑 */ });
    }
}
```

**前提条件**：插件必须导入 `PluginCommonModule`，才能注入 EventBus。

---

## 6. 事务边界协作：核心机制

这是整个事件系统最精巧的部分。问题背景：

> Service 在事务内 `eventBus.publish(event)` 时，事务可能尚未提交。如果订阅者立即读取数据库，会看不到事务中的数据；如果订阅者拿到 `ctx` 中的 `EntityManager` 继续操作，事务完成后 QueryRunner 已释放，会抛 `QueryRunnerAlreadyReleasedError`。

### 6.1 事务传播链路

```
@Transaction() 装饰器
    │
    ▼
TransactionInterceptor.intercept()
    │
    ▼
TransactionWrapper.executeInTransaction(originalCtx, work, mode, isolationLevel, connection)
    │
    │  1. ctx = originalCtx.copy()     ← 浅拷贝 RequestContext
    │  2. queryRunner = connection.createQueryRunner()
    │  3. queryRunner.startTransaction()
    │  4. ctx[TRANSACTION_MANAGER_KEY] = queryRunner.manager  ← 将 EntityManager 挂到 ctx 上
    │
    │  5. result = await work(ctx)     ← 执行 resolver + service 逻辑
    │     └── service 内: eventBus.publish(new XxxEvent(ctx, ...))
    │         └── eventStream.next(event)          ← 异步订阅者收到事件
    │         └── executeBlockingEventHandlers()    ← 同步执行阻塞处理器
    │
    │  6. await queryRunner.commitTransaction()    ← 事务提交
    │  7. TypeORM 触发 afterTransactionCommit       ← TransactionSubscriber 收到信号
    │
    ▼
    finally:
        await queryRunner.release()
```

### 6.2 awaitActiveTransactions — 异步订阅者的事务等待

这是 `EventBus.ofType()` / `EventBus.filter()` 的关键管道操作：

```typescript
private async awaitActiveTransactions<T extends VendureEvent>(event: T): Promise<T | undefined> {
    // 1. 查找事件中的 RequestContext 属性
    const entry = Object.entries(event).find(([_, value]) => value instanceof RequestContext);
    if (!entry) return event;   // 无 ctx → 无事务 → 直接放行

    const [key, ctx] = entry;

    // 2. 从 ctx 取出事务 EntityManager
    const transactionManager: EntityManager | undefined = (ctx as any)[TRANSACTION_MANAGER_KEY];
    if (!transactionManager?.queryRunner) return event;  // 无事务 → 直接放行

    try {
        // 3. 等待事务提交
        await this.transactionSubscriber.awaitCommit(transactionManager.queryRunner);

        // 4. 复制 ctx 并移除事务管理器，防止订阅者使用已释放的 QueryRunner
        const newContext = ctx.copy();
        delete (newContext as any)[TRANSACTION_MANAGER_KEY];
        (event as any)[key] = newContext;

        return event;
    } catch (e: any) {
        if (e instanceof TransactionSubscriberError) {
            // 事务回滚而非提交 → 返回 undefined → 被 filter(notNullOrUndefined) 过滤掉
            return;
        }
        throw e;
    }
}
```

### 6.3 TransactionSubscriber — TypeORM 事务监听器

源文件：`core/src/connection/transaction-subscriber.ts`

```typescript
@Injectable()
export class TransactionSubscriber implements EntitySubscriberInterface {
    private subject$ = new Subject<TransactionSubscriberEvent>();

    constructor(@InjectConnection() private connection: Connection) {
        connection.subscribers.push(this);   // 注册为 TypeORM 订阅者
    }

    afterTransactionCommit(event: TransactionCommitEvent) {
        this.subject$.next({ type: 'commit', ...event });
    }

    afterTransactionRollback(event: TransactionRollbackEvent) {
        this.subject$.next({ type: 'rollback', ...event });
    }

    awaitCommit(queryRunner: QueryRunner): Promise<QueryRunner> {
        return this.awaitTransactionEvent(queryRunner, 'commit');
    }

    private awaitTransactionEvent(queryRunner, type?): Promise<QueryRunner> {
        if (queryRunner.isTransactionActive) {
            return lastValueFrom(this.subject$.pipe(
                filter(event => !event.queryRunner.isTransactionActive
                             && event.queryRunner === queryRunner),
                take(1),
                tap(event => {
                    if (type && event.type !== type) {
                        throw new TransactionSubscriberError(`Unexpected event type: ${event.type}. Expected ${type}.`);
                    }
                }),
                map(event => event.queryRunner),
                delay(0),   // 必要的：给 TypeORM v0.2.41+ 一个事件循环让 queryRunner 完全释放
            ));
        } else {
            return Promise.resolve(queryRunner);  // 事务已结束 → 直接返回
        }
    }
}
```

### 6.4 事务回滚时的事件处理

```
事务回滚 → TransactionSubscriber.subject$ emit { type: 'rollback' }
         → awaitCommit 抛出 TransactionSubscriberError
         → awaitActiveTransactions catch 返回 undefined
         → ofType/filter 管道中 filter(notNullOrUndefined) 过滤掉
         → 订阅者永远不会收到回滚事务中的事件
```

### 6.5 Blocking Handler 与事务的关系

阻塞处理器在 `publish()` 中同步执行，**不经过 `awaitActiveTransactions`**。这意味着：

1. 阻塞处理器在事务**提交之前**执行
2. 阻塞处理器通过 `event.ctx` 可以拿到活跃的 `EntityManager`
3. 阻塞处理器中的 DB 操作**与发布者处于同一事务**
4. 阻塞处理器抛异常 → 事务回滚 → ofType/filter 订阅者不会收到该事件

### 6.6 RequestContext.copy() 的作用

```typescript
copy(): RequestContext {
    return Object.assign(Object.create(Object.getPrototypeOf(this)), this);
}
```

浅拷贝。`TransactionWrapper.executeInTransaction()` 在事务开始前 `copy()` 原始 ctx，保证：
- 原始 ctx 不被事务 EntityManager 污染
- `awaitActiveTransactions` 中 `ctx.copy()` 后删除 `TRANSACTION_MANAGER_KEY`，保证订阅者拿到干净的 ctx

---

## 7. 时序图：典型事务内事件发布与消费

```
Resolver              TransactionWrapper      Service          EventBus         TransactionSubscriber     ofType订阅者
  │                        │                    │                 │                     │                      │
  │──@Transaction()───────>│                    │                 │                     │                      │
  │                        │──copy(ctx)────────>│                 │                     │                      │
  │                        │──startTransaction()│                 │                     │                      │
  │                        │──work(txCtx)──────>│                 │                     │                      │
  │                        │                    │──publish(event)─>│                     │                      │
  │                        │                    │                 │──next(eventStream)──>│────(异步等待)────────>│
  │                        │                    │                 │──blockingHandlers───>│                      │
  │                        │                    │                 │                     │   awaitCommit()       │
  │                        │                    │<───await────────│                     │<─────────────────────│
  │                        │<───result──────────│                 │                     │                      │
  │                        │──commit()──────────│                 │                     │                      │
  │                        │                    │                 │                     │──afterCommit()──>     │
  │                        │                    │                 │                     │              awaitCommit
  │                        │                    │                 │                     │              resolve   │
  │                        │                    │                 │<──awaitCommit完成────│                      │
  │                        │                    │                 │──ctx.copy()+del TMK─>│──────event───────────>│
  │                        │                    │                 │                     │                      │──handler()
  │<───response────────────│                    │                 │                     │                      │
```

---

## 8. 设计要点总结

### 8.1 为什么 ofType 用 `e.constructor === type` 而非 `instanceof`？

精确匹配防止子类事件意外触发父类订阅者。如果需要 `instanceof` 语义，使用 `eventBus.filter()`。

### 8.2 为什么阻塞处理器不等待事务？

因为阻塞处理器的设计目标就是**在同一事务中执行**，让处理器的 DB 写入和发布者原子提交。如果等事务提交再执行，就无法保证原子性。

### 8.3 事件在事务内发布但订阅者等事务外消费 — 这意味着什么？

- ofType/filter 订阅者收到的事件中的 `ctx` 已被**清理掉事务管理器**
- 订阅者的 DB 操作会在**新事务**中执行
- 如果订阅者需要原子性操作，应该使用 `registerBlockingEventHandler` 或 `JobQueueService`

### 8.4 `delay(0)` 的必要性

`TransactionSubscriber.awaitTransactionEvent` 中的 `delay(0)` 是 TypeORM v0.2.41+ 的兼容处理。没有这个延迟，`afterTransactionCommit` 回调中 `queryRunner.isTransactionActive` 可能仍为 `true`，导致事件订阅者拿到一个仍然活跃的 QueryRunner。

### 8.5 事件发布的 `await`

几乎所有 `eventBus.publish()` 调用都有 `await`。这确保阻塞处理器在 `publish` 返回前执行完毕。对于 ofType/filter 订阅者，`publish` 不等待它们（它们在 RxJS 管道中异步执行）。

---

### 8.6 "Return ErrorResult" vs "Throw Exception" 的事务语义差异

这是 Vendure 最核心的设计模式之一，直接决定事件是否会被订阅者收到。

**详细分支条件分析见第 4.5.2 节**。本节从设计模式角度总结。

#### 8.6.1 两种错误处理模式

| 模式 | 事务行为 | 事件是否被过滤 | 适用场景 |
|------|---------|-------------|---------|
| `return new InvalidCredentialsError()` | **提交** | ❌ 不被过滤（订阅者收到） | 验证失败、业务规则不满足等预期内的错误 |
| `throw new InternalServerError()` | **回滚** | ✅ 被过滤（订阅者收不到） | 数据库错误、空指针等意外错误 |

#### 8.6.2 代码对比

**模式 1：返回 ErrorResult（事务提交）**

```typescript
// auth.service.ts:68-69
if (typeof authenticateResult === 'string') {
    return new InvalidCredentialsError({ authenticationError: authenticateResult });
}
```

TransactionWrapper 中的处理：
```typescript
try {
    const result = await lastValueFrom(from(work(ctx)).pipe(...));  // result 是 ErrorResult 对象
    if (queryRunner.isTransactionActive) {
        await queryRunner.commitTransaction();  // ✅ 不抛异常就 commit
    }
    return result;
} catch (error) {
    // 不会走到这里
}
```

**模式 2：抛出异常（事务回滚）**

```typescript
// customer.service.ts:496
if (!customer) {
    throw new InternalServerError('error.cannot-locate-customer-for-user');
}
```

TransactionWrapper 中的处理：
```typescript
try {
    const result = await lastValueFrom(from(work(ctx)).pipe(...));  // 抛异常跳出
} catch (error) {
    if (queryRunner.isTransactionActive) {
        await queryRunner.rollbackTransaction();  // ❌ 回滚
    }
    throw error;
}
```

#### 8.6.3 对事件系统的影响

由于 `awaitActiveTransactions()` 会在事务回滚时返回 `undefined`，而 `ofType()` 管道中有 `filter(notNullOrUndefined)`：

```typescript
// event-bus.ts ofType() 管道
mergeMap(event => this.awaitActiveTransactions(event)),
filter(notNullOrUndefined),  // 事务回滚时 event 变为 undefined，被过滤
```

因此：
- **返回 ErrorResult** → 事务 commit → `awaitCommit()` resolve → 事件传递给订阅者
- **抛出异常** → 事务 rollback → `awaitCommit()` reject → 返回 `undefined` → 被过滤掉

#### 8.6.4 AttemptedLoginEvent 的设计意图

`AttemptedLoginEvent` 无论认证成功还是失败都会被订阅者收到，这是有意设计的：

1. **发布时机**：在认证流程的最开头发布（`auth.service.ts:57`），此时还未进行密码验证
2. **失败处理**：密码错误时 `return InvalidCredentialsError`（不抛异常）→ 事务 commit
3. **用途**：登录审计、登录失败频率限制、暴力破解检测等安全监控

如果认证失败时事件被过滤掉，安全监控插件就无法统计失败的登录尝试，这会是一个严重的安全缺陷。

#### 8.6.5 各事件的实际设计选择（与第 4.5.3 节对照）

**详细分支分析见第 4.5.3 节统一口径表**。以下从设计意图角度总结：

| 事件 | 实际行为（与代码一致） | 设计意图 |
|------|----------------------|---------|
| `AttemptedLoginEvent` | 方法开头无条件发布；失败时 return ErrorResult 但**已发布** → 订阅者总能收到 | 安全审计必须记录所有登录尝试，无论成功失败 |
| `AccountRegistrationEvent` | 邮箱冲突时**提前 return** → 不走到 publish → 收不到；仅用户未验证且无冲突时才发布 | 冲突时不发邮件，避免骚扰；仅真正的新用户注册才发验证邮件 |
| `IdentifierChangeRequestEvent` | 新邮箱冲突时**提前 return** → 不走到 publish → 收不到；仅无冲突且需要验证时才发布 | 冲突时不需要发验证邮件 |
| `PasswordResetEvent` | 用户不存在时**不进入 if 块** → 静默跳过；仅用户存在且有本地认证时才发布 | 不泄露"该邮箱是否已注册"信息（防止账户枚举攻击） |
| `AccountVerifiedEvent` | 验证失败时 return ErrorResult，**不走到 publish** → 收不到；仅验证成功才发布 | 只有验证成功才发欢迎邮件 |

**关键修正**：之前的"邮箱冲突时也需要发通知邮件"与代码不符。实际上所有冲突/校验失败场景都是**提前 return，不走到 publish**，订阅者收不到。只有 `AttemptedLoginEvent` 是例外——它在方法开头就已经发布了。

**两类"收不到"的区别**：
1. **提前 return 导致收不到**（占 90%）：publish() 调用根本没执行到，事件从未进入 EventBus
2. **事务回滚导致收不到**（占 10%，边缘情况）：publish() 执行了，但事务回滚时被 filter 过滤掉

这个模式贯穿整个 Vendure 代码库：**审计事件（AttemptedLoginEvent）在方法开头无条件发布，业务失败用 return ErrorResult 保证已发布的审计事件能被收到；其他事件在成功路径末尾发布，失败时提前 return 保证不发布；意外的系统错误用 throw Exception，回滚事务并丢弃事件。**
