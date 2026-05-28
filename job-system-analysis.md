# Immich Job 系统深度分析

## 一、JobRepository.setup 服务扫描与校验逻辑

`JobRepository.setup()` 是 Immich 任务系统的核心初始化方法，位于 [job.repository.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/job.repository.ts#L36-L84)。该方法在应用启动时被 [QueueService.onBootstrap()](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/queue.service.ts#L78-L80) 调用，执行两项关键校验。

### 1.1 核心执行流程

```typescript
setup(services: (new (...args: any[]) => unknown)[]) {
  const reflector = this.moduleRef.get(Reflector, { strict: false });

  // 第一阶段：服务扫描 - 发现所有 @OnJob 装饰的方法
  for (const Service of services) {
    const instance = this.moduleRef.get<any>(Service);
    for (const methodName of getMethodNames(instance)) {
      const handler = instance[methodName];
      const config = reflector.get<JobConfig>(MetadataKey.JobConfig, handler);
      if (!config) continue;

      // 校验1：每个 JobName 只能有一个 handler
      if (this.handlers[jobName]) {
        // 抛出重复注册错误
      }

      // 注册 handler
      this.handlers[jobName] = { label, jobName, queueName, handler: handler.bind(instance) };
    }
  }

  // 第二阶段：完整性校验 - 确保所有 JobName 都有 handler
  for (const [jobKey, jobName] of Object.entries(JobName)) {
    const item = this.handlers[jobName];
    if (!item) {
      // 抛出缺少 handler 错误
    }
  }
}
```

---

## 二、场景分析：启动时报错机制

### 2.1 场景一：新增 JobName 但忘记添加 @OnJob

**问题描述**：在 [enum.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/enum.ts#L793-L870) 的 `JobName` 枚举中新增 `MyCleanup = 'MyCleanup'`，但没有给任何 service 方法添加 `@OnJob` 装饰器。

**启动时的报错**：

在 `setup()` 的第二阶段（第74-83行），代码会遍历 `JobName` 枚举的所有值，检查是否在 `this.handlers` 中存在对应的注册项。当发现 `JobName.MyCleanup` 没有对应的 handler 时：

1. 记录错误日志：
   ```
   [JobRepository] Failed to find job handler for Job.MyCleanup ("MyCleanup"). 
   Make sure to add the @OnJob({ name: JobName.MyCleanup, queue: QueueName.XYZ }) 
   decorator for the new job.
   ```

2. 抛出 `ImmichStartupError` 异常，导致应用启动失败。

**报错源码位置**：[job.repository.ts#L76-L82](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/job.repository.ts#L76-L82)

### 2.2 场景二：两个 Service 为同一个 JobName 注册 Handler

**问题描述**：例如 `CleanupService.handleMyCleanup()` 和 `OtherService.alsoHandleMyCleanup()` 都使用了 `@OnJob({ name: JobName.MyCleanup, ... })`。

**启动时的报错**：

在 `setup()` 的第一阶段（第53-60行），当扫描到第二个 handler 时，会检查 `this.handlers[jobName]` 是否已存在：

1. 记录错误日志：
   ```
   [JobRepository] Failed to add job handler for OtherService.alsoHandleMyCleanup. 
   JobName.MyCleanup is already handled by CleanupService.handleMyCleanup.
   ```

2. 抛出 `ImmichStartupError` 异常，导致应用启动失败。

**报错源码位置**：[job.repository.ts#L53-L59](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/job.repository.ts#L53-L59)

---

## 三、为什么所有 JobName 都必须有 Handler？

这是一个**防御性编程**设计，核心原因在于 `queueAll` 方法的实现依赖 `handlers` 映射反查队列名。

### 3.1 运行时依赖链

当调用 `jobRepository.queueAll(jobs)` 时，方法内部需要知道每个 job 应该发送到哪个队列。如果没有强制要求所有 JobName 都有 handler，可能导致：

1. **运行时 TypeError**：如果某个 JobName 没有 handler，`getQueueName()` 会返回 `undefined`，后续访问 `.queueName` 会抛出异常
2. **静默失败**：job 可能被发送到错误的队列或丢失，难以调试
3. **类型安全破坏**：TypeScript 类型系统无法在编译时检测到缺失的 handler

### 3.2 设计权衡

| 设计选择 | 优点 | 缺点 |
|---------|------|------|
| **启动时强制校验** | 早发现问题，运行时安全 | 新增 JobName 必须同时添加 handler |
| 运行时动态查找 | 灵活，允许动态注册 | 错误延迟到运行时，难以排查 |

Immich 选择了**启动时强制校验**，确保系统在运行前处于一致状态。

---

## 四、queueAll 如何依赖 Handler 反查队列

### 4.1 核心方法：getQueueName

```typescript
private getQueueName(name: JobName) {
  return (this.handlers[name] as JobMapItem).queueName;
}
```

**源码位置**：[job.repository.ts#L155-L157](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/job.repository.ts#L155-L157)

### 4.2 queueAll 完整流程

```typescript
async queueAll(items: JobItem[]): Promise<void> {
  if (items.length === 0) return;

  const promises = [];
  const itemsByQueue = {} as Record<string, JobItem[]>;

  for (const item of items) {
    // 关键：通过 handler 反查队列名
    const queueName = this.getQueueName(item.name);
    
    const job = {
      name: item.name,
      data: item.data || {},
      options: this.getJobOptions(item) || undefined,
    };

    // 有特殊选项（jobId/deduplication）的 job 需要单独 add()
    if (job.options?.jobId || job.options?.deduplication) {
      promises.push(this.getQueue(queueName).add(item.name, item.data, job.options));
    } else {
      // 普通 job 按队列分组，使用 addBulk() 提高性能
      itemsByQueue[queueName] = itemsByQueue[queueName] || [];
      itemsByQueue[queueName].push(job);
    }
  }

  // 批量添加分组后的 jobs
  for (const [queueName, jobs] of Object.entries(itemsByQueue)) {
    const queue = this.getQueue(queueName as QueueName);
    promises.push(queue.addBulk(jobs));
  }

  await Promise.all(promises);
}
```

**源码位置**：[job.repository.ts#L159-L189](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/job.repository.ts#L159-L189)

### 4.3 性能优化设计

1. **按队列分组**：相同队列的 jobs 使用 `addBulk()` 批量添加，减少 I/O 次数
2. **特殊处理分离**：需要 `jobId` 或 `deduplication` 的 job 必须使用 `add()` 单独处理（BullMQ 的 `addBulk()` 不支持这些选项）

---

## 五、特殊 JobName 在 getJobOptions 中的行为分析

`getJobOptions()` 方法为特定 JobName 提供定制化的 BullMQ 作业选项。

**源码位置**：[job.repository.ts#L218-L245](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/job.repository.ts#L218-L245)

### 5.1 FacialRecognitionQueueAll

```typescript
case JobName.FacialRecognitionQueueAll: {
  return { deduplication: { id: JobName.FacialRecognitionQueueAll } };
}
```

**特殊行为**：启用基于 ID 的去重

**原因分析**：

1. **重量级任务**：该 job 会扫描所有人脸数据，可能队列数千个 `FacialRecognition` 子任务，消耗大量计算资源
2. **多触发源**：该 job 可能被多个来源触发：
   - 用户在管理界面手动点击"重新识别"
   - 定时任务（NightlyJobs）
   - 上传新资产后的自动触发
3. **去重必要性**：避免同一时间有多个全量扫描任务在运行，防止：
   - 数据库连接耗尽
   - 机器学习服务过载
   - 重复计算浪费资源

**对应 Handler**：[person.service.ts#L403-L438](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/person.service.ts#L403-L438)

### 5.2 DatabaseBackup

```typescript
case JobName.DatabaseBackup: {
  return { deduplication: { id: JobName.DatabaseBackup } };
}
```

**特殊行为**：启用基于 ID 的去重

**原因分析**：

1. **极高资源消耗**：数据库备份操作：
   - 占用大量 CPU（压缩）
   - 占用大量 I/O（读取整个数据库）
   - 占用大量磁盘空间（备份文件）
   - 可能导致数据库性能下降
2. **多触发源**：
   - 定时备份任务（cron）
   - 用户手动触发备份
   - 系统维护前自动备份
3. **去重必要性**：
   - 防止多个备份同时运行导致磁盘空间耗尽
   - 确保备份完整性（并发备份可能导致数据不一致）
   - 避免数据库同时承受多个备份的读取压力

**对应 Handler**：[database-backup.service.ts#L92-L106](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/database-backup.service.ts#L92-L106)

### 5.3 PersonGenerateThumbnail

```typescript
case JobName.PersonGenerateThumbnail: {
  return { priority: 1 };
}
```

**特殊行为**：设置优先级为 1（默认优先级为 0，数值越大优先级越高）

**原因分析**：

1. **用户体验敏感**：人物缩略图直接显示在 UI 的"人物"页面，用户打开该页面时期望立即看到人物头像
2. **与 Asset 缩略图竞争**：`ThumbnailGeneration` 队列中可能有大量普通资产缩略图任务在排队
3. **高优先级必要性**：
   - 确保人物缩略图优先生成，提升用户体验
   - 避免用户看到空白的人物头像占位符
   - 人物缩略图通常数量远少于资产缩略图，优先处理不会显著影响整体性能

**对应 Handler**：[media.service.ts#L407-L460](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/media.service.ts#L407-L460)

### 5.4 其他特殊 JobName 对比

| JobName | 特殊选项 | 目的 |
|---------|---------|------|
| `NotifyAlbumUpdate` | `jobId: ${id}/${recipientId}` | 避免重复通知同一用户 |
| `StorageTemplateMigrationSingle` | `jobId: item.data.id` | 避免同一资产重复迁移 |
| `VersionCheck` | `deduplication: { id: ... }` | 避免重复版本检查 |

---

## 六、总结

| 机制 | 设计目的 | 代码位置 |
|------|---------|---------|
| **重复 Handler 校验** | 确保 job 语义明确，避免冲突 | [job.repository.ts#L53-L60](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/job.repository.ts#L53-L60) |
| **缺失 Handler 校验** | 运行时安全，防止 queueAll 失败 | [job.repository.ts#L74-L83](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/job.repository.ts#L74-L83) |
| **Handler 反查队列** | 简化 API，调用方无需指定队列 | [job.repository.ts#L155-L157](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/job.repository.ts#L155-L157) |
| **FacialRecognitionQueueAll 去重** | 保护 ML 服务和数据库 | [job.repository.ts#L232-L234](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/job.repository.ts#L232-L234) |
| **DatabaseBackup 去重** | 保护数据库和磁盘资源 | [job.repository.ts#L238-L240](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/job.repository.ts#L238-L240) |
| **PersonGenerateThumbnail 高优先级** | 提升用户体验 | [job.repository.ts#L229-L231](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/job.repository.ts#L229-L231) |

Immich 的 Job 系统设计体现了**"安全优先"**和**"用户体验优先"**的原则，通过启动时强校验和运行时细粒度控制，确保系统稳定可靠。
