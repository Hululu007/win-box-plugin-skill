# Win-Box 插件 SKILL

Win-Box 插件的**开发套件**：一份**插件制作规范**（含**皮肤与主题规范**）+ 一个给 AI 用的**开发技能（AI Agent skill）**。

Win-Box 是一个基于 Electron 的 Windows 桌面影视播放器，支持通过 **`.winbox` 插件**扩展功能。

## 内容

| 路径 | 说明 |
|---|---|
| [**`SPEC.md`**](SPEC.md) | **插件制作规范**（唯一权威）：能力分级 L1~L4、`.winbox` 包格式、`plugin.json` 字段与校验、`contributes` / `permissions` 白名单、运行库与在线更新、发布前校验清单；**§5.4 = 声明式字幕源（L2）**、**§5.5 = 声明式元数据源（L2）**、**§5.6 = 声明式发现页数据源（L2）**、**§5.7 = 声明式弹幕源（L2，三步链路）**、**§5.8 = 声明式直播输出模板（L2，★ 不发请求）**、**§5.9 = 受限动作按钮（L3，动作词表）**、**§5.10 = 追加快捷键（L3）**、**§5.11 = 插件设置面板（L3，受限词汇表 4 类字段）**、**§5.12 = 受限 UI 槽位（L3，6 类受限组件）**；**§15 = 皮肤与主题规范**（39 项令牌词表、`layout` 结构枚举、校验规则与拒装原因码 —— 内置皮肤与插件皮肤共用同一套契约，v1 **不允许任何 CSS**） |
| [**`skills/winbox-plugin/`**](skills/winbox-plugin/SKILL.md) | **AI Agent skill**：让 AI 直接按规范帮你**写 / 校验 / 打包**插件（宿主速览、字段速查、皮肤令牌参考、字幕/元数据/发现页/弹幕源/直播输出模板/受限动作按钮/追加快捷键/设置面板/受限 UI 槽位参考、校验工作流、`.winbox` 骨架与可安装的示例皮肤包） |

## 快速开始

1. 先读 [**`SPEC.md`**](SPEC.md) 了解规范（做皮肤看 **§15**；做字幕源看 **§5.4**；做元数据源看 **§5.5**；做发现页数据源看 **§5.6**；做弹幕源看 **§5.7**；做直播输出模板看 **§5.8**；做动作按钮看 **§5.9**；做追加快捷键看 **§5.10**；做设置面板看 **§5.11**；做界面槽位看 **§5.12**）；
2. 要做插件时，让 AI 加载 [`skills/winbox-plugin/`](skills/winbox-plugin/SKILL.md) 这个 skill —— 它会按规范生成 `plugin.json`、组织资源，并打包成 `.winbox`；
3. 想直接看皮肤效果：安装 [`theme-sakura.winbox`](skills/winbox-plugin/assets/theme-sakura.winbox)（纯令牌 + 布局声明的示例，装完在设置页「外观主题」里选中即可）。

## 目录

```
.
├── SPEC.md                              # 插件制作规范（唯一权威；§5.4 字幕源 / §5.5 元数据源 / §5.6 发现页源 / §5.7 弹幕源 / §5.8 直播输出模板 / §5.9 动作按钮 / §5.10 追加快捷键 / §5.11 设置面板 / §5.12 界面槽位 / §15 皮肤）
├── skills/winbox-plugin/
│   ├── SKILL.md                         # AI Agent skill 入口
│   ├── references/
│   │   ├── host-overview.md             # 宿主速览（冷启动必读）
│   │   ├── manifest-reference.md        # plugin.json 字段与白名单速查
│   │   ├── theme-reference.md           # 皮肤参考（令牌词表 / 布局枚举 / 示例包 / 自测三步）
│   │   ├── subtitle-provider-reference.md  # 字幕源参考（URL 模板 + JSON 路径 / 占位符 / 自检）
│   │   ├── meta-provider-reference.md   # 元数据源参考（兜底定位 / imageBase / 自检）
│   │   ├── discover-source-reference.md # 发现页数据源参考（分区结构 / 增补定位 / 图片中继 / 自检）
│   │   ├── danmaku-source-reference.md  # 弹幕源参考（三步链路 / 按段占位符 / 只收 XML / 自检）
│   │   ├── live-template-reference.md   # 直播输出模板参考（输出侧格式化 / 占位符 / 渲染规则 / ★ 不发请求）
│   │   ├── ui-command-reference.md      # 受限动作按钮参考（动作词表 / 挂载点 / 判定路径 / 自检）
│   │   ├── keymap-reference.md          # 追加快捷键参考（键位护栏 / 作用域 / 复用动作词表 / 自检）
│   │   ├── settings-panel-reference.md  # 插件设置面板参考（受限词汇表 4 类 / 值持久化 / 自检）
│   │   ├── ui-slot-reference.md         # 受限 UI 槽位参考（6 类受限组件 / 槽位表 / 复用动作词表 / 自检）
│   │   └── checks.md                    # 发布前校验清单（含皮肤 / 字幕源 / 元数据源 / 发现页源 / 弹幕源 / 直播输出模板 / 动作按钮 / 快捷键 / 设置面板 / 界面槽位专项自检）
│   └── assets/
│       ├── plugin.asset.json            # 声明式插件骨架
│       ├── plugin.code.json             # 代码插件骨架
│       ├── plugin.theme.json            # 皮肤插件骨架（39 项令牌给全）
│       ├── plugin.subtitle-provider.json # 声明式字幕源骨架
│       ├── plugin.meta-provider.json    # 声明式元数据源骨架
│       ├── plugin.discover-source.json  # 声明式发现页数据源骨架
│       ├── plugin.danmaku-source.json   # 声明式弹幕源骨架（三步）
│       ├── plugin.live-template.json    # 声明式直播输出模板骨架（txt + m3u 两套）
│       ├── plugin.ui-command.json       # 受限动作按钮骨架（覆盖全部动作词表）
│       ├── plugin.keymap.json           # 追加快捷键骨架（覆盖全部动作词表）
│       ├── plugin.settings-panel.json   # 插件设置面板骨架（4 类受限字段）
│       ├── plugin.ui-slot.json          # 受限 UI 槽位骨架（6 类受限组件）
│       ├── theme-sakura.winbox          # 示例皮肤包（可直接安装）
│       └── example-super-resolution.plugin.json  # 完整示例
└── LICENSE
```

## Win-Box 主仓库

[yagami-hue/Win-box](https://github.com/yagami-hue/Win-box)

## 许可

MIT（见 [LICENSE](LICENSE)）。
