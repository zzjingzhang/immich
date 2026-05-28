# External Library Import Path 场景分析

本文档分析 Immich 项目中 External Library 的 Import Path 在6种不同场景下的行为。

## 核心代码位置

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 导入路径验证 | [library.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L283-L322) | `validateImportPath` |
| 更新库 | [library.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L331-L347) | `update` |
| 文件监控 | [library.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L89-L162) | `watch` |
| 同步新文件队列 | [library.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L615-L683) | `handleQueueSyncFiles` |
| 同步已有资产队列 | [library.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L699-L775) | `handleQueueSyncAssets` |
| 资产状态检查 | [library.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L480-L574) | `handleSyncAssets` |
| 检测离线资产 | [asset.repository.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/asset.repository.ts#L1025-L1049) | `detectOfflineExternalAssets` |
| Immich路径判断 | [storage.core.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/cores/storage.core.ts#L136-L144) | `StorageCore.isImmichPath` |

---

## 场景1：路径是Immich自己的upload/profile路径

### 代码原理

`StorageCore.isImmichPath()` 检查路径是否在 Immich 的媒体目录下：

```typescript
// [storage.core.ts#L136-L144](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/cores/storage.core.ts#L136-L144)
static isImmichPath(path: string) {
  const resolvedPath = resolve(path);
  const resolvedAppMediaLocation = StorageCore.getMediaLocation();
  const normalizedPath = resolvedPath.endsWith('/') ? resolvedPath : resolvedPath + '/';
  const normalizedAppMediaLocation = resolvedAppMediaLocation.endsWith('/')
    ? resolvedAppMediaLocation
    : resolvedAppMediaLocation + '/';
  return normalizedPath.startsWith(normalizedAppMediaLocation);
}
```

### 行为分析

| 组件 | 行为 |
|------|------|
| **validateImportPath 返回** | `{ importPath: "path", isValid: false, message: "Cannot use media upload folder for external libraries" }` |
| **update 是否拒绝保存** | ✅ **拒绝**。`update()` 中调用 `validate()`，只要有一个路径 `isValid: false`，就抛出 `BadRequestException`。 |
| **watch 是否排队** | ❌ **不会**。因为 update 失败，路径不会被保存，watch 不会触发。 |
| **handleQueueSyncFiles 过滤** | ❌ **不会进入**。路径不会被保存到 `validImportPaths` 中，会被跳过并记录 warning。 |
| **handleQueueSyncAssets offline/online** | ❌ **不相关**。路径无效，不会有资产关联到此路径。 |

### 关键代码验证

```typescript
// [library.service.ts#L288-L291](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L288-L291)
if (StorageCore.isImmichPath(importPath)) {
  validation.message = 'Cannot use media upload folder for external libraries';
  return validation;
}

// [library.service.ts#L334-L343](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L334-L343)
if (dto.importPaths) {
  const validation = await this.validate(id, { importPaths: dto.importPaths });
  if (validation.importPaths) {
    for (const path of validation.importPaths) {
      if (!path.isValid) {
        throw new BadRequestException(`Invalid import path: ${path.message}`);
      }
    }
  }
}
```

---

## 场景2：路径是相对路径

### 代码原理

使用 Node.js 的 `path.isAbsolute()` 判断路径是否为绝对路径：

```typescript
// [library.service.ts#L293-L296](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L293-L296)
if (!isAbsolute(importPath)) {
  validation.message = `Import path must be absolute, try ${path.resolve(importPath)}`;
  return validation;
}
```

### 行为分析

| 组件 | 行为 |
|------|------|
| **validateImportPath 返回** | `{ importPath: "relative/path", isValid: false, message: "Import path must be absolute, try /resolved/absolute/path" }` |
| **update 是否拒绝保存** | ✅ **拒绝**。同上，`isValid: false` 会导致抛出异常。 |
| **watch 是否排队** | ❌ **不会**。路径不会被保存。 |
| **handleQueueSyncFiles 过滤** | ❌ **不会进入**。`validateImportPath` 返回 `isValid: false`，会被跳过。 |
| **handleQueueSyncAssets offline/online** | ❌ **不相关**。 |

---

## 场景3：路径不可读

### 代码原理

通过 `stat()` 和 `checkFileExists(path, R_OK)` 检查路径是否存在且可读：

```typescript
// [library.service.ts#L298-L318](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L298-L318)
try {
  const stat = await this.storageRepository.stat(importPath);
  if (!stat.isDirectory()) {
    validation.message = 'Not a directory';
    return validation;
  }
} catch (error: any) {
  if (error.code === 'ENOENT') {
    validation.message = 'Path does not exist (ENOENT)';
    return validation;
  }
  validation.message = String(error);
  return validation;
}

const access = await this.storageRepository.checkFileExists(importPath, R_OK);
if (!access) {
  validation.message = 'Lacking read permission for folder';
  return validation;
}
```

### 行为分析

| 组件 | 行为 |
|------|------|
| **validateImportPath 返回** | 可能的几种情况：<br>`isValid: false, message: "Path does not exist (ENOENT)"`<br>`isValid: false, message: "Lacking read permission for folder"`<br>`isValid: false, message: "Not a directory"` |
| **update 是否拒绝保存** | ✅ **拒绝**。同上，验证失败抛出异常。 |
| **watch 是否排队** | ❌ **不会**（update 阶段）。但如果路径在保存后变为不可读，请看下面的 handleQueueSyncFiles 行为。 |
| **handleQueueSyncFiles 过滤** | ✅ **会跳过**。在 `handleQueueSyncFiles` 中会再次调用 `validateImportPath`，无效路径会被加入 warning 日志并跳过。 |
| **handleQueueSyncAssets offline/online** | ✅ **会标记为 offline**。`detectOfflineExternalAssets` 会先将不在 importPath 下的资产标记为 offline；然后 `handleSyncAssets` 中 `stat()` 失败返回 `AssetSyncResult.OFFLINE`，标记 `isOffline: true, deletedAt: new Date()`。 |

### 关键代码验证 - handleQueueSyncFiles 中的过滤

```typescript
// [library.service.ts#L625-L640](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L625-L640)
const validImportPaths: string[] = [];
for (const importPath of library.importPaths) {
  const validation = await this.validateImportPath(importPath);
  if (validation.isValid) {
    validImportPaths.push(path.normalize(importPath));
  } else {
    this.logger.warn(`Skipping invalid import path: ${importPath}. Reason: ${validation.message}`);
  }
}

if (validImportPaths.length === 0) {
  this.logger.warn(`No valid import paths found for library ${library.id}`);
  return JobStatus.Skipped;
}
```

### 关键代码验证 - 资产离线逻辑

```typescript
// [library.service.ts#L576-L613](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L576-L613)
private checkExistingAsset(asset, stat: Stats | null): AssetSyncResult {
  if (!stat) {
    // 文件不存在或权限错误
    if (asset.isOffline) {
      return AssetSyncResult.DO_NOTHING;
    }
    return AssetSyncResult.OFFLINE;
  }
  // ...
}

// [library.service.ts#L545-L547](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L545-L547)
if (assetIdsToOffline.length > 0) {
  promises.push(this.assetRepository.updateAll(assetIdsToOffline, { isOffline: true, deletedAt: new Date() }));
}
```

---

## 场景4：路径可读但被 exclusionPatterns 命中

### 代码原理

排除模式在多处使用 `picomatch` 进行匹配：

```typescript
// watch 中的匹配
const matcher = picomatch(`**/*{${extensions}}`, {
  nocase: true,
  ignore: library.exclusionPatterns,
});

// walk 中的排除
const stream = globStream(globbedPaths, {
  ignore: exclusionPatterns,
});

// 离线检测使用 SQL LIKE
const exclusions = exclusionPatterns.map((pattern) => globToSqlPattern(pattern));
```

### 行为分析

| 组件 | 行为 |
|------|------|
| **validateImportPath 返回** | `{ importPath: "path", isValid: true }`。验证只检查路径本身，不检查 exclusionPatterns。 |
| **update 是否拒绝保存** | ✅ **允许保存**。路径本身有效，exclusionPatterns 是单独的配置。 |
| **watch 是否排队** | ❌ **不会触发 import**。watch 中的 `matcher` 会排除匹配 exclusionPatterns 的文件，`handler` 不会被调用。但文件删除事件 `deletionHandler` **没有** matcher 过滤，会直接排队 `LibraryRemoveAsset`。 |
| **handleQueueSyncFiles 过滤** | ✅ **会过滤**。`storageRepository.walk()` 的 `ignore` 参数会排除匹配的文件；另外 `filterNewExternalAssetPaths` 会过滤已存在的资产路径。 |
| **handleQueueSyncAssets offline/online** | ✅ **会标记为 offline**。`detectOfflineExternalAssets` 会用 SQL LIKE 匹配 exclusionPatterns，直接将匹配的资产标记为 offline。在 `handleSyncAssets` 中，offline 资产即使文件恢复，如果仍在 exclusionPatterns 中，也不会恢复 online。 |

### 关键代码验证 - watch 中的排除

```typescript
// [library.service.ts#L103-L121](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L103-L121)
const matcher = picomatch(`**/*{${mimeTypes.getSupportedFileExtensions().join(',')}}`, {
  nocase: true,
  ignore: library.exclusionPatterns,
});

const handler = async (event: string, path: string) => {
  if (matcher(path)) {
    // 匹配成功才会排队
    await this.jobRepository.queue({ name: JobName.LibrarySyncFiles, ... });
  } else {
    this.logger.verbose(`Ignoring file ${event} event for ${path}`);
  }
};
```

### 关键代码验证 - 离线检测中的排除

```typescript
// [asset.repository.ts#L1025-L1049](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/asset.repository.ts#L1025-L1049)
async detectOfflineExternalAssets(libraryId, importPaths, exclusionPatterns) {
  const paths = importPaths.map((importPath) => `${importPath}%`);
  const exclusions = exclusionPatterns.map((pattern) => globToSqlPattern(pattern));

  return this.db
    .updateTable('asset')
    .set({ isOffline: true, deletedAt: new Date() })
    .where('isOffline', '=', false)
    .where('isExternal', '=', true)
    .where('libraryId', '=', libraryId)
    .where((eb) => eb.or([
      eb.not(eb.or(paths.map((path) => eb('originalPath', 'like', path)))),
      eb.or(exclusions.map((path) => eb('originalPath', 'like', path))),
    ]))
    .executeTakeFirstOrThrow();
}
```

### 关键代码验证 - offline 资产的恢复检查

```typescript
// [library.service.ts#L513-L540](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L513-L540)
case AssetSyncResult.CHECK_OFFLINE: {
  const isInImportPath = job.importPaths.find((path) => asset.originalPath.startsWith(path));
  if (!isInImportPath) break;

  const isExcluded = job.exclusionPatterns.some((pattern) => picomatch.isMatch(asset.originalPath, pattern));
  if (!isExcluded) {
    // 只有不在 exclusionPatterns 中才恢复 online
    assetIdsToOnline.push(asset.id);
  }
  break;
}
```

---

## 场景5：路径里已有资产已离线

### 代码原理

资产有 `isOffline` 字段，`checkExistingAsset()` 根据文件状态返回不同的 `AssetSyncResult`：

```typescript
export enum AssetSyncResult {
  DO_NOTHING,   // 0
  UPDATE,       // 1
  OFFLINE,      // 2
  CHECK_OFFLINE, // 3
}
```

### 行为分析

| 组件 | 行为 |
|------|------|
| **validateImportPath 返回** | `{ isValid: true }`。路径本身有效。 |
| **update 是否拒绝保存** | ✅ **允许保存**。 |
| **watch 是否排队** | ✅ **会触发重新扫描**。文件 add/change 事件会触发 `handler`，如果文件不在 exclusionPatterns 中，会排队 `LibrarySyncFiles`。 |
| **handleQueueSyncFiles 过滤** | ✅ **会过滤已存在的资产**。`filterNewExternalAssetPaths` 会排除数据库中已存在的路径，不管是否 offline。offline 资产路径不会重新排队 import。 |
| **handleQueueSyncAssets offline/online** | ✅ **会恢复 online（如果条件满足）**。<br>1. `checkExistingAsset()` 对已 offline 且文件存在的资产返回 `CHECK_OFFLINE`<br>2. 检查是否在 importPath 中：如果不在，保持 offline<br>3. 检查是否被 exclusionPatterns 命中：如果命中，保持 offline<br>4. 都通过则标记为 online：`isOffline: false, deletedAt: null` |

### 关键代码验证 - 过滤已存在的资产

```typescript
// [asset.repository.ts#L1052-L1071](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/asset.repository.ts#L1052-L1071)
async filterNewExternalAssetPaths(libraryId: string, paths: string[]): Promise<string[]> {
  const result = await this.db
    .selectFrom(unnest(paths).as('path'))
    .select('path')
    .where((eb) =>
      eb.not(
        eb.exists(
          this.db
            .selectFrom('asset')
            .select('originalPath')
            .whereRef('asset.originalPath', '=', eb.ref('path'))
            .where('libraryId', '=', libraryId)
            .where('isExternal', '=', true),
        ),
      ),
    )
    .execute();
  return result.map((row) => row.path);
}
```

> **注意**：这个查询不区分 `isOffline` 状态，只要路径存在于 asset 表中就会被过滤。

### 关键代码验证 - offline 转 online

```typescript
// [library.service.ts#L601-L604](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L601-L604)
if (asset.isOffline && asset.status !== AssetStatus.Deleted) {
  // 只有 offline 且未删除的资产才进行检查
  return AssetSyncResult.CHECK_OFFLINE;
}

// [library.service.ts#L553-L555](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L553-L555)
if (assetIdsToOnline.length > 0) {
  promises.push(this.assetRepository.updateAll(assetIdsToOnline, { isOffline: false, deletedAt: null }));
}
```

---

## 场景6：路径里出现新文件

### 行为分析

| 组件 | 行为 |
|------|------|
| **validateImportPath 返回** | `{ isValid: true }`。路径本身有效。 |
| **update 是否拒绝保存** | ✅ **允许保存**。 |
| **watch 是否排队** | ✅ **会排队**。watcher 检测到 `add` 事件，通过 matcher 检查（文件扩展名匹配 + 不被 exclusionPatterns 排除），然后排队 `LibrarySyncFiles`。 |
| **handleQueueSyncFiles 过滤** | ✅ **会处理新文件**。<br>1. `walk()` 遍历 importPaths，排除 hidden 文件和 exclusionPatterns<br>2. `filterNewExternalAssetPaths()` 过滤已存在的路径<br>3. 新路径排队 `LibrarySyncFiles` 进行 import<br>4. `processEntity()` 创建资产记录，`checksum` 使用 `sha1(path)` |
| **handleQueueSyncAssets offline/online** | ❌ **不相关**。新文件还没有资产记录，不会进入这个流程。 |

### 关键代码验证 - watcher 触发新文件

```typescript
// [library.service.ts#L143-L145](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L143-L145)
onAdd: (path) => {
  return handlePromiseError(handler('add', path), this.logger);
},
```

### 关键代码验证 - 新文件导入流程

```typescript
// [library.service.ts#L248-L281](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L248-L281)
@OnJob({ name: JobName.LibrarySyncFiles, queue: QueueName.Library })
async handleSyncFiles(job: JobOf<JobName.LibrarySyncFiles>): Promise<JobStatus> {
  const assetImports: Insertable<AssetTable>[] = [];
  await Promise.all(
    job.paths.map((path) =>
      this.processEntity(path, library.ownerId, job.libraryId)
        .then((asset) => assetImports.push(asset))
        .catch(...)
    ),
  );

  const assetIds = await this.assetRepository.createAll(assetImports);
  await this.queuePostSyncJobs(assetIds); // sidecar discovery, metadata extraction
  return JobStatus.Success;
}

// [library.service.ts#L400-L419](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/library.service.ts#L400-L419)
private async processEntity(filePath, ownerId, libraryId) {
  const assetPath = path.normalize(filePath);
  const stat = await this.storageRepository.stat(assetPath);
  return {
    ownerId,
    libraryId,
    checksum: this.cryptoRepository.hashSha1(`path:${assetPath}`),
    checksumAlgorithm: ChecksumAlgorithm.sha1Path,
    originalPath: assetPath,
    fileCreatedAt: stat.mtime,
    fileModifiedAt: stat.mtime,
    type: mimeTypes.isVideo(assetPath) ? AssetType.Video : AssetType.Image,
    originalFileName: parse(assetPath).base,
    isExternal: true,
  };
}
```

---

## 行为汇总表

| 场景 | validateImportPath | update 拒绝 | watch 排队 | handleQueueSyncFiles 过滤 | handleQueueSyncAssets offline/online |
|------|-------------------|------------|-----------|-------------------------|--------------------------------------|
| **Immich 内部路径** | `isValid: false` + 消息 | ✅ 是 | ❌ 否 | ❌ 跳过 | ❌ 不相关 |
| **相对路径** | `isValid: false` + 消息 | ✅ 是 | ❌ 否 | ❌ 跳过 | ❌ 不相关 |
| **路径不可读** | `isValid: false` + 消息 | ✅ 是 | ❌ 否（保存前）<br>✅ 跳过（保存后） | ✅ 跳过 | ✅ 文件不存在 → offline |
| **被 exclusionPatterns 命中** | `isValid: true` | ❌ 否 | ❌ 不触发 import（add/change）<br>✅ 删除仍会触发 | ✅ walk 时排除 | ✅ SQL 匹配 → offline<br>❌ 恢复时仍被排除 → 保持 offline |
| **已有资产离线** | `isValid: true` | ❌ 否 | ✅ 文件变化会触发 | ✅ 已存在路径被过滤 | ✅ CHECK_OFFLINE → 检查 importPath 和 exclusion → online |
| **出现新文件** | `isValid: true` | ❌ 否 | ✅ add 事件触发 import | ✅ walk + filterNew → 排队导入 | ❌ 不相关 |

---

## 关键流程图示

### 1. Import Path 验证与保存流程

```
管理员添加/更新 importPath
        ↓
validateImportPath()
        ├─ 检查 isImmichPath → ❌ 拒绝
        ├─ 检查 isAbsolute → ❌ 拒绝
        ├─ 检查 stat() 存在且是目录 → ❌ 拒绝
        └─ 检查 R_OK 读权限 → ❌ 拒绝
        ↓
update() 发现 isValid: false → 抛出 BadRequestException
        ↓ (验证通过)
保存到数据库
        ↓
watch() 启动文件监控（如果 watch 启用）
```

### 2. 新文件发现流程

```
磁盘出现新文件
        ↓
watcher onAdd 事件
        ↓
picomatch 匹配（扩展名 + exclusionPatterns）
        └─ 不匹配 → 记录 verbose 日志，忽略
        ↓
queue LibrarySyncFiles
        ↓
handleSyncFiles → processEntity → createAll → queuePostSyncJobs
```

### 3. 已有资产状态同步流程

```
LibrarySyncAssetsQueueAll 触发
        ↓
detectOfflineExternalAssets()  [SQL UPDATE]
        ├─ 不在 importPath 下 → mark offline
        └─ 匹配 exclusionPatterns → mark offline
        ↓
streamAssetIds() 分批处理
        ↓
handleSyncAssets
        ├─ stat() 失败 → OFFLINE
        ├─ stat() 成功且 isOffline → CHECK_OFFLINE
        │   ├─ 不在 importPath → 保持 offline
        │   ├─ 匹配 exclusion → 保持 offline
        │   └─ ✅ 都通过 → ONLINE
        ├─ mtime 变化 → UPDATE（重新提取元数据）
        └─ 其他 → DO_NOTHING
```
