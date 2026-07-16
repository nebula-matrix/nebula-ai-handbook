# svn:externals 创建icv-skills使用说明 —— 引入 icv-sim 系列 skill

## 省流版本

在 `<你的模块>` 验证环境根目录(如 `verification/bt/pa`)下直接执行脚本即可:

```bash
cd <你的验证环境根目录>          # 例如 verification/bt/pa
bash setup_icv_externals.sh
```

**成功标志**:脚本末尾输出 `install 成功。/.../.claude/skills 已包含 4 个 icv-sim-* skill:`,`.claude/skills/` 下出现 `icv-sim-analyze`、`icv-sim-cli`、`icv-sim-cov`、`icv-sim-run` 四个子目录。

**后续 skill 更新**(`bt/uvn` 那边的 skill 内容更新提交后,无需重新部署):

```bash
cd <你的验证环境根目录>/.claude/skills
svn up .                          # 自动同步 externals 指向的最新 skill
```

> ⚠️ **关键约束**:externals 挂接的 4 个目录是 `bt/uvn` 的「软链接」,**禁止在当前模块内修改并提交**(会回传影响所有模块)。需通用改动请去 `bt/uvn` 改;需自行迭代请复制副本到非 externals 路径(如 `.claude/skills_local/`)另行维护。

> 详细原理、四个 skill 的作用、部署细节与故障排查见下文。

## 1. 概述

`bt/uvn` 模型下统一维护了一套 IC 验证调试用的 Claude Code skill(`icv-sim-*`),覆盖编译仿真、仿真分析、覆盖率查询与 tw CLI 工具层。为避免在每个模型(pa、ped、ipro 等模块)里重复维护一份,使用 Subversion 的 **`svn:externals`** 机制,把这些 skill 以「外部引用」的方式挂接到各模型自己的 `.claude/skills` 目录。

效果:在 `verification/<bt|it|st>/<你的模块>/.claude/skills` 下执行一次属性设置 + `svn up`,该目录就会多出 4 个 skill 子目录,内容始终指向 `bt/uvn` 仓库版本,后续 `svn up` 即可同步更新,无需手动拷贝。

> 说明:`svn:externals` 是 svn 的目录属性,值是一份「外部目录映射列表」,格式为 `<本地子目录名> <远端URL>`,`svn up` 时会自动把远端 URL 检出到对应子目录。

### 脚本位置

部署脚本 `setup_icv_externals.sh` 已纳入模型仓库,位于验证环境根目录(模型目录):

```
verification/<bt|it|st>/<你的模块>/setup_icv_externals.sh
```

以 pa 模型为例,即 `verification/bt/pa/setup_icv_externals.sh`。

## 2. 执行环境与回退机制

脚本**默认在本地直接执行 svn 操作**。本地 svn 不可用时,自动回退为 qsub 提交到 SGE 集群节点执行。

### 本地优先

进入模型目录直接执行即可:

```bash
cd verification/<bt|it|st>/<你的模块>
bash setup_icv_externals.sh
```

脚本会检查本地 `svn` 是否可用、模型目录是否为有效 svn 工作副本,然后直接在本地完成 `svn:externals` 设置 / 删除 + `svn up` + 自动 `svn commit`。

### 回退到集群 qsub 的触发条件

只有在以下两类环境性问题时才会回退:

1. **本地未找到 `svn` 命令**;
2. **本地 `svn info` 报 `E155036`(工作副本格式过旧)**。

上述两种情况会生成 worker 脚本 `qsub` 到 SGE 集群(默认队列 `all.q`),由集群节点上的 svn 环境完成操作。本地阻塞等待 `qstat` 结束,并自动判定结果。

### 不会回退、会直接报错的情况

以下 svn 业务错误即使回退到集群也大概率同样失败,脚本会直接报错退出,避免掩盖真因:

- 模型目录根本不是 svn 工作副本(`E155007` 等);
- 网络超时、认证失败(`E170013`/`E170001`);
- 工作副本存在冲突(`C`);
- externals URL 错误、服务器无响应等。

## 3. 四个 icv 开头 skill 的作用与应用场景

四个 skill 均来自 `bt/uvn/.claude/skills/`,按「工具层 + 业务层」分工:`icv-sim-cli` 是工具底座(安装/总览),另三个是业务层,业务层均以前置「tw 已装」为条件。

### 3.1 icv-sim-cli —— tw CLI 工具层(安装/环境/总览)

- **作用**:`tw` CLI 的安装、环境检查与功能总览。
- **应用场景**:
  - 首次安装 tw(`scripts/install.sh` 一次性离线安装);
  - 检查 Verdi / VCS / FSDB 环境(`tw env check`);
  - 发现项目结构 / 有哪些 testcase(`tw project discover`);
  - 纯粹想了解 tw 有哪些命令组(功能总览,不附带任何执行动作)。
- **定位**:工具层,**不负责**编译/仿真/调试/覆盖率等业务执行。业务请求直接进对应业务 skill,不要把本 skill 当成业务 skill 的前置入口。
- **前置**:无(它本身就是安装入口)。

### 3.2 icv-sim-run —— 编译 / 仿真 / 作业管理

- **作用**:封装项目 Makefile 的 `cmp`/`run`/`batch`/`urg`/`merge`/`plan`,提交 SGE 集群并跟踪作业。
- **应用场景**(任何「把设计 build 出来、跑起来」的执行诉求):
  - 编译(`make cmp`)、重新编译;
  - 跑仿真(`make run`/`batch`),含补跑波形(`--wave fsdb`);
  - 覆盖率后处理(`make urg`/`merge`/`plan`);
  - 提交作业到集群、跟踪作业状态/日志(`status`/`wait`/`stop`/`log`);
  - 典型话术:「simv/KDB 没生成要先编」「代码改完重编再跑」「帮我编译一下项目」「跑带覆盖率的仿真」。
- **前置**:tw 已装(见 icv-sim-cli)。

### 3.3 icv-sim-analyze —— 仿真分析(日志/波形/连通性/层次)

- **作用**:分析仿真到底怎么跑的——从日志、波形、连通性、层次结构入手,既包括失败定位,也包括事后量化。
- **应用场景**:
  - 波形查询(信号值/跳变/窗口/scope);
  - 信号与握手统计(跳变计数、transfer 次数、占空比、仿真时长);
  - log 解析与失败定位(`UVM_ERROR`/`UVM_FATAL`/X 传播);
  - 连通性追踪(driver/loads、active trace、X 溯源);
  - TB 层次结构查找。
- **前置**:tw 已装;KDB(`tw make cmp` 产生);FSDB 波形(`tw make run/batch --wave fsdb` 产生,非默认)。

### 3.4 icv-sim-cov —— 覆盖率查询

- **作用**:查询 VCS 覆盖率数据库(VDB)。
- **应用场景**:
  - 覆盖率多少(scope 聚合)、哪些代码没覆盖到(holes)、列出 test handles;
  - 设计覆盖:line / toggle / branch / condition / fsm / assert 六类指标;
  - 功能覆盖:covergroup / coverpoint / cross / bin(`tw cov functional`);
  - 与 urgReport 口径对齐(`tw cov report`,整体 SCORE + assert)。
- **前置**:tw 已装;VDB 数据(`tw make run --ccov` 产生,`tw cov` 只查询不生成)。

### 3.5 四者关系速记

| skill | 层次 | 一句话 | 前置 |
|-------|------|--------|------|
| icv-sim-cli | 工具层 | 装 tw、查环境、看总览 | 无 |
| icv-sim-run | 业务层 | 编译/跑仿真/管作业 | tw 已装 |
| icv-sim-analyze | 业务层 | 看日志/波形/trace 调试 | tw 已装 + KDB/FSDB |
| icv-sim-cov | 业务层 | 查覆盖率 VDB | tw 已装 + VDB |

## 4. 部署方式(bash 一键部署)

### 4.1 脚本放置位置

脚本应放在验证环境根目录(模型目录),即 `<你的模块>` 目录下。以 pa 模型为例:

```
verification/bt/pa/setup_icv_externals.sh
```

### 4.2 运行环境预检查

脚本启动时会校验:

- 脚本所在目录下存在 `sim/` 目录;
- `sim/` 下存在 `Makefile`(不区分大小写);
- `svn` 命令可用、模型目录是 svn 工作副本。

若 `.claude` 或 `.claude/skills` 尚未被 svn 跟踪,脚本会自动处理:

- 目录不存在时逐级 `mkdir -p` 创建;
- 使用 `svn add --depth empty` 只把 `.claude` 和 `.claude/skills` 这两个**空目录节点**纳入版本控制,
  **不会递归添加其内部已有的其它文件或目录**(如 `settings.json`、`.claude/tmp/`、`.claude/skills/xxx_demo_skills` 等)。

前提是:

- 脚本所在模型目录本身必须是 svn 工作副本;
- `.claude` 没有被父目录的 `svn:ignore` 排除(若被 ignore,`svn add` 会失败,脚本会提示先解除 ignore)。

### 4.3 执行步骤(脚本内部按顺序完成)

1. **运行环境预检查**:校验 `sim/Makefile`、`svn` 可用、模型目录是 svn 工作副本。若本地 svn 环境不可用(见 §2 回退条件),自动回退到集群 qsub。
2. **确保 `.claude` 与 `.claude/skills` 被 svn 跟踪**:目录不存在则创建;未跟踪则 `svn add --depth empty` 只添加空目录节点,**不递归**添加内部已有文件/目录。
3. **生成 externals 列表文件**:在 `.claude/skills/` 下生成 `icv-externals-list.txt`,内容优先取自脚本同目录的 `icv验证调试导入说明.md`(已写好 4 行 externals 条目);该文件缺失时回退到脚本内嵌默认值。
4. **设置属性**:在 `.claude/skills` 下执行
   ```bash
   svn propset svn:externals -F icv-externals-list.txt .
   ```
5. **范围检查与自动 commit**:脚本只提交 icv-sim-* 相关变更:
   - `.claude` 目录本身(首次为空目录节点);
   - `.claude/skills` 目录本身(携带 `svn:externals` 属性);
   - `.claude/skills/icv-externals-list.txt`。

   `.claude` 及 `.claude/skills` 下其它任意名称的未提交/无关项一律忽略,不报错也不提交。
6. **拉取外部目录**:在 `.claude/skills` 下执行 `svn up .`,检出 4 个 skill 子目录。注意:必须先 commit 父目录(步骤 5),否则 `.claude/skills` 尚无版本化父目录,`svn up` 会跳过 external。
7. **成功校验**:检查 `.claude/skills` 下是否包含 `icv-sim-analyze`、`icv-sim-cli`、`icv-sim-cov`、`icv-sim-run`,齐了即成功。

### 4.4 一键执行

```bash
cd verification/<bt|it|st>/<你的模块>     # 例如 cd verification/bt/pa
bash setup_icv_externals.sh
```

### 4.5 成功标志

脚本末尾输出 `install 成功。/path/to/.claude/skills 已包含 4 个 icv-sim-* skill:`,且 `.claude/skills` 目录下出现 4 个子目录:

```
.claude/skills/
├── icv-sim-analyze/
├── icv-sim-cli/
├── icv-sim-cov/
└── icv-sim-run/
```

### 4.6 后续步骤

**后续 skills 更新同步(类似软链接):** `svn:externals` 的 4 个 skill 目录始终指向 `bt/uvn` 仓库版本,相当于对远端的「软链接」。`bt/uvn` 那边的 skill 内容更新并提交后,**你无需重新部署、也无需手动拷贝**——只需在自己的验证环境执行一次 `svn up`, externals 挂接的 4 个目录就会自动同步到最新版本:

```bash
cd verification/<bt|it|st>/<你的模块>/.claude/skills
svn up .                                  # 拉取 externals 指向的最新 skill 内容
```

> 同理,团队其他成员首次 `svn up` 该目录时会自动挂接 4 个 skill;之后每次 `svn up` 即同步更新,无需各自跑部署脚本。

## 5. 注意事项与故障排查

- **externals 是只读引用,禁止在当前模块内修改提交**:4 个 `icv-sim-*` 目录由 `bt/uvn` 统一维护,通过 `svn:externals` 挂接进来后相当于远端的「软链接」(详见 §4.6)。**不要在当前模块内对这些目录做修改并提交**——externals 目录的修改会回传到 `bt/uvn` 源仓库,影响所有挂接该 skill 的模块,易造成他人环境异常。
  - **需要通用改动时**:请到 `bt/uvn/.claude/skills/` 修改并提交,各模块 `svn up` 即可同步。
  - **有自行迭代需求时(不想影响他人、也不想被远端更新覆盖)**:不要动 externals 挂接的目录,而是**复制一份源文件到自己的工作目录另行维护,做好版本隔离**。例如复制到 `.claude/skills_local/`(或其它非 externals 路径)并重命名,改用本地副本;同时从 `icv-externals-list.txt` 中移除对应条目,避免本地副本与 externals 引用并存冲突。这样你的定制版本独立演进,既不会回传影响 `bt/uvn`,也不会在 `svn up` 时被远端更新覆盖。
- **`.claude` 与 `.claude/skills` 的创建与 svn 跟踪**:脚本会自动处理——目录不存在时逐级 `mkdir -p` 创建,并只把 `.claude` 和 `.claude/skills` 这两个目录本身(`svn add --depth empty`)纳入版本控制,**不会递归添加目录内部的其它文件/目录**(如 `settings.json`、`.claude/worktrees/`、`.claude/tmp/`、`.claude/skills/xxx_demo_skills` 等)。目录已存在但未被跟踪时,同样只 add 空目录节点,内部已有内容保持未跟踪。前提是:
  - **脚本所在模型目录本身必须是 svn 工作副本**(脚本启动时会 `svn info` 校验);
  - `.claude` 没有被父目录的 `svn:ignore` 排除。若 `.claude` 被 ignore,`svn add` 会失败,脚本会提示先解除 ignore。

  若模型目录不是 svn 工作副本,脚本会直接报错退出;若 `.claude` 被 `svn:ignore` 排除,需先在模型目录解除忽略(例如 `svn propdel svn:ignore .` 或从 ignore 列表中移除 `.claude`)后再运行本脚本。
- **`.claude` 及 `.claude/skills` 下其它未提交项**:脚本只提交 icv-sim-* 相关的目录节点、属性与 `icv-externals-list.txt`,其它任意名称的未跟踪或改动项(如 `ptype-fanout`、`xxx_demo_skills`、`xxx_yyy`)会被忽略,不会报错也不会被提交。如果你希望它们不再出现在 `svn status` 里,可手动加入 `svn:ignore`。
- **首次安装的 commit 顺序**:首次安装时 `.claude` 与 `.claude/skills` 可能还是新增(`A`)状态,脚本会先 commit 这些空目录节点与 `svn:externals` 属性,然后再执行 `svn up` 拉取 4 个 `icv-sim-*` 外部目录。如果顺序颠倒,`svn up` 会因“无版本化父目录”而跳过 external。
- **临时文件**:本地执行时脚本可能在模型根目录生成 `.setup_icv_*` 开头的临时文件(回退 qsub 时还会生成 worker 脚本和 out 日志)。成功执行后这些临时文件会被自动清理;失败时会保留以便排查,已被 `.gitignore` 忽略,不会误入 git 提交。

## 6. 脚本内容

脚本已纳入模型仓库,位于验证环境根目录,直接查看或执行即可:

```
verification/<bt|it|st>/<你的模块>/setup_icv_externals.sh
```

脚本逻辑概要(详见 §4.3 执行步骤):预检查运行环境 → 确保 `.claude` 与 `.claude/skills` 被 svn 跟踪(必要时 `--depth empty` add 空目录节点)→ 在 `.claude/skills` 生成 `icv-externals-list.txt`(内嵌默认 4 行 externals)→ `svn propset svn:externals -F` → 范围检查并 commit(只提交 icv-sim 相关项:`.claude`、`.claude/skills`、`icv-externals-list.txt`)→ `svn up` 拉取 external → 校验 4 个 `icv-sim-*` 目录到位。
