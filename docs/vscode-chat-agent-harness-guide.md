# VS Code 聊天窗口 Agent Harness 选择指南

> **适用范围**：Visual Studio Code 聊天窗口或 Agents 窗口中的 **Session Target** 选项：Copilot、Local、Claude、Codex 和 Cloud。本文重点解释日常使用最多的 Copilot 与 Local 的差别。  
> **信息核对日期**：2026-10-19。VS Code 的 Agent 功能和预览状态持续变化；若界面、账号要求或可用能力与本文不同，请以当前版本的 VS Code 和官方文档为准。

## 先说结论

- **日常编码优先选 Copilot**：这是 VS Code 官方推荐的日常 Agent harness。它使用 Copilot SDK 和 Agent Host，适合跨文件理解、实现、测试和持续跟进任务；支持的会话可在 VS Code、Copilot CLI 等界面间继续。
- **Local 适合在当前编辑器窗口里交互式地完成工作**：它使用 VS Code 内置 harness 和当前窗口的 Extension Host，直接在当前工作区迭代；但会话依赖该编辑器窗口，关掉窗口会停止仍在进行的工作。
- **“Local”说的是 harness，不等于“模型在本机运行”**。Local 可以使用 VS Code 配置的 Copilot 模型，也可以配置支持的自带模型凭据。Copilot、Claude、Codex harness 同样可以在本机工作区中执行。
- **Cloud 是执行目标，不是一个独立的 harness**：任务在云端服务提供方的基础设施上针对 GitHub 仓库运行，通常以 Pull Request 返回结果；它看不到本机编辑器中的未提交上下文。
- **Claude 和 Codex 是各自生态的 Agent harness**。如果只是想换模型，不一定要换 harness；例如在 Copilot harness 里选择 Claude 模型，仍然是 Copilot harness。

## 先把四个概念分开

| 概念 | 它决定什么 | 例子 |
|---|---|---|
| **Harness（Agent 执行框架）** | 如何组织 Agent 循环、上下文、工具调用、权限和会话状态 | Copilot、Local、Claude、Codex |
| **模型 / 模型访问来源** | 用什么模型推理，以及通过哪个账号、订阅或 API 凭据结算 | Copilot 提供的模型、Claude 模型、ChatGPT 模型、BYOK |
| **执行环境** | 代码和工具实际在哪个工作区或机器上运行 | 本机文件夹、Worktree、Dev Container、远程主机 |
| **Session Target** | 为当前会话选用哪个 harness，或选择 Cloud 执行目标 | Chat / Agents 窗口中的目标选择器 |

可以把一次 Agent 会话理解为：

```text
Harness 负责“怎么协作和调用工具”
模型负责“怎么推理和生成”
执行环境负责“在哪些文件和机器上动手”
界面负责“如何开始、查看和控制会话”
```

因此，选择 **Claude 模型**不等于选择 **Claude harness**；选择 **Local harness**也不等于选择**本地模型**；在本机 VS Code 里启动会话，也不代表 Agent 一定使用本地推理。

## 五种选项概览

| 选项 | 核心是什么 | 适合的场景 | 需要记住的限制 |
|---|---|---|---|
| **Copilot** | Copilot SDK 驱动的共享 Agent 运行时，由 Agent Host 承载 | 日常编码、跨文件功能、代码库问答、修复与测试、需要在支持的 Copilot 界面间继续会话 | 需要 GitHub 登录和可用的 Copilot 权限；Copilot harness 的工具集合不保证与 Local 完全相同 |
| **Local** | VS Code 内置 Agent harness，在编辑器 Extension Host 中交互式运行 | 当前工作区内快速问答、计划、编辑和迭代；希望使用 VS Code 当前配置的模型 | 依赖当前编辑器窗口；关闭窗口会停止进行中的工作；它不是“本地模型”开关 |
| **Claude** | VS Code 集成的 Anthropic Claude Agent SDK | 需要 Claude Agent 工作流、Claude 项目配置或在当前工作区 / 隔离 Worktree 执行任务 | 模型来源与账单取决于所选的 Copilot 路由或 BYOK；可用功能受配置和权限模式影响 |
| **Codex** | OpenAI Codex；通过 Codex 扩展或实验性的 Agent Host 集成接入 | 已使用 Codex / ChatGPT 工作流、希望使用 Codex 的交互式或后台编码能力 | 具体入口和会话能力取决于所用集成；Agent Host 集成仍为实验性选项 |
| **Cloud** | 在提供方管理的远程基础设施上运行所选 Agent | 适合交办边界清楚、可在 GitHub 仓库独立完成的任务，之后以 PR 审查和合并 | 需要 GitHub 仓库与云端提供方权限；不能使用本机编辑器上下文、未提交文件或本地开发环境 |

## Copilot 与 Local：日常使用的关键区别

| 比较点 | Copilot | Local |
|---|---|---|
| **运行框架** | Copilot SDK + Agent Host | VS Code 内置 harness + Extension Host |
| **典型交互方式** | 共享 Copilot Agent 体验；Agent 可在后台继续工作 | 在当前聊天和工作区中交互式迭代 |
| **关闭编辑器窗口** | 会话可独立于该编辑器窗口继续；前提是 Agent Host 仍在运行。关闭窗口不等于退出 VS Code 应用 | 会话绑定当前编辑器窗口；关闭该窗口会停止进行中的工作 |
| **使用模型** | Copilot harness 支持的模型；通常使用 GitHub Copilot 登录上下文 | VS Code 配置的模型，可使用 Copilot 提供的模型，也可配置兼容的自带模型凭据 |
| **工具和个性化配置** | 使用 Copilot harness 支持的工具与配置；并非 Local 所有内置或扩展工具都必然可用 | 使用 VS Code 内置 harness 的工具与其对应的个性化设置 |
| **适合的任务规模** | 从日常问题到需要多轮、多文件执行和验证的任务 | 当前窗口内的短回合、小步交互任务，以及依赖 Local 特定工具或设置的任务 |
| **典型选择理由** | 希望采用推荐的日常 Agent 体验，或需要会话跨支持的 Copilot 界面继续 | 希望在当前窗口里直接协作，并明确需要 Local harness 的行为或工具 |

### 实际上怎么选

**大多数日常编码任务：先选 Copilot。**例如：

- 解释当前模块、追踪调用关系，或先调查代码库再提出修改计划。
- 添加一个跨多个文件的小功能，要求补测试并运行相关检查。
- 修复 Bug，要求先定位原因、最小化修改，再用测试验证。
- 需要暂时关掉某个编辑器窗口，但希望任务在 Agent Host 仍运行时继续。

**明确有 Local 需求时，再选 Local。**例如：

- 任务必须使用某个仅在 Local harness 中可用的内置或扩展工具。
- 想使用 VS Code 中为 Local 配置的模型凭据或 Local 专属的指令、Hooks 等设置。
- 任务是短而明确的交互式编辑，希望操作始终围绕当前编辑器窗口。

不要仅因为“代码在本机”就选 Local。Copilot、Claude 和 Codex 会话也能针对本机工作区工作；选择 Local 的真正理由应该是你需要 Local harness 的工具、配置或窗口内工作方式。

### 常见误解

1. **“Copilot 会话都是云端改代码，Local 才会改本地文件。”**  
   不准确。Harness、模型托管位置和代码执行位置是不同维度。Copilot harness 可以针对本机工作区调用工具并编辑文件；模型请求则可能由托管服务处理。

2. **“选了 Local，代码和提示词就不会离开电脑。”**  
   不能据此推断。Local 指 VS Code 内置 harness，不代表模型推理必然发生在本机。实际数据处理取决于所选模型、模型访问来源、VS Code 配置和组织策略。

3. **“我在 Copilot harness 里选了 Claude 模型，所以现在用的是 Claude harness。”**  
   不是。模型选择和 harness 选择相互独立；Claude 模型可以经 Copilot 路由使用，但会话仍由 Copilot harness 管理。

4. **“关掉 VS Code 窗口，Copilot 会话一定被杀掉 / 一定继续。”**  
   两种说法都过于绝对。Copilot Agent Host 会话可以独立于编辑器窗口继续，但 Agent Host 本身必须仍在运行；关闭一个窗口和退出整个 VS Code 应用不是一回事。

5. **“Copilot 和 Local 的工具、权限、指令配置完全一样。”**  
   不保证。不同 harness 的工具清单、模型支持、权限模式和个性化配置可能不同；切换 harness 前应确认需要的工具、MCP 配置和 Hooks 是否兼容。

## 什么时候用 Claude、Codex 或 Cloud

### Claude

当任务更适合 Anthropic Claude Agent SDK 的项目配置或工作流，或者你明确需要 Claude harness 时选择它。Claude 会话可以在当前工作区或隔离 Worktree 中工作。模型访问和账单可能来自 Copilot 订阅路由，也可能来自你配置的 Claude API Key 等 BYOK 方式；具体看模型选择器和账号配置。

**注意**：选择 Copilot harness 后再选 Claude 系列模型，不会自动获得 Claude harness 的配置和工作流。

### Codex

当你想使用 OpenAI Codex 的 Agent 工作流，或希望在 VS Code 中接续 Codex / ChatGPT 相关任务时选择它。Codex 可通过 OpenAI Codex 扩展提供，也有实验性的 Agent Host 集成；两种集成的入口、授权方式、模型选项和会话能力可能不同。按当前 VS Code 的提示安装、登录和确认权限，不要假设某个 Codex 选项一定已启用。

### Cloud

当任务可以脱离本机上下文、可以针对一个 GitHub 仓库独立完成，并且你希望由云端 Agent 处理后提交 PR 时选择 Cloud。当前 VS Code 文档列出 Copilot cloud agent，以及预览中的 Claude 和 Codex cloud agents；第三方 Cloud Agent 还可能需要在 Copilot 账号或仓库策略中启用。

Cloud 不是“更强的 Copilot harness”，也不是当前工作区的远程终端：它运行在服务提供方管理的基础设施上，不会自动看到本机未提交文件、本地终端状态、编辑器选区或开发容器。把本地还没提交的改动交给 Cloud 之前，先提交到 GitHub，或改用能访问该工作区的本地 / 远程执行环境。

## 用任务特征做选择

| 任务情况 | 建议选择 | 原因 |
|---|---|---|
| 日常问答、读代码、解释错误 | **Copilot** | 作为日常默认 harness，能在 IDE 上下文中进行代码库问答 |
| 实现小功能、修 Bug、补测试并验证 | **Copilot** | 适合 Agent 多步骤执行；交付前仍要审阅 Diff 和验证结果 |
| 希望使用特定 Local 工具或 VS Code Local 配置 | **Local** | 选择理由是 Local harness 的工具和配置，而不是“本地运行”的字面含义 |
| 项目要求 Claude Agent SDK 的配置或工作流 | **Claude** | 使用对应 Claude harness，而不是只在 Copilot 中换成 Claude 模型 |
| 工作流明确依赖 OpenAI Codex | **Codex** | 根据需要选择已启用的扩展集成或实验性 Agent Host 集成 |
| 与当前工作区无关、可独立针对 GitHub 仓库交付 PR | **Cloud** | 云端运行并返回 PR，但不能使用本机工作区上下文 |
| 代码必须运行在远程开发机或 Dev Container | **选 harness 后另选执行环境** | 远程机器 / 容器是执行环境，与 Copilot、Local、Claude、Codex 的选择分开 |

## 建议的日常工作流

1. 新建会话后，通常先选 **Copilot**；只有明确需要其他 harness 特有的工具或工作流时才切换。
2. 先让 Agent 阅读相关代码并说明计划；对跨文件或高风险改动，确认边界、测试范围和不应修改的部分。
3. 让 Agent 做范围受控的修改，并运行项目实际使用的测试、Lint 或构建命令。
4. 检查变更 Diff、测试输出和未解决的问题。Agent 的自述不是测试证据，也不能替代人工审查。
5. 独立、可交付为 PR 的任务可以考虑 Cloud；依赖未提交文件、本地服务或编辑器上下文时留在本地工作区。
6. 需要改用其他 harness 时，在提交首条提示前选择目标最清楚；已经开始的 Local 会话也可按 VS Code 提供的目标选项移交。移交前检查目标 harness 是否能访问所需工具和上下文。

## 隐私、安全与团队策略

- Harness 名称不能单独说明提示词、代码或模型请求是否离开设备。确认模型提供方、账号 / API Key、数据处理条款及组织策略。
- Agent 可能读取文件、修改代码、运行终端命令或调用外部工具。对高影响操作保留人工确认；特别检查部署、删除、凭据访问和外网请求。
- 开启自动批准或放宽沙箱限制前，了解 Agent 可读写的文件路径、网络权限、终端权限以及当前工作区是否可信。
- Cloud 会访问 GitHub 仓库内容，不会继承本机未提交变更。仅在该仓库和任务获准使用云端 Agent 时使用。
- 企业策略可能隐藏 harness、限定模型、阻止第三方 Cloud Agent，或在受管环境下强制采用特定运行时；选项缺失时先检查 VS Code 版本、扩展、登录状态和组织策略。

## 参考资料

- [VS Code：Choose an agent harness](https://code.visualstudio.com/docs/agents/run/agent-harnesses) — 选项对照、设置要求、Local 会话移交和 Cloud 目标。
- [VS Code：Understand agent harnesses](https://code.visualstudio.com/docs/agents/concepts/agent-harnesses) — Harness、模型访问、Agent 角色和执行环境的区别。
- [VS Code：Build with AI in VS Code](https://code.visualstudio.com/docs/agents/overview) — VS Code Agent 的整体工作方式。
- [VS Code：Manage approvals and permissions](https://code.visualstudio.com/docs/agents/run/approvals) — 审批、权限和不同 harness 的安全控制。
- [VS Code：Run and manage remote agent sessions](https://code.visualstudio.com/docs/agents/run/remote-agent-sessions) — 远程主机、Dev Container 与浏览器端 Agents 窗口。

> 产品名称、设置键、账户要求和预览功能会变动。本文描述的是 VS Code 官方文档在信息核对日期所记录的行为，不应替代当前 UI 提示、服务条款或组织管理员的配置。
