# exp 使用指南：在HPC上方便的长时间使用claude code

`exp` 是一个基于 bash 的小工具，用于在 HPC 集群上管理跨多个 CPU 节点的实验会话。

**它解决什么问题？** 用 `salloc`/`srun` 申请交互式作业，和登录终端是绑死的——网络一抖、SSH 一断、或者笔记本合盖休眠，终端没了，作业和正在跑的实验也跟着一起没了。
此外，同时开多个实验时，很难记住哪个作业分到了哪个节点、该 ssh 到哪台机器上去看进度，很快就理不清。

**核心思路**：把「资源」和「会话」解耦：**一个实验 = 一个独立的 Slurm 作业（独占一段节点资源）+ 一个跑在该节点上的 tmux 持久会话**。
会话活在计算节点上，不依赖你的 SSH 连接。你可以随时进入会话干活、随时脱离，断线也不影响实验继续运行；每个实验用一个名字就能找回来，不用手动追踪 job ID 和节点名。


---

## 1. 快速上手

### 首次使用：把 exp 注册进 PATH

脚本放到以下路径：`~/bin/exp`

```bash
chmod +x ~/bin/exp                                  # 赋予执行权限
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc   # 把 ~/bin 注册进 PATH
source ~/.bashrc                                    # 让当前终端立即生效（新开的终端自动生效）
```

验证：敲 `exp`，能打印出用法说明就说明注册成功

### 日常使用
```bash
exp init            # 首次使用：生成默认配置文件
exp start myrun     # 申请一个节点，起一个叫 myrun 的实验，2 秒后自动进入
                    # （如果 2 秒内没排上队，会提示 PENDING，稍后 exp attach myrun 即可）
# ... 干活（请claude code大人出山）...
# 按 Ctrl+B 松开再按 D 脱离会话（实验继续在节点上跑）
exp attach myrun    # 随时再进去
exp list            # 看作业状态
exp stop myrun      # 实验做完，释放节点
```

---

## 2. 命令参考

| 命令 | 作用 | 背后动作 |
|---|---|---|
| `exp init` | 生成默认配置文件（已存在则不覆盖） | 写入 `~/.config/exp/exp.conf` |
| `exp start <name>` | 申请节点 + 建持久会话，2 秒后自动 attach；同名实验已存在时会拒绝 | `sbatch --wrap="tmux new-session -d; sleep infinity"`，然后 `sleep 2` + attach |
| `exp attach <name>` | 进入实验的 tmux 会话 | `srun --jobid=… --overlap --pty tmux attach` |
| `exp list` | 列出所有 exp 管理的作业 | `squeue -u $USER` + 前缀过滤 |
| `exp stop <name>` | 释放该实验的节点（**杀掉里面所有进程**） | `scancel --name=cc-<name>` |
| `exp stopall` | 释放全部实验节点 | 批量 `scancel` |

所有子命令前都可加 `-c <路径>` 指定其他配置文件：

```bash
exp -c ~/.config/exp/gpu.conf start bigrun
```

配置文件的查找优先级：命令行 `-c` > 环境变量 `EXP_CONFIG` > 默认路径 `~/.config/exp/exp.conf`。多套配置适合"不同项目 / 不同分区 / 不同时长"的场景，建议给每套配置用不同的 `PREFIX`，这样 `list` / `stopall` 互不干扰。

---

## 3. 配置项说明

`~/.config/exp/exp.conf` 就是一个被 `source` 的 shell 片段：

```bash
WORKDIR=/path/to/your/workdir   # tmux 会话的初始工作目录
PREFIX=cc-           # 作业名前缀，exp 靠它识别"自己的"作业
TIME=9:00:00         # 每个实验的 walltime（时:分:秒），到点作业被强制结束
EXCLUDE=xcng0,xcng1  # 黑名单节点，逗号分隔（比如已知有问题的机器）
# PARTITION=long     # 取消注释可指定分区（对应 sbatch --partition）
```

关于 `TIME` 要理解一点：这是向调度器做出的**承诺**，不是"用多少算多少"。到点后 Slurm 会直接杀掉作业，不管里面的程序跑没跑完；而申请的时间越长，排队优先级通常越低。按实验的真实需要设置，长任务宁可拆分或适当留余量。

另外，本集群会根据 `TIME` 自动把作业路由到合适的分区（≤3 小时 → `normal`，更长 → `long`），所以一般不需要手动设 `PARTITION`，详见第 7 节。

---

## 4. 背景原理：这套东西是怎么工作的
>作者第一次上手NUS HPC时遇到了很多坑，官方文档偏简略，登录节点又有严格的资源和时长限制。谨此记录下原理，希望能帮到同样刚上手的人。

要理解 `exp` 在做什么，需要先理解两个角色：**Slurm**（管资源）、**tmux**（管会话）、以及它们如何配合。

### 4.1 Slurm：集群的资源调度器

集群上的机器分两类：

- **登录节点（login node）**：你 SSH 进来的地方，只用来编辑代码、提交作业，**不能**在这里跑重活；
- **计算节点（compute node）**：真正干活的机器，不能直接 SSH 上去，必须通过 Slurm 申请。

Slurm 的工作模式是：你向调度器描述"我要多少资源、用多久"，调度器找一台满足条件的空闲节点分配给你（这段分配叫 **allocation**，对应一个**作业 job**，有一个数字 **job ID**）。作业有生命周期：`PENDING`（排队中）→ `RUNNING`（已分配到节点）→ 结束（跑完 / 超时 / 被取消），结束时节点上属于这个作业的**所有进程都会被杀掉**，资源回收。

常用的四个命令：

| 命令 | 作用 | 特点 |
|---|---|---|
| `sbatch` | 提交**批处理作业** | 异步：提交后立即返回，作业排队等调度 |
| `srun` | 在作业内启动一个**作业步（job step）** | 同步：前台运行，可交互 |
| `squeue` | 查看作业队列 | 查状态、job ID、所在节点 |
| `scancel` | 取消作业 | 立即杀掉作业内所有进程并释放节点 |

### 4.2 sbatch：申请节点并挂起一个"守护"进程

`sbatch` 提交的是一个脚本（或用 `--wrap="命令"` 直接包一条命令），Slurm 分配到节点后，在该节点上执行它。**关键规则：这条命令退出的那一刻，作业就结束，节点就被回收。**

`exp start <name>` 实际执行的是：

```bash
sbatch --job-name="cc-<name>" \
       --time=9:00:00 \
       --exclude=xcng0,xcng1 \
       --wrap="tmux kill-session -t 'cc-<name>' 2>/dev/null; \
               tmux new-session -d -s 'cc-<name>' -c \$WORKDIR; \
               sleep infinity"
```

拆开看这条 `--wrap` 里的三段：

1. `tmux kill-session ...`：防御性清理。万一这个节点上残留着同名的旧会话（比如上次作业异常退出没清干净），先杀掉，保证新会话是干净的；
2. `tmux new-session -d -s <会话名> -c <工作目录>`：在**计算节点上**启动一个 tmux 会话。`-d`（detached）表示后台创建、不进入——因为 sbatch 作业没有终端，也没人在看它；`-c` 指定会话的初始工作目录（配置里的 `WORKDIR`）；
3. `sleep infinity`：**这是整个设计的关键。** `tmux new-session -d` 创建完会话就立即返回了，如果 `--wrap` 的命令到此结束，Slurm 会认为"作业跑完了"，立刻回收节点、杀掉刚建好的 tmux。所以用一个永远不退出的 `sleep infinity` 把作业"占住"，让 allocation 一直活着，直到 `--time` 用完或被 `scancel`。

所以 `exp start` 之后，节点上的进程结构是：

```
Slurm 作业 cc-<name>（占住节点，寿命 = TIME）
 ├── sleep infinity          ← 撑住作业不结束
 └── tmux server
      └── 会话 cc-<name>
           └── bash（在 WORKDIR 下，等你进来敲 claude）
```

### 4.3 tmux：让会话和终端解耦

tmux（terminal multiplexer，终端复用器）采用 **server–client 架构**：

- **tmux server** 是一个后台守护进程，真正持有你的 shell 和里面跑的所有程序；
- 你在终端里看到的只是一个 **tmux client**，它连上 server，把画面转发给你。

这带来的核心性质是：**client 断开（detach）不影响 server 里的程序继续运行。** 你 SSH 断线、`srun` 退出、笔记本合盖——都只是 client 没了，节点上 tmux server 里的 claude 进程照常跑。下次再 attach 上去，画面和状态原样还在。

这正是 `exp` 需要 tmux 的原因：Slurm 作业提供的是"资源存活"，tmux 提供的是"交互会话存活"，两者合起来才是一个可以随时进出的持久实验环境。

最常用的 tmux 操作（tmux 的快捷键都是先按前缀 `Ctrl+B`，松开，再按一个键）：

| 操作 | 按键 |
|---|---|
| **脱离会话（detach）** | `Ctrl+B` 再 `D` |
| 新建窗口 | `Ctrl+B` 再 `C` |
| 切换到下一个/上一个窗口 | `Ctrl+B` 再 `N` / `P` |
| 左右分屏 / 上下分屏 | `Ctrl+B` 再 `%` / `"` |
| 在分屏间移动 | `Ctrl+B` 再方向键 |
| 翻看历史输出（滚屏） | `Ctrl+B` 再 `[`，之后用方向键/PgUp，按 `q` 退出 |

注意：在会话里敲 `exit` 会关掉 shell，如果那是会话里最后一个窗口，tmux 会话就没了（但 Slurm 作业还占着节点——见第 4 节 FAQ）。想离开请用 detach，不要用 exit。

### 4.4 srun --overlap --pty：钻进已有作业里开终端

`exp attach <name>`（以及 `exp start` 的自动进入）实际执行的是：

```bash
srun --jobid=<该作业的ID> --overlap --pty tmux attach -t "cc-<name>"
```

`srun` 通常用来提交交互作业，但配合 `--jobid` 它有另一个用途：**不新申请资源，而是在一个已经 RUNNING 的作业内部再起一个作业步（job step）**——相当于"借道"进入那台已经分配给你的计算节点。逐个参数看：

- `--jobid=<ID>`：指定钻进哪个作业。这就是为什么 attach 前脚本要先用 `squeue` 查出 job ID，也是为什么作业还在 PENDING 时无法 attach——节点都还没分配，没有地方可进；
- `--overlap`：**没有它这一步会永远卡住。** 一个作业的 CPU 资源已经被第一个作业步（就是 `--wrap` 里那个 `sleep infinity`，Slurm 称之为 batch step）占用了。srun 新起的 step 默认要等到有空闲资源才运行，于是会一直等下去。`--overlap` 明确告诉 Slurm："这个 step 允许和已有 step 共享资源，不用等"；
- `--pty`：给这个 step 分配一个**伪终端（pseudo-terminal）**。srun 默认只是转发 stdin/stdout 字节流，没有真正的 TTY，而 `tmux attach` 是全屏交互程序，必须有 TTY 才能工作（否则会报 `not a terminal` 之类的错误）；
- `tmux attach -t <会话名>`：在计算节点上执行，连上第 5.2 节里建好的那个 tmux 会话。

于是数据链路是：你的终端 ←(SSH)→ 登录节点 ←(srun 转发)→ 计算节点上的 tmux client ←→ tmux server。你按 `Ctrl+B D` detach 时，tmux client 退出 → srun 这个 step 结束 → 你回到登录节点的 shell；而计算节点上 tmux server 和 `sleep infinity` 都没受影响，实验继续跑。

### 4.5 squeue / scancel：按名字管理作业

`exp` 的一个设计约定是：**用作业名（job name）作为实验的身份标识**，格式为 `PREFIX + 实验名`（默认前缀 `cc-`，如实验 `myrun` → 作业名 `cc-myrun`）。前缀的作用是把 exp 管理的作业和你手动提交的其他作业区分开，`stopall` 只会杀带前缀的作业，不会误伤。

- `exp list` = `squeue -u $USER` 后按前缀过滤，显示 job ID、作业名、状态（`R`=运行 / `PD`=排队）、已运行时长、所在节点；排队中的作业会在最后一列显示排队原因（如 `(Priority)`、`(Resources)`）；
- `exp stop <name>` = `scancel -u $USER --name=cc-<name>`，按作业名精确取消；
- `exp stopall` = 列出所有 `cc-` 开头的作业，逐个 `scancel`。

`scancel` 杀作业时，Slurm 会终止作业内所有进程（tmux、claude、你跑的一切），节点立即回收。**stop 前确认实验里没有还需要的东西。**

---

## 5. 一图流：整体生命周期

```
 exp start foo                          自动/手动 attach            Ctrl+B D
──────────────►  PENDING ──► RUNNING ──────────────────►  交互中  ──────────►  后台继续跑
   (sbatch)      (排队)     节点上启动:    (srun --overlap        │                │
                            tmux server     --pty tmux attach)    │                │ 可反复 attach/detach
                            + sleep inf                           ▼                ▼
                                                        ┌──────────────────────────────┐
                                                        │  作业结束条件(二选一):        │
                                                        │  · exp stop / stopall        │
                                                        │  · TIME 用尽                 │
                                                        │  → tmux 及全部进程被杀,      │
                                                        │    节点回收                  │
                                                        └──────────────────────────────┘
```

---

## 6. 节点类型说明：CPU 节点

**exp 申请到的是 CPU 节点，申请 GPU 节点的功能目前未实现** 
> 你问为什么不能申请GPU节点？这种事嘛，我想就交给万能的 Claude Code 大人吧。
