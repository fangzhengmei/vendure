# 后台动态表单字段元数据渲染流程解析

## 概述

Vendure 后台的动态表单系统基于 GraphQL 元数据驱动，实现了从字段类型注册、控件分发到值校验的完整接力过程。整个系统采用分层架构，核心围绕以下几个模块展开：

| 阶段 | 核心职责 | 关键文件 |
|------|----------|----------|
| **元数据获取** | 从 GraphQL Schema 和服务器配置提取字段定义 | `get-document-structure.ts` |
| **类型注册** | 内建/自定义表单组件注册到全局注册表 | `input-component-extensions.tsx`, `global-registry.ts` |
| **Schema 生成** | 将元数据转换为 Zod 验证 Schema 和默认值 | `form-schema-tools.ts` |
| **控件分发** | 根据字段类型路由到对应的表单控件 | `form-control-adapter.tsx`, `default-input-for-type.tsx` |
| **值校验** | 实时验证表单值并处理数据转换 | `use-generated-form.tsx` |
| **表单渲染** | 动态生成表单 UI 并绑定状态 | `custom-fields-form.tsx` |

---

## 第一阶段：元数据获取

### 1.1 GraphQL Schema 元数据注入

表单系统依赖于构建时注入的 GraphQL Schema 信息（`virtual:admin-api-schema`），该信息包含了所有类型、输入类型、标量和枚举的完整元数据。

**关键代码位置**：`get-document-structure.ts:11-14`

```typescript
import { schemaInfo } from 'virtual:admin-api-schema';
(window as any).schemaInfo = schemaInfo;
```

`schemaInfo` 结构：
- `types` - 所有 GraphQL 对象类型及其字段定义
- `inputs` - 所有输入类型及其字段定义  
- `scalars` - 所有标量类型名称
- `enums` - 所有枚举类型及其值

### 1.2 从 GraphQL 文档提取字段结构

`getOperationVariablesFields()` 函数解析 mutation 文档，提取输入类型的字段结构，返回 `FieldInfo[]`：

**关键代码位置**：`get-document-structure.ts:282-322`

```typescript
export interface FieldInfo {
    name: string;           // 字段名
    type: string;           // 类型名（如 String, Int, Product）
    nullable: boolean;      // 是否可空
    list: boolean;          // 是否为数组
    isPaginatedList: boolean;  // 是否为分页列表
    isScalar: boolean;      // 是否为标量类型
    typeInfo?: FieldInfo[]; // 嵌套对象的字段信息（递归）
}
```

**提取流程**：
1. 解析 GraphQL mutation 的变量定义
2. 解包变量类型（移除 NonNullType、ListType 包装）
3. 根据类型名称从 `schemaInfo.inputs` 获取输入类型字段
4. 递归处理嵌套对象类型，构建完整的字段树

---

## 第二阶段：字段类型注册

### 2.1 全局注册表机制

由于 Vite 打包的特性，不能依赖闭包变量在不同 bundle 间共享状态，因此使用单例 `GlobalRegistry` 挂载到 `globalThis`：

**关键代码位置**：`global-registry.ts:11-50`

```typescript
class GlobalRegistry {
    private static instance: GlobalRegistry;
    private registry: Map<string, any> = new Map();
    
    public register<T>(key: T, value: GlobalRegistryContents[T]);
    public get<T>(key: T): GlobalRegistryContents[T];
    public set<T>(key: T, updater: (oldValue) => newValue);
}

const _globalRegistry: GlobalRegistry = (globalThis as any).globalRegistry ?? new GlobalRegistry();
(globalThis as any).globalRegistry = _globalRegistry;
```

### 2.2 内建输入组件注册

系统启动时自动注册内建表单组件到 `inputComponents` 注册表：

**关键代码位置**：`input-component-extensions.tsx:18-44`

```typescript
globalRegistry.register('inputComponents', new Map<string, DashboardFormComponent>());

const inputComponents = globalRegistry.get('inputComponents');
inputComponents.set('text-form-input', TextInput);
inputComponents.set('number-form-input', NumberInput);
inputComponents.set('textarea-form-input', TextareaInput);
inputComponents.set('rich-text-form-input', RichTextInput);
inputComponents.set('password-form-input', PasswordFormInput);
inputComponents.set('select-form-input', SelectWithOptions);
inputComponents.set('relation-form-input', DefaultRelationInput);
// ... 更多内建组件
```

### 2.3 自定义组件注册

插件可以通过 `addCustomFieldInputComponent()` 注册自定义表单组件：

**关键代码位置**：`input-component-extensions.tsx:86-99`

```typescript
export function addCustomFieldInputComponent({
    id,
    component,
}: {
    id: string;
    component: DashboardFormComponent;
}) {
    const inputComponents = globalRegistry.get('inputComponents');
    if (inputComponents.has(id)) {
        console.warn(`Input component with key "${id}" is already registered and will be overwritten.`);
    }
    inputComponents.set(id, component);
}
```

**自定义组件示例**（来自 reviews 插件）：

**关键代码位置**：`custom-form-components.tsx:16-82`

```typescript
export const TextareaCustomField: DashboardFormComponent = props => {
    return <Textarea {...props} rows={4} />;
};

export const ReviewStateSelect: DashboardFormComponent = props => {
    return (
        <Select value={props.value} onValueChange={props.onChange}>
            <SelectTrigger><SelectValue placeholder="Select state..." /></SelectTrigger>
            <SelectContent>
                <SelectItem value="new">New</SelectItem>
                <SelectItem value="approved">Approved</SelectItem>
                <SelectItem value="rejected">Rejected</SelectItem>
            </SelectContent>
        </Select>
    );
};
```

### 2.4 DashboardFormComponent 类型契约

所有表单组件必须实现统一的类型契约：

**关键代码位置**：`form-engine-types.ts:92-168`

```typescript
export type DashboardFormComponentProps<
    TFieldValues extends FieldValues = FieldValues,
    TName extends FieldPath<TFieldValues> = FieldPath<TFieldValues>,
> = ControllerRenderProps<TFieldValues, TName> & {
    fieldDef?: ConfigurableFieldDef;
};

export type DashboardFormComponent = React.ComponentType<DashboardFormComponentProps> & {
    metadata?: DashboardFormComponentMetadata;
};

export type DashboardFormComponentMetadata = {
    isListInput?: boolean | 'dynamic';  // 是否能处理数组类型
    isFullWidth?: boolean;              // 是否占满整行宽度
};
```

**Props 说明**：
- `onChange`, `onBlur`, `value`, `name`, `ref`, `disabled` - 来自 react-hook-form
- `fieldDef` - 字段元数据定义，包含类型、约束、UI 配置等

---

## 第三阶段：表单 Schema 生成

### 3.1 Zod 验证 Schema 构建

`createFormSchemaFromFields()` 将 `FieldInfo[]` 转换为 Zod 验证 Schema：

**关键代码位置**：`form-schema-tools.ts:294-336`

```typescript
export function createFormSchemaFromFields(
    fields: FieldInfo[],
    customFieldConfigs?: CustomFieldConfig[],
    isTranslationContext = false,
) {
    const schemaConfig: ZodRawShape = {};

    for (const field of fields) {
        const isScalar = isScalarType(field.type);
        const isEnum = isEnumType(field.type);

        if ((isScalar || isEnum) && field.name !== 'customFields') {
            // 1. 标量和枚举字段 - 使用 getZodTypeFromField
            schemaConfig[field.name] = getZodTypeFromField(field);
        } else if (field.name === 'customFields') {
            // 2. customFields 字段 - 特殊处理，递归处理所有自定义字段
            const customFieldsSchema = customFieldConfigs?.length
                ? processCustomFieldsSchema(customFieldConfigs, isTranslationContext)
                : {};
            schemaConfig[field.name] = z.object(customFieldsSchema).passthrough().optional();
        } else if (field.typeInfo) {
            // 3. 嵌套对象类型 - 递归调用
            const isNestedTranslationContext = field.name === 'translations' || isTranslationContext;
            let nestedType = createFormSchemaFromFields(
                field.typeInfo, customFieldConfigs, isNestedTranslationContext,
            );
            // 应用 nullable 和 list 修饰符
            if (field.nullable) nestedType = nestedType.optional().nullable();
            if (field.list) nestedType = z.array(nestedType);
            schemaConfig[field.name] = nestedType;
        }
    }
    return z.object(schemaConfig);
}
```

### 3.2 自定义字段 Schema 处理

`processCustomFieldsSchema()` 处理自定义字段的验证规则：

**关键代码位置**：`form-schema-tools.ts:262-292`

```typescript
function processCustomFieldsSchema(
    customFieldConfigs: CustomFieldConfig[],
    isTranslationContext: boolean,
): ZodRawShape {
    const customFieldsSchema: ZodRawShape = {};
    const translatableTypes = ['localeString', 'localeText'];

    // 根据上下文过滤字段（翻译上下文只处理本地化字段）
    const filteredCustomFields = customFieldConfigs.filter(cf => {
        return isTranslationContext 
            ? translatableTypes.includes(cf.type)
            : !translatableTypes.includes(cf.type);
    });

    for (const customField of filteredCustomFields) {
        let zodType: ZodType;
        if (customField.type === 'struct') {
            // struct 类型需要递归处理其内部字段
            zodType = createStructFieldSchema(customField as StructCustomFieldConfig);
        } else {
            zodType = createCustomFieldValidationSchema(customField);
        }
        // 应用 list、nullable、readonly 修饰符
        zodType = applyCustomFieldModifiers(zodType, customField);
        const schemaPropertyName = getGraphQlInputName(customField);
        customFieldsSchema[schemaPropertyName] = zodType;
    }
    return customFieldsSchema;
}
```

### 3.3 各类型验证 Schema 生成器

| 字段类型 | 验证逻辑 | 关键函数 |
|---------|----------|----------|
| `string` | 可选正则表达式验证 | `createStringValidationSchema()` |
| `int/float` | 可选 min/max 范围验证 | `createNumberValidationSchema()` |
| `datetime` | 可选日期范围验证，支持 string/Date 输入 | `createDateValidationSchema()` |
| `boolean` | 布尔值验证 | `z.boolean()` |
| `struct` | 递归构建嵌套对象 Schema | `createStructFieldSchema()` |
| `relation` | 关系字段特殊处理 ID 命名 | `getGraphQlInputName()` |

**修饰符应用**：`applyCustomFieldModifiers()` 根据配置应用：
- `list: true` → `z.array(type)`
- `nullable: true` → `.optional().nullable()`
- `readonly: true` → `.readonly()`

### 3.4 默认值生成

`getDefaultValuesFromFields()` 根据字段类型生成合理的默认值：

**关键代码位置**：`form-schema-tools.ts:338-399`

```typescript
export function getDefaultValueFromField(field: FieldInfo, defaultLanguageCode?: string) {
    if (field.list) return [];
    if (field.nullable) {
        switch (field.type) {
            case 'String': return '';
            case 'ID': return '';
            case 'LanguageCode': return defaultLanguageCode || 'en';
            case 'Boolean': return false;
            case 'JSON': return {};
            default: return null;
        }
    }
    switch (field.type) {
        case 'String': case 'DateTime': return '';
        case 'Int': case 'Float': case 'Money': return 0;
        case 'Boolean': return false;
        case 'ID': return '';
        case 'JSON': return {};
        default: return '';
    }
}
```

---

## 第四阶段：控件分发

### 4.1 FormControlAdapter - 控件分发中枢

`FormControlAdapter` 是表单控件分发的核心协调者，负责根据字段元数据选择最合适的控件：

**关键代码位置**：`form-control-adapter.tsx:151-193`

```typescript
export function FormControlAdapter({ fieldDef, field, valueMode, ...rest }) {
    const isList = fieldDef.list ?? false;
    const isReadonly = isCustomFieldConfig(fieldDef) ? fieldDef.readonly === true : false;
    const componentId = fieldDef.ui?.component as string | undefined;

    // 1. 值转换包装（parse/serialize）
    const fieldWithTransform = useMemo(() => ({
        ...field,
        value: transformValue(field.value, fieldDef, valueMode, 'parse'),
        onChange: (newValue: any) => {
            const serializedValue = transformValue(newValue, fieldDef, valueMode, 'serialize');
            field.onChange(serializedValue);
        },
    }), [field.name, field.value, field.disabled, field.onChange, fieldDef, valueMode]);

    // 2. 尝试使用自定义组件（优先）
    const CustomComponent = getInputComponent(componentId);
    if (canUseCustomComponent(CustomComponent, isList)) {
        return <CustomComponent {...fieldWithTransform} fieldDef={fieldDef} />;
    }

    // 3. struct 字段特殊处理
    if (fieldDef.type === 'struct' && valueMode === 'native') {
        return renderStructField(fieldDef, field, fieldWithTransform, isList, isReadonly);
    }

    // 4. list 字段特殊处理
    if (isList) {
        return renderListField(fieldDef, field, fieldWithTransform, valueMode, isReadonly);
    }

    // 5. 默认分发
    return <DefaultInputForType {...fieldWithTransform} fieldDef={fieldDef} {...rest} />;
}
```

### 4.2 自定义组件兼容性检查

`canUseCustomComponent()` 验证自定义组件是否能处理当前字段：

**关键代码位置**：`form-control-adapter.tsx:69-82`

```typescript
function canUseCustomComponent(
    CustomComponent: DashboardFormComponent | undefined,
    isList: boolean,
): CustomComponent is DashboardFormComponent {
    if (!CustomComponent) return false;
    const listInputMode = CustomComponent.metadata?.isListInput;
    // 'dynamic' 表示组件可同时处理单值和数组
    if (listInputMode === 'dynamic') return true;
    // 精确匹配：都是数组或都不是数组
    return (isList && listInputMode === true) || (!isList && listInputMode !== true);
}
```

### 4.3 DefaultInputForType - 类型到控件映射

`DefaultInputForType` 根据字段类型分发到默认控件：

**关键代码位置**：`default-input-for-type.tsx:13-35`

```typescript
export function DefaultInputForType({ fieldDef, ...fieldProps }) {
    const type = fieldDef?.type;
    switch (type) {
        case 'int':
        case 'float':
            return <NumberInput {...fieldProps} fieldDef={fieldDef} />;
        case 'boolean':
            return <BooleanInput {...fieldProps} fieldDef={fieldDef} />;
        case 'datetime':
            return <DateTimeInput {...fieldProps} fieldDef={fieldDef} />;
        case 'relation':
            return <DefaultRelationInput {...fieldProps} fieldDef={fieldDef} />;
        case 'string': {
            if (fieldDef && isStringFieldWithOptions(fieldDef)) {
                return <SelectWithOptions {...fieldProps} fieldDef={fieldDef} />;
            } else {
                return <TextInput {...fieldProps} fieldDef={fieldDef} />;
            }
        }
        default:
            return <TextInput {...fieldProps} fieldDef={fieldDef} />;
    }
}
```

### 4.4 List 字段渲染

`renderListField()` 根据字段类型选择不同的数组输入控件：

**关键代码位置**：`form-control-adapter.tsx:112-139`

```typescript
function renderListField(fieldDef, field, fieldWithTransform, valueMode, isReadonly) {
    if (valueMode === 'json-string') {
        // 可配置操作参数模式 - JSON 字符串编辑
        return <ConfigurableOperationListInput {...fieldWithTransform} fieldDef={fieldDef} />;
    }
    if (fieldDef.type === 'relation') {
        // 关系类型数组使用专用组件
        return <DefaultInputForType {...fieldWithTransform} fieldDef={fieldDef} />;
    }
    if (fieldDef.type === 'string') {
        // 字符串数组使用专用组件
        return <StringListInput {...fieldWithTransform} fieldDef={fieldDef} />;
    }
    // 通用数组输入 - 动态添加/删除项
    return (
        <CustomFieldListInput
            {...field}
            disabled={isReadonly}
            renderInput={(index, inputField) => 
                <DefaultInputForType {...inputField} fieldDef={fieldDef} />
            }
            defaultValue={getDefaultValueForType(fieldDef.type)}
        />
    );
}
```

---

## 第五阶段：值校验

### 5.1 useGeneratedForm - 表单生命周期管理

`useGeneratedForm` Hook 整合了表单生成、状态管理和验证逻辑：

**关键代码位置**：`use-generated-form.tsx:96-201`

```typescript
export function useGeneratedForm(options) {
    const { document, entity, setValues, onSubmit, varName, customFieldConfig } = options;

    // 1. 从 GraphQL 文档提取字段结构
    const updateFields = useMemo(
        () => document ? getOperationVariablesFields(document, varName) : [],
        [document, varName],
    );

    // 2. 生成验证 Schema 和默认值
    const schema = useMemo(
        () => createFormSchemaFromFields(updateFields, customFieldConfig),
        [updateFields, customFieldConfig],
    );
    const defaultValues = useMemo(
        () => getDefaultValuesFromFields(updateFields, activeChannel?.defaultLanguageCode),
        [updateFields, activeChannel?.defaultLanguageCode],
    );

    // 3. 处理实体数据，确保翻译字段完整
    const processedEntity = useMemo(
        () => ensureTranslationsForAllLanguages(entity, availableLanguages, defaultValues),
        [entity, availableLanguages, defaultValues],
    );

    // 4. 转换关系字段（提取 ID）
    const values = useMemo(() => 
        processedEntity
            ? transformRelationFields(updateFields, setValuesRef.current(processedEntity))
            : processedDefaultValues,
        [processedEntity, processedDefaultValues, updateFields],
    );

    // 5. 初始化 react-hook-form
    const form = useForm({
        resolver: async (values, context, options) => {
            const result = await zodResolver(schema)(values, context, options);
            if (Object.keys(result.errors).length > 0) {
                console.log('Zod form validation errors:', result.errors);
            }
            return result;
        },
        mode: 'onChange',  // 实时验证
        defaultValues: processedDefaultValues,
        values,
    });

    // 6. 提交处理
    const submitHandler = async (event: FormEvent) => {
        event.preventDefault();
        const isValid = await form.trigger(); // 触发全字段验证
        if (!isValid) {
            console.log(`Form invalid!`);
            event.stopPropagation();
            return;
        }
        const onSubmitWrapper = (values: any) => {
            let processed = convertEmptyStringsToNull(
                removeEmptyIdFields(values, updateFields),
                updateFields,
            );
            if (!entity) {
                processed = stripNullNullableFields(processed, updateFields);
            }
            onSubmit(processed);
        };
        form.handleSubmit(onSubmitWrapper)(event);
    };

    return { form, submitHandler };
}
```

### 5.2 提交前数据转换

| 转换函数 | 作用 | 关键代码位置 |
|---------|------|-------------|
| `transformRelationFields` | 从关系对象中提取 ID，转换为 `xxxId` 或 `xxxIds` | `utils.ts:39-94` |
| `removeEmptyIdFields` | 删除空字符串的 ID 字段（避免 create mutation 问题） | `utils.ts:102-141` |
| `convertEmptyStringsToNull` | 将可空非字符串字段的空字符串转为 null | `utils.ts:148-173` |
| `stripNullNullableFields` | create mutation 时移除 null 值字段，让服务器应用默认值 | `utils.ts:182-204` |

### 5.3 Zod 验证流程

1. **Schema 构建**：`createFormSchemaFromFields` → 递归构建嵌套 Zod Schema
2. **Resolver 绑定**：`zodResolver(schema)` 包装为 react-hook-form 可用的验证器
3. **实时验证**：`mode: 'onChange'` 字段变化时自动验证
4. **提交验证**：`form.trigger()` 手动触发全字段验证
5. **错误处理**：验证错误通过 `fieldState.error` 传递到 UI 组件

---

## 第六阶段：表单渲染

### 6.1 CustomFieldsForm - 自定义字段表单渲染

`CustomFieldsForm` 展示了如何将上述流程整合为完整的表单 UI：

**关键代码位置**：`custom-fields-form.tsx:31-122`

```typescript
export function CustomFieldsForm({ entityType, control, formPathPrefix, disabled }) {
    const customFields = useCustomFieldConfig(entityType);

    // 1. 按 tab 分组字段
    const groupedFields = useMemo(() => {
        if (!customFields) return [];
        const tabMap = new Map<string, CustomFieldConfig[]>();
        const defaultTabName = '__default_tab__';
        for (const field of customFields) {
            const tabName = field.ui?.tab ?? defaultTabName;
            tabMap.has(tabName) 
                ? tabMap.get(tabName)?.push(field) 
                : tabMap.set(tabName, [field]);
        }
        return Array.from(tabMap.entries())
            .sort((a, b) => a[0] === defaultTabName ? -1 : 1)
            .map(([tabName, customFields]) => ({
                tabName: tabName === defaultTabName ? 'general' : tabName,
                customFields,
            }));
    }, [customFields]);

    // 2. 根据是否有多个 tab 决定渲染方式
    const shouldShowTabs = useMemo(() => {
        if (!customFields) return false;
        const hasTabbedFields = customFields.some(field => field.ui?.tab);
        return hasTabbedFields && groupedFields.length > 1;
    }, [customFields, groupedFields.length]);

    // 3. 渲染（单 tab 或多 tab 视图）
    if (!shouldShowTabs) {
        return (
            <div className="grid @md:grid-cols-2 gap-6">
                {customFields?.map(fieldDef => (
                    <CustomFieldItem
                        key={fieldDef.name}
                        fieldDef={fieldDef}
                        control={control}
                        fieldName={getFieldName(fieldDef)}
                        disabled={disabled}
                    />
                ))}
            </div>
        );
    }
    // ... Tab 视图渲染
}
```

### 6.2 CustomFieldItem - 单字段渲染

`CustomFieldItem` 根据字段类型分发到不同的渲染路径：

**关键代码位置**：`custom-fields-form.tsx:196-365`

```typescript
function CustomFieldItem({ fieldDef, control, fieldName, disabled }) {
    const hasCustomFormComponent = fieldDef.ui?.component;
    const isLocaleField = fieldDef.type === 'localeString' || fieldDef.type === 'localeText';
    const isReadonly = (fieldDef as CustomFieldConfig).readonly ?? false;

    // 1. 本地化字段 - 使用 TranslatableFormField 包装
    if (isLocaleField) {
        return (
            <div className={containerClassName}>
                <TranslatableFormField
                    control={control}
                    name={fieldName}
                    disabled={disabled}
                    render={({ field, fieldState }) => {
                        const inputElement = hasCustomFormComponent 
                            ? <CustomFormComponent fieldDef={fieldDef} {...field} />
                            : <FormControlAdapter fieldDef={fieldDef} field={field} valueMode="native" />;
                        return (
                            <CustomFieldFormItem {...}>
                                {inputElement}
                            </CustomFieldFormItem>
                        );
                    }}
                />
            </div>
        );
    }

    // 2. 有自定义组件的非本地化字段
    if (hasCustomFormComponent) {
        return (
            <div className={containerClassName}>
                <Controller
                    control={control}
                    name={fieldName}
                    disabled={disabled}
                    render={({ field, fieldState }) => (
                        <CustomFieldFormItem {...}>
                            <CustomFormComponent fieldDef={fieldDef} {...field} />
                        </CustomFieldFormItem>
                    )}
                />
            </div>
        );
    }

    // 3. struct 字段
    if (fieldDef.type === 'struct') {
        // ... 渲染 StructFormInput 或 CustomFieldListInput 包装
    }

    // 4. 普通字段
    return (
        <div className={containerClassName}>
            <Controller
                control={control}
                name={fieldName}
                disabled={disabled}
                render={({ field, fieldState }) => (
                    <CustomFieldFormItem {...}>
                        <FormControlAdapter fieldDef={fieldDef} field={field} valueMode="native" />
                    </CustomFieldFormItem>
                )}
            />
        </div>
    );
}
```

---

## 完整接力流程图

```
GraphQL Schema + 服务器配置
       │
       ▼
getOperationVariablesFields() 提取 FieldInfo[]
       │
       ▼
createFormSchemaFromFields() 生成 Zod Schema
getDefaultValuesFromFields()  生成默认值
       │
       ▼
useGeneratedForm() 初始化 react-hook-form
       │
       ├─ 绑定 zodResolver 验证器
       ├─ 设置 mode: 'onChange' 实时验证
       └─ 处理 entity 数据和关系字段转换
       │
       ▼
CustomFieldsForm 渲染
       │
       ├─ 按 ui.tab 分组字段
       ├─ 决定是否使用 Tab 视图
       └─ 遍历字段调用 CustomFieldItem
           │
           ▼
CustomFieldItem 字段渲染路由
       │
       ├─ isLocaleField? → TranslatableFormField
       ├─ hasCustomFormComponent? → CustomFormComponent
       ├─ type === 'struct'? → StructFormInput
       └─ 其他 → FormControlAdapter
           │
           ▼
FormControlAdapter 控件分发
       │
       ├─ 值转换包装（parse/serialize）
       ├─ getInputComponent() 查找自定义组件
       ├─ canUseCustomComponent() 兼容性检查
       ├─ isList? → renderListField()
       └─ 其他 → DefaultInputForType
           │
           ▼
DefaultInputForType 类型→控件映射
       │
       ├─ int/float → NumberInput
       ├─ boolean → BooleanInput
       ├─ datetime → DateTimeInput
       ├─ relation → DefaultRelationInput
       ├─ string + options → SelectWithOptions
       └─ string → TextInput
       │
       ▼
用户交互 → onChange 触发
       │
       ├─ transformValue 序列化
       ├─ react-hook-form 更新状态
       ├─ zodResolver 实时验证
       └─ fieldState.error → FieldError 显示
       │
       ▼
提交 → form.trigger() 全量验证
       │
       ├─ convertEmptyStringsToNull
       ├─ removeEmptyIdFields
       ├─ stripNullNullableFields（create 时）
       └─ onSubmit(processedValues)
```

---

## 核心设计要点

1. **元数据驱动**：所有表单行为由 GraphQL Schema 和服务器配置驱动，无需硬编码字段
2. **分层架构**：元数据提取 → Schema 生成 → 控件分发 → 值校验 → UI 渲染，各层职责清晰
3. **扩展性**：通过 `GlobalRegistry` 和 `addCustomFieldInputComponent` 支持插件扩展
4. **类型安全**：全程使用 TypeScript 类型约束，`FieldInfo` 和 `DashboardFormComponentProps` 确保类型一致性
5. **递归处理**：嵌套对象、struct 类型、翻译字段均采用递归处理，支持复杂数据结构
6. **数据转换管道**：值在 UI 和存储之间经过 parse/serialize 转换，确保数据格式正确
