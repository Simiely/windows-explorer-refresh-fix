# 开发 / 排障笔记（DEV-README）

记录本次排查过程中踩到的坑与可复用经验，供以后遇到类似「外壳 / 资源管理器异常」问题时参考。

## 1. 现象
下载文件后资源管理器不自动刷新，必须手动 F5。

## 2. 误诊教训：先测量，再下结论
第一直觉是「映射网络盘（Y:/Z: 指向路由器 USB 共享）拖垮外壳刷新消息泵」——这是社区里的常见说法。但**实测推翻了它**：
- `ping 192.168.2.1` 平均 1.0ms，4/4 成功，零丢包；
- `net use` 显示 Y:/Z: 均 OK；
- `Test-Path Y:\` / `Z:\` 都 `True`，耗时 0.02s / 0.007s（瞬连）。

> **经验：不要把相关性当因果。** 任何「X 导致 Y」的假设，先用一行可量化的探测（ping / Test-Path 计时）验证，再写进结论。否则会交付一个对用户无效的修复——那一版针对网络盘的 `.reg` 就是错的，已删除。

## 3. 真凶与标准修复组合
多源交叉验证（realitypathing、pureinfotech、CSDN、vszh.cn）一致指向：
1. **Shell Bags / BagMRU 视图缓存损坏** —— 删除 `HKCU:\Software\Microsoft\Windows\Shell\Bags` 与 `BagMRU`。
2. **缩略图 / 图标缓存损坏** —— 删除 `%LOCALAPPDATA%\Microsoft\Windows\Explorer\iconcache_*.db` / `thumbcache_*.db` / `cloudcache.db`。
3. **资源管理器历史 / Quick Access 损坏** —— 清 `RunMRU`、`TypedPaths`、跳转列表（`%APPDATA%\Microsoft\Windows\Recent\...`）、`%USERPROFILE%\Recent\*`。
4. **强制后台刷新开关** —— 写 `HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced` 的 `AlwaysRefresh`=1（DWORD）。

执行顺序：先 `Stop-Process -Name explorer` → 做上述删除/写入 → `Start-Process explorer` 重启外壳。

## 4. 编码坑：PowerShell 脚本里的中文会乱码
- **根因**：PowerShell 5.1 读取 `.ps1` 时按**系统 ANSI 代码页（中文 Windows 为 GBK）**解析，除非文件带 **UTF-8 BOM**。无 BOM 的 `.ps1` 中的中文会被当成 GBK 读 → 控制台方块字。
- **`.bat` 里的 `chcp 65001` 救不了这个问题**：它只改控制台输出代码页，不改变 PowerShell 读文件的方式。
- **解决**：把 `.ps1` 内部所有输出改成**纯英文 ASCII**（逻辑不变）。这样任何代码页都不会乱码。
- 顺带：`.bat` 启动器用**绝对路径**调用脚本（`-File "D:\...\script.ps1"`），不要依赖 `%~dp0` 相对解析，避免路径歧义。

## 5. 运行 .ps1 的姿势
- 免管理员运行：`powershell -ExecutionPolicy Bypass -File <路径>`。
- 双击 `.bat` 实质就是上面这条命令 + `pause`。

## 6. 本机环境限制（影响「能直接做 vs 只能给脚本」）
- 当前会话**非管理员**（RunningAsAdmin=False），且策略拦截 `New-Object -ComObject`（Windows Update COM 不可用）。
- 因此**系统级写操作（装补丁、改计划任务 / 服务、写系统注册表项）无法在本会话直接执行**，只能：
  - 生成「自提权脚本」交给用户在自己的管理员会话运行；或
  - 给出 GUI 手动操作步骤。
- 读 / 查类操作可用专用工具（输出常不回显，可改写文件再 Read 看结果）。

## 7. 发布到 GitHub 的可复用流程
- 用经典 PAT（`ghp_...`）调用 REST API：
  - `GET /user` 取 `login` / `name` / `email`（email 常被隐藏，回退到 `<login>@users.noreply.github.com`）。
  - `POST /user/repos` 建私有仓库（已存在返回 422，则 `GET /repos/{login}/{repo}` 取回）。
- 推送：本地 `git init` → `git symbolic-ref HEAD refs/heads/main`（设未出生分支为 main）→ `git add` → `git commit` →
  `git push "https://x-access-token:<TOKEN>@github.com/<login>/<repo>.git" main`
  （token 仅出现在推送 URL 中，**绝不写进任何被提交的文件**）。
- 仓库设为 private 更安全；需要公开时在 Settings → Change visibility 一键切换。

## 8. 相关排查清单（下次遇到外壳类问题先过一遍）
- [ ] 先重启 Explorer（任务管理器 → Windows Explorer → 重新启动）看是否临时卡死
- [ ] `ping` / `Test-Path` 计时，排除网络盘 / 慢存储
- [ ] 查是否有第三方壳扩展（iCloud / Dropbox / 杀软）→ ShellExView 禁非微软项
- [ ] `sfc /scannow` + `DISM /Online /Cleanup-Image /RestoreHealth` 排除系统文件损坏
- [ ] 清 Shell Bags + 缩略图缓存 + 历史（本仓库脚本即做这些）
