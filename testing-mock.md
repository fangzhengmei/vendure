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

`TestServer.init()` 实际上启动了两次 NestJS 应用：
1. 第一次：仅用于数据填充，填充完立即关闭
2. 第二次：正式启动，连接填充好的数据库，供测试使用

这种设计解耦了"数据准备"和"测试运行"两个阶段，也让 `bootstrap()` 方法可以独立使用（例如需要重启服务但不重新填充数据的场景）。

### 6.4 确定性随机

`MockDataService` 通过 `faker.seed(1)` 确保每次生成的客户数据完全相同。这意味着不同机器、不同时间运行的测试面对的是同一组客户数据，消除了随机性带来的测试不稳定。
