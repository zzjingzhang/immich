# 资产删除与恢复流程分析

## 一、三个入口流程比较

### 1. 回收站页面恢复选中资产
**路径**: `POST /trash/restore/assets`

| 层级 | 代码位置 | 关键操作 |
|------|----------|----------|
| Controller | [trash.controller.ts#L40-L50](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/controllers/trash.controller.ts#L40-L50) | `restoreAssets()` 接收 `BulkIdsDto` |
| Service | [trash.service.ts#L12-L25](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/trash.service.ts#L12-L25) | `restoreAssets()` 权限校验后调用 `trashRepository.restoreAll(ids)` |
| Repository | [trash.repository.ts#L39-L52](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/trash.repository.ts#L39-L52) | `restoreAll()` 更新 `status=Active, deletedAt=null` |

**核心SQL**:
```sql
UPDATE asset 
SET status = 'active', deletedAt = null 
WHERE status = 'trashed' AND id IN (...)
```

---

### 2. restoreTrash 恢复全部
**路径**: `POST /trash/restore`

| 层级 | 代码位置 | 关键操作 |
|------|----------|----------|
| Controller | [trash.controller.ts#L28-L38](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/controllers/trash.controller.ts#L28-L38) | `restoreTrash()` |
| Service | [trash.service.ts#L27-L33](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/trash.service.ts#L27-L33) | `restore()` 调用 `trashRepository.restore(userId)` |
| Repository | [trash.repository.ts#L15-L24](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/trash.repository.ts#L15-L24) | `restore()` 按用户恢复所有 `status=Trashed` 的资产 |

**核心SQL**:
```sql
UPDATE asset 
SET status = 'active', deletedAt = null 
WHERE ownerId = ? AND status = 'trashed'
```

---

### 3. emptyTrash 清空全部
**路径**: `POST /trash/empty`

| 层级 | 代码位置 | 关键操作 |
|------|----------|----------|
| Controller | [trash.controller.ts#L16-L26](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/controllers/trash.controller.ts#L16-L26) | `emptyTrash()` |
| Service | [trash.service.ts#L35-L41](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/trash.service.ts#L35-L41) | `empty()` 先更新状态，再排队 `JobName.AssetEmptyTrash` |
| Repository | [trash.repository.ts#L27-L36](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/trash.repository.ts#L27-L36) | `empty()` 将 `Trashed` 改为 `Deleted` |
| Job | [trash.service.ts#L48-L71](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/trash.service.ts#L48-L71) | `handleEmptyTrash()` 遍历 `status=Deleted` 资产排队删除 |

**核心SQL（第一步）**:
```sql
UPDATE asset 
SET status = 'deleted' 
WHERE ownerId = ? AND status = 'trashed'
```

---

## 二、Duplicate Resolve 在 trash.enabled=false 时的路径

**代码位置**: [duplicate.service.ts#L218-L233](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/duplicate.service.ts#L218-L233)

```typescript
if (idsToTrash.length > 0) {
  const { trash } = await this.getConfig({ withCache: true });
  const force = !trash.enabled;  // 关键：trash.enabled=false 时 force=true

  await this.assetRepository.updateAll(idsToTrash, {
    deletedAt: new Date(),
    status: force ? AssetStatus.Deleted : AssetStatus.Trashed,  // 直接设为 Deleted
    duplicateId: null,
  });

  await this.eventRepository.emit(force ? 'AssetDeleteAll' : 'AssetTrashAll', { ... });
}
```

**后续流程**:
1. 触发 `AssetDeleteAll` 事件
2. [TrashService.onAssetsDelete()](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/trash.service.ts#L43-L46) 监听事件，排队 `JobName.AssetEmptyTrash`
3. `handleEmptyTrash()` 处理所有 `status=Deleted` 的资产

---

## 三、状态转换时机分析

### 1. 什么时候把 asset.status/deletedAt 改回 Active

**触发场景**:
- **恢复选中资产**: `TrashRepository.restoreAll()` [trash.repository.ts#L48](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/trash.repository.ts#L48)
- **恢复全部**: `TrashRepository.restore()` [trash.repository.ts#L20](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/trash.repository.ts#L20)

**共同点**: 都是将 `status=Trashed` 且 `deletedAt` 有值的资产，改为 `status=Active` 且 `deletedAt=null`。

---

### 2. 什么时候把 Trashed 改成 Deleted

| 触发场景 | 代码位置 | 条件 |
|----------|----------|------|
| emptyTrash 清空回收站 | [trash.repository.ts#L32](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/trash.repository.ts#L32) | 用户主动调用 empty |
| AssetService.deleteAll(force=true) | [asset.service.ts#L376](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/asset.service.ts#L376) | 调用时传入 `force=true` |
| Duplicate Resolve (trash.enabled=false) | [duplicate.service.ts#L225](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/duplicate.service.ts#L225) | 配置 `trash.enabled=false` |
| 定时任务自动清理 (trash.days 到期) | [asset.service.ts#L273-L305](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/asset.service.ts#L273-L305) | `handleAssetDeletionCheck` 定期检查 `deletedAt` 超过配置天数 |

> **注意**: `Trashed -> Deleted` 只是数据库状态变更，还没有真正删除文件或数据库记录。

---

## 四、TrashService.handleEmptyTrash 何时真正排 JobName.AssetDelete

### 触发 `handleEmptyTrash` 的三个途径

| 途径 | 触发点 | 代码位置 |
|------|--------|----------|
| 1. 用户主动 emptyTrash | `TrashService.empty()` 排队 `JobName.AssetEmptyTrash` | [trash.service.ts#L38](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/trash.service.ts#L38) |
| 2. AssetDeleteAll 事件 | `TrashService.onAssetsDelete()` 监听事件后排队 | [trash.service.ts#L45](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/trash.service.ts#L45) |
| 3. 定时任务 AssetDeleteCheck | 直接排 `JobName.AssetDelete`（不经过 AssetEmptyTrash） | [asset.service.ts#L286-L288](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/asset.service.ts#L286-L288) |

### handleEmptyTrash 内部逻辑

**代码位置**: [trash.service.ts#L48-L84](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/trash.service.ts#L48-L84)

```typescript
@OnJob({ name: JobName.AssetEmptyTrash, queue: QueueName.BackgroundTask })
async handleEmptyTrash() {
  const assets = this.trashRepository.getDeletedIds();  // 查询 status = 'deleted' 的资产
  
  for await (const { id } of assets) {
    batch.push(id);
    if (batch.length === JOBS_ASSET_PAGINATION_SIZE) {
      await this.handleBatch(batch);  // 批量排队 JobName.AssetDelete
    }
  }
}

private async handleBatch(ids: string[]) {
  await this.jobRepository.queueAll(
    ids.map((assetId) => ({
      name: JobName.AssetDelete,
      data: {
        id: assetId,
        deleteOnDisk: true,  // 关键：这里传入 deleteOnDisk: true
      },
    })),
  );
}
```

**关键条件**: `getDeletedIds()` 查询的是 `status = 'deleted'` 的资产，而不是 `status = 'trashed'`。

---

## 五、deleteOnDisk: true 传入位置汇总

| 位置 | 代码文件 | 行号 | 场景 |
|------|----------|------|------|
| 1 | [trash.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/trash.service.ts#L80) | L80 | `handleBatch()` 处理 emptyTrash 时 |
| 2 | [asset.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/asset.service.ts#L345) | L345 | `handleAssetDeletion()` 递归删除关联的 livePhotoVideoId 时传递原值 |
| 3 | [metadata.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/metadata.service.ts#L791) | L791 | 上传新 motion photo 时删除旧的 video 资产 |
| 4 | [asset.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/asset.service.ts#L287) | L287 | `handleAssetDeletionCheck` 定时任务：`deleteOnDisk: !isOffline` |

### deleteOnDisk 的作用

**代码位置**: [asset.service.ts#L361-L365](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/asset.service.ts#L361-L365)

```typescript
if (deleteOnDisk && !asset.isOffline) {
  files.push(assetFiles.sidecarFile?.path, asset.originalPath);
}

await this.jobRepository.queue({ name: JobName.FileDelete, data: { files: files.filter(Boolean) } });
```

- **deleteOnDisk=true**: 删除缩略图 + 原始文件 + sidecar
- **deleteOnDisk=false**: 只删除缩略图等派生文件，保留原始文件（用于外部库离线文件等场景）

---

## 六、完整流程图

```
用户删除 (force=false)
    ↓
AssetStatus.Trashed + deletedAt=now
    ↓
[可选] 用户恢复 → status=Active, deletedAt=null
    ↓ 或 等待 trash.days 到期 或 用户 emptyTrash
AssetStatus.Deleted
    ↓
JobName.AssetEmptyTrash (handleEmptyTrash)
    ↓
批量排队 JobName.AssetDelete (deleteOnDisk=true)
    ↓
handleAssetDeletion: 删除数据库记录 + 删除磁盘文件
```

---

## 七、关键状态枚举

| 状态 | 说明 |
|------|------|
| `Active` | 正常资产 |
| `Trashed` | 在回收站中，可恢复 |
| `Deleted` | 已标记删除，等待 Job 处理，不可恢复 |

---

## 八、关键对比总结

| 操作 | status 变化 | deletedAt | 是否排删除Job | deleteOnDisk |
|------|-------------|-----------|---------------|--------------|
| 恢复选中资产 | Trashed → Active | 设为 null | ❌ | - |
| 恢复全部 | Trashed → Active | 设为 null | ❌ | - |
| emptyTrash (第一步) | Trashed → Deleted | 保留 | ❌ | - |
| emptyTrash (第二步Job) | - | - | ✅ JobName.AssetDelete | true |
| deleteAll(force=false) | Active → Trashed | 设为 now | ❌ | - |
| deleteAll(force=true) | Active → Deleted | 设为 now | ✅ 经 AssetDeleteAll 事件 | true |
| duplicate(trash.enabled=false) | Active → Deleted | 设为 now | ✅ 经 AssetDeleteAll 事件 | true |
| duplicate(trash.enabled=true) | Active → Trashed | 设为 now | ❌ | - |
| 定时任务过期清理 | Trashed → Deleted | 保留原值 | ✅ 直接排 AssetDelete | !isOffline |
