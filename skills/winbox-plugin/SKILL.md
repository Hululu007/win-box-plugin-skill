---
name: winbox-plugin
description: Author, validate, and package Win-Box .winbox plugins. Use when the user asks to build a Win-Box plugin (插件), or about plugin.json, contributes, permissions, mpv.shaders. Do not use for TVBox source rules.
---

# Win-Box 插件开发

为 Win-Box 制作 / 校验 / 打包 `.winbox` 插件。

## 冷启动（先读这两页）
1. `references/host-overview.md` —— **宿主速览**（Win-Box 是什么 / 双内核与 GLSL 只作用 MPV / M1 白名单 / 插件在 UI 何处出现）。**冷启动必读**，补齐项目背景常识。
2. `SPEC.md`（仓库根）—— **规范**：字段 / 白名单 / 校验的唯一权威；**皮肤专项口径（令牌词表 / 布局枚举 / 拒装原因码）在 §15**。

> 你的产出**不得与规范冲突**；若发现冲突，**以规范为准**，并提醒用户更新规范。
> 需要核对白名单 / 宿主实现时才**定向**看主仓（`github.com/yagami-hue/Win-box`；M1 后看 `src/shared/plugin.ts`）——**不要通读整个仓库**。

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
   - `modes[].shaders` 里的名字必须与 `files/` 里**实际文件同名**（`.glsl` / `.hook` 都行，mpv 不看扩展名）；**别照抄占位名**。
   - 着色器资源多为**第三方上游项目**：随包了第三方资源就要在 `licenses/` 归档其许可原文（copyleft 必做，见规范 §3）。
   - 要**皮肤** → `contributes:[{type:"theme", id, label, base, tokens, layout}]`：
     皮肤 = **令牌字典 + 布局声明**，**绝对不写任何 CSS / 选择器 / `.css` 文件**（v1 硬约束）。
     词表（39 项）/ 布局枚举 / `base` 口径 / 自检清单 / **可安装的发布产物示例** → `references/theme-reference.md`；
     骨架 → `assets/plugin.theme.json`；示例 `.winbox` → `assets/theme-sakura.winbox`。
   - 要**字幕源** → `contributes:[{type:"subtitle.provider", id, name, search:{url,list,map}, fetch:{url}}]`：
     宿主**代发请求**（你不跑代码）：v1 仅 `GET` + JSON 响应；URL 里用占位符、JSON 里用**受限点号路径**取值。
     需要 POST / token / 解析 HTML → **只能用 L4**（`kind:"code"`）。
     结构 / 占位符 / 路径语法 / 自检 / 常见坑 → `references/subtitle-provider-reference.md`；骨架 → `assets/plugin.subtitle-provider.json`。
   - 要**元数据源** → `contributes:[{type:"meta.provider", id, name, imageBase?, search:{url,list,map}}]`：
     同上仅 `GET` + JSON。★ 它是**兜底**（内置 TMDB / 豆瓣都未命中才轮到它），**不抢占内置来源**；
     v1 **不提供评分**（声明 `rating` 会被拒装）；海报为相对路径时用 `imageBase` 拼前缀。
     结构 / 自检 / 常见坑 → `references/meta-provider-reference.md`；骨架 → `assets/plugin.meta-provider.json`。
   - 要**发现页数据源** → `contributes:[{type:"discover.source", id, name, sections:[{id,title,mediaType?,url,list,map,imageBase?}]}]`：
     同上仅 `GET` + JSON。★ 它是**增补**：分区**追加**在发现页内置榜单**之后**（同 id 被忽略）；
     `map` **不含 `doubanId`**（会错配豆瓣剧照）；图片一律由宿主包 `/img` 中继，你只写原始地址。
     结构 / 自检 / 常见坑 → `references/discover-source-reference.md`；骨架 → `assets/plugin.discover-source.json`。
   - 要**弹幕源** → `contributes:[{type:"danmaku.source", id, name, search:{…}, episodes:{…}, comments:{…}}]`：
     ★ **三步链路缺一不可**（搜番剧 → 剧集列表 → 弹幕 XML）；三段 URL 各用占位符（`{keyword}/{season}`、`{animeId}/{title}`、`{episodeId}`）。
     ★ `comments` **只收 XML**（B 站格式；无 `list`/`map`）—— 只给 JSON 弹幕的接口本期做不了，需 L4。
     结构 / 自检 / 常见坑 → `references/danmaku-source-reference.md`；骨架 → `assets/plugin.danmaku-source.json`。
   - 要**直播输出模板** → `contributes:[{type:"live.template", id, label, format:"txt"|"m3u", group?, header?, line, ext?}]`：
     ★ **本类型不发请求、不解析** —— 只把宿主**已解析**的直播频道按你的格式串导出为 TXT / M3U（**没有** `url`/`map`）。
     ★ `line` **必须含 `{url}`**（m3u 可用 `\n` 让 EXTINF 与地址分两行）；一个插件可声明**多个**模板。
     结构 / 自检 / 常见坑 → `references/live-template-reference.md`；骨架 → `assets/plugin.live-template.json`。
4. **打包**：zip → 改名 `<id>.winbox`（`plugin.json` 必须在**根级**，不得多套目录层）。
5. **自检**：逐条走 `references/checks.md`；对照规范 §10 的拒装原因码。

## 速查
| 级别 | kind | contributes | permissions | 例子 |
|---|---|---|---|---|
| L1 声明式资源 | `asset` | 必填 | 空 | `mpv.shaders`、`theme` |
| L2 声明式数据源 | `asset` | 必填 | 空 | 弹幕 / 字幕 / 元数据源；`live.template`（输出模板） |
| L3 UI 槽位 | `asset` | 必填 | 空 | `ui.slot`、`ui.command` |
| L4 沙箱代码 | `code` | 可组合 | 必填（白名单） | 需出网 / 存数据 / 控制播放 |

## 参考
- 宿主速览（冷启动必读）：`references/host-overview.md`
- 字段与白名单速查：`references/manifest-reference.md`
- **皮肤（theme）参考**：`references/theme-reference.md`（令牌词表 + 布局枚举 + 自检 + **发布产物示例与自测三步**）
- **字幕源（subtitle.provider）参考**：`references/subtitle-provider-reference.md`（结构 + 占位符 + JSON 路径 + 自检 + 常见坑）
- **元数据源（meta.provider）参考**：`references/meta-provider-reference.md`（结构 + 兜底定位 + imageBase + 自检 + 常见坑）
- **发现页数据源（discover.source）参考**：`references/discover-source-reference.md`（分区结构 + 增补定位 + /img 中继 + 自检 + 常见坑）
- **弹幕源（danmaku.source）参考**：`references/danmaku-source-reference.md`（三步链路 + 按段占位符 + 只收 XML + 自检 + 常见坑）
- **直播输出模板（live.template）参考**：`references/live-template-reference.md`（结构 + 占位符 + 渲染规则 + ★ 不发请求 + 自检 + 常见坑）
- 发布前校验清单：`references/checks.md`
- 骨架与示例产物：`assets/`（`plugin.asset.json` / `plugin.code.json` / `plugin.theme.json` / `plugin.subtitle-provider.json` / `plugin.meta-provider.json` / `plugin.discover-source.json` / `plugin.danmaku-source.json` / `plugin.live-template.json` / `example-super-resolution.plugin.json` / **`theme-sakura.winbox`（可直接装的示例皮肤包）**）
