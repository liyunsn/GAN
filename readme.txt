GitHub Copilot / GitHub Models 接入 OpenClaw 使用说明

一、先说结论
1. 普通用户不能直接“获取一个 Copilot API key”后再像 OpenAI key 一样填给 OpenClaw。
2. OpenClaw 使用 Copilot 的推荐方式是它自带的 `github-copilot` provider：先登录 GitHub，后续由 OpenClaw 自动换取短期 Copilot token。
3. 如果你的目标其实是“在 OpenClaw 里调用 GitHub 托管模型”，也可以不用 Copilot provider，而是走 GitHub Models 的 OpenAI-compatible 接口，此时使用的是 GitHub PAT，不是 Copilot API key。

二、推荐方案：直接使用 OpenClaw 内置的 GitHub Copilot Provider

适用场景：
- 你已经有可用的 GitHub Copilot 账号/套餐。
- 你希望在 OpenClaw 中直接使用 Copilot 可访问的模型。
- 你不想自己维护第三方代理或手动抓 token。

前置条件：
- 已安装 OpenClaw
- 有 GitHub 账号
- Copilot 权限可用
- 当前终端可交互（可打开浏览器完成设备登录）

步骤：
1. 在终端执行登录命令：
   `openclaw models auth login-github-copilot`
2. 按终端提示打开网页。
3. 输入一次性验证码并完成 GitHub 授权。
4. 等待终端登录完成，不要提前关闭窗口。
5. 设置默认模型，例如：
   `openclaw models set github-copilot/claude-opus-4.7`
6. 如果某个模型不可用，换成你当前 Copilot 套餐可访问的模型，例如：
   `github-copilot/gpt-4.1`
7. 完成后即可在 OpenClaw 中正常使用该模型。

说明：
- 这条路径下，你不需要手动找 Copilot API key。
- OpenClaw 会保存 GitHub 登录信息，并在运行时自动换取 Copilot API token。
- 模型是否可用取决于你的 GitHub/Copilot 套餐。

三、服务器/无交互方案：导入已有 GitHub Token

适用场景：
- 你在远程服务器、容器、自动化环境里部署 OpenClaw。
- 不方便在当前环境执行浏览器设备登录。

OpenClaw 支持以下环境变量优先级：
1. `COPILOT_GITHUB_TOKEN`
2. `GH_TOKEN`
3. `GITHUB_TOKEN`

推荐步骤：
1. 准备一个已经可用于 Copilot 的 GitHub token。
2. 在环境中写入变量，例如：
   `export COPILOT_GITHUB_TOKEN=你的_token`
3. 执行无交互初始化：
   `openclaw onboard --non-interactive --accept-risk --auth-choice github-copilot --github-copilot-token "$COPILOT_GITHUB_TOKEN" --skip-channels --skip-health`
4. 设置默认模型，例如：
   `openclaw models set github-copilot/gpt-4.1`

说明：
- 这里导入的是 GitHub token，不是单独发放的 Copilot API key。
- 不建议使用抓包、提取 IDE 内部临时凭证等非官方方式。

四、替代方案：使用 GitHub Models 作为 OpenAI-compatible Provider

适用场景：
- 你的真正需求是“给 OpenClaw 一个可兼容 OpenAI 协议的模型接口”。
- 你不强依赖 Copilot 原生 provider。
- 你希望直接通过 GitHub Models API 调用模型。

这条方案使用的是 GitHub PAT，不是 Copilot API key。

准备步骤：
1. 在 GitHub 里创建一个 PAT。
2. 给 PAT 开启 models 相关权限。
3. 保存为环境变量，例如：
   `export GITHUB_PAT=你的_pat`

GitHub Models 接口地址：
- `https://models.github.ai/inference/chat/completions`

在 OpenClaw 中可按 OpenAI-compatible provider 思路配置，示例：

{
  agents: {
    defaults: { model: { primary: "github-models/openai/gpt-4.1" } },
  },
  models: {
    mode: "merge",
    providers: {
      "github-models": {
        baseUrl: "https://models.github.ai/inference",
        apiKey: "${GITHUB_PAT}",
        api: "openai-completions",
        models: [
          { id: "openai/gpt-4.1", name: "GitHub Models - GPT-4.1" }
        ],
      },
    },
  },
}

调用思路：
1. OpenClaw 把该 provider 当作 OpenAI-compatible 接口使用。
2. 请求会发到 GitHub Models endpoint。
3. 鉴权使用你的 GitHub PAT。

五、如何选择

如果你的目标是“在 OpenClaw 里直接用 Copilot”：
- 选方案二（内置 `github-copilot` provider），这是首选。

如果你的目标是“只要能在 OpenClaw 里调用 GitHub 的模型接口”：
- 选方案四（GitHub Models + PAT）。

如果你在服务器或容器里部署：
- 选方案三（无交互 token 导入）。

六、常见问题

1. 我到底能不能拿到 Copilot API key？
- 常规情况下不能。官方支持的思路是账号登录、token 交换，或使用 GitHub Models 的 PAT。

2. 为什么登录后还可能报模型不可用？
- 因为不同 Copilot 套餐可访问的模型不同，需要换成你的账号实际可用的模型 ID。

3. OpenClaw 里最省事的做法是什么？
- 直接执行：
  `openclaw models auth login-github-copilot`
  然后再执行：
  `openclaw models set github-copilot/gpt-4.1`

七、最终建议

最佳实践如下：
- 不要再寻找“Copilot API key”这一类长期密钥。
- 直接使用 OpenClaw 官方支持的 `github-copilot` 登录流程。
- 若必须走 OpenAI-compatible 接口，则改用 GitHub Models + PAT。
- 所有 token 都应通过环境变量或安全密钥管理，不要明文写入仓库。
