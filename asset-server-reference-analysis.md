# Vendure 资源服务协作分析

## 概述

Vendure 中的资源（Asset）系统涉及三个核心组件的协同工作：

1. **资源服务（AssetService）** - 位于 `@vendure/core`，负责资产生命周期管理
2. **媒体处理插件（AssetServerPlugin）** - 负责文件存储、预览图生成和实时图像转换
3. **业务实体（Product / ProductVariant / Collection）** - 通过引用关系使用资源

本文档分析三者之间的协作机制，所有结论均有对应代码证据支撑。

---

## 一、核心数据模型

### 1.1 Asset 实体 (`packages/core/src/entity/asset/asset.entity.ts`)

```typescript
@Entity()
export class Asset extends VendureEntity {
    @Column('varchar') type: AssetType;
    @Column() mimeType: string;
    @Column() fileSize: number;
    @Column() source: string;           // 源文件存储标识（路径/URL）
    @Column() preview: string;          // 预览文件存储标识
    @Column() width: number;
    @Column() height: number;
    
    // 反向引用 - 哪些实体将此资源作为特色图
    @OneToMany(type => Collection, c => c.featuredAsset)
    featuredInCollections?: Collection[];
    @OneToMany(type => ProductVariant, v => v.featuredAsset)
    featuredInVariants?: ProductVariant[];
    @OneToMany(type => Product, p => p.featuredAsset)
    featuredInProducts?: Product[];
}
```

**关键点**：
- `source` 和 `preview` 字段存储文件标识符（相对路径或 S3 Key）
- **没有显式的引用计数字段**，引用检查通过查询关联表动态计算
- 反向引用仅用于查询 `featuredAsset` 关联，不包含 OrderableAsset 列表引用

### 1.2 OrderableAsset 关联实体 (`packages/core/src/entity/asset/orderable-asset.entity.ts`)

```typescript
export abstract class OrderableAsset extends VendureEntity {
    @Column() assetId: ID;
    
    @Index()
    @ManyToOne(type => Asset, { eager: true, onDelete: 'CASCADE' })
    asset: Asset;
    
    @Column() position: number;
}
```

**具体实现**：
- `ProductAsset` - 产品与资源的多对多关联（带排序）
- `ProductVariantAsset` - 产品变体与资源的多对多关联
- `CollectionAsset` - 集合与资源的多对多关联

### 1.3 业务实体引用方式

以 Product 为例 (`packages/core/src/entity/product/product.entity.ts`)：

```typescript
@Entity()
export class Product {
    // 特色图：单个引用，删除资源时设为 NULL
    @Index()
    @ManyToOne(type => Asset, asset => asset.featuredInProducts, { onDelete: 'SET NULL' })
    featuredAsset: Asset;
    
    // 资源列表：通过 OrderableAsset 关联表
    @OneToMany(type => ProductAsset, productAsset => productAsset.product)
    assets: ProductAsset[];
}
```

**级联策略总结**：

| 关联关系 | 实体 | onDelete 策略 | 说明 |
|---------|------|--------------|------|
| OrderableAsset.asset | Asset | CASCADE | 删除资源时，关联表记录自动删除 |
| Product.featuredAsset | Asset | SET NULL | 删除资源时，特色图字段置空 |
| ProductVariant.featuredAsset | Asset | SET NULL | 同上 |
| Collection.featuredAsset | Asset | SET NULL | 同上 |
| ProductAsset.product | Product | CASCADE | 删除产品时，关联记录自动删除 |

---

## 二、资源上传流程与失败分支

### 2.1 上传入口 (`packages/core/src/service/services/asset.service.ts:307`)

```typescript
async create(ctx: RequestContext, input: CreateAssetInput): Promise<Asset> {
    const { createReadStream, filename, mimetype } = await input.file;
    const { stream, errorPromise } = this.makeStreamGuard(createReadStream);
    const result = await Promise.race([
        this.createAssetInternal(ctx, stream, filename, mimetype, input.customFields, input.translations),
        errorPromise,
    ]);
    // ... 后续处理
}
```

### 2.2 核心上传逻辑 (`createAssetInternal`)

`packages/core/src/service/services/asset.service.ts:565`

```typescript
private async createAssetInternal(
    ctx: RequestContext,
    stream: Stream,
    filename: string,
    mimetype: string,
): Promise<Asset> {
    const { assetPreviewStrategy, assetStorageStrategy } = this.configService.assetOptions;
    
    // 步骤 1: 生成文件名（已包含 source/ 前缀和哈希目录）
    const sourceFileName = await this.getSourceFileName(ctx, filename);
    const previewFileName = await this.getPreviewFileName(ctx, sourceFileName);
    
    // 步骤 2: 写入源文件 ← 失败点 1
    const sourceFileIdentifier = await assetStorageStrategy.writeFileFromStream(
        sourceFileName, stream
    );
    
    // 步骤 3: 读取源文件到 Buffer ← 失败点 2
    const sourceFile = await assetStorageStrategy.readFileToBuffer(sourceFileIdentifier);
    
    // 步骤 4: 生成预览图 ← 失败点 3
    let preview: Buffer;
    try {
        preview = await assetPreviewStrategy.generatePreviewImage(ctx, mimetype, sourceFile);
    } catch (e: any) {
        Logger.error(`Could not create Asset preview image: ${message}`, undefined, e.stack);
        throw e;  // 直接抛出，不清理已写入的源文件！
    }
    
    // 步骤 5: 写入预览文件 ← 失败点 4
    const previewFileIdentifier = await assetStorageStrategy.writeFileFromBuffer(
        previewFileName, preview
    );
    
    // 步骤 6: 获取图片尺寸 ← 失败点 5 (imageSize 可能抛错)
    const { width, height } = this.getDimensions(type === AssetType.IMAGE ? sourceFile : preview);
    
    // 步骤 7: 创建并保存 Asset 实体 ← 失败点 6 (数据库操作)
    const asset = new Asset({
        type, width, height, fileSize: sourceFile.byteLength, mimeType,
        source: sourceFileIdentifier,
        preview: previewFileIdentifier,
    });
    await this.channelService.assignToCurrentChannel(asset, ctx);
    return this.connection.getRepository(ctx, Asset).save(asset);
}
```

### 2.3 文件名生成链

**HashedAssetNamingStrategy** (`packages/asset-server-plugin/src/config/hashed-asset-naming-strategy.ts`)：

```typescript
export class HashedAssetNamingStrategy extends DefaultAssetNamingStrategy {
    generateSourceFileName(ctx, originalFileName, conflictFileName?) {
        const filename = super.generateSourceFileName(...);
        return path.join('source', this.getHashedDir(filename), filename);
    }
    
    generatePreviewFileName(ctx, sourceFileName, conflictFileName?) {
        const filename = super.generatePreviewFileName(...);
        return path.join('preview', this.getHashedDir(filename), filename);
    }
    
    private getHashedDir(filename: string): string {
        return createHash('md5').update(filename).digest('hex').slice(0, 2);
    }
}
```

**生成结果示例**：
- 源文件：`source/0a/product-image__01.jpg`
- 预览文件：`preview/0a/product-image__01__preview.jpg`

> **代码证据**：`createAssetInternal` 中 `getSourceFileName()` 调用 `assetNamingStrategy.generateSourceFileName()`，该方法返回的文件名已经包含 `source/` 前缀和哈希目录。这个完整路径会直接传递给 `assetStorageStrategy.writeFileFromStream()`。

### 2.4 失败分支与孤儿文件分析

**孤儿文件定义**：文件已写入存储，但对应的 Asset 实体未成功创建，导致文件无法被引用和清理。

| 失败步骤 | 失败原因 | 已写入的文件 | 是否产生孤儿 | 当前补偿机制 |
|---------|---------|-------------|-------------|-------------|
| 步骤 2: 写入源文件 | 磁盘满、权限错误、网络中断(S3) | 无 | 否 | - |
| 步骤 3: 读取源文件 | 文件损坏、磁盘错误 | source 文件 | **是** | ❌ 无补偿，源文件残留 |
| 步骤 4: 生成预览图 | Sharp 处理失败、格式不支持 | source 文件 | **是** | ❌ 无补偿，源文件残留 |
| 步骤 5: 写入预览文件 | 磁盘满、权限错误 | source 文件 | **是** | ❌ 无补偿，源文件残留 |
| 步骤 6: 获取尺寸 | imageSize 解析失败 | source + preview 文件 | **是** | ❌ 无补偿，两文件都残留 |
| 步骤 7: 保存数据库 | 数据库事务失败、连接中断 | source + preview 文件 | **是** | ❌ 无补偿，两文件都残留 |

**关键问题**：`createAssetInternal` 中没有任何 `try-catch` 或 `finally` 块来清理失败前已写入的文件。一旦在步骤 3-7 中任何一步失败，已写入的源文件和/或预览文件将成为**孤儿文件**。

**孤儿文件特征**：
- 路径符合命名规则（如 `source/0a/xxx.jpg`）
- 数据库中无对应 Asset 记录
- 无法通过正常删除流程清理
- 只能通过扫描文件系统并比对数据库来发现和清理

---

## 三、存储策略的 identifier 生成路径对比

### 3.1 LocalAssetStorageStrategy (`packages/asset-server-plugin/src/config/local-asset-storage-strategy.ts`)

```typescript
export class LocalAssetStorageStrategy implements AssetStorageStrategy {
    constructor(
        private readonly uploadPath: string,
        private readonly toAbsoluteUrlFn?: (reqest: Request, identifier: string) => string,
    ) {
        fs.ensureDirSync(this.uploadPath);
        if (toAbsoluteUrlFn) {
            this.toAbsoluteUrl = toAbsoluteUrlFn;
        }
    }
    
    async writeFileFromStream(fileName: string, data: ReadStream): Promise<string> {
        // 拼接完整路径
        const filePath = path.join(this.uploadPath, fileName);
        await fs.ensureDir(path.dirname(filePath));
        const writeStream = fs.createWriteStream(filePath, 'binary');
        return new Promise<string>((resolve, reject) => {
            data.pipe(writeStream);
            writeStream.on('close', () => resolve(this.filePathToIdentifier(filePath)));
            writeStream.on('error', reject);
        });
    }
    
    async writeFileFromBuffer(fileName: string, data: Buffer): Promise<string> {
        const filePath = path.join(this.uploadPath, fileName);
        await fs.ensureDir(path.dirname(filePath));
        await fs.writeFile(filePath, data, 'binary');
        return this.filePathToIdentifier(filePath);
    }
    
    // Local 特有：将完整文件路径转换为相对路径 identifier
    private filePathToIdentifier(filePath: string): string {
        const filePathDirname = path.dirname(filePath);
        const deltaDirname = filePathDirname.replace(this.uploadPath, '');
        const identifier = path.join(deltaDirname, path.basename(filePath));
        return identifier.replace(/^[\\/]+/, '');
    }
    
    private identifierToFilePath(identifier: string): string {
        return path.join(this.uploadPath, identifier);
    }
}
```

### 3.2 S3AssetStorageStrategy (`packages/asset-server-plugin/src/config/s3-asset-storage-strategy.ts`)

```typescript
export class S3AssetStorageStrategy implements AssetStorageStrategy {
    async writeFileFromBuffer(fileName: string, data: Buffer) {
        return this.writeFile(fileName, data);
    }
    
    async writeFileFromStream(fileName: string, data: Readable) {
        return this.writeFile(fileName, data);
    }
    
    private async writeFile(fileName: string, data: ...) {
        const { Upload } = this.libStorage;
        const upload = new Upload({
            client: this.s3Client,
            params: {
                Bucket: this.s3Config.bucket,
                Key: fileName,  // 直接使用传入的 fileName 作为 S3 Key
                Body: data,
            },
        });
        return upload.done().then(result => {
            return result.Key;  // 直接返回 S3 Key，无需转换
        });
    }
    
    private getObjectParams(identifier: string) {
        return {
            Bucket: this.s3Config.bucket,
            Key: path.join(identifier.replace(/^\//, '')),
        };
    }
}
```

### 3.3 两种策略的 identifier 生成对比

| 对比项 | LocalAssetStorageStrategy | S3AssetStorageStrategy |
|--------|--------------------------|------------------------|
| 输入 fileName | `source/0a/photo.jpg` | `source/0a/photo.jpg` |
| 内部处理 | 拼接 `uploadPath` 写入文件系统 | 直接作为 S3 Key 上传 |
| identifier 生成 | `filePathToIdentifier()` 去除 `uploadPath` 前缀 | 直接返回 S3 返回的 `result.Key` |
| 输出 identifier | `source/0a/photo.jpg` | `source/0a/photo.jpg` |
| 特有逻辑 | `filePathToIdentifier` / `identifierToFilePath` 路径转换 | 无，直接使用 Key |

**代码证据**：
- Local 策略：`writeFileFromStream` → `path.join(this.uploadPath, fileName)` → 写入 → `filePathToIdentifier(filePath)` 返回相对路径
- S3 策略：`writeFile` → `Key: fileName` → 上传 → `result.Key` 返回原始 fileName

**结论**：
- **最终 identifier 格式完全相同**：都是 `source/{hash}/{filename}` 格式
- **生成路径不同**：
  - Local 策略需要 `filePathToIdentifier` 在完整路径和相对路径间转换
  - S3 策略直接使用传入的 fileName 作为 Key，无需路径转换
- **读取时的差异**：
  - Local：`identifierToFilePath(identifier)` 拼接完整路径后 `fs.readFile()`
  - S3：直接将 identifier 作为 Key 调用 `GetObjectCommand`

### 3.4 存储目录结构

```
assets/ (uploadPath)
├── source/
│   ├── 0a/
│   │   └── product-image__01.jpg
│   └── 1b/
│       └── document.pdf
├── preview/
│   ├── 0a/
│   │   └── product-image__01__preview.jpg
│   └── 1b/
│       └── document__preview.png
└── cache/
    └── 0a/
        └── product-image__01_transform_w500_h300_mcrop_<hash>.jpg
```

---

## 四、标识符到 URL 的转换 (toAbsoluteUrl)

### 4.1 转换机制概述

Asset 实体中存储的 `source` 和 `preview` 是**内部标识符**，不能直接对外暴露。需要通过 `AssetStorageStrategy.toAbsoluteUrl()` 方法转换为可访问的 URL。

**转换时机**：
- GraphQL 响应输出时，通过 `AssetInterceptorPlugin` 自动转换
- 邮件模板等场景手动调用

### 4.2 AssetInterceptorPlugin (`packages/core/src/api/middleware/asset-interceptor-plugin.ts`)

```typescript
export class AssetInterceptorPlugin implements ApolloServerPlugin {
    private readonly toAbsoluteUrl: AssetStorageStrategy['toAbsoluteUrl'] | undefined;

    constructor(private configService: ConfigService) {
        const { assetOptions } = this.configService;
        if (assetOptions.assetStorageStrategy.toAbsoluteUrl) {
            this.toAbsoluteUrl = assetOptions.assetStorageStrategy.toAbsoluteUrl.bind(
                assetOptions.assetStorageStrategy,
            );
        }
    }
    
    async requestDidStart(): Promise<GraphQLRequestListener<any>> {
        return {
            willSendResponse: async requestContext => {
                const { document } = requestContext;
                if (document) {
                    const { body } = requestContext.response;
                    const req = requestContext.contextValue.req;
                    if (body.kind === 'single') {
                        this.prefixAssetUrls(req, document, body.singleResult.data);
                    }
                }
            },
        };
    }
    
    private prefixAssetUrls(request: any, document: DocumentNode, data?: Record<string, unknown> | null) {
        const typeTree = this.graphqlValueTransformer.getOutputTypeTree(document);
        const toAbsoluteUrl = this.toAbsoluteUrl;
        if (!toAbsoluteUrl || !data) {
            return;
        }
        this.graphqlValueTransformer.transformValues(typeTree, data, (value, type) => {
            if (!type) {
                return value;
            }
            const isAssetType = this.isAssetType(type);
            const isUnionWithAssetType = isUnionType(type) && type.getTypes().find(t => this.isAssetType(t));
            if (isAssetType || isUnionWithAssetType) {
                if (value && !Array.isArray(value)) {
                    if (value.preview) {
                        value.preview = toAbsoluteUrl(request, value.preview);
                    }
                    if (value.source) {
                        value.source = toAbsoluteUrl(request, value.source);
                    }
                }
            }
            return value;
        });
    }
}
```

### 4.3 URL 前缀生成 (`packages/asset-server-plugin/src/common.ts`)

```typescript
export function getAssetUrlPrefixFn(options: AssetServerOptions) {
    const { assetUrlPrefix, route } = options;
    if (assetUrlPrefix == null) {
        // 自动推断：从请求头获取协议和 host
        return (request: Request, identifier: string) => {
            const protocol = request.headers['x-forwarded-proto'] ?? request.protocol;
            return `${Array.isArray(protocol) ? protocol[0] : protocol}://${
                request.get('host') ?? 'could-not-determine-host'
            }/${route}/`;
        };
    }
    if (typeof assetUrlPrefix === 'string') {
        // 静态前缀：直接返回配置的字符串
        return (...args: any[]) => assetUrlPrefix;
    }
    if (typeof assetUrlPrefix === 'function') {
        // 自定义函数：传入 RequestContext 和 identifier
        return (request: Request, identifier: string) => {
            const ctx = internal_getRequestContext(request);
            return assetUrlPrefix(ctx, identifier);
        };
    }
    throw new Error(`The assetUrlPrefix option was of an unexpected type`);
}
```

### 4.4 Local 与 S3 策略的 toAbsoluteUrl 实现

**Local 策略工厂** (`packages/asset-server-plugin/src/config/default-asset-storage-strategy-factory.ts`)：
```typescript
export function defaultAssetStorageStrategyFactory(options: AssetServerOptions) {
    const prefixFn = getAssetUrlPrefixFn(options);
    const toAbsoluteUrlFn = (request: Request, identifier: string): string => {
        if (!identifier) {
            return '';
        }
        const prefix = prefixFn(request, identifier);
        return identifier.startsWith(prefix) ? identifier : `${prefix}${identifier}`;
    };
    return new LocalAssetStorageStrategy(assetUploadDir, toAbsoluteUrlFn);
}
```

**S3 策略工厂** (`packages/asset-server-plugin/src/config/s3-asset-storage-strategy.ts`)：
```typescript
export function configureS3AssetStorage(s3Config: S3Config) {
    return (options: AssetServerOptions) => {
        const prefixFn = getAssetUrlPrefixFn(options);
        const toAbsoluteUrlFn = (request: Request, identifier: string): string => {
            if (!identifier) {
                return '';
            }
            const prefix = prefixFn(request, identifier);
            return identifier.startsWith(prefix) ? identifier : `${prefix}${identifier}`;
        };
        return new S3AssetStorageStrategy(s3Config, toAbsoluteUrlFn);
    };
}
```

**关键发现**：Local 和 S3 策略的 `toAbsoluteUrl` 实现**完全相同**！

| 对比项 | Local 策略 | S3 策略 |
|--------|-----------|---------|
| identifier 格式 | `source/0a/photo.jpg` | `source/0a/photo.jpg` |
| toAbsoluteUrl 逻辑 | `{prefix}{identifier}` | `{prefix}{identifier}` |
| prefix 配置示例 | `https://cdn.example.com/assets/` | `https://my-bucket.s3.amazonaws.com/` |
| 最终 URL | `https://cdn.example.com/assets/source/0a/photo.jpg` | `https://my-bucket.s3.amazonaws.com/source/0a/photo.jpg` |

---

## 五、引用计数与删除机制

### 5.1 引用检查 (`AssetService.findAssetUsages`)

`packages/core/src/service/services/asset.service.ts:755`

```typescript
private async findAssetUsages(
    ctx: RequestContext,
    asset: Asset,
): Promise<{ products: Product[]; variants: ProductVariant[]; collections: Collection[] }> {
    const products = await this.connection.getRepository(ctx, Product).find({
        where: { featuredAsset: { id: asset.id }, deletedAt: IsNull() },
    });
    const variants = await this.connection.getRepository(ctx, ProductVariant).find({
        where: { featuredAsset: { id: asset.id }, deletedAt: IsNull() },
    });
    const collections = await this.connection.getRepository(ctx, Collection).find({
        where: { featuredAsset: { id: asset.id } },
    });
    return { products, variants, collections };
}
```

> **代码证据**：`findAssetUsages` 只查询了 `featuredAsset` 字段，**完全没有查询 OrderableAsset 关联表**（ProductAsset、ProductVariantAsset、CollectionAsset）。

### 5.2 删除流程 (`AssetService.delete`)

`packages/core/src/service/services/asset.service.ts:375`

```typescript
async delete(ctx, ids, force = false, deleteFromAllChannels = false): Promise<DeletionResponse> {
    const assets = await this.connection.findByIdsInChannel(ctx, Asset, ids, ctx.channelId, {
        relations: ['channels'],
    });
    
    // 1. 统计引用（仅 featuredAsset）
    const usageCount = { products: 0, variants: 0, collections: 0 };
    for (const asset of assets) {
        const usages = await this.findAssetUsages(ctx, asset);
        usageCount.products += usages.products.length;
        usageCount.variants += usages.variants.length;
        usageCount.collections += usages.collections.length;
    }
    
    // 2. 如果有引用且未强制删除，返回错误
    const hasUsages = !!(usageCount.products || usageCount.variants || usageCount.collections);
    if (hasUsages && !force) {
        return {
            result: DeletionResult.NOT_DELETED,
            message: ctx.translate('message.asset-to-be-deleted-is-featured', { ... }),
        };
    }
    
    // 3. 从 Channel 移除
    if (!deleteFromAllChannels) {
        await Promise.all(assets.map(async asset => {
            await this.channelService.removeFromChannels(ctx, Asset, asset.id, [ctx.channelId]);
        }));
        
        const isOnlyChannel = channelsOfAssets.length === 1;
        if (isOnlyChannel) {
            await this.deleteUnconditional(ctx, assets);
        }
        return { result: DeletionResult.DELETED };
    }
    
    // 4. 从所有 Channel 移除并删除
    await Promise.all(assets.map(async asset => {
        await this.channelService.removeFromChannels(ctx, Asset, asset.id, channelsOfAssets);
    }));
    return this.deleteUnconditional(ctx, assets);
}
```

### 5.3 仅被 assets 列表引用时的删除放行逻辑

**关键问题**：如果一个 Asset **仅被 OrderableAsset 关联表引用**（即只出现在某个 Product 的 `assets` 列表中，但没有被任何实体设为 `featuredAsset`），`findAssetUsages()` 将返回空结果，删除检查会放行！

**放行路径**：
```
Asset 仅在 Product.assets 列表中（通过 ProductAsset 关联）
      ↓
findAssetUsages() 只查询 featuredAsset
      ↓
返回 usageCount = { products: 0, variants: 0, collections: 0 }
      ↓
hasUsages = false
      ↓
✅ 删除放行，不提示警告
      ↓
deleteUnconditional() 执行
      ↓
数据库 CASCADE 自动删除 ProductAsset 记录
      ↓
Product.assets 列表中的该 Asset 被移除
```

**影响**：
- 删除前的警告信息不准确，用户可能不知道该 Asset 正在被某些实体的资源列表使用
- 删除后，引用该 Asset 的 Product.assets 列表会"丢失"这个资源
- 但数据库层面是安全的（CASCADE 保证），不会出现无效外键

### 5.4 无条件删除 (`deleteUnconditional`)

```typescript
private async deleteUnconditional(ctx: RequestContext, assets: Asset[]): Promise<DeletionResponse> {
    for (const asset of assets) {
        const deletedAsset = new Asset(asset);
        
        // 1. 从数据库删除
        await this.connection.getRepository(ctx, Asset).remove(asset);
        
        // 2. 删除源文件和预览文件
        try {
            await this.configService.assetOptions.assetStorageStrategy.deleteFile(asset.source);
            await this.configService.assetOptions.assetStorageStrategy.deleteFile(asset.preview);
            // ❌ 注意：不删除 cache/ 目录下的缓存文件！
        } catch (e) {
            Logger.error('error.could-not-delete-asset-file', undefined, e.stack);
        }
        
        await this.eventBus.publish(new AssetEvent(ctx, deletedAsset, 'deleted', deletedAsset.id));
    }
    return { result: DeletionResult.DELETED };
}
```

### 5.5 数据库级联删除的作用

当 Asset 被删除时：
1. **OrderableAsset 记录自动删除**：因为 `OrderableAsset.asset` 设置了 `onDelete: 'CASCADE'`
2. **业务实体的 featuredAsset 自动置空**：因为设置了 `onDelete: 'SET NULL'`

这意味着即使 `findAssetUsages()` 没有检查 OrderableAsset 中的引用，数据库也会自动清理这些关联记录。

---

## 六、实时图像转换与缓存

### 6.1 AssetServer 中间件 (`packages/asset-server-plugin/src/asset-server.ts`)

```typescript
createAssetServer(serverConfig): express.Router {
    const assetServer = express.Router();
    assetServer.use(this.sendAsset(), this.generateTransformedImage());
    return assetServer;
}

private sendAsset() {
    return async (req: Request, res: Response, next: NextFunction) => {
        let params: ImageTransformParameters;
        try {
            params = await this.getImageTransformParameters(req);
        } catch (e: any) {
            res.status(400).send('Invalid parameters');
            return;
        }
        // 生成缓存键
        const key = this.getFileNameFromParameters(req.path, params);
        try {
            // 尝试直接读取缓存文件
            const file = await this.assetStorageStrategy.readFileToBuffer(key);
            // ... 设置响应头并返回
            res.send(file);
        } catch (e: any) {
            // 缓存未命中，进入下一个中间件生成
            const err = new Error('File not found');
            (err as any).status = 404;
            return next(err);
        }
    };
}

private generateTransformedImage() {
    return async (err: any, req: Request, res: Response, next: NextFunction) => {
        if (err && (err.status === 404 || err.statusCode === 404)) {
            if (req.query) {
                const decodedReqPath = this.sanitizeFilePath(req.path);
                let file: Buffer;
                try {
                    // 读取原始源文件
                    file = await this.assetStorageStrategy.readFileToBuffer(decodedReqPath);
                } catch (_err: any) {
                    res.status(404).send('Resource not found');
                    return;
                }
                try {
                    const parameters = await this.getImageTransformParameters(req);
                    const image = await transformImage(file, parameters);
                    const imageBuffer = await image.toBuffer();
                    const cachedFileName = this.getFileNameFromParameters(req.path, parameters);
                    if (!req.query.cache || req.query.cache === 'true') {
                        // 保存到缓存
                        await this.assetStorageStrategy.writeFileFromBuffer(cachedFileName, imageBuffer);
                    }
                    // ... 设置响应头并返回
                    res.send(imageBuffer);
                    return;
                } catch (e: any) {
                    res.status(500).send('An error occurred when generating the image');
                    return;
                }
            }
        }
        next();
    };
}
```

### 6.2 缓存键生成规则 (`getFileNameFromParameters`)

`packages/asset-server-plugin/src/asset-server.ts:214`

```typescript
private getFileNameFromParameters(filePath: string, params: ImageTransformParameters): string {
    const { width: w, height: h, mode, preset, fpx, fpy, format, quality: q } = params;
    
    // 1. 构建参数字符串
    const focalPoint = fpx && fpy ? `_fpx${fpx}_fpy${fpy}` : '';
    const quality = q ? `_q${q}` : '';
    const imageFormat = getValidFormat(format);
    let imageParamsString = '';
    
    if (w || h) {
        const width = w || '';
        const height = h || '';
        imageParamsString = `_transform_w${width}_h${height}_m${mode}`;
    } else if (preset) {
        if (this.presets && !!this.presets.find(p => p.name === preset)) {
            imageParamsString = `_transform_pre_${preset}`;
        }
    }
    
    if (focalPoint) {
        imageParamsString += focalPoint;
    }
    if (imageFormat) {
        imageParamsString += imageFormat;
    }
    if (quality) {
        imageParamsString += quality;
    }
    
    // 2. 计算参数哈希
    const decodedReqPath = this.sanitizeFilePath(filePath);
    if (imageParamsString !== '') {
        const imageParamHash = this.md5(imageParamsString);
        // 3. 生成缓存文件名：cache/{原路径}_{hash}.{ext}
        return path.join(this.cacheDir, this.addSuffix(decodedReqPath, imageParamHash, imageFormat));
    } else {
        // 无转换参数时，直接返回原路径（即源文件本身）
        return decodedReqPath;
    }
}

private addSuffix(fileName: string, suffix: string, ext?: string): string {
    const originalExt = path.extname(fileName);
    const effectiveExt = ext ? `.${ext}` : originalExt;
    const baseName = path.basename(fileName, originalExt);
    const dirName = path.dirname(fileName);
    return path.join(dirName, `${baseName}${suffix}${effectiveExt}`);
}
```

### 6.3 缓存键生成示例

| 请求 URL | 生成的缓存键 |
|---------|-------------|
| `/assets/source/0a/photo.jpg?w=500&h=300&mode=crop` | `cache/source/0a/photo_transform_w500_h300_mcrop_<md5>.jpg` |
| `/assets/source/0a/photo.jpg?preset=thumb` | `cache/source/0a/photo_transform_pre_thumb_<md5>.jpg` |
| `/assets/source/0a/photo.jpg?w=500&format=webp&q=75` | `cache/source/0a/photo_transform_w500__mcrop_webp_q75_<md5>.webp` |
| `/assets/source/0a/photo.jpg` (无参数) | `source/0a/photo.jpg` (直接返回源文件路径) |

**哈希输入示例**：
- 参数字符串：`_transform_w500_h300_mcrop`
- MD5：`a1b2c3d4e5f6...`
- 缓存键：`cache/source/0a/photo_a1b2c3d4.jpg`

### 6.4 Asset 删除后缓存命中条件分析

**核心问题**：Asset 删除后，缓存文件是否仍可能被访问？

**完整读取链路**：
```
请求 /assets/source/0a/photo.jpg?w=500&h=300
      ↓
sendAsset()
      ├─ params = getImageTransformParameters(req)
      ├─ key = getFileNameFromParameters(req.path, params)
      │  → "cache/source/0a/photo_transform_w500_h300_mcrop_<hash>.jpg"
      ├─ 尝试 readFileToBuffer(key)
      ├─ ✅ 缓存文件存在 → 直接返回
      └─ ❌ 缓存不存在 → next(err) 进入 generateTransformedImage()
                          ↓
                          generateTransformedImage()
                              ├─ decodedReqPath = sanitizeFilePath(req.path)
                              │  → "source/0a/photo.jpg"
                              ├─ 尝试 readFileToBuffer(decodedReqPath)
                              ├─ ✅ 源文件存在 → 转换、缓存、返回
                              └─ ❌ 源文件不存在 → 返回 404
```

**Asset 删除后的访问结果**：

| 场景 | 缓存文件存在？ | 源文件存在？ | 结果 |
|-----|--------------|-------------|------|
| 请求带转换参数 | ✅ 是 | ❌ 否 | ✅ **返回缓存文件**（sendAsset 直接命中缓存） |
| 请求带转换参数 | ❌ 否 | ❌ 否 | ❌ 404（generateTransformedImage 读源文件失败） |
| 请求不带参数 | - | ❌ 否 | ❌ 404（key 就是源文件路径，直接读取失败） |

> **代码证据**：`getFileNameFromParameters` 当有转换参数时返回 `cache/` 路径，sendAsset 直接尝试读取该路径。只要缓存文件还在，就能成功返回，**不会检查源文件是否存在**。只有当缓存不存在时，才会进入 generateTransformedImage 尝试读取源文件。

**结论**：Asset 删除后，**带转换参数的请求仍可能命中缓存并成功返回**，只要缓存文件未被清理。这意味着：
- 已删除的 Asset 的缓存变体可能在一段时间内仍然可访问
- 缓存文件需要单独的清理机制（如 TTL 过期、定期扫描）

### 6.5 缓存文件与删除流程的关系

**删除流程中涉及的文件**：
```typescript
private async deleteUnconditional(ctx: RequestContext, assets: Asset[]): Promise<DeletionResponse> {
    for (const asset of assets) {
        await this.connection.getRepository(ctx, Asset).remove(asset);
        
        try {
            // 只删除 source 和 preview
            await this.configService.assetOptions.assetStorageStrategy.deleteFile(asset.source);
            await this.configService.assetOptions.assetStorageStrategy.deleteFile(asset.preview);
            // ❌ 不删除 cache/ 目录下的任何文件！
        } catch (e) {
            Logger.error('error.could-not-delete-asset-file', undefined, e.stack);
        }
    }
}
```

**缓存文件残留问题**：
- 每个源文件可能生成多个缓存变体（不同尺寸、格式、质量、焦点）
- 删除 Asset 后，这些缓存文件仍然存在于 `cache/` 目录
- 这些文件成为"僵尸缓存"，占用存储空间但永远不会被新请求生成（因为源文件已删除）
- 但**已存在的缓存文件仍可能被直接访问**（如果有人知道完整的带参数 URL）

**缓存文件定位困难**：
- 缓存键包含参数哈希，无法简单通过源文件名推导所有缓存变体
- 如果要彻底清理，需要：
  1. 扫描 `cache/` 目录下所有文件
  2. 解析文件名，提取原始路径部分（去掉 `_transform_*_<hash>` 后缀）
  3. 检查该原始路径对应的 Asset 是否还存在
  4. 不存在则删除缓存文件

### 6.6 焦点参数 (fpx/fpy) 0 值处理不一致问题

#### 6.6.1 文档语义

`packages/asset-server-plugin/src/plugin.ts:77` 明确说明：
> These are normalized coordinates (i.e. a number between 0 and 1), so the `fpx=0&fpy=0` corresponds to the top left of the image.

即 `fpx=0&fpy=0` 表示左上角，是**合法有效的焦点坐标**。

#### 6.6.2 三个环节对 0 值的处理对比

| 环节 | 代码位置 | 处理逻辑 | 对 0 值的处理 |
|-----|---------|---------|-------------|
| **参数解析** | `asset-server.ts:198-199` | `const fpx = +queryParams.fpx || undefined;` | ❌ 0 被当作 falsy 值，被替换为 `undefined` |
| **缓存键生成** | `asset-server.ts:217` | `const focalPoint = fpx && fpy ? ... : '';` | ❌ 0 被当作 falsy 值，不加入缓存键 |
| **裁剪执行** | `transform-image.ts:32` | `if (parameters.fpx && parameters.fpy && ...)` | ❌ 0 被当作 falsy 值，不执行焦点裁剪 |

#### 6.6.3 详细代码证据

**1. 参数解析环节 (`asset-server.ts:190-212`)**：
```typescript
private getInitialImageTransformParameters(
    queryParams: Record<string, string>,
): ImageTransformParameters {
    const fpx = +queryParams.fpx || undefined;  // +'0' = 0 → 0 || undefined → undefined
    const fpy = +queryParams.fpy || undefined;  // 同上
    return { fpx, fpy, ... };
}
```
**问题**：使用 `||` 短路运算符，`0` 被当作 falsy 值，合法的 `fpx=0` 会被替换为 `undefined`。

**2. 缓存键生成环节 (`asset-server.ts:214-248`)**：
```typescript
private getFileNameFromParameters(filePath: string, params: ImageTransformParameters): string {
    const { fpx, fpy } = params;
    const focalPoint = fpx && fpy ? `_fpx${fpx}_fpy${fpy}` : '';  // 0 && 0 = 0 → falsy
    // ...
    if (imageParamsString !== '') {
        const imageParamHash = this.md5(imageParamsString);  // 不包含 fpx/fpy
        return path.join(this.cacheDir, this.addSuffix(decodedReqPath, imageParamHash, ...));
    }
}
```
**问题**：同样使用 `&&` 运算符，`fpx=0&fpy=0` 时 `focalPoint` 为空字符串，焦点参数不参与哈希计算。

**3. 裁剪执行环节 (`transform-image.ts:14-51`)**：
```typescript
export async function transformImage(
    originalImage: Buffer,
    parameters: ImageTransformParameters,
): Promise<sharp.Sharp> {
    const options: ResizeOptions = {};
    if (mode === 'crop') {
        options.position = sharp.strategy.entropy;  // 默认使用熵裁剪
    }
    // ...
    if (parameters.fpx && parameters.fpy && width && height && mode === 'crop') {
        // 0 && 0 = 0 → falsy，条件不成立
        // 焦点裁剪逻辑永远不会执行
        const xCenter = parameters.fpx * metadata.width;
        // ...
        return image.resize(resizedWidth, resizedHeight).extract(region);
    }
    return image.resize(width, height, options);  // 回退到默认熵裁剪
}
```
**问题**：同样使用 `&&` 运算符，`fpx=0&fpy=0` 时条件不成立，直接跳过焦点裁剪，使用默认的熵裁剪。

#### 6.6.4 实际行为偏差

| 场景 | 用户预期（按文档语义） | 实际行为 | 偏差说明 |
|-----|----------------------|---------|---------|
| `fpx=0&fpy=0&mode=crop` | 以左上角为焦点裁剪 | 使用默认熵裁剪 | 完全忽略焦点参数，裁剪结果可能不包含左上角 |
| `fpx=0&fpy=0.5&mode=crop` | 以左边缘中点为焦点裁剪 | 使用默认熵裁剪 | 因为 fpx=0 被忽略，整个焦点参数失效 |
| `fpx=0.5&fpy=0&mode=crop` | 以上边缘中点为焦点裁剪 | 使用默认熵裁剪 | 因为 fpy=0 被忽略，整个焦点参数失效 |

**缓存一致性问题**：
```
请求 1: ?w=100&h=100&mode=crop&fpx=0&fpy=0
  → fpx=undefined, fpy=undefined
  → 缓存键: cache/source/0a/photo_transform_w100_h100_mcrop_<hash1>.jpg

请求 2: ?w=100&h=100&mode=crop
  → fpx=undefined, fpy=undefined
  → 缓存键: cache/source/0a/photo_transform_w100_h100_mcrop_<hash1>.jpg

结果：两个不同的请求命中同一个缓存！
```
因为 `fpx=0&fpy=0` 被当作无焦点参数处理，所以带 `fpx=0&fpy=0` 的请求会与不带焦点参数的请求共享缓存。如果后者先访问，前者会得到熵裁剪的结果而不是焦点裁剪的结果。

#### 6.6.5 测试用例佐证

`transform-image.spec.ts:22-36` 中的测试用例：
```typescript
it('no resize, crop top left', () => {
    const original: Dimensions = { w: 200, h: 100 };
    const target: Dimensions = { w: 100, h: 100 };
    const focalPoint: Point = { x: 0, y: 0 };  // 传入 0 值
    const result = resizeToFocalPoint(original, target, focalPoint);
    expect(result.region).toEqual({
        left: 0,
        top: 0,
        width: 100,
        height: 100,
    });
});
```

> **重要发现**：测试用例直接调用 `resizeToFocalPoint()` 函数，传入 `x: 0, y: 0` 是有效的。但在实际请求链路中，参数解析环节已经把 `fpx=0` 转换成了 `undefined`，所以这个测试用例**无法覆盖实际的 HTTP 请求场景**。

### 6.7 焦点参数范围限制缺失与缓存一致性问题

#### 6.7.1 文档语义与实现对比

**文档语义** (`packages/asset-server-plugin/src/plugin.ts:77`)：
> These are normalized coordinates (i.e. a number between 0 and 1), so the `fpx=0&fpy=0` corresponds to the top left of the image.

文档明确说明 fpx/fpy 是 **0 到 1 之间的归一化坐标**。

**实际实现**：三个环节都**没有对参数范围进行限制**：

| 环节 | 代码位置 | 处理逻辑 | 范围限制 |
|-----|---------|---------|---------|
| **参数解析** | `asset-server.ts:198-199` | `const fpx = +queryParams.fpx || undefined;` | ❌ 无限制，可传入任意数字（负数、>1、甚至NaN） |
| **缓存键生成** | `asset-server.ts:217` | `const focalPoint = fpx && fpy ? \`_fpx${fpx}_fpy${fpy}\` : '';` | ❌ 直接使用原始参数值，不做范围检查 |
| **裁剪执行** | `transform-image.ts:35-36` | `const xCenter = parameters.fpx * metadata.width;` | ❌ 直接相乘，不限制 fpx 范围 |

**对比 quality 参数**（有正确的范围限制）：
```typescript
// asset-server.ts:195-196
const quality =
    queryParams.q != null ? Math.round(Math.max(Math.min(+queryParams.q, 100), 1)) : undefined;
```
quality 参数使用了 `Math.max(Math.min(..., 100), 1)` 明确限制在 1-100 范围内，而 fpx/fpy 没有类似处理。

#### 6.7.2 越界值在裁剪阶段的 clamp 处理

虽然参数解析不限制范围，但在裁剪执行的**最后一步**，`getExtractionRegion` 函数会对最终的裁剪区域进行 clamp：

```typescript
// transform-image.ts:144-148
if (intermediate.h < intermediate.w) {
    region.left = clamp(0, intermediate.w - target.w, Math.round(newXCenter - target.w / 2));
} else {
    region.top = clamp(0, intermediate.h - target.h, Math.round(newYCenter - target.h / 2));
}

// transform-image.ts:155-157
function clamp(min: number, max: number, input: number) {
    return Math.min(Math.max(min, input), max);
}
```

**越界值处理流程**（以 fpx 为例，图片宽度 1000px，目标裁剪宽度 200px）：

```
fpx = 1.5 (越界，大于 1)
  ↓
xCenter = 1.5 * 1000 = 1500
  ↓
newXCenter = 1500 / factor (假设 factor=1) = 1500
  ↓
region.left = 1500 - 200/2 = 1400
  ↓
clamp(0, 1000-200=800, 1400) → 800 (被限制到右边界)
  ↓
最终裁剪区域 left=800, width=200 → 裁剪图片最右侧 200px
```

**不同越界值的裁剪结果**：

| fpx 值 | 计算 xCenter | 计算 region.left | clamp 后 left | 实际裁剪区域 |
|--------|-------------|-----------------|--------------|------------|
| 0.0 | 0 | -100 | 0 | 最左侧 200px |
| 0.5 | 500 | 400 | 400 | 中间 200px |
| 1.0 | 1000 | 900 | 800 | 最右侧 200px |
| 1.5 | 1500 | 1400 | 800 | 最右侧 200px |
| 2.0 | 2000 | 1900 | 800 | 最右侧 200px |
| -0.5 | -500 | -600 | 0 | 最左侧 200px |

**关键发现**：
- `fpx=1.0`、`fpx=1.5`、`fpx=2.0` 最终裁剪结果**完全相同**（都是最右侧 200px）
- `fpx=-0.5` 和 `fpx=0.0` 最终裁剪结果**完全相同**（都是最左侧 200px）
- 越界值最终都会被 clamp 到边界，但中间计算过程使用了原始值

#### 6.7.3 缓存一致性问题：相同结果，不同缓存键

**核心问题**：缓存键使用**原始参数值**生成，但裁剪结果被 clamp 到边界。这导致**不同参数可能得到相同裁剪结果，但写入不同缓存键**。

**示例场景**（图片宽度 1000px，目标裁剪宽度 200px）：

```
请求 1: ?w=200&h=200&mode=crop&fpx=1.0&fpy=0.5
  → 缓存键包含: _fpx1_fpy0.5
  → 实际裁剪: 最右侧 200px
  → 写入缓存: cache/..._fpx1_fpy0.5_<hash1>.jpg

请求 2: ?w=200&h=200&mode=crop&fpx=1.5&fpy=0.5
  → 缓存键包含: _fpx1.5_fpy0.5
  → 实际裁剪: 最右侧 200px (与请求 1 完全相同)
  → 写入缓存: cache/..._fpx1.5_fpy0.5_<hash2>.jpg  ❌ 重复缓存！

请求 3: ?w=200&h=200&mode=crop&fpx=2.0&fpy=0.5
  → 缓存键包含: _fpx2_fpy0.5
  → 实际裁剪: 最右侧 200px (与请求 1 完全相同)
  → 写入缓存: cache/..._fpx2_fpy0.5_<hash3>.jpg  ❌ 再次重复缓存！
```

**缓存浪费分析**：
- 3 个不同的请求
- 产生完全相同的裁剪结果
- 写入 3 个不同的缓存文件
- 占用 3 倍存储空间

**越界值的边界范围**：

对于 fpx，理论上：
- `fpx <= target.w / (2 * original.w)` 时，都会被 clamp 到 left=0
- `fpx >= 1 - target.w / (2 * original.w)` 时，都会被 clamp 到 left=original.w - target.w

对于 1000px 宽的图片，裁剪 200px：
- `fpx <= 0.1` → left=0
- `fpx >= 0.9` → left=800

这意味着在 `(-∞, 0.1]` 范围内的所有 fpx 值都会得到相同的裁剪结果，但每个不同的值都会生成不同的缓存键。

#### 6.7.4 完整证据链总结

| 环节 | 输入语义 | 实际行为 | 与缓存一致性的关系 |
|-----|---------|---------|-------------------|
| **参数解析** | 0-1 归一化坐标 | 接受任意数字，无范围限制 | 越界值进入后续流程 |
| **缓存键生成** | 应使用有效参数哈希 | 使用原始参数值哈希 | 不同越界值生成不同缓存键 |
| **裁剪计算** | 基于归一化坐标计算 | 直接使用原始值计算 | 越界值参与中间计算 |
| **区域提取** | 应得到合理裁剪区域 | 最终结果被 clamp 到边界 | 不同越界值可能得到相同结果 |
| **缓存写入** | 相同结果应共享缓存 | 相同结果写入不同缓存 | 产生大量重复缓存 |

**根本原因**：参数范围验证缺失 + 缓存键在 clamp 之前生成。

正确的设计应该是：
1. 参数解析时就将 fpx/fpy clamp 到 [0, 1] 范围
2. 使用 clamp 后的值生成缓存键
3. 使用 clamp 后的值进行裁剪计算

这样可以保证：
- 相同的有效裁剪结果使用相同的缓存键
- 避免缓存空间浪费

---

## 七、业务实体更新资源引用

### 7.1 AssetService.updateEntityAssets

`packages/core/src/service/services/asset.service.ts:273`

```typescript
async updateEntityAssets<T extends EntityWithAssets>(
    ctx: RequestContext,
    entity: T,
    input: EntityAssetInput,
): Promise<T> {
    const { assetIds } = input;
    if (assetIds && assetIds.length) {
        const assets = await this.connection.findByIdsInChannel(ctx, Asset, assetIds, ctx.channelId, {});
        const sortedAssets = assetIds.map(id => assets.find(a => idsAreEqual(a.id, id))).filter(notNullOrUndefined);
        
        await this.removeExistingOrderableAssets(ctx, entity);
        if (sortedAssets.length > 0) {
            entity.assets = await this.createOrderableAssets(ctx, entity, sortedAssets);
        } else {
            entity.assets = [];
        }
    } else if (assetIds && assetIds.length === 0) {
        await this.removeExistingOrderableAssets(ctx, entity);
    }
    return entity;
}
```

**工作原理**：
1. 先删除该实体所有现有的 OrderableAsset 记录
2. 根据传入的 `assetIds` 顺序创建新的 OrderableAsset 记录
3. 没有引用计数维护，每次都是全量替换

---

## 八、协同机制总结

### 8.1 上传时序

```
GraphQL Upload
      ↓
AssetService.create()
      ├─→ AssetNamingStrategy.generateSourceFileName()
      │   → "source/0a/photo.jpg"
      ├─→ AssetStorageStrategy.writeFileFromStream()  [写入源文件]
      ├─→ 🔴 失败点：此处及之后失败会留孤儿文件
      ├─→ AssetStorageStrategy.readFileToBuffer()
      ├─→ AssetPreviewStrategy.generatePreviewImage() [生成预览]
      ├─→ AssetStorageStrategy.writeFileFromBuffer()  [写入预览文件]
      └─→ 保存 Asset 实体到数据库
```

### 8.2 引用维护

1. **业务实体引用 Asset**：通过 `featuredAsset`（单值）或 `assets`（通过 OrderableAsset 关联表）
2. **无显式引用计数器**：删除前通过 `findAssetUsages()` 动态查询引用
3. **引用检查不完整**：只检查 `featuredAsset`，不检查 OrderableAsset 列表引用
4. **数据库级联兜底**：
   - Asset 删除 → OrderableAsset 自动删除（CASCADE）
   - Asset 删除 → 业务实体 featuredAsset 自动置空（SET NULL）

### 8.3 文件存储标识协同

| 组件 | 存储字段 | 含义 | 生成者 |
|-----|---------|------|--------|
| Asset | source | 源文件标识符 | AssetStorageStrategy.writeFileFromStream() |
| Asset | preview | 预览文件标识符 | AssetStorageStrategy.writeFileFromBuffer() |
| 缓存 | - | 转换后图像 | AssetServer.generateTransformedImage() |

所有标识符都通过 `AssetStorageStrategy` 生成，读取和删除时通过同一策略反向解析。

### 8.4 删除时序

```
AssetService.delete(ids, force?)
      ↓
findAssetUsages() → 仅检查 featuredAsset 引用
      ↓
有 featured 引用 && !force → 返回 NOT_DELETED
      ↓
从 Channel 移除
      ↓
是最后一个 Channel？
      ├─ 是 → deleteUnconditional()
      │        ├─ 数据库删除 Asset
      │        ├─ 级联删除 OrderableAsset (CASCADE)
      │        ├─ 级联置空 featuredAsset (SET NULL)
      │        ├─ AssetStorageStrategy.deleteFile(source)
      │        ├─ AssetStorageStrategy.deleteFile(preview)
      │        └─ ❌ 不删除 cache/ 下的缓存文件
      └─ 否 → 仅从当前 Channel 移除
```

### 8.5 toAbsoluteUrl 转换链路

```
GraphQL 查询
      ↓
AssetInterceptorPlugin.willSendResponse()
      ↓
遍历响应数据，查找 Asset / SearchResultAsset 类型
      ↓
调用 AssetStorageStrategy.toAbsoluteUrl(request, identifier)
      ↓
getAssetUrlPrefixFn(request, identifier) 获取前缀
      ↓
返回 {prefix}{identifier} 作为最终 URL
```

### 8.6 缓存读取链路

```
请求 /assets/source/0a/photo.jpg?w=500
      ↓
sendAsset()
      ├─ key = getFileNameFromParameters(req.path, params)
      │  → "cache/source/0a/photo_transform_w500_<hash>.jpg"
      ├─ readFileToBuffer(key)
      ├─ ✅ 缓存存在 → 返回
      └─ ❌ 缓存不存在 → next(404)
                      ↓
                      generateTransformedImage()
                          ├─ readFileToBuffer("source/0a/photo.jpg")
                          ├─ ✅ 源文件存在 → 转换 → 写入缓存 → 返回
                          └─ ❌ 源文件不存在 → 返回 404
```

---

## 九、关键结论与注意事项

### 1. **上传失败产生孤儿文件**
   - `createAssetInternal` 无失败回滚机制
   - 步骤 3-7 中任何一步失败，已写入的文件无法自动清理
   - 需要定期扫描文件系统比对数据库来清理孤儿文件

### 2. **Local 与 S3 策略的 identifier 异同**
   - **相同点**：最终 identifier 格式完全相同（`source/0a/photo.jpg`）
   - **不同点**：
     - Local 策略需要 `filePathToIdentifier` 在完整路径和相对路径间转换
     - S3 策略直接使用传入的 fileName 作为 Key，无需路径转换
   - **toAbsoluteUrl 逻辑完全相同**：都是 `{prefix}{identifier}`

### 3. **findAssetUsages 不检查 OrderableAsset**
   - 只检查 `featuredAsset` 引用，不检查 OrderableAsset 关联表
   - 仅被资源列表引用的 Asset 可以被"静默"删除
   - 删除前的警告信息可能不准确
   - 但数据库 CASCADE 会自动清理，不会出现无效外键

### 4. **Asset 删除后缓存仍可能被访问**
   - 带转换参数的请求：先查缓存，缓存存在直接返回，**不检查源文件**
   - 不带转换参数的请求：直接读源文件，删除后返回 404
   - 已删除 Asset 的缓存变体可能在一段时间内仍然可访问
   - 需要单独的缓存清理机制

### 5. **缓存文件不跟踪**
   - 实时转换生成的缓存文件没有被 Asset 实体跟踪
   - 删除 Asset 时不会自动删除缓存文件
   - 缓存键包含参数哈希，难以枚举所有变体
   - 需要单独的清理机制（如定时任务扫描并删除过期缓存）

### 6. **多 Channel 共享**
   - 一个 Asset 可以属于多个 Channel
   - 只有从所有 Channel 移除后才会真正删除文件
   - 这意味着文件存储是跨 Channel 共享的

### 7. **无引用计数列**
   - 每次删除都需要执行 3 个查询（Product、ProductVariant、Collection）
   - 高并发删除场景可能有性能问题

### 8. **标识符与 URL 分离**
   - 数据库存储内部标识符，输出时通过 `toAbsoluteUrl` 转换
   - Local 和 S3 策略的标识符格式相同，仅前缀配置不同
   - 切换存储策略时需要注意迁移现有文件的 URL

### 9. **fpx/fpy 焦点参数设计缺陷汇总**

#### 9.1 0 值处理不一致 Bug
   - **文档语义**：`fpx=0&fpy=0` 表示左上角，是合法有效的焦点坐标
   - **实现问题**：三个环节都使用 `||` 或 `&&` 运算符，0 被当作 falsy 值忽略
     - 参数解析：`+queryParams.fpx || undefined` → 0 变成 undefined
     - 缓存键生成：`fpx && fpy ? ... : ''` → 0 不参与哈希
     - 裁剪执行：`if (parameters.fpx && parameters.fpy)` → 0 跳过焦点裁剪
   - **实际影响**：
     - `fpx=0&fpy=0` 等合法焦点坐标被完全忽略
     - 回退到默认熵裁剪，结果可能不符合用户预期
     - 带 `fpx=0&fpy=0` 的请求与不带焦点参数的请求共享缓存
   - **测试漏洞**：单元测试直接调用 `resizeToFocalPoint()` 传入 0 值，未覆盖实际 HTTP 请求链路

#### 9.2 范围限制缺失与缓存一致性问题
   - **文档语义**：fpx/fpy 是 0 到 1 之间的归一化坐标
   - **实现问题**：参数解析环节完全没有范围限制（对比 quality 参数有 `Math.max(Math.min(..., 100), 1)`）
   - **越界值处理**：仅在裁剪区域提取的最后一步通过 `clamp()` 限制到边界
   - **缓存一致性问题**：
     - 缓存键使用原始参数值生成，但裁剪结果被 clamp 到边界
     - 不同越界参数可能得到相同裁剪结果，但写入不同缓存键
     - 例如 `fpx=1.0`、`fpx=1.5`、`fpx=2.0` 裁剪结果相同，但生成 3 个不同缓存
     - 造成缓存空间浪费
   - **边界范围**（以 1000px 宽图片裁剪 200px 为例）：
     - `fpx <= 0.1` → 都被 clamp 到 left=0，生成不同缓存键
     - `fpx >= 0.9` → 都被 clamp 到 left=800，生成不同缓存键
   - **根本原因**：参数范围验证缺失 + 缓存键在 clamp 之前生成
