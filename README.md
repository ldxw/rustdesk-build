# rustdesk-build

从上游 [rustdesk/rustdesk](https://github.com/rustdesk/rustdesk) 拉取指定版本的 tag，在本仓库创建**同名分支**存放（例如分支 `1.5.3` 对应上游 tag `1.5.3`）。

## 工作流

本仓库有三个工作流：两个手动触发，一个在构建完成后自动触发。

| 文件 | 用途 |
| --- | --- |
| `.github/workflows/sync-upstream-tags.yml` | 从上游拉取指定版本的 tag，在本仓库创建同名分支存放源码 |
| `.github/workflows/tag.yml` | 给上游 commit 打同名 tag，触发上游自带的 Flutter 构建（见「打标签构建」） |
| `.github/workflows/publish-release-assets.yml` | Flutter Tag Build 成功后发布签名更新元数据，并刷新 `update-latest` 发现入口（见「软件更新地址」） |

下面「使用方法 / 运行结果 / 前置设置 / 失败排查 / 行为细节 / 常见操作」描述的是**同步工作流**
`.github/workflows/sync-upstream-tags.yml`；打标签构建与更新元数据发布各自单独成节。

同步工作流的配置：

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

## 前置设置

### 1. 配置 PAT（必须）

上游 `rustdesk/rustdesk` 的 tag 树里带有 `.github/workflows/*` 文件（1.5.3 有 11 个）。
GitHub 规定 **GitHub App token 不能推送新增或修改 `.github/workflows/*` 的提交**，报错形如：

```
refusing to allow a GitHub App to create or update workflow `.github/workflows/ci.yml` without `workflows` permission
```

`github.token` 就是 GitHub App token；而 YAML 的 `permissions:` 键里没有 `workflows` 这个 scope
（合法值只有 actions / checks / contents / deployments / id-token / issues / packages / pages /
pull-requests / security-events / statuses 等），所以**无法在工作流里授予该权限**。因此必须用 PAT。

创建方式（二选一）：

| 类型 | 权限 |
| --- | --- |
| classic PAT | 勾选 `repo`（整段）+ `workflow` |
| fine-grained PAT | Repository access 选中本仓库；Permissions 里 **Contents: Read and write**、**Workflows: Read and write** |

然后在本仓库 **Settings → Secrets and variables → Actions → New repository secret**，
名称填 **`SYNC_TOKEN`**，值填这个 PAT。

工作流里写的是 `${{ secrets.SYNC_TOKEN || github.token }}`：配了 secret 就用 PAT，没配才退回
`github.token`（并打一条 warning，push 时会被拒，Summary 里给出明确原因）。

### 2. 允许 Actions 写仓库

仓库级 **Settings → Actions → General → Workflow permissions** 选择 **Read and write permissions**。

新建的个人仓库默认是只读。工作流第 2 步「预检 token 权限」会先调
`GET /repos/{owner}/{repo}` 检查 `permissions.push`，只读时立刻失败并在 Summary 里给出修改路径，
不会等 fetch 完成才在 push 处报 403。

## 失败排查

任何一步失败都会把原因写进该次运行的 **Summary**（不依赖下载日志），格式：

| 项 | 值 |
| --- | --- |
| 目标版本 | `1.5.3` |
| 失败阶段 | git fetch / git rev-parse / git ls-remote / git push（新建）/ git push（强制） |
| 原因 | 具体错误，含命令最后 3 行输出 |

- **push 被拒且提到 workflow**：说明用的是 `github.token` 而不是 PAT。GitHub App token 不能推送含 `.github/workflows/*` 的提交，上游 tag 树里恰好带着这些文件。按上面「前置设置 1」配置 `SYNC_TOKEN` 后重跑即可。

## 行为细节

- **只识别版本 tag**：正则 `^[0-9]+\.[0-9]+\.[0-9]+(-[0-9]+)?$`，即 `x.y.z` 或 `x.y.z-N`；上游的 `nightly`、`fdroid-version` 会被忽略。
- **最新版本判定**：对全部版本 tag 做 `sort -V`，取排序末尾。当前上游最新为 `1.5.3`。
- **版本不存在**：任务失败，并在 Summary 中倒序列出最近 20 个可用版本。
- **只拉目标 tag**：`git fetch --no-tags upstream refs/tags/<version>`，不会遍历全部 tag；网络抖动时重试 3 次（间隔 5 秒）。
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

## 打标签构建

文件：`.github/workflows/tag.yml`

同步工作流产出的是**分支**，而 `push: tags` 事件只认 tag，所以另有一个工作流专门负责「打 tag」。
它不自己定义构建，构建交给上游 tag commit 里自带的 `.github/workflows/flutter-tag.yml`。
它与同步工作流互不依赖：自己解析版本、自己解析 hbb_common ref、自己构造派生 commit 并推送 tag。

### 为什么打在 tag 上就能跑上游构建

GitHub 在处理 `push: tags` 事件时，读取的是 **tag 指向的 commit** 里的工作流文件，而不是默认分支上的。
`flutter-tag.yml` 里 `uses: ./.github/workflows/flutter-build.yml` 也解析到同一个 commit。
同步过来的上游 commit 本身已经带着这两个文件（以及它们依赖的 `bridge.yml`、
`third-party-RustDeskTempTopMostWindow.yml`），因此：

- main 分支上不需要复制任何上游构建定义，不存在 3000 行 YAML 的维护漂移；
- 上游更新构建逻辑后，重新同步一次即可自动用上新版本。

### 使用方法

1. Actions → **Tag build (trigger upstream Flutter build)** → **Run workflow**。

| 参数 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `version` | 字符串 | `latest` | 要构建的版本，如 `1.5.3`（也接受 `v1.5.3`）。留空或填 `latest` 取上游最新版本 tag |
| `force` | 布尔 | `false` | tag 已存在但指向不同 commit 时是否强制覆盖 |
| `hbb_common_ref` | 字符串 | `latest` | `libs/hbb_common` 子模块使用的 ref，可填分支 / tag / commit。`latest` 表示取 fork 上最新的版本 tag（`x.y.z` / `x.y.z-N`）；fork 没有任何版本 tag 时退回 fork 的默认分支（当前是 `main`） |
| `admin_url` | 字符串 | 空 | 客户端内置的 **web 面板地址**，如 `https://admin.example.com`。留空取 secret `ADMIN_URL`，再空则用内置默认 `https://admin-hy.rs.ldxw.top`；填 `none` 表示不改、保持上游默认（见「web 面板地址」） |

> 更新链没有单独的输入：客户端更新固定取自 `github.repository`（即本仓库），
> 公钥取自 `vars.RUSTDESK_UPDATE_ED25519_PUBLIC_KEY`，详见「软件更新地址」。

2. 运行结束后看该次运行的 **Summary**：目标版本、上游 commit、hbb_common 指向、web 面板地址、tag
   指向的派生 commit、tag 名、是否强制、结果。

工作流做的事：

1. 校验凭据。没有 `SYNC_TOKEN`（PAT）就直接失败——原因见下。
2. 查上游版本 tag 列表，`sort -V` 取最新，或按 `version` 精确匹配；版本不存在时列出最近 20 个可用版本。
3. 解析 hbb_common ref：`latest` 时查 fork 的版本 tag 列表，`sort -V` 取最新；
   fork 没有版本 tag 时用 `git ls-remote --symref <fork> HEAD` 解析默认分支（取不到则用 `main`）；
   手填的分支 / tag / commit 原样使用（会去掉 `refs/tags/`、`refs/heads/` 前缀）。
4. 只 fetch 目标 tag（`git fetch --no-tags upstream refs/tags/<version>`），
   `git rev-parse FETCH_HEAD^{commit}` 解引用到上游 commit。
5. 构造派生 commit：以该上游 commit 为父，按下面各节的规则改写若干文件，tag 打在派生 commit 上：
   `.gitmodules` 里 `libs/hbb_common` 的 url 与对应 gitlink、`src/common.rs` 的 web 面板地址、
   `src/common.rs` 的更新发现逻辑、`src/updater/artifact.rs` 的下载前缀与 owner/repo 守卫、
   `src/update_metadata.rs` 的可信公钥，以及两处 Flutter 里的下载地址。每一项都有开关与幂等判断，
   见下面各节。
6. 对比远端同名 tag：

| 远端状态 | 行为 |
| --- | --- |
| 不存在 | `git push origin <commit>:refs/tags/<version>`，触发构建 |
| 已存在且指向同一 commit | 跳过（GitHub 不会产生新事件） |
| 已存在且指向不同 commit，`force=false` | 跳过并在 Summary 提示 |
| 已存在且指向不同 commit，`force=true` | `--force-with-lease` 覆盖后触发构建 |

### hbb_common 子模块指向 fork

上游 `.gitmodules` 把 `libs/hbb_common` 指向 `https://github.com/rustdesk/hbb_common`。
本项目要产出连自己服务器的客户端，因此构建时必须换成
`https://github.com/ldxw/rustdesk_hbb_common`（该 fork 的 `src/config.rs` 里
`RENDEZVOUS_SERVERS` / `RS_PUB_KEY` 已改成自建服务器）。

子模块 URL 的唯一来源是**被 checkout 的 commit 里的 `.gitmodules`**：上游
`flutter-build.yml` 里所有 `actions/checkout` 都带 `submodules: recursive`，它按这个文件克隆。
所以只改仓库设置或 workflow 环境变量都无效，必须让 tag 指向一个 `.gitmodules` 已被改写的 commit。

做法是打 tag 前先派生一个新 commit：

| 项 | 值 |
| --- | --- |
| 父 commit | 上游 tag 指向的 commit |
| 改动 1 | `.gitmodules` 中 `[submodule "libs/hbb_common"]` 段的 `url` 改为 fork 地址 |
| 改动 2 | `libs/hbb_common` 的 gitlink 指向 `hbb_common_ref` 解析出的 commit |
| 其余 | 整棵树照抄上游 commit，`flutter-build.yml` 等文件字节不变 |
| 提交信息 | `build: point libs/hbb_common at <fork>@<ref> (<sha>)`，后面按实际改写项追加 `build: set client api server to <地址>` 与 `build: use <UPDATE_REPO> for client updates` 两段（段与段之间空一行） |

派生 commit 的 author / committer / 日期全部固定（日期取上游 commit 的 committer 时间），
提交信息里带 hbb_common 的 commit SHA，因此同样输入得到同样的 SHA。重跑时若远端 tag
已指向该 SHA，走「已存在且一致」分支，不会误报冲突。

三个 ref 怎么选：

| `hbb_common_ref` | 效果 |
| --- | --- |
| `latest`（默认） | 取 fork 上最新的版本 tag（`x.y.z` / `x.y.z-N`，`sort -V` 比较）；fork 没有版本 tag 时退回默认分支 `main` |
| `main` | 用 fork 默认分支，含自定义服务器配置，同时带上 fork 比 1.5.3 pin 多出的几个 upstream 提交（`src/tcp.rs`、`src/websocket.rs`） |
| 上游 pin 的 commit（如 `fae47e0`） | 与上游 1.5.3 完全对齐，但**不带**自定义服务器配置，产物仍连官方服务器 |

截至编写时 `ldxw/rustdesk_hbb_common` **没有任何版本 tag**（只有分支），所以 `latest` 实际会解析成
`main`；等 fork 上打了 `1.5.3` 之类的 tag 后，`latest` 会自动切到那个 tag。Summary 里会写明
「输入 `latest` 解析得到 `X`」，便于确认这次到底用了哪个 ref。

用含自定义服务器配置的 ref 才符合「连自己的服务器」这个目的；填 pin commit 只适合排查
「是不是 fork 引入的问题」。

注意：Windows 打印机驱动从 `https://github.com/rustdesk/hbb_common/releases/download/driver/...`
下载，那是上游的 release 资产，不受子模块 URL 影响，也不在 fork 上。

### web 面板地址（`admin_url`）

客户端「地址簿 → Web 控制台」调用 `bind.mainGetApiServer()`，拿到的就是这里的 **API server**，
也就是浏览器里打开的 web 管理面板地址。取值链在 `src/common.rs` 的 `get_api_server_()`，
按优先级：

| 顺序 | 来源 |
| --- | --- |
| 1 | Windows 许可证里的 api |
| 2 | 运行期选项 `api-server`（用户在客户端里手填并保存过） |
| 3 | 编译期 `option_env!("API_SERVER")` |
| 4 | 由自定义中继推导 `http://<host>:21114` |
| 5 | 兜底硬编码 `"https://admin.rustdesk.com"` |

本工作流改的是第 5 项——这是源码里唯一的硬编码兜底点。前四项在构建产物里无法预置：
上游 `flutter-build.yml` 的 `env` 里**没有** `API_SERVER`（只有 `playground.yml` 有），
所以编译期覆盖走不通，只能改源码，也就必须放在派生 commit 里。

派生 commit 的 web 面板改动：

| 项 | 值 |
| --- | --- |
| 文件 | `src/common.rs` |
| 改动 | 把 `"https://admin.rustdesk.com"` 替换成输入的地址 |
| 匹配方式 | 只替换**恰好带双引号**的字面量，注释里的同域名不受影响 |
| 归一化 | 先去掉末尾的 `/`（`get_api_server()` 本来也会去，提前做可让幂等判断稳定） |

`admin_url` 的取值优先级与行为：

| 输入 / secret | 行为 | Summary 行 |
| --- | --- | --- |
| 输入非空 | 用输入 | 已改写 / 与上游一致 |
| 输入空，secret `ADMIN_URL` 非空 | 用 secret | 同上 |
| 都空 | 用内置默认 `https://admin-hy.rs.ldxw.top` | 同上 |
| 输入或 secret 为 `none` | 完全不动 `src/common.rs` | 保持上游默认 |
| 等于 `https://admin.rustdesk.com` | 不动（改写没有意义） | 与上游一致 |
| 不以 `http://` / `https://` 开头 | 硬失败，不推 tag | — |
| 上游 `src/common.rs` 里找不到该字面量 | 硬失败并在 Summary 指明阶段，不推 tag | — |

换掉域名后 `is_public()`（`url.contains("rustdesk.com/") || url.ends_with("rustdesk.com")`）
自然不再命中，不需要额外改动。

**幂等性**：改写只依赖「上游 commit + 目标地址」。`admin_url` 产生改写时，派生 commit 的
提交信息会多一行 `build: set client api server to <地址>`，因此不同地址得到不同 SHA；
更新链产生改写时同样会多一行 `build: use <UPDATE_REPO> for client updates`，因此不同
`UPDATE_REPO` 也得到不同 SHA；同一组输入重跑会命中「tag 已存在且一致」而跳过，不会误报冲突。

如果只想先拿到能连自己服务器、但 web 面板仍指官方的客户端，把 `admin_url` 填 `none` 即可。

### 软件更新地址（`UPDATE_REPO`）

客户端「检查更新」不再走上游 `rustdesk/rustdesk`，而是走**本仓库自己构建的 release 资产**。
`UPDATE_REPO` 固定取 `github.repository`（即本仓库），没有单独的输入。

完整发现流程：

1. 客户端 GET `https://github.com/<本仓库>/releases/download/update-latest/version.json`；
2. 响应体 `{"url":"https://github.com/<本仓库>/releases/tag/<tag>"}`；
3. 从该 URL 解析出 release id（即 tag 名）与版本号；
4. 取该 tag release 里的 `rustdesk-update.json` 与 `rustdesk-update.json.sig`；
5. 验签后按平台/架构/格式选中安装包，下载并用 `sha256` 校验。

这要求 `src/common.rs` 的发现逻辑改成 GET 静态文件：`hbb_common::version_check_request`
返回的是 POST 请求体，打 GitHub 静态资产会得到 405，所以派生 commit 里把这段改成
`client.get(&url).send().await`，`url` 直接指向上面的 `version.json`。
`hbb_common` 的 fork 不需要任何改动。

#### 派生的改动点

| 文件 | 改动 |
| --- | --- |
| `src/common.rs` | `release_id_from_update_url` 增加本仓库的两条 match arm；`do_check_software_update` 改为 GET `update-latest/version.json` |
| `src/updater/artifact.rs` | 下载前缀从 `rustdesk/rustdesk` 改为本仓库；owner/repo 守卫同时接受上游与本仓库 |
| `src/update_metadata.rs` | `TRUSTED_UPDATE_KEYS` 的内嵌公钥替换为 `vars.RUSTDESK_UPDATE_ED25519_PUBLIC_KEY` 解码出的 32 字节 |
| `flutter/lib/desktop/pages/desktop_home_page.dart` | 两处下载地址指向本仓库 releases |
| `flutter/lib/mobile/pages/connection_page.dart` | 下载地址指向本仓库 releases |

`UPDATE_REPO` 等于上游 `rustdesk/rustdesk` 时整段改写跳过（否则会与上游 match arm 重复，
Rust 编译报 unreachable pattern，而且上游没有 `update-latest` release），Summary 显示「保持上游」。

#### 7 个安装包与 selector

| 资产文件名 | platform | arch | format |
| --- | --- | --- | --- |
| `rustdesk-<版本>-x86_64.exe` | windows | x86_64 | exe |
| `rustdesk-<版本>-x86_64.msi` | windows | x86_64 | msi |
| `rustdesk-<版本>-aarch64.exe` | windows | aarch64 | exe |
| `rustdesk-<版本>-aarch64.msi` | windows | aarch64 | msi |
| `rustdesk-<版本>-x86-sciter.exe` | windows | x86 | exe |
| `rustdesk-<版本>-aarch64.dmg` | macos | aarch64 | dmg |
| `rustdesk-<版本>-x86_64.dmg` | macos | x86_64 | dmg |

必须精确挑选这 7 个：上游 `flutter-build.yml` 里负责发布签名元数据的 job 带
`github.repository == 'rustdesk/rustdesk'` 守卫，在本仓库不跑；7 个安装包是三个平台 job 内的
`softprops/action-gh-release` 直接传到 tag release 的。同一个 release 里还混有 Android/Linux
资产，所以下载时按文件名逐个 `--pattern`，并校验目录里恰好 7 个。

#### `publish-release-assets.yml` 做的事

触发条件是 `Flutter Tag Build` 完成且成功（`workflow_run`），因此该文件必须在默认分支上。

1. 解析构建 tag：先看 `workflow_run.head_branch` 是否为已存在的 tag；否则用 `head_sha`
   反查唯一 tag（多个 tag 指向同一提交时直接失败）；
2. 若该 release 已有 `rustdesk-update.json`，拒绝覆盖（要修正产物只能换新 tag）；
3. checkout 该 tag，装 Python 3.12 与带 hash 锁定的依赖；
4. 下载 7 个安装包并校验数量；
5. `check-key`：确认 secret 的 seed 推导出的公钥、`vars` 里的公钥、`src/update_metadata.rs`
   内嵌的公钥三者一致；
6. `sign`：生成并签名 `rustdesk-update.json`；
7. `verify`：按同一组参数重新校验签名与本地资产；
8. 先上传 `.sig` 再上传 `.json`（避免出现正文已存在、签名还没到的窗口）；
9. 刷新滚动 release `update-latest` 里的 `version.json`。

#### 需要注意

- `workflow_run` 只在工作流位于**默认分支**时触发。
- `head_branch` 在 tag push 下是否等于 tag 名未实测；工作流带了按 `head_sha` 反查 tag 的回退。
- 非三段纯数字 tag（如 `1.5.3-1`）仍会发布资产，但客户端解析不了，工作流只打 warning。
- 已发布过 `rustdesk-update.json` 的 release 不会覆盖，需要换新 tag。
- 三处密钥必须成对：secret `RUSTDESK_UPDATE_ED25519_SEED`（base64 32 字节私钥种子）、
  vars `RUSTDESK_UPDATE_ED25519_PUBLIC_KEY`（base64 32 字节公钥）、`src/update_metadata.rs`
  里内嵌的公钥。`check-key` 在 `sign` 之前跑，不一致会立刻失败。
- `sign` 的 `--artifact` 是 4 段（platform arch format 路径），`verify` 的 `--artifact` 是单值路径；
  `verify` 没有 `--published-at`；`--release-base-url` 不能带尾 `/`。
- 客户端当前版本必须**严格小于** release 版本才会提示更新。
- 没有配置代码签名时，Windows 会跳过签名段、macOS 出 unsigned DMG，安装包仍可下载。
- 移动端的更新入口直接跳到本仓库的 releases 页面。

### 前置条件

**必须配置 PAT（secret `SYNC_TOKEN`）**，且它要有 **workflow 权限**。两个原因：

1. `GITHUB_TOKEN` 推送的 tag **不会触发其它工作流**（GitHub 的防递归机制），
   构建永远不会启动。工作流在预检阶段就硬失败，不会静默成功。
2. 若目标 commit 尚未同步到本仓库，这次 push 会连带 `.github/workflows/*` 一起上传，
   而 GitHub App token 不允许推送这类文件。PAT 需要 workflow 权限才能推。

PAT 权限要求与「前置设置 1」相同（classic: `repo` + `workflow`；fine-grained:
Contents 读写 + Workflows 读写）。

**建议先跑一次同步工作流**：先把该版本的源码推成同名分支，再打 tag。打 tag 推的是派生 commit
（父 = 上游 commit），父提交已在仓库里时，这次 push 只新增 1 个 commit + 1 个 tree，
不涉及 `.github/workflows/*`，对 PAT 权限的要求最宽松。反之若父提交不在仓库里，push 会连同
上游整棵树（含 11 个 workflow 文件）一起上传，需要 PAT 具备 workflow 权限。

### 构建本身的行为

| 项 | 说明 |
| --- | --- |
| 运行名 | **Flutter Tag Build**（上游 `flutter-tag.yml` 的 `name`） |
| 构建定义 | tag commit 内的 `flutter-build.yml`，18 个平台 job |
| 产物 | 上传为 Actions artifact；同时用 `softprops/action-gh-release` 挂到 tag 的 release，统一 `prerelease: true` |
| 版本号来源 | `upload-tag: ${{ github.ref_name }}`，即本次打的 tag 名 |

不配置任何签名 secret 也能出可下载产物：Windows 跳过签名段、Android 出 unsigned APK、
macOS 出 unsigned DMG 并额外打包 `rustdesk-<版本>-unsigned.tar.gz`。

### 版本号必须三处一致

上游把版本号写在三个地方，任意一处不一致都会在构建或自检时失败：

| 位置 | 内容 |
| --- | --- |
| `Cargo.toml` | `version = "1.5.3"` |
| `Cargo.lock` | 同一版本的两行（`name` / `version`） |
| `.github/workflows/flutter-build.yml` | `env.VERSION: "1.5.3"` |

`flutter-build.yml` 里的 `VERSION` 既用于拼产物文件名，也用于构建后校验：
Windows 报错版本与它不一致会 `throw`，macOS 会直接失败。构建里大量命令带 `--locked`，
所以 `Cargo.lock` 的版本也必须与 `Cargo.toml` 对上。

三处构建期自检：Windows flutter job、Windows sciter job、macOS job 各有一段版本比对。

因此**当前只有 tag `1.5.3` 能完整通过**；其它版本会跑到构建后校验处失败。
这是上游文件的内容，本仓库不修改它（改了就等于维护一份 fork 的 3000 行构建定义）。

### 重建同一个版本

tag 已存在且指向同一 commit 时，GitHub 不会产生新的 `push` 事件，本工作流会跳过。
要重新构建：

- 对该次 **Flutter Tag Build** 运行使用 **Re-run all jobs**；或
- 先删掉远端 tag，再重跑本工作流；或
- 用 `force: true` 并让上游该 tag 指向不同 commit（通常不适用）。

## 更换上游或换仓库

- 换上游：同时修改 `.github/workflows/sync-upstream-tags.yml` 的 `jobs.sync.env.UPSTREAM_REPO`
  和 `.github/workflows/tag.yml` 的 `jobs.tag.env.UPSTREAM_REPO`。
- 换 hbb_common fork：改 `.github/workflows/tag.yml` 的 `jobs.tag.env.HBB_COMMON_REPO`；
  默认 ref 改 `HBB_COMMON_REF` 的默认值或运行时的 `hbb_common_ref` 输入。
- 换 web 面板默认地址：改 `.github/workflows/tag.yml` 的 `jobs.tag.env.ADMIN_URL` 里的内置默认
  （目前是 `https://admin-hy.rs.ldxw.top`），或直接配 secret `ADMIN_URL`、运行时填 `admin_url`。
- 推送目标固定为当前仓库（`github.repository`），不需要改。
