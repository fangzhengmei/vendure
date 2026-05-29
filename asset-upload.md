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

## 四、缓存链路深度剖析

### 4.1 核心问题：两个中间件如何共享同一个缓存键

`sendAsset()` 和 `generateTransformedImage()` 是两个独立的 Express 中间件，它们通过**两次独立调用同一组函数**来保证缓存键一致：

```
sendAsset()                             generateTransformedImage()
    │                                         │
    ├─ getImageTransformParameters(req) ──────├─ getImageTransformParameters(req)
    │    ↓                                    │    ↓
    │  同一个 req，同样的 query params         │  同一个 req，同样的 query params
    │  同样的 strategy 管道，确定性输出         │  同样的 strategy 管道，确定性输出
    │    ↓                                    │    ↓
    ├─ getFileNameFromParameters(path, params)├─ getFileNameFromParameters(path, params)
    │    ↓                                    │    ↓
    │  key = "cache/source/ab/xx_<hash>.jpg"  │  cachedFileName = "cache/source/ab/xx_<hash>.jpg"
    │    ↓                                    │    ↓
    ├─ readFileToBuffer(key)                  ├─ writeFileFromBuffer(cachedFileName, buffer)
    │    ├─ 命中 → 返回                        │
    │    └─ 404 → next(err) ──────────────────┤
    │                                         │  (key === cachedFileName，写入的文件即下次命中的文件)
```

**关键保证**：由于 `req` 对象在两个中间件间共享，`req.query` 不变，且 `getImageTransformParameters()` 是纯函数（对相同输入产生相同输出），因此两次调用生成的缓存键**一定相同**。这是整个缓存链路能够闭合的根本前提。

---

### 4.2 缓存键生成算法逐行解析

**文件**：`packages/asset-server-plugin/src/asset-server.ts:214-248`

```typescript
private getFileNameFromParameters(filePath: string, params: ImageTransformParameters): string {
    const { width: w, height: h, mode, preset, fpx, fpy, format, quality: q } = params;

    const focalPoint = fpx && fpy ? `_fpx${fpx}_fpy${fpy}` : '';
    const quality = q ? `_q${q}` : '';
    const imageFormat = getValidFormat(format);

    let imageParamsString = '';
    if (w || h) {
        // 分支 A：有明确宽高 → 用尺寸 + 模式构建
        const width = w || '';
        const height = h || '';
        imageParamsString = `_transform_w${width}_h${height}_m${mode}`;
    } else if (preset) {
        // 分支 B：只有 preset，无宽高 → 用预设名构建
        if (this.presets && !!this.presets.find(p => p.name === preset)) {
            imageParamsString = `_transform_pre_${preset}`;
        }
    }
    // 分支 C：无宽高也无 preset → imageParamsString 为空

    if (focalPoint) imageParamsString += focalPoint;
    if (imageFormat) imageParamsString += imageFormat;
    if (quality) imageParamsString += quality;

    const decodedReqPath = this.sanitizeFilePath(filePath);
    if (imageParamsString !== '') {
        const imageParamHash = this.md5(imageParamsString);
        return path.join(this.cacheDir, this.addSuffix(decodedReqPath, imageParamHash, imageFormat));
    } else {
        return decodedReqPath;  // 无变换 → 返回原路径，不做缓存
    }
}
```

**三个分支的命中判定**：

| 场景 | 分支 | 缓存键 | 说明 |
|------|------|--------|------|
| `?w=500&h=300` | A | `cache/.../xx_<md5("_transform_w500_h300_mcrop")>.jpg` | 宽高明确，直接 MD5 |
| `?preset=medium`（经解析后 w=500,h=500） | A | `cache/.../xx_<md5("_transform_w500_h500_mresize")>.jpg` | preset 被解析为宽高，走分支 A |
| `?w=500`（只指定宽） | A | `cache/.../xx_<md5("_transform_w500_h_mcrop")>.jpg` | h 为空字符串 |
| 无任何参数 | C | `source/ab/xx.jpg`（原路径） | 不走缓存，直接读源文件/预览文件 |

**重要**：`preset` 参数在 `getImageTransformParameters()` 中已被解析为具体的 `width`/`height`/`mode`，所以 `getFileNameFromParameters()` 拿到的 `w`/`h` 总是有值的（只要指定了有效 preset），**永远不会走分支 B**。分支 B 仅在 `w` 和 `h` 都为 `undefined`、但 `preset` 仍有值时才可能触发——这在正常流程中不会发生。

**缓存键示例**：
```
URL：/assets/source/ab/product.jpg?w=500&h=300&mode=crop&q=80
参数字符串："_transform_w500_h300_mcrop_q80"
MD5 哈希：如 "a1b2c3d4e5f6..."
缓存键："cache/source/ab/product_a1b2c3d4e5f6....jpg"
```

---

### 4.3 首次未命中 → 写回缓存：完整流程

以请求 `/assets/source/ab/product.jpg?w=500&h=300` 为例，首次访问时缓存文件尚不存在：

#### 阶段一：`sendAsset()` 尝试读取

```
1. getImageTransformParameters(req)
   → getInitialImageTransformParameters({ w: '500', h: '300', mode: undefined })
   → { width: 500, height: 300, mode: 'crop', ... }    // mode 默认为 'crop'
   → 无 ImageTransformStrategy → 参数不变
   → 无 preset → 参数不变
   → 最终: { width: 500, height: 300, mode: 'crop', quality: undefined, format: undefined, ... }

2. getFileNameFromParameters('/source/ab/product.jpg', params)
   → w=500, h=300 → 分支 A
   → imageParamsString = "_transform_w500_h300_mcrop"
   → 无 focalPoint、format、quality → 不追加
   → md5("_transform_w500_h300_mcrop") = "<hash>"
   → key = "cache/source/ab/product_<hash>.jpg"

3. assetStorageStrategy.readFileToBuffer("cache/source/ab/product_<hash>.jpg")
   → 抛出异常！文件不存在

4. 构造 404 错误 → next(err)  ──→  进入阶段二
```

#### 阶段二：`generateTransformedImage()` 接管

```
5. 检查 err.status === 404 ✓

6. assetStorageStrategy.readFileToBuffer("/source/ab/product.jpg")  ← 注意：读的是源文件
   → 成功，拿到原图 Buffer

7. getImageTransformParameters(req)  ← 同一个 req，产出与阶段一完全相同的参数
   → { width: 500, height: 300, mode: 'crop', ... }

8. transformImage(原图Buffer, params)  ← Sharp 执行 resize
   → 变换后的 sharp 对象

9. image.toBuffer()  → imageBuffer

10. getFileNameFromParameters(req.path, params)
    → cachedFileName = "cache/source/ab/product_<hash>.jpg"
    → 与阶段一的 key 完全一致！

11. 写入缓存判定：
    !req.query.cache || req.query.cache === 'true'
    → 无 cache 参数 → !undefined = true → 写入！

12. assetStorageStrategy.writeFileFromBuffer(cachedFileName, imageBuffer)
    → 缓存文件落盘

13. res.send(imageBuffer)  → 返回变换结果
```

**写入后的文件系统状态**：
```
asset-upload-dir/
└── cache/
    └── source/
        └── ab/
            └── product_<hash>.jpg   ← 新写入的缓存文件
```

---

### 4.4 同参数再次访问：入口短路命中

同一参数的第二次请求 `/assets/source/ab/product.jpg?w=500&h=300`：

```
1. sendAsset() 执行
2. getImageTransformParameters(req) → { width: 500, height: 300, mode: 'crop', ... }
3. getFileNameFromParameters(...) → "cache/source/ab/product_<hash>.jpg"
4. assetStorageStrategy.readFileToBuffer("cache/source/ab/product_<hash>.jpg")
   → 成功！文件已在上次请求中写入
5. res.contentType(mimeType)
6. res.setHeader('Cache-Control', 'public, max-age=15552000')
7. res.send(file)
8. ← 返回，不进入 generateTransformedImage()
```

**短路命中路径**：只经过 `sendAsset()` 一个中间件，完全不触发 `generateTransformedImage()`。无需读取源文件、无需 Sharp 变换、无需写入缓存。这就是"入口短路"的含义——缓存命中时，请求在第一个中间件就已经终结。

**性能对比**：
```
首次请求（未命中）：
  readFileToBuffer(缓存键) → 失败
  + readFileToBuffer(源文件) → 成功
  + sharp 变换
  + writeFileFromBuffer(缓存键)
  = 2次存储读取 + 1次CPU密集变换 + 1次存储写入

二次请求（命中）：
  readFileToBuffer(缓存键) → 成功
  = 1次存储读取
```

---

### 4.5 `cache=false` 的行为与陷阱

#### 写入判定逻辑

**文件**：`packages/asset-server-plugin/src/asset-server.ts:132`

```typescript
if (!req.query.cache || req.query.cache === 'true') {
    await this.assetStorageStrategy.writeFileFromBuffer(cachedFileName, imageBuffer);
}
```

| `req.query.cache` 值 | `!req.query.cache` | `=== 'true'` | 结果 |
|---|---|---|---|
| `undefined`（未传） | `true` | — | **写入缓存** |
| `'true'` | `false` | `true` | **写入缓存** |
| `'false'` | `false` | `false` | **不写入缓存** |
| `'1'` | `false` | `false` | **不写入缓存** |

#### `cache` 不参与缓存键计算

`cache` 参数**不属于** `ImageTransformParameters` 接口，它只在 `generateTransformedImage()` 中作为写入条件的判断依据。因此：

- `?w=500` 和 `?w=500&cache=false` 生成**完全相同的缓存键**
- `cache=false` 仅控制当前请求是否写入缓存文件，**不影响读取**

#### 连续请求的行为矩阵

| 请求序号 | URL | 缓存文件是否存在 | 写入？ | 结果 |
|----------|-----|------------------|--------|------|
| 1 | `?w=500&cache=false` | 否 | 不写入 | 实时变换，不缓存 |
| 2 | `?w=500&cache=false` | 否 | 不写入 | 再次实时变换，再次不缓存 |
| 3 | `?w=500`（无 cache 参数） | 否 | 写入 | 实时变换，缓存落盘 |
| 4 | `?w=500&cache=false` | **是** | 不写入 | **命中缓存！** 直接返回 |

**关键陷阱**：第 4 次请求虽然带了 `cache=false`，但因为第 3 次请求已经写入了缓存文件，`sendAsset()` 会直接命中并返回。`cache=false` 只阻止写入，**不阻止读取已有的缓存文件**。

这意味着 `cache=false` 的语义是"本次不要把变换结果写入缓存"，而不是"绕过缓存"。

#### `cache=false` 的典型用例

- **一次性预览**：管理员在后台预览某个变换效果，不希望污染缓存空间
- **调试**：开发者临时查看变换效果，不希望影响生产缓存
- **注意**：如果同一参数组合的缓存已存在，`cache=false` 并不会阻止命中该缓存

---

### 4.6 PresetOnlyStrategy 下的缓存命中判定

**文件**：`packages/asset-server-plugin/src/config/preset-only-strategy.ts`

#### 策略的核心行为

`PresetOnlyStrategy.getImageTransformParameters()` 对参数做了**强制归一化**：

```typescript
getImageTransformParameters({ input, availablePresets }) {
    // 1. 强制使用预设：无 preset 则用 defaultPreset
    const presetName = input.preset ?? this.options.defaultPreset;
    const matchingPreset = availablePresets.find(p => p.name === presetName);
    if (!matchingPreset) {
        throw new Error(`Preset "${presetName}" not found`);  // 无效预设 → 抛错
    }

    // 2. 宽高模式强制覆盖为预设值，忽略 URL 中的 w/h/mode
    // 3. quality/format 仅保留允许的值，否则置 undefined
    const permittedQuality = this.options.permittedQuality ?? [0, 50, 75, 85, 95];
    const permittedFormats = this.options.permittedFormats ?? ['jpg', 'webp', 'avif'];
    const quality = input.quality && permittedQuality.includes(input.quality) ? input.quality : undefined;
    const format = input.format && permittedFormats.includes(input.format) ? input.format : undefined;

    // 4. 焦点：默认禁用
    const fpx = this.options.allowFocalPoint ? input.fpx : undefined;
    const fpy = this.options.allowFocalPoint ? input.fpy : undefined;

    return {
        width: matchingPreset.width,
        height: matchingPreset.height,
        mode: matchingPreset.mode,
        quality, format, fpx, fpy,
        preset: input.preset,   // 保留原始 preset 名
    };
}
```

#### 参数归一化对缓存键的影响

由于 `PresetOnlyStrategy` 将 `width`/`height`/`mode` 强制设为预设值，以下 URL 会产生**相同的参数**，因而**命中同一缓存键**：

```
?preset=medium                → w=500, h=500, mode=resize
?w=999&h=888&preset=medium    → w=500, h=500, mode=resize  (w/h 被忽略)
?preset=medium&mode=crop      → w=500, h=500, mode=resize  (mode 被忽略)
?preset=medium&q=75           → w=500, h=500, mode=resize, q=75 (如果 75 在 permittedQuality 中)
```

它们都会生成缓存键 `cache/.../xx_<md5("_transform_w500_h500_mresize_q75")>.jpg`。

#### 无效参数的过滤与缓存

`PresetOnlyStrategy` 会将不在白名单中的 `quality`/`format` 置为 `undefined`：

```
?preset=medium&q=42     → q=42 不在 permittedQuality [0,50,75,85,95] → quality=undefined
?preset=medium&q=75     → q=75 在白名单中 → quality=75
?preset=medium&format=bmp → bmp 不在 permittedFormats → format=undefined
```

因此：
- `?preset=medium&q=42` 和 `?preset=medium`（不传 q）生成**相同的缓存键**
- `?preset=medium&q=75` 生成**不同的缓存键**（包含 `_q75`）

#### 无效预设的错误短路

```
请求：/assets/xx.jpg?preset=nonexistent
       ↓
sendAsset():
  getImageTransformParameters(req)
    → PresetOnlyStrategy: find('nonexistent') → undefined → throw Error
  → catch (e):
      res.status(400).send('Invalid parameters')
      return  ← 请求在此终止！
```

当 `ImageTransformStrategy` 抛错时，`sendAsset()` 的 try-catch（第 83-88 行）直接返回 HTTP 400。**请求不会进入 `generateTransformedImage()`，不涉及任何缓存读写。**

#### 无预设参数时的 defaultPreset 行为

```
请求：/assets/xx.jpg  （完全无变换参数）
       ↓
PresetOnlyStrategy:
  input.preset = undefined → presetName = this.options.defaultPreset
  → 例如 defaultPreset = 'thumbnail' → w=150, h=150, mode=crop
       ↓
getFileNameFromParameters():
  w=150, h=150 → 分支 A
  → imageParamsString = "_transform_w150_h150_mcrop"
  → key = "cache/.../xx_<hash>.jpg"
       ↓
sendAsset():
  readFileToBuffer(key) → 首次不存在 → 404 → generateTransformedImage()
  → 变换 → 写入缓存 → 返回
```

**注意**：配置了 `PresetOnlyStrategy` 后，即使 URL 完全不带参数，也会因为 `defaultPreset` 被强制应用变换，而不是直接返回原图。

---

### 4.7 完整缓存命中判定决策树

#### 正常路径（无错误）

```
请求到达 sendAsset()
       ↓
getImageTransformParameters(req)
       ├─ ImageTransformStrategy 抛错 → 400 终止（不涉及缓存）
       └─ 成功，返回 params
              ↓
getFileNameFromParameters(req.path, params)
       ├─ params 无 w/h/preset → key = 原路径（不经过缓存目录）
       │      ↓
       │  readFileToBuffer(原路径)
       │      ├─ 命中 → 返回源/预览文件
       │      └─ 未命中 → 404 → generateTransformedImage()
       │                      → 读源文件失败 → 404 终止
       │
       └─ params 有 w/h 或 preset → key = cache/.../<hash>.jpg
              ↓
         readFileToBuffer(缓存键)
              ├─ 命中 → 直接返回缓存文件 ← 短路！不进入下一个中间件
              └─ 未命中 → 404 → generateTransformedImage()
                              ├─ 读源文件失败 → 404 终止
                              ├─ 变换失败 → 500 终止
                              └─ 变换成功
                                    ├─ cache=false → 不写入，返回结果
                                    └─ cache=true/未传 → 写入缓存键，返回结果
                                           ↓
                                    下次同参数请求 → sendAsset() 短路命中
```

#### sendAsset 读取失败后的 404 异常分流机制

**文件**：`packages/asset-server-plugin/src/asset-server.ts:101-104, 114-116`

sendAsset() 捕获 `readFileToBuffer` 的异常后，不是直接返回 404，而是**构造一个新的 Error 并设置 status 属性**，通过 Express 错误中间件机制传递：

```typescript
// sendAsset() 中
catch (e: any) {
    const err = new Error('File not found');
    (err as any).status = 404;   // 关键：给 error 对象附加 status
    return next(err);            // 传递给错误处理中间件
}

// generateTransformedImage() 中
if (err && (err.status === 404 || err.statusCode === 404)) {
    // 只处理 404 错误，其他错误直接 pass
}
```

**为什么要这样设计？**
- Express 的 `next(err)` 会跳过所有普通中间件，只进入错误处理中间件
- `generateTransformedImage()` 作为错误处理中间件（4 个参数：err, req, res, next），只处理 `status === 404` 的情况
- 这是一种**异常驱动的控制流**：用 404 错误作为"缓存未命中"的信号，触发实时变换流程

---

### 4.8 三种错误路径的响应头与缓存影响

| 错误类型 | 触发场景 | 响应代码 | 响应体 | Cache-Control | CSP 头 | Content-Type | 浏览器/CDN 缓存行为 |
|----------|---------|---------|--------|---------------|--------|--------------|---------------------|
| **400** | ImageTransformStrategy 抛错<br>（如无效 preset） | 400 | `Invalid parameters` | ❌ 无 | ❌ 无 | ❌ 无 | 通常不缓存 |
| **404a** | sendAsset() 中 `readFileToBuffer` 失败<br>（但**这不是最终响应！**会进入变换流程） | — | — | — | — | — | — |
| **404b** | generateTransformedImage() 中读源文件失败 | 404 | `Resource not found` | ❌ 无 | ❌ 无 | ❌ 无 | 通常不缓存 |
| **500** | sharp 变换异常、文件解码失败等 | 500 | `An error occurred when generating the image` | ❌ 无 | ❌ 无 | ❌ 无 | 通常不缓存 |
| **200（命中）** | sendAsset() 缓存命中 | 200 | 文件内容 | ✅ 有 | ✅ 有 | ✅ 有 | 按 max-age 缓存 |
| **200（变换）** | generateTransformedImage() 变换成功 | 200 | 文件内容 | ❌ 无 | ✅ 有 | ✅ 有 | 可能不缓存 |

**关键发现**：
1. **所有错误路径（400/404/500）都不设置 Cache-Control 头**，也不设置 CSP 头，浏览器/CDN 通常不会缓存错误响应
2. **实时变换成功的 200 响应也没有 Cache-Control 头**，只有缓存命中的 200 响应才有
3. 404a（sendAsset 读缓存失败）**不是最终响应**，它只是一个控制流信号，真正的响应由 generateTransformedImage() 决定

---

### 4.9 fpx/fpy = 0 的特殊行为

#### 参数解析阶段

**文件**：`packages/asset-server-plugin/src/asset-server.ts:198-199`

```typescript
const fpx = +queryParams.fpx || undefined;
const fpy = +queryParams.fpy || undefined;
```

JavaScript 的 `||` 运算符会把 `0` 当作 falsy 值：
- `?fpx=0` → `+0 = 0` → `0 || undefined = undefined`
- `?fpx=0.5` → `+0.5 = 0.5` → `0.5 || undefined = 0.5`

**结论**：`fpx=0` 或 `fpy=0` 会被解析为 `undefined`，就像没传这个参数一样。

#### 缓存键生成阶段

**文件**：`packages/asset-server-plugin/src/asset-server.ts:217`

```typescript
const focalPoint = fpx && fpy ? `_fpx${fpx}_fpy${fpy}` : '';
```

由于 `fpx`/`fpy` 已被解析为 `undefined`：
- `undefined && undefined = false`
- `focalPoint = ''` → 不追加到缓存键字符串中

**结论**：`?fpx=0&fpy=0` 和不传入 fpx/fpy 生成**完全相同的缓存键**。

#### 实际变换阶段

**文件**：`packages/asset-server-plugin/src/transform-image.ts:32`

```typescript
if (parameters.fpx && parameters.fpy && width && height && mode === 'crop') {
    // 焦点裁剪逻辑
}
```

同样由于 `undefined && undefined = false`，这个条件不成立，**不会进入焦点裁剪分支**，而是执行普通的熵裁剪：

```typescript
const options: ResizeOptions = {};
if (mode === 'crop') {
    options.position = sharp.strategy.entropy;  // 熵裁剪
}
return image.resize(width, height, options);
```

#### 行为对比表

| URL | fpx 参数值 | fpy 参数值 | 缓存键是否包含 focalPoint | 实际裁剪方式 |
|-----|-----------|-----------|--------------------------|-------------|
| `?w=200&h=200&mode=crop` | `undefined` | `undefined` | ❌ 不包含 | 熵裁剪 |
| `?w=200&h=200&mode=crop&fpx=0&fpy=0` | `undefined` | `undefined` | ❌ 不包含 | **熵裁剪**（不是焦点裁剪！） |
| `?w=200&h=200&mode=crop&fpx=0.0&fpy=0.0` | `undefined` | `undefined` | ❌ 不包含 | 熵裁剪 |
| `?w=200&h=200&mode=crop&fpx=0.1&fpy=0.2` | `0.1` | `0.2` | ✅ 包含 `_fpx0.1_fpy0.2` | 焦点裁剪 |

**⚠️ 重要陷阱**：用户以为 `fpx=0&fpy=0` 是"把焦点放在左上角"，但实际上它会被当作**没有设置焦点**，使用熵裁剪。如果真的想要焦点在左上角，需要用一个非常小的非零值如 `fpx=0.0001&fpy=0.0001`。

---

### 4.10 q=0 的边界行为分析

#### 初始参数解析阶段

**文件**：`packages/asset-server-plugin/src/asset-server.ts:195-196`

```typescript
const quality =
    queryParams.q != null ? Math.round(Math.max(Math.min(+queryParams.q, 100), 1)) : undefined;
```

**关键细节**：质量参数的下限是 **1**，不是 0！
- `Math.min(+queryParams.q, 100)` → 上限 100
- `Math.max(..., 1)` → **下限 1**

| URL 参数 | 解析后的值 | 说明 |
|----------|-----------|------|
| `?q=0` | `1` | 被 clamp 到下限 1 |
| `?q=0.5` | `1` | Math.round(0.5) = 1 |
| `?q=1` | `1` | 正常 |
| `?q=50` | `50` | 正常 |
| `?q=100` | `100` | 正常 |
| `?q=101` | `100` | 被 clamp 到上限 100 |
| 不传 `q` | `undefined` | 不设置质量 |

#### PresetOnlyStrategy 过滤阶段

**文件**：`packages/asset-server-plugin/src/config/preset-only-strategy.ts:97-99`

```typescript
const permittedQuality = this.options.permittedQuality ?? [0, 50, 75, 85, 95];
const quality = input.quality && permittedQuality.includes(input.quality) ? input.quality : undefined;
```

默认 `permittedQuality = [0, 50, 75, 85, 95]`，**包含 0 但不包含 1！**

#### q=0 的完整流向

```
URL: ?q=0
   ↓
getInitialImageTransformParameters():
   Math.round(Math.max(Math.min(0, 100), 1)) → Math.round(1) → 1
   ↓
PresetOnlyStrategy (默认配置):
   permittedQuality = [0, 50, 75, 85, 95]
   1 && [0,50,75,85,95].includes(1) → 1 && false → false
   → quality = undefined
   ↓
缓存键生成:
   const quality = q ? `_q${q}` : '' → undefined ? ... : '' → 不追加 _q 后缀
   ↓
实际变换 applyFormat():
   if (quality) → undefined → 不设置质量，使用 Sharp 默认值
```

**结论**：`?q=0` 经过初始解析变成 `1`，然后被默认的 `PresetOnlyStrategy` 过滤掉，最终质量参数为 `undefined`，等同于不传入 `q` 参数。

#### 对缓存键和输出的影响

| 配置场景 | URL | 最终 quality 值 | 缓存键是否包含 _q | 实际输出质量 |
|---------|-----|----------------|-------------------|-------------|
| 无 PresetOnlyStrategy | `?q=0` | `1` | ✅ `_q1` | 质量 1 |
| 无 PresetOnlyStrategy | `?q=1` | `1` | ✅ `_q1` | 质量 1 |
| 默认 PresetOnlyStrategy | `?q=0` | `undefined` | ❌ 不包含 | Sharp 默认 |
| 默认 PresetOnlyStrategy | `?q=50` | `50` | ✅ `_q50` | 质量 50 |
| 默认 PresetOnlyStrategy | `?q=1` | `undefined` | ❌ 不包含 | Sharp 默认 |

**⚠️ 注意**：`PresetOnlyStrategy` 的默认 `permittedQuality` 包含 `0`，但初始解析器的下限是 `1`，这意味着默认配置下 `q=0` 永远无法生效——它会被初始解析器变成 1，然后被策略过滤掉。这是一个设计上的不一致。

---

### 4.11 fpx/fpy 超出 0-1 区间的行为

#### 参数解析阶段

**文件**：`packages/asset-server-plugin/src/asset-server.ts:198-199`

```typescript
const fpx = +queryParams.fpx || undefined;
const fpy = +queryParams.fpy || undefined;
```

只要不是 0/falsy，**超出范围的值会被原样保留**：
- `?fpx=-0.5` → `-0.5`（负数保留）
- `?fpx=1.5` → `1.5`（大于 1 保留）
- `?fpx=2` → `2`（整数保留）

#### 缓存键生成阶段

**文件**：`packages/asset-server-plugin/src/asset-server.ts:217`

```typescript
const focalPoint = fpx && fpy ? `_fpx${fpx}_fpy${fpy}` : '';
```

超出范围的值会**原样写入缓存键**：
- `?fpx=1.5&fpy=-0.2` → `_fpx1.5_fpy-0.2`
- 不同的 fpx/fpy 值（即使超出范围）会生成不同的缓存键

#### 实际变换阶段

**文件**：`packages/asset-server-plugin/src/transform-image.ts:32-47, 144-148`

```typescript
if (parameters.fpx && parameters.fpy && width && height && mode === 'crop') {
    const metadata = await image.metadata();
    const xCenter = parameters.fpx * metadata.width;
    const yCenter = parameters.fpy * metadata.height;
    // ... 调用 resizeToFocalPoint
}

// 在 getExtractionRegion 中
region.left = clamp(0, intermediate.w - target.w, Math.round(newXCenter - target.w / 2));
region.top = clamp(0, intermediate.h - target.h, Math.round(newYCenter - target.h / 2));

function clamp(min: number, max: number, input: number) {
    return Math.min(Math.max(min, input), max);
}
```

**完整流程（以 1000x800 原图，裁剪到 200x200，fpx=1.5 为例）**：

```
1. 计算焦点像素坐标：
   xCenter = 1.5 * 1000 = 1500 （超出图片宽度！）

2. 等比缩放中间图：
   假设 hRatio = 800/200 = 4, wRatio = 1000/200 = 5
   → factor = 4（较小的那个）
   → intermediate = 250x200（等比缩放后）

3. 计算中间图中的焦点：
   newXCenter = 1500 / 4 = 375

4. 计算裁剪区域左边界：
   left = 375 - 200/2 = 375 - 100 = 275

5. clamp 到有效范围：
   intermediate.w - target.w = 250 - 200 = 50
   clamp(0, 50, 275) → 50

6. 最终裁剪区域：
   left = 50（紧贴右边界）
```

**行为总结**：
- fpx/fpy 超出 0-1 范围时，**不会被过滤**，会直接参与焦点计算
- 但 `clamp()` 函数会确保最终提取区域不会超出图片边界
- fpx < 0 会被 clamp 到 0（左边界）
- fpx > 1 会被 clamp 到最大值（右边界）
- 同理 fpy < 0 → 上边界，fpy > 1 → 下边界

#### 超出范围的效果对比表

| URL | 理论焦点 | 实际裁剪效果 | 缓存键 |
|-----|---------|-------------|--------|
| `?fpx=-0.5&fpy=0.5` | 图片左侧外 | 紧贴左边界裁剪 | `_fpx-0.5_fpy0.5` |
| `?fpx=0&fpy=0` | 左上角 | 熵裁剪（不是焦点裁剪！） | 不包含 focalPoint |
| `?fpx=0.0001&fpy=0.0001` | 近似左上角 | 左上角焦点裁剪 | `_fpx0.0001_fpy0.0001` |
| `?fpx=0.5&fpy=0.5` | 中心 | 中心裁剪 | `_fpx0.5_fpy0.5` |
| `?fpx=1.5&fpy=0.5` | 图片右侧外 | 紧贴右边界裁剪 | `_fpx1.5_fpy0.5` |
| `?fpx=2&fpy=-1` | 图片外 | 右下角裁剪 | `_fpx2_fpy-1` |

**重要结论**：
1. fpx/fpy 超出 0-1 范围不会报错，会被 clamp 到边界
2. 不同的超出值（如 1.5 vs 2）会生成**不同的缓存键**，但裁剪效果可能相同（都紧贴右边界）
3. 这可能导致缓存膨胀——多个不同的 fpx 值产生相同的裁剪结果，但占用不同的缓存条目

---

### 4.12 缓存目录结构

```
asset-upload-dir/
├── source/              # 原图目录
│   ├── 00/
│   ├── 01/
│   └── ... (256 个子目录)
├── preview/             # 预览图目录
│   ├── 00/
│   └── ...
└── cache/               # 变换缓存目录（onApplicationBootstrap 时 fs.ensureDirSync 创建）
    └── source/
        ├── 00/
        │   ├── product_<hash1>.jpg   # w=500&h=300 版本
        │   ├── product_<hash2>.jpg   # w=500&h=300&mode=resize 版本
        │   └── product_<hash3>.webp  # w=500&h=300&format=webp 版本
        └── ...
```

---

### 4.11 HTTP 缓存头

#### 两条路径的响应头差异

**文件**：`packages/asset-server-plugin/src/asset-server.ts:80-153`

| 路径 | 响应头设置 | Cache-Control | 说明 |
|------|-----------|---------------|------|
| **sendAsset()（缓存命中）** 第 97-99 行 | `res.contentType(mimeType)`<br>`res.setHeader('content-security-policy', "default-src 'self'")`<br>`res.setHeader('Cache-Control', this.cacheHeader)` | ✅ **有** | 服务端缓存命中时返回 |
| **generateTransformedImage()（实时变换）** 第 140-142 行 | `res.set('Content-Type', mimeType)`<br>`res.setHeader('content-security-policy', "default-src 'self'")`<br>`res.send(imageBuffer)` | ❌ **没有！** | 首次请求、缓存未命中时返回 |

**关键发现**：实时变换路径（`generateTransformedImage()`）**完全没有设置 `Cache-Control` 响应头**！这意味着：
- 首次请求（缓存未命中）：浏览器/CDN 收到的响应**没有** `Cache-Control` 头，可能使用默认缓存策略（取决于浏览器实现，通常不会缓存或缓存时间很短）
- 第二次及以后请求（缓存命中）：浏览器/CDN 收到的响应**有** `Cache-Control` 头，按配置的 `max-age` 缓存

#### cache=false 对浏览器/CDN 缓存的实际影响

`cache=false` 是**服务端文件缓存控制**，与 **HTTP 响应头缓存控制** 是完全独立的两个维度：

| 场景 | 服务端缓存 | 响应头 Cache-Control | 浏览器/CDN 行为 |
|------|-----------|----------------------|-----------------|
| `?w=500`（首次） | 写入缓存文件 | ❌ 无（实时变换路径） | 可能不缓存 |
| `?w=500`（第二次） | 命中缓存文件 | ✅ 有（命中路径） | 按 max-age 缓存 |
| `?w=500&cache=false`（首次） | 不写入缓存文件 | ❌ 无（实时变换路径） | 可能不缓存 |
| `?w=500&cache=false`（缓存已存在） | 命中缓存文件 | ✅ 有（命中路径） | 按 max-age 缓存 |

**cache=false 的真实含义**：
- 它只控制 `generateTransformedImage()` 中是否执行 `writeFileFromBuffer()`（第 132-134 行）
- 它**不影响** `sendAsset()` 的缓存命中读取
- 它**不影响** HTTP `Cache-Control` 响应头的设置

**常见误区**：
- ❌ 错误理解：`cache=false` 让浏览器不要缓存
- ✅ 正确理解：`cache=false` 让服务端不要把本次变换结果写入缓存目录
- ✅ 补充：如果同一参数的缓存已存在，即使带 `cache=false` 也会命中，此时响应仍有 `Cache-Control` 头

#### cacheHeader 配置与 max-age 字符串拼接细节

**文件**：`packages/asset-server-plugin/src/asset-server.ts:45-57`

```typescript
// Configure Cache-Control header
const { cacheHeader } = this.options;
if (!cacheHeader) {
    this.cacheHeader = DEFAULT_CACHE_HEADER;
} else {
    if (typeof cacheHeader === 'string') {
        this.cacheHeader = cacheHeader;
    } else {
        this.cacheHeader = [cacheHeader.restriction, `max-age: ${cacheHeader.maxAge}`]
            .filter(value => !!value)
            .join(', ');
    }
}
```

**配置方式 1：字符串**
```typescript
AssetServerPlugin.init({
    cacheHeader: 'public, max-age=86400',
})
```
→ `this.cacheHeader = 'public, max-age=86400'` ✅ 正确

**配置方式 2：对象**
```typescript
AssetServerPlugin.init({
    cacheHeader: {
        maxAge: 86400,
        restriction: 'public',
    },
})
```
→ `this.cacheHeader = 'public, max-age: 86400'` ⚠️ **注意冒号！**

**⚠️ 重要细节**：对象配置时生成的是 `max-age: 86400`（冒号 + 空格），而标准 HTTP `Cache-Control` 指令语法是 `max-age=86400`（等号，无空格）。

**拼接逻辑逐行解析**：
1. `cacheHeader.restriction` → 如 `'public'` 或 `undefined`
2. `` `max-age: ${cacheHeader.maxAge}` `` → 注意这里用的是**冒号** `:` 而不是**等号** `=`
3. `.filter(value => !!value)` → 过滤掉 `undefined`
4. `.join(', ')` → 用逗号加空格拼接

**默认值（constants.ts）**：
```typescript
export const DEFAULT_CACHE_HEADER = 'public, max-age=15552000';  // 6 个月
```
默认值用的是**等号** `=`，与对象配置的冒号 `:` 不一致。

**实际效果对比**：
| 配置方式 | 生成的 Header 值 | 标准兼容性 |
|----------|-----------------|------------|
| 字符串配置 | `public, max-age=86400` | ✅ 标准 |
| 对象配置 | `public, max-age: 86400` | ⚠️ 非标准（冒号） |

虽然大部分浏览器/CDN 对 `max-age:` 也能兼容解析，但严格来说 `max-age=` 才是 RFC 7234 规定的标准格式。

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

3. **中间件链式错误驱动流转**：`sendAsset()` 通过 `readFileToBuffer` 的异常来区分缓存命中/未命中，404 错误驱动 `generateTransformedImage()` 接管。这种设计使得两条路径共享同一个缓存键空间而无需额外通信机制。

4. **缓存键哈希化的确定性保证**：变换参数先序列化为字符串再做 MD5。由于 `getImageTransformParameters()` 对相同 `req.query` 产生确定性输出，两个中间件独立计算出的缓存键**一定相同**，这是缓存链路闭合的根本前提。

5. **`cache=false` 是服务端文件缓存写入控制**：`cache` 不属于 `ImageTransformParameters`，不参与缓存键计算，只控制 `generateTransformedImage()` 是否执行 `writeFileFromBuffer()`。这意味着：带 `cache=false` 的请求可能命中其他请求写入的缓存文件；反过来，一直用 `cache=false` 则服务端永远不会生成缓存文件，每次请求都走完整变换流程。

6. **两条路径的 Cache-Control 响应头不一致**：`sendAsset()`（缓存命中）设置了 `Cache-Control` 头，而 `generateTransformedImage()`（实时变换）**完全没有设置**。因此首次请求（未命中）浏览器/CDN 可能不缓存，二次请求（命中）才会按 `max-age` 缓存。`cache=false` 与 HTTP 响应头缓存完全无关。

7. **PresetOnlyStrategy 的参数归一化带来缓存合并**：强制使用预设宽高、过滤非法 quality/format，使得不同 URL 若归一化到相同参数就命中同一缓存。无效预设直接抛错，在 `sendAsset()` 的 try-catch 中被拦截为 HTTP 400，请求不会到达变换阶段。

8. **cacheHeader 对象配置的冒号问题**：字符串配置用 `max-age=`（等号，标准），但对象配置拼接时用的是 `max-age:`（冒号）。虽然大部分浏览器兼容冒号格式，但 RFC 7234 标准是等号。

9. **焦点裁剪算法**：先等比缩放使目标尺寸"覆盖"裁剪区域，再基于焦点坐标计算提取区域，保证焦点始终在裁剪结果中心。

10. **变换策略管道**：支持多个 `ImageTransformStrategy` 顺序执行，可用于参数校验、权限控制、预设强制等场景。策略抛错时在 `sendAsset()` 中被截获，返回 400，不触及缓存层。

11. **异常驱动的控制流**：用 404 错误作为"缓存未命中"的信号，sendAsset() 通过 `next(err)` 触发 generateTransformedImage() 接管。这种设计使得两条路径共享缓存键空间而无需显式通信。

12. **fpx/fpy=0 的隐式降级**：由于 JavaScript 的 `||` 运算符会把 0 当作 falsy 值，`fpx=0&fpy=0 会被解析为 undefined，降级为熵裁剪而非左上角焦点裁剪。

13. **错误响应统一无缓存头**：所有 400/404/500 响应都不设置 Cache-Control 和 CSP 头，浏览器/CDN 通常不缓存错误。

14. **q=0 的设计不一致**：初始解析器质量下限是 1（`Math.max(..., 1)`），但 PresetOnlyStrategy 默认 `permittedQuality` 包含 0。这导致 `?q=0` 先被 clamp 到 1，再被策略过滤掉，最终变成 undefined，等同于不传 q 参数。

15. **fpx/fpy 超范围的 clamp 机制**：超出 0-1 的焦点值不会报错，会直接参与计算，但最终通过 `clamp()` 函数限制在图片边界内。不同超范围值（如 1.5 vs 2）可能生成不同缓存键但裁剪效果相同，存在缓存膨胀风险。

16. **流错误防护**：`makeStreamGuard` 处理 `fs-capacitor` 的边界情况，确保流错误能被正确捕获。
