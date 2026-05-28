# SearchService.searchSmart + SmartInfoService.onConfigUpdate 场景分析

本文分析 `SearchService.searchSmart` 与 `SmartInfoService.onConfigUpdate`（以及相关任务队列）交互时，在以下四种典型场景下的代码路径与用户可见结果：

1. 模型名改变但维度相同
2. 模型名改变且维度不同
3. 强制重新编码（force re-encode）
4. `queryAssetId` 以图搜图但 asset embedding 已被删除

> 关键代码位置：
> - 搜索入口：[search.service.ts#L121-L162](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/search.service.ts#L121-L162)
> - 配置变更处理：[smart-info.service.ts#L18-L21](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/smart-info.service.ts#L18-L21)
> - 核心初始化逻辑 `init`：[smart-info.service.ts#L34-L65](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/smart-info.service.ts#L34-L65)
> - 强制重新入队 `handleQueueEncodeClip`：[smart-info.service.ts#L67-L93](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/smart-info.service.ts#L67-L93)
> - 单个 asset 的 CLIP 编码 `handleEncodeClip`：[smart-info.service.ts#L95-L127](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/smart-info.service.ts#L95-L127)

---

## 公共逻辑铺垫

### `init` 的关键分支判断

```ts
const modelChange =
  oldConfig && oldConfig.machineLearning.clip.modelName !== newConfig.machineLearning.clip.modelName;
const dimSizeChange = dbDimSize !== dimSize;
if (!modelChange && !dimSizeChange) return;

if (dimSizeChange) {
  // 更新向量列类型与索引
  await this.databaseRepository.setDimensionSize(dimSize);
} else {
  // 仅删除旧 embedding，不修改列类型
  await this.databaseRepository.deleteAllSearchEmbeddings();
}
```

也就是说：

- `dimSizeChange = true`（无论是模型换了导致维度变，还是“模型没换但 DB 维度异常”）→ **删除所有旧 embedding + 修改列类型 `vector(N)` + 重建索引**（`setDimensionSize` 内部会 `delete from smart_search` 并 alter 列）。
- `dimSizeChange = false` 但 `modelChange = true` → **只删除旧 embedding**，列类型保持不变。
- 二者都没变化 → 早退，不做任何事。
- 两条分支都只删旧 embedding，不会自动调度重新编码任务（源码中留了 `TODO`）。

### `handleQueueEncodeClip` 的 force 分支

```ts
if (force) {
  const { dimSize } = getCLIPModelInfo(machineLearning.clip.modelName);
  await this.databaseRepository.setDimensionSize(dimSize);   // 额外兜底：再次同步维度
}
// 之后把 streamForEncodeClip(force) 遍历到的 asset 全部入队
```

当 `force = true` 时：
- 会显式再次 `setDimensionSize(dimSize)`，即使此前 `onConfigUpdate` 已经同步过。
- `streamForEncodeClip(true)` 会把 **全部资产**（而非仅“缺少 embedding 的”）送入 `JobName.SmartSearch`。

### `handleEncodeClip` 的模型名再次校验

```ts
const embedding = await this.machineLearningRepository.encodeImage(..., machineLearning.clip);
// 等待 CLIPDimSize 锁释放 ...
const newConfig = await this.getConfig({ withCache: true });
if (machineLearning.clip.modelName !== newConfig.machineLearning.clip.modelName) {
  return JobStatus.Skipped;
}
await this.searchRepository.upsert(asset.id, embedding);
```

这是一个“编码完成后再看一次当前模型名”的校验：若编码期间模型被用户切换，则该次编码结果不写入 DB，避免错的 embedding 污染索引。

### `searchSmart` 的 LRU key

```ts
const key = machineLearning.clip.modelName + dto.query + dto.language;
embedding = this.embeddingCache.get(key);
if (!embedding) {
  embedding = await this.machineLearningRepository.encodeText(dto.query, { modelName, language });
  this.embeddingCache.set(key, embedding);
}
```

LRU key 由 `modelName + query + language` 拼接组成。**模型名一变，缓存一定失效**；但这只影响文本 query 分支。`queryAssetId` 分支完全不经过 LRU。

---

## 场景一：模型名改变但维度相同

### 触发条件

- `oldConfig.machineLearning.clip.modelName ≠ newConfig.machineLearning.clip.modelName`
- 但 `getCLIPModelInfo(newName).dimSize === dbDimSize`（例如 `ViT-B-16__openai` → `ViT-B-32__openai`，都是 512 维；或同一结构不同训练集）

### `onConfigUpdate` 路径

1. `onConfigUpdate` → `init(newConfig, oldConfig)`
2. `modelChange = true`，`dimSizeChange = false`
3. 进入 `else` 分支：`deleteAllSearchEmbeddings()`
4. 所有资产的 CLIP embedding 被清空；**不触发重新编码任务**

### `searchSmart` 路径（用户发起文本搜索）

1. 校验 `isSmartSearchEnabled` 通过
2. 构造 LRU key = `新模型名 + query + language`
   - 旧模型名对应的缓存条目依然在 LRU 里，但由于新的 key 不同，`get` 返回 `undefined`
   - 必然进入 `encodeText(...)` 用新模型生成文本 embedding
3. 新 embedding 进入 LRU
4. 调用 `searchRepository.searchSmart({ page, size }, { userIds, embedding, ... })`

### 用户可见结果

- **空结果**（`items.length === 0`），但 **不会抛异常**
- 原因：DB 里所有 asset embedding 都已被 `deleteAllSearchEmbeddings` 清空，向量相似度检索返回 0 行
- 前端看到一个正常的 200 响应，只是 `assets.items` 为空数组

> 典型触发：`ViT-B-16__openai` → `ViT-B-32__openai`

---

## 场景二：模型名改变且维度不同

### 触发条件

- `modelChange = true`
- 且 `getCLIPModelInfo(newName).dimSize !== dbDimSize`（例如 `ViT-B-16__openai` 512 维 → `ViT-L-14-quickgelu__dfn2b` 768 维）

### `onConfigUpdate` 路径

1. `onConfigUpdate` → `init(newConfig, oldConfig)`
2. `modelChange = true`，`dimSizeChange = true`
3. 进入 `if (dimSizeChange)` 分支：`this.databaseRepository.setDimensionSize(dimSize)`
   - 内部执行：
     - `delete from smart_search`（清空旧 embedding）
     - `alter table smart_search ... alter column embedding set data type vector(dimSize)`（改列类型）
     - 重建 `clip_index`
     - `vacuum analyze smart_search`
4. **不触发重新编码任务**

### `searchSmart` 路径（用户发起文本搜索）

与场景一完全一致：

1. 校验通过
2. 新模型名 → LRU key 不同 → 重新 `encodeText` 用新模型生成文本 embedding
3. `searchRepository.searchSmart(...)`

### 用户可见结果

- **空结果**（`items.length === 0`），**不会抛异常**
- 场景与一相同：asset embedding 已被 `setDimensionSize` 内部的 `delete from smart_search` 清空

> 典型触发：`ViT-B-16__openai` → `ViT-L-14-quickgelu__dfn2b`

---

## 场景三：强制重新编码（force re-encode）

### 触发条件

用户在管理界面或 API 中触发“重新编码所有 CLIP 资产”，对应 `JobName.SmartSearchQueueAll` 任务。force 可以独立于模型切换发生（例如用户觉得 embedding 质量差，想对当前模型重新跑一遍）。

### 路径

1. `handleQueueEncodeClip({ force: true })` 被调度
   - `force === true` → `setDimensionSize(dimSize)` 再跑一次（幂等兜底，确保维度一致）
   - `streamForEncodeClip(true)` 返回 **所有 asset**
   - 全部入队 `JobName.SmartSearch`
2. 每个 `JobName.SmartSearch` 任务进入 `handleEncodeClip({ id })`：
   - `encodeImage(...)` 用当前模型得到向量
   - 等待 `CLIPDimSize` 锁
   - **再次读取配置**，若此时模型名又变了 → `Skipped`，该 asset 不会写入
   - 否则 `searchRepository.upsert(asset.id, embedding)` 写回
3. 全部任务完成后，DB 中每个 asset 都有一条与当前模型匹配的 embedding。

### `searchSmart` 路径

与场景一、二相同，唯一不同的是 **此时 DB 中 asset embedding 已经被重新填充**。

### 用户可见结果

- **正常返回结果**（搜索成功且有命中的 asset）
- 若任务尚未全部跑完，搜索会返回部分结果（数量取决于已经 upsert 完成的 asset 数量）

> 备注：如果在入队后、`handleEncodeClip` 执行期间用户再次切换模型名，`handleEncodeClip` 中的二次校验会把这批任务全部 `Skipped`，结果回到“空结果”状态。

---

## 场景四：`queryAssetId` 以图搜图，但 asset embedding 已被删除

### 触发条件

- 用户使用 `queryAssetId`（而不是 `query`）发起搜索
- 被引用的 asset 的 embedding 在之前的某个场景（1/2 或其他清理）中已被删除

### `searchSmart` 路径

```ts
} else if (dto.queryAssetId) {
  await this.requireAccess({ auth, permission: Permission.AssetRead, ids: [dto.queryAssetId] });
  const getEmbeddingResponse = await this.searchRepository.getEmbedding(dto.queryAssetId);
  const assetEmbedding = getEmbeddingResponse?.embedding;
  if (!assetEmbedding) {
    throw new BadRequestException(`Asset ${dto.queryAssetId} has no embedding`);
  }
  embedding = assetEmbedding;
}
```

1. 先做权限检查 `requireAccess`，失败直接抛异常（不在本文讨论范围内）
2. 从 DB 取出该 asset 的 embedding
3. 如果 `getEmbeddingResponse` 不存在或者 `embedding` 为 `null`/`undefined`
   → 直接 **抛出 `BadRequestException`**
4. 只有取出了非空 embedding 才会进入 `searchRepository.searchSmart(...)`

这个分支 **不经过 LRU**，也不会触发 `encodeText`。

### 用户可见结果

- **HTTP 400 BadRequestException**：`Asset <id> has no embedding`
- 不会返回空结果，也不会返回有结果的响应，而是直接异常

> 典型触发：场景 1 或 2 把所有 embedding 清空后，用户立刻拿某张 asset 做“以图搜图”。

---

## 汇总表

| # | 场景 | onConfigUpdate/队列动作 | searchSmart 路径 | 用户可见结果 |
|---|---|---|---|---|
| 1 | 模型名改变，维度相同 | `deleteAllSearchEmbeddings`；不重编码 | 新 LRU key → `encodeText`（新模型）→ DB 检索 | 空结果（200，`items = []`） |
| 2 | 模型名改变，维度不同 | `setDimensionSize`（删旧 + 改列 + 重建索引）；不重编码 | 新 LRU key → `encodeText`（新模型）→ DB 检索 | 空结果（200，`items = []`） |
| 3 | 强制重新编码（force） | `setDimensionSize`（兜底） + 所有 asset 入队 `SmartSearch` 重新编码 | 与上同，但 DB 已有新 embedding | 正常结果（取决于任务完成进度） |
| 4 | `queryAssetId` 但 asset embedding 已删 | （与 1/2 无关）直接查 `getEmbedding` 为空 | 分支内 `if (!assetEmbedding)` 命中 | `BadRequestException`（400） |

## 总结观察

1. **模型名参与 LRU key** 是一个非常关键的设计：模型换了必然触发 `encodeText`，避免把旧模型的文本 embedding 去跟新模型的 asset embedding 做相似度检索。
2. **onConfigUpdate 只“删”不“建”**：删除旧 embedding 是强制的（要么 `deleteAllSearchEmbeddings`，要么 `setDimensionSize` 里隐式 delete），但不会自动调度重新编码，所以切换模型后的第一体验几乎总是“空结果”，必须等用户手动触发 force 或等后台的增量任务。
3. **force 路径是唯一的重建入口**：`handleQueueEncodeClip(force=true)` 是把资产 embedding 填回来的唯一显式路径。
4. **以图搜图（`queryAssetId`）对缺失 embedding 极度不宽容**：直接抛 400，不做回退（例如临时为该 asset 即时编码），所以场景 1/2 之后所有以图搜图请求都会失败，直到 force 完成。
5. **handleEncodeClip 的“编码后再看一次模型名”** 是一个防御式检查，避免编码过程中模型再次切换导致错向量写入；代价是刚入队又被管理员切模型时整批任务会被丢弃，用户体验就是“又要再触发一次 force”。
