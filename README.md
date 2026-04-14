# 星云智联 AI 手册

> 本项目为公司内部大模型的应用与开发提供一站式指引

## 📖 项目简介

本项目旨在帮助开发者快速上手内部大模型技术，涵盖内网环境下的模型部署配置、API 接入规范、安全合规指南以及常见使用场景的最佳实践，确保大模型技术在内部业务中的高效、安全落地。

## 📚 文档目录

### 虚拟机配置 (2)

- [星云智联配置VM基础说明](docs/vm-configuration/星云智联配置VM基础说明.md) - 详细的虚拟机创建、配置和SSH连接指南
- [Windows配置Telnet指南](docs/vm-configuration/Window配置telnet.md) - Windows系统启用Telnet客户端及网络端口测试方法

### AI开发环境配置 (1)

- [Claude Code开发环境配置指南](docs/environment-setup/Claude-Code开发环境配置指南.md) - 完整的Claude Code安装、配置和使用指南

### Agent Skills 使用指南 (1)

- [NBL PPT Builder使用指南](docs/agent-skills/nbl-ppt-builder使用指南.md) - AI驱动的企业PPT自动生成工具，支持Markdown输入和智能模板选择

## 🔧 快速导航

### 新手入门

1. 首先阅读[VM配置指南](docs/vm-configuration/星云智联配置VM基础说明.md)了解如何创建和连接虚拟机
2. 准备SSH密钥（集群部署的VM不允许密码登录）
3. 使用VSCode Remote-SSH插件连接到虚拟机

### 常见场景

**虚拟机配置：**
- 🔐 SSH密钥生成与配置
- 🖥️ 虚拟机克隆与硬件资源配置
- 🌐 静态IP地址规划
- 🔗 VSCode远程开发环境搭建
- 🔍 网络端口连通性测试

**Agent Skills应用：**
- 📊 自动生成专业PPT演示文稿
- 📝 Markdown文档转换为可视化演示
- 🎨 智能模板选择与内容排版

## 🌟 主要内容

- **虚拟机配置**: 完整的VM创建、网络配置和远程连接指南
- **开发环境**: Claude Code等AI开发工具的安装和配置说明
- **Agent Skills**: 自动化工具使用方法和最佳实践分享
- **故障排查**: 解决网络连接、权限配置等常见问题

## 🤝 贡献指南

如需补充或更新文档，请：

1. 确保使用Markdown格式
2. 遵循现有文档的结构和风格
3. 图片等资源放置在对应文档的assets目录下
4. 提交前检查链接的有效性

## 📧 联系方式

如有问题或建议，请联系星云智联技术团队。

---

**维护者**: 星云智联技术团队
**最后更新**: 2026年4月
