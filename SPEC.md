# 插件制作规范（Win-Box 插件作者指南）

> 面向：**插件作者 + AI**（本文是 [`skills/winbox-plugin/`](skills/winbox-plugin/SKILL.md) 的**唯一事实源**；SKILL.md 不得手写第二套）。
> 状态：**v1 规范**；L4 沙箱 API 面为**契约冻结、按里程碑实现**。

---

## 1. 什么是 Win-Box 插件

- 插件 = 一个 **`.winbox` 包**（zip 容器），内含根级 `plugin.json` 清单与可选资源，安装进 `<userData>/plugins/<id>.winbox/`。
### 1.0 一张表看懂：级别 / `kind` / `contributes` / `permissions`
> ★ **先记住这一句**：**`permissions` 只属于 L4（`kind:"code"`）**。L1~L3 是**声明式**（不跑你的代码），`permissions` **必须是空数组**，能力全部由 `contributes` 表达。

| 级别 | 形态 | `kind` | 跑你的代码？ | `contributes` | `permissions` | 例子 |
|---|---|---|---|---|---|---|
| **L1 声明式资源** | 只"交出资源 / 占个槽位" | `asset` | 否 | ✅ 必填 | **必须为空** | 超分着色器、皮肤 |
| **L2 声明式数据源** | 声明式 provider（宿主按你的模板取数） | `asset` | 否 | ✅ 必填 | **必须为空** | 弹幕源、字幕源、元数据源 |
| **L3 UI 槽位** | 受限 UI（词汇表 / `iframe`） | `asset` | 否（iframe 内） | ✅ 必填 | **必须为空** | 发现页槽位、动作按钮 |
| **L4 沙箱代码** | 在宿主沙箱 worker 里跑你的 JS | `code` | 是 | ✅ 可组合 | ✅ **必填（白名单）** | 需出网 / 存数据 / 控制播放的插件 |

### 1.0.1 「我的插件属于哪一级？」判定路径
1. 你的插件**要不要自己跑逻辑**（出网请求、存数据、控制播放、自己算）？
   - **不要** → **L1/L2/L3**（`kind:"asset"`）：用 `contributes` 描述你贡献什么，**`permissions` 留空**。
   - **要** → **L4**（`kind:"code"`）：**必须**声明 `permissions`（见 §6），并把贡献写进 `contributes`。
2. 只是"给宿主一批文件"（着色器 / 皮肤）？ → **L1**。
3. "给宿主一个能自动取数的来源"？ → **L2**（声明式 provider）。
4. "往某个界面塞一段受限 UI"？ → **L3**（`ui.slot`）。

### 1.0.2 两个正交维度（**别混**）
- **`contributes` = 你贡献了什么**（注册到哪个槽位 / 交出什么资源）—— L1~L4 **都要写**；
- **`permissions` = 你的沙箱代码能调哪些宿主 API** —— **只有 L4 才有**。

### 1.1 三条红线（违反即拒装/下架）
1. **第三方代码绝不进主进程**——L4 只能在**沙箱 worker** 内跑。
2. **不做任意 React 组件注入**——UI 只用受限词汇表或 iframe。
3. **不触碰引擎解析行为**（1:1 复刻为门禁）；**不读写主进程凭据 / 不碰 `9978` 代理**。
> ★ **L4 是权限天花板**：宿主**不提供**任何"在主进程执行第三方代码"的能力；需要更深能力请走 §6 的授权 API。

---

## 2. 快速上手（最小可跑插件）

```
super-resolution.winbox            # 就是一个 zip，改后缀名
├── plugin.json                    # ★ 必备，根级
├── files/                         # 插件自带资源（此处为着色器）
│   ├── Anime4K_Clamp_Highlights.glsl
│   └── …
└── LICENSE
```

打包（Windows PowerShell）：
```powershell
# 注意：plugin.json 必须在压缩包【根级】，不要多套一层目录
Compress-Archive -Path plugin.json, files, LICENSE -DestinationPath super-resolution.zip
Rename-Item super-resolution.zip super-resolution.winbox
```

`plugin.json`（L1 最小示例）：
```jsonc
{
  "manifestVersion": 1,
  "id": "super-resolution",
  "name": "超分辨率",
  "version": "1.0.0",
  "apiVersion": "1.x",
  "kind": "asset",
  "author": "your-name",
  "license": "MIT",
  "description": "Anime4K / RAVU-Lite / FSRCNNX 实时超分辨率（MPV 内核）",
  "permissions": [],
  "contributes": [
    { "type": "mpv.shaders", "modes": [
      { "id": "efficiency", "label": "效率档", "shaders": ["Anime4K_Clamp_Highlights.glsl", "Anime4K_Upscale_CNN_x2_M.glsl"] }
    ] }
  ]
}
```

---

## 3. `.winbox` 包格式规范

| 项 | 规定 |
|---|---|
| 容器 | **zip**（宿主用 `extractArchiveEntries` 解，支持 store/deflate；7z/rar 不可用作 `.winbox`） |
| 根级必备 | `plugin.json`（**不得**多套目录层） |
| 可选 | `files/`（插件自带资源）、`LICENSE`、`README.md`、`licenses/`（第三方许可原文，见下） |
| 文件命名 | 一律 UTF-8、相对路径；**禁止绝对路径 / `..` / 盘符** |
| 包命名 | `<id>.winbox`（id 与 manifest 一致） |
| 第三方许可 | ★ 插件随包**第三方资源**（着色器等）时：根级 `LICENSE` = **插件自身**许可（与 manifest `license` 一致）；第三方许可原文放 `licenses/<上游名>.txt` **原样归档**，并在根 `LICENSE` 里加「署名 / 上游 / 许可对照」段。若上游是 copyleft（如 GPL-3 / LGPL-3），**必须**与资源一起分发其许可原文 + 上游地址（资源本身即"源码形态"） |
| 完整性 | 宿主按**内置许可清单**的 `sha256` 校验运行库；`.winbox` 本体可选签名字段（v1 未启用） |
| 大小上限 | v1 建议 ≤ 50MB（大资源请走 `runtimeDeps`，见 §7） |

---

## 4. `plugin.json` 字段参考

| 字段 | 类型 | 必填 | 约束 / 说明 |
|---|---|---|---|
| `manifestVersion` | int | ✅ | 恒 `1`（清单格式版本） |
| `id` | string | ✅ | 唯一；`[a-z0-9-]{3,40}`；决定安装目录 `<id>.winbox` |
| `name` | string | ✅ | 显示名 |
| `version` | string | ✅ | 语义化版本（更新检测用） |
| `apiVersion` | string | ✅ | 依赖的宿主 API **大版本**（如 `1.x`）；不匹配 → 拒装 |
| `kind` | `"asset"｜"code"` | ✅ | asset = 声明式；code = L4 沙箱代码 |
| `main` | string | `code` 必填 | **沙箱入口文件**相对名（如 `main.js`）；`kind=asset` **不得有** |
| `author` / `license` / `homepage` | string | ✅/✅/可选 | 署名与许可 |
| `description` | string | 可选 | 一句话说明 |
| `permissions` | string[] | ✅ | `kind=asset` **必须为空**；`kind=code` **必须**取 §6 白名单 |
| `permissionNotes` | object | 可选 | 每个权限的用途说明（安装时展示给用户） |
| `contributes` | object[] | ✅ | 声明式贡献点，见 §5 |
| `runtimeDeps` | object[] | 可选 | **只引用内置许可清单的 dep id**，见 §7 |
| `updateCheckUrl` | string | 可选 | `https://`，作者自配的更新源，见 §8 |

### 4.1 字段级校验（宿主逐条执行）
- `id` 唯一 + 命名规则；`apiVersion` 主版本须匹配宿主；
- `kind=asset` ⇒ `permissions` 为空、**不得有 `main`**；`kind=code` ⇒ `permissions` 非空且全部白名单内、**必须含 `main`（沙箱入口）**；
- `contributes[].type` 必须在宿主白名单内（M1 = `mpv.shaders`）；
- `runtimeDeps[].id` 必须在**内置许可清单**内（**禁止自报 url/sha256**）；
- `runtimeDeps[].files` 与 `modes[].shaders` 仅相对名、禁 `..`/绝对路径；后者须是前者子集；
- `updateCheckUrl` 仅 `https://`。

---

## 5. 能力：`contributes`（白名单）

### 5.1 白名单（含允许的 `kind`）
> `contributes` 与 `permissions` **正交**：下表 `type` 只有标 **`code` 才需要在 `permissions` 里额外声明**沙箱 API；标 `asset` 的纯声明式、不需要权限。

| type | 含义 | 允许 `kind` | 阶段 | 风险 | 宿主接线点 |
|---|---|---|---|---|---|
| `mpv.shaders` | MPV 着色器链（多档位） | `asset` | **M1** | 低 | `mpvCore.ts` 注入 |
| `theme` | 皮肤（**纯令牌 + 声明式 layout**，不含样式表） | `asset` | M2 | 低 | 见 §5.3 |
| `danmaku.source` | 弹幕源 | `asset` / `code` | M2 | 低 | 弹幕聚合 |
| `subtitle.provider` | 字幕源 | `asset` / `code` | M2 | 低 | 字幕 provider 注册表 |
| `meta.provider` | 元数据源（海报 / 简介 / 评分） | `asset` / `code` | M2 | 中（出网） | 元数据 provider 注册表 |
| `discover.source` | 发现页数据源（榜单 / 分类浏览） | `asset` / `code` | M2 | 中（出网） | 发现页数据 |
| `live.template` | 直播源**模板**（仅格式化输出，**不解析规则**） | `asset` | M2 | 中 | 直播输出 |
| `player.engine` | 追加播放内核 | **`code`** | M2 | **高**（本机二进制，需显式授权） | 播放器内核 |
| `ui.slot` | 受限 UI 槽位（声明式词汇表 / `iframe`） | `asset` | M2 | 低 | `WebHomeWindow` / `fm` 桥 |
| `ui.command` | 注册受限动作按钮（配置页 / 播放器） | `asset` / `code` | M2 | 低 | 渲染层受限渲染 |
| `settings.panel` | 插件设置面板（受限词汇表） | `asset` | M2 | 低 | 配置页 |
| `keymap` | 追加快捷键（不与宿主冲突） | `asset` | M2 | 低 | 快捷键分发 |

**禁用的 contributes（红线，声明即拒装）**：
- ❌ `code.spider` 及任何**引擎解析行为**相关贡献 —— 红线③（1:1 复刻）
- ❌ `main.hook` 及任何**主进程执行**贡献 —— 红线①
- ❌ `drive.provider`（网盘 provider）—— 触碰凭据（高危面）；M2 起**最多允许声明式只读描述**，**不得注册取流逻辑**

### 5.2 `mpv.shaders` 结构
```jsonc
{ "type": "mpv.shaders",
  "modes": [ { "id": "efficiency", "label": "效率档", "hint": "轻量，弱机可用", "shaders": ["a.glsl", "b.glsl"] } ] }
```
- `modes[].shaders` **顺序即生效顺序**（先 restore/clamp，再 upscale，最后 downscale 保护）；
- `shaders` 名称解析：**先在插件 `files/` 找，再到所声明的 `runtimeDeps` 目录找**，最终由宿主拼成**绝对路径**喂给 mpv（你不写绝对路径）；
- `shaders` 条目**只要求"相对名 + 路径安全"**，**不限扩展名**：`.glsl` 与 `.hook`（mpv-prescalers 形态）都可 —— mpv 按文件头 `//!HOOK` 等指令识别用户着色器，**不看扩展名**；**但必须与 `files/` 里实际文件同名**（宿主安装期校验存在性）；
- ★ 同一档位里的多条着色器**不要重复占用同一槽位**（如 RAVU-Lite 的 `ravu-lite-r4` 与 `ravu-lite-ar-r4` 都写 `//!SAVE ravu_lite_int`）——同槽位是**互斥**的，串联会互相覆盖；这类"变体"应**二选一**；
- 用户可见档位由宿主统一渲染为 `off / <各 mode.label>`；多插件都提供 `mpv.shaders` 时以**用户选择**为准。

> 宿主不认识 `contributes.type` → **忽略并告警**，不报错（前向兼容，便于宿主旧版跑新插件）。

### 5.3 `theme` 结构（皮肤）
```jsonc
{ "type": "theme",
  "id": "sakura",               // [a-z0-9-]{3,40}；生效值 = data-theme='plugin:sakura'
  "label": "🌸 樱花",            // 设置页显示名
  "base": "dark",               // dark | light（未覆盖令牌回落到该基底）
  "tokens": { "--accent": "#e879a6", "--surface-0": "#120d12" },
  "layout": { "nav": "top", "density": "comfy" } }
```
- **令牌词表、取值范围、layout 字段、校验规则、拒装原因码** 见 **§15**（机器可读口径 = `src/shared/plugin.ts` 的 `THEME_TOKENS` / `validateThemeContribute`）。
- ★ **v1 不允许任何 CSS**：`tokens` 只能取词表内的令牌名；出现词表外的名字（含 `--nf-*` 一类自造前缀）或夹带 `style` / `.css` → 拒装。
- 内置 4 套皮肤与插件皮肤**同一套契约**（内置皮肤同样只允许一段令牌声明）。

### 5.4 `subtitle.provider` 结构（**L2 声明式字幕源**，2026-10-18 已实装 asset 形）
```jsonc
{ "type": "subtitle.provider",
  "id": "my-subs",                        // [a-z0-9-]{3,40}；不得占用内置源 assrt / subtitlecat
  "name": "我的字幕站",
  "search": {
    "url": "https://sub.example.com/api/search?q={keyword}&ep={ep}",
    "headers": { "Referer": "https://sub.example.com/" },   // 可选
    "list": "data.list",                  // ★ 响应里**数组**所在的点号路径
    "map": { "file": "id", "subname": "filename",          // ★ file / subname 必填
             "title": "video_title", "lang": "language", "format": "ext", "detail": "note" }
  },
  "fetch": { "url": "https://sub.example.com/api/download/{file}", "headers": {} } }
```
- ★ **宿主代发请求，插件不跑代码** → `kind:"asset"`、`permissions` **必须为空**、**不得有 `main`**（这正是 L2 的意义）。
- **v1 能力边界**：仅 `GET`、仅 JSON 响应、**无鉴权**。需要 `POST` / token / 解析 HTML → **只能用 L4**（`kind:"code"`）。
- **占位符白名单**：`search` 用 `{keyword}` `{keywords}` `{title}` `{ep}`；`fetch` 用 `{file}` `{subname}` `{title}` `{ep}`。
  越界（如 `search.url` 写 `{file}`）→ 拒装。URL 里的值经 **`encodeURIComponent`**；请求头的值**原样**替换。
- **`fetch.url` 必须含 `{file}`**（据候选的下载标识定位具体字幕）。
- **JSON 路径语法**：仅 `a.b` / `a.b[0]`（点号 + 下标）；**无 `[*]`、无函数、无过滤**。非法 → 拒装。
- **请求头**：禁 `Host` / `Content-Length` / `Connection` / `Transfer-Encoding`；键值皆禁 CRLF 注入。
- 装上后**自动出现在「字幕设置」的源开关列表**（与内置源同列，可单独关掉）。
- **拒装原因码**：`SUBTITLE_PROVIDER_INVALID`（结构/字段越界）· `SUBTITLE_ID_TAKEN`（占用内置源 id）·
  `BAD_PROVIDER_URL`（协议 / 占位符 / 请求头）· `BAD_PROVIDER_PATH`（JSON 路径语法）。
- **机器可读口径**：`src/shared/plugin.ts` 的 `SubtitleProviderContribute` / `validateSubtitleProviderContribute` /
  `RESERVED_SUBTITLE_IDS` / `expandSubTemplate` / `jsonPathGet`。
- 可直接安装的示例皮肤包：[`skills/winbox-plugin/assets/theme-sakura.winbox`](skills/winbox-plugin/assets/theme-sakura.winbox)
  （打包命令与自测三步见 [`skills/winbox-plugin/references/theme-reference.md`](skills/winbox-plugin/references/theme-reference.md) §6）。


### 5.5 `meta.provider` 结构（**L2 声明式元数据源**，2026-10-18 已实装 asset 形）
```jsonc
{ "type": "meta.provider",
  "id": "my-meta",                        // [a-z0-9-]{3,40}；不得占用内置来源 tmdb / douban
  "name": "我的元数据站",
  "imageBase": "https://img.example.com/t/p/w342",   // 可选：海报/横图为相对路径时的前缀
  "search": {
    "url": "https://api.example.com/search?q={keyword}&year={year}",
    "headers": { "Referer": "https://api.example.com/" },      // 可选
    "list": "data.results",               // 命中项：**数组**（取第一项）或**单个对象**
    "map": { "title": "name", "year": "year", "poster": "poster_path",
             "backdrop": "wide_path", "overview": "overview", "mediaType": "type" } } }
```
- ★ **定位 = 兜底**：宿主链路是 `TMDB（有用户 Key 时）→ 豆瓣 → ★ 插件源`；**内置命中时不走插件**，
  调用方**显式指定来源**（`provider=tmdb|douban`）时也**不走插件**。同理不抢占、也**不写入内置 `meta.json` 查询缓存**
  （避免停用插件后旧缓存仍命中）。
- **v1 能力边界**与字幕源相同：仅 `GET`、仅 JSON 响应、无鉴权（POST / token / HTML → 用 L4）。
- **占位符白名单**：`{keyword}` / `{title}`（等价，均传剧名）/ `{year}`；越界 → 拒装。
- **命中字段**：`title`（必填）/ `year` / `poster` / `backdrop` / `overview` / `mediaType`
  （值含 `tv`/`series`/`剧`/`集` → `tv`，否则 `movie`）。
- ★ **v1 不提供评分**：宿主 UI 当前不展示评分 → 声明 `rating` 等未提供字段**拒不装**（不给假能力）。
- **图片**：绝对 URL 原样用；相对路径拼 `imageBase`；**未声明 `imageBase` 的相对路径图直接丢弃**（绝不产出坏图）。
- **JSON 路径语法 / 请求头限制 / 路径安全**：与 §5.4 同一套口径。
- **拒装原因码**：`META_PROVIDER_INVALID`（结构 / 字段越界）· `META_ID_TAKEN`（占用内置来源 id）·
  `BAD_PROVIDER_URL`（协议 / 占位符 / imageBase / 请求头）· `BAD_PROVIDER_PATH`（JSON 路径语法）。
- **机器可读口径**：`src/shared/plugin.ts` 的 `MetaProviderContribute` / `validateMetaProviderContribute` /
  `RESERVED_META_IDS` / `META_PLACEHOLDERS` / `META_MAP_FIELDS`。

### 5.6 `discover.source` 结构（**L2 声明式发现页数据源**，2026-10-18 已实装 asset 形）
```jsonc
{ "type": "discover.source",
  "id": "my-discover", "name": "我的榜单",
  "sections": [                            // ★ ≥1；顺序 = 发现页上的显示顺序
    { "id": "my-top-movie",                // [a-z0-9-]{3,40}；同一插件内不得重复
      "title": "我的电影热榜",              // 页面上这一行的标题
      "mediaType": "movie",                // movie | tv（缺省 movie）
      "url": "https://api.example.com/top?type={mediaType}&limit=20",
      "headers": { "Referer": "https://api.example.com/" },
      "list": "data.items",                // ★ 响应里**数组**所在的点号路径
      "imageBase": "https://img.example.com/t/p/w342",
      "map": { "title": "name", "year": "year", "poster": "poster_path",
               "backdrop": "wide_path", "mediaType": "type", "tmdbId": "tmdb_id" } } ] }
```
- ★ **定位 = 增补**：插件分区**追加**在发现页内置榜单（TMDB / 豆瓣）**之后**；**内置分区先占 id**（同 id 的插件分区被忽略），
  插件之间重复的分区也会被丢弃；内置榜单**整条失败**时插件分区**照样显示**（增补 ≠ 替代）。
- **v1 能力边界**与前两者相同：仅 `GET`、仅 JSON 响应、无鉴权（POST / token / HTML → 用 L4）。
- **占位符白名单**：**仅 `{mediaType}`**（展开为该 section 声明的 `mediaType`）；越界 → 拒装。
- **`sections`**：≥1；`section.id` 同插件内不得重复；`title` 必填；`mediaType` 可选（缺省 `movie`，
  也是条目未映射 `mediaType` 时的缺省）。
- **字段白名单**：`title`（必填）/ `year` / `poster` / `backdrop` / `mediaType` / `tmdbId`。
  ★ **不含 `doubanId`** —— 该字段会让宿主去取「豆瓣剧照」，插件源并非豆瓣 ⇒ 声明即拒装（避免图源错配）。
- **图片口径**：插件给的 `poster` / `backdrop` **一律由宿主包成本地 `/img` 中继**（与内置榜单同口径：渲染层直连图床可能被 DNS 污染）；
  值若为**相对路径**则先拼 `imageBase`，**未声明 `imageBase` 的相对图直接丢弃**（绝不产出坏图）。
- **空分区不显示**：`list` 取到 0 条（或不是数组）时该分区不出现（不留空行）；单分区失败**只跳过该分区**。
- **JSON 路径语法 / 请求头限制 / 路径安全**：与 §5.4 同一套口径。
- **拒装原因码**：`DISCOVER_PROVIDER_INVALID`（结构 / 字段越界 / section 重名）· `BAD_PROVIDER_URL` · `BAD_PROVIDER_PATH`。
- **机器可读口径**：`src/shared/plugin.ts` 的 `DiscoverSourceContribute` / `DiscoverSectionContribute` /
  `validateDiscoverSourceContribute` / `DISCOVER_PLACEHOLDERS` / `DISCOVER_MAP_FIELDS`。

### 5.7 `danmaku.source` 结构（**L2 声明式弹幕源**，2026-10-18 已实装 asset 形）
> 弹幕是软件既有的**三步链路**：**① 搜番剧 → ② 取剧集列表 → ③ 按剧集 id 拉弹幕 XML**（内置外部接口即此形，见 `main/danmaku/logvarProvider.ts`）。
> 声明式弹幕源把这三步**各声明一段受限模板**，三段之间由宿主用**合成 source 键** `plugin:<contributeId>` 路由。

```jsonc
{ "type": "danmaku.source",
  "id": "my-danmaku",                      // [a-z0-9-]{3,40}
  "name": "我的弹幕站",                     // 候选里标注来源用
  "search":   { "url": "https://dm.example.com/api/search?keyword={keyword}&season={season}",
                "headers": { "Referer": "https://dm.example.com/" },
                "list": "animes",           // ★ 数组路径
                "map": { "animeId": "animeId", "title": "animeTitle",   // animeId / title 必填
                         "bangumiId": "bangumiId", "episodeCount": "episodeCount", "kind": "type" } },
  "episodes": { "url": "https://dm.example.com/api/bangumi/{animeId}",
                "list": "bangumi.episodes",
                "map": { "episodeId": "episodeId",                      // 必填
                         "episodeTitle": "episodeTitle", "episodeNumber": "episodeNumber" } },
  "comments": { "url": "https://dm.example.com/api/comment/{episodeId}?format=xml" } }
```
- ★ **三段缺一不可**（缺任一段即拒装）：没有 `episodes` 就无法从番剧走到剧集，没有 `comments` 就拿不到弹幕。
- ★ **`comments` 只收 XML 文本**（B 站弹幕格式，由宿主引擎解析器消费）——**没有 `list` / `map`**；
  只给 JSON 弹幕的接口本期做不了（走 L4）。
- **占位符按段各一套白名单**：`search` 用 `{keyword}` / `{season}`；`episodes` 用 `{animeId}` / `{title}`；`comments` 用 `{episodeId}`。
  `comments.url` **必须含 `{episodeId}`**；越界 → 拒装。
- **字段映射**：`search.map` 取 `animeId`（必填）/ `title`（必填）/ `bangumiId`（缺省用 `animeId`）/ `episodeCount` / `kind`；
  `episodes.map` 取 `episodeId`（必填）/ `episodeTitle` / `episodeNumber`（1~4 位数字串）。
- ★ **生效位置 = 与内置外部接口并列**：插件弹幕源一起并行检索、一起按匹配度排序、**共用同一套失败冷却**（失败只影响该源）。
  ★ v1 **不进入**播放器「弹幕源」面板的接口开关清单 —— 要停用请在 **配置 → 插件** 停用该插件。
- **JSON 路径语法 / 请求头限制 / 路径安全**：与 §5.4 同一套口径。
- **拒装原因码**：`DANMAKU_PROVIDER_INVALID`（结构 / 段缺失 / 字段越界）· `BAD_PROVIDER_URL` · `BAD_PROVIDER_PATH`。
- **机器可读口径**：`src/shared/plugin.ts` 的 `DanmakuSourceContribute` / `DanmakuSearchSpec` / `DanmakuEpisodesSpec` /
  `validateDanmakuSourceContribute` / `DANMAKU_SEARCH_PLACEHOLDERS` / `DANMAKU_EPISODES_PLACEHOLDERS` /
  `DANMAKU_COMMENTS_PLACEHOLDERS` / `DANMAKU_ANIME_MAP_FIELDS` / `DANMAKU_EPISODE_MAP_FIELDS`。
---

## 6. 权限：`permissions`（**只属 L4 / `kind:"code"`**）

> ★ 本节**只对 `kind:"code"` 有意义**。L1/L2/L3（`kind:"asset"`）**必须** `permissions: []`。
> `permissions` 管的是**你的沙箱代码能调用哪些宿主 API**（"注册槽位"是 `contributes` 的事，不在这里）。
> ★ `kind:"code"` **必须**有 `main`（沙箱入口文件相对名，如 `main.js`）—— 宿主在沙箱 worker 里加载并执行它。

### 6.1 白名单
| 权限 | 授予 | 安全级 | 宿主注入 API（契约，M2 实装） |
|---|---|---|---|
| `net` | 经**宿主代理**的 HTTP（可带 UA/Referer/自定义头） | 中 | `winbox.http(url, opts) → {status, headers, text()}` |
| `net.download` | 下载（落插件目录，走加速池/代理） | 中 | `winbox.download(url, name, onProgress)` |
| `kv` | **命名空间隔离**的 KV | 低 | `winbox.kv.get/set/delete`（键前缀 `plugin:<id>:`） |
| `settings` | 读写**插件自身**设置 | 低 | `winbox.settings.get/set`（`plugins/<id>.winbox/settings.json`） |
| `fs.plugin` | **仅**插件自身目录读写 | 中 | `winbox.fs.*`（根 = `plugins/<id>.winbox/`，越权即拒） |
| `log` | 写日志 | 低 | `winbox.log.i/w/e`（前缀 `plugin:<id>`） |
| `notify` | 系统通知 | 低 | `winbox.notify(title, body)` |
| `clipboard.read` | 读剪贴板 | 中 | `winbox.clipboard.read()` |
| `clipboard.write` | 写剪贴板 | 低 | `winbox.clipboard.write(text)` |
| `dialog` | 原生消息框（**仅展示**） | 低 | `winbox.dialog.alert/confirm` |
| `player.control` | 播放控制（play/pause/seek/next/音轨/字幕/音量） | 中 | `winbox.player.*`（经宿主接口，不碰内核） |
| `openExternal` | 打开外部链接/文件 | 中 | `winbox.openExternal(url)`（仅 `http/https`） |

> 建议同时填 `permissionNotes`（每个权限一句话用途）—— **安装时会展示给用户**，便于授权透明。
> （`ui.slot` / `ui.command` **不在本表**：它们是 `contributes`（注册什么槽位），与权限无关。）

### 6.2 禁止项（**永不提供**，声明即拒装）
- `main`（主进程执行 / 任意 JS 注入）、`spawn`（子进程 / `child_process`）、`eval`
- `fs`（**任意**文件系统；受限的 `fs.plugin` 才允许）
- `cred`（读写凭据 / DPAPI / `drive-tokens`）
- `proxy`（`9978` 代理 / **直连 socket 绕过宿主**，含 `raw.net`）
- `registry`（注册表）

> ★ **L4 是权限天花板**：宿主不提供"在主进程执行第三方代码"的任何能力。沙箱内**没有** `require`/`process`/`fs`/`child_process`，只有上表显式授权的 API。

---

## 7. 运行库：`runtimeDeps`（**安装之后**才下载）

- 用途：插件的**重运行库**（如 ncnn、CPython 型依赖），**不随安装包**，在插件**安装成功后**由宿主按需下载。
- 你**只能引用**宿主**内置许可清单**里的 `dep id`（清单在宿主源码 `shared/pluginDeps.ts`，含 url/sha256/版本）：
```jsonc
"runtimeDeps": [ { "id": "anime4k-shaders", "files": ["Anime4K_Upscale_CNN_x2_M.glsl"] } ]
```
- **不得自报 url/sha256**（限"内部许可"内容，防供应链风险）。需要收录新的运行库 → 向项目提 **PR** 把 dep 登记进清单（带 url/sha256/许可）。
- 失败时插件停在 `runtime-failed`，宿主给「重试」；**绝不静默降级**。
- ★ 若你的**资源本身很小**（如着色器 <1MB 合计），建议**直接打进 `files/` 随包**，不必走 `runtimeDeps`。

---

## 8. 在线更新：`updateCheckUrl`

- 由**作者自配**（宿主不代管、不托管）：指向你自己的 GitHub 仓库，两种形态皆可：
  - `https://api.github.com/repos/<owner>/<repo>/releases/latest`（读 `tag_name`）；或
  - 你自建的 `https://…/latest.json`（返回 `{ "version": "1.1.0" }`）。
- 宿主检测：远端 `version` > 本地 → 管理器显示**「可更新」**；
- **仅提示，不自动下载**：更新由用户手动完成（下载新 `.winbox` 后重新安装）；
- 检测走**加速池 + 代理链**，失败**只告警、不锁死软件**。

---

## 9. 生命周期（对作者的约定）

| 阶段 | 你的插件该是怎样的 |
|---|---|
| 安装 | 解包 → 校验 → 原子落盘 `<id>.winbox/`；被拒时宿主给 `reasonCode`（见 §10） |
| 拉运行库 | 安装成功后才拉 `runtimeDeps`；失败可重试 |
| 启用 / 停用 | 停用保留磁盘，仅不参与装配；**你的插件应可随时被停用而不报错** |
| 更新 | 仅提示；用户重装新 `.winbox`（宿主原子替换，失败回滚旧版） |
| 移除 | 删目录（含 `.runtime`）；若你的贡献正生效，宿主会**回落默认**（如超分档位 → off） |

---

## 10. 发布前校验清单（自检）

- [ ] 根级有 `plugin.json`，**没有多套目录层**；
- [ ] `id` 符合 `[a-z0-9-]{3,40}` 且与他人不冲突；
- [ ] `apiVersion` 与目标宿主大版本一致；
- [ ] `kind` 与 `permissions` 匹配（asset → 空；code → 白名单内且**说明用途**）；
- [ ] `contributes[].type` 在宿主白名单内；
- [ ] 所有 `files`/`shaders` 是**相对名**，无 `..`、无绝对路径；
- [ ] `modes[].shaders` ⊆ 实际提供的文件；
- [ ] `runtimeDeps` 只引用**内置清单**的 id（未自报 url）；
- [ ] `updateCheckUrl` 是 `https://` 且指向你自己的仓库；
- [ ] 附 `LICENSE`；第三方资源**许可兼容**并署名（见 §12）。

**常见拒装原因码**：`MISSING_MANIFEST` · `ID_INVALID` · `API_VERSION_MISMATCH` · `KIND_PERM_MISMATCH` · `UNKNOWN_CONTRIBUTE` · `PATH_ESCAPE` · `SHA_MISMATCH` · `INCOMPLETE` · `DEP_NOT_ALLOWED`。

---

## 11. 打包与命名（速查）

- 打包：`zip` → 改后缀 `.winbox`；文件名 = `<id>.winbox`；
- 安装位置（宿主）：官方内置插件 `resources/plugins/<id>.winbox`（**随包**，只读，源标记 `builtin`）；第三方插件 `<userData>/plugins/<id>.winbox/`；
- **不要**打包 `node_modules`、`.git`、缓存、测试夹具。

---

## 12. 许可、署名与合规

- 你**必须**在包内附 `LICENSE`，并在 `plugin.json` 填 `license`/`author`/`homepage`；
- 引用第三方资源（如 Anime4K、RAVU、FSRCNNX）**必须**遵守其各自许可并署名；**打包分发前自行确认许可允许再分发**；
- 插件**不得**内置任何影视内容、破解凭据或侵权资源。

---

## 13. FAQ

- **Q：我的插件没生效？** A：先看配置页「组件/插件」里状态是否为 `ready`；未装运行库会停在 `runtime-failed`。
- **Q：`mpv.shaders` 在 HTML5 内核下没反应？** A：超分**仅 MPV 内核**生效；宿主**只提示、不自动切内核**。
- **Q：能读主进程凭据 / 自己发网络请求吗？** A：不能。网络只能经 `net` 权限走宿主代理；凭据一律不可读写。
- **Q：写出绝对路径给 mpv 行不行？** A：不行，只写相对名；绝对路径会被校验拒绝（`PATH_ESCAPE`）。
- **Q：L4 代码插件能做什么？** A：见 §6 白名单；想加新能力请向项目提 contract 变更（宿主 API 走 semver）。

---

## 14. 版本与兼容

- 宿主 API 用 `apiVersion` **语义化版本**；**大版本不匹配即拒装**，小版本向后兼容；
- 新增 `contributes.type` / `permission` 属**小版本**增量（旧插件不受影响）；
- 破坏性变更**只在大版本**发生，并提前在发布说明公示。

---

---

## 15. 皮肤与主题规范（`contributes[].type = "theme"`）

> 适用范围：**内置皮肤**与**插件皮肤**共用同一套契约。
> 插件侧 `contributes.theme` 的字段/白名单归 `SPEC.md` §5.3（本节定义它的**令牌词表与校验规则**）。
> 机器可读口径：`src/shared/plugin.ts`（`THEME_TOKENS` / `validateThemeContribute`）。
>
> ★ 本规范解决的问题：皮肤长期是「**整份 CSS + `:root[data-theme='X']` 前缀 + 重写元素级选择器 + 自造私有令牌**」，
> 导致改一处 UI 要同步 4 份皮肤、且反复踩特异度坑（`.tag` 被 `button` 通配压掉、`.vp-ctl` 被撑成 54×38、`.pill-switch` 看不出选中态）。

---

### 15.1 四条红线

1. **皮肤只能覆盖令牌**，不得写任何 CSS 选择器（v1）。皮肤 = 一份**令牌字典** + 一份**布局声明**。
2. **禁止自造令牌名**（`--nf-*` / `--b-*` / `--ap-*` / `--db-*` 一律并入统一词表）；未知令牌名 → 拒。
3. **禁止按皮肤 id 分支布局**（`isNetflix()` / `useTheme() === 'douban'`）；布局差异一律走 `layout` 声明。
4. **禁止元素级 / 高特异度选择器**（`button`、`input`、`::selection`、`:root[data-theme='X'] button`）。

> 例外口子（v2 再议）：若确需「隐藏侧边栏」这类结构改造，走 `layout.nav` 等声明项；仍不够时另开受限样式表白名单（强制 `:root[data-theme='plugin:<id>']` 前缀 + 只允许宿主 class 词表）。

---

### 15.2 令牌词表 v1

三列含义：**令牌** = 唯一合法名；**取代** = 现行散落的旧名（迁移时全部收敛到这里）；值类型/范围由校验器强制。

#### 15.2.1 表面与层级

| 令牌 | 语义 | 类型 | 范围 | 基线值 | 取代 |
|---|---|---|---|---|---|
| `--surface-0` | 应用底色（最底） | color | — | `#0a0c10` | `--bg0`、`--db-bg`、`--b-page` |
| `--surface-1` | 主面板底 | color | — | `rgba(15,18,24,.85)` | `--bg1` |
| `--surface-2` | 抬升面板 | color | — | `rgba(20,24,31,.82)` | `--bg-elev`、`--nf-surface` |
| `--surface-3` | 浮层 / 更高抬升 | color | — | `rgba(27,32,42,.82)` | `--bg-elev2`、`--nf-surface-2` |
| `--surface-glass` | 玻璃层（窗口底，须含透明） | color | 需 alpha | `rgba(9,11,15,.9)` | `--bg-glass` |

#### 15.2.2 品牌与强调

| 令牌 | 语义 | 类型 | 范围 | 基线 | 取代 |
|---|---|---|---|---|---|
| `--accent` | 主色 | color | — | `#5b8cff` | `--nf-red`、`--b-blue`、`--ap-blue`、`--db-orange` |
| `--accent-strong` | 主色悬停/按下 | color | — | = `--accent` | `--nf-red-hover`、`--b-blue-hover`、`--ap-blue-hover`、`--accent-strong` |
| `--accent-container` | 主色容器底（选中态填充） | color | — | = `--accent` | `--accent-container` |
| `--accent-soft` | 主色淡底（hover / 焦点环，须含透明） | color | 需 alpha | `rgba(91,140,255,.14)` | `--accent-soft` |
| `--on-accent` | **主色之上的前景色** | color | — | `#fff` | （新；网飝的白底按钮用 `#141414`） |
| `--accent-2` | 副色 | color | — | `#63e6be` | `--accent-2`、`--nf-teal` |

#### 15.2.3 文本

| 令牌 | 语义 | 类型 | 基线 | 取代 |
|---|---|---|---|---|
| `--text` | 主文本 | color | `#e8ecf4` | `--text` |
| `--text-dim` | 次文本 | color | `#8a93a6` | `--nf-dim`、`--b-dim`、`--ap-dim`、`--b-text2` |
| `--text-faint` | 三级 / 禁用文本 | color | = `--text-dim` | （新） |

#### 15.2.4 分隔与语义色

| 令牌 | 语义 | 类型 | 基线 | 取代 |
|---|---|---|---|---|
| `--border` | 边框 | color | `#262c3b` | `--b-line` |
| `--border-soft` | 弱边框（须含透明） | color | `rgba(38,44,59,.5)` | `--border-soft` |
| `--hairline` | 1px 分隔 / 高光 | color | = `--border-soft` | `--ap-hairline` |
| `--danger` | 危险 | color | `#ff6b6b` | `--ap-red` |
| `--warn` | 警告 | color | `#ffc25e` | `--ap-yellow` |
| `--success` | 成功 | color | `#63e6be` | `--nf-green` |
| `--info` | 信息 | color | = `--accent` | `--db-navy`、`--accent-2` |

#### 15.2.5 形状与光影

| 令牌 | 语义 | 类型 | 范围 | 基线 | 取代 |
|---|---|---|---|---|---|
| `--radius` | 基础圆角 | length | 0–24px | `12px` | — |
| `--radius-sm` | 小圆角 | length | 0–16px | `8px` | — |
| `--radius-lg` | 大圆角（卡片 / 面板） | length | 0–32px | `16px` | （新） |
| `--shadow` | 基础阴影 | shadow | 受限语法 | `0 8px 24px rgba(0,0,0,.35)` | — |
| `--shadow-lg` | 抬升阴影 | shadow | 受限语法 | = `--shadow` | `--db-card-shadow` |

#### 15.2.6 动效（全皮肤共用节奏，皮肤只可微调）

| 令牌 | 语义 | 类型 | 范围 | 取代 |
|---|---|---|---|---|
| `--ease-standard` | 状态切换（M3 Standard） | easing | `cubic-bezier(a,b,c,d)` | `--nf-ease`、`--b-ease`、`--ap-ease`、`--db-ease` |
| `--ease-enter` | 进场（Emphasized decelerate） | easing | 同上 | `--nf-enter`、`--b-enter`、`--ap-enter`、`--nf-ease` |
| `--ease-exit` | 出场（Emphasized accelerate） | easing | 同上 | `--db-ease-out` |
| `--ease-spring` | 按压/悬停反馈（Expressive 快档） | easing | 同上 | `--nf-spring`、`--b-spring`、`--ap-spring` |
| `--dur-instant` | 输入确认（按压 / 开关） | time | 40–200ms | — |
| `--dur-quick` | 微交互（颜色 / 边框） | time | 40–300ms | — |
| `--dur-state` | 状态变化（悬停 / 选中 / 浮层） | time | 80–400ms | — |
| `--dur-panel` | 面板 / 对话框 | time | 120–600ms | `--nf-dur-m`、`--b-dur-m`、`--ap-dur-m`、`--db-dur-m` |
| `--dur-page` | 页面进场 | time | 120–800ms | `--db-dur-l` |

#### 15.2.7 字级

| 令牌 | 语义 | 范围 | 基线 |
|---|---|---|---|
| `--fs-xs` / `--fs-sm` / `--fs-md` / `--fs-lg` / `--fs-xl` / `--fs-2xl` | 排版级差 | 9–34px | 11 / 12 / 13.5 / 15 / 18 / 22 |

---

### 15.3 `layout` 声明（取代组件里的皮肤 id 分支）

| 字段 | 取值 | 效果 | 取代 |
|---|---|---|---|
| **`shell`** | `topbar` \| `dualbar` \| `sidebar` \| `pillbar` | 外壳：单行顶栏 / 双行头 / 玻璃侧边栏 / 胶囊导航+浮动圆钮 | `App.tsx` 的 `douban`/`apple`/`bilibili` 三分支 + 默认分支 |
| `nav` | `side` \| `top` | 侧边栏 / 顶部导航 | `isNetflix()`、`netflix.css` 里 `display:none` 侧栏 |
| `density` | `compact` \| `comfy` | 行高与内边距整体缩放 | — |
| `radiusScale` | `0.5` \| `1` \| `1.5` | 全局圆角倍率（作用于三个 radius 令牌） | — |
| `navHeight` | 40–72px | 顶栏高 | `--nf-nav-h`、`--ap-toolbar-h` |
| `sideWidth` | 160–280px | 侧栏宽 | `--side-w` |
| `cardWidth` | 120–260px | 卡片宽 | `--nf-card-w` |
| `motion` | `full` \| `reduced` | 动效强度（与系统 `prefers-reduced-motion` 取更弱者） | （新） |
| **`detailStyle`** | `cinema` \| `glass` \| `poster` | 详情页排版：影院大图 / 玻璃信息卡 / 左海报右信息（**缺省 = 基础排版**） | `DetailPage.tsx` 的 `nf`/`ap`/`db` 分支 |
| **`discoverStyle`** | `shelf` \| `shelfWide` \| `wall` | 发现页：内容行 / 宽卡内容行 / 海报墙（**缺省 = 基础行**） | `DiscoverPage.tsx` 的 `nf`/`ap`/`db` 分支 |
| **`filterStyle`** | `plain` \| `card` | 筛选条：朴素 / 卡片容器（缺省 = `plain`） | `CascadeBar.tsx` 的 `db` 分支 |

> ★ 加粗项都是**结构枚举**：组件按「声明值」分支，而不是按皮肤 id。
> 这些结构样式**由宿主提供**（`global.css` 的「宿主结构层」）：宿主把 layout 声明写成 `<html>` 上的**宿主属性**，
> 结构规则按属性作用域（**不按皮肤 id**）——
>
> | 宿主属性 | 来源 | 用它的结构规则 |
> |---|---|---|
> | `data-theme` | 皮肤 id（含 `plugin:<id>`） | 只给「令牌块」用（皮肤文件的 `:root[data-theme=…]`） |
> | `data-base` | `theme.base` | 选**基线**：`:root`（深色基线）/ `:root[data-base='light']`（亮色基线） |
> | `data-shell` | `layout.shell`（缺省 `topbar`） | 外壳结构（各外壳的顶栏 / 侧栏 / 导航条 / 页脚…） |
> | `data-detail` | `layout.detailStyle`（缺省 `basic`） | 详情页排版 |
> | `data-discover` | `layout.discoverStyle`（缺省 `basic`） | 发现页排版 |
> | `data-filter` | `layout.filterStyle`（缺省 `plain`） | 筛选条形态 |
> | `--nav-h` / `--side-w` / `--card-w` | `navHeight` / `sideWidth` / `cardWidth` | 布局度量（宿主自用变量，**不在皮肤词表内**；缺省 54 / 208 / 132px） |
>
> ⇒ 插件皮肤声明既有枚举即可复用宿主排版；要新增一种排版 = 先在组件里加枚举分支（宿主能力扩展）。
> **插件皮肤无法越过宿主既有枚举** —— 这是「v1 不允许任意 CSS」的配套约束。
> ⚠️ 落地顺序：结构类必须**先搬进 `global.css`**，插件皮肤声明枚举才有样式可用（见 §15.4 与 §15.7 验收）。

组件只读 **令牌 + layout 声明**，不得再读皮肤 id。

---

### 15.4 皮肤形态与装配

#### 15.4.1 内置皮肤
- 文件：`renderer/styles/<skin>.css`，**只允许存在一段** `:root[data-theme='<skin>'] { <令牌>: <值>; … }`。
- 其余结构样式（布局、滚动条、组件骨架）**全部回到 `global.css`**，宿主把它们按 `data-shell` / `data-detail` /
  `data-discover` / `data-filter` / `data-base` 作用域（见 §15.3 的属性表），由令牌与 layout 声明驱动。
- 迁移后每个皮肤文件的预期体量：**≤ 60 行**（现状 616–1114 行 → 迁移后 36–39 行）。
- 静态自检（可脚本化）：皮肤文件里出现 **任何选择器**（除那一段 `:root[data-theme=…]`）、`--nf-`/`--b-`/`--ap-`/`--db-` 前缀、
  或词表外的自定义属性 → 视为不合规。

#### 15.4.2 插件皮肤
```jsonc
{ "type": "theme",
  "id": "sakura",                 // [a-z0-9-]{3,40}；生效值 = data-theme='plugin:sakura'
  "label": "🌸 樱花",              // 设置页显示名
  "base": "dark",                 // dark | light（决定 nativeTheme 与未覆盖令牌的基线）
  "tokens": { "--accent": "#e879a6", "--surface-0": "#120d12" },
  "layout": { "nav": "top", "density": "comfy" } }
```
- 装配：`PluginHost` 读 `contributes.theme` → IPC 下发 → 渲染层把 `tokens` 写成 `:root[data-theme='plugin:<id>']` 的 CSS 变量（运行时注入 `<style>`）。
- `base` 决定回落基线的取哪一份（深色基线 / 亮色基线），未覆盖的令牌不回落到别的皮肤。

#### 15.4.3 皮肤注册表
`renderer/lib/themeTokens.ts` 的 `Theme` **从写死的联合类型改为动态注册表**：
```ts
interface SkinEntry { id: string; label: string; kind: 'builtin' | 'plugin'; base: 'dark' | 'light'; layout: SkinLayout }
```
- `normalizeTheme()` = 查注册表，未命中回落默认皮肤；
- 皮肤被停用/卸载 → 当前选择回落默认皮肤并广播 `winbox:theme-changed`；
- 新增皮肤**不改任何组件代码**（这是验收口径）。

---

### 15.5 校验规则（拒装原因码）

| 原因码 | 触发 |
|---|---|
| `THEME_TOKENS_REQUIRED` | `contributes.theme` 缺 `tokens` / `id` / `label` |
| `UNKNOWN_THEME_TOKEN` | `tokens` 出现词表外的令牌名（含自造 `--nf-*` 类） |
| `BAD_THEME_TOKEN_VALUE` | 值类型或范围不符（颜色/长度/时长/阴影/easing） |
| `THEME_STYLE_NOT_ALLOWED` | v1 下在 `tokens` 外夹带样式（`style` / `css` 字段或 `.css` 文件） |
| `BAD_THEME_LAYOUT` | `layout` 字段名或枚举值非法 |
| `THEME_ID_TAKEN` | 皮肤 id 与内置皮肤或已装插件冲突 |

**静态自检（内置皮肤同口径）**：皮肤 CSS 里出现 `--nf-`/`--b-`/`--ap-`/`--db-` 前缀、或出现元素级选择器（`button`/`input`/`::selection`）、或出现未在词表内的自定义属性 → 视为不合规。

---

### 15.6 迁移映射（4 套内置皮肤）

| 皮肤 | 需并入统一词表的私有令牌 | 需收敛的硬编码颜色（实测） |
|---|---|---|
| `netflix` | 12 个：`--nf-dim/red/red-hover/green/teal/surface/surface-2/card-w/nav-h/ease/spring/enter/dur-m` | **232 处** |
| `bilibili` | 11 个：`--b-blue/blue-hover/dim/line/page/pink/text2/ease/spring/enter/dur-m` | 待实测 |
| `apple` | 16 个：`--ap-blue/blue-hover/dim/fill/fill-hover/gloss/green/hairline/indigo/red/spring/yellow/toolbar-h/ease/enter/dur-m` | 待实测 |
| `douban` | 11 个：`--db-bg/navy/navy-soft/orange/card-shadow/ease/ease-out/dur-m/dur-l` + `--accent-container/strong` | **460 处** |

其中 `--nf-card-w` / `--nf-nav-h` / `--ap-toolbar-h` / `--side-w` 属**布局度量**，迁到 `layout` 声明（不留在令牌里）。

---

### 15.7 验收口径

1. **静态**：4 套内置皮肤 + 插件皮肤样例均满足 §15.5 的自检（脚本扫描，0 违规）。
2. **动态（CDP 探针，逐皮肤）**：切换皮肤后断言关键盒子与状态——控制条图标按钮 26×26、控制条占宽一致、`.tag` 选中态可见、浮层不遮挡（沿用既有 `.tmp/player-ui-verify.cjs` 口径并扩充到 4 皮肤）。
3. **回归**：双 `tsc` 0 错误；`vitest` 全绿（含新增 `themeTokens` / `validateThemeContribute` 用例）。
4. **插件链路**：装一个 `theme` 插件 → 设置页出现该皮肤 → 选中即时生效 → 卸载回落默认皮肤，全程无组件代码改动。

---

### 15.8 明确不做（v2 才议）

- 受限样式表白名单（结构改造能力）；
- 皮肤继承/叠加（`extends`）；
- 插件皮肤覆盖 `--fs-*` 之外的自定义字体族（涉及跨平台字体回退，单独立项）。
