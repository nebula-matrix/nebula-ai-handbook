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
   - [常见问题排查](#常见问题排查)
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
sudo cp /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.backup
```

#### 步骤3：修改为阿里云源

```bash
# 编辑源配置文件
sudo nano /etc/apt/sources.list.d/ubuntu.sources
```

将配置修改为：

```yaml
Types: deb
URIs: https://mirrors.aliyun.com/ubuntu/
Suites: noble noble-updates noble-backports noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
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

### 安装Node.js 22

Claude Code需要Node.js 18或更高版本，本指南使用Node.js 22。

#### 步骤1：安装必要依赖

```bash
sudo apt install -y ca-certificates curl gnupg
```

#### 步骤2：添加NodeSource GPG密钥

```bash
mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
```

#### 步骤3：添加NodeSource 22.x仓库

```bash
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_22.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list
```

#### 步骤4：安装Node.js

```bash
sudo apt update
sudo apt install -y nodejs
```

#### 步骤5：验证安装

```bash
node --version
npm --version
```

预期输出：
```
v22.x.x
10.x.x
```

---

