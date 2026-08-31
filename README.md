# 曾美团队-育儿科普

这是“曾美团队-育儿科普”的公开安装仓库，只提供 Codex 插件外壳和 ChatGPT 接入配置。

核心创作规则、完整 Skill、长期记忆 TXT、四种类型 TXT、授权后台和服务器程序均不在本仓库，也不包含在插件安装包中。

## Codex 安装

在 Codex 的“插件”页面添加此 GitHub 市场仓库，然后安装“曾美团队-育儿科普”。首次使用时，按授权页面提示输入手机号和激活码。

## ChatGPT 使用

在自定义 GPT 的 Actions 中导入 [chatgpt/openapi.yaml](chatgpt/openapi.yaml)，再按授权页面提示完成激活。

## 公开文件

- `.agents/plugins/marketplace.json`：Codex 市场入口
- `.codex-plugin/plugin.json`：插件信息
- `.mcp.json`：远程服务连接地址
- `skills/`：不含核心规则的公开入口
- `chatgpt/`：ChatGPT Action 配置
