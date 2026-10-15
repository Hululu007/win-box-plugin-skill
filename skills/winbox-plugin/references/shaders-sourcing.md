# 着色器资源从哪来（官方超分插件的溯源）

`mpv.shaders` 插件的资源是**上游第三方着色器**（不是自己写的）。这里记录官方超分插件 `super-resolution` 的**上游来源 / 许可 / 真实文件名**，其余插件照此格式自行溯源并在 `licenses/` 归档许可原文。

## 1. 三套上游
| 档位 | 上游项目 | 获取物 | 我方包内路径 | 许可 |
|---|---|---|---|---|
| 效率档 / 质量档 | [bloc97/Anime4K](https://github.com/bloc97/Anime4K)（release **v4.0.1** 的 `Anime4K_v4.0.zip`，解包后 39 个 `.glsl`） | 从解包里**挑出用到的 13 个** `.glsl` | `files/Anime4K_*.glsl` | **MIT** |
| 通用档 | [bjin/mpv-prescalers](https://github.com/bjin/mpv-prescalers)（master） | `ravu-lite-r4.hook`（另有 `-ar-` 变体，见下） | `files/ravu-lite-r4.hook` | **LGPL-3.0** |
| 高质量档 | [igv/FSRCNN-TensorFlow](https://github.com/igv/FSRCNN-TensorFlow)（release 1.1） | `FSRCNNX_x2_8-0-4-1.glsl` | `files/FSRCNNX_x2_8-0-4-1.glsl` | **GPL-3.0** |

> ★ 许可原文必须随包：`licenses/LICENSE-anime4k.txt` / `LICENSE-ravu-lite.txt` / `LICENSE-fsrcnnx.txt`，根 `LICENSE` 写插件自身许可（MIT）+ 上游对照段。GPL-3 / LGPL-3 是 copyleft：**必须**连许可原文与上游地址一起分发（着色器文件本身即其"源码形态"）。

## 2. 档位 → 文件（顺序即生效顺序；**以 `files/` 里真实存在的文件名为准**）
- **效率档**（Anime4K Lite，7 条）：
  `Anime4K_Clamp_Highlights → Anime4K_Restore_CNN_M → Anime4K_Restore_CNN_S → Anime4K_Upscale_CNN_x2_M → Anime4K_AutoDownscalePre_x2 → Anime4K_AutoDownscalePre_x4 → Anime4K_Upscale_CNN_x2_S`
- **质量档**（Anime4K 完整，6 条）：
  `Anime4K_Clamp_Highlights → Anime4K_Restore_CNN_VL → Anime4K_Upscale_CNN_x2_VL → Anime4K_AutoDownscalePre_x2 → Anime4K_AutoDownscalePre_x4 → Anime4K_Upscale_CNN_x2_M`
- **通用档**（1 条）：`ravu-lite-r4.hook`
- **高质量档**（1 条）：`FSRCNNX_x2_8-0-4-1.glsl`

> ★ **不要照抄"占位名"**：`RAVU-Lite.glsl` / `FSRCNNX_x2.glsl` 这类名字上游**并不存在**；清单里写的名字必须与 `files/` 里放的文件**完全一致**（宿主安装期会逐条校验存在性）。

## 3. 两个易踩的坑
1. **RAVU-Lite 的两个变体是"二选一"**：`ravu-lite-r4.hook`（LUT 内嵌，单文件自足）与 `ravu-lite-ar-r4.hook`（`-ar-` 变体）都写 `//!SAVE ravu_lite_int`，**同槽位互斥**（串联会互相覆盖）。只挑一个放 `files/`，另一个别打包（避免留无用文件）。
2. **`.hook` 扩展名是合法的**：mpv 的 `glsl-shaders` 按文件头 `//!HOOK` 等指令识别用户着色器，**不看扩展名**；`.glsl` / `.hook` 都能用。别为了"看起来统一"去重命名上游文件（重命名会让溯源变难）。

## 4. 从哪拿（本机网络约束）
`github.com` 直连被墙，用加速前缀拼原地址（宿主侧同一套池，见 `src/shared/update.ts` 的 `GH_ACCEL_PREFIXES`）：
- release 资产：`https://<加速前缀>/https://github.com/<owner>/<repo>/releases/download/<tag>/<asset>`
- 仓库文件：`https://<加速前缀>/https://raw.githubusercontent.com/<owner>/<repo>/<ref>/<path>`

拿到的字节**记 sha256**（供宿主内置许可清单固化，见规范 §7）。
