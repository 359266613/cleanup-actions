# GitHub Actions 构建记录清理 —— 操作步骤

目标账号：https://github.com/359266613 （个人账号，非组织）
目标效果：清掉该账号名下仓库的 Actions 运行记录与 artifact，**每个仓库的每个工作流只留最新 3 条历史 + 当前 1 条 = Actions 页显示 4 条**，从而释放 artifact 存储配额。

当前工作目录已就位的内容：

```
E:\opencode\7.清理仓库构建记录\
└─ .github\
   └─ workflows\
      └─ cleanup-other-repos.yml   ← 清理工作流（无需再改）
```

> 该目录目前**还不是 git 仓库**，下面的步骤会新建仓库并推送。

---

## 1. 用 359266613 账号新建一个私有仓库

网页打开 https://github.com/new （确认右上角登录的是 **359266613**）：

- Repository name：`cleanup-actions`
- 可见性：**Private**
- 不要勾选 “Add a README file”“.gitignore”“license”（避免推送冲突）
- 点 **Create repository**

---

## 2. 生成 Personal Access Token（同账号）

### 方式 A：Classic PAT（简单，推荐）

1. 头像 → **Settings** → 左侧最下 **Developer settings**
2. **Personal access tokens** → **Tokens (classic)** → **Generate new token (classic)**
3. 填写：
   - Note：`cleanup-actions`
   - Expiration：`30 days`（到期重做一次即可）
   - Scopes：**只勾 `repo`**（classic 没有独立的 Actions 权限，`repo` 已包含 Actions 读写）
4. **Generate token** → 复制 `ghp_xxxxxxxx`（页面只显示一次，先存到记事本）

### 方式 B：Fine-grained PAT（权限更细）

1. 同上路径，选 **Tokens (fine-grained)** → Generate new token
2. Resource owner：**359266613**（必须选这个账号）
3. Repository access：**All repositories**
4. Permissions：
   - **Actions**：`Read and write`
   - **Metadata**：`Read-only`（自动带，不用改）
5. Generate → 复制 `github_pat_xxxxxxxx`

> 说明：必须用 359266613 自己的 token。仓库自带的 `GITHUB_TOKEN` 只能操作当前那一个仓库，无法跨仓库清理。

---

## 3. 把工作流推到仓库（PowerShell）

```powershell
cd "E:\opencode\7.清理仓库构建记录"

git init
git branch -M main
git add .
git commit -m "chore: add actions cleanup workflow"
git remote add origin https://github.com/359266613/cleanup-actions.git
git push -u origin main
```

推送弹窗里的认证信息：

- Username：`359266613`
- Password：粘贴第 2 步的 **PAT**（不是 GitHub 登录密码）

不想用命令行也可以：用 GitHub Desktop 或 VS Code 登录 359266613 后直接 Publish / Push，效果一样。

推送成功后确认文件在正确位置：
https://github.com/359266613/cleanup-actions/blob/main/.github/workflows/cleanup-other-repos.yml

> 关键点：workflow 文件必须在**默认分支**的 `.github/workflows/` 下，否则 Actions 页面不会出现 “Run workflow” 按钮。若用网页上传，注意保留 `.github` 这一层目录。

---

## 4. 添加 Secret

打开 https://github.com/359266613/cleanup-actions/settings/secrets/actions

**New repository secret**

| 字段 | 值 |
|---|---|
| Name | `CLEANUP_TOKEN` |
| Secret | 第 2 步的 PAT（`ghp_...` 或 `github_pat_...`） |

点 **Add secret**。名字必须严格是 `CLEANUP_TOKEN`（工作流里靠这个名字取）。

---

## 5. 先跑一次 dry-run（只预览，不删除）

打开 https://github.com/359266613/cleanup-actions/actions/workflows/cleanup-other-repos.yml

点右侧 **Run workflow**，填：

| 输入项 | 值 | 说明 |
|---|---|---|
| repos | `359266613/*` | `owner/*` 会展开成该账号下全部仓库（含私有） |
| keep_runs | `3` | 每个仓库、每个工作流保留最新 3 条历史 |
| run_max_age_days | `30` | 超过 30 天的记录无条件删除；填 `0` = 只按条数保留 |
| artifact_max_age_hours | `24` | 保留的 run 里，超过 24 小时的产物也删；填 `0` = 不动 |
| dry_run | **true** | 仅打印清单 |

运行结束后打开这次 run 的 **Summary** 页，会有一张表：

| 仓库 | 运行记录 | 删除运行 | 产物 | 删除产物 | 实际释放 |
|---|---|---|---|---|---|
| 359266613/xxx | 42 | 36（预览） | 40 | 36（预览） | 1200.50 MB |

同时 Job 日志里有 `[dry-run] 将删除 run #12（xxx，2026-10-01T..., push）` 之类的逐条明细。**先核对这张表**，尤其是仓库数量是否符合预期（日志开头会打印 `共 N 个目标仓库：...`）。

---

## 6. 正式执行

再点一次 **Run workflow**，参数同上，只把 `dry_run` 改成 **false**。

跑完后 Summary 表格里的“删除运行/删除产物”就是实际删除数量。

---

## 7. 验证结果

1. 打开 https://github.com/359266613?tab=repositories 任选一个仓库 → **Actions** 标签页：
   - 每个工作流只剩 4 条（最新 1 条 + 3 条历史）
2. 仓库 → **Actions** → 右侧 **Artifacts**（或 run 详情页底部）：只剩 24 小时内产生的产物。
3. 存储配额页面（https://github.com/settings/billing）数字**不会立刻下降**，GitHub 每 6–12 小时重算一次；但配额一旦释放，新上传会马上恢复。

删除的影响范围：

- 只删 Actions 运行记录、日志、artifact。
- **不删**代码、提交、分支、tag、Release、已发布的 Pages 站点、仓库设置。

---

## 8. 可选：改成定时自动清理

如果打算每周自动跑一次，把 `.github/workflows/cleanup-other-repos.yml` 顶部的 `on:` 改成：

```yaml
on:
  workflow_dispatch:
    inputs:
      # ... 原有输入保持不变 ...
  schedule:
    - cron: '0 20 * * 0'   # 每周一 04:00（北京时间）
```

定时触发时没有输入参数，所以脚本里这一行要加默认值：

```js
const repos = (core.getInput('repos') || '359266613/*')
  .split(/[\s,]+/)
  .map((s) => s.trim())
  .filter(Boolean);
```

建议定时任务里保持 `dry_run: false` 的实际删除语义（脚本中 `dry_run` 缺省值是 true，定时运行时 `core.getInput('dry_run')` 为空 → 会按 true 走，只预览）。如需定时真删，把该行改成：

```js
const dryRun = String(core.getInput('dry_run') || 'false').toLowerCase() === 'true';
```

---

## 9. 常见问题

| 现象 | 原因 / 处理 |
|---|---|
| Actions 页没有 “Run workflow” 按钮 | workflow 文件不在默认分支的 `.github/workflows/` 下，或仓库的 Actions 被禁用（Settings → Actions → Allow all actions） |
| 日志：`没找到 359266613 的仓库` | `CLEANUP_TOKEN` 不是 359266613 的 PAT，或 fine-grained 没选 All repositories |
| `403 Resource not accessible by personal access token` | classic 没勾 `repo`；或该仓库已归档（归档仓库不能删 run，只能忽略） |
| `repos 不能为空` | Run workflow 时 `repos` 没填 |
| 删完后配额没变化 | 正常，6–12 小时重算；可先看 Summary 里的“释放 MB”是否为 0 |
| 提示 `Artifact storage quota has been hit` 仍然存在 | 说明配额还没重算，或还有其它账号/组织在占用账户配额；等重算后再跑构建 |
| 跑得慢 / 超时 | 仓库多时属正常（分页 + 限速），可把 `repos` 拆成几个批次跑，或把 `run_max_age_days` 设大一些减少 API 调用量 |
| 想只清某个仓库 | `repos` 填 `359266613/仓库名` |
| 想连 org 一起清 | `repos` 填 `359266613/*, 组织名/*`（组织需 token 有权限，classic PAT 可能要做 SSO 授权） |

---

## 10. 收尾（安全）

- 清理完成后，如不想保留该 token：Settings → Developer settings → Personal access tokens → 找到 `cleanup-actions` → **Delete / Revoke**，同时删掉仓库里的 `CLEANUP_TOKEN` secret。
- 该仓库本身也可以设为 private 后保留，以后需要清理时直接再点一次 Run workflow（保留 token 更省事）。
