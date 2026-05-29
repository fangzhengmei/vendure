# Vendure 测试工具集与 Mock 数据体系详解

## 全局架构总览

Vendure 的 e2e 测试体系由三个核心层次衔接而成：

```
┌─────────────────────────────────────────────────────────┐
│  场景准备层 (Test Environment)                           │
│  createTestEnvironment() → TestServer + GraphQL Clients │
├─────────────────────────────────────────────────────────┤
│  夹具加载层 (Fixture & Data Population)                  │
│  Initializer → populateForTesting → CSV/InitialData     │
├─────────────────────────────────────────────────────────┤
│  依赖替换层 (Mock / Stub / Test Strategy)                │
│  TestConfig 替换项 + e2e/fixtures/ 下可插拔实现          │
└─────────────────────────────────────────────────────────┘
```

核心包路径：`packages/testing/src/`
e2e 夹具路径：`packages/core/e2e/fixtures/`
共享配置路径：`e2e-common/`

---

## 一、场景准备层：Test Environment 的搭建

### 1.1 入口函数 `createTestEnvironment()`

**文件**：`packages/testing/src/create-test-environment.ts`

这是所有 e2e 测试的起点。它接收一份 `VendureConfig`，创建出三个对象：

```ts
const { server, adminClient, shopClient } = createTestEnvironment(testConfig());
```

| 返回值 | 类型 | 职责 |
|--------|------|------|
| `server` | `TestServer` | 真实的 Vendure 服务端实例，负责 bootstrap 和数据填充 |
| `adminClient` | `SimpleGraphQLClient` | 指向 Admin API (`/admin-api`) 的轻量 GraphQL 客户端 |
| `shopClient` | `SimpleGraphQLClient` | 指向 Shop API (`/shop-api`) 的轻量 GraphQL 客户端 |

内部实现非常简单——从 config 中取出 `port`、`adminApiPath`、`shopApiPath` 来构建两个客户端的 URL：

```ts
const adminClient = new SimpleGraphQLClient(config, `http://localhost:${port}/${adminApiPath}`);
const shopClient  = new SimpleGraphQLClient(config, `http://localhost:${port}/${shopApiPath}`);
```

### 1.2 `TestServer` —— 服务端生命周期管理

**文件**：`packages/testing/src/test-server.ts`

`TestServer` 是整个测试体系的心脏，它管理"启动 → 填充数据 → 可用 → 销毁"的完整生命周期：

```
init()  →  initializer.init()  →  initializer.populate()  →  initializer.destroy()  →  bootstrap()
                                                     ↓
                                            populateForTesting()
                                                     ↓
                                    populateInitialData / populateProducts / populateCollections / populateCustomers
```

#### `init()` 方法的四步流程

1. **获取 DB 初始化器**：根据 `dbConnectionOptions.type`（sqljs/postgres/mysql）查找已注册的 `TestDbInitializer`
2. **`initializer.init()`**：为当前测试文件创建专属数据库（sqljs 生成 `.sqlite` 文件；postgres/mysql 创建 `e2e_xxx` 数据库）
3. **`initializer.populate()`**：执行数据填充函数（只在数据库不存在时填充，sqljs 有缓存机制）
4. **`initializer.destroy()`**：关闭初始化阶段的临时数据库连接
5. **`bootstrap()`**：用填充好的数据库正式启动 NestJS 应用

关键细节：`init()` 内部通过 `getCallerFilename()` 捕获调用者的文件路径，以此作为数据库命名的依据，确保每个测试文件有独立的数据库实例。

#### `bootstrapForTesting()` 与正式 bootstrap 的区别

这个私有方法跳过了正常 bootstrap 中的一些重型操作，直接使用 `preBootstrapConfig()` 处理配置，然后 `NestFactory.create()` 创建 NestJS 应用并监听端口。

### 1.3 `SimpleGraphQLClient` —— 测试用的 GraphQL 客户端

**文件**：`packages/testing/src/simple-graphql-client.ts`

它不是用 Apollo 而是用 `node-fetch` 手写的极简 GraphQL 客户端，专门为测试场景设计：

| 能力 | 方法 | 说明 |
|------|------|------|
| 查询/变更 | `query()` | 执行 GraphQL query/mutation |
| 身份切换 | `asSuperAdmin()` / `asUserWithCredentials()` / `asAnonymousUser()` | 自动管理 auth token |
| 渠道切换 | `setChannelToken()` | 设置 channel token header |
| 原始 HTTP | `fetch()` | 携带 auth headers 的裸 HTTP 请求，用于测试 REST 端点 |
| 文件上传 | `fileUploadMutation()` | 支持 graphql-multipart-request-spec |

身份管理的关键：每次登录后，服务端返回的 auth token 会通过 response header 自动被捕获并存入 `this.authToken`，后续请求自动带上 `Authorization: Bearer xxx`。

---

## 二、夹具加载层：数据库初始化与数据填充

### 2.1 数据库初始化器 —— `TestDbInitializer` 接口

**文件**：`packages/testing/src/initializers/test-db-initializer.ts`

```ts
interface TestDbInitializer<T> {
    init(testFileName, connectionOptions): Promise<T>;   // 创建数据库
    populate(populateFn): Promise<void>;                  // 执行填充
    destroy(): void | Promise<void>;                      // 清理
}
```

三种内置实现：

| 初始化器 | 数据库 | `init()` 行为 | `populate()` 缓存策略 |
|----------|--------|---------------|----------------------|
| `SqljsInitializer` | sqljs (SQLite) | 计算出 `.sqlite` 文件路径并写入 `connectionOptions.location` | **有缓存**：文件已存在则跳过 populate |
| `PostgresInitializer` | PostgreSQL | 连接 postgres 库，DROP + CREATE 新数据库 | 无缓存，每次都 populate |
| `MysqlInitializer` | MySQL/MariaDB | 同上 | 无缓存 |

#### SqljsInitializer 的缓存机制（性能关键）

```ts
async populate(populateFn) {
    if (!fs.existsSync(this.dbFilePath)) {    // 关键：文件已存在则跳过
        // 临时开启 autoSave 和 synchronize
        connectionOptions.autoSave = true;
        connectionOptions.synchronize = true;
        await populateFn();                   // 执行耗时的数据填充
        connectionOptions.autoSave = false;   // 填充完关闭自动保存
        connectionOptions.synchronize = false;
    }
}
```

这就是为什么 e2e 测试"第一次慢、后续快"——`__data__/` 目录下的 `.sqlite` 文件充当了缓存。修改 schema 后需要删除 `__data__/` 目录来强制重建。

#### 注册机制

**文件**：`packages/testing/src/initializers/initializers.ts`

```ts
// 在 e2e-common/test-config.ts 中注册
registerInitializer('sqljs', new SqljsInitializer(path.join(packageDir, '__data__')));
registerInitializer('postgres', new PostgresInitializer());
registerInitializer('mysql', new MysqlInitializer());
```

`getInitializerFor(type)` 根据数据库类型查找已注册的初始化器，未注册则抛错。

### 2.2 数据填充流程 —— `populateForTesting()`

**文件**：`packages/testing/src/data-population/populate-for-testing.ts`

这是填充逻辑的编排者，按固定顺序执行四步填充：

```
populateForTesting()
  │
  ├─ 1. bootstrapFn(config)          ← 先启动一次临时 NestJS 应用
  ├─ 2. awaitOutstandingJobs(app)    ← 等待 bootstrap 期间产生的异步任务完成
  │
  ├─ 3. populateInitialData(app, initialData)   ← 填充基础数据（国家、税率、配送方式等）
  ├─ 4. populateProducts(app, productsCsvPath)  ← 从 CSV 导入产品数据
  ├─ 5. populateCollections(app, initialData)   ← 创建集合
  ├─ 6. populateCustomers(app, customerCount)   ← 生成随机客户数据
  │
  └─ 7. 返回 app，之后在 TestServer.init() 中关闭
```

关键细节：
- 填充期间临时关闭 `requireVerification`，填充完恢复原值
- `awaitOutstandingJobs()` 解决了一个 Node v20 上 sql.js 测试的竞态问题——bootstrap 中插件可能触发搜索索引更新任务，必须在填充前等这些任务完成
- `populateProducts()` 只有在提供了 `productsCsvPath` 时才执行

### 2.3 `initialData` —— 非产品类的种子数据

**文件**：`e2e-common/e2e-initial-data.ts`

```ts
export const initialData: InitialData = {
    defaultLanguage: LanguageCode.en,
    defaultZone: 'Europe',
    taxRates: [
        { name: 'Standard Tax', percentage: 20 },
        { name: 'Reduced Tax', percentage: 10 },
        { name: 'Zero Tax', percentage: 0 },
    ],
    shippingMethods: [
        { name: 'Standard Shipping', price: 500 },
        { name: 'Express Shipping', price: 1000 },
    ],
    paymentMethods: [],
    countries: [...],
    collections: [...],
};
```

这是 `@vendure/core` 的 `populateInitialData()` 函数所需的数据结构，负责创建 Channel、Zone、Country、TaxCategory、TaxRate、ShippingMethod、PaymentMethod 等基础实体。

### 2.4 产品数据 —— CSV 导入

产品数据通过 CSV 文件批量导入，不同测试场景使用不同的 CSV：

| CSV 文件 | 用途 |
|----------|------|
| `e2e-products-minimal.csv` | 最小集，大多数测试用这个 |
| `e2e-products-full.csv` | 完整产品集 |
| `e2e-products-promotions.csv` | 促销测试专用 |
| `e2e-products-stock-control.csv` | 库存控制测试 |
| `e2e-products-money-handling.csv` | 金额策略测试 |
| `e2e-products-order-taxes.csv` | 订单税收测试 |

### 2.5 客户数据 —— `MockDataService` + `populateCustomers()`

**文件**：`packages/testing/src/data-population/mock-data.service.ts`

`MockDataService` 使用 `faker` 库生成确定性的随机客户数据（`faker.seed(1)` 保证可重现）。`populateCustomers()` 通过 NestJS 的 `CustomerService` 直接创建客户和地址，而非通过 GraphQL API，这样更快也更可靠。

`getSuperadminContext()` (`packages/testing/src/utils/get-superadmin-context.ts`) 是支撑函数，创建一个以 superadmin 身份运行的 `RequestContext`，用于绕过权限限制来创建数据。

---

## 三、依赖替换层：Mock / Stub / Test Strategy

Vendure 的依赖替换不像传统单元测试那样用 jest.mock() 替换模块，而是利用框架自身的**策略模式**和**可插拔配置**来实现——在创建测试环境时传入测试专用的实现即可。

### 3.1 `testConfig` —— 测试专用配置

**文件**：`packages/testing/src/config/test-config.ts`

`testConfig` 基于 `defaultConfig` 通过 `mergeConfig` 生成，做了以下替换：

| 替换项 | 原始值 | 测试值 | 作用 |
|--------|--------|--------|------|
| 数据库 | (各生产数据库) | `sqljs` + 内存 `Uint8Array` | 无需外部数据库，数据在内存中 |
| EntityIdStrategy | 自增 ID | `TestingEntityIdStrategy` | ID 编码为 `T_1`、`T_2`，用于验证 ID 编解码正确性 |
| AssetStorageStrategy | 文件系统存储 | `TestingAssetStorageStrategy` | 不写磁盘，返回假路径 |
| AssetPreviewStrategy | 图片处理 | `TestingAssetPreviewStrategy` | 返回硬编码的 48×48 小图 Buffer |
| Logger | DefaultLogger | `NoopLogger`（除非 `LOG=true`） | 默认静默，CI 输出干净 |
| auth.tokenMethod | (多种) | `'bearer'` | 统一用 bearer token |
| paymentMethodHandlers | (生产支付) | `[]` 空 | 不预装任何支付处理器 |

### 3.2 测试专用的策略实现（Stub）

#### `TestingEntityIdStrategy`

```ts
encodeId(primaryKey) => 'T_' + primaryKey   // 1 → 'T_1'
decodeId(id)         => parseInt(id.replace('T_', ''))  // 'T_1' → 1
```

这不是"假数据"，而是**验证工具**——如果某个 ID 字段没有经过正确的编解码，测试就会因为找不到 `T_` 前缀而失败，从而暴露 bug。

#### `TestingAssetStorageStrategy`

所有方法都是 no-op 或返回固定值：
- `writeFileFromBuffer()` → 返回 `test-assets/{fileName}`
- `readFileToBuffer()` → 返回固定图片 Buffer
- `toAbsoluteUrl()` → 加 `test-url/` 前缀
- `fileExists()` → 始终返回 `false`

#### `TestingAssetPreviewStrategy`

`generatePreviewImage()` 直接返回 `getTestImageBuffer()`——一个硬编码的 48×48 PNG 图片的 base64 解码 Buffer。完全不做图片处理。

### 3.3 e2e 夹具中的可插拔实现

`packages/core/e2e/fixtures/` 下存放的是**业务逻辑层面**的测试替代品，它们不是 noop stub，而是具有特定行为的策略实现，让测试可以验证各种业务场景：

#### 支付方法夹具 (`test-payment-methods.ts`)

| 名称 | 行为 | 测试场景 |
|------|------|----------|
| `testSuccessfulPaymentMethod` | createPayment → Settled，settlePayment → success | 正常支付流程 |
| `twoStagePaymentMethod` | createPayment → Authorized，settlePayment → success | 两阶段支付（授权→结算） |
| `partialPaymentMethod` | 支付部分金额 | 多次支付同一订单 |
| `singleStageRefundablePaymentMethod` | 支持退款 | 退款流程 |
| `singleStageRefundFailingPaymentMethod` | 首次退款失败，第二次成功 | 退款重试 |
| `failsToSettlePaymentMethod` | settlePayment 始终失败 | 结算失败处理 |
| `failsToCancelPaymentMethod` | cancelPayment 始终失败 | 取消失败处理 |
| `testFailingPaymentMethod` | createPayment → Declined | 支付被拒 |
| `testErrorPaymentMethod` | createPayment → Error | 支付异常 |

注意 `onTransitionSpy = vi.fn()` —— 这是 vitest 的 mock 函数，注入到 `onStateTransitionStart` 回调中，让测试可以断言状态转换是否被调用。

#### 认证策略夹具 (`test-authentication-strategies.ts`)

| 名称 | 行为 |
|------|------|
| `TestAuthenticationStrategy` | token = `'valid-auth-token'` 则通过，否则拒绝；`'expired-token'` 返回错误消息 |
| `TestSSOStrategyAdmin` | 自动创建 Administrator 用户 |
| `TestSSOStrategyShop` | 自动创建 Customer 用户 |

#### 配送策略夹具 (`test-shipping-eligibility-checkers.ts`)

| 名称 | 行为 |
|------|------|
| `countryCodeShippingEligibilityChecker` | 按国家代码判断是否可用 |
| `hydratingShippingEligibilityChecker` | hydrate Order 的关联数据后返回 true（用于测试 #2548 竞态问题） |

#### 其他策略

- `TestMoneyStrategy`：覆盖 `round()` 为 `Math.round(value * quantity)`，测试金额精度
- `TestOrderItemPriceCalculationStrategy`：加 $5 礼品包装费，3 件以上半价——测试自定义计价逻辑

### 3.4 测试专用的 JobQueue 和 JobBuffer

| 类 | 位置 | 作用 |
|----|------|------|
| `TestingJobQueueStrategy` | `core/src/job-queue/testing-job-queue-strategy.ts` | 继承 `InMemoryJobQueueStrategy`，增加 `prePopulate()` 方法，可在测试开始前预注入任务 |
| `TestingJobBufferStorageStrategy` | `core/src/job-queue/job-buffer/testing-job-buffer-storage-strategy.ts` | 继承 `InMemoryJobBufferStorageStrategy`，暴露 `getBufferedJobs()` 用于断言缓冲区中的任务 |

这两个不是通过 testConfig 替换的，而是在测试代码中手动注入到 NestJS 容器或 plugin 配置中。

### 3.5 `TestingLogger` —— 可断言的日志器

**文件**：`packages/testing/src/testing-logger.ts`

```ts
const testingLogger = new TestingLogger(() => jest.fn());
// 使用：testingLogger.errorSpy.mockClear();
// 断言：expect(testingLogger.errorSpy).toHaveBeenCalled();
```

每个日志级别（debug/error/info/verbose/warn）对应一个 spy，可以用来验证"是否产生了错误日志"这类断言。

### 3.6 `ErrorResultGuard` —— GraphQL 联合类型的断言工具

**文件**：`packages/testing/src/error-result-guard.ts`

Vendure 的 GraphQL mutation 返回值通常是 `SuccessType | ErrorResult` 的联合类型。`ErrorResultGuard` 提供类型窄化 + 断言：

```ts
const orderGuard = createErrorResultGuard<Order>(order => !!order.lines);

// 断言成功，TypeScript 知道结果是 Order 类型
orderGuard.assertSuccess(result);
expect(result.code).toBeDefined();

// 断言失败，TypeScript 知道结果是 ErrorResult 类型
orderGuard.assertErrorResult(result);
expect(result.errorCode).toBe(ErrorCode.NegativeQuantityError);
```

---

## 四、三层如何衔接：一次完整 e2e 测试的流程

以 `order.e2e-spec.ts` 为例：

```ts
// 第 1 步：组合配置——共享的 testConfig + 测试专用的策略替换
const { server, adminClient, shopClient } = createTestEnvironment(
    mergeConfig(testConfig(), {
        paymentOptions: {
            paymentMethodHandlers: [
                twoStagePaymentMethod,              // ← 依赖替换：用测试支付方法
                failsToSettlePaymentMethod,
            ],
        },
    }),
);

beforeAll(async () => {
    // 第 2 步：初始化服务器 + 填充数据
    await server.init({
        initialData,                                                    // ← 夹具：基础种子数据
        productsCsvPath: path.join(__dirname, 'fixtures/e2e-products-minimal.csv'),  // ← 夹具：产品 CSV
        customerCount: 2,                                               // ← 夹具：2 个随机客户
    });
    // 第 3 步：客户端身份认证
    await adminClient.asSuperAdmin();
});

afterAll(async () => {
    await server.destroy();
});
```

执行时序：

```
1. createTestEnvironment(config)
   ├── new TestServer(config)              ← 存储配置，此时不启动
   ├── new SimpleGraphQLClient(AdminURL)   ← 存储地址，此时不连接
   └── new SimpleGraphQLClient(ShopURL)    ← 存储地址，此时不连接

2. server.init(options)
   ├── getCallerFilename() → 拿到 "order.e2e-spec.ts"
   ├── getInitializerFor('sqljs') → SqljsInitializer
   ├── SqljsInitializer.init("order.e2e-spec.ts", connOpts)
   │   └── 计算 .sqlite 文件路径: __data__/order.e2e-spec.ts.sqlite
   ├── SqljsInitializer.populate(populateFn)
   │   ├── 文件不存在? → 执行 populateFn:
   │   │   ├── bootstrapForTesting(config)   ← 临时启动 NestJS
   │   │   ├── awaitOutstandingJobs()
   │   │   ├── populateInitialData(app, initialData)  ← 国家、税率、配送
   │   │   ├── importProductsFromCsv(app, csvPath)    ← 产品数据
   │   │   ├── populateCollections(app, initialData)  ← 集合
   │   │   ├── populateCustomers(app, 2)              ← 2个客户
   │   │   └── app.close()
   │   └── 文件已存在? → 跳过（缓存命中）
   ├── SqljsInitializer.destroy()
   └── bootstrapForTesting(config)          ← 正式启动服务，连接填充好的数据库

3. adminClient.asSuperAdmin()
   └── login mutation → 拿到 token → 存入 headers

4. 测试用例中:
   ├── shopClient.query(ADD_ITEM_TO_ORDER, {...})   ← 模拟购物
   ├── adminClient.query(UPDATE_ORDER, {...})       ← 管理员操作
   └── orderGuard.assertSuccess(result)             ← 类型安全断言
```

---

## 五、共享配置 `e2e-common/`

**文件**：`e2e-common/test-config.ts`

这个文件承担了所有 e2e 测试共享的初始化工作：

1. **注册数据库初始化器**：在模块加载时就执行 `registerInitializer()`，确保 `getInitializerFor()` 能找到对应的初始化器
2. **端口分配**：通过 `getBasePort() + index` 为每个测试文件分配唯一端口，避免并行测试冲突。core 包使用 3010-3199
3. **数据库选择**：通过 `DB` 环境变量切换数据库类型（默认 sqljs）

---

## 六、关键设计洞察

### 6.1 "策略替换"而非"模块 Mock"

Vendure 不依赖 jest/vitest 的模块 mock 能力来替换依赖，而是利用框架自身的策略模式：所有可替换的依赖（支付、认证、配送、资产存储、ID 策略……）都是通过配置注入的接口实现。测试时只需在 `createTestEnvironment(mergeConfig(testConfig(), {...}))` 中传入不同的实现即可。

**优点**：
- 替换的是真实的业务逻辑实现，不是 mock 桩，测试更接近真实场景
- 不依赖测试框架的 mock 机制，vitest/jest 切换无感
- 每个测试文件可以按需组合不同的策略，互不干扰

### 6.2 数据库文件缓存策略

sqljs 的 `.sqlite` 文件缓存是一个精妙的设计：
- 每个测试文件一个独立的 `.sqlite` 文件
- 只在首次运行时执行耗时的 populate 流程
- 后续运行直接加载已有文件，速度提升 10 倍以上
- schema 变更后只需删除 `__data__/` 目录即可强制重建

### 6.3 两阶段 Bootstrap

`TestServer.init()` 对 `bootstrapForTesting()` 进行了**两次独立调用**，而非"重启"同一实例：

1. **第一次调用**（populate 阶段）：由 `populateInitialData()` → `populateForTesting()` → `bootstrapFn(config)` 触发。此实例仅用于数据填充，填充完成后调用 `app.close()` 销毁。这个 NestJS 实例监听的端口与第二次相同（因为用的是同一份 `config`），但由于 `app.close()` 在 `populateInitialData()` 返回前就已执行，端口会先释放再被第二次调用占用。

2. **第二次调用**（正式阶段）：由 `this.bootstrap()` → `bootstrapForTesting(this.vendureConfig)` 触发。此实例连接同一数据库文件，供测试用例使用。

两次调用之间不存在"重启"关系——它们是**两次完全独立的 `NestFactory.create()` + `app.listen()`**，中间有一次 `app.close()` 确保端口释放。`bootstrap()` 方法也可独立使用，适用于需要重新启动服务但不重新填充数据的场景。

### 6.4 确定性随机

`MockDataService` 通过 `faker.seed(1)` 确保每次生成的客户数据完全相同。这意味着不同机器、不同时间运行的测试面对的是同一组客户数据，消除了随机性带来的测试不稳定。

---

## 七、异常边界处理：初始化报错时的数据库清理与配置回滚

### 7.1 `TestServer.init()` 的异常捕获——"裸" try-catch

**文件**：`packages/testing/src/test-server.ts:30-43`

```ts
async init(options: TestServerOptions): Promise<void> {
    const { type } = this.vendureConfig.dbConnectionOptions;
    const { dbConnectionOptions } = this.vendureConfig;
    const testFilename = this.getCallerFilename(1);
    const initializer = getInitializerFor(type);
    try {
        await initializer.init(testFilename, dbConnectionOptions);
        const populateFn = () => this.populateInitialData(this.vendureConfig, options);
        await initializer.populate(populateFn);
        await initializer.destroy();
    } catch (e: any) {
        throw e;   // ← 仅重新抛出，没有 finally 块做清理
    }
    await this.bootstrap();
}
```

**关键发现**：这个 try-catch 实质上是透传异常，**没有 `finally` 块**。这意味着：

| 异常发生位置 | initializer.destroy() 是否执行 | 数据库清理 | 后续 bootstrap() 是否执行 |
|-------------|-------------------------------|-----------|-------------------------|
| `initializer.init()` 抛异常 | ❌ 不执行 | 不清理 | ❌ 不执行 |
| `initializer.populate()` 抛异常 | ❌ 不执行 | 不清理 | ❌ 不执行 |
| `initializer.destroy()` 抛异常 | （自身失败） | 部分清理 | ❌ 不执行 |
| 全部成功 | ✅ 正常执行 | ✅ 正常清理 | ✅ 执行 |

**潜在问题**：
- 如果 `populate()` 失败，`initializer.destroy()` 不会被调用，对于 postgres/mysql 来说意味着**初始化阶段创建的数据库连接不会被关闭**，可能造成连接泄漏
- 对于 sqljs，`destroy()` 本身就是 no-op（返回 `undefined`），所以即使不调用也没有资源泄漏，但**可能留下不完整的 `.sqlite` 文件**

### 7.2 `populateForTesting()` 的配置回滚——缺少 finally 保护

**文件**：`packages/testing/src/data-population/populate-for-testing.ts:15-37`

```ts
const originalRequireVerification = config.authOptions.requireVerification;
config.authOptions.requireVerification = false;   // ← 临时修改

const app = await bootstrapFn(config);
await awaitOutstandingJobs(app);

await populateInitialData(app, options.initialData);
await populateProducts(app, options.productsCsvPath, logging);
await populateCollections(app, options.initialData);
await populateCustomers(app, options.customerCount ?? 10, logFn);

config.authOptions.requireVerification = originalRequireVerification;  // ← 恢复
return app;
```

**关键发现**：`requireVerification` 的恢复**不在 finally 块中**。如果 `populateInitialData()`、`populateProducts()`、`populateCollections()` 或 `populateCustomers()` 中任何一个抛出异常：

1. `requireVerification` **不会恢复**为原值
2. `dbConnectionOptions.logging = false` 也不会恢复
3. 由于 `config` 是引用传递，如果测试代码手动捕获 `init()` 异常后单独调用 `server.bootstrap()`，被污染的值会影响后续服务
4. 但在正常流程中，异常冒泡到 `TestServer.init()` 的 catch 后 `this.bootstrap()` 不会执行，所以**被污染的 config 不会自动到达第二次 bootstrap**
5. `app`（临时启动的 NestJS 实例）不会被关闭，可能导致端口未释放——这是一个独立的资源泄漏问题

### 7.3 `populateInitialData()` 的 app 关闭——另一个无保护的 close

**文件**：`packages/testing/src/test-server.ts:92-101`

```ts
private async populateInitialData(testingConfig, options): Promise<void> {
    const app = await populateForTesting(testingConfig, this.bootstrapForTesting, {
        logging: false,
        ...options,
    });
    await app.close();  // ← 如果 populateForTesting 内部抛异常，close() 不会执行
}
```

如果 `populateForTesting()` 在 `bootstrapFn()` 成功后的某个步骤抛异常，临时启动的 NestJS 应用不会被关闭，端口会一直占用。

### 7.4 `bootstrapForTesting()` 的异常处理

**文件**：`packages/testing/src/test-server.ts:106-138`

```ts
try {
    DefaultLogger.hideNestBoostrapLogs();
    const app = await NestFactory.create(appModule.AppModule, {
        abortOnError: false,   // ← NestJS 不因模块初始化错误而中止
    });
    // ... listen, start JobQueue ...
    return app;
} catch (e: any) {
    console.log(e);   // ← 仅打印，不做清理
    throw e;
}
```

`abortOnError: false` 意味着即使某些 Provider 初始化失败，NestJS 仍会尝试创建应用。如果应用创建成功但 `app.listen()` 抛异常（如端口被占用），则**已经创建的 NestJS 实例不会调用 `app.close()`**。

### 7.5 各 Initializer 的 destroy() 行为对比

| Initializer | destroy() 行为 | 不调用时的后果 |
|-------------|---------------|--------------|
| `SqljsInitializer` | `return undefined`（no-op） | **无影响**——sqljs 没有外部连接需要关闭 |
| `PostgresInitializer` | `return this.client.end()` | **pg Client 连接泄漏**——初始化时创建的管理连接不会被释放 |
| `MysqlInitializer` | `await this.conn.end()` | **mysql2 Connection 泄漏**——同上 |

这是 sqljs 与 postgres/mysql 在错误恢复上的一个重要差异：sqljs 的 `destroy()` 是安全的 no-op，而 postgres/mysql 必须被调用才能释放资源。

### 7.6 `clearAllTables()` —— 手动数据库重置工具

**文件**：`packages/testing/src/data-population/clear-all-tables.ts`

这是唯一一个使用了 `try...finally` 模式的清理函数：

```ts
export async function clearAllTables(config: VendureConfig, logging = true) {
    config = await preBootstrapConfig(config);
    const connection = await createConnection({ ...config.dbConnectionOptions });
    try {
        await connection.synchronize(true);   // ← DROP + CREATE 所有表
    } catch (err: any) {
        console.error('Error occurred when attempting to clear tables!');
        console.log(err);
    } finally {
        await connection.close();             // ← 无论成功失败都关闭连接
    }
}
```

但注意这个函数**不在 e2e 测试的自动流程中使用**，它只在以下场景被手动调用：
- `packages/dev-server/populate-dev-server.ts`：开发服务器重置数据
- `packages/dev-server/load-testing/`：负载测试前清理数据库

这意味着 e2e 测试框架本身**没有自动的"脏数据清理"机制**——测试失败后的数据库状态取决于 initializer 的类型和失败的位置。

### 7.7 `awaitOutstandingJobs()` 的容错设计

**文件**：`packages/testing/src/data-population/populate-for-testing.ts:48-71`

```ts
async function awaitOutstandingJobs(app: INestApplicationContext) {
    const maxAttempts = 10;
    let attempts = 0;
    if (isInspectableJobQueueStrategy(jobQueueStrategy)) {
        function waitForJobQueueToBeIdle() {
            return new Promise<void>(resolve => {
                const interval = setInterval(async () => {
                    attempts++;
                    const { items } = await inspectableJobQueueStrategy.findMany();
                    const jobsOutstanding = items.filter(i => i.state === 'RUNNING' || i.state === 'PENDING');
                    if (jobsOutstanding.length === 0 || attempts >= maxAttempts) {
                        clearInterval(interval);
                        resolve();    // ← 超时后也 resolve，不抛异常
                    }
                }, 500);
            });
        }
        await waitForJobQueueToBeIdle();
    }
}
```

这是一个**软容错**设计：
- 最多轮询 10 次，每次间隔 500ms，总计最多等待 ~5s
- 如果 10 次后仍有未完成的任务，**不会报错**而是静默继续
- 如果 JobQueueStrategy 不支持 `findMany()` 检查（`isInspectableJobQueueStrategy` 为 false），直接跳过
- 这种设计避免了测试框架因 JobQueue 的正常异步行为而卡死，但也意味着偶尔可能在 Job 未完成时就开始 populate，导致数据不一致

**`findMany()` 抛异常的未处理分支**：

```ts
const interval = setInterval(async () => {
    attempts++;
    const { items } = await inspectableJobQueueStrategy.findMany();  // ← 无 try-catch
    // ...
}, 500);
```

`setInterval` 的回调是 `async` 函数，其中 `await inspectableJobQueueStrategy.findMany()` **没有 try-catch 保护**。如果 `findMany()` 抛出异常（例如数据库连接中断、策略内部状态异常），会产生一个 **unhandled promise rejection**。由于 `setInterval` 不会因回调内的异常而停止：

- `interval` 不会被 `clearInterval()` 清除——**定时器泄漏**
- 每隔 500ms 会持续触发同一个未处理的异常——**错误风暴**
- `waitForJobQueueToBeIdle()` 返回的 Promise **永远不会 resolve**——**测试框架挂起**

这是一个比超时更严重的故障模式：超时至少会静默继续，而 `findMany()` 异常会导致整个测试进程卡死，只能通过外部信号杀死。

### 7.8 配置污染的完整传播链

当 `init()` 过程中发生异常时，配置对象的临时修改会沿以下路径传播：

```
SqljsInitializer.populate() 失败
  ├─ connectionOptions.autoSave = true     ← 未回滚
  ├─ connectionOptions.synchronize = true  ← 未回滚
  │
  └─ 异常冒泡到 TestServer.init() 的 catch
       └─ throw e;  ← 没有清理，也没有 finally
            └─ this.bootstrap() 不会执行
                 └─ 污染的 config 不会被第二次 bootstrap 使用

populateForTesting() 内部失败（bootstrap 成功后）
  ├─ config.authOptions.requireVerification = false  ← 未回滚
  ├─ config.dbConnectionOptions.logging = false      ← 未回滚
  │
  └─ 异常冒泡到 SqljsInitializer.populate() 的 populateFn()
       └─ 进而冒泡到 TestServer.init() 的 catch
            └─ this.bootstrap() 不会执行
                 └─ 污染的 config 不会被第二次 bootstrap 使用
```

**在 `init()` 失败的默认路径上，污染的 config 不会被二次使用**，因为 `this.bootstrap()` 在异常后不会执行。但如果测试代码自行处理了 `init()` 的异常并随后单独调用 `bootstrap()`，就会触发以下风险：

```ts
// 风险场景：手动捕获 init 异常后继续使用
try {
    await server.init(options);
} catch (e) {
    // 忽略错误，尝试用已有数据库启动
    await server.bootstrap();  // ← 此时 config 已被污染
}
```

| 被污染的字段 | 在 `bootstrap()` 中的影响 |
|-------------|-------------------------|
| `requireVerification = false` | 所有认证操作跳过验证步骤，可能让需要验证的测试通过但不应该通过 |
| `dbConnectionOptions.logging = false` | 影响较小，只是禁止了 TypeORM 的 SQL 日志 |
| `autoSave = true`（sqljs） | 第二次 bootstrap 的 TypeORM 在每次写操作后都将整个数据库序列化到文件，**严重降低性能** |
| `synchronize = true`（sqljs） | TypeORM 启动时自动同步 schema——如果 `.sqlite` 文件已存在且 schema 有变更，可能**意外修改生产 schema**；如果是全新数据库则无影响 |

**sqljs 特有的连锁风险**：如果 `populateFn()` 在 `autoSave = true` 期间抛异常，且 `.sqlite` 文件已被部分写入（但 `autoSave` 来不及在异常前保存最终状态），那么：
- 留下的 `.sqlite` 文件可能处于**不一致的中间状态**
- 下次运行时 `fs.existsSync()` 返回 `true`，跳过 populate
- `bootstrap()` 连接到这个损坏的数据库，可能在查询时报错

这种情况下唯一的恢复方式是手动删除对应的 `.sqlite` 文件。

---

## 八、sqljs 与 postgres/mysql 在缓存命中与重建触发条件上的详细对比

### 8.1 核心差异总表

| 维度 | SqljsInitializer | PostgresInitializer / MysqlInitializer |
|------|-----------------|---------------------------------------|
| 存储模型 | 单个 `.sqlite` 文件 | 独立的 `e2e_xxx` 数据库 |
| 缓存检测 | `fs.existsSync(this.dbFilePath)` | **无缓存检测**——每次 `init()` 都 DROP + CREATE |
| 缓存命中行为 | 跳过整个 `populateFn()` | N/A（永远不命中） |
| 重建触发 | 文件不存在时 | 每次运行 |
| `synchronize` 时机 | 仅 populate 期间 `true`，之后改回 `false` | `init()` 时写入 `connectionOptions`，此后一直为 `true` |
| `autoSave` 时机 | 仅 populate 期间 `true`，之后改回 `false` | N/A（postgres/mysql 没有 autoSave 概念） |
| `destroy()` 行为 | no-op | 关闭管理连接 |
| 初始化前的破坏性操作 | 无 | `DROP DATABASE IF EXISTS` + `CREATE DATABASE` |
| 部分写入风险 | **有**——populate 中途失败可能留下不完整文件 | **无**——每次都从头创建 |

### 8.2 sqljs 缓存命中的判断逻辑详解

```ts
async populate(populateFn: () => Promise<void>): Promise<void> {
    if (!fs.existsSync(this.dbFilePath)) {   // ← 唯一判断条件
        // ... 执行 populate ...
    }
    // 文件已存在 → 什么都不做
}
```

缓存命中的条件**极其简单**：目标路径上存在同名文件。这带来几个微妙的行为：

1. **空文件也算"命中"**：如果之前的 populate 在写入数据前就崩溃了（比如 `mkdirSync` 成功但 `populateFn()` 抛异常），可能留下一个空的或损坏的 `.sqlite` 文件。下次运行会认为"缓存命中"而跳过填充，然后 bootstrap 阶段就会因为数据库为空而报错。

2. **schema 变更不会自动触发重建**：如果代码中的 entity 定义改变了（加字段、改类型等），但 `.sqlite` 文件还在，sqljs 会加载旧 schema 的数据库。由于 populate 被跳过，`synchronize` 也被设为 `false`，TypeORM 不会自动同步 schema，导致运行时错误。**必须手动删除 `__data__/` 目录下的 `.sqlite` 文件**。

3. **`mkdirSync` 自行保证目录存在**：`SqljsInitializer.populate()` 在执行 `populateFn()` 之前会先检查目录是否存在：

   ```ts
   if (!fs.existsSync(this.dbFilePath)) {
       const dirName = path.dirname(this.dbFilePath);
       if (!fs.existsSync(dirName)) {
           fs.mkdirSync(dirName);   // ← 不带 { recursive: true }
       }
       // ...
   }
   ```

   注意 `fs.mkdirSync(dirName)` 不带 `{ recursive: true }`，所以它只能创建一级子目录。但在实际使用中，`dirName` 总是形如 `packages/core/e2e/__data__`，其父目录 `packages/core/e2e/` 作为源码目录在 git clone 后就已存在，所以 `mkdirSync` 不会因父目录不存在而失败。

   `__data__/` 目录下的 `.gitkeep` 文件的作用**不是**保证 `mkdirSync` 不失败——即使没有 `.gitkeep`，只要 `packages/core/e2e/` 存在，`mkdirSync` 就能成功创建 `__data__/`。`.gitkeep` 的实际作用是让 git 追踪这个空目录，这样在 clone 后目录就已存在，`fs.existsSync(dirName)` 会返回 `true`，从而跳过 `mkdirSync` 调用。这只是一个微小的性能优化，不影响正确性。

4. **gitignore 保护**：`.gitignore` 规则 `e2e/__data__/*.sqlite` + `!e2e/__data__/.gitkeep` 确保 `.sqlite` 文件不被提交，但 `.gitkeep` 保留。

### 8.3 sqljs populate 期间的临时配置变异

```ts
// populate 开始前
connectionOptions.autoSave = true;      // 让 TypeORM 自动保存变更到文件
connectionOptions.synchronize = true;    // 让 TypeORM 自动同步 schema

await populateFn();                      // 执行耗时填充

await new Promise(resolve => setTimeout(resolve, this.postPopulateTimeoutMs));

// populate 完成后
connectionOptions.autoSave = false;      // 关闭自动保存（文件已固化）
connectionOptions.synchronize = false;   // 关闭自动同步
```

**如果 `populateFn()` 抛异常**：
- `autoSave` 和 `synchronize` **不会被恢复**为 `false`
- 但由于异常会冒泡到 `TestServer.init()` 的 catch 块，`this.bootstrap()` 不会执行，所以**在正常流程中，被污染的 config 不会到达第二次 bootstrap**
- 只有在测试代码手动捕获 `init()` 异常后单独调用 `bootstrap()` 时，这些被污染的值才会生效：
  - `synchronize = true`：TypeORM 启动时自动同步 schema——如果 `.sqlite` 文件已存在且 schema 有变更，可能意外修改已有 schema
  - `autoSave = true`：TypeORM 在每次写操作后都将整个数据库序列化到文件，严重降低性能

**`postPopulateTimeoutMs` 的作用**：

```ts
await new Promise(resolve => setTimeout(resolve, this.postPopulateTimeoutMs));
```

这是一个延迟保护，默认值为 `0`（不等待）。`default-search-plugin.bench.ts` 中设为 `1000`（1秒），原因是 populate 完成后 JobQueue 中可能还有搜索索引更新的异步任务在执行。如果在这些任务完成前就关闭应用（`app.close()`），sqljs 的内存数据库可能来不及保存最终状态。这个 timeout 给异步任务一个"宽限期"来完成。

### 8.4 postgres/mysql 的无缓存策略详解

```ts
// PostgresInitializer.init()
const dbName = this.getDbNameFromFilename(testFileName);
this.client = await this.getPostgresConnection(connectionOptions);
(connectionOptions as any).database = dbName;
(connectionOptions as any).synchronize = true;
await this.client.query(`DROP DATABASE IF EXISTS ${dbName}`);
await this.client.query(`CREATE DATABASE ${dbName}`);
return connectionOptions;
```

**关键区别**：
1. **每次都是全新开始**：`DROP + CREATE` 确保每次测试运行面对的是完全干净的数据库，不存在"脏数据"或"旧 schema"问题
2. **不需要缓存**：postgres/mysql 的 populate 操作通常比 sqljs 快得多（无需序列化整个数据库到文件），所以没有缓存优化的需求
3. **`synchronize` 始终为 `true`**：与 sqljs 不同，postgres/mysql 的 `synchronize` 在 `init()` 时写入 `connectionOptions` 后**永远不会再改回 `false`**。这意味着第二次 bootstrap 时 TypeORM 仍会尝试同步 schema——但由于数据库是新建的，这不会造成问题
4. **`init()` 阶段的破坏性**：DROP DATABASE 是不可逆的。如果在并行测试中有另一个进程正在使用同名数据库，会导致严重问题。不过由于数据库名基于文件名（`e2e_<test-file-name>`），只要不同测试文件不重名就不会冲突

### 8.5 三种 initializer 的完整生命周期对比

```
SqljsInitializer:
  init()     → 计算文件路径，写入 connectionOptions.location
  populate() → 文件不存在? → 临时变异 config → 执行 populateFn → 恢复 config
               文件已存在? → 什么都不做（缓存命中）
  destroy()  → no-op

PostgresInitializer:
  init()     → 连接 postgres 库 → DROP DATABASE → CREATE DATABASE → 写入 connectionOptions
  populate() → 直接执行 populateFn（无条件）
  destroy()  → 关闭管理连接

MysqlInitializer:
  init()     → 连接 mysql → DROP DATABASE → CREATE DATABASE → 写入 connectionOptions
  populate() → 直接执行 populateFn（无条件）
  destroy()  → 关闭管理连接
```

### 8.6 缓存失效场景汇总

| 场景 | sqljs 行为 | postgres/mysql 行为 |
|------|-----------|-------------------|
| 首次运行 | 文件不存在 → populate → 创建 .sqlite | 新建数据库 → populate |
| 再次运行（无变更） | 文件存在 → **跳过 populate**（缓存命中） | DROP + CREATE → **重新 populate** |
| Entity schema 变更 | 文件存在但 schema 旧 → 跳过 populate → **运行时报错** | DROP + CREATE → 重新 populate → 自动适配 |
| 数据库损坏 | 文件存在但损坏 → 跳过 populate → **运行时报错** | 不存在"损坏"概念（每次重建） |
| 测试数据调整 | 文件存在但数据旧 → 跳过 populate → **测试断言失败** | DROP + CREATE → 重新 populate → 数据总是最新 |
| 手动清除缓存 | `rm -rf __data__/*.sqlite` | 不需要 |

**实践建议**：
- sqljs 模式下，任何涉及 entity 或 initialData 的代码变更后，都应删除 `__data__/` 目录
- CI 环境中每次都是全新 checkout，不存在缓存文件，所以不存在此问题
- postgres/mysql 模式没有缓存问题，但代价是每次都要重新 populate
