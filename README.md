# technical-notes

技术笔记归档。这里收集在开发/运维/折腾中沉淀下来的可复用文档，方便后续学习、维护和复现。

> 来源：最初散落在 OpenClaw 的 agent workspace，2026-08-11 统一整理到本目录，准备提交到 git 服务长期保存。

## 文档索引

| 文档 | 主题 | 日期 | 状态 |
|---|---|---|---|
| [cloudflared-tunnel-macos_20260810.md](cloudflared-tunnel-macos_20260810.md) | Mac + cloudflared 多端口隧道 + launchd 常驻（含 QUIC 被掐改 http2、codebuddy 端口/WebSocket 坑、eu.org 免费域名申请） | 2026-08-10 | ✅ 已闭环 |
| [flutter_json_parse_perf_20260810.md](flutter_json_parse_perf_20260810.md) | Flutter JSON 解析性能优化 | 2026-08-10 | — |

> 收纳标准：仅限**对用户本人有学习/使用价值的文档**。任务产物摘要（`task-summary_*.md`）、agent 自身学习笔记（如 rms-log-decrypt 的 learn 笔记）等只服务于 agent 运行的内容，不归入本目录，留在 OpenClaw workspace。

## 目录约定

- 命名格式：`<主题>_<YYYYMMDD>.md`，便于按时间排序和定位。
- 新增笔记建议沿用同款命名，并在上表登记一行，保持索引可查。

## Git

本目录为独立 git 仓库，仅纳管技术笔记，不影响 `Documents/work` 下的其他内容。

```bash
git init            # 首次
git add .
git commit -m "init: technical-notes 归档"
# 关联远程后：
git remote add origin <repo-url>
git push -u origin main
```
