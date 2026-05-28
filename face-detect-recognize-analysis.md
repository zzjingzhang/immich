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

注意