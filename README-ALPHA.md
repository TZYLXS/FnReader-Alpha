# FnReader-Alpha

这是 FnReader `阅读-debug 0.1.0` 的独立 Alpha 仓库，保留原 FnReader 根目录结构。

- 服务端 debug：`legado-web-debug`，版本 `0.1.0`，端口 `1123`。
- 运行时：Node.js `v26.13.1`、Java 21 虚拟线程、Python 3.11 持久进程池。
- 正式包名、正式版本、生产 1122 不属于本仓库发布目标。
- Android 客户端本轮修改位于 `patches/yuedu-pro/`、`patches/yuedu-android4/`，覆盖到对应客户端仓库即可。
- 本轮产物不提交到 Git 仓库；GitHub Actions `build-debug-fpk` 在 main 推送或手动触发时，从源码执行 `tools/build-debug.sh` 并上传 FPK Artifact，不创建 Release、不打 tag。本地最终产物保留在当前用户的 `D:\iot\FnReader\dist`，不属于本仓库内容。

后续接手的 AI 请先阅读 [`ALPHA_HANDOFF.md`](ALPHA_HANDOFF.md)。
