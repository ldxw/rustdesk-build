# rustdesk-build

从上游 [rustdesk/rustdesk](https://github.com/rustdesk/rustdesk) 拉取指定版本的 tag，在本仓库创建**同名分支**存放（例如分支 `1.5.3` 对应上游 tag `1.5.3`）。

## 工作流

文件：`.github/workflows/sync-upstream-tags.yml`

| 项 | 说明 |
| --- | --- |
| 触发方式 | 仅手动触发（`workflow_dispatch`），没有定时任务 |
| 上游仓库 | `https://github.com/rustdesk/rustdesk.git`（写在 `env.UPSTREAM_REPO`，改这里即可换源） |
| 权限 | `contents: write`，用 `github.token` 推送，无需额外 secret |
| 并发 | 同组 `sync-upstream-tags`，不取消进行中的运行 |

## 使用方法

1. 打开仓库的 **Actions** 页面。
2. 左侧选择 **Sync upstream tag to branch**。
3. 点击 **Run workflow**，填写参数：

| 参数 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `version` | 字符串 | `latest` | 要同步的版本，如 `1.5.3`；也接受 `v1.5.3`。留空或填 `latest` 表示取上游最新版本 tag |
| `force` | 布尔 | `false` | 强制拉取。同名分支已存在且指向不同 commit 时：关闭 = 跳过并提示，开启 = 强制更新到上游 commit |

4. 运行结束后查看该次运行的 **Summary**，里面有目标版本、上游 commit、分支名、是否强制、结果。

## 运行结果

| 结果 | 含义 |
| --- | --- |
| 新建分支 | 远端没有同名分支，已创建并指向该 tag 的 commit |
| 已存在且一致 | 远端分支已指向同一个 commit，无需操作 |
| 已存在，未更新 | 远端分支指向不同 commit 且 `force=false`，跳过；需要覆盖请勾选强制拉取重跑 |
| 已强制更新 | 远端分支指向不同 commit 且 `force=true`，已用 `--force-with-lease` 更新 |

## 行为细节

- **只识别版本 tag**：正则 `^[0-9]+\.[0-9]+\.[0-9]+(-[0-9]+)?$`，即 `x.y.z` 或 `x.y.z-N`；上游的 `nightly`、`fdroid-version` 会被忽略。
- **最新版本判定**：对全部版本 tag 做 `sort -V`，取排序末尾。当前上游最新为 `1.5.3`。
- **版本不存在**：任务失败，并在 Summary 中倒序列出最近 20 个可用版本。
- **只拉目标 tag**：`git fetch --no-tags upstream refs/tags/<version>`，不会遍历全部 tag。
- **注解 tag 处理**：用 `git rev-parse FETCH_HEAD^{commit}` 解引用到 commit，推的是 commit 而非 tag 对象。
- **覆盖保护**：强制更新使用 `--force-with-lease=<branch>:<远端旧 SHA>`，若期间有人推了新内容会失败而不是覆盖。
- **空仓库可用**：不使用 `actions/checkout`（无 commit 时会失败），改为 `git init` + 手工配置 remote。
- **凭据处理**：token 通过 `git config --local http.https://github.com/.extraheader` 注入，不出现在 URL、命令行参数和日志里，并做了 `::add-mask::`。

## 常见操作

拉取上游最新版本：

```
version: latest
force:   false
```

拉取指定版本：

```
version: 1.5.1
force:   false
```

把已存在的分支纠正到上游对应 commit：

```
version: 1.5.1
force:   true
```

## 更换上游或换仓库

- 换上游：修改 `.github/workflows/sync-upstream-tags.yml` 中 `jobs.sync.env.UPSTREAM_REPO`。
- 推送目标固定为当前仓库（`github.repository`），不需要改。
