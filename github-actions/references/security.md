# GitHub Actions 安全最佳实践

## 目录

1. [最小权限原则](#1-最小权限原则)
2. [脚本注入防护](#2-脚本注入防护)
3. [机密管理](#3-机密管理)
4. [OpenID Connect (OIDC)](#4-openid-connect-oidc)
5. [被入侵运行器缓解](#5-被入侵运行器缓解)
6. [Action 引用安全](#6-action-引用安全)
7. [GITHUB_TOKEN 安全](#7-github_token-安全)
8. [CodeQL 代码扫描](#8-codeql-代码扫描)
9. [跨仓库访问安全优先级](#9-跨仓库访问安全优先级)

---

## 1. 最小权限原则

每个工作流必须显式声明所需的最小权限。不声明 `permissions` 时，GITHUB_TOKEN 获得仓库级别的读写权限。

### 权限配置层级

```yaml
# 工作流级别 — 应用于所有作业
permissions: contents: read

# 作业级别 — 覆盖工作流级别（更精细）
jobs:
  lint:
    permissions: contents: read
  deploy:
    permissions:
      contents: read
      packages: write
      id-token: write
```

### 常见场景权限配置

```yaml
# 普通构建和测试
permissions: contents: read

# 发布到 GitHub Packages
permissions:
  contents: read
  packages: write

# 部署到 GitHub Pages
permissions:
  contents: read
  pages: write
  id-token: write

# CodeQL 安全扫描
permissions:
  contents: read
  security-events: write

# 操作 Issue/PR
permissions:
  contents: read
  issues: write
  pull-requests: write

# 创建 Release
permissions:
  contents: write

# OIDC 认证到云提供商
permissions:
  contents: read
  id-token: write
```

### 快捷设置（仅用于快速原型，生产环境应明确指定）

```yaml
permissions: read-all       # 所有只读
permissions: write-all      # 所有读写（危险）
permissions: {}             # 无权限
```

---

## 2. 脚本注入防护

### 不受信任输入来源

以下 GitHub 上下文属性应视为不受信任输入，**绝不能直接插入 shell 命令**：

- `github.event.issue.title` / `body`
- `github.event.pull_request.title` / `body`
- `github.event.comment.body`
- `github.event.head_commit.message`
- 任何 `body`、`title`、`name`、`label`、`message` 结尾的属性
- 分支名称（如 `zzz";echo${IFS}"hello";#`）

### 错误模式 — 脚本注入

```yaml
# 错误：直接插入 shell（可被注入）
- name: Check PR title
  run: |
    title="${{ github.event.pull_request.title }}"
    if [[ $title =~ ^octocat ]]; then
      echo "OK"
    fi

# 攻击者创建标题: a"; ls $GITHUB_WORKSPACE"
# 实际执行: title="a"; ls $GITHUB_WORKSPACE"
```

### 正确模式 — 通过环境变量传递

```yaml
# 正确：通过环境变量传递（安全）
- name: Check PR title
  run: |
    title="${TITLE}"
    if [[ $title =~ ^octocat ]]; then
      echo "OK"
    fi
  env:
    TITLE: ${{ github.event.pull_request.title }}
```

### 防护原则

1. 不受信任值不能直接流入 `run`、`uses`、API 调用或任何可被解释为代码的地方
2. 使用 `devops-actions/actionlint` 检测脚本注入风险
3. 对不受信任输入进行验证和清理后再使用

---

## 3. 机密管理

### 机密特性

- 使用 Libsodium 密封箱客户端加密（上传前即已加密）
- GitHub Actions 自动屏蔽日志中的机密内容
- 不可手动编辑，必须通过 API 或 UI 设置

### 机密命名规则

- 只能包含 `[a-zA-Z0-9_]`
- 不能以 `GITHUB_` 或数字开头
- 不区分大小写
- 命名约定：大写单词 + 下划线分隔（如 `MY_API_KEY`）

### 机密层级

| 层级 | 特性 |
|------|------|
| 组织级 | 多仓库共享，可配置访问策略 |
| 仓库级 | 仅限单个仓库 |
| 环境级 | 可要求审阅者批准，最安全 |

### 使用方式

```yaml
# 方式 1：作为环境变量
env:
  MY_SECRET: ${{ secrets.MY_SECRET }}

# 方式 2：作为 Action 输入
- uses: some/action@v1
  with:
    token: ${{ secrets.MY_SECRET }}

# 方式 3：可重用工作流传递
jobs:
  call-deploy:
    uses: ./.github/workflows/deploy.yml
    secrets: inherit           # 继承所有机密
    # 或显式传递：
    secrets:
      API_KEY: ${{ secrets.API_KEY }}
```

### 最佳实践

- 尽可能授予最低权限
- 优先使用部署密钥或服务帐户而非个人凭据
- 优先使用 GitHub App 而非 PAT（细粒度权限、短时效令牌、不绑定用户）
- 敏感信息用 `secrets`，非敏感配置用 `variables`

---

## 4. OpenID Connect (OIDC)

替代长期凭据的最佳安全实践。工作流直接从云提供商获取短期令牌，无需存储长期凭据。

### 核心优势

- **无云机密**：无需将云凭据存储为 GitHub 机密
- **精细控制**：可限制到特定仓库、分支、环境
- **自动轮换**：短期令牌仅对单个作业有效

### 配置步骤

```yaml
# 1. 工作流中声明权限
permissions:
  contents: read
  id-token: write    # OIDC 必需

# 2. 使用 OIDC 登录云提供商（以 Azure 为例）
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production    # 限制到特定环境
    steps:
      - name: Login with OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          audience: api://AzureADTokenExchange
```

### OIDC 与传统凭据条件切换

```yaml
- name: Login with OIDC
  uses: azure/login@v2
  if: ${{ inputs.oidc-login == true }}
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

- name: Login with credentials (fallback)
  uses: azure/login@v2
  if: ${{ inputs.oidc-login == false }}
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}
```

### 支持的云提供商

AWS (`aws-actions/configure-aws-credentials`)、Azure (`azure/login`)、GCP (`google-github-actions/auth`)、HashiCorp Vault、PyPI (`pypa/gh-action-pypi-publish`) 等。

---

## 5. 被入侵运行器缓解

### 主要风险

- `printenv` 可读取环境变量中的机密
- 分段输出可绕过自动屏蔽：`echo ${SOME_SECRET:0:4}; echo ${SOME_SECRET:4:200}`
- GITHUB_TOKEN 可在几分之一秒内被窃取
- 如果有写入权限，可修改仓库内容

### 防护措施

1. **绝不**在允许 fork 的公共仓库中使用自托管运行器
2. 使用 `step-security/harden-runner` 监控/阻止出站网络流量
3. 配置 `ACTIONS_RUNNER_HOOK_JOB_COMPLETED` 清理脚本
4. 使用 `--ephemeral` 标记的临时运行器
5. 对自托管运行器实施出站网络限制
6. 使用容器化运行器确保隔离

```yaml
# 使用 harden-runner 限制出站网络
- uses: step-security/harden-runner@v2
  with:
    egress-policy: block
    allowed-endpoints: >
      github.com:443
      api.github.com:443
      *.actions.githubusercontent.com:443
```

---

## 6. Action 引用安全

### 引用规则

```yaml
# 推荐：使用语义化版本标签
uses: actions/checkout@v5

# 更安全：使用提交 SHA（防篡改）
uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11

# 禁止：使用分支名
uses: actions/checkout@main    # 可能随时变更
```

### 可重用工作流引用

```yaml
# 推荐：使用提交 SHA
uses: owner/repo/.github/workflows/ci.yml@abc123sha

# 可接受：版本标签
uses: owner/repo/.github/workflows/ci.yml@v1.0.0
```

---

## 7. GITHUB_TOKEN 安全

### 关键特性

- 每个工作流作业开始时自动创建，作业完成后过期
- GitHub 托管运行器：最长 6 小时
- 自托管运行器：最长可刷新 24 小时
- 权限自动限定为包含工作流的仓库

### 重要行为

- 使用 GITHUB_TOKEN 执行任务时，除 `workflow_dispatch` 和 `repository_dispatch` 外不会创建新的工作流运行（防止递归）
- 即使工作流没有明确传递 GITHUB_TOKEN，操作也可通过 `github.token` 上下文访问
- 应始终通过 `permissions` 键限制权限

### 跨仓库访问优先级

| 优先级 | 方法 | 适用场景 |
|--------|------|----------|
| 1 | GITHUB_TOKEN | 仓库内操作（推荐） |
| 2 | 部署密钥 | 仅 Git 克隆/推送 |
| 3 | GitHub App 令牌 | 跨仓库、细粒度权限 |
| 4 | Fine-grained PAT | 跨仓库，应使用组织账户 |
| 5 | SSH 密钥 | 不推荐，用部署密钥替代 |

---

## 8. CodeQL 代码扫描

### 基础配置

```yaml
name: Code scanning
on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 19 * * 0'    # 每周扫描

permissions:
  contents: read
  security-events: write

jobs:
  CodeQL:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: github/codeql-action/init@v3
        with:
          languages: javascript-typescript
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3
```

### 高级模式（多语言矩阵 + 自定义构建）

```yaml
strategy:
  fail-fast: false
  matrix: ${{ fromJSON(vars.CODEQL_LANGUAGES || '["javascript"]') }}
steps:
  - uses: github/codeql-action/init@v4
    with:
      languages: ${{ matrix.language }}
      build-mode: ${{ matrix.build-mode }}
  - name: Run manual build
    if: matrix.build-mode == 'manual'
    run: make build
  - uses: github/codeql-action/analyze@v4
    with:
      category: "/language:${{ matrix.language }}"
```

---

## 9. 跨仓库访问安全优先级

| 优先级 | 方法 | 说明 |
|--------|------|------|
| 1 | GITHUB_TOKEN | 自动限定作用域为当前仓库，作业完成后过期 |
| 2 | 部署密钥 | 仅授予对单个仓库的读/写访问权限 |
| 3 | GitHub App 令牌 | 可设置细粒度的仓库级别权限 |
| 4 | Fine-grained PAT | 仅向特定仓库授予访问权限 |
| 5 | 个人 SSH 密钥 | 不推荐，使用专用部署密钥替代 |

**关键原则**：绝不使用 personal access token (classic)。
