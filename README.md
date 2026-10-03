# ghstat

一次看清一组本地 git 仓库相对 GitHub 的真实状态。

```
repo            branch                           ↑↓     dirty  pr  标记                最后提交
--------------  -------------------------------  -----  -----  --  ------------------  ------------------------
PlumProject     master                           0/0    0      0                       3分钟前 · davidekingsss
dsh-github      main                             0/0    1      0                       4分钟前 · davidekingsss
dsh-mac-screen  main                             0/0    0      0                       43分钟前 · davidekingsss
dsh-manager     main                             0/0    0      0                       6天前 · davidekingsss
dsh-mneme       test/llm-audit-usage-boundaries  0/215  0      0   fork fork主落后222  14天前 · davidekingsss

↑↓ 基准：origin/main, origin/master
已 fetch：5/5 个仓库成功

需要处理 1 个：
  dsh-github：1 个文件未提交

fork 的 main 落后上游（不影响当前分支，但别拿它当新分支的基线）：
  dsh-mneme：fork/main 落后 origin/main 222 个提交
```

这一屏就是它存在的理由。`dsh-mneme` 那一行如果按「本地分支跟踪谁」算，会显示成无害的 `1/1`；按上游算，才看得出基线落后了 215 个提交。

## 为什么有这个工具

管着六个仓库，全部由 AI 执行 git，单人开发，没有协作者也没有 CI。

失败模式不是命令写错，是**没人主动去看**。单个 `git status` 看不出哪个仓库忘了推；六个仓库各跑一次是在拼手工，不是流程。这个脚本把「记得去看」从自觉变成一条命令。

这是全部动机，所以它刻意做得很小：无依赖、单文件、跑完就退。不做面板、不做守护、不做定时。它坏掉的后果是回到手打命令，不是工作中断。

## 安装

脚本本身是单文件 Python 3，只用标准库，不需要安装依赖。

```bash
# 1. 挂到 PATH（二选一）
ln -sf "/Volumes/Data/Deepseek Harness/dsh-github/bin/ghstat" ~/.local/bin/ghstat

# 2. 配置默认扫描根（可选）
mkdir -p ~/.config/ghstat
printf '/Volumes/Data/Deepseek Harness\n' > ~/.config/ghstat/roots
```

没配 `~/.config/ghstat/roots` 时，默认扫 `~/code` 与 `/Volumes/Data`。

## 用法

```bash
ghstat                                   # 扫默认根
ghstat --fetch                           # 先 fetch --all --prune 再扫描
ghstat -p ~/code -p /Volumes/Data        # 指定根，可重复
ghstat -p /path --depth 1                # 限制仓库发现深度（默认 2）
ghstat --no-pr                           # 跳过 gh 网络查询，离线可用
ghstat --json                            # 机器可读
ghstat --check                           # 有漂移时退出码 1，供钩子用
```

## 字段含义

| 字段 | 说明 |
|---|---|
| `branch` | 当前分支 |
| `↑↓` | ahead/behind，基准是**上游集成分支**（fork 工作流里即 `origin` 的默认分支，也就是 PR 要合进去的那条） |
| `dirty` | 未提交文件数，含未跟踪 |
| `pr` | open PR 数。查不到或 `--no-pr` 时显示 `-` |
| `标记` | `fork`：远端分叉；`fork主落后N`：fork 的 main 落后上游 N 个提交 |
| `最后提交` | 相对时间 · 作者 |

表尾会写明本次的 `↑↓` 基准和是否 fetch 过。

## 为什么 ↑↓ 不以「本地分支跟踪谁」为基准

这是踩出来的。fork 工作流里本地分支常跟踪 `fork/main`，而 `fork/main` 可能几周没同步。以它为基准算出的是「领先 1 落后 1」，看着无害；真正的情况是分支的基线已经落后上游两百多个提交，拿它开 PR 会带一堆无关提交。

所以基准固定取上游集成分支。分支自己跟踪谁、落后多少，放在 `--json` 的 `tracking` / `track_ahead` / `track_behind` 里，不占表格。

## 设计取舍

**默认只读，`--fetch` 是唯一写动作。** 不 fetch 时只用本地已有的远端引用，所以不改变任何仓库状态、不需要网络。代价是 ahead/behind 相对的是**上次 fetch 时**的远端状态，这个代价会在输出里明说，不藏着。加 `--fetch` 时先对每个仓库 `fetch --all --prune`，只更新远端引用，不动工作树。

**fetch 失败要吵。** fetch 集中做、失败逐条列出来。塞进并发扫描里的话，一个仓库 fetch 失败会退化成「字段显示 `-`」，看不出是网络问题还是真的没有上游。

**失败一律降级，不报错。** 某个仓库的某个字段拿不到就显示 `-`，不中断整张表。扫描工具的价值在于有结果；一个仓库查询超时不该让另外五个仓库的信息也消失。

**并发查 PR。** `gh` 调用走线程池（上限 8），否则六个仓库串行会让脚本跑十几秒。加 `--fetch` 后总耗时约 12 秒，其中大部分是 fetch 的网络时间。

**表格按东亚字符宽度对齐。** 仓库名含中日韩字符时 `len()` 会让表格错位。

**默认根可配置。** 写死成一个目录会出现「跑了一遍没找到仓库」的假阴性，那比不跑更坏——它让人以为干净。

## 已知问题

- 不 fetch 时 ahead/behind 可能落后于真实远端状态（见上，用 `--fetch` 解决）。
- `git status --porcelain` 统计的是**行数**，重命名会占两行，所以 `dirty` 不是严格的「文件数」。
- 只认 GitHub 远端。GitLab、Gitea 的仓库会被扫到但 `pr` 恒为 `-`。
- 上游远端的判定依赖远端名称：有 `fork` 远端时取 `origin` 为上游，否则取 `origin`。远端命名不规范的仓库会取错基准。

## 文件

```
bin/ghstat                     脚本本体
skills/dsh-github/SKILL.md     GitHub 流程（何时跑、何时开分支、何时必须先问用户）
```
