# 配送方式选择到履约创建流程分析

本文档详细分析 Vendure 中从配送方式选择到订单履约生成与状态推进的完整流程。

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
    ▼
[3] 订单状态推进到支付完成 (OrderStateMachine)
    │  Created → AddingItems → ArrangingPayment → PaymentAuthorized/PaymentSettled
    │  (关键检查点: arrangingPaymentRequiresShipping)
    ▼
[4] 创建履约 (OrderService.createFulfillment → FulfillmentService.create)
    │  ├─ 验证: 履约数量 ≤ 未履约数量
    │  ├─ 验证: 库存充足
    │  ├─ FulfillmentHandler.createFulfillment()  生成物流信息
    │  ├─ 创建 Fulfillment 实体 (初始状态: Created)
    │  ├─ 创建 FulfillmentLine 关联
    │  └─ 自动转换状态: Created → Pending
    ▼
[5] 履约状态推进 (FulfillmentStateMachine)
    │  Pending → Shipped → Delivered
    │  (触发: transitionFulfillmentToState API)
    ▼
[6] 订单状态自动联动 (defaultFulfillmentProcess.onTransitionEnd)
    ├─ 履约 Shipped → 检查: 全部发货? Shipped : PartiallyShipped
    └─ 履约 Delivered → 检查: 全部送达? Delivered : PartiallyDelivered
```

---

## 二、详细流程分析

### 1. 配送方式计算与资格检查

#### 核心实体与组件

| 组件 | 文件 | 职责 |
|------|------|------|
| `ShippingMethod` | `entity/shipping-method/shipping-method.entity.ts` | 配送方式实体，关联 checker 和 calculator |
| `ShippingEligibilityChecker` | `config/shipping-method/shipping-eligibility-checker.ts` | 配送资格检查器，判断订单是否适用该配送方式 |
| `ShippingCalculator` | `config/shipping-method/shipping-calculator.ts` | 运费计算器，计算具体运费金额 |
| `ShippingCalculator` (服务) | `service/helpers/shipping-calculator/shipping-calculator.ts` | 配送计算服务，协调 checker 和 calculator |

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

#### ShippingMethod 实体的核心方法

`shipping-method.entity.ts:76-99`:
```typescript
// 运费计算
async apply(ctx: RequestContext, order: Order): Promise<ShippingCalculationResult | undefined> {
    const calculator = this.allCalculators[this.calculator.code];
    if (calculator) {
        return calculator.calculate(ctx, order, this.calculator.args, this);
    }
}

// 资格检查
async test(ctx: RequestContext, order: Order): Promise<boolean> {
    const checker = this.allCheckers[this.checker.code];
    if (checker) {
        return checker.check(ctx, order, this.checker.args, this);
    }
    return false;
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

### 3. 订单状态推进到支付完成

#### 订单状态机配置

`default-order-process.ts:174-236` 定义了完整的状态转换图：

```
Created → AddingItems → ArrangingPayment → PaymentAuthorized
                                          ↘ PaymentSettled
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

---

### 4. 履约创建流程

#### API 入口

`admin/order.resolver.ts:98-103`:
```typescript
@Mutation()
@Allow(Permission.UpdateOrder)
async addFulfillmentToOrder(@Ctx() ctx, @Args() args) {
    return this.orderService.createFulfillment(ctx, args.input);
}
```

#### OrderService.createFulfillment 完整流程

`order.service.ts:1736-1781`:

```typescript
async createFulfillment(ctx, input) {
    // ========== 验证阶段 ==========
    
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
            handlerCode: fulfillmentHandler.code,
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

#### FulfillmentHandler 示例

`manual-fulfillment-handler.ts:5-24` - 手动履约处理器：
```typescript
export const manualFulfillmentHandler = new FulfillmentHandler({
    code: 'manual-fulfillment',
    args: {
        method: { type: 'string', required: false },
        trackingCode: { type: 'string', required: false },
    },
    createFulfillment: (ctx, orders, orderItems, args) => {
        return {
            method: args.method,
            trackingCode: args.trackingCode,
        };
    },
});
```

---

### 5. 履约状态推进

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
        await result.finalize();
        
        return { fulfillment, orders, fromState, toState: state };
    });
}
```

---

### 6. 订单状态自动联动（最关键的衔接点！）

这是履约状态变化自动推动订单状态变化的核心机制，位于 `default-fulfillment-process.ts:93-125`。

#### onTransitionEnd 钩子

```typescript
async onTransitionEnd(fromState, toState, { ctx, fulfillment, orders }) {
    // ... 库存处理、历史记录 ...
    
    // 关键: 为每个关联订单触发状态联动
    await Promise.all(
        orders.map(order =>
            handleFulfillmentStateTransitByOrder(ctx, order, fulfillment, fromState, toState)
        )
    );
}
```

#### 状态联动逻辑

`default-fulfillment-process.ts:128-162`:
```typescript
async function handleFulfillmentStateTransitByOrder(
    ctx, order, fulfillment, fromState, toState
) {
    // 获取订单当前可转换的下一个状态
    const nextOrderStates = orderService.getNextOrderStates(order);
    
    // 尝试转换订单状态
    const transitionOrderIfStateAvailable = async (state: OrderState) => {
        if (nextOrderStates.includes(state)) {
            const result = await orderService.transitionToState(ctx, order.id, state);
            if (isGraphQlErrorResult(result)) {
                throw new InternalServerError(result.message);
            }
        }
    };
    
    // 履约已发货 → 检查订单发货状态
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
    
    // 履约已送达 → 检查订单送达状态
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

`order-utils.ts:48-53` - 判断是否全部送达：
```typescript
export function orderItemsAreDelivered(order: Order) {
    return (
        getOrderLinesFulfillmentStates(order).every(state => state === 'Delivered') &&
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

`order-utils.ts:128-138` - 判断是否部分履约：
```typescript
function isOrderPartiallyFulfilled(order: Order) {
    const fulfillmentLines = getOrderFulfillmentLines(order);
    const lines = fulfillmentLines.reduce((acc, item) => {
        acc[item.orderLineId] = (acc[item.orderLineId] || 0) + item.quantity;
        return acc;
    }, {} as { [orderLineId: string]: number });
    
    // 只要有一个订单行的履约数量 < 订购数量，就是部分履约
    return order.lines.some(line => line.quantity > lines[line.id]);
}
```

#### 订单状态转换的前置校验

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

### 衔接点 1: 配送方式 → 订单支付
- **位置**: `default-order-process.ts:324-329`
- **逻辑**: `arrangingPaymentRequiresShipping` 检查确保必须先设置配送方式才能进入支付流程
- **意义**: 保证订单在支付前已经确定了配送方式和运费

### 衔接点 2: 订单支付完成 → 履约创建
- **位置**: `order.service.ts:1736-1781`
- **逻辑**: 只有订单处于可履约状态（通常是 PaymentAuthorized/PaymentSettled）后，才能调用 `createFulfillment`
- **验证**: 数量校验、库存校验确保履约的合理性

### 衔接点 3: 履约创建 → 自动转 Pending
- **位置**: `order.service.ts:1776`
- **逻辑**: 履约创建后立即调用 `transitionToState(ctx, fulfillment.id, 'Pending')`
- **意义**: 触发库存扣减（Sale 库存移动）

### 衔接点 4: 履约状态变化 → 订单状态自动联动
- **位置**: `default-fulfillment-process.ts:128-162`
- **逻辑**: 履约状态变为 Shipped/Delivered 时，自动检查整个订单的履约情况，推动订单状态
- **双向校验**: 订单状态转换时也会反向校验履约状态，确保数据一致性

---

## 四、关键数据模型

### ShippingMethod
```
├─ id
├─ code
├─ checker: ConfigurableOperation (关联 ShippingEligibilityChecker)
├─ calculator: ConfigurableOperation (关联 ShippingCalculator)
├─ fulfillmentHandlerCode: string (关联 FulfillmentHandler)
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
├─ handlerCode: string (关联 FulfillmentHandler)
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

## 五、核心文件索引

| 模块 | 文件路径 |
|------|----------|
| 配送方式实体 | `packages/core/src/entity/shipping-method/shipping-method.entity.ts` |
| 配送资格检查器 | `packages/core/src/config/shipping-method/shipping-eligibility-checker.ts` |
| 运费计算器 | `packages/core/src/config/shipping-method/shipping-calculator.ts` |
| 配送计算服务 | `packages/core/src/service/helpers/shipping-calculator/shipping-calculator.ts` |
| 订单修改器 | `packages/core/src/service/helpers/order-modifier/order-modifier.ts` |
| 订单服务 | `packages/core/src/service/services/order.service.ts` |
| 履约服务 | `packages/core/src/service/services/fulfillment.service.ts` |
| 履约处理器 | `packages/core/src/config/fulfillment/fulfillment-handler.ts` |
| 默认订单流程 | `packages/core/src/config/order/default-order-process.ts` |
| 默认履约流程 | `packages/core/src/config/fulfillment/default-fulfillment-process.ts` |
| 订单状态机 | `packages/core/src/service/helpers/order-state-machine/order-state-machine.ts` |
| 履约状态机 | `packages/core/src/service/helpers/fulfillment-state-machine/fulfillment-state-machine.ts` |
| 订单工具函数 | `packages/core/src/service/helpers/utils/order-utils.ts` |
| 管理端订单 API | `packages/core/src/api/resolvers/admin/order.resolver.ts` |
