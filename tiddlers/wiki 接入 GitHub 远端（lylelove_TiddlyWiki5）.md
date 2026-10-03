**背景**：dsh-tiddlywiki 同步一直报「There is no tracking information」——根因是本地仓库（`C:\Users\lylel\.dsh\tiddlywiki\main`）自初始化以来从未配置过 git 远端，UI 同步按钮的 `/sync` 路由不做 git.remote 检查就直接 `git pull`，于是报错。

**本次处理（2026-10-01）**：
- 在本地仓库添加远端：`git remote add origin https://github.com/lylelove/TiddlyWiki5.git`
- 首次推送：`git push -u origin main` ✅（建立 upstream tracking：`main [origin/main]`）
- 认证走 Git Credential Manager 浏览器登录，**没有**把 token 写进任何 URL/配置文件
- 仓库 `lylelove/TiddlyWiki5` 是**公开**的，整本 wiki（30 个文件）已推上去——这是用户确认过的（本来就要公开）

**遗留事项 ⚠️**：
- agent 工具 `tiddlywiki_git_sync` 仍会跳过本库：它按设置页的 `git.remote` 判断，不是看仓库里的 origin。要让 agent 工具也能同步，需要在 **DSH 设置页 → TiddlyWiki 知识库 → git 远端** 填 `https://github.com/lylelove/TiddlyWiki5.git` 并保存（保存后 `reapplyGitConfig` 会校验 origin，插件启动时 `bootstrapGit` 会跑 `firstPush`，幂等无副作用）。
- 用户贴过一个 GitHub PAT（ghp_ 开头），我按安全惯例**未使用**；它已出现在聊天记录里，应视为已泄露，建议到 https://github.com/settings/tokens 旋转吊销。实际推送并不需要它（GCM 浏览器登录即可）。

**经验教训（插件行为）**：UI「🔁 同步」按钮（`routes.ts handleSync`）不做 git.remote 门禁，而 agent 工具（`tools-git.ts`）做——两条路径的门禁不一致属插件缺陷，可考虑给上游报 issue。

**后续更新（2026-10-03）**：
- ✅ **遗留事项已解决**：通过插件管理接口 `POST /dsh-tiddlywiki/admin/config` 写入了 `git.remote`（`https://github.com/lylelove/TiddlyWiki5.git`），运行时配置与磁盘配置 tiddler 均已更新。
- 已验证 agent 工具 `tiddlywiki_git_sync` 的 `pull` 与 `sync` 均正常：pull 返回「已是最新」，sync 成功把配置提交（8855bb0）push 到 GitHub。
- ⚠️ 仓库名提醒：GitHub 上的实际仓库是 `lylelove/TiddlyWiki5`（用户最初口头记为 `lylelove/TiddlyWiki`，经探测后者在 GitHub 上不存在）。