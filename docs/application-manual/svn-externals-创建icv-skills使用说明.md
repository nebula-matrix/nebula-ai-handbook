# svn:externals 创建icv-skills使用说明 —— 引入 icv-sim 系列 skill

## 省流版本

在 `<你的模块>` 验证环境根目录(如 `verification/bt/pa`)下直接执行脚本即可:

```bash
cd <你的验证环境根目录>          # 例如 verification/bt/pa
bash setup_icv_externals.sh                    # 默认 install, 装 6 个 icv-sim-* skill
```

**成功标志**:脚本末尾输出 `install 成功。/.../.claude/skills 已包含 6 个 icv-sim-* skill:`,`.claude/skills/` 下出现 `icv-sim-analyze`、`icv-sim-cli`、`icv-sim-cov`、`icv-sim-protocol`、`icv-sim-reg`、`icv-sim-run` 六个子目录。

**三条命令**:

| 命令 | 作用 |
|------|------|
| `bash setup_icv_externals.sh install` | 挂接 6 个 skill(默认)。重复执行有幂等保护,已完整挂接则跳过设置/commit 仅 `svn up` 刷新 |
| `bash setup_icv_externals.sh uninstall` | 卸载 6 个 skill(只删 icv-sim 行,保留其它 externals) |
| `bash setup_icv_externals.sh update` | 先 uninstall 再 install,重新挂接到最新(两次 commit)。NEED 清单变更或想彻底重置时用 |

**部署到用户 home**(`~/.claude/skills`,让 icv-sim-* 在所有项目共享可用)——加 `--home` 标志即可,可与上述三条命令任意顺序组合:

```bash
bash setup_icv_externals.sh install --home     # 部署到 ~/.claude/skills(svn sparse checkout)
bash setup_icv_externals.sh update --home      # 刷新到最新(svn up)
bash setup_icv_externals.sh uninstall --home   # 仅删 6 个 icv-sim,不动其它 skill
```

> home 模式目标 `~/.claude` 不是 svn 工作副本,脚本自动改用 **svn sparse checkout** 拉取(不挂 externals、不 commit),需本地 svn 可用、不回退集群。详见 §2.5、§4.9。

**后续 skill 更新**(`bt/uvn` 那边的 skill 内容更新提交后,无需重新部署):

```bash
cd <你的验证环境根目录>/.claude/skills
svn up .                          # 自动同步 externals 指向的最新 skill
```

> ⚠️ **关键约束**:externals 挂接的 6 个目录是 `bt/uvn` 的「软链接」,**禁止在当前模块内修改并提交**(会回传影响所有模块)。需通用改动请去 `bt/uvn` 改;需自行迭代请复制副本到非 externals 路径(如 `.claude/skills_local/`)另行维护。

> 详细原理、六个 skill 的作用、部署细节与故障排查见下文。

## 1. 概述

`bt/uvn` 模型下统一维护了一套 IC 验证调试用的 Claude Code skill(`icv-sim-*`),覆盖 tw CLI 工具层、编译仿真、仿真分析、覆盖率查询、协议级事务分析、寄存器 metadata 与 CIF 联动。为避免在每个模型(pa、ped、ipro 等模块)里重复维护一份,使用 Subversion 的 **`svn:externals`** 机制,把这些 skill 以「外部引用」的方式挂接到各模型自己的 `.claude/skills` 目录。

效果:在 `verification/<bt|it|st>/<你的模块>/.claude/skills` 下执行一次属性设置 + `svn up`,该目录就会多出 6 个 skill 子目录,内容始终指向 `bt/uvn` 仓库版本,后续 `svn up` 即可同步更新,无需手动拷贝。

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

## 2.5 部署目标与机制自适应(externals / sparse checkout)

脚本支持两类部署目标,按目标是否 svn 工作副本**自动选择机制**(`detect_deploy_mode`):

| 目标 | 是否 svn 工作副本 | 部署机制 | 命令 |
|------|------------------|----------|------|
| 仿真模型根(如 `verification/bt/pa`) | 是 | **externals**:挂 `svn:externals` + `svn up` + 自动 `commit`,内容始终指向 `bt/uvn` 远端 | `bash setup_icv_externals.sh install`(默认) |
| 用户 home(`~/.claude/skills`) | 否(`E155007`) | **sparse checkout**:`svn checkout --depth=empty` 建骨架 + 逐个 `--set-depth=infinity` 拉满 6 个 icv-sim,单次 `svn up` 刷新 | `bash setup_icv_externals.sh install --home` |

**`--home` 标志**:把部署目标从「脚本所在模型目录」切到固定的 `~/.claude/skills`。因 `~/.claude` 不是 svn 工作副本,自动走 checkout 模式。`--home` 可与 install/uninstall/update 任意顺序组合,不带则默认工作目录(externals)模式。

**为什么 home 不能用 externals**:`svn:externals` 是目录属性,要求目标目录处于 svn 工作副本内(才能 `svn propset` + `svn commit`)。`~/.claude` 不在任何 svn 仓库里,这些命令会失败。故 home 改用 svn checkout 直接拉一份工作副本到本地(不挂属性、不 commit)。

**home 模式不回退集群**:集群节点的 `$HOME` 属不同用户,且 `~/.claude` 是用户私有目录,集群回退无意义。home 模式需本地 svn 可用,不可用时直接报错(不 qsub)。

> 工作目录模式下,若脚本所在目录恰好也不是 svn 工作副本(罕见),脚本同样会自动走 checkout 模式——机制自适应不只服务于 home。

## 3. 六个 icv 开头 skill 的作用与应用场景

六个 skill 均来自 `bt/uvn/.claude/skills/`,按「工具层 + 业务层」分工:`icv-sim-cli` 是工具底座(安装/总览),另五个是业务层,业务层均以前置「tw 已装」为条件。

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

### 3.5 icv-sim-protocol —— 协议级事务分析(自研接口)

- **作用**:自研接口(CIF/DIF/F)的协议级事务分析 + 项目专属协议 YAML 构建。通用方法论在本 skill(两步流程 / YAML 构建 / 选观测点 / 联动),具体协议细节按协议分 reference(`references/protocols/<proto>.md`,加新协议 = 加一个 ref)。
- **应用场景**:
  - DIF/F 接口传了什么数据(`tw protocol extract`,VLD 原子层;F 双 FIFO 多通道 `--channel`);
  - DIF/CIF/F 握手异常、吞吐、outstanding(`tw protocol analyze`,套传输模型算 metrics + 异常);
  - DIF/F 接口切事务导出每包数据(`tw protocol transactions`,F 切包 + info 位域解析);
  - 波形里有哪些 DIF/CIF/F 接口实例、接口是否完整(`tw protocol discover`,自动聚类发现 + 三态校验);
  - 新接口/新项目如何从 RTL 构建协议 YAML(见 SKILL「构建项目专属 YAML」5 步)。
- **前置**:tw 已装(见 icv-sim-cli);波形已带 fsdb(见 icv-sim-analyze);`tw kdb` 需 KDB(`tw make cmp` 产出,见 icv-sim-run)。

### 3.6 icv-sim-reg —— 寄存器 metadata + CIF 联动

- **作用**:寄存器视角的验证辅助——从 Excel 寄存器手册提取结构化 metadata(含字段功能描述),再把仿真中 CIF 配置总线访问反查到具体寄存器域段与功能含义,让「CIF 读写了一个地址」变成「CIF 读写 uvn_int_mask 的 cif_err 位」。
- **应用场景**:
  - 从 Excel 寄存器手册提取寄存器/字段/功能描述存到 icvcache(`tw reg extract`);
  - CIF 访问 addr 反查寄存器/字段/含义(`tw reg lookup`,含批量 CIF 事务联动);
  - 列模块/寄存器概览(`tw reg list`);
  - 多维度筛选寄存器/字段(`tw reg filter`)。
  - 覆盖 BT/IT/TOP 三环境地址口径(模块内 offset vs 全局 baseaddr+offset)。
- **定位**:不负责编译/仿真/波形/覆盖率——跑仿真用 icv-sim-run,分析波形/协议事务用 icv-sim-analyze,查覆盖率用 icv-sim-cov。
- **前置**:tw 已装(见 icv-sim-cli)。

### 3.7 六者关系速记

| skill | 层次 | 一句话 | 前置 |
|-------|------|--------|------|
| icv-sim-cli | 工具层 | 装 tw、查环境、看总览 | 无 |
| icv-sim-run | 业务层 | 编译/跑仿真/管作业 | tw 已装 |
| icv-sim-analyze | 业务层 | 看日志/波形/trace 调试 | tw 已装 + KDB/FSDB |
| icv-sim-cov | 业务层 | 查覆盖率 VDB | tw 已装 + VDB |
| icv-sim-protocol | 业务层 | 自研接口协议级事务分析/YAML | tw 已装 + fsdb + KDB |
| icv-sim-reg | 业务层 | 寄存器 metadata + CIF 联动 | tw 已装 |

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
3. **幂等检测**:读取 `.claude/skills` 现有 `svn:externals`,拆出 icv-sim 行与其它(OTHER)行。
   - 若 icv-sim 行集合已等于 6 个 NEED(不多不少,可能还含 OTHER)→ **已完整挂接**:跳过后续 propset/commit,直接 `svn up` 刷新内容 + 校验后结束(幂等保护,避免空 commit 误报 E155010)。
   - 否则(缺失/含多余 icv-sim 条目/为空)→ 进入步骤 4 重新设置。
4. **生成 externals 列表文件**:在 `.claude/skills/` 下生成 `icv-externals-list.txt`,内容**由脚本按 NEED 数组生成** 6 行 externals 条目(单一来源,新增/移除 skill 只改脚本 NEED);若存在 OTHER 非 icv-sim externals,原样追加保留(防覆盖式 propset 误删)。
5. **设置属性**:在 `.claude/skills` 下执行
   ```bash
   svn propset svn:externals -F icv-externals-list.txt .
   ```
6. **范围检查与自动 commit**:脚本只提交 icv-sim-* 相关变更:
   - `.claude` 目录本身(首次为空目录节点);
   - `.claude/skills` 目录本身(携带 `svn:externals` 属性);
   - `.claude/skills/icv-externals-list.txt`。

   `.claude` 及 `.claude/skills` 下其它任意名称的未提交/无关项一律忽略,不报错也不提交。commit 前还会兜底检查这三个 path 是否真有待提交变更,无变更则跳过 commit(防 E155010)。
7. **拉取外部目录**:在 `.claude/skills` 下执行 `svn up .`,检出 6 个 skill 子目录。注意:必须先 commit 父目录(步骤 6),否则 `.claude/skills` 尚无版本化父目录,`svn up` 会跳过 external。
8. **成功校验**:检查 `.claude/skills` 下是否包含 6 个 `icv-sim-*` 目录(analyze/cli/cov/protocol/reg/run),齐了即成功。

### 4.4 一键执行

```bash
cd verification/<bt|it|st>/<你的模块>     # 例如 cd verification/bt/pa
bash setup_icv_externals.sh
```

### 4.5 成功标志

脚本末尾输出 `install 成功。/path/to/.claude/skills 已包含 6 个 icv-sim-* skill:`,且 `.claude/skills` 目录下出现 6 个子目录:

```
.claude/skills/
├── icv-sim-analyze/
├── icv-sim-cli/
├── icv-sim-cov/
├── icv-sim-protocol/
├── icv-sim-reg/
└── icv-sim-run/
```

### 4.6 后续步骤

**后续 skills 更新同步(类似软链接):** `svn:externals` 的 6 个 skill 目录始终指向 `bt/uvn` 仓库版本,相当于对远端的「软链接」。`bt/uvn` 那边的 skill 内容更新并提交后,**你无需重新部署、也无需手动拷贝**——只需在自己的验证环境执行一次 `svn up`, externals 挂接的 6 个目录就会自动同步到最新版本:

```bash
cd verification/<bt|it|st>/<你的模块>/.claude/skills
svn up .                                  # 拉取 externals 指向的最新 skill 内容
```

> 同理,团队其他成员首次 `svn up` 该目录时会自动挂接 6 个 skill;之后每次 `svn up` 即同步更新,无需各自跑部署脚本。

### 4.7 update 命令(重新挂接到最新)

```bash
bash setup_icv_externals.sh update
```

**语义**:`update` = 先 `uninstall` 再 `install`,串行执行,各自独立 commit(共两次 commit)。它会先把现有 icv-sim-* externals **全部拆除**(只删 icv-sim 行,保留其它非 icv-sim externals),再按最新 NEED 清单重新挂接 6 个 skill。

**适用场景**:
- **NEED 清单变更后重新对齐**:如本次 4 个 → 6 个(新增 protocol/reg),旧挂接缺新 skill,`install` 的幂等检测发现「不完整」会自动补全;但若想彻底清掉旧残留重建,用 `update` 更干净。
- **externals 指向需彻底重置**:`svn up` 只刷新内容不动属性,`update` 连属性一起重建。
- **想清掉旧残留重建**:怀疑挂接状态异常时,`update` 一步到位。

**中间状态风险**:`update` 在 `uninstall` 完成、`install` 尚未开始的窗口期,工作副本处于「icv-sim 已卸载」状态。若此时脚本中途失败(如网络中断),会停在已卸载未重装的中间态——**重跑一次 `update`(或直接 `install`)即可恢复**,`install` 对「已卸载」状态会正常重新设置。两次 commit 均有范围检查与 commit 兜底,不会误伤无关项。

### 4.8 重复 install 的幂等保护(可安全重复执行)

`install` 可重复执行,不会因「已装好」而报错失败:

- **已完整挂接**(icv-sim 行 == 6 个 NEED):脚本检测到后**跳过 propset 与 commit**,仅执行一次 `svn up` 刷新 externals 内容,输出 `install 幂等完成(无需变更)` 后正常结束。不会产生空 commit,也不会因 `svn commit` 无变更报 `E155010` 而误报失败。
- **挂接不完整**(如旧的 4 个安装,缺 protocol/reg):脚本检测到 icv-sim 行集合 ≠ NEED,会重新 `propset` 补全到 6 个并 commit,自动完成升级。
- **非 icv-sim externals 会被保留**:若 `.claude/skills` 的 `svn:externals` 里还有用户自加的其它 external 条目,`install`/`update`/`uninstall` 都**只动 icv-sim 行**,其它条目原样保留,不会被覆盖式 propset 误删。

> 结论:重复 `install` 是安全的,无需手动先卸载。要彻底重置才用 `update`。

### 4.9 home 模式部署细节(sparse checkout)

`--home` 把目标切到 `~/.claude/skills`,因该目录非 svn 工作副本,脚本自动用 **svn sparse checkout** 拉取(不挂 externals、不 commit):

**install(`install --home`)内部步骤**:
1. 确保 `~/.claude/skills` 存在(不存在则 `mkdir -p`,home 模式跳过 sim/Makefile 检查)。
2. 若 `~/.claude/skills` 还不是 svn 工作副本:`svn checkout --depth=empty <bt/uvn/.claude/skills>` 建**稀疏骨架**——目录只多出 `.svn`,不拉任何子目录。**到非空目录也安全**:用户已有的其它 skill(如 codebase-memory)原样保留。
3. 逐个对 6 个 `icv-sim-*` 执行 `svn update --set-depth=infinity <skill>`,把该 skill 拉满。
4. 校验 6 个 `SKILL.md` 到位。

**幂等与已存在保护**:
- 子目录已是 svn 检出(之前拉满过)→ `svn up` 刷新,不重复 checkout。
- **子目录已存在但非 svn 检出**(疑似用户手动放置的同名目录)→ **报错中止**,不覆盖。因盲目 `--set-depth=infinity` 到非 WC 目录会产生 tree conflict 且拉不到内容。请手动 `rm -rf ~/.claude/skills/<skill>` 后重跑。
- 重复 `install --home`:6 个已是 WC → 全部 `svn up` 刷新,不报错。

**update(`update --home`)**:直接 `svn up ~/.claude/skills` 一次性刷新所有 icv-sim。**裸 `svn up` 严格保持各目录 depth**——skills 保持 `empty`(不拉满 uvn-wiki 等非目标目录),6 个 icv-sim 保持 `infinity` 刷新。故单次 up 即可,不会污染出多余目录。

**uninstall(`uninstall --home`)**:**仅删 6 个 `icv-sim-*` 目录**,不动用户其它 skill(codebase-memory 等)。**保留 `~/.claude/skills/.svn`**(sparse WC 骨架)——避免破坏同目录下其它 skill 的工作副本归属。如需彻底移除 WC 元数据,手动 `rm -rf ~/.claude/skills/.svn`。

**`.svn` 元数据说明**:`install --home` 后 `~/.claude/skills` 成为 svn 工作副本(指向 `bt/uvn` 远端 skills),目录下会多出 `.svn`。用户原有的其它 skill(codebase-memory、frontend-design 等)变为该 WC 内的「未版本控制文件」,`svn status` 会显示 `?`,但**不影响正常使用**。注意:与 externals 模式不同,checkout 模式下这些 icv-sim 目录是**本地独立工作副本**,在 home 内修改**不会回传** `bt/uvn`(无 commit 路径);但仍建议不要直接改,保持与远端一致便于 `svn up`。

## 5. 注意事项与故障排查

- **externals 是只读引用,禁止在当前模块内修改提交**:6 个 `icv-sim-*` 目录由 `bt/uvn` 统一维护,通过 `svn:externals` 挂接进来后相当于远端的「软链接」(详见 §4.6)。**不要在当前模块内对这些目录做修改并提交**——externals 目录的修改会回传到 `bt/uvn` 源仓库,影响所有挂接该 skill 的模块,易造成他人环境异常。
  - **需要通用改动时**:请到 `bt/uvn/.claude/skills/` 修改并提交,各模块 `svn up` 即可同步。
  - **有自行迭代需求时(不想影响他人、也不想被远端更新覆盖)**:不要动 externals 挂接的目录,而是**复制一份源文件到自己的工作目录另行维护,做好版本隔离**。例如复制到 `.claude/skills_local/`(或其它非 externals 路径)并重命名,改用本地副本;同时从 `icv-externals-list.txt` 中移除对应条目,避免本地副本与 externals 引用并存冲突。这样你的定制版本独立演进,既不会回传影响 `bt/uvn`,也不会在 `svn up` 时被远端更新覆盖。
- **`.claude` 与 `.claude/skills` 的创建与 svn 跟踪**:脚本会自动处理——目录不存在时逐级 `mkdir -p` 创建,并只把 `.claude` 和 `.claude/skills` 这两个目录本身(`svn add --depth empty`)纳入版本控制,**不会递归添加目录内部的其它文件/目录**(如 `settings.json`、`.claude/worktrees/`、`.claude/tmp/`、`.claude/skills/xxx_demo_skills` 等)。目录已存在但未被跟踪时,同样只 add 空目录节点,内部已有内容保持未跟踪。前提是:
  - **脚本所在模型目录本身必须是 svn 工作副本**(脚本启动时会 `svn info` 校验);
  - `.claude` 没有被父目录的 `svn:ignore` 排除。若 `.claude` 被 ignore,`svn add` 会失败,脚本会提示先解除 ignore。

  若模型目录不是 svn 工作副本,脚本会直接报错退出;若 `.claude` 被 `svn:ignore` 排除,需先在模型目录解除忽略(例如 `svn propdel svn:ignore .` 或从 ignore 列表中移除 `.claude`)后再运行本脚本。
- **`.claude` 及 `.claude/skills` 下其它未提交项**:脚本只提交 icv-sim-* 相关的目录节点、属性与 `icv-externals-list.txt`,其它任意名称的未跟踪或改动项(如 `ptype-fanout`、`xxx_demo_skills`、`xxx_yyy`)会被忽略,不会报错也不会被提交。如果你希望它们不再出现在 `svn status` 里,可手动加入 `svn:ignore`。
- **首次安装的 commit 顺序**:首次安装时 `.claude` 与 `.claude/skills` 可能还是新增(`A`)状态,脚本会先 commit 这些空目录节点与 `svn:externals` 属性,然后再执行 `svn up` 拉取 6 个 `icv-sim-*` 外部目录。如果顺序颠倒,`svn up` 会因“无版本化父目录”而跳过 external。
- **临时文件**:本地执行时脚本可能在模型根目录生成 `.setup_icv_*` 开头的临时文件(回退 qsub 时还会生成 worker 脚本和 out 日志)。成功执行后这些临时文件会被自动清理;失败时会保留以便排查,已被 `.gitignore` 忽略,不会误入 git 提交。
- **重复 install 安全(幂等)**:`install` 可重复执行。已完整挂接 6 个 skill 时,脚本跳过 `propset`/`commit` 仅 `svn up` 刷新内容,不会因 `svn commit` 无变更报 `E155010` 而失败;挂接不完整时自动补全到 6 个。详见 §4.8。`update` 适合需要彻底重置的场景(先卸载再重装,两次 commit)。
- **home 模式(`--home`)注意**:
  - `~/.claude/skills` 经 `install --home` 后成为 svn 工作副本(多出 `.svn`),用户原有其它 skill 变为 WC 内未版本控制文件(`svn status` 显示 `?`),不影响使用(详见 §4.9)。
  - home 模式下 icv-sim 是**本地独立工作副本**,在 home 内修改不会回传 `bt/uvn`(无 commit 路径),但仍建议保持与远端一致以便 `svn up`。
  - `uninstall --home` 仅删 6 个 icv-sim,保留 `.svn` 与其它 skill;彻底清 WC 元数据需手动删 `.svn`。
  - home 模式需本地 svn 可用,**不回退集群**(集群节点 HOME 属不同用户)。

## 6. 脚本内容

脚本已纳入模型仓库,位于验证环境根目录,直接查看或执行即可:

```
verification/<bt|it|st>/<你的模块>/setup_icv_externals.sh
```

脚本逻辑概要(详见 §4.3 执行步骤):预检查运行环境 → 确保 `.claude` 与 `.claude/skills` 被 svn 跟踪(必要时 `--depth empty` add 空目录节点)→ 幂等检测现有 externals(已完整则仅 `svn up` 刷新)→ 在 `.claude/skills` 生成 `icv-externals-list.txt`(脚本按 NEED 生成 6 行 externals,保留非 icv-sim 行)→ `svn propset svn:externals -F` → 范围检查并 commit(只提交 icv-sim 相关项:`.claude`、`.claude/skills`、`icv-externals-list.txt`,无变更则跳过)→ `svn up` 拉取 external → 校验 6 个 `icv-sim-*` 目录到位。
