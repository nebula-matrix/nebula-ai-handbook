# Claude Code开发环境配置指南

> 本手册指导开发者在虚拟机中配置Claude Code开发环境，涵盖从环境准备到测试验证的完整流程。

---

## 目录

1. [前置条件](#前置条件)
2. [环境准备](#环境准备)
   - [切换apt源为阿里云](#切换apt源为阿里云)
   - [安装Node.js 22](#安装nodejs-22)
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
   - [基本连接测试](#基本连接测试)
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
# 备份原配置文件
sudo mkdir /etc/apt/backup_config
sudo mv /etc/apt/sources.list /etc/apt/backup_config/ 2>/dev/null
sudo mv /etc/apt/sources.list.d/ubuntu.sources /etc/apt/backup_config/ 2>/dev/null
```

#### 步骤3：修改为阿里云源

```bash
# 编辑源配置文件
sudo nano /etc/apt/sources.list
```

将配置修改为（仅对ubuntu24有效）：

```yaml
deb http://mirrors.aliyun.com/ubuntu/ noble main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ noble main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ noble-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ noble-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ noble-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ noble-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ noble-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ noble-backports main restricted universe multiverse
```

保存并退出（`Ctrl+X`，然后 `Y`，再 `Enter`）

#### 步骤4：更新软件包列表

```bash
sudo apt update
```

预期输出：应该看到 "Hit" 和 "Get" 消息，表示正在从阿里云镜像获取包列表

#### 其他发行版参考

| 发行版 | 镜像站地址 |
|--------|-----------|
| Debian | [阿里云Debian镜像](https://developer.aliyun.com/mirror/debian) |
| CentOS/Rocky Linux | [阿里云CentOS镜像](https://developer.aliyun.com/mirror/centos) |
| 其他发行版 | [阿里云镜像站](https://developer.aliyun.com/mirror/) |

---

### 安装Node.js

Claude Code需要Node.js 18或更高版本。推荐使用 **nvm (Node Version Manager)** 进行安装，便于管理多个Node.js版本。

#### 步骤1：下载并安装nvm

```bash
# 下载并安装 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

# 代替重启shell
\. "$HOME/.nvm/nvm.sh"
```

#### 步骤2：安装Node.js

```bash
# 下载并安装Node.js（LTS版本）
nvm install 24
```

#### 步骤3：验证安装

```bash
# 验证Node.js版本
node -v

# 验证npm版本
npm -v
```

预期输出：
```
v24.14.1
11.11.0
```

💡 **提示**：使用nvm可以方便地在不同Node.js版本之间切换，适合开发环境。

#### 常见问题

**nvm命令找不到**

如果在执行 `nvm` 命令时提示找不到命令，请尝试：

1. 重启终端
2. 或手动加载nvm：`\. "$HOME/.nvm/nvm.sh"`

**安装速度慢**

如果从官方源下载速度较慢，可以考虑配置国内镜像源加速下载。

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

| GLM模型 | 对应Claude模型 | 环境变量 | 用途 | 特点 |
|---------|---------------|---------|------|------|
| **GLM-4.7** | Claude Opus/Sonnet | `ANTHROPIC_DEFAULT_OPUS_MODEL`<br>`ANTHROPIC_DEFAULT_SONNET_MODEL` | 日常开发、复杂任务 | 性能与速度平衡 |
| **GLM-4.5-Air** | Claude Haiku | `ANTHROPIC_DEFAULT_HAIKU_MODEL` | 快速响应、简单任务 | 速度最快，成本最低 |

#### 模型选择建议

- **日常开发**：推荐使用 `GLM-4.7`（对应Opus/Sonnet）
- **快速查询/简单任务**：推荐使用 `GLM-4.5-Air`（对应Haiku）

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
    "ANTHROPIC_AUTH_TOKEN": "your_company_assigned_token",
    "ANTHROPIC_BASE_URL": "https://open.bigmodel.cn/api/anthropic",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "glm-4.5-air",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "glm-4.7",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "glm-4.7"
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
    "ANTHROPIC_AUTH_TOKEN": "064a10a745c94dd1beb5ce7a09929681.3BYcgsP13aoVLMgT",
    "ANTHROPIC_BASE_URL": "https://open.bigmodel.cn/api/anthropic",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "glm-4.5-air",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "glm-4.7",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "glm-4.7"
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
| `ANTHROPIC_AUTH_TOKEN` | `064a10a745c94d...` | 公司分配的API凭证，必填 |
| `ANTHROPIC_BASE_URL` | `https://open.bigmodel.cn/api/anthropic` | 智谱AI的API端点，必填 |
| `API_TIMEOUT_MS` | `3000000` | API请求超时时间（毫秒），推荐值 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | `"1"` | 禁用非必要流量，提升性能 |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `glm-4.5-air` | 快速任务模型 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | `glm-4.7` | 日常开发模型（默认） |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | `glm-4.7` | 复杂任务模型 |

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
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "glm-4.7"
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

#### 方式一：通过命令面板

1. 在VSCode中按 `Ctrl+Shift+P`（Windows/Linux）或 `Cmd+Shift+P`（Mac）打开命令面板
2. 输入 `Claude Code` 查看可用命令
3. 选择对应的功能执行

**常用命令**：
- `Claude Code: Start New Session` - 启动新会话
- `Claude Code: Explain Code` - 解释选中代码
- `Claude Code: Refactor Code` - 重构选中代码

#### 方式二：通过Chat面板

1. 在VSCode左侧点击 Claude Code 图标
2. 在Chat面板中直接输入问题
3. 可以选中代码后右键选择 `Ask Claude Code`

#### 示例操作

```
# 在Chat面板中输入
帮我优化这个函数的性能

# 或选中代码后
帮我解释这段代码的作用
```

---

## 测试验证

### 基本连接测试

#### 步骤1：启动Claude Code

```bash
claude
```

#### 步骤2：检查配置状态

在Claude Code中输入：

```
/status
```

#### 预期输出

```
Claude Code Version: 2.1.x.x
API Configuration:
  Base URL: https://open.bigmodel.cn/api/anthropic
  Default Model: glm-4.7
  Status: ✅ Connected
```

---

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
