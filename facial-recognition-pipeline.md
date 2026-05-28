# Immich 人脸识别流水线（Face Detection → Recognition）数据流解析

本文结合 Immich Server 与 Immich ML 子项目源码，自顶向下描述一次 `detectFaces` 调用从 `FormData` 到 `FacialRecognitionOutput` 的完整数据流，并解释响应里额外携带 `imageHeight`/`imageWidth` 的原因，以及依赖缺失时会抛出什么错误。

---

## 1. Server 端：`MachineLearningRepository.detectFaces` 如何组装请求

入口位于 [machine-learning.repository.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/machine-learning.repository.ts#L194-L207)：

```ts
async detectFaces(imagePath: string, { modelName, minScore }: FaceDetectionOptions) {
  const request = {
    [ModelTask.FACIAL_RECOGNITION]: {
      [ModelType.DETECTION]: { modelName, options: { minScore } },
      [ModelType.RECOGNITION]: { modelName },
    },
  };
  const response = await this.predict<FacialRecognitionResponse>({ imagePath }, request);
  return {
    imageHeight: response.imageHeight,
    imageWidth: response.imageWidth,
    faces: response[ModelTask.FACIAL_RECOGNITION],
  };
}
```

关键点：

- 一次请求同时声明了两个模型：`detection`（人脸检测）和 `recognition`（特征嵌入），两个模型共享同一个 `modelName`（例如 `buffalo_l`）。
- 请求体由私有方法 [getFormData](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/machine-learning.repository.ts#L232-L246) 组装为 `multipart/form-data`：

```ts
formData.append('entries', JSON.stringify(config));      // ①
formData.append('image',   new Blob([new Uint8Array(fileBuffer)])); // ②
// 如果是文本任务则改为 formData.append('text', payload.text)
```

| FormData 字段 | 内容 | 说明 |
|---|---|---|
| `entries` | `{"facial-recognition":{"detection":{"modelName":"...","options":{"minScore":0.7}},"recognition":{"modelName":"..."}}}` | 描述要执行的 pipeline，一个 task 下可能含多个 type。 |
| `image`   | 原始图片字节（Blob） | 对于视觉类任务使用。 |
| `text`    | 原始文本字符串 | 对于 CLIP-text 等文本类任务使用，与 `image` 互斥。 |

随后 `predict` 把它 `POST` 到 ML 服务的 `/predict` 端点，JSON 响应会被解析成 `FacialRecognitionResponse`：

```ts
type FacialRecognitionResponse =
  { [ModelTask.FACIAL_RECOGNITION]: Face[] } & VisualResponse;
// VisualResponse = { imageHeight: number; imageWidth: number }
```

---

## 2. ML 端：`main.py` 如何把 entries 拆成 `without_deps` / `with_deps`

ML 服务入口在 [immich_ml/main.py](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/main.py)。

### 2.1 `get_entries`：解析 pipeline 并做依赖拆分

[predict 路由](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/main.py#L165-L181) 通过 FastAPI 的 `Depends(get_entries)` 解析 FormData：

```python
def get_entries(entries: str = Form()) -> InferenceEntries:
    request: PipelineRequest = orjson.loads(entries)
    without_deps: list[InferenceEntry] = []
    with_deps: list[InferenceEntry] = []
    for task, types in request.items():
        for type, entry in types.items():
            parsed: InferenceEntry = {
                "name": entry["modelName"],
                "task": task,
                "type": type,
                "options": entry.get("options", {}),
            }
            dep = get_model_deps(parsed["name"], type, task)
            (with_deps if dep else without_deps).append(parsed)
    return without_deps, with_deps
```

返回的 `InferenceEntries` 类型在 [schemas.py](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/schemas.py#L112)：

```python
InferenceEntries = tuple[list[InferenceEntry], list[InferenceEntry]]
```

即 `(without_deps, with_deps)` 的二元组。

`get_model_deps` 在 [models/\_\_init\_\_.py](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/models/__init__.py#L47-L48) 里委托给具体模型类的类属性 `depends`：

```python
def get_model_deps(...):
    return get_model_class(model_name, model_type, model_task).depends
```

对于人脸识别任务：

| 模型类 | `depends` | `identity` |
|---|---|---|
| [FaceDetector](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/models/facial_recognition/detection.py#L12-L14) | `[]` | `(detection, facial-recognition)` |
| [FaceRecognizer](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/models/facial_recognition/recognition.py#L26-L28) | `[(detection, facial-recognition)]` | `(recognition, facial-recognition)` |

因此 `detection` 会被放入 `without_deps`，`recognition` 会被放入 `with_deps`。

### 2.2 `run_inference`：先无依赖，再把输出喂给有依赖模型

[run_inference](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/main.py#L184-L211) 是调度核心：

```python
async def run_inference(payload, entries):
    outputs: dict[ModelIdentity, Any] = {}
    response: InferenceResponse = {}

    async def _run_inference(entry):
        model = await model_cache.get(entry["name"], entry["type"], entry["task"], ...)
        inputs = [payload]
        for dep in model.depends:
            inputs.append(outputs[dep])           # ★ 从 outputs 里取依赖结果
        model = await load(model)
        output = await run(model.predict, *inputs, **entry["options"])
        outputs[model.identity] = output           # ★ 把自己的输出挂到 outputs
        response[entry["task"]] = output

    without_deps, with_deps = entries
    await asyncio.gather(*[_run_inference(e) for e in without_deps])  # 并行
    if with_deps:
        await asyncio.gather(*[_run_inference(e) for e in with_deps])   # 再并行
    if isinstance(payload, Image):
        response["imageHeight"], response["imageWidth"] = payload.height, payload.width
    return response
```

要点：

- **执行顺序被强制保证**：`without_deps` 全部跑完后才跑 `with_deps`，而不是在模型声明顺序上投机。
- **数据桥**：`outputs[ModelIdentity]` 以 `(type, task)` 作为 key，使得依赖方可以直接按 identity 取到上游输出。
- **`response[entry["task"]] = output`** 是按 task 覆盖写入的。同一 task 下后执行的模型会覆盖前一个，所以 `facial-recognition` 任务下 `recognition` 总会覆盖 `detection`，最终客户端拿到的是识别结果（嵌入），而不是纯检测结果。

---

## 3. 从原始 bytes 到 `FaceDetectionOutput` 再到 `FacialRecognitionOutput`

### 3.1 图像解码（bytes → PIL.Image）

[predict 路由](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/main.py#L165-L181)：

```python
if image is not None:
    decoded = await run(lambda: decode_pil(image))
    if decoded.width == 0 or decoded.height == 0:
        raise HTTPException(400, "Image has zero width or height")
    inputs = decoded
```

[decode_pil](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/models/transforms.py#L50-L57) 用 Pillow 打开 bytes，强制转成 RGB：

```python
def decode_pil(image_bytes):
    image = Image.open(BytesIO(image_bytes))
    image.load()
    if not image.mode == "RGB":
        image = image.convert("RGB")
    return image
```

得到的 `PIL.Image.Image` 既是后续推理的输入，也是响应里 `imageWidth`/`imageHeight` 的来源。

### 3.2 检测阶段（PIL → `FaceDetectionOutput`）

[FaceDetector._predict](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/models/facial_recognition/detection.py#L27-L35)：

```python
def _predict(self, inputs) -> FaceDetectionOutput:
    inputs = decode_cv2(inputs)           # PIL → BGR ndarray
    bboxes, landmarks = self._detect(inputs)
    return {
        "boxes":     bboxes[:, :4].round(),
        "scores":    bboxes[:, 4],
        "landmarks": landmarks,
    }
```

`FaceDetectionOutput` 在 [schemas.py](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/schemas.py#L82-L85)：

```python
class FaceDetectionOutput(TypedDict):
    boxes:     NDArray[np.float32]     # (N, 4)，已经 .round() 到整数
    scores:    NDArray[np.float32]     # (N,)
    landmarks: NDArray[np.float32]     # (N, 5, 2)
```

- `decode_cv2`（见 [transforms.py](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/models/transforms.py#L60-L67)）把 PIL 转成 BGR numpy 数组供 RetinaFace 使用。
- `RetinaFace.detect()` 返回 `(boxes, landmarks)`，其中 `boxes` 的第 5 列是置信度分数。

这一份 dict 会被挂到 `outputs[(detection, facial-recognition)]`。

### 3.3 识别阶段（`FaceDetectionOutput` → `FacialRecognitionOutput`）

[FaceRecognizer._predict](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/models/facial_recognition/recognition.py#L46-L54)：

```python
def _predict(self, inputs, faces: FaceDetectionOutput) -> FacialRecognitionOutput:
    if faces["boxes"].shape[0] == 0:
        return []
    inputs = decode_cv2(inputs)
    cropped_faces = self._crop(inputs, faces)      # 按 landmarks 做对齐裁剪
    embeddings = self._predict_batch(cropped_faces) # ArcFace 生成 (N, 512)
    return self.postprocess(faces, embeddings)
```

[postprocess](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/models/facial_recognition/recognition.py#L66-L74) 把 numpy 转成可 JSON 化的 list：

```python
def postprocess(self, faces, embeddings):
    return [
        {
            "boundingBox": {"x1": x1, "y1": y1, "x2": x2, "y2": y2},
            "embedding":   serialize_np_array(embedding),   # → JSON 字符串
            "score":       score,
        }
        for (x1,y1,x2,y2), embedding, score
        in zip(faces["boxes"], embeddings, faces["scores"])
    ]
```

`FacialRecognitionOutput` 即 `list[DetectedFace]`，`DetectedFace` 在 [schemas.py](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/schemas.py#L88-L94)：

```python
class DetectedFace(TypedDict):
    boundingBox: BoundingBox
    embedding:   str        # 已序列化成 JSON 字符串，便于客户端直接入库/传输
    score:       float
```

---

## 4. 为什么响应里还会带 `imageHeight`/`imageWidth`

原因在 [run_inference 末尾](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/main.py#L208-L209)：

```python
if isinstance(payload, Image):
    response["imageHeight"], response["imageWidth"] = payload.height, payload.width
```

这是一个**通用约定**，对所有视觉类 task 都生效（人脸、OCR、CLIP-visual）。客户端之所以需要它：

1. **检测框坐标通常是** 基于模型内部 `input_size`（RetinaFace 默认 `(640,640)`）的比例/像素值，`boxes.round()` 返回的是数值，不代表原图绝对坐标；Server 端在后续把框写进数据库或 UI 渲染时，需要知道原图尺寸才能进行正确的缩放/归一化。
2. `VisualResponse` 在 TS 侧被 `FacialRecognitionResponse`、`OcrResponse`、`ClipVisualResponse` 交叉继承，说明所有视觉模型都统一约定输出图像元信息，便于 SDK 层泛化处理（参见 [machine-learning.repository.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/machine-learning.repository.ts#L40-L75)）。

注意：只有 `payload` 是 `PIL.Image.Image` 实例时才附带这两个字段；纯文本任务（CLIP textual）的响应里不会有它们。

---

## 5. 如果依赖输出不存在，会返回什么错误

依赖查找在 [_run_inference 内的 inputs 组装](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/main.py#L188-L199)：

```python
inputs = [payload]
for dep in model.depends:
    try:
        inputs.append(outputs[dep])
    except KeyError:
        message = f"Task {entry['task']} of type {entry['type']} depends on output of {dep}"
        raise HTTPException(400, message)
```

因此缺失依赖时的错误信息为：

> `HTTP 400 Bad Request`，消息形如：
> `Task facial-recognition of type recognition depends on output of (detection, facial-recognition)`

触发场景一般是：

1. `entries` 里只声明了 `recognition`，没有声明 `detection`（用户把 pipeline 写错了）。
2. `without_deps` 中某个模型抛错后提前返回，导致 `outputs` 里没有对应 key（但 `without_deps` 阶段是用 `gather`，一个异常会把整个协程取消，通常不会继续跑 `with_deps`）。
3. 手工通过 `curl` 调 `/predict` 时只提交了 `recognition` 这一项。

另外还有两个与输入相关的 `400`，虽然不是“依赖输出不存在”，但在同一条链路上常被一起遇到：

- `Either image or text must be provided`：FormData 里既没有 `image` 也没有 `text`。
- `Image has zero width or height`：`decode_pil` 解出来的图像宽/高为 0。

以及 `get_entries` 解析失败时：

- `HTTP 422 Unprocessable Entity`：`orjson.JSONDecodeError` / `ValidationError` / `KeyError` / `AttributeError`，消息 `Invalid request format.`（见 [main.py:147-149](file:///c:/Users/10244/Desktop/0508-under/immich/machine-learning/immich_ml/main.py#L147-L149)）。

---

## 6. 全链路数据形态一览

| 阶段 | 载体 | 数据形态 |
|---|---|---|
| Server `getFormData` | `multipart/form-data` | `entries` = JSON 字符串；`image` = Blob |
| ML `get_entries` | Python tuple | `(without_deps, with_deps)`，每项是 `InferenceEntry` |
| ML `run_inference` 无依赖 | `PIL.Image.Image` | 输入：`[pil_image]` |
| `FaceDetector._predict` | `FaceDetectionOutput` | `{boxes, scores, landmarks}`，均为 numpy |
| ML `run_inference` 有依赖 | 拼出来的 inputs | `[pil_image, FaceDetectionOutput]` |
| `FaceRecognizer._predict` | `FacialRecognitionOutput` | `[{"boundingBox":..., "embedding":"...", "score":...}, ...]` |
| ML `run_inference` 收尾 | `InferenceResponse` | `{"facial-recognition": [...], "imageHeight": H, "imageWidth": W}` |
| Server `predict` 反序列化 | `FacialRecognitionResponse` | TS 侧对象 |
| Server `detectFaces` 返回 | `DetectedFaces` | `{faces, imageHeight, imageWidth}` |

整条链路通过两个“隐式契约”贯通：

1. **模型类的 `depends` 类属性**决定了 `without_deps`/`with_deps` 的划分，也决定了 `outputs[dep]` 的查找顺序。
2. **`InferenceResponse` 的 key 空间混用了 `ModelTask` 和字面量 `imageHeight`/`imageWidth`**，这就是为什么响应里“多出两个字段”——它们不属于任何一个 task，而是 pipeline 级的元信息。
