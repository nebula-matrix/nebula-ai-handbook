# Claude Code开发环境配置指南

> 本手册指导开发者在虚拟机中配置Claude Code开发环境，涵盖从环境准备到测试验证的完整流程。

---

## 目录

1. [前置条件](#前置条件)
2. [环境准备](#环境准备)
   - [切换apt源为阿里云](#切换apt源为阿里云)
   - [安装Node.js 24](#安装nodejs-24)
   - [安装uv（Python包管理器）](#安装uvpython包管理器)
   - [配置Git SSH密钥](#配置git-ssh密钥)
   - [挂载NFS共享目录](#挂载nfs共享目录)
3. [安装Claude Code](#安装claude-code)
4. [配置API](#配置api)
   - [获取公司分配的API凭证](#获取公司分配的api凭证)
   - [模型选择说明](#模型选择说明)
   - [配置环境变量](#配置环境变量)
   - [settings.json样本实例](#settingsjson-样本实例)
   - [后续如何修改API配置](#后续如何修改api配置)
5. [使用Claude Code](#使用claude-code)
   - [命令行调用](#命令行调用)
   - [VSCode插件调用](#vscode插件调用)
6. [测试验证](#测试验证)
   - [功能测试](#功能测试)
7. [补充说明](#补充说明)
8. [相关资源](#相关资源)

---

**文档版本**：v1.0
**最后更新**：2026年4月13日
**维护者**：星云智联技术团队

## 前置条件

在开始本指南之前，您需要：

### 已完成虚拟机创建和连接

- ✅ 已按照[星云智联配置VM基础说明](../vm-configuration/星云智联配置VM基础说明.md)完成虚拟机创建
- ✅ 已通过VSCode Remote-SSH成功连接到虚拟机

### 基础知识要求

- 熟悉基本Linux命令操作：`cd`、`ls`、`sudo`、`cat`、`nano` 等
- 了解文件权限概念

### 系统访问权限

- 具有sudo权限的系统账户
- 可访问外网（用于下载安装包）

---

## 环境准备

### 切换apt源为阿里云

⚠️ **重要提示**：不同的Linux发行版使用不同的包管理器和配置方式。以下以 **Ubuntu 24.04 LTS** 为例。

#### 步骤1：检测Linux发行版

```bash
# 检查Linux发行版信息
lsb_release -a
# 或
cat /etc/os-release
```

#### 步骤2：备份原配置

```bash
# 备份和清理原配置文件
sudo mkdir /etc/apt/backup_config
sudo mv /etc/apt/sources.list /etc/apt/backup_config/
sudo mv /etc/apt/sources.list.d/ubuntu.sources /etc/apt/backup_config/
```

#### 步骤3：修改为阿里云源

```bash
# 编辑源配置文件
sudo nano /etc/apt/sources.list.d/ubuntu.sources
```

将配置修改为（仅对ubuntu24有效）：

```yaml
Types: deb
URIs: http://mirrors.aliyun.com/ubuntu/
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

Types: deb
URIs: http://mirrors.aliyun.com/ubuntu/
Suites: noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

保存并退出（`Ctrl+X`，然后 `Y`，再 `Enter`）

#### 步骤4：更新软件包列表

```bash
sudo apt update
```

预期输出：应该看到 "Hit" 和 "Get" 消息，表示正在从阿里云镜像获取包列表

#### 步骤5：解决Docker源网络问题

运行 `sudo apt update` 时，可能会遇到以下错误：

```
Err:9 https://download.docker.com/linux/ubuntu noble InRelease
  Could not handshake: Error in the pull function. [IP: 108.139.10.78 443]

W: Failed to fetch https://download.docker.com/linux/ubuntu/dists/noble/InRelease
  Could not handshake: Error in the pull function. [IP: 108.139.10.78 443]
W: Some index files failed to download. They have been ignored, or old ones used instead.
```

**原因**：Docker官方源可能因网络问题不可达，导致更新失败。

**解决方案**：禁用Docker源，让apt忽略它。

```bash
# 1. 查找 Docker 源文件的准确名称
ls /etc/apt/sources.list.d/ | grep docker

例如终端给出
docker.sources

# 2. 将其重命名（禁用它）
sudo mv /etc/apt/sources.list.d/docker.sources /etc/apt/sources.list.d/docker.sources.disable

# 3. 重新运行更新
sudo apt update

# 4. 运行结果，可以看到docker源已被忽略
Hit:1 http://mirrors.aliyun.com/ubuntu noble InRelease
Hit:2 http://mirrors.aliyun.com/ubuntu noble-updates InRelease
Hit:3 http://mirrors.aliyun.com/ubuntu noble-backports InRelease
Hit:4 http://mirrors.aliyun.com/ubuntu noble-security InRelease
Hit:5 https://deb.nodesource.com/node_24.x nodistro InRelease
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
33 packages can be upgraded. Run 'apt list --upgradable' to see them.
N: Ignoring file 'docker.sources.disable' in directory '/etc/apt/sources.list.d/' as it has an invalid filename extension
```

#### 步骤6：解决重启后文件被清空的问题 (cloud-init 拦截)

服务器如果运行 cloud-init 服务。默认情况下，它会在系统启动时读取云厂商的元数据（Metadata），并根据默认模板重新生成 ubuntu.sources，从而覆盖你的手动修改。

##### 解决方案：通过 Cloud-init 配置文件规范拦截

我们需要告诉 cloud-init 的 apt 模块：“保留系统现有的源列表，不要去动它”。

```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-apt-overwrite.cfg
```

写入以下 YAML 内容（注意缩进，YAML对空格敏感）：

```YAML
apt:
  preserve_sources_list: true
```

保存并退出。下次重启时，cloud-init 看到 preserve_sources_list: true，就会放弃对 APT 源的覆写。

#### 其他发行版参考

| 发行版 | 镜像站地址 |
|--------|-----------|
| Debian | [阿里云Debian镜像](https://developer.aliyun.com/mirror/debian) |
| CentOS/Rocky Linux | [阿里云CentOS镜像](https://developer.aliyun.com/mirror/centos) |
| 其他发行版 | [阿里云镜像站](https://developer.aliyun.com/mirror/) |

---

### 安装Node.js 24

Claude Code需要Node.js 18或更高版本，本指南使用Node.js 24。

#### 卸载旧版本（如有）

如果之前通过apt安装过Node.js，建议先卸载：

```bash
sudo apt-get remove nodejs npm
sudo apt-get autoremove
```

#### 为什么不推荐通过apt安装Node.js？

通过apt安装Node.js存在以下隐患：

1. **版本过旧**：apt源中的Node.js版本通常较旧，可能不满足Claude Code的最低版本要求
2. **权限问题**：apt安装的Node.js位于系统目录（如`/usr/bin`），后续使用npm全局安装包时需要sudo权限
3. **官方不推荐**：Claude Code官方文档明确不推荐使用`sudo npm install -g`进行全局安装，因为这可能导致权限混乱和安全隐患

因此，本指南采用nvm（Node Version Manager）进行安装，这是Node.js官方推荐的安装方式，可以将Node.js安装在用户目录下，无需sudo权限。

⚠️ **重要提示**：由于受国内网络影响，nvm和Node.js的下载可能较慢或失败。需要先设置代理服务器进行加速下载。

#### 步骤1：设置终端代理
（临时方案，后续支持透明代理后，此部分再做更新说明）

```bash
export http_proxy="http://10.10.14.15:9999"
export https_proxy="http://10.10.14.15:9999"
export HTTP_PROXY="http://10.10.14.15:9999"
export HTTPS_PROXY="http://10.10.14.15:9999"
```

💡 **提示**：以上代理设置为临时生效，仅对当前终端会话有效。

#### 步骤2：下载并安装nvm

以[nodejs官网说明](https://nodejs.org/en/download)为准，以下是2026.4.14的官网说明，供参考：

```bash
# 下载并安装 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash

# 加载nvm（无需重启终端）
\. "$HOME/.nvm/nvm.sh"
```

#### 步骤3：使用nvm安装Node.js

```bash
# 下载并安装 Node.js 24
nvm install 24
```

#### 步骤4：验证安装

```bash
# 验证 Node.js 版本
node -v
# 应输出: v24.14.1

# 验证 npm 版本
npm -v
# 应输出: 11.11.0

# 确认安装位置（应在home目录，而非全局根目录）
which npm
# 应输出: /home/你的用户名/.nvm/versions/node/v24.14.1/bin/npm
```

#### 步骤5：取消代理设置

安装完成后，建议取消代理环境变量，避免代理服务器持续生效影响其他网络请求。

```bash
unset http_proxy
unset https_proxy
unset HTTP_PROXY
unset HTTPS_PROXY
```

💡 **提示**：关闭当前终端窗口重新打开，代理设置也会自动失效。

---

### 安装uv（Python包管理器）

uv是一个现代化的Python包管理器，速度极快，可以替代pip和pip-tools。

本节内容以[uv官网](https://docs.astral.sh/uv/getting-started/installation/)为准，以下是基于2026年4月16日的官网说明，供参考。

#### 步骤1：安装uv

在Linux系统上，可以使用以下命令安装：
（如果速度过慢，可以参考nodejs章节采用的代理安装方法，注意安装完毕后取消代理）

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

安装完成后，需要重新加载shell配置或重新打开终端：

```bash
# 重新加载shell配置（以bash为例）
source ~/.bashrc
```

#### 步骤2：验证安装

```bash
# 验证uv版本
uv --version
```

#### 步骤3：配置Shell自动补全（bash）

如果使用bash，执行以下命令配置自动补全：

```bash
# 生成补全脚本并添加到.bashrc
echo 'eval "$(uv generate-shell-completion bash)"' >> ~/.bashrc

# 重新加载配置
source ~/.bashrc
```

#### 自动补全使用方法

配置完成后，在输入uv命令时可以：

- 输入命令部分字符后按 `Tab` 键，自动补全命令或显示可选选项
- 例如：
  ```bash
  uv pip in<Tab>    # 自动补全为 uv pip install
  uv <Tab><Tab>     # 显示所有可用子命令
  ```

💡 **提示**：自动补全功能需要重新打开终端或执行 `source ~/.bashrc` 后生效。

---

### 配置Git SSH密钥

配置SSH密钥后，可以使用SSH方式拉取和推送代码，无需每次输入密码。

#### 步骤1：检查现有SSH密钥

```bash
# 检查是否已有SSH密钥
ls -la ~/.ssh
```

如果看到 `id_rsa` 和 `id_rsa.pub`（或 `id_ed25519` 和 `id_ed25519.pub`），说明已有密钥，可以直接跳到[步骤3](#步骤3添加ssh密钥到git平台)。

#### 步骤2：生成新的SSH密钥

```bash
# 生成ED25519类型密钥（推荐）
ssh-keygen -t ed25519 -C "your_email@example.com"

# 或使用RSA类型（兼容性更好）
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

执行过程：
1. 按Enter接受默认保存位置（`~/.ssh/id_ed25519` 或 `~/.ssh/id_rsa`）
2. 输入密码短语（可选，建议设置以增强安全性）
3. 确认密码短语

⚠️ **注意**：请将 `your_email@example.com` 替换为您的实际公司邮箱地址。

#### 步骤3：添加SSH密钥到Git平台

##### 查看公钥内容

```bash
# ED25519密钥
cat ~/.ssh/id_ed25519.pub

# RSA密钥
cat ~/.ssh/id_rsa.pub
```

复制输出的完整内容（从 `ssh-` 开始到邮箱结束）。

##### 添加到Git平台

**GitHub**：
1. 访问 [GitHub SSH设置](https://github.com/settings/ssh)
2. 点击 "New SSH key"
3. 粘贴公钥内容
4. 点击 "Add SSH key"


#### 步骤4：测试SSH连接

```bash
# 测试GitHub
ssh -T git@github.com
```

首次连接会提示确认指纹，输入 `yes` 即可。

成功示例（GitHub）：
```
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

#### 步骤5：拉取公司仓库

完成SSH密钥配置后，可以拉取星云智联的内部仓库。

##### 拉取nebula-ai-handbook仓库

```bash
# 克隆仓库到本地
git clone git@github.com:nebula-matrix/nebula-ai-handbook.git

# 进入仓库目录
cd nebula-ai-handbook
```

##### 常用Git操作

```bash
# 拉取最新代码
git pull

# 查看当前状态
git status

# 查看分支
git branch -a

# 切换分支
git checkout <branch-name>
```

##### 提交代码流程

```bash
# 1. 查看修改内容
git status
git diff

# 2. 添加修改的文件
git add <file-name>
# 或添加所有修改
git add .

# 3. 提交修改
git commit -m "描述修改内容"

# 4. 推送到远程仓库
git push
```

⚠️ **重要提示**：
- 推送代码前请确保已获得仓库的写入权限
- 提交信息请清晰描述修改内容
- 建议在推送前先执行 `git pull` 确保代码是最新的

---

#### 步骤6：配置Git用户信息

```bash
# 配置全局用户名
git config --global user.name "Your Name"

# 配置全局邮箱
git config --global user.email "your_email@example.com"
```

⚠️ **重要**：请将用户名和邮箱替换为您的实际信息。

---

### 挂载NFS共享目录

红区复制出来的文件会存放在NFS共享目录中，需要手动挂载才能访问。

#### 步骤1：安装NFS客户端工具

```bash
sudo apt install -y nfs-common
```

#### 步骤2：创建挂载目录

```bash
sudo mkdir -p /mnt/share
```

#### 步骤3：挂载服务端的共享目录

```bash
sudo mount -t nfs 10.10.110.10:/share /mnt/share
```

#### 步骤4：验证挂载

```bash
# 查看挂载情况
df -h | grep share

# 或查看挂载目录内容
ls /mnt/share
```

💡 **提示**：以上挂载为临时挂载，系统重启后需要重新挂载。如需开机自动挂载，请将以下内容添加到 `/etc/fstab`：
```
10.10.110.10:/share /mnt/share nfs defaults 0 0
```

---

## 安装Claude Code

### 安装步骤

#### 步骤1：全局安装Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

#### 步骤2：验证安装

```bash
claude --version
```

预期输出：
```
2.1.x.x (Claude Code)
```

⚠️ **重要**：安装完成后，**不要立即运行** `claude` 命令。需要先完成API配置才能正常使用。

### 版本说明

- **最低要求**：2.1.42 或更高
- **推荐使用**：最新稳定版

#### 检查和升级版本

```bash
# 检查当前版本
claude --version

# 升级到最新版本
claude update
```

---

## 配置API

### 配置说明

📖 **配置方法参考**：本章节配置方法参考自[智谱AI开放文档 - Claude Code配置指南](https://docs.bigmodel.cn/cn/guide/develop/claude#claude-code)。

API访问凭证由星云智联技术团队统一分配和管理。

---

### 获取公司分配的API凭证

⚠️ **重要**：Claude Code的API访问凭证由星云智联技术团队统一管理和分配。

**获取步骤**：

1. 联系星云智联技术负责人
2. 申请智谱AI编码套餐访问权限
3. 获取分配的API凭证

🔐 **安全提示**：获取到的API凭证请妥善保管，不要分享给他人或上传到公开仓库。

---

### 模型选择说明

#### 可用模型

| 模型 | 对应模型 | 环境变量 | 用途 | 特点 |
|---------|---------------|---------|------|------|
| **llm-large-claude** | GLM-4.7 | `ANTHROPIC_DEFAULT_OPUS_MODEL`<br>`ANTHROPIC_DEFAULT_SONNET_MODEL`<br>`ANTHROPIC_DEFAULT_HAIKU_MODEL` | 日常开发、复杂任务 | 性能与速度平衡 |
| **llm-ultra-claude** | Kimi-2.5 | `ANTHROPIC_DEFAULT_OPUS_MODEL`<br>`ANTHROPIC_DEFAULT_SONNET_MODEL`<br>`ANTHROPIC_DEFAULT_HAIKU_MODEL` | 复杂推理、深度分析 | 更强的推理能力 |

#### 模型选择建议

- **日常开发**：推荐使用 `llm-large-claude`（对应 GLM-4.7）
- **复杂推理任务**：推荐使用 `llm-ultra-claude`（对应 Kimi-2.5）

---

### 配置环境变量

#### 步骤1：配置 settings.json

```bash
# 编辑或新增配置文件
nano ~/.claude/settings.json
```

添加以下内容（**替换 `your_company_assigned_token` 为您获取的API凭证**）：

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "<密钥>",
    "ANTHROPIC_BASE_URL": "https://gateway.ai.dpu.tech",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "llm-large-claude",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "llm-large-claude",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "llm-large-claude"
  }
}
```

保存并退出（`Ctrl+X`，然后 `Y`，再 `Enter`）

#### 步骤2：配置 .claude.json

```bash
# 编辑或新增配置文件
nano ~/.claude.json
```

添加以下内容：

```json
{
  "hasCompletedOnboarding": true
}
```

保存并退出（`Ctrl+X`，然后 `Y`，再 `Enter`）

#### 步骤3：验证配置

```bash
# 查看配置内容
cat ~/.claude/settings.json
cat ~/.claude.json
```

💡 **提示**：配置完成后，请打开新的终端窗口或重新连接VSCode以确保配置生效。

---

### settings.json 样本实例

以下是一个完整的 `~/.claude/settings.json` 配置示例：

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "<密钥>",
    "ANTHROPIC_BASE_URL": "https://gateway.ai.dpu.tech",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "llm-large-claude",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "llm-large-claude",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "llm-large-claude"
  }
}
```

⚠️ **重要提醒**：
- 上面的 `ANTHROPIC_AUTH_TOKEN` 值仅为示例格式
- 请务必替换为公司分配给您的实际API凭证
- 不要使用示例中的凭证值

#### 配置项详细说明

| 配置项 | 示例值 | 说明 |
|--------|--------|------|
| `ANTHROPIC_AUTH_TOKEN` | `<密钥>` | 公司分配的API凭证，必填 |
| `ANTHROPIC_BASE_URL` | `https://gateway.ai.dpu.tech` | API网关端点，必填 |
| `API_TIMEOUT_MS` | `3000000` | API请求超时时间（毫秒），推荐值 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | `"1"` | 禁用非必要流量，提升性能 |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `llm-large-claude` | 快速任务模型 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | `llm-large-claude` | 日常开发模型（默认） |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | `llm-large-claude` | 复杂任务模型 |

---

### 后续如何修改API配置

#### 修改API凭证

```bash
# 编辑配置文件
nano ~/.claude/settings.json
```

将 `ANTHROPIC_AUTH_TOKEN` 的值替换为新的API凭证。

#### 切换默认模型

```bash
# 编辑配置文件
nano ~/.claude/settings.json
```

修改对应的环境变量值，例如：

```json
{
  "env": {
    "ANTHROPIC_API_KEY": "company_assigned_api_key",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "llm-large-claude"
  }
}
```

#### 验证配置生效

```bash
# 在Claude Code中运行
claude
# 然后输入命令
/status
```

---

## 使用Claude Code

### 命令行调用

#### 启动步骤

```bash
# 进入项目目录
cd ~/your-project

# 启动Claude Code
claude
```

#### 首次启动

首次启动时，Claude Code会提示是否信任当前目录：

```
Do you want to allow Claude Code to access files in this directory?
```

- 输入 `y` 或选择 `Yes` 允许访问
- 输入 `n` 或选择 `No` 拒绝访问

#### 基本使用

启动后，您可以直接与Claude对话：

```
You: 帮我创建一个简单的Python Hello World程序

Claude: 我来帮您创建一个简单的Python Hello World程序...

[生成代码]
```

#### 常用命令

| 命令 | 说明 |
|------|------|
| `/help` | 显示帮助信息 |
| `/status` | 查看当前配置状态 |
| `/clear` | 清空对话历史 |
| `/exit` 或 `Ctrl+D` | 退出Claude Code |

---

### VSCode插件调用

可以在VSCode的扩展页面搜索`Claude Code for VS Code`安装Anthropic官方插件。

#### 方式一：通过命令面板

1. 在VSCode中按 `Ctrl+Shift+P`（Windows/Linux）或 `Cmd+Shift+P`（Mac）打开命令面板
2. 输入 `Claude Code` 查看可用命令
3. 选择对应的功能执行

**常用命令**：
- `Claude Code: Open in New Window` - 启动新窗口会话
- `Claude Code: Open in Terminal` - 启动新终端打开会话

#### 方式二：通过Chat面板

1. 在VSCode左侧点击 Claude Code 图标
2. 在Chat面板中直接输入问题


## 测试验证

### 功能测试

#### 测试1：简单对话测试

```
You: 请用一句话介绍什么是AI

Claude: 人工智能（AI）是计算机科学的一个分支...
```

#### 测试2：代码生成测试

```
You: 用Python写一个计算斐波那契数列的函数

Claude: [生成Python代码]
```

#### 测试3：文件操作测试

```
You: 在当前目录创建一个名为test.txt的文件，内容为"Hello Claude"

Claude: [执行文件创建操作]
```

---

## 补充说明

### 版本升级

定期检查并升级Claude Code以获得最新功能和安全更新：

```bash
# 检查当前版本
claude --version

# 升级到最新版本
claude update
```

### 推荐版本

- **最低要求**：2.1.42 或更高
- **推荐使用**：最新稳定版

---

## 相关资源

### 官方文档

- [Claude Code 官方文档](https://docs.anthropic.com/zh-CN/docs/claude-code/overview)
- [智谱AI开放文档](https://docs.bigmodel.cn/cn/guide/develop/claude)

### 内部文档

- [星云智联VM配置指南](../vm-configuration/星云智联配置VM基础说明.md)

### 技术支持

如遇问题，请联系星云智联技术团队。

---

**文档版本**：v1.0
**最后更新**：2026年4月13日
**维护者**：星云智联技术团队
