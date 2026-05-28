# PersonService.handleDetectFaces 与 handleRecognizeFaces 流程分析

> 涉及源码文件:
> - [person.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/person.service.ts#L303-L543)
> - [queue.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/queue.service.ts#L231)
> - [job.repository.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/job.repository.ts#L232)

---

## 1. handleDetectFaces 的数据准备

核心逻辑位于 [handleDetectFaces](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/person.service.ts#L303-L384)。

```typescript
const facesToAdd: (Insertable<AssetFaceTable> & { id: string })[] = [];
const embeddings: FaceSearchTable[] = [];
const mlFaceIds = new Set<string>();

for (const face of asset.faces) {
  if (face.sourceType === SourceType.MachineLearning) {
    mlFaceIds.add(face.id);
  }
}
```

- `mlFaceIds` 初始化为当前资产上**所有** `SourceType.MachineLearning` 的人脸 ID。
- 随后对 ML 服务本次返回的每个检测框（`boundingBox`, `embedding`）进行 IoU 匹配。

关键判定段:

```typescript
const match = asset.faces.find((face) => this.iou(face, scaledBox) > 0.5);

if (match && !mlFaceIds.delete(match.id)) {
  embeddings.push({ faceId: match.id, embedding });
} else if (!match) {
  const faceId = this.cryptoRepository.randomUUID();
  facesToAdd.push({ id: faceId, assetId: asset.id, ... });
  embeddings.push({ faceId, embedding });
}
```

`Set.delete()` 成功删除返回 `true`，失败返回 `false`。因此:
- `mlFaceIds.delete(match.id) === true` → match 是本轮 ML 人脸（本来就在 `mlFaceIds` 中）。
- `mlFaceIds.delete(match.id) === false` → match **不是** ML 人脸（即手工创建，`sourceType === SourceType.User`），因为手工人脸从未加入 `mlFaceIds`。

循环结束后，`mlFaceIds` 中**剩余**的 ID 即为「本轮未被任何新检测框命中的旧 ML 人脸」，将作为 `faceIdsToRemove` 删除。

---

## 2. 四种情况逐条分析

### 情况 A：新检测框匹配到「已有 ML face」

- 判定：`match` 存在 && `mlFaceIds.delete(match.id)` 返回 `true`。
- 行为：**不**进入 `embeddings` 分支，也不新建 face。
- 语义：旧 ML 人脸框与新检测框重叠率 > 0.5，视为「同一人脸再次出现」，**既不写 embedding，也不删除、也不新建**。该 ML face 的旧 embedding 保持不变。
- 入队：不会在本次 `handleDetectFaces` 中产生任何 FacialRecognition job，也不会加入 `facesToAdd`。该 face 只有在后续 `FacialRecognitionQueueAll` 被调度且符合条件（`personId === null`）时才会被批量排队。

### 情况 B：新检测框匹配到「手工创建的 face」

- 判定：`match` 存在 && `mlFaceIds.delete(match.id)` 返回 `false`（因为手工 face 不在 `mlFaceIds` 里）。
- 行为：进入 `embeddings.push({ faceId: match.id, embedding })`。**只更新 embedding，不新建 face**。
- 语义：用户手工框定的脸被 ML 检测命中，用本次提取的 embedding 覆盖/写入，供后续向量检索。
- 入队：同样不会触发 FacialRecognition job。且在 `handleRecognizeFaces` 内有显式跳过：

```typescript
if (face.sourceType !== SourceType.MachineLearning) {
  this.logger.warn(`Skipping face ${id} due to source ${face.sourceType}`);
  return JobStatus.Skipped;
}
```

即该手工 face 永远不会进入识别阶段。

### 情况 C：新检测框「没有匹配到」任何已有 face

- 判定：`match` 为 `undefined`。
- 行为：生成新 `faceId`，加入 `facesToAdd`（新建一条 `AssetFace`），同时把新 embedding 写入 `embeddings`。
- 语义：出现了一张全新检测到的脸。
- 入队：本 face 会在 `handleDetectFaces` 末尾被立即排队：

```typescript
if (facesToAdd.length > 0) {
  const jobs = facesToAdd.map(
    (face) => ({ name: JobName.FacialRecognition, data: { id: face.id } }) as const
  );
  await this.jobRepository.queueAll(
    [{ name: JobName.FacialRecognitionQueueAll, data: { force: false } }, ...jobs]
  );
}
```

注意：这里直接入队的 `FacialRecognition` job **没有 `deferred` 字段**，后续 `handleRecognizeFaces` 解构时 `deferred` 为 `undefined`，等价于 `false`。

### 情况 D：原有 ML face「本轮未匹配」

- 判定：`match` 不存在，且此 face 在 `mlFaceIds` 中残留到循环结束。
- 行为：`faceIdsToRemove = [...mlFaceIds]`，在 `personRepository.refreshFaces` 中被删除（连同其 embedding/关联 person 关系）。
- 语义：旧 ML 人脸在本次检测中已消失（被遮挡或置信度下降），被清理。
- 入队：不会，直接删除。

---

## 3. 四种情况汇总表

| 情况 | match 来源 | `mlFaceIds.delete` 结果 | 写入 embedding | 新建 face | 结果 |
|------|-----------|------------------------|----------------|-----------|------|
| A | 已有 ML face | `true`（删除成功） | ✗ | ✗ | 复用旧 ML face，保持原样 |
| B | 手工 face | `false`（不在集合中） | ✓ | ✗ | 只更新手工 face 的 embedding |
| C | 无 | N/A | ✓ | ✓ | 生成新 ML face |
| D | 未被本轮命中 | 残留到循环结束 | ✗ | ✗ | 旧 ML face 被删除 |

---

## 4. 如何进入 FacialRecognitionQueueAll / handleRecognizeFaces

### 4.1 FacialRecognitionQueueAll 的触发来源

1. **handleDetectFaces** 末尾：当有新 face（`facesToAdd.length > 0`）时，会把 `FacialRecognitionQueueAll(force: false)` 与新 face 的 `FacialRecognition` job 一并入队（见 [person.service.ts#L376](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/person.service.ts#L376)）。
2. **QueueService.queueAllFacialRecognition**（用户主动触发，见 [queue.service.ts#L231](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/queue.service.ts#L231)）。
3. **夜间任务**（[queue.service.ts#L289](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/queue.service.ts#L289)）。

同时该任务在 `job.repository.ts` 中被声明为 `deduplication: { id: JobName.FacialRecognitionQueueAll }`，意味着同队列中已存在时将被**去重合并**。

### 4.2 handleQueueRecognizeFaces 的过滤与排队

见 [handleQueueRecognizeFaces](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/person.service.ts#L404-L459)。

关键过滤条件：

```typescript
const facePagination = this.personRepository.getAllFaces(
  force ? undefined : { personId: null, sourceType: SourceType.MachineLearning },
);
```

- `force === false` 时：只遍历「无 person 关联」且「来源为 ML」的人脸。手工 face、以及已经归属到 person 的 ML face 都被直接排除。
- `force === true` 时：先执行 `unassignFaces({ sourceType: SourceType.MachineLearning })` 清除所有 ML face 的 `personId`，再对所有 ML face 重新排队。

入队时固定为 `deferred: false`：

```typescript
jobs.push({ name: JobName.FacialRecognition, data: { id: face.id, deferred: false } });
```

### 4.3 四种情况如何进入识别阶段

| 情况 | 源 | 何时进入 FacialRecognition job |
|------|----|--------------------------------|
| A (匹配已有 ML) | ML | 仅当 `personId === null`，在 `FacialRecognitionQueueAll` 中被重新捞起 |
| B (匹配手工) | User | **永远不会**（`sourceType !== ML` 直接跳过） |
| C (新 ML) | ML | 两种路径：① handleDetectFaces 末尾直接入队；② 若它还未完成 person 归属，也会被 `FacialRecognitionQueueAll` 再次捞起 |
| D (未命中旧 ML) | — | 已删除，不会进入 |

---

## 5. handleRecognizeFaces 中「deferred」的含义

见 [handleRecognizeFaces](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/person.service.ts#L462-L543)。

### 5.1 定义：什么是「core」face

```typescript
const isCore =
  matches.length >= machineLearning.facialRecognition.minFaces &&
  face.asset.visibility === AssetVisibility.Timeline;
```

即：
- 向量检索返回的「相似人脸数量」≥ `minFaces`（意味着该人脸在库里有足够多的同伴，能被聚类）；**且**
- 该人脸所在资产在时间线上可见（非归档、非隐藏）。

### 5.2 deferred 的触发条件

```typescript
if (!isCore && !deferred) {
  this.logger.debug(`Deferring non-core face ${id} for later processing`);
  await this.jobRepository.queue({ name: JobName.FacialRecognition, data: { id, deferred: true } });
  return JobStatus.Skipped;
}
```

**deferred 的语义**：`deferred` 作为一个「已经被推迟过一次」的标记。

- 初次进入：`deferred` 为 `false`/`undefined`。若判定为 non-core，**重新入队**一个同 ID 的 `FacialRecognition` job，但这次带 `deferred: true`，然后 `Skipped` 返回。
- 再次进入（`deferred === true`）：无论是否 core，都**不再推迟**，继续执行后续识别逻辑。

### 5.3 为什么要 defer

- 新检测到的人脸往往在向量库中**还没有足够的邻居**（`matches.length < minFaces`），过早聚类会产生很多「单人 person」或误分类。
- defer 相当于把「non-core」的识别延后一轮，等更多其他资产的人脸被写入向量库后，再回来重新聚类。
- 第二轮（`deferred: true`）时即使仍不满足 core，也会**强制继续执行**：
  - 若已有 `personId` 的匹配：直接分配。
  - 若是 core 但没匹配到 person：**新建 person**。
  - 若非 core 且没匹配到 person：保留「无 person 状态」，等待下次 `FacialRecognitionQueueAll` 再捞起（由于 `personId === null` 依然成立）。

### 5.4 其它提前跳过的情况

即使不涉及 deferred，`handleRecognizeFaces` 也会在以下情况返回 `Skipped`/`Failed`：
- `face.sourceType !== SourceType.MachineLearning` → Skipped
- `!face.faceSearch?.embedding` → Failed
- `face.personId` 已存在 → Skipped
- `matches.length <= 1` 且 `minFaces > 1` → Skipped（只命中了自己，没有邻居）

---

## 6. 完整时序图（以情况 C 新 ML face 为例）

```
handleDetectFaces(asset)
  ├─ 初始化 mlFaceIds = 所有已有 ML face
  ├─ 对每个新检测框:
  │   └─ IoU > 0.5 ?
  │        ├─ 命中 ML (情况A)     → 不写 emb, 不新建
  │        ├─ 命中 User (情况B)   → 只写 emb
  │        └─ 无命中 (情况C)      → 新建 + 写 emb
  ├─ mlFaceIds 残留 → faceIdsToRemove (情况D, 删除)
  ├─ refreshFaces(facesToAdd, faceIdsToRemove, embeddings)
  └─ 若 facesToAdd 非空:
       queueAll([FacialRecognitionQueueAll{force:false},
                 ...FacialRecognition{id, deferred:false}])

FacialRecognitionQueueAll 被消费:
  └─ 捞取 { personId: null, sourceType: MachineLearning } 的所有 face
     └─ 入队 FacialRecognition{id, deferred:false}

handleRecognizeFaces(job):
  ├─ 过滤：非 ML / 无 embedding / 已有 personId → 跳过
  ├─ 向量搜索 neighbors
  ├─ isCore = (matches >= minFaces) && 资产在 Timeline
  ├─ !isCore && !deferred → 重新入队(deferred:true), 退出
  ├─ 第二轮: 无论 core 与否都继续执行
  │    ├─ 若已有 personId 匹配 → reassignFaces
  │    ├─ 若 isCore && 无 person → 创建新 person
  │    └─ 否则保留 personId=null, 等待下次 QueueAll
```

---

## 7. 一句话总结

- **情况 A**：旧 ML face 与新框重叠，保持原状，靠 `FacialRecognitionQueueAll` 再排队。
- **情况 B**：手工 face 命中，只更新 embedding，永不进入识别。
- **情况 C**：全新 ML face，立即入队 `FacialRecognition` + `FacialRecognitionQueueAll`，若首轮聚类不满足 core，会被带 `deferred: true` 再入队一次。
- **情况 D**：旧 ML face 被新检测淘汰，直接删除。
- **deferred 的触发**：首轮识别时「neighbors 不足或资产不在时间线」导致 `!isCore`，将其延迟到下一轮；第二轮无论是否 core 都强制完成识别。
