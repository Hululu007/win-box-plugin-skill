# Win-Box 插件 SKILL

Win-Box 插件的**开发套件**：一份**插件制作规范** + 一个给 AI 用的**开发技能（AI Agent skill）**。

Win-Box 是一个基于 Electron 的 Windows 桌面影视播放器，支持通过 **`.winbox` 插件**扩展功能。

## 内容

| 路径 | 说明 |
|---|---|
| [**`SPEC.md`**](SPEC.md) | **插件制作规范**（唯一权威）：能力分级 L1~L4、`.winbox` 包格式、`plugin.json` 字段与校验、`contributes` / `permissions` 白名单、运行库与在线更新、发布前校验清单 |
| [**`skills/winbox-plugin/`**](skills/winbox-plugin/SKILL.md) | **AI Agent skill**：让 AI 直接按规范帮你**写 / 校验 / 打包**插件（字段速查、校验工作流、`.winbox` 骨架与完整示例） |

## 快速开始

1. 先读 [**`SPEC.md`**](SPEC.md) 了解规范；
2. 要做插件时，让 AI 加载 [`skills/winbox-plugin/`](skills/winbox-plugin/SKILL.md) 这个 skill —— 它会按规范生成 `plugin.json`、组织资源，并打包成 `.winbox`。

## 目录

```
.
├── SPEC.md                              # 插件制作规范（唯一权威）
├── skills/winbox-plugin/
│   ├── SKILL.md                         # AI Agent skill 入口
│   ├── references/
│   │   ├── manifest-reference.md        # plugin.json 字段与白名单速查
│   │   └── checks.md                    # 发布前校验清单
│   └── assets/
│       ├── plugin.asset.json            # 声明式插件骨架
│       ├── plugin.code.json             # 代码插件骨架
│       └── example-super-resolution.plugin.json  # 完整示例
└── LICENSE
```

## Win-Box 主仓库

[yagami-hue/Win-box](https://github.com/yagami-hue/Win-box)

## 许可

MIT（见 [LICENSE](LICENSE)）。
