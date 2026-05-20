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
**文件**: `packages/email-plugin/src/plugin.ts`

#### 初始化流程：

```typescript
// 静态初始化
static init(options: EmailPluginOptions) {
    // 设置模板加载器
    if (options.templateLoader) {
        // 使用自定义模板加载器
    } else if (options.templatePath) {
        // 兼容旧版 templatePath，创建默认 FileBasedTemplateLoader
        options.templateLoader = new FileBasedTemplateLoader(options.templatePath);
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
**文件**: `packages/email-plugin/src/handler/event-handler.ts`

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
    
    // 3. 确定语言代码（支持自定义或使用 ctx 默认值）
    const languageCode = this.setLanguageCodeFn?.(event) || ctx.languageCode;
    
    // 4. 根据渠道和语言匹配最佳配置（如果有 addTemplate 配置）
    const configuration = this.getBestConfiguration(ctx.channel.code, languageCode);
    
    // 5. 确定邮件主题（优先级：配置 > setSubjectFn > defaultSubject）
    const subject = configuration
        ? configuration.subject
        : this.setSubjectFn
          ? await this.setSubjectFn(event, ctx, injector)
          : this.defaultSubject;
    
    // 6. 提取必要信息
    const recipient = this.setRecipientFn(event);
    const templateVars = this.setTemplateVarsFn ? this.setTemplateVarsFn(event, globals) : {};
    
    // 7. 确定 templateFile（优先级：配置 > 默认 'body.hbs'）
    const templateFile = configuration ? configuration.templateFile : 'body.hbs';
    
    // 8. 返回 IntermediateEmailDetails
    return {
        ctx: event.ctx.serialize(),
        type: this.type,
        recipient,
        from: this.from,
        templateVars: { ...globals, ...templateVars },
        subject,
        templateFile,
        attachments,
        ...
    };
}
```

#### 多渠道多语言模板配置命中机制

**EmailTemplateConfig 结构**：
```typescript
// types.ts:369-393
export interface EmailTemplateConfig {
    channelCode: string | 'default';      // 渠道代码或 'default'
    languageCode: LanguageCode | 'default'; // 语言代码或 'default'
    templateFile: string;                  // 模板文件名
    subject: string;                       // 邮件主题
}
```

**getBestConfiguration 匹配算法（真实代码逻辑）**：
```typescript
// event-handler.ts:473-496
private getBestConfiguration(channelCode: string, languageCode: LanguageCode) {
    if (this.configurations.length === 0) {
        return;
    }
    
    // ⚠️  重要：exactMatch 阶段会把 default 渠道一起参与匹配
    // Array.find() 返回第一个满足条件的元素，所以匹配结果受 addTemplate 配置顺序影响！
    const exactMatch = this.configurations.find(c => {
        return (
            // 条件：(渠道匹配 OR 渠道是default) AND 语言完全匹配
            (c.channelCode === channelCode || c.channelCode === 'default') &&
            c.languageCode === languageCode
        );
    });
    if (exactMatch) {
        return exactMatch;
    }
    
    // 第二阶段：渠道完全匹配 + 语言是 default
    const channelMatch = this.configurations.find(
        c => c.channelCode === channelCode && c.languageCode === 'default',
    );
    if (channelMatch) {
        return channelMatch;
    }
    
    // 无匹配，返回 undefined，使用默认配置
    return;
}
```

**关键理解点**：

1. **exactMatch 阶段的匹配逻辑**：
   - 条件表达式：`(c.channelCode === channelCode || c.channelCode === 'default') && c.languageCode === languageCode`
   - 这意味着：只要 languageCode 匹配，无论是 `channelCode=当前渠道` 还是 `channelCode='default'` 都会被纳入候选
   - `Array.find()` 按数组顺序遍历，返回**第一个**满足条件的元素

2. **匹配结果受 addTemplate 配置顺序影响**：
   - `this.configurations` 数组的顺序由 `addTemplate()` 调用顺序决定
   - 先调用 `addTemplate()` 的配置排在数组前面，优先被匹配
   - **没有固定优先级**，完全取决于配置顺序

**⚠️ 陷阱示例**：

假设配置顺序如下：
```typescript
orderConfirmationHandler
    // 先添加 default 渠道的英文配置
    .addTemplate({
        channelCode: 'default',
        languageCode: LanguageCode.en,
        templateFile: 'body.default.en.hbs',
        subject: 'Default: Order #{{ order.code }}',
    })
    // 后添加 my-channel 渠道的英文配置
    .addTemplate({
        channelCode: 'my-channel',
        languageCode: LanguageCode.en,
        templateFile: 'body.my-channel.en.hbs',
        subject: 'MyChannel: Order #{{ order.code }}',
    });
```

当 `channelCode='my-channel', languageCode='en'` 时：
- 遍历第一个配置：`('my-channel' === 'my-channel' || 'default' === 'default') && 'en' === 'en'` → **true**
- 命中第一个配置 `body.default.en.hbs`，而不是预期的 `body.my-channel.en.hbs`！

**正确的配置顺序**（精确渠道配置放前面）：
```typescript
orderConfirmationHandler
    // 先添加具体渠道的配置
    .addTemplate({
        channelCode: 'my-channel',
        languageCode: LanguageCode.en,
        templateFile: 'body.my-channel.en.hbs',
        subject: 'MyChannel: Order #{{ order.code }}',
    })
    // 后添加 default 渠道的兜底配置
    .addTemplate({
        channelCode: 'default',
        languageCode: LanguageCode.en,
        templateFile: 'body.default.en.hbs',
        subject: 'Default: Order #{{ order.code }}',
    });
```

**最小配置验证示例**：

```typescript
import { EmailEventListener, LanguageCode } from '@vendure/email-plugin';
import { OrderStateTransitionEvent } from '@vendure/core';

const testHandler = new EmailEventListener('test-order')
    .on(OrderStateTransitionEvent)
    .filter(event => event.toState === 'PaymentSettled')
    .setRecipient(event => event.order.customer.emailAddress)
    .setFrom('no-reply@example.com')
    // 配置顺序1：default 在前，my-channel 在后
    .addTemplate({
        channelCode: 'default',
        languageCode: LanguageCode.en,
        templateFile: 'body.default.hbs',
        subject: '[DEFAULT] Order #{{ order.code }}',
    })
    .addTemplate({
        channelCode: 'my-channel',
        languageCode: LanguageCode.en,
        templateFile: 'body.my-channel.hbs',
        subject: '[MY-CHANNEL] Order #{{ order.code }}',
    });

// 当 channelCode='my-channel', languageCode='en' 时
// 预期命中：body.my-channel.hbs
// 实际命中：body.default.hbs （因为 default 排在前面！）
```

**匹配流程总结**：
```
获取 channelCode 和 languageCode
    │
    ▼
遍历 configurations 数组（按 addTemplate 顺序）
    │
    ├─▶ 检查 (c.channelCode === channelCode || c.channelCode === 'default') 
    │    && c.languageCode === languageCode
    │
    ├─▶ 第一个满足条件的 → 返回该配置
    │
    └─▶ 都不满足 → 进入第二阶段
            │
            ▼
        检查 c.channelCode === channelCode && c.languageCode === 'default'
            │
            ├─▶ 找到 → 返回该配置
            │
            └─▶ 未找到 → 返回 undefined（使用默认 subject 和 'body.hbs'）
```

#### templateFile 确定流程

```
事件触发
   │
   ▼
获取 ctx.channel.code 和 ctx.languageCode
   │
   ▼
调用 getBestConfiguration(channelCode, languageCode)
   │
   ├─▶ 找到匹配配置 → 使用 configuration.templateFile
   │
   └─▶ 无匹配配置 → 使用默认值 'body.hbs'
   │
   ▼
返回 IntermediateEmailDetails.templateFile
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
    .setTemplateVars(event => ({ order: event.order, shippingLines: event.data.shippingLines }));
```

**多语言模板配置示例（已废弃，推荐使用自定义 TemplateLoader）**：
```typescript
// 旧版方式 - 使用 addTemplate（已废弃）
orderConfirmationHandler
    .addTemplate({
        channelCode: 'default',
        languageCode: LanguageCode.en,
        templateFile: 'body.en.hbs',
        subject: 'Order confirmation for #{{ order.code }}',
    })
    .addTemplate({
        channelCode: 'default',
        languageCode: LanguageCode.zh,
        templateFile: 'body.zh.hbs',
        subject: '订单确认 #{{ order.code }}',
    });
```

---

### 4. EmailProcessor (邮件处理器)
**文件**: `packages/email-plugin/src/email-processor.ts`

#### 核心处理方法 process():

```typescript
async process(data: IntermediateEmailDetails) {
    const ctx = RequestContext.deserialize(data.ctx);
    let emailDetails: EmailDetails = {} as any;
    try {
        // 1. 加载模板（调用 TemplateLoader）
        const bodySource = await this.options.templateLoader.loadTemplate(
            new Injector(this.moduleRef),
            ctx,
            {
                templateName: data.templateFile,  // 来自 IntermediateEmailDetails
                type: data.type,                  // handler type，如 'order-confirmation'
                templateVars: data.templateVars,  // 合并后的模板变量
            },
        );
        
        // 2. 生成邮件内容（调用 EmailGenerator）
        const generated = await this.generator.generate(
            data.from,           // 发件人（可能包含 Handlebars 变量）
            data.subject,        // 主题（可能包含 Handlebars 变量）
            bodySource,          // 模板内容字符串（MJML + Handlebars）
            data.templateVars,   // 模板变量
        );
        
        // 3. 组装最终邮件详情
        emailDetails = {
            ...generated,
            recipient: data.recipient,
            attachments: deserializeAttachments(data.attachments),
            cc: data.cc,
            bcc: data.bcc,
            replyTo: data.replyTo,
        };
        
        // 4. 发送邮件
        const transportSettings = await this.getTransportSettings(ctx);
        await this.emailSender.send(emailDetails, transportSettings);
        
        // 5. 发布 EmailSendEvent（成功）
        await this.eventBus.publish(
            new EmailSendEvent(ctx, emailDetails, true, undefined, data.metadata),
        );
        return true;
    } catch (err: unknown) {
        // 发布 EmailSendEvent（失败）
        await this.eventBus.publish(
            new EmailSendEvent(ctx, emailDetails, false, err as Error, data.metadata),
        );
        throw err;
    }
}
```

---

### 5. TemplateLoader (模板加载器)
**文件**: `packages/email-plugin/src/template-loader/template-loader.ts`

#### 接口定义：

```typescript
export interface TemplateLoader {
    loadTemplate(
        injector: Injector, 
        ctx: RequestContext, 
        input: LoadTemplateInput
    ): Promise<string>;
    
    loadPartials?(): Promise<Partial[]>;
}

// LoadTemplateInput 结构
export interface LoadTemplateInput {
    type: string;              // handler type，如 'order-confirmation'
    templateName: string;      // 模板文件名，如 'body.hbs'
    templateVars: any;         // 模板变量（可用于动态模板逻辑）
}
```

#### FileBasedTemplateLoader 实现：

```typescript
// file-based-template-loader.ts
export class FileBasedTemplateLoader implements TemplateLoader {
    async loadTemplate(_injector, _ctx, { type, templateName }) {
        // 路径拼接: templatePath / type / templateName
        const templatePath = path.join(this.templatePath, type, templateName);
        return fs.readFile(templatePath, 'utf-8');
    }
    
    async loadPartials() {
        // 加载 partials 目录下的所有 .hbs 文件
        const partialsPath = path.join(this.templatePath, 'partials');
        const partialsFiles = await fs.readdir(partialsPath);
        return Promise.all(
            partialsFiles.map(async file => {
                return {
                    name: path.basename(file, '.hbs'),
                    content: await fs.readFile(path.join(partialsPath, file), 'utf-8'),
                };
            }),
        );
    }
}
```

**模板路径规则**:
- 基础路径: `templatePath/[handler-type]/[template-file]
- 例如: `templates/order-confirmation/body.hbs`
- partials 路径: `templates/partials/header.hbs`

**自定义 TemplateLoader 实现多语言**：
```typescript
class CustomLanguageAwareTemplateLoader implements TemplateLoader {
    async loadTemplate(_injector, ctx, { type, templateName }) {
        // 根据 ctx.languageCode 加载对应语言的模板
        const filePath = path.join(
            this.templateDir, 
            type, 
            `${templateName.replace('.hbs', '')}.${ctx.languageCode}.hbs`
        );
        return fs.readFile(filePath, 'utf-8');
    }
}
```

---

### 6. EmailGenerator (邮件生成器)
**文件**: `packages/email-plugin/src/generator/email-generator.ts`

#### 接口定义：

```typescript
interface EmailGenerator {
    onInit?(options: EmailPluginOptions): void | Promise<void>;
    
    generate(
        from: string,                    // 发件人（可能包含 Handlebars 变量）
        subject: string,                 // 主题（可能包含 Handlebars 变量）
        body: string,                    // 模板内容（MJML + Handlebars）
        templateVars: { [key: string]: any },  // 模板变量
    ): EmailGeneratorResult | Promise<EmailGeneratorResult>;
}

// 返回值类型
type EmailGeneratorResult = Pick<EmailDetails, 'from' | 'subject' | 'body'>;
```

#### HandlebarsMjmlGenerator 真实实现：

```typescript
// handlebars-mjml-generator.ts
export class HandlebarsMjmlGenerator implements EmailGenerator {
    async onInit(options: InitializedEmailPluginOptions) {
        // 1. 加载并注册 partials
        if (options.templateLoader.loadPartials) {
            const partials = await options.templateLoader.loadPartials();
            partials.forEach(({ name, content }) => Handlebars.registerPartial(name, content));
        }
        // 2. 注册 helpers
        this.registerHelpers();
    }
    
    generate(from: string, subject: string, template: string, templateVars: any) {
        // 1. 分别编译 from、subject、body 三个模板
        const compiledFrom = Handlebars.compile(from, { noEscape: true });
        const compiledSubject = Handlebars.compile(subject);
        const compiledTemplate = Handlebars.compile(template);
        
        // 2. Handlebars 运行时配置
        const templateOptions: RuntimeOptions = { 
            allowProtoPropertiesByDefault: true  // 允许访问实体原型属性（如 Order.total getter）
        };
        
        // 3. 注入变量生成最终内容
        const fromResult = compiledFrom(templateVars, templateOptions);
        const subjectResult = compiledSubject(templateVars, templateOptions);
        const mjml = compiledTemplate(templateVars, templateOptions);
        
        // 4. MJML 转响应式 HTML
        const body = mjml2html(mjml).html;
        
        // 5. 返回渲染结果
        return { from: fromResult, subject: subjectResult, body };
    }
    
    private registerHelpers() {
        // formatDate: 日期格式化
        Handlebars.registerHelper('formatDate', (date: Date | undefined, format: string | object) => {
            if (!date) return date;
            if (typeof format !== 'string') format = 'default';
            return dateFormat(date, format);
        });
        
        // formatMoney: 金额格式化（支持 Intl.NumberFormat）
        Handlebars.registerHelper(
            'formatMoney',
            (amount?: number, currencyCode?: string, locale?: string) => {
                if (amount == null) return amount;
                // 支持仅传金额、金额+货币、金额+货币+区域三种调用方式
                if (!currencyCode || typeof currencyCode === 'object') {
                    return new Intl.NumberFormat(typeof locale === 'object' ? undefined : locale, {
                        style: 'decimal',
                    }).format(amount / 100);
                }
                return new Intl.NumberFormat(typeof locale === 'object' ? undefined : locale, {
                    style: 'currency',
                    currency: currencyCode,
                }).format(amount / 100);
            },
        );
    }
}
```

**入参说明**:
- `from`: 发件人模板字符串，如 `'{{ fromAddress }}'`
- `subject`: 主题模板字符串，如 `'Order #{{ order.code }} confirmation'`
- `body`: MJML + Handlebars 模板内容
- `templateVars`: 合并后的变量对象 `{ ...globalTemplateVars, ...handlerTemplateVars }`

**返回值说明**:
```typescript
{
    from: string;      // 渲染后的发件人，如 'no-reply@example.com'
    subject: string;   // 渲染后的主题，如 'Order #12345 confirmation'
    body: string;      // 渲染后的完整 HTML 内容
}
```

**内置 Helpers**:
- `formatDate`: 日期格式化（基于 dateformat 库）
- `formatMoney`: 金额格式化（支持 Intl.NumberFormat，自动处理 Vendure 整数金额转小数）

---

### 7. EmailSender (邮件发送器)
**文件**: `packages/email-plugin/src/sender/nodemailer-email-sender.ts`

#### NodemailerEmailSender 实现：

```typescript
export class NodemailerEmailSender implements EmailSender {
    async send(email: EmailDetails, options: EmailTransportOptions) {
        switch (options.type) {
            case 'none':
                return;
            case 'file':
                // 开发模式：输出到文件
                const fileName = normalizeString(
                    `${new Date().toISOString()} ${email.recipient} ${email.subject}`,
                    '_',
                );
                const filePath = path.join(options.outputPath, fileName);
                if (options.raw) {
                    await this.sendFileRaw(email, filePath);
                } else {
                    await this.sendFileJson(email, filePath);
                }
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
1. 业务逻辑发布事件（如 OrderService 触发 OrderStateTransitionEvent）
   │
   ▼
2. EventBus 发布事件
   │
   ▼
3. EmailPlugin.setupEventSubscribers() 订阅的回调被触发
   │
   ▼
4. EmailPlugin.handleEvent(handler, event) 被调用
   │
   ├─▶ 执行 filter 函数过滤（不满足则终止）
   │
   ├─▶ 处理 globalTemplateVars（支持异步函数）
   │
   ▼
5. EmailEventHandler.handle(event, globals, injector)
   │
   ├─▶ 执行 loadData 异步加载数据（如果有）
   │
   ├─▶ 确定 languageCode（setLanguageCodeFn 或 ctx.languageCode）
   │
   ├─▶ getBestConfiguration(channelCode, languageCode) 匹配配置
   │
   ├─▶ 确定 subject（配置 > setSubjectFn > defaultSubject）
   │
   ├─▶ 确定 templateFile（配置.templateFile 或 'body.hbs'）
   │
   ├─▶ 提取 recipient, templateVars, attachments 等
   │
   ▼
6. 生成 IntermediateEmailDetails，添加到 JobQueue（重试5次）
```

### 阶段二：模板加载与邮件生成

```
7. JobQueue 消费任务（异步）
   │
   ▼
8. EmailProcessor.process(data)
   │
   ├─▶ RequestContext.deserialize(data.ctx) 反序列化上下文
   │
   ├─▶ TemplateLoader.loadTemplate(injector, ctx, {
   │        templateName: data.templateFile,
   │        type: data.type,
   │        templateVars: data.templateVars,
   │    })
   │    └─▶ FileBasedTemplateLoader 从文件系统读取模板
   │
   ├─▶ EmailGenerator.generate(from, subject, bodySource, templateVars)
   │    ├─▶ Handlebars.compile 编译 from/subject/body
   │    ├─▶ 注入 templateVars 渲染变量
   │    └─▶ mjml2html 转换为响应式 HTML
   │
   ▼
9. 生成 EmailDetails（from, subject, body, recipient, attachments, ...）
```

### 阶段三：邮件发送

```
10. EmailSender.send(emailDetails, transportSettings)
    │
    ├─▶ 根据 transport type 选择发送方式
    │    ├─▶ smtp: Nodemailer SMTP 发送
    │    ├─▶ sendmail: 本地 sendmail 命令
    │    ├─▶ ses: AWS SES 服务
    │    ├─▶ file: 开发模式输出到文件
    │    └─▶ testing: 调用 onSend 回调
    │
    ▼
11. 发布 EmailSendEvent（成功/失败）
```

---

## 关键数据结构

### IntermediateEmailDetails（中间邮件详情）

```typescript
// types.ts:347-360
// 从 EventHandler 传递到 EmailProcessor 的数据
export type IntermediateEmailDetails = {
    ctx: SerializedRequestContext;      // 序列化的请求上下文
    type: string;                      // 事件类型（handler type）
    recipient: string;                  // 收件人
    from: string;                       // 发件人（可能含模板变量）
    subject: string;                    // 邮件主题（可能含模板变量）
    templateVars: any;                  // 合并后的模板变量
    templateFile: string;               // 模板文件名（如 'body.hbs'）
    attachments: SerializedAttachment[]; // 序列化后的附件
    cc?: string;
    bcc?: string;
    replyTo?: string;
    metadata?: EmailMetadata;           // 元数据
};
```

### EmailDetails（最终邮件详情）

```typescript
// types.ts:288-297
export interface EmailDetails<Type extends 'serialized' | 'unserialized' = 'unserialized'> {
    from: string;
    recipient: string;
    subject: string;
    body: string;                           // 渲染后的 HTML 内容
    attachments: Array<Type extends 'serialized' ? SerializedAttachment : Attachment>;
    cc?: string;
    bcc?: string;
    replyTo?: string;
}
```

### LoadTemplateInput（模板加载输入）

```typescript
// types.ts:402-420
export interface LoadTemplateInput {
    type: string;              // handler type，如 'order-confirmation'
    templateName: string;      // 模板文件名，如 'body.hbs'
    templateVars: any;         // 模板变量（可用于动态加载逻辑）
}
```

---

## 变量注入机制

### 变量来源

1. **globalTemplateVars**（全局变量）
   - 静态对象或异步函数
   - 所有模板共享
   - 例如: `{ fromAddress: 'no-reply@example.com', storeName: 'My Store' }`

2. **handler.setTemplateVars()**（事件级变量）
   - 从事件中提取
   - 例如: `{ order: event.order, verificationToken: 'xxx' }`

3. **handler.loadData()**（异步加载数据）
   - 异步加载的额外数据
   - 存储在 `event.data` 中，然后合并到 templateVars

### 变量合并

```typescript
// 在 EmailEventHandler.handle() 中合并
// globals = globalTemplateVars 处理后的值
// templateVars = setTemplateVarsFn 返回的值
templateVars: { ...globals, ...templateVars }

// 事件级变量会覆盖全局同名变量
```

### Handlebars 模板使用

```handlebars
<!-- 模板中使用变量 -->
<p>Dear {{ order.customer.firstName }},</p>
<p>Thank you for your order #{{ order.code }}!</p>

<!-- 使用 helpers -->
<p>Total: {{ formatMoney order.total order.currencyCode }}</p>
<p>Date: {{ formatDate order.orderPlacedAt 'mmmm d, yyyy' }}</p>

<!-- 使用 partials -->
{{> header title="Order Confirmation" }}
{{> footer }}
```

---

## 扩展点

### 自定义 TemplateLoader

```typescript
class CustomTemplateLoader implements TemplateLoader {
    async loadTemplate(injector, ctx, { type, templateName }) {
        // 从数据库或远程 API 加载模板
        return loadTemplateFromDB(ctx.channel.id, type, templateName, ctx.languageCode);
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

## 模板配置与加载路径示例

假设配置：
```typescript
EmailPlugin.init({
    templateLoader: new FileBasedTemplateLoader(path.join(__dirname, '../static/email/templates')),
    handlers: [orderConfirmationHandler],
});
```

**目录结构**：
```
static/email/templates/
├── order-confirmation/
│   ├── body.hbs              # 默认模板
│   ├── body.en.hbs           # 英文模板（需自定义 TemplateLoader）
│   └── body.zh.hbs           # 中文模板（需自定义 TemplateLoader）
├── email-verification/
│   └── body.hbs
├── password-reset/
│   └── body.hbs
├── email-address-change/
│   └── body.hbs
└── partials/
    ├── header.hbs
    └── footer.hbs
```

**默认加载路径**：
- handler type: `'order-confirmation'`
- templateFile: `'body.hbs'`
- 完整路径: `static/email/templates/order-confirmation/body.hbs`

---

## 总结

Vendure 邮件插件采用了**事件驱动**的架构设计，通过以下核心流程实现邮件发送：

1. **事件订阅**: 通过 EventBus 监听业务事件
2. **Handler 处理**: 每个 EmailEventHandler 定义邮件的收件人、主题、模板变量等
3. **多渠道多语言匹配**: 通过 `getBestConfiguration()` 匹配最佳模板配置（已废弃，推荐自定义 TemplateLoader）
4. **templateFile 确定**: 配置匹配则使用配置的 templateFile，否则使用默认 `'body.hbs'`
5. **模板加载**: TemplateLoader 根据 type 和 templateName 从各种来源加载模板
6. **变量注入**: Handlebars 模板引擎注入 `{ ...globalVars, ...handlerVars }` 合并后的变量
7. **邮件生成**: MJML 编译为响应式 HTML，from/subject/body 分别渲染
8. **异步发送**: 通过 JobQueue 异步发送，支持重试机制
9. **事件反馈**: 通过 EmailSendEvent 通知发送结果

这种设计具有良好的扩展性，可以自定义模板加载器、邮件生成器和邮件发送器。
