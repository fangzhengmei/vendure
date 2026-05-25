# 配送方式选择到履约创建流程分析

本文档详细分析 Vendure 中从配送方式选择到订单履约生成与状态推进的完整流程。

## 重要修正说明

> **之前的分析存在以下错误，已在本文档中修正：**
> 1. ❌ ~~创建履约必须先到支付完成状态~~ → ✅ **创建履约本身不校验订单状态**，只校验数量和库存
> 2. ❌ ~~未明确 ShippingMethod 与 FulfillmentHandler 的连接~~ → ✅ **ShippingMethod.fulfillmentHandlerCode 是前端默认选择 FulfillmentHandler 的依据**
> 3. ❌ ~~状态联动触发时机不明确~~ → ✅ **只有履约状态变为 Shipped/Delivered 时才触发订单状态联动**，Created→Pending 不触发

---

## 一、整体流程图

```
用户/管理员
    │
    ▼
[1] 获取可用配送方式 (ShippingCalculator.getEligibleShippingMethods)
    │  ├─ ShippingEligibilityChecker.check()  资格检查
    │  └─ ShippingCalculator.calculate()     运费计算
    ▼
[2] 设置订单配送方式 (OrderModifier.setShippingMethods)
    │  ├─ 再次验证配送方式资格
    │  ├─ 创建/更新 ShippingLine
    │  └─ 分配 ShippingLine 到 OrderLine
    │
    │  ╔══════════════════════════════════════════════════════════╗
    │  ║  关键连接: ShippingMethod.fulfillmentHandlerCode        ║
    │  ║  → 存储该配送方式关联的 FulfillmentHandler 的 code      ║
    │  ║  → 前端创建履约时自动匹配对应 handler（可手动更改）     ║
    │  ╚══════════════════════════════════════════════════════════╝
    ▼
[3] 订单状态推进 (OrderStateMachine)
    │  Created → AddingItems → ArrangingPayment → PaymentAuthorized/PaymentSettled
    │  (关键检查点: arrangingPaymentRequiresShipping)
    ▼
[4] 创建履约 (OrderService.createFulfillment → FulfillmentService.create)
    │  ╔══════════════════════════════════════════════════════════╗
    │  ║  ✅ 实际校验条件（无订单状态检查！）                     ║
    │  ║  1. 不能空选商品                                        ║
    │  ║  2. 履约数量 ≤ 未履约数量                               ║
    │  ║  3. 库存充足                                           ║
    │  ╚══════════════════════════════════════════════════════════╝
    │  ├─ FulfillmentHandler.createFulfillment()  生成物流信息
    │  ├─ 创建 Fulfillment 实体 (初始状态: Created)
    │  ├─ 创建 FulfillmentLine 关联
    │  └─ 自动转换状态: Created → Pending
    │     (触发库存扣减，但不触发订单状态联动)
    ▼
[5] 履约状态推进 (FulfillmentStateMachine)
    │  Pending → Shipped → Delivered
    │  (触发: transitionFulfillmentToState API)
    │
    │  ╔══════════════════════════════════════════════════════════╗
    │  ║  状态联动触发条件:                                      ║
    │  ║  仅当 toState === 'Shipped' 或 'Delivered' 时           ║
    │  ║  才会尝试自动转换订单状态                               ║
    │  ╚══════════════════════════════════════════════════════════╝
    ▼
[6] 订单状态自动联动 (defaultFulfillmentProcess.onTransitionEnd)
    ├─ 检查: 订单当前状态是否允许转换到目标状态?
    ├─ 履约 Shipped → 检查: 全部发货? Shipped : PartiallyShipped
    └─ 履约 Delivered → 检查: 全部送达? Delivered : PartiallyDelivered
```

---

## 二、详细流程分析

### 1. 配送方式计算与资格检查

#### 核心实体与组件

| 组件 | 文件 | 职责 |
|------|------|------|
| `ShippingMethod` | `entity/shipping-method/shipping-method.entity.ts` | 配送方式实体，关联 checker、calculator 和 fulfillmentHandlerCode |
| `ShippingEligibilityChecker` | `config/shipping-method/shipping-eligibility-checker.ts` | 配送资格检查器，判断订单是否适用该配送方式 |
| `ShippingCalculator` | `config/shipping-method/shipping-calculator.ts` | 运费计算器，计算具体运费金额 |
| `ShippingCalculator` (服务) | `service/helpers/shipping-calculator/shipping-calculator.ts` | 配送计算服务，协调 checker 和 calculator |

#### ShippingMethod 实体的核心字段

`shipping-method.entity.ts:63-64`:
```typescript
@Column()
fulfillmentHandlerCode: string;  // 关键：关联 FulfillmentHandler 的 code
```

#### 关键代码流程

**获取可用配送方式** (`shipping-calculator.ts:26-41`):
```typescript
async getEligibleShippingMethods(ctx, order, skipIds) {
    // 1. 获取所有激活的配送方式
    const shippingMethods = await this.shippingMethodService.getActiveShippingMethods(ctx);
    
    // 2. 并行检查每个配送方式的资格
    const eligibleMethods = await Promise.all(
        shippingMethods.map(method => this.checkEligibilityByShippingMethod(ctx, order, method))
    );
    
    // 3. 按价格排序返回
    return eligibleMethods.filter(notNullOrUndefined)
        .sort((a, b) => a.result.price - b.result.price);
}
```

**资格检查与运费计算** (`shipping-calculator.ts:57-69`):
```typescript
private async checkEligibilityByShippingMethod(ctx, order, method) {
    // 调用 ShippingEligibilityChecker.check()
    const eligible = await method.test(ctx, order);
    if (eligible) {
        // 调用 ShippingCalculator.calculate()
        const result = await method.apply(ctx, order);
        if (result) {
            return { method, result };
        }
    }
}
```

---

### 2. 设置订单配送方式

#### 调用链路

```
API: setOrderShippingMethod
    ↓
OrderService.setShippingMethod
    ↓
OrderModifier.setShippingMethods  (order-modifier.ts:699-760)
```

#### 关键验证与操作

`order-modifier.ts:699-738`:
```typescript
async setShippingMethods(ctx, order, shippingMethodIds) {
    for (const [i, shippingMethodId] of shippingMethodIds.entries()) {
        // 1. 再次验证配送方式资格
        const shippingMethod = await this.shippingCalculator.getMethodIfEligible(
            ctx, order, shippingMethodId
        );
        if (!shippingMethod) {
            return new IneligibleShippingMethodError();
        }
        
        // 2. 创建或更新 ShippingLine
        let shippingLine = order.shippingLines[i];
        if (shippingLine) {
            shippingLine.shippingMethod = shippingMethod;
        } else {
            shippingLine = await this.connection.getRepository(ctx, ShippingLine).save(
                new ShippingLine({ shippingMethod, order, ... })
            );
            order.shippingLines.push(shippingLine);
        }
    }
    
    // 3. 移除多余的 ShippingLine
    if (shippingMethodIds.length < order.shippingLines.length) {
        const shippingLinesToDelete = order.shippingLines.splice(shippingMethodIds.length - 1);
        await this.connection.getRepository(ctx, ShippingLine).remove(shippingLinesToDelete);
    }
    
    // 4. 分配 ShippingLine 到 OrderLine (通过 shippingLineAssignmentStrategy)
    const { shippingLineAssignmentStrategy } = this.configService.shippingOptions;
    for (const shippingLine of order.shippingLines) {
        // ... 分配逻辑
    }
}
```

---

### 3. ShippingMethod 与 FulfillmentHandler 的连接

#### 连接关系图

```
配置阶段:
  VendureConfig.shippingOptions.fulfillmentHandlers
    ↓ (注册所有可用 handler)
  ShippingMethod.fulfillmentHandlerCode
    ↓ (存储选中的 handler code)

创建履约阶段:
  前端: order.shippingLines[0].shippingMethod.fulfillmentHandlerCode
    ↓ (查找匹配的 handler，作为默认选择)
  FulfillOrderInput.handler (可手动修改)
    ↓ (传递给后端)
  Fulfillment.handlerCode (保存到履约实体)
    ↓ (后续状态转换时使用)
  FulfillmentHandler.onFulfillmentTransition()
```

#### 关键代码

**ShippingMethod 创建/更新时验证 handlerCode** (`shipping-method.service.ts:117-120`):
```typescript
beforeSave: method => {
    method.fulfillmentHandlerCode = this.ensureValidFulfillmentHandlerCode(
        method.code,
        input.fulfillmentHandler,
    );
    // ...
}
```

**前端创建履约时自动选择 handler** (`fulfill-order-dialog.component.ts:50-58`):
```typescript
this.dataService.shippingMethod
    .getShippingMethodOperations()
    .mapSingle(data => data.fulfillmentHandlers)
    .subscribe(handlers => {
        // 关键：根据 ShippingMethod.fulfillmentHandlerCode 匹配 handler
        this.fulfillmentHandlerDef =
            handlers.find(
                h => h.code === this.order.shippingLines[0]?.shippingMethod?.fulfillmentHandlerCode,
            ) || handlers[0];  // 找不到则用第一个
        // ...
    });
```

**后端创建履约时使用传入的 handler** (`fulfillment.service.ts:56-57`):
```typescript
// 直接使用前端传入的 handler.code，不验证是否与 ShippingMethod 匹配
const fulfillmentHandler = this.configService.shippingOptions.fulfillmentHandlers.find(
    h => h.code === handler.code,
);
```

**履约状态转换时调用对应 handler 的钩子** (`default-fulfillment-process.ts:82-91`):
```typescript
async onTransitionStart(fromState, toState, data) {
    const { fulfillmentHandlers } = configService.shippingOptions;
    // 根据 Fulfillment.handlerCode 找到对应的 handler
    const fulfillmentHandler = fulfillmentHandlers.find(
        h => h.code === data.fulfillment.handlerCode
    );
    if (fulfillmentHandler) {
        // 调用该 handler 的状态转换钩子
        const result = await awaitPromiseOrObservable(
            fulfillmentHandler.onFulfillmentTransition(fromState, toState, data),
        );
        if (result === false || typeof result === 'string') {
            return result;
        }
    }
}
```

---

### 4. 订单状态推进

#### 订单状态机配置

`default-order-process.ts:174-236` 定义了完整的状态转换图：

```
Created → AddingItems → ArrangingPayment → PaymentAuthorized
                                          ↘ PaymentSettled → PartiallyShipped/Shipped
                                                          ↘ PartiallyDelivered/Delivered
```

#### 关键检查点

`default-order-process.ts:317-351` - 进入 `ArrangingPayment` 状态前的检查：

```typescript
if (toState === 'ArrangingPayment') {
    // 检查1: 订单不能为空
    if (options.arrangingPaymentRequiresContents !== false && order.lines.length === 0) {
        return 'message.cannot-transition-to-payment-when-order-is-empty';
    }
    // 检查2: 必须有客户信息
    if (options.arrangingPaymentRequiresCustomer !== false && !order.customer) {
        return 'message.cannot-transition-to-payment-without-customer';
    }
    // 检查3: 必须设置配送方式 (关键衔接点!)
    if (options.arrangingPaymentRequiresShipping !== false &&
        (!order.shippingLines || order.shippingLines.length === 0)) {
        return 'message.cannot-transition-to-payment-without-shipping-method';
    }
    // 检查4: 库存充足
    // ...
}
```

#### 支付完成检查

`default-order-process.ts:353-362`:
```typescript
if (toState === 'PaymentAuthorized') {
    if (!orderTotalIsCovered(order, ['Authorized', 'Settled']) || !hasAnAuthorizedPayment) {
        return 'message.cannot-transition-without-authorized-payments';
    }
}
if (toState === 'PaymentSettled' && !orderTotalIsCovered(order, 'Settled')) {
    return 'message.cannot-transition-without-settled-payments';
}
```

#### 订单可转换到履约相关状态的前提

`default-order-process.ts:190-199` - **只有这些状态才能转换到 Shipped/Delivered**:
```typescript
PaymentSettled: {
    to: [
        'PartiallyDelivered',
        'Delivered',
        'PartiallyShipped',
        'Shipped',
        'Cancelled',
        'Modifying',
        'ArrangingAdditionalPayment',
    ],
},
PartiallyShipped: {
    to: ['Shipped', 'PartiallyDelivered', 'Cancelled', 'Modifying'],
},
Shipped: {
    to: ['PartiallyDelivered', 'Delivered', 'Cancelled', 'Modifying'],
},
// ...
```

---

### 5. 履约创建流程（重点修正！）

#### API 入口

`admin/order.resolver.ts:98-103`:
```typescript
@Mutation()
@Allow(Permission.UpdateOrder)
async addFulfillmentToOrder(@Ctx() ctx, @Args() args) {
    return this.orderService.createFulfillment(ctx, args.input);
}
```

#### FulfillOrderInput 结构

```typescript
type FulfillOrderInput = {
    handler: ConfigurableOperationInput;  // 前端根据 ShippingMethod.fulfillmentHandlerCode 选择
    lines: Array<OrderLineInput>;         // 订单行及数量
};
```

#### OrderService.createFulfillment 完整流程

`order.service.ts:1736-1781`:

```typescript
async createFulfillment(ctx, input) {
    // ========== 验证阶段 ==========
    // ⚠️ 重要：此处没有任何订单状态检查！
    
    // 验证1: 不能空选商品
    if (!input.lines || input.lines.length === 0 || summate(input.lines, 'quantity') === 0) {
        return new EmptyOrderLineSelectionError();
    }
    
    // 验证2: 履约数量不能超过未履约数量
    if (await this.requestedFulfillmentQuantityExceedsLineQuantity(ctx, input)) {
        return new ItemsAlreadyFulfilledError();
    }
    
    // 验证3: 库存充足
    const stockCheckResult = await this.ensureSufficientStockForFulfillment(ctx, input);
    if (isGraphQlErrorResult(stockCheckResult)) {
        return stockCheckResult;
    }
    
    // ========== 创建阶段 ==========
    
    // 步骤1: 调用 FulfillmentService 创建履约
    const fulfillment = await this.fulfillmentService.create(ctx, orders, input.lines, input.handler);
    if (isGraphQlErrorResult(fulfillment)) {
        return fulfillment;
    }
    
    // 步骤2: 关联订单与履约
    await this.connection
        .getRepository(ctx, Order)
        .createQueryBuilder()
        .relation('fulfillments')
        .of(orders)
        .add(fulfillment);
    
    // 步骤3: 记录历史
    for (const order of orders) {
        await this.historyService.createHistoryEntryForOrder({
            ctx, orderId: order.id,
            type: HistoryEntryType.ORDER_FULFILLMENT,
            data: { fulfillmentId: fulfillment.id },
        });
    }
    
    // 步骤4: 自动转换履约状态 Created → Pending
    // 注意：此转换会触发库存扣减，但不会触发订单状态联动！
    const result = await this.fulfillmentService.transitionToState(ctx, fulfillment.id, 'Pending');
    if (isGraphQlErrorResult(result)) {
        return result;
    }
    
    return result.fulfillment;
}
```

#### 数量验证逻辑

`order.service.ts:1783-1821`:
```typescript
private async requestedFulfillmentQuantityExceedsLineQuantity(ctx, input) {
    // 查询该订单行已有的非取消履约
    const existingFulfillmentLines = await this.connection
        .getRepository(ctx, FulfillmentLine)
        .createQueryBuilder('fulfillmentLine')
        .leftJoinAndSelect('fulfillmentLine.orderLine', 'orderLine')
        .leftJoinAndSelect('fulfillmentLine.fulfillment', 'fulfillment')
        .where('fulfillmentLine.orderLineId IN (:...orderLineIds)', { ... })
        .andWhere('fulfillment.state != :state', { state: 'Cancelled' })
        .getMany();
    
    // 计算未履约数量 = 订单行数量 - 已履约数量
    const fulfilledQuantity = summate(fulfillmentLinesForOrderLine, 'quantity');
    const unfulfilledQuantity = orderLine.quantity - fulfilledQuantity;
    
    // 验证: 本次履约数量 ≤ 未履约数量
    return unfulfilledQuantity < inputLine.quantity;
}
```

#### FulfillmentService.create 核心

`fulfillment.service.ts:50-122`:
```typescript
async create(ctx, orders, lines, handler) {
    // 1. 查找并验证 FulfillmentHandler
    const fulfillmentHandler = this.configService.shippingOptions.fulfillmentHandlers.find(
        h => h.code === handler.code
    );
    if (!fulfillmentHandler) {
        return new InvalidFulfillmentHandlerError();
    }
    
    // 2. 调用 FulfillmentHandler.createFulfillment()
    // 这里可以集成第三方物流 API，生成运单号等
    let fulfillmentPartial;
    try {
        fulfillmentPartial = await fulfillmentHandler.createFulfillment(
            ctx, orders, lines, handler.arguments
        );
    } catch (e) {
        return new CreateFulfillmentError({ fulfillmentHandlerError: message });
    }
    
    // 3. 创建 Fulfillment 实体，初始状态为 Created
    const newFulfillment = await this.connection.getRepository(ctx, Fulfillment).save(
        new Fulfillment({
            method: '',
            trackingCode: '',
            ...fulfillmentPartial,
            lines: [],
            state: this.fulfillmentStateMachine.getInitialState(), // 'Created'
            handlerCode: fulfillmentHandler.code,  // 保存 handler code
        })
    );
    
    // 4. 创建 FulfillmentLine 关联 (订单行 ↔ 履约，带数量)
    const fulfillmentLines: FulfillmentLine[] = [];
    for (const { orderLineId, quantity } of lines) {
        const fulfillmentLine = await this.connection.getRepository(ctx, FulfillmentLine).save(
            new FulfillmentLine({ orderLineId, quantity })
        );
        fulfillmentLines.push(fulfillmentLine);
    }
    
    // 5. 关联 FulfillmentLine 到 Fulfillment
    await this.connection
        .getRepository(ctx, Fulfillment)
        .createQueryBuilder()
        .relation('lines')
        .of(newFulfillment)
        .add(fulfillmentLines);
    
    // 6. 发布事件
    await this.eventBus.publish(new FulfillmentEvent(ctx, fulfillmentWithRelations, { ... }));
    
    return newFulfillment;
}
```

---

### 6. 履约状态推进与订单状态联动（重点修正！）

#### 履约状态机配置

`default-fulfillment-process.ts:46-62`:
```typescript
transitions: {
    Created: { to: ['Pending'] },
    Pending: { to: ['Shipped', 'Delivered', 'Cancelled'] },
    Shipped: { to: ['Delivered', 'Cancelled'] },
    Delivered: { to: ['Cancelled'] },
    Cancelled: { to: [] },
}
```

#### 状态转换 API

`admin/order.resolver.ts:180-185`:
```typescript
@Mutation()
@Allow(Permission.UpdateOrder)
async transitionFulfillmentToState(@Ctx() ctx, @Args() args) {
    return this.orderService.transitionFulfillmentToState(ctx, args.id, args.state);
}
```

#### 履约状态转换核心

`fulfillment.service.ts:154-205`:
```typescript
async transitionToState(ctx, fulfillmentId, state) {
    return this.connection.withTransaction(ctx, async txCtx => {
        // 1. 获取履约及其关联的订单行
        const fulfillment = await this.connection.getEntityOrThrow(
            txCtx, Fulfillment, fulfillmentId, { relations: ['lines'] }
        );
        
        // 2. 查询关联的订单
        const orderLinesIds = unique(fulfillment.lines.map(l => l.orderLineId));
        const orders = await this.connection
            .getRepository(txCtx, Order)
            .createQueryBuilder('order')
            .leftJoinAndSelect('order.lines', 'line')
            .where('line.id IN (:...lineIds)', { lineIds: orderLinesIds })
            .getMany();
        
        // 3. 执行状态转换
        const fromState = fulfillment.state;
        const result = await this.fulfillmentStateMachine.transition(
            txCtx, fulfillment, orders, state
        );
        
        // 4. 保存状态
        await this.connection.getRepository(txCtx, Fulfillment).save(fulfillment, { reload: false });
        
        // 5. 发布事件
        await this.eventBus.publish(
            new FulfillmentStateTransitionEvent(fromState, state, txCtx, fulfillment)
        );
        
        // 6. 执行 finalize 钩子 (触发 onTransitionEnd)
        // ⚠️ 订单状态联动就是在这里触发的
        await result.finalize();
        
        return { fulfillment, orders, fromState, toState: state };
    });
}
```

#### 状态联动触发的完整时间线

```
调用 transitionFulfillmentToState(state)
    ↓
1. FulfillmentStateMachine.transition()
    ├─ onTransitionStart (调用 FulfillmentHandler.onFulfillmentTransition)
    └─ 执行状态转换
    ↓
2. 保存 Fulfillment.state
    ↓
3. 发布 FulfillmentStateTransitionEvent
    ↓
4. 调用 result.finalize() → 触发 onTransitionEnd
    ├─ 库存处理 (Cancelled 时恢复库存，Created→Pending 时扣减库存)
    ├─ 记录历史
    └─ ⚡ 调用 handleFulfillmentStateTransitByOrder() → 尝试转换订单状态
        ├─ 检查 toState 是否为 'Shipped' 或 'Delivered'
        │  (Created→Pending 不会触发订单状态联动！)
        ├─ 检查订单当前的 nextStates 是否包含目标状态
        │  (如果订单还在 ArrangingPayment，就无法转换到 Shipped)
        ├─ 检查所有订单行的履约状态
        └─ 调用 orderService.transitionToState()
            └─ OrderStateMachine.onTransitionStart
                └─ checkFulfillmentStates 校验 (反向验证履约状态)
```

#### onTransitionEnd 钩子（状态联动入口）

`default-fulfillment-process.ts:93-125`:
```typescript
async onTransitionEnd(fromState, toState, { ctx, fulfillment, orders }) {
    // 库存处理...
    
    // 记录历史...
    
    // 关键: 为每个关联订单触发状态联动
    await Promise.all(
        orders.map(order =>
            handleFulfillmentStateTransitByOrder(ctx, order, fulfillment, fromState, toState)
        )
    );
}
```

#### 状态联动逻辑（重点！）

`default-fulfillment-process.ts:128-162`:
```typescript
async function handleFulfillmentStateTransitByOrder(
    ctx, order, fulfillment, fromState, toState
) {
    // 获取订单当前可转换的下一个状态
    const nextOrderStates = orderService.getNextOrderStates(order);
    
    // 尝试转换订单状态
    const transitionOrderIfStateAvailable = async (state: OrderState) => {
        // ⚠️ 关键检查：订单当前状态是否允许转换到目标状态
        if (nextOrderStates.includes(state)) {
            const result = await orderService.transitionToState(ctx, order.id, state);
            if (isGraphQlErrorResult(result)) {
                throw new InternalServerError(result.message);
            }
        }
        // 如果不允许，则静默跳过，不报错
    };
    
    // ⚠️ 只有 Shipped 和 Delivered 状态才会触发订单状态联动
    // Created → Pending 的转换不会走到这里！
    
    if (toState === 'Shipped') {
        const orderWithFulfillment = await getOrderWithFulfillments(ctx, order.id);
        if (orderItemsAreShipped(orderWithFulfillment)) {
            // 全部已发货 → 订单状态转 Shipped
            await transitionOrderIfStateAvailable('Shipped');
        } else {
            // 部分发货 → 订单状态转 PartiallyShipped
            await transitionOrderIfStateAvailable('PartiallyShipped');
        }
    }
    
    if (toState === 'Delivered') {
        const orderWithFulfillment = await getOrderWithFulfillments(ctx, order.id);
        if (orderItemsAreDelivered(orderWithFulfillment)) {
            // 全部已送达 → 订单状态转 Delivered
            await transitionOrderIfStateAvailable('Delivered');
        } else {
            // 部分送达 → 订单状态转 PartiallyDelivered
            await transitionOrderIfStateAvailable('PartiallyDelivered');
        }
    }
}
```

#### 订单发货/送达判断逻辑

`order-utils.ts:101-106` - 判断是否全部发货：
```typescript
export function orderItemsAreShipped(order: Order) {
    return (
        getOrderLinesFulfillmentStates(order).every(state => state === 'Shipped') &&
        !isOrderPartiallyFulfilled(order)
    );
}
```

`order-utils.ts:66-85` - 获取订单行的履约状态：
```typescript
function getOrderLinesFulfillmentStates(order: Order): Array<FulfillmentState | undefined> {
    const fulfillmentLines = getOrderFulfillmentLines(order);
    return unique(
        order.lines
            .filter(line => line.quantity !== 0)
            .map(line => {
                const matchingFulfillmentLines = fulfillmentLines.filter(fl =>
                    idsAreEqual(fl.orderLineId, line.id)
                );
                const totalFulfilled = summate(matchingFulfillmentLines, 'quantity');
                if (0 < totalFulfilled) {
                    return matchingFulfillmentLines.map(l => l.fulfillment.state);
                } else {
                    return undefined;
                }
            })
            .flat()
    );
}
```

#### 订单状态转换的前置校验（反向校验）

`default-order-process.ts:375-400` - 订单状态转换时会反向校验履约状态：
```typescript
if (options.checkFulfillmentStates !== false) {
    if (toState === 'PartiallyShipped') {
        const orderWithFulfillments = await findOrderWithFulfillments(ctx, order.id);
        if (!orderItemsArePartiallyShipped(orderWithFulfillments)) {
            return 'message.cannot-transition-unless-some-order-items-shipped';
        }
    }
    if (toState === 'Shipped') {
        const orderWithFulfillments = await findOrderWithFulfillments(ctx, order.id);
        if (!orderItemsAreShipped(orderWithFulfillments)) {
            return 'message.cannot-transition-unless-all-order-items-shipped';
        }
    }
    // PartiallyDelivered、Delivered 同理...
}
```

---

## 三、关键衔接点总结

### 衔接点 1: ShippingMethod → FulfillmentHandler（配置关联）
- **位置**: `shipping-method.entity.ts:63-64` + `fulfill-order-dialog.component.ts:55-56`
- **逻辑**: 
  - `ShippingMethod.fulfillmentHandlerCode` 存储关联的 handler code
  - 前端创建履约时，根据订单配送方式的 `fulfillmentHandlerCode` 自动匹配对应的 `FulfillmentHandler`
  - 后端不强制校验，允许手动选择其他 handler
- **意义**: 实现"配送方式"与"履约处理方式"的解耦关联

### 衔接点 2: 配送方式 → 订单支付
- **位置**: `default-order-process.ts:324-329`
- **逻辑**: `arrangingPaymentRequiresShipping` 检查确保必须先设置配送方式才能进入支付流程
- **意义**: 保证订单在支付前已经确定了配送方式和运费

### 衔接点 3: 履约创建校验条件（重要修正！）
- **位置**: `order.service.ts:1736-1781`
- **逻辑**: 创建履约**不检查订单状态**，只检查三个条件：
  1. 不能空选商品
  2. 履约数量 ≤ 未履约数量
  3. 库存充足
- **注意**: 理论上可以在 `AddingItems` 状态就创建履约，但订单状态无法自动推进到 `Shipped`

### 衔接点 4: 履约创建 → 自动转 Pending
- **位置**: `order.service.ts:1776`
- **逻辑**: 履约创建后立即调用 `transitionToState(ctx, fulfillment.id, 'Pending')`
- **意义**: 触发库存扣减（Sale 库存移动）
- **⚠️ 重要**: 此状态转换**不会**触发订单状态联动

### 衔接点 5: 履约状态变化 → 订单状态自动联动（最重要！）
- **位置**: `default-fulfillment-process.ts:128-162`
- **触发条件**: 仅当 `toState === 'Shipped'` 或 `toState === 'Delivered'` 时
- **逻辑**:
  1. 检查订单当前状态是否允许转换到目标状态（通过 `getNextOrderStates`）
  2. 检查所有订单行的履约状态（全部还是部分）
  3. 尝试转换订单状态
- **双向校验**: 订单状态转换时也会反向校验履约状态，确保数据一致性
- **静默失败**: 如果订单状态不允许转换（如还在 `ArrangingPayment`），则静默跳过，不报错

---

## 四、关键数据模型

### ShippingMethod
```
├─ id
├─ code
├─ checker: ConfigurableOperation (关联 ShippingEligibilityChecker)
├─ calculator: ConfigurableOperation (关联 ShippingCalculator)
├─ fulfillmentHandlerCode: string  ⭐ 关键：关联 FulfillmentHandler
└─ channels: Channel[]
```

### ShippingLine
```
├─ id
├─ order: Order
├─ shippingMethod: ShippingMethod
├─ shippingMethodId: ID
├─ listPrice: number
├─ adjustments: Adjustment[]
└─ taxLines: TaxLine[]
```

### Fulfillment
```
├─ id
├─ state: FulfillmentState (Created, Pending, Shipped, Delivered, Cancelled)
├─ method: string
├─ trackingCode: string
├─ handlerCode: string  ⭐ 关键：创建时使用的 FulfillmentHandler code
├─ lines: FulfillmentLine[]
└─ customFields: any
```

### FulfillmentLine
```
├─ id
├─ orderLineId: ID (关联 OrderLine)
├─ quantity: number
└─ fulfillment: Fulfillment
```

---

## 五、常见场景分析

### 场景 1: 正常流程（推荐）
```
订单状态: Created → AddingItems → ArrangingPayment → PaymentSettled
                                                     ↓ (创建履约)
                                                履约: Created → Pending
                                                     ↓ (发货)
                                                履约: Pending → Shipped
                                                     ↓ (自动联动)
                                                订单: PaymentSettled → Shipped
                                                     ↓ (送达)
                                                履约: Shipped → Delivered
                                                     ↓ (自动联动)
                                                订单: Shipped → Delivered
```

### 场景 2: 提前创建履约（不推荐，但技术上允许）
```
订单状态: AddingItems
    ↓ (创建履约)
履约: Created → Pending  ✅ 成功（只检查数量和库存）
    ↓ (发货)
履约: Pending → Shipped  ✅ 成功
    ↓ (尝试自动联动)
订单: nextStates 不包含 'Shipped' ❌ 静默跳过，订单状态仍为 AddingItems
```

### 场景 3: 先发货后收款
```
订单状态: PaymentAuthorized（已授权未结算）
    ↓ (创建履约并发货)
履约: Created → Pending → Shipped
    ↓ (自动联动检查)
订单: PaymentAuthorized 的 nextStates 包含 'Shipped' 吗？
    ↓ (看 default-order-process.ts:187-189)
PaymentAuthorized: {
    to: ['PaymentSettled', 'Cancelled', 'Modifying', 'ArrangingAdditionalPayment'],
    // ⚠️ 不包含 PartiallyShipped/Shipped！
}
    ↓
结果: ❌ 订单状态无法自动推进到 Shipped
    ↓ (需要先收款)
订单: PaymentAuthorized → PaymentSettled
    ↓ (手动转换或再次触发履约状态)
订单: PaymentSettled → Shipped
```

---

## 六、核心文件索引

| 模块 | 文件路径 |
|------|----------|
| 配送方式实体 | `packages/core/src/entity/shipping-method/shipping-method.entity.ts` |
| 配送资格检查器 | `packages/core/src/config/shipping-method/shipping-eligibility-checker.ts` |
| 运费计算器 | `packages/core/src/config/shipping-method/shipping-calculator.ts` |
| 配送计算服务 | `packages/core/src/service/helpers/shipping-calculator/shipping-calculator.ts` |
| 配送方式服务 | `packages/core/src/service/services/shipping-method.service.ts` |
| 订单修改器 | `packages/core/src/service/helpers/order-modifier/order-modifier.ts` |
| 订单服务 | `packages/core/src/service/services/order.service.ts` |
| 履约服务 | `packages/core/src/service/services/fulfillment.service.ts` |
| 履约处理器 | `packages/core/src/config/fulfillment/fulfillment-handler.ts` |
| 手动履约处理器 | `packages/core/src/config/fulfillment/manual-fulfillment-handler.ts` |
| 默认订单流程 | `packages/core/src/config/order/default-order-process.ts` |
| 默认履约流程 | `packages/core/src/config/fulfillment/default-fulfillment-process.ts` |
| 订单状态机 | `packages/core/src/service/helpers/order-state-machine/order-state-machine.ts` |
| 履约状态机 | `packages/core/src/service/helpers/fulfillment-state-machine/fulfillment-state-machine.ts` |
| 订单工具函数 | `packages/core/src/service/helpers/utils/order-utils.ts` |
| 管理端订单 API | `packages/core/src/api/resolvers/admin/order.resolver.ts` |
| 前端履约对话框 | `packages/admin-ui/src/lib/order/src/components/fulfill-order-dialog/fulfill-order-dialog.component.ts` |
