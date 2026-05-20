# Vendure 邮件插件与事件模板流转关系

本文从代码角度梳理 Vendure 邮件插件的完整流转过程，包括事件订阅、模板加载、变量注入和邮件发送的全过程。

## 整体架构图

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   EventBus      │────▶│  EmailPlugin    │────▶│  JobQueue       │
│  (发布事件)     │     │  (事件订阅)    │     │  (异步处理)      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                           │
                                                           ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  EmailSender    │◀────│ EmailGenerator  │◀────│  TemplateLoader │
│  (邮件发送)     │     │  (模板渲染)      │     │  (模板加载)      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

## 核心模块说明

### 1. EmailPlugin (插件入口)
**文件**: `packages/email-plugin/src/plugin.ts

#### 初始化流程：

```typescript
// 静态初始化
static init(options: EmailPluginOptions) {
    // 设置模板加载器
    if (options.templateLoader) {
        // 使用自定义模板加载器
    } else if (options.templatePath) {
        // 兼容旧版 templatePath，创建默认 FileBasedTemplateLoader
    }
}
```

**启动阶段 (onApplicationBootstrap)**:
- `initInjectableStrategies()` - 初始化 emailGenerator 和 emailSender
- `setupEventSubscribers()` - 注册所有事件监听器
- 创建 `JobQueue` - 创建名为 "send-email" 的任务队列

```typescript
private async setupEventSubscribers() {
    for (const handler of EmailPlugin.options.handlers) {
        this.eventBus.ofType(handler.event).subscribe(event => {
            return this.handleEvent(handler, event);
        });
    }
}
```

---

### 2. 事件订阅与处理

#### 事件监听配置：

```typescript
// plugin.ts:397-403
private async setupEventSubscribers() {
    for (const handler of EmailPlugin.options.handlers) {
        this.eventBus.ofType(handler.event).subscribe(event => {
            return this.handleEvent(handler, event);
        });
    }
}
```

#### 事件处理流程 (handleEvent):

```typescript
// plugin.ts:405-432
private async handleEvent(handler, event) {
    // 1. 处理全局模板变量（支持异步函数）
    let globalTemplateVars = this.options.globalTemplateVars;
    if (typeof globalTemplateVars === 'function') {
        globalTemplateVars = await globalTemplateVars(event.ctx, injector);
    }
    
    // 2. 调用 handler.handle() 生成邮件详情
    const result = await handler.handle(event, globalTemplateVars, injector);
    
    // 3. 添加到 JobQueue（重试5次）
    if (this.jobQueue) {
        await this.jobQueue.add(result, { retries: 5 });
    }
}
```

---

### 3. EmailEventHandler (事件处理器)
**文件**: `packages/email-plugin/src/handler/event-handler.ts

#### 核心处理方法 handle():

```typescript
async handle(event, globals, injector): Promise<IntermediateEmailDetails | undefined> {
    // 1. 执行所有 filter 函数
    for (const filterFn of this.filterFns) {
        if (!filterFn(event)) {
            return; // 不满足过滤条件，跳过
        }
    }
    
    // 2. 如果有 loadData，异步加载额外数据
    if (this instanceof EmailEventHandlerWithAsyncData) {
        (event as any).data = await this._loadDataFn({ event, injector });
    }
    
    // 3. 提取必要信息
    const recipient = this.setRecipientFn(event);
    const templateVars = this.setTemplateVarsFn ? this.setTemplateVarsFn(event, globals) : {};
    
    // 4. 返回 IntermediateEmailDetails
    return {
        ctx: event.ctx.serialize(),
        type: this.type,
        recipient,
        from: this.from,
        templateVars: { ...globals, ...templateVars },
        subject,
        templateFile: 'body.hbs',
        attachments,
        ...
    };
}
```

#### EmailEventHandler 链式调用示例（默认处理器）:

```typescript
// default-email-handlers.ts
export const orderConfirmationHandler = new EmailEventListener('order-confirmation')
    .on(OrderStateTransitionEvent)
    .filter(event => event.toState === 'PaymentSettled')
    .loadData(async ({ event, injector }) => {
        // 异步加载数据
        const entityHydrator = injector.get(EntityHydrator);
        await entityHydrator.hydrate(event.ctx, event.order, {
            relations: ['lines.featuredAsset', 'shippingLines.shippingMethod'],
        });
        return { shippingLines };
    })
    .setRecipient(event => event.order.customer.emailAddress)
    .setFrom('{{ fromAddress }}')
    .setSubject('Order confirmation for #{{ order.code }}')
    .setTemplateVars(event => ({ order: event.order, shippingLines: event.data.shippingLines });
```

---

### 4. EmailProcessor (邮件处理器)
**文件**: `packages/email-plugin/src/email-processor.ts

#### 核心处理方法 process():

```typescript
async process(data: IntermediateEmailDetails) {
    const ctx = RequestContext.deserialize(data.ctx);
    
    // 1. 加载模板
    const bodySource = await this.options.templateLoader.loadTemplate(
        new Injector(this.moduleRef),
        ctx,
        {
            templateName: data.templateFile,
            type: data.type,
            templateVars: data.templateVars,
        },
    );
    
    // 2. 生成邮件内容
    const generated = await this.generator.generate(
        data.from,
        data.subject,
        bodySource,
        bodySource,
        data.templateVars,
    );
    
    // 3. 发送邮件
    const emailDetails = { ...generated, recipient: data.recipient, ... };
    await this.emailSender.send(emailDetails, transportSettings);
    
    // 4. 发布 EmailSendEvent
    await this.eventBus.publish(new EmailSendEvent(ctx, emailDetails, true, undefined, data.metadata));
}
```

---

### 5. TemplateLoader (模板加载器)
**文件**: `packages/email-plugin/src/template-loader/template-loader.ts`

#### 接口定义：

```typescript
export interface TemplateLoader {
    loadTemplate(injector: Injector, ctx: RequestContext, input: LoadTemplateInput): Promise<string>;
    loadPartials?(): Promise<Partial[]>;
}
```

#### FileBasedTemplateLoader 实现：

```typescript
// file-based-template-loader.ts
export class FileBasedTemplateLoader implements TemplateLoader {
    async loadTemplate(_injector, _ctx, { type, templateName }) {
        const templatePath = path.join(this.templatePath, type, templateName);
        return fs.readFile(templatePath, 'utf-8');
    }
    
    async loadPartials() {
        // 加载 partials 目录下的所有 .hbs 文件
        const partialsPath = path.join(this.templatePath, 'partials');
        // 注册为 Handlebars partials
    }
}
```

**模板路径规则**:
- 基础路径: `templatePath/[handler-type/[template-file]
- 例如: `templates/order-confirmation/body.hbs`

---

### 6. EmailGenerator (邮件生成器)
**文件**: `packages/email-plugin/src/generator/handlebars-mjml-generator.ts

#### HandlebarsMjmlGenerator 实现：

```typescript
export class HandlebarsMjmlGenerator implements EmailGenerator {
    async onInit(options) {
        // 1. 加载并注册 partials
        if (options.templateLoader.loadPartials) {
            const partials = await options.templateLoader.loadPartials();
            partials.forEach(({ name, content }) => Handlebars.registerPartial(name, content));
        }
        // 2. 注册 helpers
        this.registerHelpers();
    }
    
    generate(from, subject, template, templateVars) {
        // 1. 编译 Handlebars 模板
        const compiledFrom = Handlebars.compile(from, { noEscape: true });
        const compiledSubject = Handlebars.compile(subject);
        const compiledTemplate = Handlebars.compile(template);
        
        // 2. 注入变量生成 MJML
        const templateOptions = { allowProtoPropertiesByDefault: true };
        const fromResult = compiledFrom(templateVars, templateOptions);
        const subjectResult = compiledSubject(templateVars, templateOptions);
        const mjml = compiledTemplate(templateVars, templateOptions);
        
        // 3. MJML 转 HTML
        const body = mjml2html(mjml).html;
        
        return { from: fromResult, subject: subjectResult, body };
    }
    
    private registerHelpers() {
        Handlebars.registerHelper('formatDate', ...);
        Handlebars.registerHelper('formatMoney', ...);
    }
}
```

**内置 Helpers**:
- `formatDate`: 日期格式化
- `formatMoney`: 金额格式化（支持货币格式化（支持 Intl.NumberFormat）

---

### 7. EmailSender (邮件发送器)
**文件**: `packages/email-plugin/src/sender/nodemailer-email-sender.ts

#### NodemailerEmailSender 实现：

```typescript
export class NodemailerEmailSender implements EmailSender {
    async send(email: EmailDetails, options: EmailTransportOptions) {
        switch (options.type) {
            case 'none':
                return;
            case 'file':
                // 开发模式：输出到文件
                await this.sendFileJson(email, filePath);
                break;
            case 'sendmail':
                await this.sendMail(email, this.getSendMailTransport(options));
                break;
            case 'ses':
                await this.sendMail(email, this.getSesTransport(options));
                break;
            case 'smtp':
                await this.sendMail(email, this.getSmtpTransport(options));
                break;
            case 'testing':
                options.onSend(email);
                break;
        }
    }
    
    private async sendMail(email, transporter) {
        return transporter.sendMail({
            from: email.from,
            to: email.recipient,
            subject: email.subject,
            html: email.body,
            attachments: email.attachments,
            cc: email.cc,
            bcc: email.bcc,
            replyTo: email.replyTo,
        });
    }
}
```

---

## 完整流转时序

### 阶段一：事件发布与订阅

```
1. 业务逻辑发布事件
   │
   ▼
2. EventBus 发布事件（如 OrderStateTransitionEvent）
   │
   ▼
3. EmailPlugin.setupEventSubscribers() 订阅事件
   │
   ▼
4. EmailPlugin.handleEvent(handler, event) 被调用
   │
   ├─▶ 执行 filter 函数过滤
   │
   ├─▶ 处理 globalTemplateVars（支持异步）
   │
   ▼
5. EmailEventHandler.handle(event, globals, injector)
   │
   ├─▶ 执行 loadData 异步加载数据（如果有）
   │
   ├─▶ 提取 recipient, subject, templateVars
   │
   ▼
6. 生成 IntermediateEmailDetails，添加到 JobQueue
```

### 阶段二：模板加载与邮件生成

```
7. JobQueue 消费任务
   │
   ▼
8. EmailProcessor.process(data)
   │
   ├─▶ TemplateLoader.loadTemplate()
   │    └─▶ 从文件系统加载模板文件
   │
   ├─▶ EmailGenerator.generate()
   │    ├─▶ 编译 Handlebars 模板
   │    ├─▶ 注入 templateVars 变量
   │    └─▶ MJML 转 HTML
   │
   ▼
9. 生成 EmailDetails（from, subject, body）
```

### 阶段三：邮件发送

```
10. EmailSender.send(emailDetails, transportSettings)
    │
    ├─▶ 根据 transport type 选择发送方式
    │
    ├─▶ SMTP/Sendmail/SES/File
    │
    ▼
11. 发布 EmailSendEvent（成功/失败）
```

---

## 关键数据结构

### IntermediateEmailDetails（中间邮件详情）

```typescript
// 从 EventHandler 传递到 EmailProcessor 的数据
{
    ctx: SerializedRequestContext;  // 序列化的请求上下文
    type: string;                  // 事件类型（handler type）
    recipient: string;              // 收件人
    from: string;                 // 发件人
    subject: string;              // 邮件主题
    templateVars: { [key: string]: any };  // 模板变量
    templateFile: string;           // 模板文件名（如 'body.hbs'）
    attachments: SerializedAttachment[];  // 附件
    cc?: string;
    bcc?: string;
    replyTo?: string;
    metadata?: { [key: string]: any };  // 元数据
}
```

### EmailDetails（最终邮件详情）

```typescript
{
    from: string;
    recipient: string;
    subject: string;
    body: string;         // 渲染后的 HTML 内容
    attachments: Attachment[];
    cc?: string;
    bcc?: string;
    replyTo?: string;
}
```

---

## 变量注入机制

### 变量来源

1. **globalTemplateVars**（全局变量）
   - 静态对象或异步函数
   - 所有模板共享
   - 例如: `{ fromAddress: 'no-reply@example.com' }`

2. **handler.setTemplateVars()**（事件级变量）
   - 从事件中提取
   - 例如: `{ order: event.order }`

3. **handler.loadData()**（异步加载数据）
   - 异步加载的额外数据
   - 存储在 `event.data` 中

### 变量合并

```typescript
// 在 EmailEventHandler.handle() 中合并
templateVars: { ...globals, ...templateVars }
```

### Handlebars 模板使用

```handlebars
<!-- 模板中使用变量 -->
<p>Dear {{ order.customer.firstName }},</p>
<p>Thank you for your order #{{ order.code }}!</p>

<!-- 使用 helpers -->
<p>Total: {{ formatMoney order.total }}</p>
<p>Date: {{ formatDate order.orderPlacedAt }}</p>
```

---

## 扩展点

### 自定义 TemplateLoader

```typescript
class CustomTemplateLoader implements TemplateLoader {
    async loadTemplate(injector, ctx, { type, templateName }) {
        // 从数据库或远程 API 加载模板
        return loadTemplateFromDB(ctx.channel.id, type, templateName);
    }
}
```

### 自定义 EmailGenerator

```typescript
class CustomEmailGenerator implements EmailGenerator {
    generate(from, subject, template, templateVars) {
        // 使用其他模板引擎（如 Pug, EJS 等）
        return renderCustomTemplate(template, templateVars);
    }
}
```

### 自定义 EmailSender

```typescript
class CustomEmailSender implements EmailSender {
    async send(email, options) {
        // 使用其他邮件服务（如 SendGrid, Mailgun 等）
        return sendViaSendGrid(email);
    }
}
```

---

## 默认邮件处理器列表

| 处理器名称 | 监听事件 | 触发条件 |
|---------|---------|---------|
| orderConfirmationHandler | OrderStateTransitionEvent | 订单状态变为 PaymentSettled |
| emailVerificationHandler | AccountRegistrationEvent | 用户注册 |
| passwordResetHandler | PasswordResetEvent | 请求密码重置 |
| emailAddressChangeHandler | IdentifierChangeRequestEvent | 请求修改邮箱 |

---

## 总结

Vendure 邮件插件采用了**事件驱动**的架构设计，通过以下核心流程实现邮件发送：

1. **事件订阅**: 通过 EventBus 监听业务事件
2. **Handler 处理**: 每个 EmailEventHandler 定义邮件的收件人、主题、模板变量等
3. **模板加载**: TemplateLoader 负责从各种来源加载模板
4. **变量注入**: Handlebars 模板引擎注入动态数据
5. **邮件生成**: MJML 编译为响应式 HTML
6. **异步发送**: 通过 JobQueue 异步发送，支持重试机制
7. **事件反馈**: 通过 EmailSendEvent 通知发送结果

这种设计具有良好的扩展性，可以自定义模板加载器、邮件生成器和邮件发送器。
