---
name: dsh-github
description: 处理本机多个 git 仓库相对 GitHub 的流程——先前状态、判断该不该开分支/提 PR/推、以及 push/PR/合并的动作规范。当任务是提交代码、推送、开 PR、合并、同步 fork、或需要看清哪些仓库有未推/未提交内容时用它。
---

# GitHub 流程

## 为什么存在这个技能

这台机器上管着六个仓库，全部由 AI 执行 git，用户不碰命令。单人开发，没有协作者，没有 CI。

失败模式不是「命令写错」，是**没人主动去看**。单个 `git status` 看不出哪个仓库忘了推；五个仓库各跑一次是在拼手工。所以流程的第一步固定是跑状态，不靠记忆。

## 第一步：先看状态，别问

```bash
/Volumes/Data/Deepseek Harness/dsh-github/bin/ghstat --fetch
```

任何 git 动作之前先跑这条。它的输出就是判断依据，不需要再逐仓库 `git status`。

- `↑↓` 的基准是**上游集成分支**（fork 工作流里即 `origin` 的默认分支，PR 要合进去的那条），不是「本地分支跟踪谁」。表尾会写明本次基准。
- 不带 `--fetch` 时远端引用可能过期，输出里会明说。要看真实差距就带 `--fetch`（只更新远端引用，不动工作树）。
- `fork` 标记表示远端分叉。`fork主落后N` 表示 fork 的 main 落后上游 N 个提交——**这种仓库绝不能用 fork/main 当新分支的基线**，否则 PR 会带一堆无关提交。
- `dirty` 含未跟踪文件。
- 加 `--check` 时，有漂移就返回退出码 1，可用于钩子。
- 加 `--no-pr` 可离线跑（跳过 `gh` 网络查询）。

## 硬边界：什么必须停下来问用户

用户的规则：**任何 commit、issue、PR，在细节敲定前不得动手。** 这条不是形式，是因为 git 工作流出了事很难在事后察觉。

以下动作一律先讲清楚内容、拿到确认再执行：

| 动作 | 为什么要问 |
|---|---|
| `git push`（任何分支） | 推出去就可能被别人看到、被引用，且改写历史代价大 |
| 开 issue / 关 issue | 对外可见，措辞会被当成承诺 |
| 开 PR / 评论 PR / 合 PR | 对外可见，合了不可逆 |
| 改写已推送的历史（rebase、amend、force push） | 破坏性 |
| 改任何仓库的 git config（user.name 等） | 会影响之后所有提交的作者身份 |

不需要问的：读操作（status、log、diff、ls-remote）、在工作区内写代码、建本地分支、本地 commit 到**尚未推送的特性分支**——但这些做完要向用户汇报内容，因为下一步通常就是 push。

判断口径：动作会不会**离开这台机器**、或**改变已离开机器的内容**。会，就问。

## 什么时候开分支

单人开发不需要为每次改动开分支。按下面的分界线走：

- 改动会**合并进上游仓库**（fork 场景，如 dsh-mneme → slow-stack/mneme）：开特性分支。上游只接受 PR，直接推 main 是推不上去的，而且会让 fork 的分支状态混乱。
- 改动是**一次能讲清楚的功能或修复**，且中途可能需要放弃：开分支。
- 改动是**文档、配置、小修补**、且你打算立刻推：直接在当前分支提交。
- **绝不**在 fork 仓库里把上游 main 当工作分支用。fork 的 main 只用来跟上上游。

分支命名（沿用已有事实，不另立新规）：

```
feat/<简短英文>
fix/<简短英文>
chore/<简短英文>
test/<简短英文>
refactor/<简短英文>
```

已存在的实例：`fix/llm-audit-usage-nesting`、`test/llm-audit-usage-boundaries`。用小写连字符，不用中文，不用下划线。

## commit 格式

沿用已经在用的约定式前缀加中文描述：

```
type(scope): 中文描述
```

- `type`：`feat` / `fix` / `chore` / `docs` / `refactor` / `test` / `perf` / `build`
- `scope` 可省。有明确模块时写模块名，中文也可以：`fix(卦例卡): …`
- 描述说清**改了什么**，能带上**为什么**更好。现有实例：`refactor: 转为 Xcode 工程，托盘换用 DeepSeek 图标`
- 一个提交只做一件事。塞多件事的提交，出问题时要整体回退。

## PR 规范

标题与 commit 同格式。正文按这个骨架，空的小节可以删掉：

```markdown
## 改动
（做了什么，一段话说完）

## 为什么
（解决什么问题；如果是改 bug，说清原来的行为错在哪）

## 验收
（能跑的命令、测试结果、截图路径。不写「应该实现了」）
```

`## 验收` 这一节不要省。用户对「声称完成却不提供证据」是明确反感的，PR 正文是唯一能把证据固定下来的地方。

fork 场景开 PR 时用 `--repo` 指向上游：

```bash
gh pr create --repo <upstream-owner>/<repo> --head <your-owner>:<branch> ...
```

推之前先 `gh pr list -R <upstream-owner>/<repo> --state open` 看有没有重复的 PR。

## 合并前检查

三条，缺一条不合并：

1. `gh pr checks <number> -R <slug>` 看 CI。没有 CI 的仓库跳过。
2. `git -C <repo> log --oneline <base>..<head>` 确认要合进去的提交就是预期的那些，没有夹带。
3. `gh pr view <number> -R <slug>` 看 diff 摘要。改动面比预想的大就停下来查原因。

合并方式：单人开发用 `--squash`，保持上游历史一行一个改动。上游有自己的惯例时跟随上游。

## fork 仓库与上游同步

本机只有一个这种仓库：`dsh-mneme`。

它的形态要记清楚，因为它和「自己的仓库」完全不同：

| 项 | 值 |
|---|---|
| 上游 | `slow-stack/mneme`（远端名 `origin`）。曾用名 `modusensus/dsh-mneme`，是重命名重定向，同一个仓库 |
| 你的 fork | `davidekingsss/dsh-mneme`（远端名 `fork`）——**仅用于提交 PR** |
| 你的角色 | 临时参与维护，不是作者 |
| 上游 issues | 开着；但 fork 侧 issues 关闭，只能在 fork 提 PR，不能在 fork 开 issue |

三条纪律：

1. **新分支从上游开，不从 fork/main 开。** fork/main 长期落后上游（2026-10-03 时落后 222 个提交）。拿它当基线，PR 会带一堆无关提交。
   ```bash
   git -C dsh-mneme fetch --all --prune
   git -C dsh-mneme switch -c fix/xxx origin/main
   ```
2. **不碰上游的 CHANGELOG 和版本号。** 那是维护者的发版流程，贡献者的改动由维护者在合并时整理。上游有 CONTRIBUTING，动手前先读。
3. **PR 合并后删本地分支。** 已合并的分支堆着会让 `ghstat` 的分支列失去意义。

同步本地分支到最新上游（rebase 属改写历史，**先问用户**）：

```bash
git -C dsh-mneme fetch --all --prune
git -C dsh-mneme rebase origin/main   # 在当前特性分支上
```

自己的仓库（其余五个）落后就是落后，没有「上游」概念，`↑↓` 基准是自己的 origin。

## 动作速查

```bash
# 状态（永远先跑）
ghstat --fetch

# 从上游开新分支（fork 仓库如 dsh-mneme，先 fetch 再切）
git -C <repo> fetch --all --prune
git -C <repo> switch -c fix/xxx origin/main

# 提交
git -C <repo> add -A && git -C <repo> commit -m "type(scope): 描述"

# 推送特性分支
git -C <repo> push -u origin <branch>

# 开 PR（同仓库）
gh pr create --title "..." --body "..." --base main

# 开 PR（fork → 上游）
gh pr create --repo <upstream>/<repo> --head <owner>:<branch> --title "..." --body "..."

# 看 PR
gh pr list -R <slug> --state open
gh pr view <number> -R <slug>
gh pr checks <number> -R <slug>

# 合并
gh pr merge <number> -R <slug> --squash --delete-branch
```

## 触发时机

- 会话开始处理某个仓库的代码改动时：先 `ghstat --fetch`，看该仓库是不是落后上游、有没有未推内容。
- 用户说「提交」「推送」「推上去」时：先 `ghstat --fetch`，再按硬边界确认内容，然后执行。
- 用户说「推到 GitHub」时：确认是推分支还是开 PR。含糊时问，不要猜。
- 一次工作结束、要收尾时：跑一次 `ghstat --fetch --check`，把剩下的漂移报出来。
