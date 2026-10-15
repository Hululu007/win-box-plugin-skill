# 发布前校验清单（工作流）

发布 `.winbox` 前，逐条核对；任一不通过就修，别发布。

## 1. 结构
- [ ] 根级有 `plugin.json`（**不得**多套目录层）
- [ ] 文件名 = `<id>.winbox`，且 `id` 与清单一致
- [ ] 包内附 `LICENSE`

## 2. 清单字段
- [ ] `manifestVersion = 1`
- [ ] `id` 符合 `[a-z0-9-]{3,40}`
- [ ] `apiVersion` 与目标宿主**大版本**一致
- [ ] `kind` 与 `permissions` 匹配：`asset` → `[]` 且无 `main`；`code` → 白名单非空且有 `main`
- [ ] `contributes[].type` 在宿主白名单内
- [ ] 所有路径（`main`/`files`/`shaders`）为**相对名**、无 `..`、无绝对路径
- [ ] `modes[].shaders` ⊆ 实际提供的文件
- [ ] `runtimeDeps[].id` 都在**内置许可清单**内（未自报 url）
- [ ] `updateCheckUrl` 为 `https://`（作者自配）

## 3. 行为约定
- [ ] 插件可被**随时停用**而不报错
- [ ] 若贡献正生效（如超分档位），**移除**时能回落默认
- [ ] 需要 `net` 的插件已在 `permissions` + `permissionNotes` 说明用途

## 4. 对照拒装原因码（规范 §10）
`MISSING_MANIFEST` · `ID_INVALID` · `API_VERSION_MISMATCH` · `KIND_PERM_MISMATCH` · `UNKNOWN_CONTRIBUTE` · `PATH_ESCAPE` · `SHA_MISMATCH` · `INCOMPLETE` · `DEP_NOT_ALLOWED`

## 5. 打包（PowerShell 示例）
```powershell
Compress-Archive -Path plugin.json, files, LICENSE -DestinationPath out.zip
Rename-Item out.zip <id>.winbox
```
