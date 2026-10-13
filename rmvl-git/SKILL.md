---
name: rmvl-git
description: 使用 RDT、原生 Git 与 gh 管理 RMVL 仓库工作流。处理仓库更新、分支创建、精确暂存、提交拆分、提交压缩或改写、推送以及 GitHub Pull Request 时使用，也用于“提交并创建 PR”等连贯发布任务。普通 RDT/LPSS CLI、项目脚手架和运行时诊断使用 rmvl-dev-tools。
---

# RMVL Git

## 信息源与安全边界

以当前安装的 RDT 脚本和目标仓库状态为准。开始时运行 `rdt git help`，需要确认实现时阅读 `~/.rmvldev/rmvl-dev-tools/scripts/bash/rdt.bash` 与 `scripts/bash/.rdt/*.bash`。

提交、推送、切换分支、改写历史或创建 PR 前确认用户授权范围。先执行只读检查：

```bash
git status --short --branch
git diff
git diff --cached
git remote -v
git log -8 --pretty=format:%h%x09%s
```

保留用户已有改动。工作树包含多个主题或未知文件时，禁止使用隐式 `git add .` 的流程，改用原生 Git 精确暂存。

## RDT Git 命令

- `rdt git commit`：执行 `git add .` 并交互式生成规范提交消息。仅在工作树全部属于当前提交时使用。
- `rdt git newbr`：交互式创建分支、暂存、提交，并可推送至 `origin`。仅在用户要求单笔提交且工作树内容单一时使用。
- `rdt git squash`：创建临时提交并压缩到上一提交，可能执行 `git push --force-with-lease`。仅在用户明确要求改写历史时使用。
- `rdt git reword`：修改上一提交消息。仅在用户明确要求时使用。
- `rdt git update`：RMVL 专用组合流程，包含切换分支、远端同步、推送和 `git reset --hard 2.x`。仅在用户明确授权完整流程，并核对 stash、工作树和分支状态后使用。

提交时优先使用 RDT。工作树全部属于当前提交且没有未知文件时，必须实际执行 `rdt git commit`，不能只运行 `rdt git help` 后直接使用原生 `git commit`。用户要求新建分支且满足上述单一主题条件时，可使用 `rdt git newbr`。

仅在 RDT 不可用、执行失败，或其隐式 `git add .` 会混入其他主题、本地配置、构建产物或未知文件时，回退到原生 Git 精确暂存和提交，并向用户说明具体原因。已有精确暂存本身不是回退理由，单独的提交与推送请求也不需要为了使用 RDT 而新建分支。

`rdt update` 更新 Dev Tools 本身，不等同于 `rdt git update`。

## 创建分支与提交

1. 确认目标基准分支以及本地、`origin`、`upstream` 的关系。RMVL 2.x 通常基于 `upstream/2.x`，但不要只按分支名称推断。
2. 在用户要求发布或明确允许同步时获取最新基准。若基准前进，先安全处理当前工作树，再决定变基或重建分支。
3. 创建语义明确的分支。工作树较复杂时使用原生 `git switch -c <branch>`。
4. 按改动主题拆分提交。单一主题且工作树全部属于当前提交时使用 `rdt git commit`，需要拆分或排除文件时使用原生 Git，以 `git add -- <paths...>` 精确暂存。不要把本地配置、构建产物或其他主题带入提交。
5. 对每笔提交检查：

```bash
git diff --cached
git diff --cached --check
git status --short --branch
```

6. 运行与改动风险匹配的构建、单元测试、KDT 集成测试或 benchmark，再使用符合仓库历史的提交标题，例如 `feat(3rdparty): 集成 rapidyaml`。
7. 提交完成后检查相对基准的提交列表、总差异和工作树，再推送到 `origin` 并设置 upstream tracking。

纯提交、推送的格式报告必须遵循以下规则：

- 提交信息和推送情况必须使用表格，不得改写为列表或普通段落。
- 表格单元格中的提交标题、SHA、远端和分支均使用普通文本，不要包裹为行内代码。
- 不单独增加“测试结果”章节，实际执行的测试及结果统一写入“其他”的“其他说明”中。
- 表格之外的分支、命令和路径仍可按一般 Markdown 规则使用行内代码。

```markdown
## 提交信息

状态：✅已完成 / ❌未完成
分支：`<owner>/<repo>:<branch>`

| Commit Title | SHA1 |
| :----------- | :-------------: |
| <commit_title> | <commit_hash, 前六位> |
| ... | ... |

## 推送情况

| 远端仓库 | 分支 | 推送状态 |
| :------- | :--- | :------: |
| origin | <branch> | 已推送 / 未推送 |
| ... | ... | ... |

## 其他

- 未提交改动：xxx（总结概括即可，没有就写“无”）
- 其他说明：xxx（没有就写“无”）
```

如果有多个仓库的提交，这些内容分开展示，不要混在一个表格中。

## 创建 GitHub Pull Request

仅在用户明确要求创建 PR 时执行推送和远端创建。先读取最新基准分支中的 `.github/PULL_REQUEST_TEMPLATE.md`，并检查 `gh auth status`。模板可能变化，不要复用旧 PR 正文。

按以下规则编写 PR：

- 以仓库模板为正文基础，只勾选真实满足的清单项。
- 在“具体内容”后概括改动，在 `### 测试` 中记录实际命令和结果。不要声称运行过未执行的 Windows、完整单元或性能测试。
- PR 标题通常不带 `feat(...)` 等 Conventional Commit 前缀。例如提交标题为 `feat(3rdparty): 集成 rapidyaml` 时，PR 标题可写为 `三方模块集成 rapidyaml`。
- 跨 fork 时显式指定仓库、基准分支和来源所有者：

```bash
gh pr create \
  --repo cv-rmvl/rmvl \
  --base 2.x \
  --head <fork-owner>:<branch> \
  --title "<title>" \
  --body-file <body-file>
```

创建后核对元数据：

```bash
gh pr view <number> \
  --repo cv-rmvl/rmvl \
  --json number,title,url,baseRefName,headRefName,headRepositoryOwner,isDraft,state
```

若 `gh` 明确报告认证失败，提示用户执行 `gh auth login -h github.com`。TLS 握手超时等瞬时网络错误可重试一次。

## 提交并创建 PR

用户要求“提交并创建 PR”时，依次完成审计、测试、分支、精确暂存、提交、推送、模板读取、PR 创建和元数据核验。多个独立主题可拆成多笔 commit，再创建一笔综合 PR；用户明确要求单笔提交且工作树内容单一时，可使用 `rdt git newbr` 简化流程。

最终使用以下格式报告，并沿用“纯提交、推送”小节的表格和测试说明规则：

```markdown
## 提交信息

状态：✅已完成 / ❌未完成
分支：`<owner>/<repo>:<branch>`

| Commit Title | SHA1 |
| :----------- | :-------------: |
| <commit_title> | <commit_hash, 前六位> |
| ... | ... |

## 推送情况

| 远端仓库 | 分支 | 推送状态 |
| :------- | :--- | :------: |
| origin | <branch> | 已推送 / 未推送 |
| ... | ... | ... |

## 合并请求

链接：<PR_url>
标题：<PR_title>
目标分支：`<base_owner>/<base_repo>:<base_branch>`

## 其他

- 未提交改动：xxx（总结概括即可，没有就写“无”）
- 其他说明：xxx（没有就写“无”）
```27 1### 提交信息23状态：✅已完成 / ❌未完成4分支：`<owner>/<repo>:<branch>`56|  Commit Title  | SHA1 (前 6 位) |7| :------------: | :------------: |8| <commit_title> | <commit_hash>  |9|      ...       |      ...       |1011### 推送情况1213| 远端仓库  | 分支 | 推送状态 |14| :-------: | :--: | :------: |15| origin   | <branch> | 已推送 /未推送 |16| ...      |  ...  |    ...    |1718### 合并请求1920链接：<PR_url>21标题：<PR_title>22目标分支：`<base_owner>/<base_repo>:<base_branch>`2324### 其他2526- 未提交改动：xxx（总结概括即可，没有就写“无”）27- 其他说明：xxx（没有就写“无”）markdown
````
