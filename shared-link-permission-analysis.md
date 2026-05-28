# SharedLink 权限语义一致性分析

## 问题背景

前端 `SharedLinkFormFields` 中 `$effect` 会在 `showMetadata=false` 时把 `allowDownload=false`，后端 `SharedLinkService.create` 也有同样的强制逻辑。但 `SharedLinkService.update` 是否也强制？如果直接调用 API 发送 `{ showMetadata: false, allowDownload: true }`，系统行为如何？

## 代码分析

### 1. 前端约束（UI层）

**文件**：[SharedLinkFormFields.svelte](file:///c:/Users/10244/Desktop/0508-under/immich/web/src/lib/components/SharedLinkFormFields.svelte#L26-L30)

```typescript
$effect(() => {
  if (!showMetadata && allowDownload) {
    allowDownload = false;
  }
});
```

同时第56行UI层面禁用开关：
```svelte
<Field label={$t('allow_public_user_to_download')} disabled={!showMetadata}>
```

**结论**：前端保证 `showMetadata=false` 时 `allowDownload` 必为 `false`。

---

### 2. 后端 `create` 强制逻辑

**文件**：[shared-link.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/shared-link.service.ts#L89-L103)

第100行核心逻辑：
```typescript
allowDownload: dto.showMetadata === false ? false : (dto.allowDownload ?? true),
showExif: dto.showMetadata ?? true,
```

**结论**：`create` 接口强制约束：`showMetadata === false` → `allowDownload = false`。

---

### 3. 后端 `update` 逻辑（关键差异）

**文件**：[shared-link.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/shared-link.service.ts#L119-L137)

第128-130行：
```typescript
allowUpload: dto.allowUpload,
allowDownload: dto.allowDownload,
showExif: dto.showMetadata,
```

**结论**：`update` 接口**没有**任何强制逻辑！直接透传 `dto.allowDownload` 和 `dto.showMetadata` 的值。

---

### 4. Repository 层

**文件**：[shared-link.repository.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/shared-link.repository.ts#L193-L209)

`update` 方法直接将传入的实体写入数据库，没有额外约束。

---

### 5. AuthSharedLink 结构

**文件**：[database.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/database.ts#L164-L173)

```typescript
export type AuthSharedLink = {
  id: string;
  expiresAt: Date | null;
  userId: string;
  albumId: string | null;
  showExif: boolean;        // 对应 showMetadata
  allowUpload: boolean;
  allowDownload: boolean;   // 独立字段
  password: string | null;
};
```

**注意**：`showExif` 和 `allowDownload` 是两个独立的数据库字段。

---

### 6. 响应 DTO 映射

**文件**：[shared-link.dto.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/dtos/shared-link.dto.ts#L86-L112)

```typescript
export function mapSharedLink(sharedLink: SharedLink, options: { stripAssetMetadata: boolean }): SharedLinkResponseDto {
  return {
    // ...
    assets: assets.map((asset) => mapAsset(asset, { stripMetadata: options.stripAssetMetadata })),
    allowDownload: sharedLink.allowDownload,
    showMetadata: sharedLink.showExif,
    // ...
  };
}
```

**关键点**：
- `stripAssetMetadata` 由 `!sharedLink.showExif` 控制（见 [shared-link.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/shared-link.service.ts#L60)）
- `allowDownload` 和 `showMetadata` 独立返回

---

### 7. 权限检查逻辑

**文件**：[access.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/utils/access.ts#L58-L98)

#### AssetDownload 权限（第74-76行）
```typescript
case Permission.AssetDownload: {
  return sharedLink.allowDownload ? await access.asset.checkSharedLinkAccess(sharedLinkId, ids) : new Set();
}
```
- **仅检查 `allowDownload`**，不检查 `showExif`

#### AssetView 权限（第70-72行）
```typescript
case Permission.AssetView: {
  return await access.asset.checkSharedLinkAccess(sharedLinkId, ids);
}
```
- 不检查 `allowDownload`，也不检查 `showExif`

#### AlbumDownload 权限（第86-88行）
```typescript
case Permission.AlbumDownload: {
  return sharedLink.allowDownload ? await access.album.checkSharedLinkAccess(sharedLinkId, ids) : new Set();
}
```
- **仅检查 `allowDownload`**，不检查 `showExif`

---

### 8. 元数据剥离逻辑（mapAsset）

**文件**：[asset-response.dto.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/dtos/asset-response.dto.ts#L193-L245)

```typescript
if (stripMetadata) {
  const sanitizedAssetResponse: SanitizedAssetResponseDto = {
    id: entity.id,
    type: entity.type,
    originalMimeType: mimeTypes.lookup(entity.originalFileName),
    thumbhash: entity.thumbhash ? hexOrBufferToBase64(entity.thumbhash) : null,
    localDateTime: asDateString(entity.localDateTime),
    duration: entity.duration,
    livePhotoVideoId: entity.livePhotoVideoId,
    hasMetadata: false,
    width: entity.width,
    height: entity.height,
  };
  return sanitizedAssetResponse as AssetResponseDto;
}
```

**被剥离的字段**：`originalFileName`, `originalPath`, `owner`, `exifInfo`, `tags`, `people`, `checksum`, `isFavorite` 等。

---

### 9. downloadOriginal 逻辑

**文件**：[asset-media.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/asset-media.service.ts#L163-L183)

```typescript
async downloadOriginal(auth: AuthDto, id: string, dto: AssetDownloadOriginalDto): Promise<ImmichFileResponse> {
  await this.requireAccess({ auth, permission: Permission.AssetDownload, ids: [id] });
  // ...
  return new ImmichFileResponse({
    path,
    fileName: getFileNameWithoutExtension(originalFileName) + getFilenameExtension(path),  // 直接使用原始文件名！
    contentType: mimeTypes.lookup(path),
    cacheControl: CacheControl.PrivateWithCache,
  });
}
```

**关键点**：
- 权限检查依赖 `sharedLink.allowDownload`
- **文件名无隐藏逻辑**，直接使用 `originalFileName`
- 下载的是原始文件，包含完整 EXIF 元数据

---

### 10. viewThumbnail 逻辑

**文件**：[asset-media.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/asset-media.service.ts#L185-L232)

```typescript
async viewThumbnail(auth: AuthDto, id: string, dto: AssetMediaOptionsDto) {
  await this.requireAccess({ auth, permission: Permission.AssetView, ids: [id] });
  // ...
  const fileNameBase =
    auth.sharedLink && !auth.sharedLink.showExif ? id : getFileNameWithoutExtension(originalFileName);
  const fileName = `${fileNameBase}_${size}${getFilenameExtension(path)}`;
  // ...
}
```

**关键点**：
- 权限检查不依赖 `allowDownload`
- **有文件名隐藏逻辑**：`!showExif` 时用 asset id 代替文件名

---

### 11. downloadArchive 逻辑

**文件**：[download.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/download.service.ts#L83-L121)

```typescript
async downloadArchive(auth: AuthDto, dto: DownloadArchiveDto): Promise<ImmichReadStream> {
  await this.requireAccess({ auth, permission: Permission.AssetDownload, ids: dto.assetIds });
  // ...
  for (const assetId of dto.assetIds) {
    // ...
    let filename = sanitize(originalFileName) || 'unnamed';  // 直接使用原始文件名！
    // ...
    zip.addFile(realpath, filename);
  }
  // ...
}
```

**关键点**：
- 权限检查依赖 `sharedLink.allowDownload`
- **文件名无隐藏逻辑**，直接使用 `originalFileName`
- 下载的是原始文件，包含完整 EXIF 元数据

---

## 场景模拟：发送 `{ showMetadata: false, allowDownload: true }` 到 update 接口

### 数据库状态
| 字段 | 值 |
|------|-----|
| `showExif` | `false` |
| `allowDownload` | `true` |

### 各行为表现

| 行为 | 权限检查 | 元数据/文件名处理 | 结果 |
|------|----------|-------------------|------|
| **获取共享链接信息** | 无需权限 | `stripAssetMetadata = true` | ✅ API 响应中元数据被剥离，`showMetadata: false, allowDownload: true` |
| **获取资产列表** | `AssetRead` | `stripMetadata = true` | ✅ 资产列表中文件名、EXIF 等被隐藏 |
| **查看缩略图** | `AssetView`（不检查 allowDownload） | `!showExif` 时用 asset id 作文件名 | ✅ 文件名被隐藏 |
| **下载原图** | `AssetDownload`（检查 `allowDownload = true`） | **直接使用原始文件名** | ❌ **原始文件名暴露！文件内 EXIF 元数据也暴露！** |
| **下载归档** | `AssetDownload`（检查 `allowDownload = true`） | **直接使用原始文件名** | ❌ **原始文件名暴露！文件内 EXIF 元数据也暴露！** |

---

## 结论

### 语义不一致判定

**存在严重的语义不一致问题：**

1. **前端约束**：UI 保证 `showMetadata=false` 时 `allowDownload` 必为 `false`
2. **后端 `create`**：API 层面强制 `showMetadata=false` 时 `allowDownload=false`
3. **后端 `update`**：**没有强制逻辑**，允许 `{ showMetadata: false, allowDownload: true }` 这种不一致状态
4. **运行时行为分裂**：
   - API 响应层面：元数据被正确剥离（`stripMetadata`）
   - 缩略图层面：文件名被正确隐藏
   - **下载层面：文件名和文件内元数据完全暴露！**

### 这是"后端兜住"还是"依赖前端"？

**两者都不是，这是一个逻辑漏洞：**

- 不是"后端兜住"：因为 `update` 接口没有强制约束，而且下载时元数据会泄露
- 不是"依赖前端"：因为 `create` 接口有后端保护
- 正确描述：**后端保护不完整 + 语义不一致**

### 风险说明

`showMetadata=false` 的语义应该是"不向共享链接接收者暴露任何元数据"，包括：
- API 响应中的元数据字段
- 文件名（可能包含拍摄时间、地点等信息）
- 文件内的 EXIF 元数据（GPS、拍摄设备等）

但当前实现中，只有前两项在部分场景下被保护，下载时所有元数据都会泄露。

### 修复建议

1. **在 `SharedLinkService.update` 中添加与 `create` 相同的强制逻辑**：
   ```typescript
   allowDownload: dto.showMetadata === false ? false : dto.allowDownload,
   ```

2. **在 `downloadOriginal` 和 `downloadArchive` 中添加文件名隐藏逻辑**，与 `viewThumbnail` 保持一致。

3. **考虑在下载时剥离 EXIF 元数据**（如果 `showMetadata=false`）。

4. **添加数据库层面的 CHECK 约束**，防止 `showExif = false AND allowDownload = true` 的状态存在。
