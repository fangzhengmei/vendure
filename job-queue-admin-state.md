# 队列任务失败重试与管理端状态展示链路分析

## 一、核心数据模型

### 1. 任务状态枚举 (JobState)

定义位置：`packages/core/src/api/schema/admin-api/job.api.graphql:27-34`

```graphql
enum JobState {
    PENDING    # 待处理
    RUNNING    # 运行中
    COMPLETED  # 已完成
    RETRYING   # 重试中
    FAILED     # 已失败
    CANCELLED  # 已取消
}
```

### 2. Job 类（内存模型）

定义位置：`packages/core/src/job-queue/job.ts:37-302`

核心字段：
- `id`: 任务唯一标识
- `queueName`: 队列名称
- `retries`: **允许的重试次数**（创建时指定，默认为 0）
- `_attempts`: **已尝试次数**（每次执行时自增）
- `_state`: 当前状态
- `_progress`: 进度 (0-100)
- `_result`: 执行结果
- `_error`: 错误信息
- `createdAt`: 创建时间
- `_startedAt`: 开始执行时间
- `_settledAt`: 结束时间（成功/失败/取消）

### 3. JobRecord 实体（数据库持久化）

定义位置：`packages/core/src/plugin/default-job-queue-plugin/job-record.entity.ts:10-47`

数据库表字段与 Job 类一一对应，增加了 `isSettled` 字段用于快速查询已结束的任务。

## 二、任务调度记录链路

### 1. 任务创建流程

```
JobQueue.add() [job-queue.ts:90-109]
    ↓
创建 Job 对象，设置 retries = options.retries ?? 0
    ↓
JobQueueStrategy.add() [sql-job-queue-strategy.ts:39-49]
    ↓
转换为 JobRecord 并保存到数据库
    ↓
返回带 id 的 Job 对象
```

关键代码 `job-queue.ts:90-95`：
```typescript
async add(data: Data, options?: JobOptions<Data>): Promise<SubscribableJob<Data>> {
    const job = new Job<any>({
        data,
        queueName: this.options.name,
        retries: options?.retries ?? 0,  // 默认为 0，即不重试
    });
    // ...
}
```

### 2. 任务执行与状态更新

轮询策略：`packages/core/src/job-queue/polling-job-queue-strategy.ts:84-251`

```
ActiveQueue.start() [polling-job-queue-strategy.ts:110-182]
    ↓
setTimeout 轮询调用 runNextJobs()
    ↓
JobQueueStrategy.next() [sql-job-queue-strategy.ts:82-113]
    ↓
getNextAndSetAsRunning() 查找 PENDING 或 RETRYING 状态的任务
    ↓
Job.start() [job.ts:127-138]
    - 设置 state = RUNNING
    - 设置 startedAt = 当前时间
    - attempts++ （关键：每次执行尝试次数自增）
    ↓
更新数据库记录 state = RUNNING
    ↓
执行 process 函数
    ↓
成功 → Job.complete() → state = COMPLETED
失败 → Job.fail() → 检查重试逻辑
```

## 三、重试策略实现机制

### 1. 失败处理逻辑

核心代码：`packages/core/src/job-queue/job.ts:166-185`

```typescript
fail(err?: any) {
    this._error = err?.message ? err.message : String(err);
    this._progress = 0;
    if (this.retries >= this._attempts) {
        // 还有重试次数可用 → 状态变为 RETRYING
        this._state = JobState.RETRYING;
        Logger.warn(
            `Job ${this.id} [${this.queueName}] failed (attempt ${
                this._attempts
            } of ${this.retries + 1})`,
        );
    } else {
        // 重试次数用尽 → 状态变为 FAILED
        if (this._state !== JobState.CANCELLED) {
            this._state = JobState.FAILED;
            Logger.warn(
                `Job ${this.id} [${this.queueName}] failed and will not retry.`,
            );
        }
        this._settledAt = new Date();
    }
}
```

**关键判断逻辑**：`this.retries >= this._attempts`

- `retries` 是**允许的重试次数**（创建时指定）
- `_attempts` 是**已尝试次数**（每次 start() 时自增）
- 总执行次数 = `retries + 1`（首次 + 重试次数）

**示例**：
- `retries = 2`，则总执行次数为 3 次
- 第 1 次失败：`attempts = 1`，`2 >= 1` → RETRYING
- 第 2 次失败：`attempts = 2`，`2 >= 2` → RETRYING
- 第 3 次失败：`attempts = 3`，`2 >= 3` 为 false → FAILED

### 2. 重试间隔（退避策略）

定义位置：`packages/core/src/job-queue/polling-job-queue-strategy.ts:22-67`

```typescript
export type BackoffStrategy = (queueName: string, attemptsMade: number, job: Job) => number;

// 默认退避策略：固定 1000ms
this.backOffStrategy = concurrencyOrConfig.backoffStrategy ?? (() => 1000);
```

重试间隔检查位置：`packages/core/src/plugin/default-job-queue-plugin/sql-job-queue-strategy.ts:144-153`

```typescript
if (record.state === JobState.RETRYING && typeof this.backOffStrategy === 'function') {
    const msSinceLastFailure = Date.now() - +record.updatedAt;
    const backOffDelayMs = this.backOffStrategy(queueName, record.attempts, job);
    if (msSinceLastFailure < backOffDelayMs) {
        // 还没到重试时间，跳过这个任务，拿下一个
        return await this.getNextAndSetAsRunning(manager, queueName, setLock, [
            ...waitingJobIds,
            record.id,
        ]);
    }
}
```

### 3. 可配置的重试策略

`PollingJobQueueStrategyConfig` 配置项：
- `setRetries`: 可覆盖任务创建时指定的 retries
- `backoffStrategy`: 自定义重试间隔策略
- `concurrency`: 并发执行数
- `pollInterval`: 轮询间隔

## 四、管理端列表渲染链路

### 1. GraphQL API 层

定义位置：`packages/core/src/api/schema/admin-api/job.api.graphql:1-63`

查询字段：
```graphql
type Job implements Node {
    id: ID!
    createdAt: DateTime!
    startedAt: DateTime
    settledAt: DateTime
    queueName: String!
    state: JobState!
    progress: Float!
    data: JSON
    result: JSON
    error: JSON
    isSettled: Boolean!
    duration: Int!
    retries: Int!      # 允许的重试次数
    attempts: Int!     # 已尝试次数
}
```

解析器：`packages/core/src/api/resolvers/admin/job.resolver.ts:27-48`

```typescript
@Query()
@Allow(Permission.ReadSettings, Permission.ReadSystem)
async jobs(@Args() args: QueryJobsArgs) {
    const strategy = this.requireInspectableJobQueueStrategy();
    return strategy.findMany(args.options || undefined);
}
```

### 2. 前端查询定义

位置：`packages/dashboard/src/app/routes/_authenticated/_system/job-queue.graphql.ts:3-54`

```typescript
export const jobInfoFragment = graphql(`
    fragment JobInfo on Job {
        id
        queueName
        createdAt
        startedAt
        settledAt
        state
        isSettled
        progress
        duration
        data
        result
        error
        retries    # 查询了但默认不显示
        attempts   # 查询了但默认不显示
    }
`);
```

### 3. 列表渲染逻辑

位置：`packages/dashboard/src/app/routes/_authenticated/_system/job-queue.tsx:60-239`

#### 状态徽章样式映射

```typescript
function getJobStateBadgeVariant(state: string) {
    switch (state) {
        case 'PENDING':
        case 'RETRYING':
            return 'warning';   // 黄色警告
        case 'COMPLETED':
            return 'success';   // 绿色成功
        case 'FAILED':
        case 'CANCELLED':
            return 'destructive'; // 红色错误
        default:
            return 'secondary';
    }
}
```

#### 状态图标配置

```typescript
const STATES = [
    { label: 'Pending',   value: 'PENDING',   icon: ClockIcon },
    { label: 'Completed', value: 'COMPLETED', icon: CheckIcon },
    { label: 'Running',   value: 'RUNNING',   icon: LoaderIcon },
    { label: 'Failed',    value: 'FAILED',    icon: CircleXIcon },
    { label: 'Retrying',  value: 'RETRYING',  icon: RotateCcw },
    { label: 'Cancelled', value: 'CANCELLED', icon: Ban },
];
```

#### 默认隐藏的关键字段

```typescript
defaultVisibility={{
    isSettled: false,
    settledAt: false,
    progress: false,
    retries: false,    // 允许重试次数 - 默认隐藏
    attempts: false,   // 已尝试次数 - 默认隐藏
    error: false,      // 错误信息 - 默认隐藏
    startedAt: false,
}}
```

**重要问题**：`retries` 和 `attempts` 字段默认隐藏，导致用户无法直观看到：
- 任务配置了多少次重试
- 当前已经尝试了多少次
- 还剩多少次重试机会

#### 状态列渲染

```typescript
state: {
    cell: ({ row, table }) => {
        const state = STATES.find(s => s.value === row.original.state);
        return (
            <div className="flex items-center gap-2">
                <Badge variant={getJobStateBadgeVariant(row.original.state)}>
                    {state && (
                        <state.icon
                            className={
                                row.original.state === 'RUNNING' ? 'animate-spin' : undefined
                            }
                        />
                    )}
                    {row.original.state}
                </Badge>
                {/* 只有 RUNNING 状态显示取消按钮 */}
                {row.original.state === 'RUNNING' && (
                    <DropdownMenu>
                        {/* 取消按钮 */}
                    </DropdownMenu>
                )}
            </div>
        );
    },
},
```

### 4. 自动刷新机制

位置：`packages/dashboard/src/lib/hooks/use-job-queue-polling.ts:1-160`

- 默认每 10 秒自动刷新列表
- 可配置刷新间隔：5s/10s/30s/60s/关闭
- 轮询时检查任务状态：`PENDING`, `RUNNING`, `RETRYING` 视为未结束

## 五、完整链路总结

### 状态流转图

```
创建任务 → PENDING
    ↓
开始执行 → RUNNING (attempts++)
    ↓
执行成功 → COMPLETED (settledAt)
    ↓
执行失败 → Job.fail()
    ├─ retries >= attempts → RETRYING (等待 backoff 时间)
    │                          ↓
    │                        再次执行 → RUNNING (attempts++)
    │                          ↓  ... 循环 ...
    └─ retries < attempts → FAILED (settledAt)
```

### 数据流转链路

```
业务代码调用 JobQueue.add(data, { retries: N })
    ↓  [内存]
创建 Job 对象: { retries: N, attempts: 0, state: PENDING }
    ↓  [持久化]
SqlJobQueueStrategy.add() → 保存 JobRecord 到数据库
    ↓  [调度]
ActiveQueue 轮询 → next() → 查找 PENDING/RETRYING 任务
    ↓  [执行]
Job.start() → state=RUNNING, attempts++, startedAt=now
    ↓  [更新]
SqlJobQueueStrategy.update() → 更新数据库
    ↓  [业务逻辑]
执行 process(job) 函数
    ↓  [结果处理]
成功 → job.complete(result) → state=COMPLETED, settledAt=now
失败 → job.fail(error)
    ├─ 还有重试 → state=RETRYING
    └─ 重试用尽 → state=FAILED, settledAt=now
    ↓  [更新]
SqlJobQueueStrategy.update() → 更新数据库
    ↓  [展示]
GraphQL Query jobs() → 查询 JobRecord 列表
    ↓  [渲染]
Dashboard JobQueuePage → 按状态映射为不同颜色的徽章
```

## 六、管理端展示不直观的问题分析

### 1. 核心问题

用户看到一个 `RETRYING` 状态的任务时，无法直观知道：
- 这个任务配置了多少次重试？
- 这是第几次重试？
- 还剩多少次重试机会？
- 上次失败的原因是什么？

### 2. 原因

- `retries`（允许重试次数）和 `attempts`（已尝试次数）字段默认隐藏
- `error`（错误信息）字段默认隐藏
- 状态徽章只显示状态文本，没有尝试次数信息

### 3. 数据对应关系

| 数据库字段 | 含义 | 管理端可见性 |
|-----------|------|-------------|
| `state` | 当前状态 | 可见，显示为徽章 |
| `retries` | 允许的重试次数 | 默认隐藏 |
| `attempts` | 已尝试次数 | 默认隐藏 |
| `error` | 错误信息 | 默认隐藏 |
| `progress` | 进度 | 默认隐藏 |
| `startedAt` | 开始时间 | 默认隐藏 |
| `settledAt` | 结束时间 | 默认隐藏 |

**建议**：在 `RETRYING` 状态的徽章旁显示 `attempts / (retries + 1)` 信息，例如 `RETRYING (2/3)`，让用户直观了解重试进度。
