# GitHub Actions 详细使用指南

> 本文面向需要为项目建立持续集成（CI）、持续交付（CD）和自动化运维流程的开发者。示例以 GitHub Actions 当前通用语法为基础，命令和版本号应根据项目实际技术栈调整。

---

## 1. GitHub Actions 是什么

GitHub Actions 是 GitHub 提供的自动化平台。它监听仓库事件，按照工作流文件中的定义启动运行器（Runner），依次执行检查、测试、构建、打包、发布和部署等任务。

一个典型流程如下：

```text
代码提交 / Pull Request
        |
        v
触发 Workflow
        |
        v
准备 Runner + 检出代码
        |
        v
安装运行环境和依赖
        |
        v
Lint / 类型检查 / 单元测试
        |
        v
构建制品并上传 Artifact
        |
        v
人工审批或自动部署
        |
        v
发布结果、日志和通知
```

### 1.1 CI、持续交付和持续部署

| 概念 | 含义 | 常见触发方式 |
|------|------|--------------|
| CI（持续集成） | 每次提交或合并请求自动检查代码 | `push`、`pull_request` |
| 持续交付 | 每次通过验证后都生成可发布制品，发布动作可以人工确认 | `workflow_run`、环境审批 |
| 持续部署 | 通过所有门禁后自动部署到目标环境 | `push` 到发布分支、标签 |

建议先建立稳定的 CI，再逐步加入制品、预发布环境和生产部署。不要一开始就让每次提交直接修改生产环境。

### 1.2 核心术语

| 术语 | 说明 |
|------|------|
| Workflow | 一个 YAML 工作流文件，位于 `.github/workflows/` |
| Event | 触发工作流的事件，如 push、PR、手工运行 |
| Job | 工作流中的独立任务单元，可运行在不同 Runner 上 |
| Step | Job 内按顺序执行的步骤 |
| Action | 可复用的步骤封装，例如 `actions/checkout` |
| Runner | 实际执行 Job 的虚拟机或自托管机器 |
| Artifact | 工作流产生的可下载文件，如构建包、覆盖率报告 |
| Environment | GitHub 中的环境，如 staging、production，可配置审批和环境密钥 |
| Context | GitHub 在运行时提供的上下文数据，如提交 SHA、分支名、事件信息 |
| Expression | `${{ ... }}` 表达式，用于读取上下文、变量和函数 |

---

## 2. 文件位置与最小工作流

工作流必须放在仓库的 `.github/workflows/` 目录中。文件扩展名通常为 `.yml` 或 `.yaml`。

最小示例：

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Check out source
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

### 2.1 创建第一个工作流

```bash
mkdir -p .github/workflows
touch .github/workflows/ci.yml
git add .github/workflows/ci.yml
git commit -m "ci: add GitHub Actions workflow"
git push
```

提交后，在仓库的 **Actions** 页面可以查看运行记录。第一次运行失败时，先打开失败的 Job，再展开失败的 Step 查看完整日志。

### 2.2 推荐的文件拆分

不要把所有事情都塞进一个超长文件。常见拆分方式：

```text
.github/
├── workflows/
│   ├── ci.yml                    # lint、类型检查、测试
│   ├── build.yml                 # 构建并上传制品
│   ├── deploy-staging.yml        # 部署预发布环境
│   ├── deploy-production.yml     # 生产发布和审批
│   └── dependency-review.yml     # 依赖和安全检查
└── dependabot.yml                # 依赖升级配置，可选
```

拆分的原则是：触发条件、权限、失败通知和发布边界明显不同的流程，优先分成不同 Workflow；同一流程中需要共享同一版本代码和制品的步骤，放在同一个 Workflow 中。

---

## 3. Workflow 的基本结构

一个工作流通常包括 `name`、`on`、顶层 `permissions`、`env`、`concurrency` 和 `jobs`。

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
    tags: ['v*.*.*']
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      deploy:
        description: Deploy after validation
        required: true
        default: false
        type: boolean

env:
  NODE_ENV: test

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
```

### 3.1 `name`

`name` 是 Actions 页面展示的工作流名称。名称应该能说明目的，例如 `Backend CI`、`Deploy Production`，不要都叫 `CI`。

### 3.2 `on`：工作流触发器

#### Push 和 Pull Request

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'
    paths:
      - 'src/**'
      - 'package.json'
      - 'package-lock.json'
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
```

`paths` 可以避免与当前项目无关的文档修改触发昂贵构建。注意：如果工作流是合并必需检查，过度使用路径过滤可能导致某些 PR 没有产生检查结果，反而阻塞合并。对必需检查要设计一个始终有结果的门禁 Job。

#### Tag 发布

```yaml
on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'
```

GitHub 的 glob 规则与完整正则表达式不同。复杂的版本格式建议在 Job 中使用脚本再次验证。

#### 手工运行 `workflow_dispatch`

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: Target environment
        required: true
        type: choice
        options:
          - staging
          - production
      version:
        description: Image tag or release version
        required: true
        type: string
      dry_run:
        description: Validate only
        required: true
        default: true
        type: boolean
```

在步骤中读取输入：

```yaml
- name: Show inputs
  run: |
    echo "environment=${{ inputs.environment }}"
    echo "version=${{ inputs.version }}"
    echo "dry_run=${{ inputs.dry_run }}"
```

生产工作流可以保留手工触发，但仍建议使用 GitHub Environment 审批和分支保护，不要仅依赖一个名为 `dry_run` 的输入框。

#### 定时运行 `schedule`

```yaml
on:
  schedule:
    - cron: '17 2 * * 1-5'
```

Cron 使用 UTC 时间。定时任务可能因为仓库不活跃、GitHub 服务负载或默认分支变化而延迟。不要把定时任务当作严格实时调度器。

#### 其他常用事件

```yaml
on:
  workflow_dispatch:
  workflow_call:
  workflow_run:
    workflows: [CI]
    types: [completed]
  release:
    types: [published]
```

`workflow_call` 用于可复用工作流；`workflow_run` 用于在另一个工作流结束后触发，但必须谨慎处理权限和不可信输入。

### 3.3 `jobs`

每个 Job 有自己的 Runner 和文件系统。Job 之间默认并行；通过 `needs` 建立依赖：

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
```

如果 `test` 失败，`build` 默认不会运行。使用 `needs.<job>.result` 可以在后续 Job 中判断结果，但不要随意使用 `if: always()` 掩盖真正的失败。

### 3.4 `steps`

```yaml
steps:
  - name: Check out source
    uses: actions/checkout@v4

  - name: Run a shell command
    run: npm test

  - name: Run multiple commands
    run: |
      npm run lint
      npm run typecheck

  - name: Set a step environment variable
    env:
      API_URL: https://example.test
    run: echo "Testing against $API_URL"
```

`uses` 调用 Action，`run` 执行 Shell 命令。尽量给步骤写清晰的 `name`，失败日志更容易定位。

---

## 4. Runner、Shell 与执行环境

### 4.1 GitHub 托管 Runner

常见 Runner：

```yaml
runs-on: ubuntu-latest
# runs-on: windows-latest
# runs-on: macos-latest
```

选择建议：

| Runner | 适合场景 |
|--------|----------|
| Ubuntu | Node、Python、Java、Go、Docker、绝大多数后端 CI，通常成本和启动速度较好 |
| Windows | Windows 专属构建、.NET Framework、PowerShell 验证 |
| macOS | Xcode、iOS/macOS 构建和签名 |

不要假设 Runner 上预装了某个特定版本的 Node、Python、Java 或 Docker。用 `setup-*` Action 显式声明版本。

### 4.2 Shell 差异

```yaml
- name: Bash script
  shell: bash
  run: |
    set -euo pipefail
    echo "${GITHUB_SHA}"

- name: PowerShell script
  shell: pwsh
  run: |
    $ErrorActionPreference = 'Stop'
    Write-Output $env:GITHUB_SHA
```

需要可预测的脚本行为时，显式指定 Shell，并在 Bash 脚本中使用 `set -euo pipefail`。跨平台项目应避免依赖 `sed`、路径分隔符和默认 Shell 的细微差异。

### 4.3 自托管 Runner

自托管 Runner 适用于内网资源、特殊硬件或无法使用托管 Runner 的构建。它也带来更高的维护和安全责任：

- 不要让来自不受信任 Pull Request 的代码运行在持久化自托管 Runner 上。
- 使用临时 Runner 或容器隔离工作目录。
- 定期更新操作系统、Runner 软件和构建工具。
- 限制 Runner 的网络访问和云权限。
- 不要在 Runner 主机上长期保存生产密钥。

---

## 5. 环境变量、Context 和 Expressions

### 5.1 三种常见数据来源

| 来源 | 适合存储 | 读取方式 |
|------|----------|----------|
| `env` | 非敏感配置，如运行模式、目录名 | Shell 中 `$NAME`，YAML 中 `${{ env.NAME }}` |
| `vars` | GitHub 仓库、组织或 Environment 级非敏感变量 | `${{ vars.NAME }}` |
| `secrets` | Token、密码、证书和私钥 | `${{ secrets.NAME }}` |

示例：

```yaml
env:
  APP_NAME: demo

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      BUILD_MODE: production
    steps:
      - name: Build
        env:
          API_URL: ${{ vars.API_URL }}
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
        run: |
          echo "Building $APP_NAME in $BUILD_MODE"
          npm run build
```

### 5.2 常用 Context

```yaml
- name: Print safe metadata
  run: |
    echo "repository=${{ github.repository }}"
    echo "ref=${{ github.ref }}"
    echo "sha=${{ github.sha }}"
    echo "event=${{ github.event_name }}"
    echo "actor=${{ github.actor }}"
```

常用字段：

- `github.repository`：`owner/repository`。
- `github.ref`：触发本次运行的完整引用。
- `github.ref_name`：分支或标签短名称。
- `github.sha`：本次运行对应的提交 SHA。
- `github.event_name`：触发事件名称。
- `github.event.pull_request.number`：Pull Request 编号，只在相关事件中存在。
- `runner.os`：当前 Runner 的操作系统。
- `job.status`：当前 Job 状态。
- `steps.<id>.outputs.<name>`：前一步输出。

### 5.3 条件表达式

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - run: echo "deploying"
```

常用函数：

```yaml
if: startsWith(github.ref, 'refs/tags/v')
if: contains(github.event.pull_request.labels.*.name, 'deploy')
if: success()
if: failure()
if: always()
```

表达式中的字符串使用单引号。不要把来自 Issue、PR 标题、分支名或用户输入的内容直接拼接到 Shell 命令中。

### 5.4 在步骤之间传递输出

```yaml
- name: Calculate image tag
  id: metadata
  run: echo "tag=${GITHUB_SHA::12}" >> "$GITHUB_OUTPUT"

- name: Use image tag
  env:
    IMAGE_TAG: ${{ steps.metadata.outputs.tag }}
  run: echo "${IMAGE_TAG}"
```

向 `$GITHUB_OUTPUT` 写入输出，而不是使用已经废弃的 `set-output` 命令。

### 5.5 多行环境变量和文件命令

```yaml
- name: Create configuration
  run: |
    {
      echo 'CONFIG<<EOF'
      echo 'line one'
      echo 'line two'
      echo 'EOF'
    } >> "$GITHUB_ENV"
```

将不可信内容写入环境变量或输出时，要避免内容中包含会结束 heredoc 的分隔符。对于外部输入，优先使用文件参数、JSON 解析器或安全的结构化接口。

---

## 6. 依赖安装、缓存和制品

### 6.1 Node.js CI 示例

```yaml
name: Node CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - run: npm ci
      - run: npm run lint --if-present
      - run: npm run typecheck --if-present
      - run: npm test
      - run: npm run build --if-present
```

`npm ci` 要求锁文件存在，并且不会修改锁文件，适合 CI。pnpm、Yarn 项目应使用对应的锁文件和安装命令，例如 `pnpm install --frozen-lockfile`。

### 6.2 Python CI 示例

```yaml
name: Python CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip

      - run: python -m pip install --upgrade pip
      - run: pip install -r requirements.txt
      - run: ruff check .
      - run: pytest -q
```

### 6.3 Java CI 示例

```yaml
- uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version: '21'
    cache: maven

- run: ./mvnw -B verify
```

Gradle 项目可以使用 `gradle` 缓存并通过 `./gradlew build` 执行构建。优先使用仓库中的 Wrapper，避免 Runner 全局工具版本漂移。

### 6.4 缓存的正确边界

缓存用于加速重复下载，不是构建产物存储，也不能代替锁文件。

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.cache/my-tool
    key: ${{ runner.os }}-my-tool-${{ hashFiles('**/lockfile') }}
    restore-keys: |
      ${{ runner.os }}-my-tool-
```

缓存注意事项：

- Key 必须包含操作系统、运行时版本和依赖锁文件哈希等影响因素。
- 不要缓存包含密钥、云凭据或用户数据的目录。
- 不要把 `node_modules` 当作默认缓存目标，除非非常清楚原生模块和 Node 版本的影响。
- 缓存命中只代表找到了旧数据，仍需保证依赖安装命令能够校验完整性。

### 6.5 上传和下载 Artifact

构建 Job 生成制品：

```yaml
- name: Build package
  run: npm run build

- name: Upload package
  uses: actions/upload-artifact@v4
  with:
    name: web-dist-${{ github.sha }}
    path: dist/
    if-no-files-found: error
    retention-days: 7
```

部署 Job 下载制品：

```yaml
- name: Download package
  uses: actions/download-artifact@v4
  with:
    name: web-dist-${{ needs.build.outputs.artifact_name }}
    path: dist/
```

推荐的制品策略：

1. 构建一次，测试同一个制品。
2. 给制品名和镜像 Tag 绑定不可变的提交 SHA。
3. 部署时下载已经验证过的制品，不要在部署 Job 中重新编译。
4. 设置合理的保留期限，避免长期积累大文件。

跨 Job 传递制品名时，可以使用 Job 输出：

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact_name: ${{ steps.meta.outputs.artifact_name }}
    steps:
      - id: meta
        run: echo "artifact_name=dist-${GITHUB_SHA}" >> "$GITHUB_OUTPUT"
```

---

## 7. Matrix：一次测试多个版本

Matrix 可以组合 Node、Python、数据库、操作系统等维度：

```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [20, 22]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: npm
      - run: npm ci
      - run: npm test
```

### 7.1 include 和 exclude

```yaml
strategy:
  matrix:
    node: [20, 22]
    database: [postgres, mysql]
    exclude:
      - node: 20
        database: mysql
    include:
      - node: 22
        database: postgres
        experimental: true
```

### 7.2 控制并发

Matrix 会快速放大 Runner 消耗。使用 `max-parallel` 限制并发：

```yaml
strategy:
  fail-fast: false
  max-parallel: 3
  matrix:
    python: ['3.11', '3.12', '3.13']
```

`fail-fast: false` 适合兼容性矩阵，因为一个版本失败时仍希望看到其他版本结果；常规 PR 检查则可以根据反馈速度和成本选择默认值。

---

## 8. Job 依赖、输出和并发控制

### 8.1 多阶段流水线

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint

  test:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test

  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build
```

如果每个 Job 都重复安装依赖，执行时间可能较长。可以在构建阶段重新安装以保持 Job 隔离，也可以把安装与测试合并到同一个 Job。不要为了节省几分钟而把所有步骤串成一个无法重试、无法定位失败的巨型 Job。

### 8.2 Concurrency

```yaml
concurrency:
  group: deploy-production
  cancel-in-progress: false
```

生产部署通常不应取消正在进行的部署。对 PR 验证可以按分支取消旧运行：

```yaml
concurrency:
  group: ci-${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
```

使用 `cancel-in-progress: true` 前要确认任务没有不可中断的外部副作用。

### 8.3 失败后清理或通知

```yaml
- name: Collect logs
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: failure-logs-${{ github.run_id }}
    path: logs/
    if-no-files-found: ignore
```

`always()` 会在成功、失败和取消时都尝试执行。对部署流程使用时要特别小心，清理步骤本身不应覆盖原始失败结果。

---

## 9. Secrets、权限和安全设计

### 9.1 Secrets 的层级

Secrets 可以配置在 Repository、Environment 或 Organization 层级。建议：

- 普通测试凭据放 Repository Secrets。
- staging 和 production 凭据放对应 Environment Secrets。
- 多仓库共享但权限受控的凭据放 Organization Secrets。
- 同名变量优先使用最接近当前 Job 的作用域，避免团队成员难以判断实际值。

绝不要把密钥写进：

- 工作流 YAML 的明文内容。
- 仓库代码、Issue、PR 评论和构建日志。
- Artifact、缓存或 Docker 镜像层。
- `echo` 输出和调试日志。

### 9.2 最小化 GITHUB_TOKEN 权限

在工作流顶层默认只读：

```yaml
permissions:
  contents: read
```

某个 Job 确实需要写入时再局部授权：

```yaml
jobs:
  release:
    permissions:
      contents: write
      id-token: write
```

常见权限包括：`contents`、`pull-requests`、`issues`、`packages`、`actions`、`deployments` 和 `id-token`。只授予当前 Job 必需的权限。

### 9.3 Fork Pull Request 的安全边界

来自 Fork 的 Pull Request 默认无法安全地使用仓库 Secrets。`pull_request_target` 会在目标仓库上下文中执行，拥有更高的权限，不能直接检出并执行不可信 PR 的代码。

高风险错误模式：

```yaml
on: pull_request_target

steps:
  - uses: actions/checkout@v4
    with:
      ref: ${{ github.event.pull_request.head.sha }}
  - run: npm test
```

这可能让不可信代码在有仓库权限的上下文中运行。更安全的原则是：

- 普通测试使用 `pull_request`，不给写权限和生产密钥。
- 需要评论或标签的机器人流程单独设计，并且不执行 PR 中的代码。
- 生产部署只允许受保护分支或受信任标签触发。
- 不要把 PR 标题、分支名、Issue 内容直接拼接进 Shell。

### 9.4 Action 版本固定

短版本标签方便升级，但标签可能移动。对高安全要求的仓库，可以固定到完整 Commit SHA，并用 Dependabot 管理升级：

```yaml
- uses: actions/checkout@v4

# 更严格的供应链控制：固定完整 SHA，并保留可读注释
# - uses: actions/checkout@<full-commit-sha> # v4.x
```

至少应：

- 只使用可信组织或经过审查的 Action。
- 避免直接运行陌生仓库提供的 Action。
- 定期升级 Actions 版本。
- 在组织策略中限制允许使用的 Actions。
- 对第三方 Action 查看其源码、权限需求和发布记录。

### 9.5 OIDC：避免长期云密钥

部署到 AWS、Azure、GCP 等云平台时，优先使用 GitHub OIDC 让云平台验证短期身份，而不是保存长期 Access Key。

工作流通常需要：

```yaml
permissions:
  contents: read
  id-token: write
```

云平台侧要配置：

- 可信 GitHub OIDC 发行方。
- 指定组织、仓库、分支或 Environment 的受信条件。
- 最小化云角色权限。
- 不允许任意分支获得生产角色。

具体登录 Action 和云平台配置以目标云厂商官方文档为准。OIDC 不是自动安全，错误的 Subject 条件仍可能授予过宽权限。

### 9.6 注入风险

不要这样写：

```yaml
- run: echo "Deploying ${{ github.event.pull_request.title }}"
```

PR 标题可能包含 Shell 元字符。更安全的方式是通过环境变量传递，并在脚本中按普通字符串处理：

```yaml
- name: Read title as data
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: printf '%s\n' "$PR_TITLE"
```

如果输入需要作为参数，使用数组形式的命令、严格的白名单和结构化解析，避免 `eval`、动态 Shell 代码和未转义拼接。

---

## 10. Environments、审批和发布门禁

GitHub Environment 可以为 staging、production 配置：

- Required reviewers：指定审批人。
- Wait timer：部署前等待一段时间。
- Environment secrets：只在进入该环境后提供。
- Environment variables：环境级非敏感配置。
- Branch/tag restrictions：限制哪些分支或标签可以部署。

部署 Job 示例：

```yaml
jobs:
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: release-${{ github.sha }}

      - name: Deploy
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
        run: ./scripts/deploy.sh
```

推荐发布链路：

```text
Pull Request
  -> lint / test / security checks
  -> merge main
  -> build immutable artifact
  -> deploy staging automatically
  -> smoke test
  -> production Environment 审批
  -> deploy exactly the same artifact
  -> health check / rollback
```

生产 Job 应明确限制触发条件：

```yaml
if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

不要仅通过前端输入或普通环境变量决定是否部署生产。

---

## 11. 可复用工作流和复合 Action

### 11.1 Reusable Workflow

被调用方：

```yaml
name: Reusable Node Test

on:
  workflow_call:
    inputs:
      node-version:
        required: false
        type: string
        default: '22'
    secrets:
      registry-token:
        required: false
    outputs:
      result:
        description: Test result
        value: ${{ jobs.test.outputs.result }}

jobs:
  test:
    runs-on: ubuntu-latest
    outputs:
      result: ${{ steps.result.outputs.value }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm
      - run: npm ci
      - run: npm test
      - id: result
        run: echo "value=passed" >> "$GITHUB_OUTPUT"
```

调用方：

```yaml
jobs:
  test:
    uses: ./.github/workflows/reusable-node-test.yml
    with:
      node-version: '22'
    secrets: inherit
```

`secrets: inherit` 会扩大可见范围，只有在调用方和被调用方边界清楚时使用。跨仓库调用应显式传递需要的 Secrets。

### 11.2 Composite Action

当多个工作流共享一组步骤时，可以创建 `.github/actions/setup-project/action.yml`：

```yaml
name: Setup Project

description: Install project dependencies
inputs:
  node-version:
    required: false
    default: '22'

runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: npm
    - shell: bash
      run: npm ci
```

使用：

```yaml
- uses: ./.github/actions/setup-project
  with:
    node-version: '22'
```

区别：Reusable Workflow 复用完整的 Job/工作流结构；Composite Action 复用一个 Job 内的步骤集合。

---

## 12. 常见项目的完整 CI 示例

### 12.1 Node.js：Lint、测试、构建和制品

```yaml
name: Node CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: node-ci-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - run: npm ci
      - run: npm run lint --if-present
      - run: npm run typecheck --if-present
      - run: npm test
      - run: npm run build

      - name: Upload build output
        uses: actions/upload-artifact@v4
        with:
          name: web-${{ github.sha }}
          path: dist/
          if-no-files-found: error
          retention-days: 7
```

### 12.2 Docker：构建并推送镜像

```yaml
name: Container Build

on:
  push:
    branches: [main]
    tags: ['v*.*.*']

permissions:
  contents: read
  packages: write

jobs:
  image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha
            type=ref,event=tag
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

生产环境不要只使用 `latest`。部署时应使用不可变的 SHA 或版本 Tag，并记录镜像 Digest。

### 12.3 Flutter：分析、测试和 Android 构建

```yaml
name: Flutter CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.32.x'
          channel: stable
          cache: true

      - run: flutter pub get
      - run: flutter analyze
      - run: flutter test --coverage
      - run: flutter build apk --release

      - uses: actions/upload-artifact@v4
        with:
          name: android-apk-${{ github.sha }}
          path: build/app/outputs/flutter-apk/*.apk
          if-no-files-found: error
```

iOS 构建需要 macOS Runner、Xcode 和签名凭据。签名证书与 Provisioning Profile 应通过受控 Secrets 或专用签名方案注入，不能提交到仓库。

---

## 13. Release、版本和部署策略

### 13.1 用标签触发发布

```yaml
on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release:
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: release-files
          path: dist/
      - uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
          files: dist/**
```

第三方 Release Action 使用前要审查其权限和维护状态。更高安全要求下，可以用 GitHub CLI 或官方 API 自己完成发布，并固定 CLI 版本。

### 13.2 版本号来源

推荐让版本号有单一来源：

- Git Tag：适合发布版本和回滚定位。
- `package.json`、`pyproject.toml` 或 Gradle 文件：适合语言生态工具。
- GitHub Release：适合对外可见的发布记录。

不要让 Workflow 自动计算多个互相不一致的版本号。部署日志至少记录：提交 SHA、版本号、镜像 Digest、触发人和目标环境。

### 13.3 回滚

一个可回滚的部署需要：

1. 制品不可变且可追踪。
2. 旧版本仍在 Artifact Registry 或发布存储中。
3. 部署脚本支持指定版本。
4. 健康检查失败时能停止流量切换。
5. 数据库迁移采用向前兼容策略，避免代码回滚后无法读取数据。

回滚通常应重新部署已验证的旧制品，而不是重新构建旧提交。

---

## 14. 测试报告、覆盖率和检查结果

仅在日志中打印测试结果不利于长期查看。可以上传 JUnit、覆盖率、截图和失败日志：

```yaml
- name: Run tests
  run: npm test -- --reporter=junit --outputFile=test-results.xml

- name: Upload test results
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: test-results-${{ github.run_id }}
    path: test-results.xml
    if-no-files-found: ignore
```

前端 E2E 测试可以在失败时上传截图和 Trace：

```yaml
- name: Run browser tests
  run: npx playwright test

- name: Upload Playwright report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: playwright-report-${{ github.run_id }}
    path: |
      playwright-report/
      test-results/
    retention-days: 14
```

报告中可能包含用户数据、Token 或内部 URL。上传前先检查脱敏策略和 Artifact 访问权限。

---

## 15. 依赖升级与安全检查

### 15.1 Dependabot

`.github/dependabot.yml` 示例：

```yaml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5

  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

Dependabot 只负责提出升级 PR。是否自动合并要结合测试、变更范围和安全等级判断。

### 15.2 依赖审查

```yaml
name: Dependency Review

on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  dependency-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high
```

安全检查不能代替代码审查。还应结合语言生态的锁文件审计、容器扫描、Secret 扫描、SAST 和 DAST，并根据项目风险设置真正的阻断等级。

---

## 16. 本地验证工作流

GitHub Actions 运行在远程 Runner 上，但很多问题可以本地尽早发现：

```bash
# 校验 YAML 语法
ruby -e "require 'yaml'; YAML.load_file('.github/workflows/ci.yml')"

# 查看本次变更
git diff -- .github/workflows/

# 使用 actionlint（需要先安装）
actionlint

# 使用 act 在本地模拟部分 GitHub Actions 流程（需要自行安装）
act pull_request
```

本地模拟不一定完全等价于 GitHub 托管 Runner，尤其是权限、OIDC、Environment 审批、缓存、服务容器和 macOS Runner。最终仍要在真实的测试分支上验证。

### 16.1 YAML 常见陷阱

- `on` 在某些 YAML 解析器中会被当作布尔值；不要用通用 YAML 工具的错误解析结果替代 GitHub 校验。
- 缩进必须使用空格，不能使用 Tab。
- `if:` 条件通常可以省略 `${{ }}`，但复杂表达式要保持一致并检查上下文是否存在。
- `on: [push, pull_request]` 简洁但不便于添加分支过滤，长期项目建议展开写。
- Shell 中的 `$VAR` 和 Expressions 中的 `${{ env.VAR }}` 不是同一种语法。
- `paths` 过滤可能让必需检查没有运行，需要额外的聚合门禁 Job。

---

## 17. 调试和排错方法

### 17.1 按顺序排查

1. 确认工作流文件在 `.github/workflows/` 且已经提交到触发分支。
2. 确认 `on` 事件、分支过滤和路径过滤匹配当前操作。
3. 查看 Workflow 是否被禁用、是否被组织策略限制。
4. 打开失败 Job，定位第一个失败的 Step。
5. 检查 Runner、运行时版本、依赖锁文件和权限。
6. 对比本地命令与 CI 命令，不要只复现最后一个失败命令。
7. 检查 Secrets 是否只在允许的事件中可用。
8. 检查缓存是否恢复了不兼容的旧数据。

### 17.2 常见问题

| 症状 | 常见原因 | 处理方式 |
|------|----------|----------|
| 推送后没有运行 | 分支、路径或事件过滤不匹配；Workflow 未启用 | 检查 `on`、默认分支和 Actions 设置 |
| `npm ci` 失败 | 锁文件与 `package.json` 不一致 | 本地重新生成锁文件并提交 |
| Secrets 为空 | Fork PR、环境未批准、名称拼写错误 | 检查事件类型、Environment 和 Secret 名称 |
| `Permission denied` | `GITHUB_TOKEN` 权限不足 | 在最小范围 Job 中增加所需权限 |
| Artifact 找不到 | 名称不一致、Job 未成功、保留期已过 | 使用固定命名并检查上传日志 |
| 必需检查卡住 | 路径过滤后没有生成检查 | 增加始终运行的聚合 Job |
| 部署重复执行 | 缺少 concurrency 或触发范围过宽 | 增加并发组和严格 `if` 条件 |
| 本地通过、CI 失败 | 工具版本、时区、大小写、系统差异 | 显式固定环境并打印版本 |
| Action 找不到 | 版本不存在、仓库拼写错误或策略禁止 | 核对 Action 来源和组织允许列表 |
| 生产密钥未生效 | Job 没有声明正确 Environment | 检查 `environment.name` 和审批状态 |

### 17.3 临时打开调试日志

在仓库或组织 Secrets 中临时设置：

- `ACTIONS_RUNNER_DEBUG=true`
- `ACTIONS_STEP_DEBUG=true`

调试完成后应立即关闭，并检查日志是否泄露路径、Token 或内部配置。不要长期保留详细调试日志。

---

## 18. 推荐的项目落地方案

### 18.1 第一阶段：只做 CI

```text
PR -> 安装依赖 -> 格式检查 -> 类型检查 -> 单元测试 -> 构建
```

验收标准：主分支保护要求 CI 通过；失败时能在几分钟内定位到具体 Step。

### 18.2 第二阶段：建立制品

```text
main -> 构建一次 -> 上传 Artifact 或镜像 -> 记录 SHA / Digest
```

验收标准：部署不重新编译，任何上线版本都能找到对应的提交和制品。

### 18.3 第三阶段：预发布部署

```text
main -> CI -> 构建 -> staging -> smoke test
```

验收标准：预发布环境可以重复部署和回滚，测试数据与生产隔离。

### 18.4 第四阶段：生产发布

```text
已验证制品 -> production Environment 审批 -> 部署 -> 健康检查 -> 记录结果
```

验收标准：生产密钥不出现在 PR 流程；发布有审批、审计记录和回滚路径。

---

## 19. 高质量 Workflow 检查清单

### 触发与结构

- [ ] Workflow 放在 `.github/workflows/`。
- [ ] `push` 和 `pull_request` 的分支范围明确。
- [ ] 路径过滤不会导致必需检查缺失。
- [ ] Job 边界清晰，失败能定位到具体步骤。
- [ ] 长耗时 PR 工作流配置了合理的 `concurrency`。

### 可重复性

- [ ] Node、Python、Java、Flutter 等版本显式声明。
- [ ] 使用锁文件和确定性安装命令。
- [ ] 构建依赖版本没有依赖 Runner 的全局状态。
- [ ] 制品包含提交 SHA 或版本标识。
- [ ] 生产部署使用已经验证的同一份制品。

### 安全

- [ ] 默认 `permissions: contents: read`。
- [ ] 每个 Job 只申请必需权限。
- [ ] Secrets 没有写入日志、Artifact、缓存或镜像。
- [ ] Fork PR 不会获得生产 Secrets 或写权限。
- [ ] 未审查的分支名、PR 标题和 Issue 内容没有直接进入 Shell。
- [ ] 高风险第三方 Action 已审查或固定版本。
- [ ] 云部署优先采用 OIDC 和短期身份。

### 发布与运维

- [ ] 生产使用 Environment 审批和分支限制。
- [ ] 有健康检查、失败处理和回滚方案。
- [ ] 测试报告和失败日志可下载，且已脱敏。
- [ ] 有依赖升级和 Actions 升级机制。
- [ ] 定期检查 Runner、Action、运行时和基础镜像的生命周期。

---

## 20. 可直接复制的团队规范

可以把下面的规则加入项目的 `.github/copilot-instructions.md` 或贡献指南：

```markdown
# GitHub Actions 约定

- 所有 Workflow 放在 `.github/workflows/`，命名反映实际用途。
- CI 必须在 Pull Request 和主分支 push 上运行。
- 默认 `permissions` 为 `contents: read`，新增权限必须说明原因。
- 依赖安装必须使用锁文件和 CI 专用的确定性命令。
- 构建一次并上传制品，部署阶段不得重新编译不同版本。
- 生产部署必须使用 GitHub Environment、审批和受保护分支。
- 禁止在日志、Artifact、缓存和镜像中输出密钥。
- 禁止在 `pull_request_target` 中直接执行不可信 PR 的代码。
- 第三方 Action 需要审查来源、权限和维护状态。
- 每次发布必须记录提交 SHA、版本、制品 Digest、目标环境和结果。
- 修改 Workflow 后必须至少运行 YAML 校验、actionlint 或真实测试分支验证。
```

---

## 21. 最终建议

GitHub Actions 的难点不在于记住几十个 YAML 字段，而在于把自动化流程设计成可重复、可审计、可回滚的交付系统：

1. 先用最小 CI 固化构建和测试命令。
2. 再引入缓存和矩阵，解决速度与兼容性问题。
3. 再把构建产物独立出来，保证部署的是已验证版本。
4. 使用 Environment、审批和分支保护控制生产边界。
5. 用最小权限、OIDC、版本固定和 Fork 隔离降低供应链风险。
6. 让日志、Artifact、版本和提交 SHA 形成完整的交付证据链。

一条成熟的流水线不应该只是“自动执行命令”，而应该能回答四个问题：这次发布基于什么代码、经过了哪些检查、谁批准了它、出现问题后怎样恢复。

---

## 参考资源

- GitHub Actions 官方文档：<https://docs.github.com/actions>
- Workflow 语法：<https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions>
- GitHub Actions 安全强化：<https://docs.github.com/actions/security-for-github-actions>
- GitHub OIDC：<https://docs.github.com/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect>
- GitHub Marketplace：<https://github.com/marketplace?type=actions>
