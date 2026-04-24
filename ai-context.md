## 2026-04-24 COMPRESS 升级竞态修复记录

- 按 `manual-fix-plan.md` 将 `Client.Compress` 从 `Upgrade + Execute` 改为 `UpgradeWithCommand` 调用路径。
- 保留调试日志：
  - `imap/compress: deflate reader read called`
  - `imap/compress: deflate connection created`
- 当前仓库依赖 `github.com/emersion/go-imap/client`，需要对应依赖侧提供 `UpgradeWithCommand` 才能完成联调。
