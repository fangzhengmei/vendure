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
- `retries`: **允许的重试次数**（创建时指定，默认为 0，可被策略层覆盖）
- `_attempts`: **已尝试次数**（每次执行时自增）
- `_state`: 当前状态
- `_progress`: 进度 (0-100)
- `_result`: 执行结果
- `_error`: 错误信息
- `createdAt`: 创建时间
- `_startedAt`: 开始执行时间
- `_settledAt`: 结束时间（成功/失败/取消）

**注意**：`Job` 类没有 `updatedAt` 字段，该字段由 TypeORM 在 `VendureEntity` 基类中自动维护。

### 3. JobRecord 实体（数据库持久化）

定义位置：`packages/core/src/plugin/default-job-queue-plugin/job-record.entity.ts:10-47`

数据库表字段与 Job 类一一对应，增加了 `isSettled` 字段用于快速查询已结束的任务。

**继承关系**：`JobRecord` → `VendureEntity`，自动获得：
- `id`: 主键
- `createdAt`: 创建时间（`@CreateDateColumn`）
- `updatedAt`: 更新时间（`@UpdateDateColumn`）— **关键：用于计算退避等待时间**

## 二、任务调度记录链路

### 1. 任务创建流程与重试次数覆盖链路

```
业务代码调用 JobQueue.add(data, { retries: N }) [job-queue.ts:90-109]
    ↓
创建 Job 对象，设置 retries = options.retries ?? 0 （初始值）
    ↓
JobQueueStrategy.add() [sql-job-queue-strategy.ts:39-49]
    ↓
调用 this.setRetries(queueName, job) 获取最终重试次数
    ↓  [关键覆盖点]
const finalRetries = this.setRetries(queueName, job)
    ↓
转换为 JobRecord，retries 字段使用 finalRetries
    ↓
保存到数据库，返回带 id 的 Job 对象（retries 已被覆盖）
```

**重试次数覆盖链路详细分析**：

覆盖链路配置入口：
- `DefaultJobQueuePlugin.init({ setRetries: (queueName, job) => number })`
  [default-job-queue-plugin.ts:134-142]
- 传入 `SqlJobQueueStrategy` 构造函数
- 父类 `PollingJobQueueStrategy` 构造函数初始化
  [polling-job-queue-strategy.ts:277-288]

默认值：
```typescript
// polling-job-queue-strategy.ts:281
this.setRetries = concurrencyOrConfig.setRetries ?? ((_, job) => job.retries);
```
默认行为是直接返回 `job.retries`，即不覆盖。

覆盖时机：**仅在任务首次添加到队列时调用一次**，不是每次重试时调用。
[sql-job-queue-strategy.ts:46]
```typescript
const newRecord = this.toRecord(job, constrainedData, this.setRetries(job.queueName, job));
```

**纠正**：之前的文档说 retries 是创建时指定的，这不完全准确。策略层的 `setRetries` 钩子可以在任务入队时覆盖任务创建时指定的 retries 值，最终持久化到数据库的是覆盖后的值。

**示例配置** [default-job-queue-plugin.ts:87-95]：
```typescript
DefaultJobQueuePlugin.init({
    setRetries: (queueName, job) => {
        if (queueName === 'send-email') {
            // 覆盖 send-email 队列所有任务的重试次数为 10
            return 10;
        }
        // 其他队列使用任务创建时指定的值
        return job.retries;
    }
})
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
        // 注意：RETRYING 状态不设置 settledAt
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

- `retries` 是**允许的重试次数**（已被策略层覆盖后的最终值）
- `_attempts` 是**已尝试次数**（每次 start() 时自增）
- 总执行次数 = `retries + 1`（首次 + 重试次数）

**示例**：
- `retries = 2`，则总执行次数为 3 次
- 第 1 次失败：`attempts = 1`，`2 >= 1` → RETRYING
- 第 2 次失败：`attempts = 2`，`2 >= 2` → RETRYING
- 第 3 次失败：`attempts = 3`，`2 >= 3` 为 false → FAILED

### 2. RETRYING 任务退避等待时被跳过与再次调度的条件

核心代码：`packages/core/src/plugin/default-job-queue-plugin/sql-job-queue-strategy.ts:115-161`

```typescript
private async getNextAndSetAsRunning(
    manager: EntityManager,
    queueName: string,
    setLock: boolean,
    waitingJobIds: ID[] = [],  // 待跳过的任务 ID 列表
): Promise<Job | undefined> {
    const qb = manager
        .getRepository(JobRecord)
        .createQueryBuilder('record')
        .where('record.queueName = :queueName', { queueName })
        .andWhere(
            new Brackets(qb1 => {
                qb1.where('record.state = :pending', {
                    pending: JobState.PENDING,
                }).orWhere('record.state = :retrying', { retrying: JobState.RETRYING });
            }),
        )
        .orderBy('record.createdAt', 'ASC');

    // 关键：跳过还在退避等待期的任务
    if (waitingJobIds.length) {
        qb.andWhere('record.id NOT IN (:...waitingJobIds)', { waitingJobIds });
    }

    if (setLock) {
        qb.setLock('pessimistic_write');
    }
    const record = await qb.getOne();
    if (record) {
        const job = this.fromRecord(record);
        // 退避时间检查（仅对 RETRYING 状态）
        if (record.state === JobState.RETRYING && typeof this.backOffStrategy === 'function') {
            const msSinceLastFailure = Date.now() - +record.updatedAt;
            const backOffDelayMs = this.backOffStrategy(queueName, record.attempts, job);
            if (msSinceLastFailure < backOffDelayMs) {
                // 还没到重试时间，将此任务 ID 加入 waitingJobIds，递归查询下一个
                return await this.getNextAndSetAsRunning(manager, queueName, setLock, [
                    ...waitingJobIds,
                    record.id,
                ]);
            }
        }
        // 退避时间已到，开始执行
        job.start();
        record.state = JobState.RUNNING;
        await manager.getRepository(JobRecord).save(record, { reload: false });
        return job;
    } else {
        return;
    }
}
```

**被跳过的条件**（同时满足）：
1. 任务状态为 `RETRYING`
2. 配置了 `backoffStrategy` 函数
3. `当前时间 - record.updatedAt < backoffStrategy(queueName, attempts, job)`
   - `record.updatedAt` 是 TypeORM 自动维护的最后更新时间，即上次失败时间
   - `backoffStrategy` 返回需要等待的毫秒数

**再次调度的条件**（满足任一即可）：
1. 退避时间已到：`当前时间 - record.updatedAt >= backoffDelayMs`
2. 所有更旧的 RETRYING 任务都在退避等待期，当前任务是第一个满足条件的
3. 没有其他 PENDING 任务插队（按 createdAt 升序排列）

**关键机制说明**：
- `waitingJobIds` 递归传递，确保同一轮查询中不会重复处理还在退避期的任务
- 被跳过的任务不会阻塞其他任务执行，会继续查询下一个可用任务
- 使用数据库行锁（`pessimistic_write`）防止多 worker 并发抢占同一任务

### 3. 重试间隔（退避策略）

定义位置：`packages/core/src/job-queue/polling-job-queue-strategy.ts:22-67`

```typescript
export type BackoffStrategy = (queueName: string, attemptsMade: number, job: Job) => number;

// 默认退避策略：固定 1000ms
this.backOffStrategy = concurrencyOrConfig.backoffStrategy ?? (() => 1000);
```

**退避策略参数**：
- `queueName`: 队列名称
- `attemptsMade`: 已尝试次数（注意：不是重试次数，是总尝试次数）
- `job`: Job 对象，可访问 job.data 等信息实现更智能的退避

**示例配置** [default-job-queue-plugin.ts:76-86]：
```typescript
DefaultJobQueuePlugin.init({
    backoffStrategy: (queueName, attemptsMade, job) => {
        if (queueName === 'transcode-video') {
            // 指数退避：1s, 4s, 9s, 16s...
            return (attemptsMade ** 2) * 1000;
        }
        // 其他队列固定 1s
        return 1000;
    }
})
```

### 4. 可配置的重试策略

`PollingJobQueueStrategyConfig` 配置项：
- `setRetries`: 可覆盖任务创建时指定的 retries（仅入队时调用一次）
- `backoffStrategy`: 自定义重试间隔策略
- `concurrency`: 并发执行数
- `pollInterval`: 轮询间隔
- `gracefulShutdownTimeout`: 优雅关闭超时时间

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
    retries: Int!      # 允许的重试次数（已覆盖后的最终值）
    attempts: Int!     # 已尝试次数
    # 注意：缺少 updatedAt 字段！
}
```

**重要缺失**：GraphQL Schema 没有暴露 `updatedAt` 字段，虽然 `JobRecord` 数据库表中有这个字段。

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

位置：`packages/dashboard/src/app/routes/_authenticated\_system/job-queue.graphql.ts:3-54`

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

位置：`packages/dashboard/src/app/routes/_authenticated\_system/job-queue.tsx:60-239`

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
        const cancelJobMutation = useMutation({
            mutationFn: (jobId: string) => api.mutate(cancelJobDocument, { jobId }),
            onSuccess: () => {
                refreshRef.current();
            },
        });
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

### 4. 管理端列表自动刷新机制与业务页轮询机制的边界

#### 管理端列表自动刷新（Job Queue 页面）

位置：`packages/dashboard/src/app/routes/_authenticated\_system/job-queue.tsx:121-140`

```typescript
function JobQueuePage() {
    const refreshRef = useRef<() => void>(() => {});
    const [refreshInterval, setRefreshInterval] = useState(10000);  // 默认 10 秒

    useEffect(() => {
        if (refreshInterval === 0) return;

        const interval = setInterval(() => {
            // 下拉菜单打开时暂停刷新，避免交互中断
            if (!isActionMenuOpenRef.current) {
                refreshRef.current();
            }
        }, refreshInterval);

        return () => clearInterval(interval);
    }, [refreshInterval]);
    // ...
}
```

**特点**：
- **用途**：系统管理页面，展示所有队列任务
- **触发方式**：`setInterval` 周期性调用 `refreshRef.current()` 刷新整个列表
- **可配置间隔**：Off / 5s / 10s（默认） / 30s / 60s
- **查询范围**：所有队列的所有任务，支持分页、筛选、排序
- **生命周期**：仅在 Job Queue 页面打开时运行
- **用户控制**：用户可手动选择刷新间隔，或完全关闭

#### 业务页轮询机制（useJobQueuePolling Hook）

位置：`packages/dashboard/src/lib/hooks/use-job-queue-polling.ts:1-160`

```typescript
export function useJobQueuePolling(queueName: string, onComplete: () => void) {
    // 指数退避：500ms → 875ms → 1531ms → 2679ms → 4000ms（上限）
    const pollInterval = isPolling
        ? Math.min(INITIAL_POLL_INTERVAL_MS * Math.pow(1.75, pollCount), MAX_POLL_INTERVAL_MS)
        : false;

    const { data: jobsData } = useQuery({
        queryKey: ['jobQueuePolling', queueName],
        queryFn: () => {
            setPollCount(c => c + 1);
            return api.query(jobListForPollingDocument, {
                options: {
                    filter: { queueName: { eq: queueName } },
                    sort: { createdAt: 'DESC' as const },
                    take: 10,  // 只查最新 10 条
                },
            });
        },
        enabled: isPolling,
        refetchInterval: pollInterval,
    });

    // 完成条件：startTime 之后创建的任务都不再是 PENDING/RUNNING/RETRYING
    const relevantJobs = jobsData?.jobs.items.filter(j => j.createdAt >= startTime) ?? [];
    const hasSettledJob =
        relevantJobs.length > 0 &&
        relevantJobs.every(j => j.state !== 'PENDING' && j.state !== 'RUNNING' && j.state !== 'RETRYING');
    // ...
}
```

**使用示例**（集合详情页）：
`packages/dashboard/src/app/routes/_authenticated\_collections/collections_.$id.tsx:60-63`
```typescript
const { isPolling: pendingFilterApplication, startPolling } = useJobQueuePolling(
    'apply-collection-filters',
    () => queryClient.invalidateQueries({ queryKey: ['PaginatedListDataTable'] }),
);
```

**特点**：
- **用途**：具体业务页面，等待特定队列的任务完成后触发回调（如刷新数据）
- **触发方式**：`useQuery` 的 `refetchInterval` + 指数退避
- **查询范围**：仅查询指定队列的最新 10 条任务
- **时间窗口**：只关心 `startPolling()` 调用时间点（-5s）之后创建的任务
- **完成条件**：相关任务都进入终态（COMPLETED / FAILED / CANCELLED）
- **超时保护**：最大 30 秒，超时自动停止并调用回调
- **状态持久化**：轮询状态保存到 `sessionStorage`，支持页面刷新后恢复
- **生命周期**：业务逻辑驱动，调用 `startPolling()` 开始，完成或超时后自动停止

#### 边界对比

| 维度 | 管理端列表自动刷新 | 业务页轮询机制 |
|------|-------------------|---------------|
| 用途 | 系统监控，全量展示 | 业务流程等待，触发回调 |
| 触发 | 页面生命周期，setInterval | 业务操作后，手动调用 startPolling() |
| 间隔 | 固定间隔，用户可选 | 指数退避，500ms → 4s |
| 查询范围 | 所有队列，支持筛选 | 特定队列，最新 10 条 |
| 停止条件 | 页面卸载或用户关闭 | 任务完成或 30s 超时 |
| 状态持久化 | 无 | sessionStorage 支持刷新恢复 |
| GraphQL 查询 | `jobListDocument`（全字段） | `jobListForPollingDocument`（仅 id/createdAt/state） |

### 5. 界面为何难以直接显示下一次重试时间

要显示"下一次重试时间"，需要前端计算：`nextRetryAt = updatedAt + backoffDelay`

但目前存在多层障碍：

#### 障碍 1：GraphQL API 层面缺少 updatedAt 字段

`packages/core/src/api/schema/admin-api/job.api.graphql` 中的 `Job` 类型没有 `updatedAt` 字段。

虽然数据库表 `JobRecord` 有 `updatedAt`（继承自 `VendureEntity`），但：
- `JobConfig` 接口没有 `updatedAt` 字段定义
- `Job` 类构造函数不接收 `updatedAt`
- `fromRecord()` 方法只是简单 `new Job(jobRecord)`，没有传递 `updatedAt`

```typescript
// sql-job-queue-strategy.ts:246-248
private fromRecord(this: void, jobRecord: JobRecord): Job<any> {
    return new Job<any>(jobRecord);  // updatedAt 被忽略
}
```

#### 障碍 2：退避策略是服务端内部实现

`backoffStrategy` 是 `JobQueueStrategy` 的配置项，仅存在于服务端：
- 前端无法访问策略配置
- 不同队列可能有不同的退避策略
- 退避策略可以是动态函数（根据 `attemptsMade` 或 `job.data` 计算）
- 前端无法模拟计算

#### 障碍 3：Job 类不追踪失败时间

`Job.fail()` 方法只设置 `_error` 和 `_state`，不记录失败时间戳：
- 没有 `failedAt` 字段
- `updatedAt` 是 TypeORM 自动维护的，不是业务层面的概念
- `settledAt` 仅在终态（COMPLETED/FAILED/CANCELLED）时设置，RETRYING 状态不设置

#### 障碍 4：架构设计上的关注点分离

- 退避调度是队列策略的内部实现细节
- 管理端仅展示状态和统计信息
- 下一次重试时间属于调度细节，设计上没有打算暴露给前端

#### 解决方案（如需实现）

如果需要显示下一次重试时间，需要：

1. **API 扩展**：在 GraphQL Schema 中添加 `nextRetryAt: DateTime` 字段
2. **服务端计算**：在 `findMany()` / `findOne()` 查询时计算：
   ```typescript
   if (job.state === JobState.RETRYING) {
       const backoffMs = this.backOffStrategy(job.queueName, job.attempts, job);
       (job as any).nextRetryAt = new Date(+job.updatedAt + backoffMs);
   }
   ```
3. **类型扩展**：在 `Job` 类或 GraphQL 解析层添加该字段

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
    ├─ retries >= attempts → RETRYING (不设置 settledAt)
    │   └─ 等待 updatedAt + backoffDelay → 再次调度 → RUNNING (attempts++)
    │                          ↓  ... 循环 ...
    └─ retries < attempts → FAILED (settledAt)
```

### 数据流转链路

```
业务代码调用 JobQueue.add(data, { retries: N })
    ↓  [内存]
创建 Job 对象: { retries: N, attempts: 0, state: PENDING }
    ↓  [策略层覆盖]
finalRetries = setRetries(queueName, job)  // 可覆盖 N
    ↓  [持久化]
SqlJobQueueStrategy.add() → 保存 JobRecord(retries: finalRetries) 到数据库
    ↓  [调度]
ActiveQueue 轮询 → next() → 查找 PENDING/RETRYING 任务
    ↓  [退避检查]
RETRYING 任务检查 updatedAt + backoffDelay，未到时间则跳过
    ↓  [执行]
Job.start() → state=RUNNING, attempts++, startedAt=now
    ↓  [更新]
SqlJobQueueStrategy.update() → 更新数据库
    ↓  [业务逻辑]
执行 process(job) 函数
    ↓  [结果处理]
成功 → job.complete(result) → state=COMPLETED, settledAt=now
失败 → job.fail(error)
    ├─ 还有重试 → state=RETRYING (updatedAt 自动更新)
    └─ 重试用尽 → state=FAILED, settledAt=now
    ↓  [更新]
SqlJobQueueStrategy.update() → 更新数据库
    ↓  [展示]
GraphQL Query jobs() → 查询 JobRecord 列表（缺少 updatedAt）
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
- 下一次什么时候重试？

### 2. 原因

| 问题 | 根源 |
|------|------|
| 看不到重试次数 | `retries` 和 `attempts` 字段默认隐藏 |
| 看不到错误原因 | `error` 字段默认隐藏 |
| 看不到下一次重试时间 | `updatedAt` 未通过 API 暴露，`backoffStrategy` 是服务端内部实现 |
| 状态徽章信息不足 | 仅显示状态文本，没有尝试次数信息 |

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
| `updatedAt` | 最后更新时间（用于退避计算） | **未通过 API 暴露** |

### 4. 改进建议

#### 建议 1：显示重试进度
在 `RETRYING` 状态的徽章旁显示 `attempts / (retries + 1)` 信息：
```typescript
// 伪代码
{row.original.state}
{row.original.state === 'RETRYING' && ` (${row.original.attempts}/${row.original.retries + 1})`}
```
效果：`RETRYING (2/3)`

#### 建议 2：默认显示 attempts 和 retries 列
或将其合并到状态列中显示。

#### 建议 3：悬停显示错误信息
在 `RETRYING` 或 `FAILED` 状态的徽章上添加 tooltip，显示 `error` 字段内容。

#### 建议 4（进阶）：显示下一次重试时间
如"障碍分析"所述，需要 API 扩展，在服务端计算 `nextRetryAt` 并暴露给前端。
