# ChatGPT 接入

在自定义 GPT 的 Actions 中导入 `openapi.yaml`，将 `servers.url` 改成部署后的 HTTPS 域名。

首次使用由 ChatGPT 的 OAuth 授权流程打开 `https://4lei.073955.com/oauth/authorize`，用户在授权页输入手机号和激活码。授权成功后，ChatGPT 使用 OAuth 访问令牌调用 `POST /chatgpt/run`。

手机号和激活码由服务器授权页校验。访问令牌绑定授权浏览器所在电脑，后台解绑或封禁后会立即失效。
