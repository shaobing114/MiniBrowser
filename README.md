# MiniBrowser

自带内核的极简安卓浏览器：内嵌 **GeckoView（Mozilla Firefox 引擎）**，完全不依赖系统
WebView —— 在缺少或老旧 WebView 的手表、老设备上也能正常渲染现代网页。
一套代码同时适配 **手机**（多标签、下载管理）与 **手表**（大触控目标、可拖动菜单、圆屏安全边距）。

## 功能

| 功能 | 手机版 | 手表版 |
| --- | --- | --- |
| 自带内核 | GeckoView（与系统 WebView 无关） | 同左 |
| 界面 | 地址栏 + 底部导航 + 菜单弹层 | **可拖动浮动菜单按钮**（点按开全功能菜单，拖动换位）+ 圆屏安全边距 |
| 多标签页 | ✅ 标签计数芯片 → 九宫格切换/关闭/新建 | 无多标签 UI（菜单内“替换式”新建） |
| 下载管理 | ✅ 进度 / 取消 / 重试 / 删除 / 打开 | ✅ 全屏大行列表 |
| 历史记录 | ✅ 按日分组、可搜索、可清空 | ✅ 大行高 |
| 收藏夹 | ✅ ☆ 一键收藏、起始页「＋ 添加收藏」表单 | ✅ |
| 无痕模式 | ✅ 独立无痕标签（不写历史） | ✅ 菜单一键无痕 |
| 深色 | ✅ 跟随系统 / 强制浅色 / 强制深色，**网页内容同步深色** | ✅ |
| 搜索引擎 | **默认必应**（可切百度 / Google / DuckDuckGo） | ✅ |
| 起始页 | 仅「自带 Gecko 内核」标识 + 搜索栏 + 单个添加按钮 | ✅ |
| 界面模式 | 设置里三档：自动识别 / 强制手机 / 强制手表 | 同左 |

## 构建

要求：JDK 17+、Android SDK Platform 35。

```bash
# Android Studio：直接打开本目录，配置 SDK 后 Run 即可
# 命令行：
./gradlew assembleDebug      # 调试包（自动用 debug 密钥签名）
./gradlew assembleRelease    # 发布包（未签名；配置密钥后自动签名，见下）
```

> 若 wrapper 下载 Gradle 受网络限制，可改用系统里已安装的 Gradle 8.10+ 执行同样的任务。

产物按 ABI 切分，按设备选择安装：

```
app/build/outputs/apk/
├── debug/   app-{arm64-v8a|armeabi-v7a|x86_64}-debug.apk
└── release/ app-{arm64-v8a|armeabi-v7a|x86_64}-release.apk
    # arm64-v8a：主流手机 / Wear OS 3+ 手表
    # armeabi-v7a：32 位老手表（Wear OS 1/2）、老低端机
    # x86_64：x86_64 模拟器
```

**release 签名（可选）**：在仓库根目录放置 `keystore.properties`：

```properties
storeFile=your.keystore
storePassword=...
keyAlias=...
keyPassword=...
```

存在密钥文件时 release 自动签名，否则输出未签名 APK（不影响 debug 构建）。
密钥与 `keystore.properties` 已被 `.gitignore` 排除，切勿提交。

**GeckoView 依赖**：由 `settings.gradle.kts` 中的
`https://maven.mozilla.org/maven2` 提供（首次构建需下载约 290MB 的 AAR）。
若网络不稳，可手动下载 `org/mozilla/geckoview/geckoview/<版本>/{*.aar,*.pom,*.module}`
放入本地目录，按 `settings.gradle.kts` 内注释启用本地仓库。

## 项目结构

```
app/src/main/java/com/minibrowser/app/
├── MainActivity.kt           # 边到边、主题、手机/手表界面路由、VIEW intent
├── kernel/Engine.kt          # GeckoView 封装：会话/委托/事件、mb:// 路由、被杀恢复(有界节流)
├── data/
│   ├── model.kt              # 枚举、数据类、地址栏输入→URL/搜索 解析
│   ├── SettingsStore.kt      # 主题 / 界面模式 / 搜索引擎
│   ├── HistoryStore.kt       # 历史（按 URL 去重、上限 500）
│   ├── BookmarksStore.kt     # 收藏（首次启动预置）
│   └── DownloadsStore.kt     # 下载（引擎流式写入、MediaStore/免权限两路）
└── ui/
    ├── BrowserViewModel.kt   # 多标签、导航语义、事件→状态、收藏/下载联动
    ├── HomePage.kt           # 起始页 / 添加收藏表单（base64 data URI 渲染）
    ├── PhoneRoot.kt          # 手机 UI（地址栏、底部导航、菜单、标签九宫格）
    ├── WatchRoot.kt          # 手表 UI（可拖动 FAB 菜单、圆屏适配）
    ├── LibraryScreens.kt     # 历史 / 收藏
    ├── DownloadsScreen.kt    # 下载管理
    ├── SettingsScreen.kt     # 设置
    └── theme/ / components/  # 主题与公共组件
```

## 技术要点

- **内核选择**：`org.mozilla.geckoview:geckoview`。当前锁定 138 稳定版 ——
  139 起 GeckoView 要求 compileSdk 36 + AGP 9.1；若你的构建链允许，建议直接升级依赖版本。
- **下载**：GeckoView 没有下载 API，唯一入口是
  `ContentDelegate.onExternalResponse(WebResponse)`：应用自己消费 `response.body`
  流写入文件（取消 = 关闭流）；Android 10+ 落系统「下载」目录（MediaStore），
  低版本落应用外部 Download 目录（免运行时权限）。
- **无痕**：`GeckoSessionSettings.Builder().usePrivateMode(true)`（initOnly，构造会话时锁定）。
- **深色网页**：`GeckoRuntimeSettings.setPreferredColorScheme(COLOR_SCHEME_LIGHT/DARK)`；
  起始页 CSS 同时用 `@media (prefers-color-scheme: dark)` 自适应。
- **起始页**：`Loader().data(html.toByteArray(), "text/html")` 以 **base64 data URI** 渲染。
  注意 String 重载生成的 data URI **不转义 `#`**，CSS 色值会把文档截断成空白页；
  `onLoadRequest` 拦截 `mb://`（addform / add 路由）仅作兜底。
- **多标签**：每标签一个 `GeckoSession`，单个 `GeckoView` 控件按当前标签切换绑定；
  `target=_blank` / `window.open` 经 `onNewSession` 自动开新标签。
- **被杀恢复（防无限刷新）**：内容进程被杀/崩溃后的自动重载是**有界**的 ——
  仅前台标签、连续上限 2 次、5 秒冷却，超限即停；只有用户手动导航/刷新才重置计数。
- **界面识别**：`Configuration.isScreenRound` + 特性/最小宽度自动判定，可在设置强制覆盖。

## 图标

启动器图标来自根目录 `icon.jpg`；修改后运行 `python icon_gen.py` 重新生成
圆角方形 + 正圆两套 PNG（全密度）。

## 版本历史

- **1.1.2**（versionCode 5）：侧滑/返回键改为**应用内返回上一步**（地址编辑 → 菜单弹层 →
  子页面 → 网页历史），无上一步可退时才退出应用；顺带修复"退出后进程被缓存复用导致
  再次启动崩溃"（`Only one GeckoRuntime instance is allowed`，复用进程内已有 runtime）。
- **1.1.1**（versionCode 4）：修复"部分网页一直自动刷新"——被杀内容进程的自动恢复重载
  改为有界节流（前台标签、连续 2 次上限、5 秒冷却、手动操作才重置），定点连杀实测恰好
  2 次重载后彻底停住。
- **1.1.0**（versionCode 3）：起始页精简（标识 + 搜索 + 单个添加按钮，`mb://addform` 表单
  录入收藏）；手表版导航全部收进可拖动浮动菜单按钮；圆屏安全边距适配；图标换 `icon.jpg`；
  应用改名 MiniBrowser。
- **1.0.1**（versionCode 2）：修复三个启动级 bug —— ① 收藏存储初始化顺序 NPE（启动必崩）；
  ② 起始页白屏：`Loader().data(String,…)` 的 data URI 不转义 `#`，改用 byte[] base64 重载；
  ③ 深色下起始页磁贴被同权重规则覆盖，改用提权选择器。
- **1.0.0**（versionCode 1）：首个版本。
