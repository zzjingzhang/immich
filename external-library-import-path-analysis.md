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

资产有 `isOffline` 字段，