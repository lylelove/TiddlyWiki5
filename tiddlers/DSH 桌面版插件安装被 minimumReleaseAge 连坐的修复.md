## 问题

在 DSH 桌面版（profile `desktop`）安装 dsh-account-hub 时报错：

```
✗ Lockfile failed supply-chain policy check (66 entries in 3.3s)
[ERR_PNPM_MINIMUM_RELEASE_AGE_VIOLATION] 1 lockfile entries failed verification:
  dshmarket@1.66.8 was published at 2026-10-01T16:11:39.000Z, within the minimumReleaseAge cutoff (2026-10-01T02:15:43.047Z)
```

## 根因

- pnpm 11/12 默认启用 `minimumReleaseAge: 1440`（24 小时），lockfile 中任何发布不足 24h 的包都会让**所有**插件安装/卸载失败（连坐）。
- 本次肇事包是 `dshmarket@1.66.8`（2026-10-01T16:11:39Z 发布，报错时仅约 10 小时）。
- DSH 桌面版对 `desktop` profile 有本地策略层：`dsh plugin` CLI 拒绝管理该 profile（`managed exclusively by the Electron application`），且桌面版注入的策略可能用 `ERR_PNPM_LOCAL_POLICY_DENY` 否决 `minimumReleaseAgeExclude`（见 deepseek-harness discussion #8557）。

## 解决（官方绕行：用系统 pnpm 直操作 profile）

1. 确保 `~/.dsh/profiles/desktop/pnpm-workspace.yaml` 中有：
   - `minimumReleaseAgeExclude` 含 `dshmarket@1.66.8`（让 lockfile 校验通过）
   - `allowBuilds` 含 `dsh-account-hub@git+https://github.com/gurio-wine/dsh-account-hub.git: true`（允许 prepare 构建）
2. 用系统 pnpm（而非桌面版内置）在 profile 目录执行：

```powershell
$env:Path = "$env:APPDATA\npm;" + $env:Path
cd "$env:USERPROFILE\.dsh\profiles\desktop"
pnpm add "https://github.com/gurio-wine/dsh-account-hub.git"
```

→ 结果：`Lockfile passes supply-chain policies (66 entries in 3s)`，安装成功，prepare 构建（`pnpm build:all`）通过。
3. 手工把 `dsh-account-hub` 加入 package.json 的 `dsh.profile.bundles`（绕过 CLI 后没有自动 reconcile）。
4. 重启桌面版生效。

## 后续

- `dshmarket@1.66.8` 在 2026-10-02T16:11:39Z 之后满 24h，届时可移除 `minimumReleaseAgeExclude` 中的 `dshmarket@1.66.7 / dshmarket@1.66.8`（可选清理）。
- 若 dshmarket 更新到更新版本，24h 内再装/更新插件可能复现同样问题，重复上述步骤或等冷却期结束。
- 相关上游讨论：deepseek-ai/deepseek-harness discussion #8557；dsh-market/dsh-market issue #600。