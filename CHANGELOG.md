# CHANGELOG.md · 版本记录

> 项目无正式版本号，按提交里程碑记录。

## 里程碑

- **接入文档检查（2026-11-17）**：接入 knowledge-base 的**单项目规范**（spec 2.1.3 / repo-kit 2.7.2，profile `link-only`）；新增 `schema/taxonomy.json`、`tools/repo-kit/`、`tools/update-repo-kit.ps1`、`schema/kit.lock.json` 与 CI；只查链接与入口，**不要求 frontmatter**；7 个相对链接纳入检查；验收：`REPO CHECK: PASS`（ERROR=0）、`KIT LOCK: PASS (v2.7.2, spec=2.1.3)`
- **GitHub Pages 落地页**：`index.html` 上线（仓库主页可直接预览）
- **核心脚本完成**：`reset-explorer-view.ps1`（纯 ASCII 编码，避免 GBK 乱码）+ `run-explorer-fix.bat` 启动器
- **文档完善**：README（使用说明）+ DEV-README（排障笔记，含误诊教训与性能影响交叉验证）+ explorer-refresh-guide（图文指南）
- **经验沉淀**：网络盘假设被实测推翻，真凶为 Shell Bags 缓存损坏；多源交叉验证修复组合

## 备注

- 无版本 tag / Release；交付物为脚本 + 指南文档
