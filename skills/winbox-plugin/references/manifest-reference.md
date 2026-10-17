# plugin.json 字段与白名单速查

> 唯一权威：`SPEC.md` §4~§6。本文件是**速查**，冲突时以规范为准。

## 字段
| 字段 | 类型 | 必填 | 约束 |
|---|---|---|---|
| `manifestVersion` | int | ✅ | 恒 `1` |
| `id` | string | ✅ | `[a-z0-9-]{3,40}`，唯一；决定安装目录 `<id>.winbox` |
| `name` | string | ✅ | 显示名 |
| `version` | string | ✅ | 语义化版本（更新检测用） |
| `apiVersion` | string | ✅ | 宿主 API **大版本**（如 `1.x`）；不匹配 → 拒装 |
| `kind` | `asset` \| `code` | ✅ | asset=声明式；code=L4 沙箱代码 |
| `main` | string | `code` 必填 | 沙箱入口相对名（如 `main.js`）；`asset` **不得有** |
| `author` / `license` | string | ✅ | 署名 / 许可 |
| `homepage` | string | 可选 | |
| `description` | string | 可选 | |
| `permissions` | string[] | ✅ | `asset` **必须为空**；`code` **必须**取下方白名单 |
| `permissionNotes` | object | 可选 | 每个权限的一句话用途（安装时展示给用户） |
| `contributes` | object[] | ✅ | 见下方 contributes 白名单 |
| `runtimeDeps` | object[] | 可选 | **只引用内置许可清单的 dep id** |
| `updateCheckUrl` | string | 可选 | `https://`；作者自配的更新源 |

## contributes 白名单（含允许的 kind）
> ★ **「已在宿主实装」一列**：✅ = 现在就能真正生效；空白 = 宿主已认识、但装配接线尚未落地（装上会被忽略并告警）。

| type | 允许 kind | 阶段 | 已在宿主实装 |
|---|---|---|---|
| `mpv.shaders` | asset | M1 | ✅ |
| `theme` | asset | M2 | ✅ |
| `subtitle.provider` | asset / code | M2 | ✅（asset 声明式；code 形待 L4） |
| `meta.provider` | asset / code | M2 | ✅（asset 声明式；code 形待 L4） |
| `discover.source` | asset / code | M2 | ✅（asset 声明式；code 形待 L4） |
| `danmaku.source` | asset / code | M2 | ✅（asset 声明式；code 形待 L4） |
| `live.template` | asset | M2 | ✅（输出侧格式化；★ 不发请求） |
| `player.engine` | code | M2（风险高，需显式授权） | |
| `ui.slot` / `settings.panel` / `keymap` | asset | M2 | |
| `ui.command` | asset / code | M2 | |

**禁用**：`code.spider`（引擎解析行为）· `main.hook`（主进程执行）· `drive.provider`（网盘凭据）

## permissions 白名单（**只属 L4 / `kind:"code"`**）
`net` · `net.download` · `kv` · `settings` · `fs.plugin` · `log` · `notify` · `clipboard.read` · `clipboard.write` · `dialog` · `player.control` · `openExternal`

**禁用（永不提供）**：`main` · `spawn`/`child_process` · `eval` · `fs`（任意文件系统）· `cred` · `proxy`（含 `raw.net`）· `registry`

## `mpv.shaders` 结构
```jsonc
{ "type": "mpv.shaders",
  "modes": [ { "id": "efficiency", "label": "效率档", "hint": "弱机可用", "shaders": ["a.glsl", "b.glsl"] } ] }
```
- `shaders` **顺序即生效顺序**；名称**先插件 `files/`、再 runtimeDeps 目录**解析为绝对路径（**不要写绝对路径**）。

## `subtitle.provider` 结构（L2 声明式字幕源）
```jsonc
{ "type": "subtitle.provider", "id": "my-subs", "name": "我的字幕站",
  "search": { "url": "https://sub.example.com/api/search?q={keyword}&ep={ep}",
              "list": "data.list",
              "map": { "file": "id", "subname": "filename", "title": "video_title", "lang": "language" } },
  "fetch":  { "url": "https://sub.example.com/api/download/{file}" } }
```
- **宿主代发请求**（插件不跑代码）：v1 仅 `GET` + **JSON** 响应；`fetch.url` 必须含 `{file}`。
- 占位符：search 用 `{keyword}/{keywords}/{title}/{ep}`；fetch 用 `{file}/{subname}/{title}/{ep}`。
- JSON 路径只支持**点号 + 下标**（`data.list` / `a.b[0]`），无 `[*]`、无函数。
- 全量口径 / 自检 / 常见坑 → `references/subtitle-provider-reference.md`；骨架 → `assets/plugin.subtitle-provider.json`。

## `meta.provider` 结构（L2 声明式元数据源）
```jsonc
{ "type": "meta.provider", "id": "my-meta", "name": "我的元数据站",
  "imageBase": "https://img.example.com/t/p/w342",
  "search": { "url": "https://api.example.com/search?q={keyword}&year={year}",
              "list": "data.results",
              "map": { "title": "name", "year": "year", "poster": "poster_path",
                       "backdrop": "wide_path", "overview": "overview", "mediaType": "type" } } }
```
- **兜底定位**：内置 TMDB → 豆瓣 都未命中时才轮到它；**不抢占内置来源**。
- 占位符：`{keyword}` / `{title}` / `{year}`；v1 **不提供评分**（`rating` 会被拒装）。
- 海报为相对路径时用 `imageBase` 拼前缀；未配则丢弃该图（不产出坏图）。
- 全量口径 / 自检 / 常见坑 → `references/meta-provider-reference.md`；骨架 → `assets/plugin.meta-provider.json`。

## `discover.source` 结构（L2 声明式发现页数据源）
```jsonc
{ "type": "discover.source", "id": "my-discover", "name": "我的榜单",
  "sections": [
    { "id": "my-top-movie", "title": "我的电影热榜", "mediaType": "movie",
      "url": "https://api.example.com/top?type={mediaType}",
      "list": "data.items",
      "imageBase": "https://img.example.com/t/p/w342",
      "map": { "title": "name", "year": "year", "poster": "poster_path", "backdrop": "wide_path", "mediaType": "type", "tmdbId": "tmdb_id" } }
  ] }
```
- **增补定位**：分区**追加**在发现页内置榜单**之后**；同 id 的分区被忽略。
- 占位符只有 `{mediaType}`；`sections` ≥1、同插件内 `section.id` 不重复；空分区不显示。
- **`map` 不含 `doubanId`**（会错配豆瓣剧照）；图片由宿主统一包 `/img` 中继（相对路径配 `imageBase`，未配则丢图）。
- 全量口径 / 自检 / 常见坑 → `references/discover-source-reference.md`；骨架 → `assets/plugin.discover-source.json`。

## `danmaku.source` 结构（L2 声明式弹幕源，**三步链路缺一不可**）
```jsonc
{ "type": "danmaku.source", "id": "my-danmaku", "name": "我的弹幕站",
  "search":   { "url": "https://dm.example.com/api/search?keyword={keyword}&season={season}",
                "list": "animes", "map": { "animeId": "animeId", "title": "animeTitle", "bangumiId": "bangumiId" } },
  "episodes": { "url": "https://dm.example.com/api/bangumi/{animeId}",
                "list": "bangumi.episodes", "map": { "episodeId": "episodeId", "episodeNumber": "episodeNumber" } },
  "comments": { "url": "https://dm.example.com/api/comment/{episodeId}?format=xml" } }
```
- ① 搜番剧 → ② 取剧集 → ③ 拉弹幕 **XML**（`comments` 无 `list`/`map`，**只收 XML 文本**）。
- 占位符按段白名单：`{keyword}/{season}` · `{animeId}/{title}` · `{episodeId}`；`comments.url` 必须含 `{episodeId}`。
- 与内置外部接口**并列**参与检索；★ v1 **不进**播放器「弹幕源」面板的开关清单（要停用请在「配置 → 插件」停用该插件）。
- 全量口径 / 自检 / 常见坑 → `references/danmaku-source-reference.md`；骨架 → `assets/plugin.danmaku-source.json`。

## `live.template` 结构（L2 声明式**输出**模板，★ **不发请求**）
```jsonc
{ "type": "live.template", "id": "my-live-txt", "label": "标准 TXT", "format": "txt",
  "group": "{group},#genre#", "line": "{name},{url}", "ext": "txt" }
```
- ★ 与前几个 provider **根本不同**：它**不取数、不发请求、不解析** —— 只把宿主**已解析**的直播频道
  （分组 / 频道 / 线路）按你的格式串**格式化导出**成 TXT / M3U 文件（出口：直播页顶栏 + 配置页直播线路每行「导出」）。
- 因此**没有** `url` / `headers` / `list` / `map`（写了即拒装）。
- `format` ∈ `txt` | `m3u`（决定默认扩展名）；`line` **必须含 `{url}`**（允许 `\n` 让 m3u 的 EXTINF 与地址分两行）；
  `group` / `header` 是单行；占位符仅 `{group}/{name}/{url}/{logo}/{tvgId}/{tvgName}/{tvgChno}/{index}`。
- 一个插件可声明**多个**模板（各自独立 id）；宿主自带 2 个内置模板（`builtin:txt` / `builtin:m3u`）。
- 全量口径 / 自检 / 常见坑 → `references/live-template-reference.md`；骨架 → `assets/plugin.live-template.json`。

## runtimeDeps
```jsonc
"runtimeDeps": [ { "id": "anime4k-shaders", "files": ["Anime4K_Upscale_CNN_x2_M.glsl"] } ]
```
- 小资源（<1MB）建议直接进 `files/` 随包；重运行库才走 `runtimeDeps`（**安装后**才下载）。
