---
name: github-actions
description: GitHub Actions 工作流编写最佳实践指南。TRIGGER when: (1) 用户需要创建或修改 .github/workflows/ 下的 YAML 工作流文件, (2) 用户提到 GitHub Actions、CI/CD pipeline、workflow, (3) 用户需要配置自动构建/测试/部署流程, (4) 用户需要编写 composite action 或 reusable workflow, (5) 用户需要配置 GitHub Actions 安全策略（permissions、OIDC、secrets）, (6) 用户需要配置 GitHub Actions 性能优化（缓存、并发、矩阵）。DO NOT TRIGGER when: 用户讨论的是其他 CI/CD 工具（Jenkins、GitLab CI、CircleCI 等）的配置。
---

# GitHub Actions 工作流编写指南

## 编写工作流的核心流程

1. **确定触发条件** → 选择 `on` 事件类型（push/pull_request/schedule/workflow_dispatch/workflow_call）
2. **设计作业结构** → 用 `needs` 构建 DAG 依赖图，并行化独立任务
3. **配置安全权限** → 在工作流/作业级别声明最小 `permissions`
4. **编写步骤** → checkout → setup → cache → build → test → upload artifacts
5. **添加并发控制** → 用 `concurrency` 防止重复运行或部署冲突
6. **配置部署环境** → 使用 `environment` 保护生产部署

## 工作流文件基本结构

```yaml
name: 工作流名称                    # Actions 选项卡显示的名称
run-name: ${{ github.actor }} 的构建  # 可选，单次运行的名称
on:                                  # 触发器（见下方详细说明）
  push:
    branches: [main]
    paths: ['src/**', '.github/workflows/**']
  pull_request:
    branches: [main]
  workflow_dispatch:                 # 允许手动触发
  schedule:
    - cron: '0 19 * * 0'            # 每周日 UTC 19:00

permissions: contents: read           # 工作流级最小权限（必须）

concurrency:                         # 并发控制
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true           # CI 取消旧运行；CD 设为 false

env:                                 # 工作流级环境变量
  NODE_VERSION: '20.x'
  BUILD_TYPE: release

jobs:
  build:
    runs-on: ubuntu-latest           # 运行器选择
    timeout-minutes: 30              # 作业超时（GitHub 托管上限 6h）
    outputs:                         # 作业输出（跨作业传递）
      version: ${{ steps.version.outputs.tag }}
    steps:
      - name: Checkout
        uses: actions/checkout@v5    # 始终首先检出代码

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'               # 优先使用 setup action 内置缓存

      - name: Install dependencies
        run: npm ci                   # 优先用 ci 而非 install

      - name: Build
        run: npm run build

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        if: ${{ always() }}           # 即使失败也上传
        with:
          name: build-output
          path: dist/
          retention-days: 7           # 自动清理

  deploy:
    needs: build                     # 依赖 build 作业
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production           # 部署环境（触发保护规则）
    permissions:
      contents: read
      id-token: write                # OIDC 必需
    steps:
      - uses: actions/download-artifact@v5
        with:
          name: build-output
      - run: deploy-script.sh
```

## 触发器速查

| 触发器 | 用途 | 注意事项 |
|--------|------|----------|
| `push` | 代码推送时触发 | 可用 `branches`、`paths`、`tags` 过滤 |
| `pull_request` | PR 创建/更新时触发 | `issues`/`schedule` 仅从默认分支运行 |
| `workflow_dispatch` | 手动触发（UI/CLI） | 需先合并到默认分支才可在 UI 使用 |
| `workflow_call` | 被其他工作流调用 | 可重用工作流的触发器 |
| `schedule` | cron 定时触发 | 语法：`分 时 日 月 星期`；高负载时可能延迟 |
| `repository_dispatch` | API 触发 | 需要 `GITHUB_TOKEN` 或 PAT |
| `workflow_run` | 其他工作流完成时触发 | 可监听特定工作流的成功/失败 |

**路径过滤**（限制前 300 个文件差异）：
```yaml
on:
  push:
    paths:
      - 'src/**'
      - '.github/workflows/*.yml'
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

## 运行器选择

| 标签 | OS | CPU | 内存 | 分钟倍率 |
|------|-----|-----|------|----------|
| `ubuntu-latest` | Linux 24.04 | 4 | 16 GB | 1x |
| `windows-latest` | Windows 2025 | 4 | 16 GB | 2x |
| `macos-latest` | macOS 15 (ARM) | 3 (M1) | 7 GB | 10x |
| `ubuntu-24.04-arm` | Linux ARM64 | 4 | 16 GB | 1x |

**最佳实践**：尽量在 Linux 上运行（成本最低），仅在必要时使用 Windows/macOS。

## 权限配置（安全核心）

```yaml
# 方式 1：工作流级别（推荐）
permissions: contents: read

# 方式 2：作业级别（更精细）
jobs:
  deploy:
    permissions:
      contents: read
      packages: write
      id-token: write    # OIDC 必需
```

**常见权限组合**：
- 普通构建：`contents: read`
- 发布到 GitHub Packages：`contents: read` + `packages: write`
- 部署到 GitHub Pages：`contents: read` + `pages: write` + `id-token: write`
- CodeQL 扫描：`contents: read` + `security-events: write`
- 操作 Issue：`contents: read` + `issues: write`

## 缓存策略

```yaml
# 方式 1：setup action 内置缓存（推荐，最简单）
- uses: actions/setup-node@v4
  with:
    cache: 'npm'     # 支持 npm/yarn/pnpm

- uses: actions/setup-python@v5
  with:
    cache: 'pip'     # 自动搜索 requirements.txt/Pipfile.lock/poetry.lock

- uses: actions/setup-java@v4
  with:
    cache: maven     # 支持 maven/gradle

- uses: actions/setup-go@v5
  # 默认启用缓存，搜索 go.sum

# 方式 2：actions/cache 精细控制
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
```

## 矩阵策略

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node-version: [18.x, 20.x, 22.x]
  fail-fast: false               # 不让一个失败取消其他组合
  max-parallel: 4                # 限制并行数

# 排除特定组合
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
    python-version: ["3.9", "3.11", "3.13"]
    exclude:
      - os: macos-latest
        python-version: "3.11"

# 添加额外组合
strategy:
  matrix:
    include:
      - os: ubuntu-latest
        node-version: 22.x
        experimental: true
```

## 表达式和上下文速查

```yaml
# 表达式语法（if 中可省略 ${{ }}）
if: github.ref == 'refs/heads/main'
if: success() && matrix.node-version == '20.x'
if: contains(github.event_name, 'pull_request')
if: always() || failure() || cancelled()

# 常用上下文
${{ github.actor }}              # 触发者
${{ github.event_name }}         # 事件名
${{ github.ref }}                # 分支/tag 引用
${{ github.sha }}                # 提交 SHA
${{ github.repository }}         # 仓库
${{ github.workspace }}          # 工作目录
${{ github.token }}              # GITHUB_TOKEN
${{ steps.step-id.outputs.key }} # 步骤输出
${{ needs.job-id.outputs.key }}  # 作业输出
${{ matrix.os }}                 # 矩阵变量
${{ env.VAR_NAME }}              # 环境变量
${{ secrets.SECRET_NAME }}       # 机密
${{ vars.VAR_NAME }}             # 仓库/组织变量

# 常用函数
contains(str, substr)             startsWith(str, prefix)
endsWith(str, suffix)             format(template, args)
join(array, sep)                  fromJSON(str)
hashFiles(path)                   toJSON(value)
success() / always() / failure() / cancelled()
```

## 服务容器

```yaml
services:
  postgres:
    image: postgres:15
    env:
      POSTGRES_PASSWORD: test
    ports:
      - 5432:5432               # 运行器上运行时需端口映射
    options: >-
      --health-cmd pg_isready   # 始终配置健康检查
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
```

注意：服务容器仅支持 Linux 运行器，组合操作中不能使用。

## 详细参考文档

根据具体需求，查阅以下参考文件获取完整最佳实践：

- **安全相关**：查阅 [security.md](references/security.md) — 权限、OIDC、脚本注入防护、机密管理、被入侵运行器缓解
- **性能优化**：查阅 [performance.md](references/performance.md) — 缓存策略、并发控制、作业并行化、成本优化
- **常用模式**：查阅 [patterns.md](references/patterns.md) — 可重用工作流、复合操作、YAML 锚点、条件执行、事件驱动
- **部署模式**：查阅 [deployment.md](references/deployment.md) — 环境保护、CD 流水线、容器发布、多阶段部署
- **语言模板**：查阅 [language-templates.md](references/language-templates.md) — Node.js/Python/Go/Rust/Java/.NET/Ruby/PowerShell/Swift 完整 CI 模板
- **故障排除**：查阅 [troubleshooting.md](references/troubleshooting.md) — 常见错误、调试技巧、限制汇总
- **自定义 Action**：查阅 [custom-actions.md](references/custom-actions.md) — 三种 Action 类型、元数据文件、环境文件、发布到市场

## 反模式检查清单

编写工作流时，逐项检查以下常见错误：

- [ ] 是否声明了 `permissions`？默认应至少 `contents: read`
- [ ] 是否有不受信任输入直接插入 shell？应通过 `env` 传递
- [ ] 是否使用了 `@master` 或 `@main` 引用 Action？应使用版本标签或 SHA
- [ ] 是否缺少并发控制？部署工作流必须有 `concurrency`
- [ ] CI 工作流是否未设置 `cancel-in-progress: true`？
- [ ] 是否在公共仓库的 fork PR 中使用了自托管运行器？（禁止）
- [ ] 上传 artifact 是否缺少 `if: ${{ always() }}`？
- [ ] 是否使用了 `npm install` 而非 `npm ci`？
- [ ] 是否忘记设置缓存？
- [ ] 服务容器是否缺少健康检查配置？
- [ ] 是否将敏感信息存储在变量而非机密中？
- [ ] 长时间运行的工作流是否设置了 `timeout-minutes`？
