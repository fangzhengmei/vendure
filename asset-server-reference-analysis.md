# Vendure 资源服务协作分析

## 概述

Vendure 中的资源（Asset）系统涉及三个核心组件的协同工作：

1. **资源服务（AssetService）** - 位于 `@vendure/core`，负责资产生命周期管理
2. **媒体处理插件（AssetServerPlugin）** - 负责文件存储、预览图生成和实时图像转换
3. **业务实体（Product / ProductVariant / Collection）** - 通过引用关系使用资源

本文档分析三者之间的协作机制，特别是引用计数和文件存储位置的协同。

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
- `source` 和 `preview` 字段存储文件标识符（相对路径或URL）
- 没有显式的 `引用计数字段`，引用计数通过查询关联表动态计算

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

### 2.1 上传入口 (`packages/core/src/service/services/asset.service.ts`)

```typescript
async create(ctx: RequestContext, input: CreateAssetInput): Promise<Asset> {
    const { createReadStream, filename, mimetype } = await input.file;
    const { stream, errorPromise } = this.makeStreamGuard(createReadStream);
    const result = await Promise.race([
        this.createAssetInternal(ctx, stream, filename, mimetype, input.customFields, input.translations),
        errorPromise,
    ]);
    // ...
}
```

### 2.2 核心上传逻辑 (`createAssetInternal`)

```typescript
private async createAssetInternal(
    ctx: RequestContext,
    stream: Stream,
    filename: string,
    mimetype: string,
): Promise<Asset> {
    const { assetPreviewStrategy, assetStorageStrategy } = this.configService.assetOptions;
    
    // 步骤 1: 生成文件名
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

### 2.3 失败分支与孤儿文件分析

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

### 2.4 文件命名策略 (`HashedAssetNamingStrategy`)

`packages/asset-server-plugin/src/config/hashed-asset-naming-strategy.ts`

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

**存储目录结构**：
```
assets/
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

### 2.5 本地存储策略 (`LocalAssetStorageStrategy`)

`packages/asset-server-plugin/src/config/local-asset-storage-strategy.ts`

```typescript
export class LocalAssetStorageStrategy implements AssetStorageStrategy {
    toAbsoluteUrl: ((reqest: Request, identifier: string) => string) | undefined;

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
        const filePath = path.join(this.uploadPath, fileName);
        await fs.ensureDir(path.dirname(filePath));
        const writeStream = fs.createWriteStream(filePath, 'binary');
        return new Promise<string>((resolve, reject) => {
            data.pipe(writeStream);
            writeStream.on('close', () => resolve(this.filePathToIdentifier(filePath)));
            writeStream.on('error', reject);
        });
    }
    
    deleteFile(identifier: string): Promise<void> {
        return fs.unlink(this.identifierToFilePath(identifier));
    }
    
    private filePathToIdentifier(filePath: string): string {
        const deltaDirname = path.dirname(filePath).replace(this.uploadPath, '');
        const identifier = path.join(deltaDirname, path.basename(filePath));
        return identifier.replace(/^[\\/]+/, '');
    }
    
    private identifierToFilePath(identifier: string): string {
        return path.join(this.uploadPath, identifier);
    }
}
```

**关键点**：
- `identifier` 是相对于 `uploadPath` 的相对路径
- 删除时通过 `identifier` 定位到实际文件路径
- `toAbsoluteUrl` 是可选方法，用于将 identifier 转换为对外可访问的 URL

---

## 三、预览图生成

### 3.1 SharpAssetPreviewStrategy (`packages/asset-server-plugin/src/config/sharp-asset-preview-strategy.ts`)

```typescript
export class SharpAssetPreviewStrategy implements AssetPreviewStrategy {
    async generatePreviewImage(ctx, mimeType, data): Promise<Buffer> {
        const assetType = getAssetType(mimeType);
        
        if (assetType === AssetType.IMAGE) {
            try {
                const image = sharp(data, { failOn: 'truncated' }).rotate();
                const metadata = await image.metadata();
                const width = metadata.width || 0;
                const height = metadata.height || 0;
                if (maxWidth < width || maxHeight < height) {
                    image.resize(maxWidth, maxHeight, { fit: 'inside' });
                }
                if (mimeType === 'image/svg+xml') {
                    return image.toBuffer();
                } else {
                    switch (metadata.format) {
                        case 'jpeg':
                        case 'jpg':
                            return image.jpeg(this.config.jpegOptions).toBuffer();
                        case 'png':
                            return image.png(this.config.pngOptions).toBuffer();
                        case 'webp':
                            return image.webp(this.config.webpOptions).toBuffer();
                        case 'gif':
                            return image.gif(this.config.jpegOptions).toBuffer();
                        case 'avif':
                            return image.avif(this.config.avifOptions).toBuffer();
                        default:
                            return image.toBuffer();
                    }
                }
            } catch (err: any) {
                Logger.error(
                    `An error occurred when generating preview for image with mimeType ${mimeType}: ${JSON.stringify(
                        err.message,
                    )}`,
                    loggerCtx,
                );
                return this.generateBinaryFilePreview(mimeType);
            }
        } else {
            return this.generateBinaryFilePreview(mimeType);
        }
    }
    
    private generateBinaryFilePreview(mimeType: string): Promise<Buffer> {
        return sharp(path.join(__dirname, '..', 'file-icon.png'))
            .resize(800, 800, { fit: 'outside' })
            .composite([
                {
                    input: this.generateMimeTypeOverlay(mimeType),
                    gravity: sharp.gravity.center,
                },
            ])
            .toBuffer();
    }
}
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
    
    private isAssetType(type: GraphQLNamedType): boolean {
        const assetTypeNames = ['Asset', 'SearchResultAsset'];
        return assetTypeNames.includes(type.name);
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
    throw new Error(`The assetUrlPrefix option was of an unexpected type: ${JSON.stringify(assetUrlPrefix)}`);
}
```

### 4.4 Local 与 S3 策略的标识符差异

| 策略 | identifier 格式 | toAbsoluteUrl 实现 | 示例 URL |
|------|----------------|-------------------|---------|
| **LocalAssetStorageStrategy** | 相对路径（如 `source/0a/xxx.jpg`） | `{prefix}{identifier}` | `https://cdn.example.com/assets/source/0a/xxx.jpg` |
| **S3AssetStorageStrategy** | S3 Key（如 `source/0a/xxx.jpg`） | `{prefix}{identifier}` | `https://my-bucket.s3.amazonaws.com/source/0a/xxx.jpg` |

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

**关键差异**：
- **标识符本身格式相同**：都是 `source/{hash}/{filename}` 格式
- **存储位置不同**：Local 存在本地文件系统，S3 存在对象存储
- **前缀配置不同**：
  - Local: `assetUrlPrefix` 通常配置为 CDN 或服务器地址 + `/assets/`
  - S3: `assetUrlPrefix` 通常配置为 S3 bucket 访问地址（如 `https://my-bucket.s3.amazonaws.com/`）
- **文件读取方式不同**：
  - Local: 直接 `fs.readFile()`
  - S3: 通过 AWS SDK 调用 `GetObjectCommand`

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

**⚠️ 重要发现**：此方法**只检查 `featuredAsset` 引用**，**不检查 OrderableAsset 关联表中的引用**！

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

## 六、业务实体更新资源引用

### 6.1 AssetService.updateEntityAssets

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

## 七、实时图像转换与缓存

### 7.1 AssetServer 中间件 (`packages/asset-server-plugin/src/asset-server.ts`)

```typescript
createAssetServer(serverConfig): express.Router {
    const assetServer = express.Router();
    assetServer.use(this.sendAsset(), this.generateTransformedImage());
    return assetServer;
}

private sendAsset() {
    return async (req, res, next) => {
        const params = await this.getImageTransformParameters(req);
        const key = this.getFileNameFromParameters(req.path, params);
        
        try {
            // 尝试直接读取缓存文件
            const file = await this.assetStorageStrategy.readFileToBuffer(key);
            res.send(file);
        } catch (e) {
            // 缓存未命中，进入下一个中间件生成
            next(err);
        }
    };
}

private generateTransformedImage() {
    return async (err, req, res, next) => {
        if (err && err.status === 404) {
            const file = await this.assetStorageStrategy.readFileToBuffer(decodedReqPath);
            const parameters = await this.getImageTransformParameters(req);
            const image = await transformImage(file, parameters);
            const imageBuffer = await image.toBuffer();
            
            const cachedFileName = this.getFileNameFromParameters(req.path, parameters);
            if (!req.query.cache || req.query.cache === 'true') {
                // 保存到缓存
                await this.assetStorageStrategy.writeFileFromBuffer(cachedFileName, imageBuffer);
            }            res.send(imageBuffer);
        }
    };
}
```

### 7.2 缓存键生成规则 (`getFileNameFromParameters`)

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

### 7.3 缓存键生成示例

| 请求 URL | 生成的缓存键 |
|---------|-------------|
| `/assets/source/0a/photo.jpg?w=500&h=300&mode=crop` | `cache/source/0a/photo_transform_w500_h300_mcrop_<md5>.jpg` |
| `/assets/source/0a/photo.jpg?preset=thumb` | `cache/source/0a/photo_transform_pre_thumb_<md5>.jpg` |
| `/assets/source/0a/photo.jpg?w=500&format=webp&q=75` | `cache/source/0a/photo_transform_w500__mcrop_webp_q75_<md5>.webp` |
| `/assets/source/0a/photo.jpg?w=500&fpx=0.3&fpy=0.7` | `cache/source/0a/photo_transform_w500__mcrop_fpx0.3_fpy0.7_<md5>.jpg` |

**哈希输入示例**：
- 输入：`_transform_w500_h300_mcrop`
- MD5：`a1b2c3d4e5f6...`
- 缓存键：`cache/source/0a/photo_a1b2c3d4.jpg`

### 7.4 缓存文件与删除流程的关系

**核心问题**：缓存文件**不被 Asset 实体跟踪**，删除 Asset 时不会自动清理缓存文件。

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
- 每个源文件可能生成多个缓存变体（不同尺寸、格式、质量）
- 删除 Asset 后，这些缓存文件仍然存在于 `cache/` 目录
- 这些文件成为"僵尸缓存"，占用存储空间但永远不会被访问
- 需要单独的清理机制（如定时任务扫描并删除过期缓存）

**缓存文件定位困难**：
- 缓存键包含参数哈希，无法简单通过源文件名推导所有缓存变体
- 如果要彻底清理，需要：
  1. 扫描 `cache/` 目录下所有文件
  2. 解析文件名，提取原始路径部分
  3. 检查该原始路径对应的 Asset 是否还存在
  4. 不存在则删除缓存文件

---

## 八、协同机制总结

### 8.1 上传时序

```
GraphQL Upload
      ↓
AssetService.create()
      ├─→ AssetNamingStrategy.generateSourceFileName()
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

所有标识符都通过 `AssetStorageStrategy` 的 `filePathToIdentifier` 方法生成，删除时通过同一策略的 `identifierToFilePath` 反向解析。

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

---

## 九、潜在问题与注意事项

### 1. **上传失败产生孤儿文件**
   - `createAssetInternal` 无失败回滚机制
   - 步骤 3-7 中任何一步失败，已写入的文件无法自动清理
   - 需要定期扫描文件系统比对数据库来清理孤儿文件

### 2. **findAssetUsages 不检查 OrderableAsset**
   - 只检查 `featuredAsset` 引用，不检查 OrderableAsset 关联表
   - 仅被资源列表引用的 Asset 可以被"静默"删除
   - 删除前的警告信息可能不准确
   - 但数据库 CASCADE 会自动清理，不会出现无效外键

### 3. **缓存文件不跟踪**
   - 实时转换生成的缓存文件没有被 Asset 实体跟踪
   - 删除 Asset 时不会自动删除缓存文件
   - 缓存键包含参数哈希，难以枚举所有变体
   - 需要单独的清理机制（如定时任务扫描并删除过期缓存）

### 4. **多 Channel 共享**
   - 一个 Asset 可以属于多个 Channel
   - 只有从所有 Channel 移除后才会真正删除文件
   - 这意味着文件存储是跨 Channel 共享的

### 5. **无引用计数列**
   - 每次删除都需要执行 3 个查询（Product、ProductVariant、Collection）
   - 高并发删除场景可能有性能问题

### 6. **标识符与 URL 分离**
   - 数据库存储内部标识符，输出时通过 `toAbsoluteUrl` 转换
   - Local 和 S3 策略的标识符格式相同，仅前缀配置不同
   - 切换存储策略时需要注意迁移现有文件的 URL
