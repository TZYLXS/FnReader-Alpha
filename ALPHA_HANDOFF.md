# FnReader-Alpha 交接说明（供下一个 AI）

## 1. 任务目标与边界

本仓库是 FnReader 的独立 Alpha 暂存仓库，目标是调试 `阅读-debug 0.1.0`：

- 只允许修改和部署 `legado-web-debug`，端口 `1123`。
- 禁止修改正式 `legado-web`、正式版本号、生产端口 `1122`，禁止正式 FPK、main、tag、Release 操作。
- 当前新仓库是用户明确要求创建的独立仓库；原 FnReader、`yuedu-pro`、`yuedu-android4` 的改动仍未提交、未推送。
- 不包含用户书源明文。debug 曾恢复 1150 条书源，但书源备份不在本仓库。

## 2. 当前完成状态

- NAS debug 已实际更新并运行：`http://10.5.4.70:1123/`。
- `app=legado-web-debug`，版本 `0.1.0`，显示名 `阅读-debug`。
- Node `v26.13.1`，V8 `14.6.202.34-node.40`。
- Java `21.0.2+13-58`，虚拟线程已开启，健康接口已通过。
- Python `3.11.2`，持久 worker 池健康。
- debug 数据按用户授权清空后重建，仅保留并恢复书源；当前 1150 条。
- Pro/A4 debug APK 均编译成功，并通过 ADB 安装到 MuMu Android 12 实例。
- 远程 debug smoke 6/6 通过。
- 最近一次 `appcenter-cli start` 返回 fnOS `10500`，但服务进程随后实际正常启动，健康接口和 `/api/engine` 均通过；不要把 10500 回执单独当成失败，也不要用它掩盖真实进程检查。

## 3. 架构与关键行为

### Node 26
- 继续负责 HTTP API、软总线调度、缓存、普通 CSS/XPath 规则和复杂 JS fallback。
- 复杂规则（含 `<js>`、`@js:`、`java.*`）通过 `worker_threads` 解析池 offload，避免同步 `java.ajax` 阻塞主事件循环。
- 普通规则保留原主线程路径，避免全部 worker 化的额外开销。
- 复杂搜索缓存由主线程统一管理，worker 内关闭线程级缓存。
- 启用 `NODE_COMPILE_CACHE`；debug 启动默认 `UV_THREADPOOL_SIZE=16`。
- 包内自带 Node 26 和私有 `libatomic`，不依赖 fnOS 商店 `nodejs_v22`。

### Python
- `pyengine.js` 改为持久 JSONL 进程池，默认 2 个、最多 8 个 worker。
- 支持 request id、超时、故障重建、队列统计和关闭；不再每次调用重启解释器。
- `legado_bridge.py --worker` 执行 search/toc/content；新增 `normalize-toc`。
- Python TOC 内完成去重、过滤、章节号排序，返回 `normalized:"python"`。
- TXT 等大文件仍保留后台独立进程入口。
- 修复源码目录解析：兼容源码和打包目录，不再错误指向不存在的 `app/server/src/python`。

### Java 21 TOC 引擎
- `FnReaderJavaWorker.java` 无第三方 jar 依赖，监听 loopback 动态端口。
- Java 21 通过反射启用虚拟线程，JDK 17 仅用于开发兼容。
- Node ↔ Java 使用 Base64 三列 NDJSON，避免给 FPK 引入 JSON 库。
- FPK 使用预编译 class 和 jlink Java 21 运行时；运行时模块必须包含 `jdk.httpserver`。
- `/api/engine` GET 返回状态，POST `{javaEngine:boolean}` 开关；开启前健康检查，失败拒绝开启；关闭停止 worker。
- 设置配置默认 `javaEngine:true`。
- Android 新客户端在 Kotlin 内预处理 TOC 并回传 `normalized:true`；服务端信任该标记，不重复清洗。

### 软总线与并发
- 同一设备已锁定任务不再重复派发。
- 交互式 TOC 优先级高于 search/content 等批量任务。
- 同书并发 TOC 合并，避免重复刷新。
- Android 两端 TOC 使用独立队列，不再与搜索共用串行队列。

### 阅读页
- 新增 `reader-gesture.js`：横向位移至少 56px 且明显大于纵向位移才翻章；轻点左右 22% 热区也可翻章。
- 鼠标点击热区不触发手势翻章；文本选择、交互控件、目录、弹窗屏蔽手势。
- 保留原按钮和键盘路径，增加方向反馈，尊重 `prefers-reduced-motion`。
- 窄屏安全区、44px 控件、移动正文宽度和宽屏纸张式阅读栏已调整。

## 4. 本仓库中的修改与新增文件

### 服务端修改
- `legado-web/app/server/python/legado_bridge.py`
- `legado-web/app/server/src/books.js`
- `legado-web/app/server/src/busweights.js`
- `legado-web/app/server/src/pyengine.js`
- `legado-web/app/server/src/remote.js`
- `legado-web/app/server/src/server.js`
- `legado-web/app/server/src/store.js`
- `legado-web/cmd/install_init`
- `legado-web/cmd/main`

### 服务端新增
- `legado-web/app/server/src/parsepool.js`
- `legado-web/app/server/src/parse-worker.js`
- `legado-web/app/server/src/java-worker.js`
- `legado-web/app/server/src/java/FnReaderJavaWorker.java`
- `legado-web/app/server/python/toc_normalize.py`
- `legado-web/app/server/python/legado_rule/toc.py`

### 阅读页修改
- `legado-web/app/web/app.js`
- `legado-web/app/web/index.html`
- `legado-web/app/web/style.css`

### 阅读页新增
- `legado-web/app/web/reader-gesture.js`

### 测试修改/新增
- 修改：`test/store-guard.js`
- 新增：`test/debug-remote.js`
- 新增：`test/java-python-engine.js`
- 新增：`test/parse-concurrency.js`
- 新增：`test/softbus-queue.js`
- 新增：`test/reader-gesture.js`

### 构建与仓库新增
- `tools/build-debug.sh`
- `.github/workflows/build-debug-fpk.yml`
- `README-ALPHA.md`
- `ALPHA_HANDOFF.md`
- `.gitattributes`（FPK/APK/ZIP 走 Git LFS）

### Android 改动
完整修改文件放在原路径结构下：

- `patches/yuedu-pro/app/src/main/java/io/legado/app/model/FnReaderRemote.kt`
- `patches/yuedu-pro/app/src/main/java/io/legado/app/model/FnReaderToc.kt`
- `patches/yuedu-pro/app/src/main/java/io/legado/app/service/FnReaderBridgeService.kt`
- `patches/yuedu-android4/` 下与 Pro 相同的三个相对路径文件

应用到客户端仓库时，将 `patches/<client>/` 内容覆盖到对应仓库根目录。不要直接把本仓库的完整 Android 工程当作独立客户端构建。

## 5. 构建与 Actions

`tools/build-debug.sh` 行为：

1. 生成独立 debug manifest：`legado-web-debug / 0.1.0 / 1123`，显示名 `阅读-debug`。
2. 从源码执行 esbuild 重新生成 `server.bundle.js`（target node26），然后组装服务端和 Web 阅读页。
3. 编译 Java TOC worker。
4. 内置 Node 26 和 `libatomic`，复验 Node 可执行。
5. jlink Java 21 运行时，启动 worker 并强校验 `virtualThreads:true`。
6. 内置 Python 3.11，复制标准库并解引用绝对符号链接；执行 Python probe。
7. fnpack 生成 debug FPK，并校验 tar 内关键结构。

Actions 文件 `.github/workflows/build-debug-fpk.yml`：

- 触发：main push（仅相关路径）或手动 `workflow_dispatch`。
- Node `26.13.1`、Temurin JDK 21、Python 3.11。
- 下载官方 fnpack `1.2.3`。
- 从源码执行 `tools/build-debug.sh`。
- 上传 Artifact `legado-web-debug-0.1.0-fpk`。
- 不创建 Release、不打 tag、不写正式 1122。

GitHub runner 相关环境变量由 workflow 注入：

```bash
FNREADER_NODE_HOME=$(dirname "$(dirname "$(command -v node)")")
FNREADER_JAVA_HOME=$JAVA_HOME
FNREADER_PYTHON=$(command -v python3.11)
FNREADER_PYTHON_STDLIB=${pythonLocation}/lib/python3.11
```

本地/NAS 构建可用默认路径，或用以上环境变量覆盖。

## 6. 测试证据

本地 Node 26 回归：

- `reader-gesture.js`：12/12。
- `java-python-engine.js`：10/10。
- `parse-concurrency.js`：7/7；慢 `java.ajax` 期间健康检查 4ms；首次复杂搜索 2206ms（含冷启动），第二次 5ms，慢 URL 只请求一次。
- `softbus-queue.js`：7/7。
- `e2e.js`：35/35。
- `sse-worker.js`：5/5。
- `rule-syntax.js`：13/13。
- `replace-regex.js`：8/8。
- `store-guard.js`：6/6。

NAS debug 远程 `test/debug-remote.js`：6/6。

- 0.1.0 / Node 26 / V8 版本。
- 1150 条书源。
- 手势和 Java 设置资源部署。
- Python 池和 Java 21 虚拟线程健康。
- Java 开关关闭。
- Java 开关重新开启。

复测命令：

```bash
node test/debug-remote.js
bash tools/build-debug.sh
```

`test/debug-remote.js` 强制只允许 `10.5.4.70:1123`；不要改造成生产地址。

## 7. 产物

按用户最新要求，**产物不提交到本仓库**，也不使用 Git LFS 上传 FPK/APK/ZIP。需要 FPK 时运行 `build-debug-fpk` Actions 并下载 Artifact；本地最终产物仍保留在当前用户机器的 `D:\iot\FnReader\dist`，不属于 Git 仓库内容。不要再把产物强行加入 main。

## 8. 部署与排障记录

- fnOS `appcenter-cli install-fpk` 对已安装应用曾返回 installed 但没有替换文件；最终采用备份书源后卸载 debug、清空 debug 数据、全新安装、恢复书源。
- 全新安装需要指定 `--volume 4`。
- Python 包内绝对符号链接曾导致 fnOS `code 10107`；构建时已用 `cp -aL` 解引用并显式拒绝残留绝对链接。
- Java 精简运行时缺少 `jdk.httpserver` 曾导致 `com.sun.net.httpserver.HttpServer` 缺失；已加入模块并用 HTTP 健康探针验收。
- fnOS `appcenter-cli start` 的 `10500` 可能发生但进程实际正常；必须同时检查进程、1123 HTTP 和 `/api/health`。

部署相关命令中的 SSH 凭据必须通过 `FNOS_HOST/FNOS_USER/FNOS_PASS` 环境变量提供，禁止写进仓库或日志。只允许操作 `legado-web-debug`。不要对正式 1122 或正式应用执行相同命令。

## 9. 未完成/未宣称项

- 没有完成浏览器视觉走查和模拟器真实触摸手势验收；手势目前只有 JS 单元测试与资源部署验证。
- 最终 NAS 远程 smoke 覆盖版本、书源、资源和引擎开关，不等同于全部搜索/目录/正文业务回归。完整确定性回归在本地通过，需要在安全 debug 环境继续复跑。
- Java 21 虚拟线程在 NAS 已实测；本机开发 JDK 17 的兼容测试不等同 Java 21 验收。
- Android 两端编码已编译并安装，但尚未完成真实 UI 操作回归。
- 原三个仓库的改动仍未提交、未推送；本仓库是独立副本。

## 10. 建议的下一步

1. 等待并检查 `build-debug-fpk` Actions 是否成功，下载 Artifact 与 `dist` FPK 对比 SHA。
2. 用模拟器真实操作搜索、TOC、正文和手势翻页，补充自动化 UI 或截图证据。
3. 对 1150 条书源仅选少量稳定源做受控业务回归，避免大规模外部请求。
4. 若后续要正式发布，必须另开任务并再次确认正式版本、正式 manifest、tag、Release 和 1122 边界；不要把本 Alpha 仓库直接当正式发布流水线。
