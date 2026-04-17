# Claude Code 插件市场使用指南

> 本指南介绍如何在Claude Code中添加市场并安装插件，扩展Claude Code的能力。

---

## 为什么要使用Claude市场以及插件

Claude Code作为一款强大的AI编程助手，其核心能力已经非常出色。但通过市场和插件机制，可以进一步扩展其能力：

- **定制化能力扩展**：插件可以为Claude Code添加特定领域的专业知识和技能
- **企业知识库集成**：通过企业内部插件，Claude可以了解公司特定的开发规范、技术栈和业务逻辑
- **工作流自动化**：插件可以封装常用的开发流程，提高工作效率
- **团队协作**：团队成员可以共享同一套插件配置，确保开发一致性

---

## Claude市场与插件的关系

Claude Code采用"市场（Marketplace）- 插件（Plugin）"的两层架构：

```
Marketplace（市场）
    └── Plugin 1（插件）
    └── Plugin 2（插件）
    └── Plugin 3（插件）
    └── ...
```

- **Marketplace（市场）**：插件的托管仓库，类似于npm仓库或Docker Hub。一个市场可以包含多个插件。
- **Plugin（插件）**：具体的扩展功能单元，包含Skill定义、模板文件等资源。

用户需要先添加市场，然后从市场中安装所需的插件。

---

## 添加星云市场并安装插件

### 步骤1：添加市场

在Claude Code对话中输入以下命令添加星云智联的市场：

```
/plugin marketplace add git@github.com:nebula-matrix/nebula-matrix-skills.git
```

### 步骤2：安装和管理插件

添加市场后，在Claude Code中输入：

```
/plugin
```

在打开的界面中，找到 `Marketplaces` 部分，可以在此进行插件的安装和管理：

<!-- 图片占位：后续插入插件管理界面截图 -->

---

## 常见问题

### 添加市场时提示SSH认证失败

**错误现象**：在安装市场时提示ssh或http认证失败的错误。

**问题原因**：

在[配置Git SSH密钥](../environment-setup/Claude-Code开发环境配置指南.md#配置git-ssh密钥)章节，生成SSH密钥时，如果指定了密码短语（Passphrase），由于市场添加的界面是非交互式的，无法将要求输入密码的提示框传递到界面上，导致Git进程默默挂起，最终触发超时或直接返回认证失败（Authentication failed）。

**解决方案**：使用SSH Agent托管并缓存密钥

通过系统的 `ssh-agent` 程序在后台记住您的Passphrase，这样Claude Code调用Git时系统会自动提供密钥，不再需要交互式输入。

**具体执行步骤**：

1. 打开Ubuntu终端

2. 启动ssh-agent后台进程：

   ```bash
   eval "$(ssh-agent -s)"
   ```

3. 将SSH私钥添加到agent中：

   ```bash
   # 如果使用的是RSA密钥
   ssh-add ~/.ssh/id_rsa

   # 如果使用的是ED25519密钥
   ssh-add ~/.ssh/id_ed25519
   ```

4. 重新打开Claude Code，再次尝试添加市场

⚠️ **提示**：每次重启电脑或新开隔离的终端进程后，可能需要手动重新执行 `ssh-add` 命令。

---

## 相关资源

- [Claude Code开发环境配置指南](../environment-setup/Claude-Code开发环境配置指南.md)
- [Claude Code官方文档](https://docs.anthropic.com/zh-CN/docs/claude-code/overview)

---

**文档版本**：v1.0
**最后更新**：2026年4月17日
**维护者**：星云智联技术团队
