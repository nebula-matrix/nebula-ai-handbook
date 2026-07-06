# svn:externals 创建icv-skills使用说明 —— 引入 icv-sim 系列 skill

## 省流版本

在 `<你的模块>` 验证环境根目录(如 `verification/bt/pa`)下两步完成部署:

```bash
# 1) 从共享目录复制部署脚本并加可执行权限
cp /mnt/public_share/setup/setup_icv_externals.sh <你的验证环境根目录>/
chmod +x <你的验证环境根目录>/setup_icv_externals.sh

# 2) 进入模块目录执行脚本(默认 install: 自动 qsub 到集群节点完成 externals 属性设置 + svn up + commit,检出 4 个 icv-sim-* skill)
cd <你的验证环境根目录>          # 例如 verification/bt/pa
bash setup_icv_externals.sh              # 等价于 bash setup_icv_externals.sh install
```

**卸载**(移除 4 个 icv-sim-* externals 并 commit):

```bash
bash setup_icv_externals.sh uninstall
```

**成功标志**:脚本末尾输出 `===== 集群执行成功 =====`,`.claude/skills/` 下出现(或卸载后消失)`icv-sim-analyze`、`icv-sim-cli`、`icv-sim-cov`、`icv-sim-run` 四个子目录,且本次产生的临时文件已自动清理。

**后续 skill 更新**(`bt/uvn` 那边的 skill 内容更新提交后,无需重新部署):

```bash
cd <你的验证环境根目录>/.claude/skills
svn up .                          # 自动同步 externals 指向的最新 skill
```

> ⚠️ **关键约束**:externals 挂接的 4 个目录是 `bt/uvn` 的「软链接」,**禁止在当前模块内修改并提交**(会回传影响所有模块)。需通用改动请去 `bt/uvn` 改;需自行迭代请复制副本到非 externals 路径(如 `.claude/skills_local/`)另行维护。

> 📌 **执行环境**:脚本**在 SGE 集群节点上实际执行 svn 操作**(本地 svn 1.14 工作副本格式过旧,集群 svn 1.7 才兼容)。本地运行脚本时,它会自动 `qsub` 把任务投递到集群节点,本地阻塞等待结果——对使用者而言仍是"本地一条命令搞定"。详见 §3。

> 详细原理、四个 skill 的作用、部署细节与故障排查见下文。

## 1. 概述

`bt/uvn` 模型下统一维护了一套 IC 验证调试用的 Claude Code skill(`icv-sim-*`),覆盖编译仿真、仿真分析、覆盖率查询与 tw CLI 工具层。为避免在每个模型(pa、ped、ipro 等模块)里重复维护一份,使用 Subversion 的 **`svn:externals`** 机制,把这些 skill 以「外部引用」的方式挂接到各模型自己的 `.claude/skills` 目录。

效果:在 `verification/<bt|it|st>/<你的模块>/.claude/skills` 下执行一次属性设置 + `svn up`,该目录就会多出 4 个 skill 子目录,内容始终指向 `bt/uvn` 仓库版本,后续 `svn up` 即可同步更新,无需手动拷贝。

> 说明:`svn:externals` 是 svn 的目录属性,值是一份「外部目录映射列表」,格式为 `<本地子目录名> <远端URL>`,`svn up` 时会自动把远端 URL 检出到对应子目录。

### 脚本获取方式

部署脚本 `setup_icv_externals.sh` 统一存放在共享目录,需要时复制到自己的工作目录(验证环境根目录)即可,不涉及仓库内分发:

| 位置 | 用途 |
|------|------|
| `/mnt/public_share/setup/setup_icv_externals.sh` | 统一取用入口,复制到各模型工作目录后执行 |

> 该共享目录由管理员维护,所有用户可读取/复制。如需旧版回退,备份为同目录下 `bak-setup_icv_externals.sh`(由管理员存放)。

获取与执行:

```bash
# 1) 从共享目录复制脚本到你的工作目录(验证环境根目录,即 <你的模块> 目录)
cp /mnt/public_share/setup/setup_icv_externals.sh <你的验证环境根目录>/
# 2) 复制后建议补一次 +x,cp 不带 -p 时副本权限受 umask 影响可能丢可执行位
chmod +x <你的验证环境根目录>/setup_icv_externals.sh
# 3) 在工作目录下执行
cd <你的验证环境根目录>
bash setup_icv_externals.sh
```


## 2. 四个 icv 开头 skill 的作用与应用场景

四个 skill 均来自 `bt/uvn/.claude/skills/`,按「工具层 + 业务层」分工:`icv-sim-cli` 是工具底座(安装/总览),另三个是业务层,业务层均以前置「tw 已装」为条件。

### 2.1 icv-sim-cli —— tw CLI 工具层(安装/环境/总览)

- **作用**:`tw` CLI 的安装、环境检查与功能总览。
- **应用场景**:
  - 首次安装 tw(`scripts/install.sh` 一次性离线安装);
  - 检查 Verdi / VCS / FSDB 环境(`tw env check`);
  - 发现项目结构 / 有哪些 testcase(`tw project discover`);
  - 纯粹想了解 tw 有哪些命令组(功能总览,不附带任何执行动作)。
- **定位**:工具层,**不负责**编译/仿真/调试/覆盖率等业务执行。业务请求直接进对应业务 skill,不要把本 skill 当成业务 skill 的前置入口。
- **前置**:无(它本身就是安装入口)。

### 2.2 icv-sim-run —— 编译 / 仿真 / 作业管理

- **作用**:封装项目 Makefile 的 `cmp`/`run`/`batch`/`urg`/`merge`/`plan`,提交 SGE 集群并跟踪作业。
- **应用场景**(任何「把设计 build 出来、跑起来」的执行诉求):
  - 编译(`make cmp`)、重新编译;
  - 跑仿真(`make run`/`batch`),含补跑波形(`--wave fsdb`);
  - 覆盖率后处理(`make urg`/`merge`/`plan`);
  - 提交作业到集群、跟踪作业状态/日志(`status`/`wait`/`stop`/`log`);
  - 典型话术:「simv/KDB 没生成要先编」「代码改完重编再跑」「帮我编译一下项目」「跑带覆盖率的仿真」。
- **前置**:tw 已装(见 icv-sim-cli)。

### 2.3 icv-sim-analyze —— 仿真分析(日志/波形/连通性/层次)

- **作用**:分析仿真到底怎么跑的——从日志、波形、连通性、层次结构入手,既包括失败定位,也包括事后量化。
- **应用场景**:
  - 波形查询(信号值/跳变/窗口/scope);
  - 信号与握手统计(跳变计数、transfer 次数、占空比、仿真时长);
  - log 解析与失败定位(`UVM_ERROR`/`UVM_FATAL`/X 传播);
  - 连通性追踪(driver/loads、active trace、X 溯源);
  - TB 层次结构查找。
- **前置**:tw 已装;KDB(`tw make cmp` 产生);FSDB 波形(`tw make run/batch --wave fsdb` 产生,非默认)。

### 2.4 icv-sim-cov —— 覆盖率查询

- **作用**:查询 VCS 覆盖率数据库(VDB)。
- **应用场景**:
  - 覆盖率多少(scope 聚合)、哪些代码没覆盖到(holes)、列出 test handles;
  - 设计覆盖:line / toggle / branch / condition / fsm / assert 六类指标;
  - 功能覆盖:covergroup / coverpoint / cross / bin(`tw cov functional`);
  - 与 urgReport 口径对齐(`tw cov report`,整体 SCORE + assert)。
- **前置**:tw 已装;VDB 数据(`tw make run --ccov` 产生,`tw cov` 只查询不生成)。

### 2.5 四者关系速记

| skill | 层次 | 一句话 | 前置 |
|-------|------|--------|------|
| icv-sim-cli | 工具层 | 装 tw、查环境、看总览 | 无 |
| icv-sim-run | 业务层 | 编译/跑仿真/管作业 | tw 已装 |
| icv-sim-analyze | 业务层 | 看日志/波形/trace 调试 | tw 已装 + KDB/FSDB |
| icv-sim-cov | 业务层 | 查覆盖率 VDB | tw 已装 + VDB |

## 3. 部署方式(bash 一键部署,集群执行)

### 3.1 脚本放置位置

从共享目录复制来的 `setup_icv_externals.sh` 放在**验证环境根目录**(模型目录),即 `<你的模块>` 目录下。以 pa 模型为例,复制后位于:

```
verification/bt/pa/setup_icv_externals.sh
```

### 3.2 目录层级预检查

脚本启动时会校验目录层级,必须满足:

```
<verification>/<bt|it|st>/<model>/setup_icv_externals.sh
    再上一级      上一级     脚本所在目录
```

即:脚本所在目录的**上一级**应为 `bt` / `it` / `st` 之一,**再上一级**应为 `verification`。以 pa 模型为例(`verification/bt/pa`):
- 上一级 = `bt` ✓
- 再上一级 = `verification` ✓

> 注:脚本按实际仓库结构 `verification/<bt|it|st>/<model>` 校验。若你的目录层级不同,请对应调整。

### 3.3 执行环境:本地自提交 + 集群节点执行

脚本涉及 svn 操作必须在 **SGE 集群节点**上执行(本地 svn 1.14 工作副本格式过旧,集群 svn 1.7 才兼容)。脚本采用**本地自提交**模式,对使用者完全透明:

```
本地 shell 执行 bash setup_icv_externals.sh install
        │
        ▼
本地侧: 解析参数 → 生成临时 worker 脚本 → qsub 投递到集群(队列 all.q)
        │
        ▼
集群节点: 执行 install/uninstall 主逻辑(svn propset/propdel + svn up + commit)
        │  写结果标记文件 + sync 到 NFS
        ▼
本地侧: 轮询 qstat 等作业结束 → 读结果标记 → 打印成功/失败 → 清理临时文件
```

- 本地侧只负责 qsub 投递 + 阻塞等待 + 结果判定,**不直接操作 svn 工作副本**。
- 集群节点执行实际的 svn 命令,commit message 固定为 `set svn:externals for icv-sim skills`(install)/ `remove svn:externals for icv-sim skills`(uninstall)。
- 成功后自动清理本次产生的临时文件(worker 脚本/日志/结果标记);**失败时保留**临时文件以便排查(脚本会打印日志路径)。

### 3.4 双模式参数

```bash
bash setup_icv_externals.sh [install|uninstall]
```

| 参数 | 作用 | commit message |
|------|------|----------------|
| `install`(默认,可省略) | 设置 `svn:externals` + `svn up` + 自动 commit | `set svn:externals for icv-sim skills` |
| `uninstall` | `propdel svn:externals` + `svn up` 移除 4 目录 + 删除中间文件 + 自动 commit | `remove svn:externals for icv-sim skills` |
| `-h` / `--help` | 显示用法 | — |

- **无参数**等价于 `install`。
- **多余参数**会报错退出(commit message 固定,不接受额外参数)。
- **install 幂等**:已部署状态下重跑 install,重新 propset 相同内容 + commit,安全。
- **uninstall 幂等**:已卸载状态下再跑 uninstall,脚本检测到无 icv-sim-* externals 会优雅地输出"无需卸载, 视为成功"并以 exit 0 结束(不会报错)。

### 3.5 install 执行步骤(集群节点上按顺序完成)

1. **预检查与目录准备**:目录层级、`svn` 命令可用、模型目录是 svn 工作副本。若 `.claude/skills` 不存在,脚本会逐级创建(`mkdir -p .claude/skills`)并通过 `svn add` 纳入版本控制;若已存在,则校验其已被 svn 跟踪。
2. **生成 externals 列表文件**:在 `.claude/skills/` 下生成 `icv-externals-list.txt`,内容优先取自脚本同目录的 `icv验证调试导入说明.md`(已写好 4 行 externals 条目);该文件缺失时回退到脚本内嵌默认值。
3. **设置属性 + 检出**:在 `.claude/skills` 下执行 `svn propset svn:externals -F icv-externals-list.txt .` 与 `svn up .`,检出 4 个 skill 子目录。
4. **范围检查(安全网)**:`svn status .claude/skills` 只允许 icv-sim-* 相关变更(属性变更、externals 标志、中间文件);若存在**与 icv-sim-* 无关的未提交改动**,脚本**中止 commit** 并报错,避免误提交他人改动。
5. **自动 commit**:`svn commit .claude/skills -m "set svn:externals for icv-sim skills"`。
6. **成功校验**:检查 4 个 `icv-sim-*` 目录到位。

### 3.6 uninstall 执行步骤(集群节点上按顺序完成)

1. **预检查**:目录层级、svn 工作副本、`.claude/skills` 已纳入 svn 跟踪。
2. **前置状态检查**:读取现有 `svn:externals` 属性。若不含任何 icv-sim-* 条目(未安装或已卸载),输出"无需卸载, 视为成功",exit 0(幂等);若只含部分 icv-sim-*(异常中间态),报错退出避免误删其他 externals。
3. **删除属性 + 移除外部目录**:`svn propdel svn:externals .claude/skills` + `svn up .`,svn 会移除 4 个 external 目录的版本控制部分;脚本再主动清理目录内可能的未版本控制残留(如 `__pycache__`)。
4. **删除中间文件**:删除 `.claude/skills/icv-externals-list.txt`(已版本控制则 `svn delete --keep-local` 纳入删除调度)。
5. **范围检查 + 自动 commit**:同 install 的安全网,commit message 为 `remove svn:externals for icv-sim skills`。
6. **成功校验**:确认 4 个目录已消失、属性已清空。

### 3.7 一键执行

```bash
cd verification/<bt|it|st>/<你的模块>     # 例如 cd verification/bt/pa
bash setup_icv_externals.sh               # install(默认)
bash setup_icv_externals.sh uninstall     # 卸载
```

### 3.8 成功标志

脚本末尾输出 `===== 集群执行成功 =====` 并 `exit 0`。

- **install 后**:`.claude/skills` 目录下出现 4 个子目录:
  ```
  .claude/skills/
  ├── icv-sim-analyze/
  ├── icv-sim-cli/
  ├── icv-sim-cov/
  └── icv-sim-run/
  ```
- **uninstall 后**:上述 4 个子目录与 `icv-externals-list.txt` 均移除,`svn:externals` 属性清空。

> 临时文件(`.setup_icv_worker_*` / `.setup_icv_result_*`)在成功后自动清理;失败时保留,脚本会打印其路径供排查。

### 3.9 后续步骤

**后续 skills 更新同步(类似软链接):** `svn:externals` 的 4 个 skill 目录始终指向 `bt/uvn` 仓库版本,相当于对远端的「软链接」。`bt/uvn` 那边的 skill 内容更新并提交后,**你无需重新部署、也无需手动拷贝**——只需在自己的验证环境执行一次 `svn up`, externals 挂接的 4 个目录就会自动同步到最新版本:

```bash
cd verification/<bt|it|st>/<你的模块>/.claude/skills
svn up .                                  # 拉取 externals 指向的最新 skill 内容
```

> 同理,团队其他成员首次 `svn up` 该目录时会自动挂接 4 个 skill;之后每次 `svn up` 即同步更新,无需各自跑部署脚本。

## 4. 注意事项与故障排查

- **externals 是只读引用,禁止在当前模块内修改提交**:4 个 `icv-sim-*` 目录由 `bt/uvn` 统一维护,通过 `svn:externals` 挂接进来后相当于远端的「软链接」(详见 §3.9)。**不要在当前模块内对这些目录做修改并提交**——externals 目录的修改会回传到 `bt/uvn` 源仓库,影响所有挂接该 skill 的模块,易造成他人环境异常。
  - **需要通用改动时**:请到 `bt/uvn/.claude/skills/` 修改并提交,各模块 `svn up` 即可同步。
  - **有自行迭代需求时(不想影响他人、也不想被远端更新覆盖)**:不要动 externals 挂接的目录,而是**复制一份源文件到自己的工作目录另行维护,做好版本隔离**。例如复制到 `.claude/skills_local/`(或其它非 externals 路径)并重命名,改用本地副本;同时从 `icv-externals-list.txt` 中移除对应条目,避免本地副本与 externals 引用并存冲突。这样你的定制版本独立演进,既不会回传影响 `bt/uvn`,也不会在 `svn up` 时被远端更新覆盖。
- **`.claude/skills` 的创建与 svn 跟踪**:脚本会自动处理——目录不存在时逐级 `mkdir -p` 创建并 `svn add` 纳入版本控制;目录已存在时校验其已被 svn 跟踪。前提是**脚本所在模型目录本身必须是 svn 工作副本**(脚本启动时会 `svn info` 校验)。若模型目录不是 svn 工作副本,脚本会直接退出并提示;若该目录已被 `svn:ignore` 排除,需先解除忽略或换到未被忽略的位置。
- **范围检查 abort(commit 前安全网)**:install/uninstall 在 commit 前会检查 `.claude/skills` 下的 svn 变更,**只允许 icv-sim-* 相关改动**。若该目录存在其他未提交改动(例如你自己加的文件、其他 skill 的修改),脚本会报 `范围检查失败: ...存在与 icv-sim-* 无关的未提交变更` 并**中止 commit**(不自动提交,避免误伤)。处理:先手动 commit/revert 那些无关改动,再重跑脚本。
- **uninstall 报"无需卸载, 视为成功"**:这是**正常行为**——说明当前 `.claude/skills` 已无 icv-sim-* externals(尚未安装或已卸载过),脚本幂等退出,exit 0。若你期望的是真正卸载,说明此前已卸载完成,无需再操作。
- **集群作业失败**:脚本报 `===== 集群执行失败 =====` 并显示 `FAIL: 退出 code=...` 时,按提示查看完整日志 `.setup_icv_worker_<mode>.out`(失败时临时文件保留)。常见原因:svn 凭证未缓存(节点上 `svn list` 测试)、SVN 服务器不可达、工作副本有冲突等。
- **作业异常终止(结果标记文件缺失)**:极少见,通常是作业被 `qdel` 或节点异常崩溃。查看 `.setup_icv_worker_<mode>.out` 末尾排查。
- **本地侧提示"NFS 缓存"相关 warn**:集群端写结果文件后,本地经 NFS 读取偶有缓存延迟。脚本已内置 sync + 重试 + outlog 降级判定,通常无需关注;若仍偶发,重跑一次即可。


## 5. 脚本内容

脚本统一存放在共享目录,不随仓库分发,需要时直接查看或复制:

```
/mnt/public_share/setup/setup_icv_externals.sh
```

查看脚本内容:

```bash
cat /mnt/public_share/setup/setup_icv_externals.sh
less /mnt/public_share/setup/setup_icv_externals.sh    # 翻页查看
```

脚本逻辑概要(详见 §3.5/§3.6 执行步骤):
- **本地侧**:参数解析(install/uninstall/-h)→ 生成临时 worker 脚本 → `qsub` 投递到集群(all.q)→ 轮询 `qstat` 阻塞等待 → 读结果标记判定成败 → 成功清理临时文件/失败保留并提示日志。
- **集群节点侧(worker)**:目录层级预检查 → svn 工作副本校验 → install(propset + svn up + 范围检查 + commit)/ uninstall(前置检查 + propdel + svn up + 清理残留 + 删中间文件 + 范围检查 + commit)→ 写结果标记 + `sync` + 打印降级标记。
- **安全机制**:commit 前范围检查(只允许 icv-sim-* 相关变更)、EXIT trap 兜底写失败标记、result 文件名带 jobid 避免 NFS stale、outlog 降级判定。

