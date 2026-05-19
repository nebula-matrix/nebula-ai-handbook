# VS Code 通过 JMS 堡垒机连接远程服务器

本文档介绍如何在 Windows 环境下，使用 VS Code 的 Remote-SSH 插件，通过 JMS（Jump Server，堡垒机）连接内网远程服务器进行开发。

---

## 目录

- [前置准备](#前置准备)
- [Windows 环境配置](#windows-环境配置)
- [VS Code 配置](#vs-code-配置)
- [常见问题排查](#常见问题排查)

---

## 前置准备

### 所需文件

- [scp.exe](assets/vscode连接jms_1779161782.exe) — 用于替换系统默认的 SCP 传输工具

> **提示**：请将上述文件下载到本地备用。

---

## Windows 环境配置

1. **创建 SSH 工具目录**

   在 `C` 盘根目录下新建名为 `ssh` 的文件夹：

   ```
   C:\ssh\
   ```

2. **复制系统 OpenSSH 文件**

   将 Windows 系统自带的 OpenSSH 客户端文件全部复制到上一步创建的 `C:\ssh\` 目录中。源文件位置：

   ```
   C:\Windows\System32\OpenSSH\
   ```

3. **替换 SCP 可执行文件**

   将前置准备中下载的 `scp.exe` 复制到 `C:\ssh\` 目录，**覆盖**同名文件。

---

## VS Code 配置

### 1. 配置 SSH Remote 连接

打开 VS Code，按 `Ctrl + Shift + P`，输入并选择 **Remote-SSH: Open SSH Configuration File**，编辑配置文件，添加如下主机条目：

```ssh
Host 10.10.3.33
  HostName 10.10.3.33
  User JMS-TODO
  Port 2222
  ForwardAgent no
```

> **说明**：
> - `User` 以及下文用到的密钥，需通过内网 `10.10.3.33` 登录 `10.10.5.7` 的 SSH 向导获取。
> - 登录后界面如下图所示：
>
> ![SSH 向导页面](assets/vscode连接jms_1779161783.png)

### 2. 修改 VS Code 用户设置

按 `Ctrl + Shift + P`，输入并选择 **首选项: 打开用户设置 (JSON)**，在配置文件中新增以下内容：

> **注意**：配置中的 `10.10.3.33` 应与 SSH 配置中的 `Host` 保持一致。

```json
{
  "remote.SSH.path": "C:\\ssh\\ssh.exe",
  "remote.SSH.useExecServer": false,
  "remote.SSH.remotePlatform": {
    "10.10.3.33": "linux"
  }
}
```

### 3. 发起连接

1. 按 `Ctrl + Shift + P`，选择 **Remote-SSH: Connect to Host...**，选择 `10.10.3.33`。
2. 当提示输入密码时，填入从 `10.10.5.7` SSH 向导获取的密码。
3. 等待连接建立，即可在远程服务器上打开文件夹进行开发。

---

## 常见问题排查

### 如何查看连接日志

如果连接失败，可通过以下步骤查看详细日志：

1. 在 VS Code 右下角点击连接状态提示。
2. 打开 **输出** 面板（`Ctrl + Shift + U`）。
3. 在右上角下拉菜单中选择 **Remote-SSH**。
4. 根据日志中的错误信息进行排查。

![查看 Remote-SSH 输出日志](assets/vscode连接jms_1779161784.png)

### 错误：LocalDownloadFailed

**现象**：

![LocalDownloadFailed 错误](assets/vscode连接jms_1779161785.png)

**解决方案**：

- 检查并关闭本地代理软件（如 Clash、V2RayN、Proxifier 等）。
- 重新尝试连接。

---
