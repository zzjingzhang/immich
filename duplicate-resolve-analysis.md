# DuplicateService.resolveGroup 场景分析文档

## 场景描述

构造一个复杂的去重场景，同一重复组内包含以下特征：

| 资产ID | Visibility | Description | GPS坐标 | 标签 | 所属相册 | 文件大小 | 用户相册权限 |
|--------|------------|-------------|---------|------|----------|----------|--------------|
| A1 | Timeline | "Line1\nLine2" | (40.7, -74.0) | Tag1, Tag2 | Album1, Album2 | 5MB | Album1: 有, Album2: 无 |
| A2 | Archive | "Line2\nLine3" | (40.7, -74.0) | Tag2, Tag3 | Album2, Album3 | 8MB | Album2: 有, Album3: 无 |
| A3 | Locked | "Line1" | 无 | Tag3, Tag4 | Album3, Album4 | 3MB | Album3: 有, Album4: 有 |
| A4 | Hidden | "Line3\nLine4" | (51.5, -0.1) | Tag4, Tag5 | Album1 | 6MB | Album1: 有 |

## 一、前端 handleDeduplicateAll 执行流程

### 1.1 suggestedKeepAssetIds 选择逻辑

根据 [suggestDuplicateKeepAssetIds](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/utils/duplicate.ts#L57-L60) 函数：

- **选择标准**：优先选择文件大小最大的资产
- **结果**：A2（8MB）被选为保留资产

### 1.2 生成 keepAssetIds 和 trashAssetIds

根据 [handleDeduplicateAll](file:///c:/Users/10244/Desktop/0508-under/immich/web/src/routes/(user)/utilities/duplicates/[[photos=photos]]/[[assetId=id]]/+page.svelte#L134-L182) 逻辑：

```typescript
keepAssetIds = ['A2']
trashAssetIds = ['A1', 'A3', 'A4']
```

## 二、DuplicateService.resolveGroup 详细分析

### 2.1 验证阶段（第121-145行）

**验证通过条件**：
- 所有资产ID都在组内 ✓
- 没有资产同时在keep和trash列表中 ✓
- 所有资产都在其中一个列表中 ✓

### 2.2 删除权限检查（第147-157行）

检查用户对 `trashAssetIds = ['A1', 'A3', 'A4']` 的 `AssetDelete` 权限。
- **假设**：用户有删除权限 → 继续执行

---

## 三、getSyncMergeResult 字段合并分析

### 3.1 assetUpdate 字段分析

#### 3.1.1 isFavorite（第247行）
```typescript
response.assetUpdate.isFavorite = assets.some((asset) => asset.isFavorite);
```
- **逻辑**：只要有一个资产是收藏的，保留资产就设为收藏
- **场景假设**：所有资产都未收藏
- **结果**：`isFavorite = false`（不写入，因为默认是false）

#### 3.1.2 visibility（第249-256行）
```typescript
const visibilityOrder = [AssetVisibility.Locked, AssetVisibility.Archive, AssetVisibility.Timeline];
let visibility = visibilityOrder.find((level) => assets.some((asset) => asset.visibility === level));
if (!visibility && assets.some((asset) => asset.visibility === AssetVisibility.Hidden)) {
  visibility = AssetVisibility.Hidden;
}
```

**优先级顺序**：Locked > Archive > Timeline > Hidden（Hidden仅当前三者都不存在时才生效）

**场景分析**：
- A3: Locked ✓（优先级最高）
- A2: Archive（被覆盖）
- A1: Timeline（被覆盖）
- A4: Hidden（被覆盖）

**结果**：`visibility = AssetVisibility.Locked`

---

### 3.2 exifUpdate 字段分析

#### 3.2.1 rating（第258-267行）
```typescript
let rating = 0;
for (const asset of assets) {
  const assetRating = asset.exifInfo?.rating ?? 0;
  if (assetRating > rating) {
    rating = assetRating;
  }
}
if (rating > 0) {
  response.exifUpdate.rating = rating;
}
```
- **场景假设**：所有资产rating都为0
- **结果**：不写入rating字段

#### 3.2.2 description（第269-273行）
```typescript
const descriptionLines = uniqueNonEmptyLines(assets.map((asset) => asset.exifInfo?.description));
const description = descriptionLines.length > 0 ? descriptionLines.join('\n') : null;
```

**[uniqueNonEmptyLines](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/duplicate.service.ts#L35-L52) 逻辑**：
1. 遍历所有description值
2. 按换行符分割成行
3. 去除空白行
4. 按出现顺序去重

**场景合并**：
- A1: "Line1\nLine2" → ['Line1', 'Line2']
- A2: "Line2\nLine3" → ['Line2', 'Line3']
- A3: "Line1" → ['Line1']
- A4: "Line3\nLine4" → ['Line3', 'Line4']

**去重后**（按首次出现顺序）：`['Line1', 'Line2', 'Line3', 'Line4']`

**结果**：`description = "Line1\nLine2\nLine3\nLine4"`

#### 3.2.3 latitude & longitude（第275-280行）
```typescript
const latitude = getUniqueCoordinate(assets, 'latitude');
const longitude = getUniqueCoordinate(assets, 'longitude');
if (latitude !== null && longitude !== null) {
  response.exifUpdate.latitude = latitude;
  response.exifUpdate.longitude = longitude;
}
```

**[getUniqueCoordinate](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/duplicate.service.ts#L54-L65) 逻辑**：
1. 收集所有非空坐标值
2. 如果所有值都相同，返回该值；否则返回null

**场景分析**：
- A1: (40.7, -74.0)
- A2: (40.7, -74.0)
- A3: 无（跳过）
- A4: (51.5, -0.1)

**坐标值集合**：
- latitude: {40.7, 51.5} → 不唯一 → null
- longitude: {-74.0, -0.1} → 不唯一 → null

**结果**：不写入latitude和longitude字段

> **关键结论**：只有当**所有有GPS的资产**坐标都相同时，才会写入GPS坐标。部分资产坐标相同不会被保留。

---

### 3.3 mergedAlbumIds 分析（第282-288行）
```typescript
const albumIdSet = new Set<string>();
for (const [, albumIds] of assetAlbumMap) {
  for (const albumId of albumIds) {
    albumIdSet.add(albumId);
  }
}
response.mergedAlbumIds = [...albumIdSet];
```

**场景合并**：
- A1: Album1, Album2
- A2: Album2, Album3
- A3: Album3, Album4
- A4: Album1

**结果**：`mergedAlbumIds = [Album1, Album2, Album3, Album4]`

---

### 3.4 mergedTagIds & mergedTagValues 分析（第290-296行）
```typescript
const allTags = assets.flatMap((asset) => asset.tags ?? []);
const tagIds = [...new Set(allTags.map((tag) => tag.id).filter((id): id is string => !!id))];
const tagValues = [...new Set(allTags.map((tag) => tag.value).filter((v): v is string => !!v))];
```

**场景合并**：
- A1: Tag1, Tag2
- A2: Tag2, Tag3
- A3: Tag3, Tag4
- A4: Tag4, Tag5

**结果**：
- `mergedTagIds = [Tag1.id, Tag2.id, Tag3.id, Tag4.id, Tag5.id]`
- `mergedTagValues = ['Tag1', 'Tag2', 'Tag3', 'Tag4', 'Tag5']`

---

## 四、相册权限过滤与处理（第166-184行）

### 4.1 权限检查
```typescript
const allowedAlbumIds = await this.checkAccess({
  auth,
  permission: Permission.AlbumAssetCreate,
  ids: mergedAlbumIds,
});
```

**用户相册权限**：
- Album1: 有 ✓
- Album2: 有 ✓
- Album3: 有 ✓
- Album4: 有 ✓

**结果**：`allowedAlbumIds = [Album1, Album2, Album3, Album4]`

### 4.2 资产分享权限检查
```typescript
const allowedShareIds = await this.checkAccess({
  auth,
  permission: Permission.AssetShare,
  ids: idsToKeep,
});
```

**结果**：`allowedShareIds = ['A2']`（假设用户有分享权限）

### 4.3 添加资产到相册
```typescript
await this.albumRepository.addAssetIdsToAlbums(
  [...allowedAlbumIds].flatMap((albumId) => [...allowedShareIds].map((assetId) => ({ albumId, assetId }))),
);
```

**执行操作**：
- 将A2添加到Album1
- 将A2添加到Album2
- 将A2添加到Album3
- 将A2添加到Album4

> **结论**：用户有权限的相册都会添加保留资产

---

## 五、标签权限过滤与处理（第186-201行）

### 5.1 权限检查
```typescript
const allowedTagIds = await this.checkAccess({
  auth,
  permission: Permission.TagAsset,
  ids: mergedTagIds,
});
```

**假设**：用户对所有标签都有TagAsset权限

**结果**：`allowedTagIds = [Tag1.id, Tag2.id, Tag3.id, Tag4.id, Tag5.id]`

### 5.2 替换资产标签
```typescript
await Promise.all(idsToKeep.map((assetId) => this.tagRepository.replaceAssetTags(assetId, [...allowedTagIds])));
```

**执行操作**：
- 将A2的标签替换为Tag1, Tag2, Tag3, Tag4, Tag5

### 5.3 更新exif.tags
```typescript
await this.assetRepository.updateAllExif(idsToKeep, { tags: mergedTagValues });
```

**执行操作**：
- 更新A2的exif.tags为`['Tag1', 'Tag2', 'Tag3', 'Tag4', 'Tag5']`

---

## 六、保留资产字段更新与SidecarWrite（第203-216行）

### 6.1 EXIF更新（第207-209行）
```typescript
if (hasExifUpdate) {
  await this.assetRepository.updateAllExif(idsToKeep, exifUpdate);
}
```

**exifUpdate内容**：
```typescript
{
  description: "Line1\nLine2\nLine3\nLine4"
}
```

### 6.2 SidecarWrite触发条件（第211-213行）
```typescript
if (hasExifUpdate || hasTagUpdate) {
  await this.jobRepository.queueAll(idsToKeep.map((id) => ({ name: JobName.SidecarWrite, data: { id } })));
}
```

**触发条件分析**：
- `hasExifUpdate = true`（description有更新）
- `hasTagUpdate = true`（标签有更新）

**结果**：触发SidecarWrite任务 → **资产A2会生成sidecar文件**

### 6.3 资产表更新（第215行）
```typescript
await this.assetRepository.updateAll(idsToKeep, { duplicateId: null, ...assetUpdate });
```

**更新内容**：
```typescript
{
  duplicateId: null,
  visibility: AssetVisibility.Locked
}
```

---

## 七、删除/回收站处理（第218-233行）

```typescript
const force = !trash.enabled;
await this.assetRepository.updateAll(idsToTrash, {
  deletedAt: new Date(),
  status: force ? AssetStatus.Deleted : AssetStatus.Trashed,
  duplicateId: null,
});

await this.eventRepository.emit(force ? 'AssetDeleteAll' : 'AssetTrashAll', {
  assetIds: idsToTrash,
  userId: auth.user.id,
});
```

### 7.1 两种场景

**场景A：回收站功能启用（trash.enabled = true）**
- `force = false`
- 资产状态设为 `AssetStatus.Trashed`
- 触发事件：`AssetTrashAll`
- 影响资产：A1, A3, A4

**场景B：回收站功能禁用（trash.enabled = false）**
- `force = true`
- 资产状态设为 `AssetStatus.Deleted`
- 触发事件：`AssetDeleteAll`
- 影响资产：A1, A3, A4

---

## 八、最终结果汇总

### 8.1 保留资产（A2）写入的字段

| 表/位置 | 字段 | 值 |
|---------|------|----|
| assets表 | duplicateId | null |
| assets表 | visibility | Locked |
| asset_exif表 | description | "Line1\nLine2\nLine3\nLine4" |
| asset_exif表 | tags | ["Tag1", "Tag2", "Tag3", "Tag4", "Tag5"] |
| asset_tags表 | tagIds | Tag1, Tag2, Tag3, Tag4, Tag5（替换原标签） |
| album_assets表 | albumIds | Album1, Album2, Album3, Album4 |

### 8.2 触发SidecarWrite的资产

- **资产A2**（因为有exifUpdate和tagUpdate）

### 8.3 被跳过的相册/标签

- **相册**：无（假设用户对所有相册都有权限）
- **标签**：无（假设用户对所有标签都有权限）

> **说明**：如果用户对某些相册或标签没有权限，那些相册/标签会被跳过，保留资产不会被添加到那些相册或获得那些标签。

### 8.4 触发AssetTrashAll或AssetDeleteAll的资产

- **资产A1, A3, A4**
- 具体触发哪个事件取决于trash.enabled配置

---

## 九、关键设计决策总结

1. **Visibility优先级**：Locked > Archive > Timeline > Hidden，Hidden作为最低优先级
2. **GPS坐标**：只有所有有GPS的资产坐标都相同时才保留
3. **Description**：按行去重合并，保留首次出现顺序
4. **SidecarWrite**：只要有exif或tag更新就触发
5. **相册/标签权限**：只处理用户有权限的相册和标签
6. **回收站逻辑**：根据配置决定是软删除（Trash）还是硬删除（Delete）
