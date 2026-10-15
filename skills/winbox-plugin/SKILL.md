---
name: winbox-plugin
description: Author, validate, and package Win-Box .winbox plugins. Use when the user asks to build a Win-Box plugin (插件), or about plugin.json, contributes, permissions, mpv.shaders. Do not use for TVBox source rules.
---

# Win-Box 插件开发

为 Win-Box 制作 / 校验 / 打包 `.winbox` 插件。

## 唯一事实源（先读）
- **规范**（字段 / 白名单 / 校验的唯一权威）：仓库根目录的 **`SPEC.md`**

> 你的产出**不得与规范冲突**；若发现冲突，**以规范为准**，并提醒用户更新规范。

## 铁律
1. **`permissions` 只属 L4**（`kind:"code"`）；L1/L2/L3（`kind:"asset"`）**必须** `permissions: []`。
2. 第三方代码**绝不进主进程**：L4 只在宿主**沙箱 worker** 里跑。
3. UI 不做任意 React 组件注入（受限词汇表 / `iframe`）。
4. **不触碰引擎解析行为**（`code.spider` 等贡献一律禁用）。
5. `runtimeDeps` **只能引用宿主内置许可清单的 dep id**，**不得自报 url/sha256**。

## 工作流
1. **定位级别** → 决定 `kind`：
   - 只是"给宿主一批文件 / 占个槽位" → `asset`（L1/L2/L3），`permissions` 留空；
   - 要"自己跑逻辑"（出网 / 存数据 / 控制播放）→ `code`（L4），且**必须**有 `main` + `permissions`。
2. **写 `plugin.json`**：骨架见 `assets/plugin.asset.json` / `assets/plugin.code.json`；字段与约束见 `references/manifest-reference.md`。
3. **放资源**：资源进 `files/`；要 mpv 档位 → `contributes:[{type:"mpv.shaders", modes:[…]}]`（**顺序即生效顺序**）。
4. **打包**：zip → 改名 `<id>.winbox`（`plugin.json` 必须在**根级**，不得多套目录层）。
5. **自检**：逐条走 `references/checks.md`；对照规范 §10 的拒装原因码。

## 速查
| 级别 | kind | contributes | permissions | 例子 |
|---|---|---|---|---|
| L1 声明式资源 | `asset` | 必填 | 空 | `mpv.shaders`、`theme` |
| L2 声明式数据源 | `asset` | 必填 | 空 | 弹幕 / 字幕 / 元数据源 |
| L3 UI 槽位 | `asset` | 必填 | 空 | `ui.slot`、`ui.command` |
| L4 沙箱代码 | `code` | 可组合 | 必填（白名单） | 需出网 / 存数据 / 控制播放 |

## 参考
- 字段与白名单速查：`references/manifest-reference.md`
- 发布前校验清单：`references/checks.md`
- 骨架与完整示例：`assets/`
