# MiniBrowser

自带内核的极简安卓浏览器：**GeckoView（Mozilla Firefox 引擎）**，不依赖系统 WebView，
因此在缺少/老旧 WebView 的手表上也能正常渲染网页。手机、手表一套代码两种界面。
启动器图标来自根目录 `icon.jpg`（圆角/圆形遮罩版由 `icon_gen.py` 生成）。

## 功能

| 功能 | 手机版 | 手表版 |
| --- | --- | --- |
| 自带内核 | GeckoView（与系统 WebView 无关） | 同左（适配无 WebView 的手表） |
| 界面 | 地址栏 + 底部导航 + 菜单弹层 | **可拖动浮动菜单按钮**（点按开全功能菜单，拖动换位）+ 圆屏安全边距 |
| 多标签页 | ✅ 标签计数芯片 → 九宫格切换/关闭/新建 | 无多标签 UI（菜单内“替换式”新建） |
| 下载管理 | ✅ 进度 / 取消 / 重试 / 删除 / 打开 | ✅ 全屏大行列表 |
| 历史记录 | ✅ 按日分组、可搜索、可清空 | ✅ 大行高 |
| 收藏夹 | ✅ ☆ 一键收藏 / 预置常用站 | ✅ |
| 无痕模式 | ✅ 独立无痕标签（不写历史） | ✅ 菜单一键无痕 |
| 深色 | ✅ 跟随系统 / 强制浅色 / 强制深色，**网页内容同步深色** | ✅ |
| 搜索引擎 | **默认必应**（可切百度 / Google / DuckDuckGo） | ✅ |
| 起始页 | 仅保留「自带 Gecko 内核」标识 + 搜索栏 + 单个「＋ 添加收藏」按钮（表单页 `mb://addform` 录入） | ✅ |

## 构建

要求：JDK 17+（本机 21）、Android SDK Platform 35（本机 `workspace1\.toolchain\android-sdk`）。

```powershell
# wrapper 依赖 services.gradle.org（本机证书链不可用），请直接用 .toolchain 里的 Gradle：
& "C:\Users\SB\Documents\deepseek-harness\workspace1\.toolchain\gradle-8.10.2\bin\gradle.bat" assembleRelease
```

> 依赖仓库说明：`settings.gradle.kts` 把 `1\local-maven` 排在首位 —— GeckoView 的 297MB AAR
> 已手动下载到该本地 Maven 仓库（maven.mozilla.org 直连易断流，VPN 环境下也建议保持此配置）。

产物（按 ABI 切分，按设备挑一个安装）：

```
app/build/outputs/apk/release/
├── app-arm64-v8a-release.apk      # 主流手机 / Wear OS 3+ 手表（已签名，推荐）
├── app-armeabi-v7a-release.apk    # 32 位老手表（Wear OS 1/2）、老低端机（已签名）
└── app-x86_64-release.apk         # x86_64 模拟器，如本机 MuMu（已签名）
```

Debug 包同名在 `app/build/outputs/apk/debug/`。签名使用根目录 `minibrowser.keystore`
（口令见 `keystore.properties`，演示密钥，发布前请自行更换）。

## 技术要点

- **内核**：`org.mozilla.geckoview:geckoview:138.0.20250512124033`（Gecko 138 稳定版）。
  选 138 的原因：139 起 GeckoView 要求 compileSdk 36 + AGP 9.1，而本机工具链是
  AGP 8.7.3 / compileSdk 35 / Gradle 8.10.2（138 是最后一个以 compileSdk 35 构建的版本）。
- **下载**：GeckoView 没有下载 API，唯一入口是 `ContentDelegate.onExternalResponse(WebResponse)`，
  浏览器自己消费 `response.body` 流写入文件（取消 = 关闭流），Android 10+ 落到系统"下载"目录（MediaStore），
  低版本落应用外部 Download 目录（免运行时权限）。
- **无痕**：`GeckoSessionSettings.Builder().usePrivateMode(true)`（initOnly，构造会话时锁定）。
- **深色网页**：`GeckoRuntimeSettings.setPreferredColorScheme(COLOR_SCHEME_LIGHT/DARK)`。
- **起始页**：上层直接 `Loader().data(html.toByteArray(), "text/html")` 以 base64 data URI 渲染
  （String 重载的 data URI 不转义 `#`，CSS 色值会把文档截断成空白——1.0.1 实测教训）；
  `onLoadRequest` 拦截 `mb://` 仅作异步兑底。
- **界面识别**：设置三档（自动/手机版/手表版）；自动 = `FEATURE_WEAR` 或 `sw < 400dp`。
- **多标签**：每个标签一个 `GeckoSession`，单个 `GeckoView` 控件按当前标签切换绑定；
  `target=_blank` / `window.open` 经 `onNewSession` 自动开新标签。

## 已知边界（v1）

- 标签页不跨进程重启持久化（进程被杀后回到起始页）。
- 桌面版 UA 未做"请求桌面站点"开关（内核默认移动 UA）。
- 网页内 JS 播放的受 DRM/特殊编解码视频受 GeckoView 自带 exoplayer 能力限制。

## 版本历史

- **1.1.1**（versionCode 4）：**修复"部分网页一直自动刷新"的严重问题** —— 低内存/被杀内容进程的
  自动恢复重载此前是**无界**的（进程被杀→重载→再被杀→…永动刷新）。现改为有界节流：
  仅前台标签、连续自动重载上限 **2 次** + **5 秒冷却**，超限即停（用户手动导航/刷新才重置计数）。
  已用 root 定点连杀内容进程 3 次实测：**恰好 2 次自动重载后彻底停住**。
  另注：个别站点（IP 检测类等）在 VPN 切换/网络抖动时会自行 location.reload，属站点行为，网络稳定后自愈。
- **1.1.0**（versionCode 3）：
  ① 起始页精简为「自带 Gecko 内核」标识 + 搜索栏 + 单个「＋ 添加收藏」按钮（收藏录入走
  `mb://addform` 表单页 → `mb://add?url=...` 路由回写，不依赖 JS prompt）；
  ② 手表版取消固定导航条，全部导航/功能集成进**可拖动的浮动菜单按钮**（点按开菜单、
  拖动换位、自动夹取内容区）；菜单改为可滚动并新增 返回/前进/刷新/主页 条目；
  ③ **圆屏适配**（记住圆屏显示不全的问题）：头部/输入页加安全边距、菜单底部给表盘下弧留白、
  网页内容留边，避免表盘裁切；
  ④ 启动器图标换成 `icon.jpg`（生成圆角方形 + 圆形两套 PNG 全密度）；
  ⑤ 应用改名 **MiniBrowser**。
- **1.0.1**（versionCode 2）：修复三个启动级 bug ——
  ① `BookmarksStore` 初始化顺序 NPE（`seed()` 在 `_items` 赋值完成前调用 `persist()`，每次启动必崩）；
  ② 起始页白屏：`Loader().data(String, String)` 生成的 data URI **不转义 `#`**，CSS 色值把文档截断，
  改用 `data(byte[])`（base64）；起始页改为上层直接加载、拦截逻辑仅作异步兑底；
  ③ 深色下磁贴白底浅字：暗色 CSS 规则先于基础规则定义被同权重覆盖，改用 `body .tile` 提权。
  已在 MuMu（V2284A）实测通过：启动 / 起始页 / 必应加载渲染 / 网页触摸与输入 / 地址栏导航 /
  历史落盘 / 收藏磁贴 / 手表大图标 UI / 深色（App 与网页同步）。
- 1.0.0（versionCode 1）：首个版本。
