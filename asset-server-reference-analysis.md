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

## 二、资源上传流程

### 2.1 上传入口 (`packages/core/src/service/services/asset.service.ts`)

```typescript
async create(ctx: RequestContext, input: CreateAssetInput): Promise<Asset> {
    const { createReadStream, filename, mimetype } = await input.file;
    return this.createAssetInternal(ctx, stream, filename, mimetype, ...);
}

private async createAssetInternal(
    ctx: RequestContext,
    stream: Stream,
    filename: string,
    mimetype: string,
): Promise<Asset> {
    const { assetPreviewStrategy, assetStorageStrategy } = this.configService.assetOptions;
    
    // 1. 生成文件名
    const sourceFileName = await this.getSourceFileName(ctx, filename);
    const previewFileName = await this.getPreviewFileName(ctx, sourceFileName);
    
    // 2. 写入源文件
    const sourceFileIdentifier = await assetStorageStrategy.writeFileFromStream(
        sourceFileName, stream
    );
    
    // 3. 读取源文件到 Buffer
    const sourceFile = await assetStorageStrategy.readFileToBuffer(sourceFileIdentifier);
    
    // 4. 生成预览图
    const preview = await assetPreviewStrategy.generatePreviewImage(
        ctx, mimetype, sourceFile
    );
    
    // 5. 写入预览文件
    const previewFileIdentifier = await assetStorageStrategy.writeFileFromBuffer(
        previewFileName, preview
    );
    
    // 6. 创建并保存 Asset 实体
    const asset = new Asset({
        type, width, height, fileSize, mimeType,
        source: sourceFileIdentifier,
        preview: previewFileIdentifier,
    });
    await this.channelService.assignToCurrentChannel(asset, ctx);
    return this.connection.getRepository(ctx, Asset).save(asset);
}
```

### 2.2 文件命名策略 (`HashedAssetNamingStrategy`)

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

### 2.3 本地存储策略 (`LocalAssetStorageStrategy`)

`packages/asset-server-plugin/src/config/local-asset-storage-strategy.ts`

```typescript
export class LocalAssetStorageStrategy implements AssetStorageStrategy {
    constructor(private uploadPath: string) {}
    
    async writeFileFromStream(fileName: string, data: ReadStream): Promise<string> {
        const filePath = path.join(this.uploadPath, fileName);
        await fs.ensureDir(path.dirname(filePath));
        // 写入文件...
        return this.filePathToIdentifier(filePath);
    }
    
    deleteFile(identifier: string): Promise<void> {
        return fs.unlink(this.identifierToFilePath(identifier));
    }
    
    private filePathToIdentifier(filePath: string): string {
        const deltaDirname = path.dirname(filePath).replace(this.uploadPath, '');
        return path.join(deltaDirname, path.basename(filePath)).replace(/^[\\/]+/, '');
    }
}
```

**关键点**：
- `identifier` 是相对于 `uploadPath` 的相对路径
- 删除时通过 `identifier` 定位到实际文件路径

---

## 三、预览图生成

### 3.1 SharpAssetPreviewStrategy (`packages/asset-server-plugin/src/config/sharp-asset-preview-strategy.ts`)

```typescript
export class SharpAssetPreviewStrategy implements AssetPreviewStrategy {
    async generatePreviewImage(ctx, mimeType, data): Promise<Buffer> {
        const assetType = getAssetType(mimeType);
        
        if (assetType === AssetType.IMAGE) {
            const image = sharp(data).rotate();
            const metadata = await image.metadata();
            
            // 超过最大尺寸则缩放
            if (maxWidth < width || maxHeight < height) {
                image.resize(maxWidth, maxHeight, { fit: 'inside' });
            }
            
            // 根据格式输出
            switch (metadata.format) {
                case 'jpeg': return image.jpeg(this.config.jpegOptions).toBuffer();
                case 'png': return image.png(this.config.pngOptions).toBuffer();
                case 'webp': return image.webp(this.config.webpOptions).toBuffer();
                // ...
            }
        } else {
            // 非图像文件：生成带MIME类型的通用预览图
            return this.generateBinaryFilePreview(mimeType);
        }
    }
}
```

---

## 四、引用计数与删除机制

### 4.1 引用检查 (`AssetService.findAssetUsages`)

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

**注意**：此方法只检查 `featuredAsset` 引用，不检查 OrderableAsset 关联表中的引用！

### 4.2 删除流程 (`AssetService.delete`)

`packages/core/src/service/services/asset.service.ts:375`

```typescript
async delete(ctx, ids, force = false, deleteFromAllChannels = false): Promise<DeletionResponse> {
    const assets = await this.connection.findByIdsInChannel(ctx, Asset, ids, ctx.channelId, {
        relations: ['channels'],
    });
    
    // 1. 统计引用
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

### 4.3 无条件删除 (`deleteUnconditional`)

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

### 4.4 数据库级联删除的作用

当 Asset 被删除时：
1. **OrderableAsset 记录自动删除**：因为 `OrderableAsset.asset` 设置了 `onDelete: 'CASCADE'`
2. **业务实体的 featuredAsset 自动置空**：因为设置了 `onDelete: 'SET NULL'`

这意味着即使 `findAssetUsages()` 没有检查 OrderableAsset 中的引用，数据库也会自动清理这些关联记录。

---

## 五、业务实体更新资源引用

### 5.1 AssetService.updateEntityAssets

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

## 六、实时图像转换与缓存

### 6.1 AssetServer 中间件 (`packages/asset-server-plugin/src/asset-server.ts`)

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
            }
            res.send(imageBuffer);
        }
    };
}
```

### 6.2 缓存文件名生成

```typescript
private getFileNameFromParameters(filePath: string, params: ImageTransformParameters): string {
    const { w, h, mode, preset, fpx, fpy, format, q } = params;
    
    let imageParamsString = '';
    if (w || h) {
        imageParamsString = `_transform_w${w}_h${height}_m${mode}`;
    } else if (preset) {
        imageParamsString = `_transform_pre_${preset}`;
    }
    // ... 添加 focalPoint、format、quality 参数
    
    if (imageParamsString !== '') {
        const imageParamHash = this.md5(imageParamsString);
        return path.join(this.cacheDir, this.addSuffix(decodedReqPath, imageParamHash, imageFormat));
    }
    return decodedReqPath;
}
```

**缓存文件位置**：`cache/{hash2}/{original-name}_{hash}.{ext}`

---

## 七、协同机制总结

### 7.1 上传时序

```
GraphQL Upload
      ↓
AssetService.create()
      ├─→ AssetNamingStrategy.generateSourceFileName()
      ├─→ AssetStorageStrategy.writeFileFromStream()  [写入源文件]
      ├─→ AssetPreviewStrategy.generatePreviewImage() [生成预览]
      ├─→ AssetStorageStrategy.writeFileFromBuffer()  [写入预览文件]
      └─→ 保存 Asset 实体到数据库
```

### 7.2 引用维护

1. **业务实体引用 Asset**：通过 `featuredAsset`（单值）或 `assets`（通过 OrderableAsset 关联表）
2. **无显式引用计数器**：删除前通过 `findAssetUsages()` 动态查询引用
3. **数据库级联**：
   - Asset 删除 → OrderableAsset 自动删除（CASCADE）
   - Asset 删除 → 业务实体 featuredAsset 自动置空（SET NULL）

### 7.3 文件存储标识协同

| 组件 | 存储字段 | 含义 | 生成者 |
|-----|---------|------|--------|
| Asset | source | 源文件标识符 | AssetStorageStrategy.writeFileFromStream() |
| Asset | preview | 预览文件标识符 | AssetStorageStrategy.writeFileFromBuffer() |
| 缓存 | - | 转换后图像 | AssetServer.generateTransformedImage() |

所有标识符都通过 `AssetStorageStrategy` 的 `filePathToIdentifier` 方法生成，删除时通过同一策略的 `identifierToFilePath` 反向解析。

### 7.4 删除时序

```
AssetService.delete(ids, force?)
      ↓
findAssetUsages() → 检查 featuredAsset 引用
      ↓
有引用 && !force → 返回 NOT_DELETED
      ↓
从 Channel 移除
      ↓
是最后一个 Channel？
      ├─ 是 → deleteUnconditional()
      │        ├─ 数据库删除 Asset
      │        ├─ 级联删除 OrderableAsset
      │        ├─ 级联置空 featuredAsset
      │        └─ AssetStorageStrategy.deleteFile(source + preview)
      └─ 否 → 仅从当前 Channel 移除
```

---

## 八、潜在问题与注意事项

1. **findAssetUsages 不检查 OrderableAsset**：
   - 只检查 `featuredAsset` 引用，不检查 OrderableAsset 关联表
   - 但数据库 CASCADE 会自动清理，所以功能上没问题
   - 只是删除前的警告信息可能不完整

2. **缓存文件不跟踪**：
   - 实时转换生成的缓存文件没有被 Asset 实体跟踪
   - 删除 Asset 时不会自动删除缓存文件
   - 缓存文件需要单独的清理机制

3. **多 Channel 共享**：
   - 一个 Asset 可以属于多个 Channel
   - 只有从所有 Channel 移除后才会真正删除文件
   - 这意味着文件存储是跨 Channel 共享的

4. **无引用计数列**：
   - 每次删除都需要执行 3 个查询（Product、ProductVariant、Collection）
   - 高并发删除场景可能有性能问题
