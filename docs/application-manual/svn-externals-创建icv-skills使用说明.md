# svn:externals 创建icv-skills使用说明 —— 引入 icv-sim 系列 skill

## 省流版本(TL;DR)

在 `<你的模块>` 验证环境根目录(如 `verification/bt/pa`)下两步完成部署:

```bash
# 1) 从共享目录复制部署脚本并加可执行权限
cp /home/ldap/alvin.xu/ForShare/setup_icv_externals.sh <你的验证环境根目录>/
chmod +x <你的验证环境根目录>/setup_icv_externals.sh

# 2) 进入模块目录执行脚本(自动完成 externals 属性设置 + svn up,检出 4 个 icv-sim-* skill)
cd <你的验证环境根目录>          # 例如 verification/bt/pa
bash setup_icv_externals.sh
```

**成功标志**:脚本末尾输出 `部署成功`,`.claude/skills/` 下出现 `icv-sim-analyze`、`icv-sim-cli`、`icv-sim-cov`、`icv-sim-run` 四个子目录。

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

### 脚本获取方式

部署脚本 `setup_icv_externals.sh` 统一存放在共享目录,需要时复制到自己的工作目录(验证环境根目录)即可,不涉及仓库内分发:

| 位置 | 用途 |
|------|------|
| `/home/ldap/alvin.xu/ForShare/setup_icv_externals.sh` | 统一取用入口,复制到各模型工作目录后执行 |

**ForShare 共享目录权限说明(已确认他人可复制执行):**

- `/home/ldap/alvin.xu`:`drwxrwxrwx`(777),其他用户可进入并遍历;
- `/home/ldap/alvin.xu/ForShare`:`drwxr-xr-x`(755),其他用户可进入、可读取目录下文件;
- `setup_icv_externals.sh`:`-rwxr-xr-x`(755),其他用户可读、可执行、可 `cp` 复制。

获取与执行:

```bash
# 1) 从 ForShare 复制脚本到你的工作目录(验证环境根目录,即 <你的模块> 目录)
cp /home/ldap/alvin.xu/ForShare/setup_icv_externals.sh <你的验证环境根目录>/
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

## 3. 部署方式(bash 一键部署)

### 3.1 脚本放置位置

从 ForShare 复制来的 `setup_icv_externals.sh` 放在**验证环境根目录**(模型目录),即 `<你的模块>` 目录下。以 pa 模型为例,复制后位于:

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

### 3.3 执行步骤(脚本内部按顺序完成)

1. **预检查与目录准备**:目录层级、`svn` 命令可用、模型目录是 svn 工作副本。若 `.claude/skills` 不存在,脚本会逐级创建(`mkdir -p .claude/skills`)并通过 `svn add` 纳入版本控制(尚未提交);若已存在,则校验其已被 svn 跟踪。
2. **生成 externals 列表文件**:在 `.claude/skills/` 下生成 `icv-externals-list.txt`,内容优先取自脚本同目录的 `icv验证调试导入说明.md`(已写好 4 行 externals 条目);该文件缺失时回退到脚本内嵌默认值。
3. **设置属性**:在 `.claude/skills` 下执行
   ```bash
   svn propset svn:externals -F icv-externals-list.txt .
   ```
4. **拉取外部目录**:在 `.claude/skills` 下执行 `svn up .`,检出 4 个 skill 子目录。
5. **成功校验**:检查 `.claude/skills` 下是否包含 `icv-sim-analyze`、`icv-sim-cli`、`icv-sim-cov`、`icv-sim-run`,齐了即成功。

### 3.4 一键执行

```bash
cd verification/<bt|it|st>/<你的模块>     # 例如 cd verification/bt/pa
bash setup_icv_externals.sh
```

### 3.5 成功标志

脚本末尾输出 `部署成功`,且 `.claude/skills` 目录下出现 4 个子目录:

```
.claude/skills/
├── icv-sim-analyze/
├── icv-sim-cli/
├── icv-sim-cov/
└── icv-sim-run/
```

### 3.6 后续步骤

**后续 skills 更新同步(类似软链接):** `svn:externals` 的 4 个 skill 目录始终指向 `bt/uvn` 仓库版本,相当于对远端的「软链接」。`bt/uvn` 那边的 skill 内容更新并提交后,**你无需重新部署、也无需手动拷贝**——只需在自己的验证环境执行一次 `svn up`, externals 挂接的 4 个目录就会自动同步到最新版本:

```bash
cd verification/<bt|it|st>/<你的模块>/.claude/skills
svn up .                                  # 拉取 externals 指向的最新 skill 内容
```

> 同理,团队其他成员首次 `svn up` 该目录时会自动挂接 4 个 skill;之后每次 `svn up` 即同步更新,无需各自跑部署脚本。

## 4. 注意事项与故障排查

- **externals 是只读引用,禁止在当前模块内修改提交**:4 个 `icv-sim-*` 目录由 `bt/uvn` 统一维护,通过 `svn:externals` 挂接进来后相当于远端的「软链接」(详见 §3.6)。**不要在当前模块内对这些目录做修改并提交**——externals 目录的修改会回传到 `bt/uvn` 源仓库,影响所有挂接该 skill 的模块,易造成他人环境异常。
  - **需要通用改动时**:请到 `bt/uvn/.claude/skills/` 修改并提交,各模块 `svn up` 即可同步。
  - **有自行迭代需求时(不想影响他人、也不想被远端更新覆盖)**:不要动 externals 挂接的目录,而是**复制一份源文件到自己的工作目录另行维护,做好版本隔离**。例如复制到 `.claude/skills_local/`(或其它非 externals 路径)并重命名,改用本地副本;同时从 `icv-externals-list.txt` 中移除对应条目,避免本地副本与 externals 引用并存冲突。这样你的定制版本独立演进,既不会回传影响 `bt/uvn`,也不会在 `svn up` 时被远端更新覆盖。
- **`.claude/skills` 的创建与 svn 跟踪**:脚本会自动处理——目录不存在时逐级 `mkdir -p` 创建并 `svn add` 纳入版本控制;目录已存在时校验其已被 svn 跟踪。前提是**脚本所在模型目录本身必须是 svn 工作副本**(脚本启动时会 `svn info` 校验)。若模型目录不是 svn 工作副本,脚本会直接退出并提示;若该目录已被 `svn:ignore` 排除,需先解除忽略或换到未被忽略的位置。


## 5. 脚本内容

脚本统一存放在共享目录,不随仓库分发,需要时直接查看或复制:

```
/home/ldap/alvin.xu/ForShare/setup_icv_externals.sh
```

查看脚本内容:

```bash
cat /home/ldap/alvin.xu/ForShare/setup_icv_externals.sh
less /home/ldap/alvin.xu/ForShare/setup_icv_externals.sh    # 翻页查看
```

脚本逻辑概要(详见 §3.3 执行步骤):预检查目录层级与 svn 环境 → 在 `.claude/skills` 生成 `icv-externals-list.txt`(内嵌默认 4 行 externals)→ `svn propset svn:externals -F` → `svn up` → 校验 4 个 `icv-sim-*` 目录到位。

