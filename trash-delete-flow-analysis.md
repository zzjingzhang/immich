# 资产删除与恢复流程分析

## 一、三个入口流程比较

### 1. 回收站页面恢复选中资产
**路径**: `POST /trash/restore/assets`

| 层级 | 代码位置 | 关键操作 |
|------|----------|----------|
| Controller | [trash.controller.ts#L40-L50](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/controllers/trash.controller.ts#L40-L50) | `restoreAssets()` 接收 `BulkIdsDto` |
| Service | [trash.service.ts#L12-L25](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/trash.service.ts#L12-L25) | `restoreAssets()` 权限校验后调用 `trashRepository.restore