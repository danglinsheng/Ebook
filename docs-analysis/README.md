# FSNotes（MyFSNotes）项目分析报告

> 分析对象：`/Users/dawn/workspace/codes/fsnotes`（远程 `glushchenko/fsnotes`，本地定制分支，App 显示名 **MyFSNotes**）
> 分析基线：commit `a738e3c` + 本地未提交改动（约 +987/-192 行）

---

## 1. 项目概述

FSNotes 是一款开源（MIT）的原生笔记应用，支持 **macOS 和 iOS**。核心理念是"拥有你的笔记"：所有笔记以**纯 Markdown / 纯文本文件**保存在用户选择的目录（本地、iCloud Drive、Dropbox 或任意外部目录）中，不依赖私有数据库，因此不会被厂商锁定。

主要能力：

- 快速的富文本 Markdown 编辑器（所见即所得的语法高亮）+ WebKit 渲染预览
- 标签（`#tag`）、双向链接（`[[link]]`）、待办（`- [ ]`）、引用、表格、脚注
- 代码块语法高亮（自研 SwiftHighlighter）、Mermaid 图表、MathJax 公式
- 图片/附件管理（`.textbundle` 包格式）、AES-256 加密笔记（`.etp`，RNCryptor）
- iCloud Drive 同步、App Groups 共享、Git 自动备份/版本历史、Web 发布（fsnot.es）
- macOS 菜单栏快速记录、全局快捷键、多窗口、打印；iOS 主屏快捷动作、分享扩展

本地版本在上游基础上做了**中文菜单定制**和**性能/稳定性优化**（详见第 6 节）。

---

## 2. 功能详情

### 2.1 macOS 应用（`FSNotes/`）

| 模块 | 文件 | 功能 |
|---|---|---|
| 应用入口 | `AppDelegate.swift` | 启动流程、存储目录选择（沙盒书签）、Dock 图标、菜单栏状态项（NSStatusItem）、窗口状态持久化（`editors.settings`）、崩溃恢复、iCloud→本地迁移提示 |
| 主窗口 | `ViewController.swift`（2662 行）+ `ViewController+Menu/Git/WebApi/Print/Web` | 侧边栏（文件夹/标签/收件箱/回收站/待办/未标签）、笔记列表、搜索、排序、Git 队列、Web API 上传、打印 |
| 编辑器 | `EditorViewController.swift`（基类，1578 行）、`EditorViewController+ScrollPosition/Sharing` | 编辑/预览切换、加密解锁对话框、自动保存定时器、Git 快照/拉取定时器、菜单校验（`validateMenuItem`） |
| 编辑控件 | `View/EditTextView*.swift` | 自定义 `NSTextView`：自动配对、Todo 勾选、拖拽、行移动、自动补全（`EditTextView+Complete`） |
| 预览 | `MPreviewView`（FSNotesCore） | `WKWebView` 渲染 HTML，JS 桥处理复选框、链接、图片 QuickLook、滚动同步、剪贴板 |
| 侧边栏/列表 | `View/Sidebar*.swift`、`NotesTableView.swift`、`NoteCellView.swift` | 大纲式文件夹树、虚拟项目（全部/待办/未标签/回收站）、备注计数 |
| 设置 | `Preferences/*.swift` | 通用/编辑器/外观/Web/Git/安全（主密码、Touch ID）/高级 多面板设置 |
| 文件监控 | `Helpers/FileWatcher.swift`、`FileSystemEventManager.swift` | FSEvents 监听所有项目目录，外部增删改实时同步到 UI；处理 `mustRescan`/事件丢失时整树重扫 |
| 打印 | `Helpers/Printer.swift`、`PrinterLegacy.swift` | 通过 WKWebView 生成分页 HTML 打印 |
| 定制功能 | `ViewController.swift` 新增方法 | ① 从模板新建（`Templates/` 目录）② 导入文件夹（目录树复制，`.txt` 转 `.md`）③ 备份存储目录（时间戳拷贝）④ 文件夹按名称排序 |

### 2.2 共享核心（`FSNotesCore/`，macOS/iOS 共用）

| 模块 | 关键文件 | 职责 |
|---|---|---|
| 数据模型 | `Business/Note.swift`（2401 行）、`Project.swift`、`Storage.swift`（1727 行） | 笔记/文件夹的加载、解析、保存、移动、删除、标签扫描、缓存、虚拟项目、回收站 |
| 文本处理 | `NotesTextProcessor.swift`（1378 行）、`TextStorageProcessor.swift`、`CodeBlockDetector.swift` | Markdown 语法高亮（标题/引用/列表/链接/图片）、代码块检测与增量高亮、图片附件加载 |
| 解析器 | `FSParser.swift` | YAML front matter、`#tag`、行内代码、图片的正则解析 |
| Markdown→HTML | `Business/Markdown.swift` | 基于 **libcmark-gfm**（C 库）渲染，启用脚注/表格/删除线/任务列表/自动链接；`{{TOC}}` 目录占位 |
| 预览模板 | `MPreviewView.swift` | 拼装 HTML 模板，注入 MathJax、Mermaid、源码定位（sourcepos）、Web 发布模式 |
| 语法高亮 | `SwiftHighlighter/` | 自研代码高亮器（多语言 + 主题，暗/亮色） |
| 图片处理 | `ImagesProcessor.swift`、`NSTextAttachment+.swift`、`URL+Image.swift` | 下载远程图片、生成缩略图、`.icloud` 占位处理、附件命名去重 |
| 加密 | `Note.swift` + `RNCryptor` | `.etp`（Encrypted Text Pack）：AES-256 加密整个笔记内容，主密码存 Keychain，支持 Touch ID 解锁 |
| 文本包 | `Note.swift`（`loadTextBundle`/`writeTextBundle` 等） | `.textbundle`（macOS 标准包：`text.md` + `assets/` + `info.json`）读写与互转 |
| Git | `Git/`（swift-git 封装）、`Extensions/Project+Git.swift`、`Storage+Git.swift` | 裸仓库管理、提交/快照/拉取/推送/冲突解决；仓库位置可配置（独立 `.git` 或集中存储） |
| 存储抽象 | `Business/Storage.swift`、`UserDefaultsManagement.swift`、`SandboxBookmark.swift` | 单例 `Storage` 维护 `noteList`/`projects`/`tags` 内存索引；macOS 用 `UserDefaults.standard`，iOS 用 App Group `group.es.fsnot.user.defaults`；沙盒安全书签持久化目录访问权限 |
| Web API | `ViewController+WebApi.swift` | multipart 上传笔记（HTML + 图片 zip）到 `https://api.fsnot.es/`，返回 `https://p.fsnot.es/{id}/` |

### 2.3 iOS 应用（`FSNotes iOS/`）

- `SceneDelegate.swift`：Scene 生命周期、冷启动快捷动作（新建/剪贴板/搜索）、URL Scheme 处理（`fsnotes://open?tag=`、`fsnotes://find?id=`）
- `ViewController.swift`（列表）+ `EditorViewController.swift`（编辑器）、`MainNavigationController`
- 组件：`RevisionsViewController`（历史版本）、`MoveViewController`（移动）、`ImagePreviewViewController`、`DatePickerViewController`、QuickLook 预览
- 多语言本地化（`fr/nl/pt/ru/uk/zh-Hans` 等 `.lproj` + `xcstrings`）
- 数据模型文件 `FSNotes_iOS.xcdatamodeld` 存在，但笔记本体仍以文件形式存储于 iCloud/App Group 容器

### 2.4 分享扩展（`FSNotes iOS Share/`）

- `ShareViewController`（`SLComposeServiceViewController`）：接收其他 App 分享的文本、URL、图片，创建笔记到共享存储（App Group），供主 App 读取

---

## 3. 技术栈

| 类别 | 选型 |
|---|---|
| 语言 | **Swift 5**（少量正则/字符串密集逻辑） |
| 平台 | macOS（AppKit，`NSViewController`/`NSTextView`/`WKWebView`）、iOS（UIKit，Scene 生命周期） |
| 构建系统 | Xcode 工程（`FSNotes.xcodeproj` + `xcworkspace`），3 个 app target：`FSNotes`(macOS)、`FSNotes iOS`、`FSNotes iOS Share`，共享代码以 `FSNotesCore` 源码组形式编译进各 target |
| 本地依赖管理 | SwiftPM 远程包（`XCRemoteSwiftPackageReference`） |
| 第三方库 | **libcmark_gfm**（Markdown→HTML，C）、**RNCryptor**（AES-256 加密）、**swift-git**（libgit2 封装，glushchenko fork）、**ZipArchive**、**SwiftHighlighter**（自研，源码内置）、**SoulverCore**（自然语言计算）、**Shout**（快捷键）、**DropDown**、**SwipeCellKit**（iOS 列表滑动）、**TOCropViewController**（图片裁剪） |
| Apple 框架 | Foundation、AppKit/UIKit、WebKit、CoreServices/UTI、FSEvents（文件监听）、Security/Keychain、LocalAuthentication（Touch ID/面容）、UserNotifications、CoreData（模型文件）、CloudKit 条件编译块（`CLOUD_RELATED_BLOCK`）、Carbon.HIToolbox（快捷键） |
| 文件格式 | Markdown/纯文本、`.textbundle` 包、`.etp` 加密包、自定义缓存目录 |
| UI 构建 | Storyboard/XIB + 代码布局（`NSLayoutConstraint`），Asset Catalog，`xcstrings` 本地化 |
| 代码质量 | SwiftLint（`.swiftlint.yml`）、`.travis.yml` CI 配置、Notarization plist |
| 部署目标 | macOS 10.14 / 12.4，iOS 12.0 / 18.0（多 target 配置并存） |
| 同步/云 | iCloud Drive（ubiquity container）、App Groups（iOS 共享）、NSUbiquitousKeyValueStore（云端 pin）、Dropbox/任意目录（安全书签） |

---

## 4. 架构

### 4.1 分层架构

```
┌─────────────────────────────────────────────────────────┐
│ UI 层（平台相关）                                          │
│  macOS: AppKit — AppDelegate / MainWindowController /     │
│        ViewController(主) / EditorViewController /        │
│        EditTextView / Sidebar* / MPreviewView(WKWebView)  │
│  iOS:   UIKit  — SceneDelegate / ViewController(列表) /   │
│        EditorViewController / Share 扩展                   │
├─────────────────────────────────────────────────────────┤
│ 业务核心层 FSNotesCore（跨平台共享源码）                     │
│  模型: Note / Project / Storage(单例索引) / NoteMeta /     │
│        SearchQuery / SidebarItem                           │
│  文本: NotesTextProcessor / TextStorageProcessor /        │
│        FSParser / CodeBlockDetector / SwiftHighlighter     │
│  渲染: Markdown(libcmark-gfm) / MPreviewView / HtmlExtractor│
│  附件: ImagesProcessor / NoteAttachment / TextBundle       │
│  加密: RNCryptor(.etp) / KeychainConfiguration            │
│  同步: Git(swift-git) / FileWatcher / SandboxBookmark     │
├─────────────────────────────────────────────────────────┤
│ 基础设施                                                  │
│  UserDefaultsManagement(平台差异封装) / Extensions /       │
│  Pasteboard / UTI / URL+Image                              │
└─────────────────────────────────────────────────────────┘
        │                    │                    │
   本地文件系统        iCloud Drive/App Group   Git 裸仓库
 (md/txt/textbundle/etp)   (跨设备同步)        (自动备份/版本)
        │                    │                    │
   FSNotes Web API(api.fsnot.es) ←—— WebKit 预览/发布
```

### 4.2 关键设计决策

1. **文件即数据库**：笔记就是文件，`Storage` 单例在内存中维护 `noteList`/`projects` 索引 + `notesByProject` 快速索引（本地定制新增），启动时扫描文件系统重建；项目（Project）= 文件夹，子目录自动成为子项目。
2. **NSTextStorage 增量高亮**：`TextStorageProcessor` 作为 `NSTextStorage` 代理，只对编辑段落增量重高亮，代码块由 `CodeBlockDetector` 单独处理，避免整篇重排。
3. **双渲染管线**：编辑器内用 `NSAttributedString` 实时高亮（SwiftHighlighter），预览/发布用 libcmark-gfm 生成 HTML + 模板注入 MathJax/Mermaid，在 WKWebView 中呈现。
4. **平台共享核心**：`FSNotesCore` 用 `#if os(OSX)`/`#else` 条件编译兼容 AppKit/UIKit，`Platform.swift` 统一 `PlatformImage/Color/Font` 类型别名。
5. **外部变更感知**：FSEvents → `FileWatcher` → `FileSystemEventManager` 去抖合并 → 与内存索引 diff（`getProjectDiffs`/`syncImportedTree`）→ 只刷新变化的行。
6. **加密透明化**：`.etp` 文件整体加密，解锁后在内存中以明文 `NSTextStorage` 编辑，保存时重新加密；密码存 Keychain，可选 Touch ID。

---

## 5. 执行流程

### 5.1 macOS 启动流程

```
AppDelegate.applicationWillFinishLaunching
  ├─ checkStorageChanges()      // 检测 iCloud→本地迁移，弹窗确认
  ├─ loadDockIcon()
  └─ constructMenu()             // 菜单栏模式
        │
AppDelegate.applicationDidFinishLaunching
  ├─ applyAppearance()           // 深/浅/跟随系统
  ├─ iCloud Documents 目录确保存在（CLOUD_RELATED_BLOCK）
  ├─ UserDefaultsManagement.storagePath == nil ?
  │     └─ requestStorageDirectory() → NSOpenPanel 选目录
  │           → SandboxBookmark 保存安全书签 → restartApp()
  └─ 从 Main.storyboard 实例化 MainWindowController
        └─ ViewController.viewDidLoad
              ├─ Storage.shared() 初始化
              │     ├─ SandboxBookmark.load()        // 恢复目录访问权限
              │     ├─ getRoot()                     // 解析存储根（iCloud/本地/自定义/Dropbox）
              │     ├─ removeCachesIfCrashed()
              │     ├─ 创建默认 Project + Trash
              │     └─ assignTrash / assignBookmarks
              ├─ loadInboxAndTrash() / loadProjects()// 扫描文件系统建索引
              ├─ FileSystemEventManager.start()      // 启动 FSEvents 监听
              ├─ installStorageMenuItems()           // 【定制】注入模板/导入/备份/排序菜单
              └─ checkWelcome()                      // 首次运行向导笔记
```

### 5.2 打开并编辑一篇笔记

```
用户点击笔记（NotesTableView / 侧边栏）
  └─ Note.uiLoad() → load(tags:) → parseURL() → 读取文件内容
       ├─ .etp → Keychain 取密码 → RNCryptor.decrypt → 明文
       ├─ .textbundle → loadTextBundle() 读取 text.md + assets
       └─ 解析 YAML front matter（标题/创建时间/标签）
  └─ EditTextView 加载 NSAttributedString
       └─ TextStorageProcessor（NSTextStorageDelegate）
             每次编辑 → process()
               ├─ NotesTextProcessor.highlightMarkdown()   // 段落级 Markdown 高亮
               ├─ CodeBlockDetector.findCodeBlocks()         // 代码块范围
               ├─ SwiftHighlighter.highlight()               // 代码块语法高亮
               └─ loadImages()                               // 附件/远程图片异步加载
  └─ 预览开关 → MPreviewView.buildPage(note)
       ├─ renderMarkdownHTML()（libcmark-gfm + table/tasklist 等扩展）
       ├─ 替换 {{TOC}} / 注入 MathJax / Mermaid / 样式模板
       └─ WKWebView.loadHTMLString → JS 桥（复选框/链接/滚动同步）
  └─ 自动保存（防抖）→ Note.save()
       ├─ getContentSaveURL()：.md 直写 / .textbundle 写包 / .etp 加密写
       └─ 写入失败进入 plainWriter/ciphertextWriter 队列重试
```

### 5.3 搜索流程

```
SearchTextField 输入 → SearchQuery.setFilter(terms/tags/引号短语)
  └─ searchQueue（OperationQueue）→ Storage.sortNotes(noteList, query)
       └─ SearchQuery.isFit(note:) 逐篇匹配（标题/内容/标签/项目）
  └─ notesTableView.doVisualChanges(results:) 局部刷新列表
```

### 5.4 外部文件变更同步

```
FSEvents 回调（FileWatcher）
  ├─ event.mustRescan（事件丢弃/需重扫）→ scheduleTreeSync(path) 整树同步
  └─ 普通事件 → 去抖（pendingSyncWork）→ Storage.syncImportedTree / getProjectDiffs
       ├─ insert(url:) 新项目/笔记 → sidebarOutlineView.insertRows
       ├─ removeBy / relocate → notesTableView.doVisualChanges
       └─ fsManager.reloadObservedFolders() 动态增删监听目录
```

### 5.5 Git 自动备份（macOS）

```
EditorViewController.snapshotsTimer（按间隔）
  └─ ViewController.gitQueue（maxConcurrentOperationCount = 1，串行）
       ├─ 有变更 → commit 快照（Project+Git.getRepositoryUrl：独立 .git 或集中 gitStorage）
       ├─ 有 remote → pull/push（swift-git / libgit2），冲突按设置自动解决或提示
       └─ 更新 UI（prevCommit / updateViews）
```

### 5.6 iOS 启动与分享

```
SceneDelegate.scene(willConnectTo:)
  ├─ 窗口 + MainNavigationController(root: 列表 ViewController)
  ├─ 预加载 editorController
  └─ 处理冷启动 shortcutItem / URL / state restoration
列表 ViewController → Storage.shared()（App Group 容器）→ 同样的 Project/Note 模型
编辑 → 同一套 FSNotesCore 文本处理与预览管线（UIKit 分支）
分享扩展 ShareViewController → 读取 NSExtensionItem 附件（文本/URL/图片）
  → 写入共享容器 → 主 App 下次启动 loadProjects 发现新笔记
```

### 5.7 Web 发布

```
用户触发分享到 Web → createAPI(note)
  ├─ MPreviewView.buildPage(web: true) 生成静态 HTML
  ├─ 收集图片/附件 + ZipArchive 打包
  └─ POST multipart 到 api.fsnot.es（key 认证）→ 返回 p.fsnot.es/{id}/
更新/删除通过 note.apiId 走 method=update/delete
```

---

## 6. 本地定制改动（相对上游 glushchenko/fsnotes）

本地工作区在上游 `a738e3c` 之上有未提交改动（约 +987/-192 行），核心是**中文功能定制 + 大库性能优化**：

1. **品牌定制**：App 显示名改为 `MyFSNotes`（Info.plist / storyboard / 菜单）。
2. **新增文件菜单（中文）**：
   - `从模板新建`：从存储根 `Templates/` 目录选模板创建笔记；
   - `导入文件夹…`：递归导入目录树，普通文本自动转同名 `.md`（`importFolderTree`/`copyImportTree`），导入后 `Storage.syncImportedTree` 增量刷新侧边栏与列表；
   - `备份存储目录…`：把整个存储目录复制到 `FSNotes Backup <时间戳>`；
   - `文件夹按名称排序`：重排项目顺序。
3. **性能优化**：
   - `Storage.notesByProject` 索引：侧边栏选中文件夹不再全量扫描 `noteList`；
   - `collectSubdirectories` 重写：移除约 200 个目录的扫描上限，修复深层目录笔记丢失；路径统一 `standardizedFileURL` 比较；
   - `Note.loadListPreview()`：列表预览只读文件前 8KB（含 YAML title 解析），iCloud 未下载文件不触发下载；`.txt` 笔记支持"文件名即标题"（`prefersFileNameTitle`）；
   - `FileWatcherEvent.mustRescan`：处理 FSEvents 内核丢事件/合并事件，触发整树重扫。
4. **EditorViewController**：导入面板支持多选目录；图片/附件相关小调整。

---

## 7. 目录结构

```
fsnotes/
├── FSNotes/                  # macOS App（AppKit）
│   ├── AppDelegate.swift     # 启动/菜单栏/窗口状态
│   ├── ViewController.swift  # 主界面（侧边栏+列表+搜索+Git+Web）
│   ├── EditorViewController.swift  # 编辑器基类（macOS/iOS 共用逻辑）
│   ├── View/                 # 自定义 UI 控件（EditTextView、Sidebar、Preview 等）
│   ├── Helpers/              # FileWatcher、FileSystemEventManager、Printer、SandboxBookmark
│   ├── Preferences/          # 设置面板
│   └── Model/                # Core Data 类（遗留）
├── FSNotesCore/              # 跨平台共享核心（~60 个 Swift 文件）
│   ├── Business/             # Note / Project / Storage / SearchQuery / Markdown
│   ├── FSParser.swift        # Markdown 结构正则解析
│   ├── NotesTextProcessor.swift / TextStorageProcessor.swift / CodeBlockDetector.swift
│   ├── SwiftHighlighter/     # 代码高亮器（语言/主题）
│   ├── MPreviewView.swift    # WKWebView 预览 + HTML 模板
│   ├── ImagesProcessor.swift / HtmlExtractor.swift
│   ├── Git/                  # swift-git 封装（repository/commit/diff/status/...）
│   ├── KeychainConfiguration.swift / UserDefaultsManagement.swift
│   └── Extensions/           # NSAttributedString/URL/FileManager 等扩展
├── FSNotes iOS/              # iOS App（UIKit + Scene）
├── FSNotes iOS Share/        # iOS 分享扩展
├── Resources/                # Web 预览资源（js/css/模板）
├── FSNotes.xcodeproj/        # Xcode 工程（SwiftPM 依赖声明）
└── docs-analysis/            # 本报告与架构图
```

---

## 8. 架构图索引

| 图 | 文件 | 内容 |
|---|---|---|
| 图 1 | `diagrams/01-总体架构图.jpg` | 系统总体架构：客户端、核心层、存储与外部服务 |
| 图 2 | `diagrams/02-模块分层图.jpg` | 代码模块分层与依赖关系 |
| 图 3 | `diagrams/03-编辑渲染流程图.jpg` | 打开/编辑/高亮/预览/保存执行流程 |
| 图 4 | `diagrams/04-启动同步流程图.jpg` | 应用启动、文件监控同步、Git 备份流程 |

图源文件（Graphviz DOT）保留在 `diagrams/src/`，可用 `dot -Tjpg` 重新生成。
