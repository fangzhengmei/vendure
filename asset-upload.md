# 资产服务上传与衍生图生成流程梳理

## 一、整体架构概览

Vendure 的资产服务分为两大核心模块：

1. **核心层 (`@vendure/core`)**：负责资产元数据管理、上传、数据库持久化
2. **插件层 (`@vendure/asset-server-plugin`)**：负责资产文件的 HTTP 服务、实时图像变换、缓存管理

```
GraphQL 上传请求
       ↓
AssetResolver (API 层)
       ↓
AssetService (业务逻辑层)
       ↓
┌─────────────────────────────────────┐
│  存储策略 (AssetStorageStrategy)    │  →  源文件/预览文件写入磁盘/S3
│  命名策略 (AssetNamingStrategy)     │  →  生成唯一文件名（含哈希目录）
│  预览策略 (AssetPreviewStrategy)    │  →  生成预览图 (Sharp)
└─────────────────────────────────────┘
       ↓
Asset 实体持久化到数据库
       ↓
HTTP 访问资产 (GET /assets/...)
       ↓
AssetServer (Express 中间件)
       ↓
┌─────────────────────────────────────┐
│  sendAsset()                        │  →  尝试直接读取缓存/源文件
│  ↓ (404 时触发)                     │
│  generateTransformedImage()         │  →  实时变换并缓存结果
│    ↓                                │
│    ImageTransformStrategy 管道      │  →  可扩展的变换参数校验
│    ↓                                │
│    transformImage() (Sharp)         │  →  实际图像处理
└─────────────────────────────────────┘
```

---

## 二、原图入库流程详解

### 2.1 入口：GraphQL Mutation

**文件**：`packages/core/src/api/resolvers/admin/asset.resolver.ts:48-63`

```typescript
@Transaction()
@Mutation()
@Allow(Permission.CreateCatalog, Permission.CreateAsset)
async createAssets(
    @Ctx() ctx: RequestContext,
    @Args() args: MutationCreateAssetsArgs,
): Promise<Array<Translated<Asset> | MimeTypeError>> {
    const assets = [];
    for (const input of args.input) {
        const asset = await this.assetService.create(ctx, input);
        assets.push(asset);
    }
    return assets;
}
```

上传采用 `GraphQL multipart request specification`，每个文件包含：
- `createReadStream` - 文件流
- `filename` - 原始文件名
- `mimetype` - MIME 类型

---

### 2.2 核心创建逻辑：`AssetService.create()`

**文件**：`packages/core/src/service/services/asset.service.ts:307-326`

```typescript
async create(ctx: RequestContext, input: CreateAssetInput): Promise<Translated<Asset> | MimeTypeError> {
    const { createReadStream, filename, mimetype } = await input.file;
    const { stream, errorPromise } = this.makeStreamGuard(createReadStream);
    
    // 竞争：创建过程 vs 流错误
    const result = await Promise.race([
        this.createAssetInternal(ctx, stream, filename, mimetype, input.customFields, input.translations),
        errorPromise,
    ]);
    
    // 后续：自定义字段关联、标签、事件发布...
}
```

**关键点**：
- 使用 `makeStreamGuard` 包装流，防止 `fs-capacitor` 写入流已销毁时 `createReadStream` 抛出同步错误
- `Promise.race` 确保流错误能中断创建流程

---

### 2.3 内部创建：`createAssetInternal()`

**文件**：`packages/core/src/service/services/asset.service.ts:565-646`

这是整个流程的核心，执行以下步骤：

#### 步骤 1：MIME 类型校验
```typescript
if (!this.validateMimeType(mimetype)) {
    return new MimeTypeError({ fileName: filename, mimeType: mimetype });
}
```
校验配置来自 `assetOptions.permittedFileTypes`。

#### 步骤 2：生成唯一文件名
```typescript
const sourceFileName = await this.getSourceFileName(ctx, filename);
const previewFileName = await this.getPreviewFileName(ctx, sourceFileName);
```

**命名策略** (`HashedAssetNamingStrategy`)：
- 源文件：`source/{hash2}/{originalName}`
- 预览文件：`preview/{hash2}/{originalName}_preview.{ext}`
- `hash2` 是文件名 MD5 的前 2 位，用于分散文件到 256 个子目录，避免单目录文件过多

**冲突处理**：循环调用 `generateNameFn` 直到 `fileExists()` 返回 `false`。

#### 步骤 3：写入源文件
```typescript
const sourceFileIdentifier = await assetStorageStrategy.writeFileFromStream(sourceFileName, stream);
const sourceFile = await assetStorageStrategy.readFileToBuffer(sourceFileIdentifier);
```

#### 步骤 4：生成预览图
```typescript
preview = await assetPreviewStrategy.generatePreviewImage(ctx, mimetype, sourceFile);
```

**SharpAssetPreviewStrategy** 逻辑：
- 图片类型：读取元数据 → 超过 maxWidth/maxHeight 则 resize → 按原格式输出
- 非图片类型：生成带 MIME 文字水印的通用文件图标
- SVG 特殊处理：栅格化为位图

#### 步骤 5：写入预览文件
```typescript
const previewFileIdentifier = await assetStorageStrategy.writeFileFromBuffer(previewFileName, preview);
```

#### 步骤 6：获取尺寸信息
```typescript
const { width, height } = this.getDimensions(type === AssetType.IMAGE ? sourceFile : preview);
```

#### 步骤 7：创建 Asset 实体并持久化
```typescript
const asset = new Asset({
    type, width, height, fileSize: sourceFile.byteLength,
    mimeType: mimetype, source: sourceFileIdentifier,
    preview: previewFileIdentifier, focalPoint: null, customFields,
});
await this.channelService.assignToCurrentChannel(asset, ctx);
const savedAsset = await this.connection.getRepository(ctx, Asset).save(asset);
```

#### 步骤 8：保存翻译信息
```typescript
const assetTranslations = [
    new AssetTranslation({
        languageCode: ctx.languageCode,
        name: defaultName,  // 原始文件名
        base: savedAsset,
    }),
];
```

---

### 2.4 Asset 实体结构

**文件**：`packages/core/src/entity/asset/asset.entity.ts`

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `AssetType` | IMAGE / VIDEO / BINARY |
| `mimeType` | `string` | 如 `image/jpeg` |
| `width` / `height` | `number` | 像素尺寸 |
| `fileSize` | `number` | 字节数 |
| `source` | `string` | 源文件标识符（路径或 URL） |
| `preview` | `string` | 预览文件标识符 |
| `focalPoint` | `{x, y}` | 焦点坐标（0-1 归一化） |
| `tags` | `Tag[]` | 标签关联 |
| `channels` | `Channel[]` | 多渠道关联 |

---

## 三、变换流水线（衍生图生成）

### 3.1 AssetServer 初始化

**文件**：`packages/asset-server-plugin/src/plugin.ts:252-273`

```typescript
configure(consumer: MiddlewareConsumer) {
    const presets = [...this.defaultPresets];  // tiny, thumb, small, medium, large
    // 合并用户自定义 presets
    const assetServerRouter = this.assetServer.createAssetServer({
        presets,
        imageTransformStrategies: this.getImageTransformStrategyArray(),
    });
    consumer.apply(assetServerRouter).forRoutes(this.options.route);
}
```

**默认预设**：
```
tiny   → 50x50 crop
thumb  → 150x150 crop
small  → 300x300 resize
medium → 500x500 resize
large  → 800x800 resize
```

---

### 3.2 Express 中间件链

**文件**：`packages/asset-server-plugin/src/asset-server.ts:66-75`

```typescript
createAssetServer(serverConfig: { ... }): express.Router {
    this.presets = serverConfig.presets;
    this.imageTransformStrategies = serverConfig.imageTransformStrategies;
    const assetServer = express.Router();
    // 关键：两个中间件顺序执行，第一个 404 时第二个接管
    assetServer.use(this.sendAsset(), this.generateTransformedImage());
    return assetServer;
}
```

---

### 3.3 第一阶段：`sendAsset()` - 尝试直接返回

**文件**：`packages/asset-server-plugin/src/asset-server.ts:80-107`

```typescript
private sendAsset() {
    return async (req: Request, res: Response, next: NextFunction) => {
        // 1. 解析变换参数
        const params = await this.getImageTransformParameters(req);
        
        // 2. 生成缓存键文件名
        const key = this.getFileNameFromParameters(req.path, params);
        
        try {
            // 3. 尝试直接读取（可能是缓存文件，也可能是源文件）
            const file = await this.assetStorageStrategy.readFileToBuffer(key);
            // 4. 返回响应，设置 Cache-Control
            res.contentType(mimeType);
            res.setHeader('Cache-Control', this.cacheHeader);
            res.send(file);
        } catch (e) {
            // 5. 文件不存在 → 触发下一个中间件
            const err = new Error('File not found');
            (err as any).status = 404;
            return next(err);
        }
    };
}
```

---

### 3.4 第二阶段：`generateTransformedImage()` - 实时变换

**文件**：`packages/asset-server-plugin/src/asset-server.ts:114-153`

```typescript
private generateTransformedImage() {
    return async (err: any, req: Request, res: Response, next: NextFunction) => {
        // 仅处理 404 错误
        if (err && (err.status === 404 || err.statusCode === 404)) {
            // 1. 读取源文件
            file = await this.assetStorageStrategy.readFileToBuffer(decodedReqPath);
            
            // 2. 解析变换参数（含策略管道）
            const parameters = await this.getImageTransformParameters(req);
            
            // 3. 执行图像变换
            const image = await transformImage(file, parameters);
            const imageBuffer = await image.toBuffer();
            
            // 4. 生成缓存文件名
            const cachedFileName = this.getFileNameFromParameters(req.path, parameters);
            
            // 5. 写入缓存（除非 cache=false）
            if (!req.query.cache || req.query.cache === 'true') {
                await this.assetStorageStrategy.writeFileFromBuffer(cachedFileName, imageBuffer);
            }
            
            // 6. 返回变换结果
            res.send(imageBuffer);
            return;
        }
        next();
    };
}
```

---

### 3.5 变换参数解析管道

**文件**：`packages/asset-server-plugin/src/asset-server.ts:155-188`

```typescript
private async getImageTransformParameters(req: Request): Promise<ImageTransformParameters> {
    // 1. 基础参数解析
    let parameters = this.getInitialImageTransformParameters(req.query as any);
    
    // 2. ImageTransformStrategy 管道（可扩展）
    for (const strategy of this.imageTransformStrategies) {
        parameters = await strategy.getImageTransformParameters({
            req,
            input: { ...parameters },
            availablePresets: this.presets,
        });
    }
    
    // 3. 预设解析（覆盖 width/height/mode）
    if (parameters.preset) {
        const matchingPreset = this.presets.find(p => p.name === parameters.preset);
        if (matchingPreset) {
            targetWidth = matchingPreset.width;
            targetHeight = matchingPreset.height;
            targetMode = matchingPreset.mode;
        }
    }
    
    return { ...parameters, width: targetWidth, height: targetHeight, mode: targetMode };
}
```

**基础参数**：
- `w` / `h` - 目标宽高
- `mode` - `crop`（默认，熵裁剪）或 `resize`（等比缩放）
- `q` - 质量 (1-100)
- `format` - 输出格式 (jpg/png/webp/avif)
- `fpx` / `fpy` - 焦点坐标（0-1）
- `preset` - 预设名称
- `cache` - 是否缓存（默认 true）

---

### 3.6 实际图像变换：`transformImage()`

**文件**：`packages/asset-server-plugin/src/transform-image.ts:14-51`

```typescript
export async function transformImage(
    originalImage: Buffer,
    parameters: ImageTransformParameters,
): Promise<sharp.Sharp> {
    const { width, height, mode, format } = parameters;
    
    const image = sharp(originalImage).rotate();  // 自动旋转（根据 EXIF）
    
    // 1. 格式转换 + 质量设置
    await applyFormat(image, parameters.format, parameters.quality);
    
    // 2. 焦点裁剪（如果指定了 fpx/fpy）
    if (parameters.fpx && parameters.fpy && width && height && mode === 'crop') {
        const metadata = await image.metadata();
        const { width: resizedWidth, height: resizedHeight, region } = resizeToFocalPoint(
            { w: metadata.width, h: metadata.height },
            { w: width, h: height },
            { x: parameters.fpx * metadata.width, y: parameters.fpy * metadata.height },
        );
        return image.resize(resizedWidth, resizedHeight).extract(region);
    }
    
    // 3. 普通 resize / crop
    const options: ResizeOptions = {};
    if (mode === 'crop') {
        options.position = sharp.strategy.entropy;  // 熵裁剪：保留最"有趣"区域
    } else {
        options.fit = 'inside';  // 等比缩放，不超过目标尺寸
    }
    
    return image.resize(width, height, options);
}
```

---

## 四、缓存机制详解

### 4.1 缓存键生成算法

**文件**：`packages/asset-server-plugin/src/asset-server.ts:214-248`

```typescript
private getFileNameFromParameters(filePath: string, params: ImageTransformParameters): string {
    const { width: w, height: h, mode, preset, fpx, fpy, format, quality: q } = params;
    
    // 构建参数字符串
    let imageParamsString = '';
    if (w || h) {
        imageParamsString = `_transform_w${width}_h${height}_m${mode}`;
    } else if (preset) {
        imageParamsString = `_transform_pre_${preset}`;
    }
    
    // 附加参数
    if (fpx && fpy) imageParamsString += `_fpx${fpx}_fpy${fpy}`;
    if (format)     imageParamsString += `.${format}`;
    if (q)          imageParamsString += `_q${q}`;
    
    // 无变换 → 直接返回原路径
    if (imageParamsString === '') {
        return decodedReqPath;
    }
    
    // 有变换 → 生成 MD5 哈希，存入 cache/ 目录
    const imageParamHash = this.md5(imageParamsString);
    return path.join(this.cacheDir, this.addSuffix(decodedReqPath, imageParamHash, imageFormat));
}
```

**缓存键示例**：
```
原路径：source/ab/product.jpg?w=500&h=300&mode=crop&q=80
缓存键：cache/source/ab/product_<md5("_transform_w500_h300_mcrop_q80")>.jpg
```

### 4.2 缓存命中流程

```
请求 /assets/source/ab/product.jpg?w=500&h=300
       ↓
1. 解析参数 → { w: 500, h: 300, mode: 'crop' }
       ↓
2. 生成缓存键 → cache/source/ab/product_<hash>.jpg
       ↓
3. 调用 assetStorageStrategy.readFileToBuffer(缓存键)
       ├─ 命中 → 直接返回，设置 Cache-Control
       └─ 未命中 → 进入变换流程
                ↓
                4. 读取源文件 source/ab/product.jpg
                ↓
                5. sharp 变换为 500x300
                ↓
                6. 写入缓存键（除非 cache=false）
                ↓
                7. 返回变换结果
```

### 4.3 缓存目录结构

```
asset-upload-dir/
├── source/              # 原图目录
│   ├── 00/
│   ├── 01/
│   └── ... (256 个子目录)
├── preview/             # 预览图目录
│   ├── 00/
│   └── ...
└── cache/               # 变换缓存目录
    └── source/
        ├── 00/
        │   ├── product_<hash1>.jpg   # w=500 版本
        │   └── product_<hash2>.jpg   # w=300&q=75 版本
        └── ...
```

### 4.4 HTTP 缓存头

**文件**：`packages/asset-server-plugin/src/asset-server.ts:45-57`

默认 `Cache-Control: public, max-age=15552000`（6 个月），可通过 `cacheHeader` 配置。

---

## 五、关键策略接口

### 5.1 AssetStorageStrategy（存储策略）

**文件**：`packages/core/src/config/asset-storage-strategy/asset-storage-strategy.ts`

```typescript
interface AssetStorageStrategy {
    writeFileFromBuffer(fileName: string, data: Buffer): Promise<string>;
    writeFileFromStream(fileName: string, data: Stream): Promise<string>;
    readFileToBuffer(identifier: string): Promise<Buffer>;
    readFileToStream(identifier: string): Promise<Stream>;
    deleteFile(identifier: string): Promise<void>;
    fileExists(fileName: string): Promise<boolean>;
    toAbsoluteUrl?(request: Request, identifier: string): string;
}
```

**内置实现**：
- `LocalAssetStorageStrategy` - 本地文件系统
- `S3AssetStorageStrategy` - AWS S3

### 5.2 AssetPreviewStrategy（预览策略）

**文件**：`packages/core/src/config/asset-preview-strategy/asset-preview-strategy.ts`

```typescript
interface AssetPreviewStrategy {
    generatePreviewImage(ctx: RequestContext, mimeType: string, data: Buffer): Promise<Buffer>;
}
```

### 5.3 ImageTransformStrategy（变换策略）

**文件**：`packages/asset-server-plugin/src/config/image-transform-strategy.ts`

```typescript
interface ImageTransformStrategy {
    getImageTransformParameters(
        args: GetImageTransformParametersArgs,
    ): Promise<ImageTransformParameters> | ImageTransformParameters;
}
```

**内置实现**：`PresetOnlyStrategy` - 仅允许使用预设，防止滥用变换 API。

---

## 六、完整时序图（一次上传 + 一次变换访问）

```
客户端                    GraphQL API             AssetService          存储策略        数据库
   │  createAssets mutation    │                      │                  │             │
   ├──────────────────────────>│                      │                  │             │
   │                          │  create(ctx, input)   │                  │             │
   │                          ├─────────────────────>│                  │             │
   │                          │                      │  MIME 校验       │             │
   │                          │                      ├───────────┐      │             │
   │                          │                      │           │      │             │
   │                          │                      │<──────────┘      │             │
   │                          │                      │  生成文件名       │             │
   │                          │                      ├───────────┐      │             │
   │                          │                      │           │      │             │
   │                          │                      │<──────────┘      │             │
   │                          │                      │  writeFileFromStream         │
   │                          │                      ├────────────────────────────>│             │
   │                          │                      │                  │ 写入源文件  │             │
   │                          │                      │<────────────────────────────┤             │
   │                          │                      │                  │ 返回标识    │             │
   │                          │                      │  readFileToBuffer            │
   │                          │                      ├────────────────────────────>│             │
   │                          │                      │                  │ 读取源文件  │             │
   │                          │                      │<────────────────────────────┤             │
   │                          │                      │  generatePreviewImage (Sharp)             │
   │                          │                      ├────────────┐                 │             │
   │                          │                      │            │                 │             │
   │                          │                      │<───────────┘                 │             │
   │                          │                      │  writeFileFromBuffer         │
   │                          │                      ├────────────────────────────>│             │
   │                          │                      │                  │ 写入预览   │             │
   │                          │                      │<────────────────────────────┤             │
   │                          │                      │  save(Asset)                                   │
   │                          │                      ├───────────────────────────────────────────>│
   │                          │                      │                                   插入记录  │
   │                          │                      │<───────────────────────────────────────────┤
   │                          │  返回 Asset          │                  │             │             │
   │<──────────────────────────┤<─────────────────────┤                  │             │             │
   │                          │                      │                  │             │             │
   │   GET /assets/xx.jpg?w=500                      │                  │             │             │
   ├───────────────────────────────────────────────────────────────────────────────────────────────>│ AssetServer
   │                          │                      │                  │             │             │ sendAsset()
   │                          │                      │                  │  读取缓存?  │             │
   │                          │                      │                  ├────────────>│             │
   │                          │                      │                  │  404 未命中 │             │
   │                          │                      │                  │<────────────┤             │
   │                          │                      │                  │             │             │ generateTransformedImage()
   │                          │                      │                  │  读取源文件  │             │
   │                          │                      │                  ├────────────>│             │
   │                          │                      │                  │<────────────┤             │
   │                          │                      │                  │  sharp 变换  │             │
   │                          │                      │                  ├───────────┐ │             │
   │                          │                      │                  │           │ │             │
   │                          │                      │                  │<──────────┘ │             │
   │                          │                      │                  │  写入缓存   │             │
   │                          │                      │                  ├────────────>│             │
   │                          │                      │                  │<────────────┤             │
   │<───────────────────────────────────────────────────────────────────────────────────────────────┤ 返回变换结果
```

---

## 七、关键设计要点

1. **策略模式的广泛应用**：存储、命名、预览、变换均可自定义替换，支持本地/S3/其他云存储。

2. **哈希目录分散**：通过文件名 MD5 前 2 位创建 256 个子目录，避免单目录文件过多导致的性能问题。

3. **中间件链式错误处理**：第一个中间件 404 时，第二个中间件接管进行实时变换，这种设计使得缓存命中路径和未命中路径无缝衔接。

4. **缓存键哈希化**：变换参数先序列化为字符串再做 MD5，确保参数顺序、格式差异不会导致同一变换生成不同缓存。

5. **焦点裁剪算法**：先等比缩放使目标尺寸"覆盖"裁剪区域，再基于焦点坐标计算提取区域，保证焦点始终在裁剪结果中心。

6. **变换策略管道**：支持多个 `ImageTransformStrategy` 顺序执行，可用于参数校验、权限控制、预设强制等场景。

7. **流错误防护**：`makeStreamGuard` 处理 `fs-capacitor` 的边界情况，确保流错误能被正确捕获。
