# GitHub Actions 常用模式

## 目录

1. [可重用工作流](#1-可重用工作流)
2. [复合操作 (Composite Actions)](#2-复合操作-composite-actions)
3. [YAML 锚点和别名](#3-yaml-锚点和别名)
4. [条件执行](#4-条件执行)
5. [事件驱动自动化 (IssueOps)](#5-事件驱动自动化-issueops)
6. [环境文件](#6-环境文件)
7. [作业摘要](#7-作业摘要)
8. [GitHub CLI 自动化](#8-github-cli-自动化)
9. [GitHub Script 快速原型](#9-github-script-快速原型)
10. [Dependabot 集成](#10-dependabot-集成)

---

## 1. 可重用工作流

### 被调用方（定义可重用工作流）

```yaml
# .github/workflows/reusable-ci.yml
name: Reusable CI Pipeline
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        required: false
        default: '20.x'
      run-tests:
        type: boolean
        required: false
        default: true
    secrets:
      API_KEY:
        required: false
    outputs:
      artifact-name:
        description: "The name of the uploaded artifact"
        value: ${{ jobs.build.outputs.artifact-name }}

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.upload.outputs.name }}
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - name: Upload
        id: upload
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ github.sha }}
          path: dist/

  test:
    needs: build
    if: ${{ inputs.run-tests }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm test
```

### 调用方（使用可重用工作流）

```yaml
# .github/workflows/main.yml
name: Main CI
on: [push, pull_request]

permissions: contents: read

jobs:
  ci:
    uses: ./.github/workflows/reusable-ci.yml
    with:
      node-version: '22.x'
      run-tests: true
    secrets: inherit              # 继承所有机密
    # 或显式传递：
    # secrets:
    #   API_KEY: ${{ secrets.API_KEY }}
```

### 可重用工作流 vs 复合操作 对比

| 特性 | 可重用工作流 | 复合操作 |
|------|-------------|---------|
| 包含作业 | 是，多个作业 | 否，仅步骤 |
| 日志记录 | 实时记录每个步骤 | 记录为单个步骤 |
| 指定运行器 | 可指定不同运行器 | 继承调用方运行器 |
| 传递机密 | 可使用 | 不可使用 |
| 发布到市场 | 否 | 可发布 |
| 嵌套深度 | 最多 10 层 | 最多嵌套 10 项 |
| 文件位置 | `.github/workflows/` | 单独目录，含 `action.yml` |
| 适用场景 | 跨仓库共享完整流水线 | 封装可复用的步骤组合 |

---

## 2. 复合操作 (Composite Actions)

### 基本结构

```yaml
# .github/actions/build-node/action.yml
name: 'Build Node.js Application'
description: 'Install dependencies and build'
inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20.x'
  build-command:
    description: 'Build command'
    required: false
    default: 'npm run build --if-present'
outputs:
  artifact-path:
    description: 'Path to build output'
    value: ${{ steps.build.outputs.path }}
runs:
  using: 'composite'
  steps:
    - name: Restore Cache
      id: cache
      uses: actions/cache@v4
      with:
        path: ~/.npm
        key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-npm-

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - name: Install dependencies
      run: npm ci
      shell: bash              # 复合操作中每个 run 必须指定 shell

    - name: Build
      id: build
      run: |
        ${{ inputs.build-command }}
        echo "path=dist" >> $GITHUB_OUTPUT
      shell: bash

    - name: Save Cache
      if: steps.cache.outputs.cache-hit != 'true'
      uses: actions/cache@v4
      with:
        path: ~/.npm
        key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
```

### 使用本地复合操作

```yaml
steps:
  - uses: actions/checkout@v5
  - uses: ./.github/actions/build-node
    with:
      node-version: '22.x'
```

### 复合操作注意事项

- 每个 `run` 步骤**必须**指定 `shell: bash`（或 `pwsh`、`sh`）
- 不支持 `env` 顶层字段，需在步骤级别设置
- 不能创建和使用服务容器
- 不能使用 `GITHUB_ENV` 在步骤间传递环境变量（需用 `GITHUB_OUTPUT`）

---

## 3. YAML 锚点和别名

减少工作流中的重复配置：

```yaml
# 定义锚点
common-setup: &common-setup
  run: |
    npm ci
    npm run build

# 使用别名
jobs:
  job-1:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - <<: *common-setup

  job-2:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - <<: *common-setup
```

---

## 4. 条件执行

### 作业条件

```yaml
jobs:
  prod-deploy:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest

  skip-on-fork:
    if: github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
```

### 步骤条件

```yaml
steps:
  - name: Upload on failure
    if: failure()
    uses: actions/upload-artifact@v4
    with:
      name: error-logs
      path: logs/

  - name: Always run
    if: always()
    run: echo "This runs even on failure or cancellation"

  - name: Run only on specific matrix
    if: matrix.experimental == true
    run: npm run test:experimental

  - name: Skip on cancelled
    if: ${{ !cancelled() }}
    run: echo "This skips when workflow is cancelled"

  - name: Detect failure in dependent jobs
    if: ${{ always() && contains(needs.*.result, 'failure') }}
    run: echo "Some dependent job failed"
```

### 状态检查函数

| 函数 | 何时返回 true |
|------|--------------|
| `success()` | 前面所有步骤都成功 |
| `failure()` | 任何前面步骤失败 |
| `always()` | 始终（即使取消） |
| `cancelled()` | 工作流被取消 |

---

## 5. 事件驱动自动化 (IssueOps)

### 评论触发命令

```yaml
name: IssueOps
on:
  issue_comment:
    types: [created]

permissions:
  issues: write

env:
  TRIGGER_WORD: "//deploy"

jobs:
  process-command:
    if: contains(github.event.comment.body, env.TRIGGER_WORD)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        env:
          COMMENT_BODY: ${{ github.event.comment.body }}
        with:
          script: |
            const body = process.env.COMMENT_BODY;
            const command = body.replace('//deploy', '').trim();
            // 处理命令...
```

### 标签触发

```yaml
on:
  issues:
    types: [labeled]

jobs:
  deploy:
    if: github.event.label.name == 'deploy:staging'
    runs-on: ubuntu-latest
    steps:
      - name: Remove label after processing
        run: |
          gh issue edit ${{ github.event.issue.number }} \
            --remove-label "deploy:staging"
        env:
          GH_TOKEN: ${{ github.token }}
```

---

## 6. 环境文件

### GITHUB_OUTPUT — 设置输出参数

```yaml
steps:
  - name: Set output
    id: my-step
    run: |
      echo "version=v1.2.3" >> $GITHUB_OUTPUT
      echo "status=success" >> $GITHUB_OUTPUT

  - name: Use output
    run: echo "Version is ${{ steps.my-step.outputs.version }}"
```

### GITHUB_ENV — 设置后续步骤可用的环境变量

```yaml
steps:
  - name: Set env
    run: |
      echo "MY_VAR=my-value" >> $GITHUB_ENV
      echo "PATH=$PATH:/custom/path" >> $GITHUB_ENV

  - name: Use env
    run: echo "MY_VAR is $MY_VAR"
```

### 多行值

```yaml
- name: Set multiline
  run: |
    echo "MY multiline<<EOF" >> $GITHUB_ENV
    echo "line 1"
    echo "line 2"
    echo "line 3"
    echo "EOF"
```

---

## 7. 作业摘要

在工作流运行页面显示 Markdown/HTML 内容：

```yaml
- name: Generate summary
  run: |
    echo "## Build Summary" >> $GITHUB_STEP_SUMMARY
    echo "" >> $GITHUB_STEP_SUMMARY
    echo "- **Version**: v1.2.3" >> $GITHUB_STEP_SUMMARY
    echo "- **Duration**: 5m 30s" >> $GITHUB_STEP_SUMMARY
    echo "- **Status**: Success" >> $GITHUB_STEP_SUMMARY
```

限制：每步 1 MiB，最多 20 个步骤可写入。

---

## 8. GitHub CLI 自动化

```yaml
env:
  GH_TOKEN: ${{ github.token }}    # 或 ${{ secrets.GITHUB_TOKEN }}

steps:
  - name: Create issue
    run: |
      gh issue create \
        --repo ${{ github.repository }} \
        --title "Automated issue" \
        --body "Created by workflow"

  - name: Comment on PR
    run: |
      gh pr comment ${{ github.event.pull_request.number }} \
        --body "Build succeeded!"

  - name: Create release
    run: |
      gh release create v1.2.3 \
        --generate-notes \
        ./dist/app.tar.gz
```

---

## 9. GitHub Script 快速原型

在步骤中快速使用 GitHub API：

```yaml
- uses: actions/github-script@v7
  id: get-pr-info
  with:
    result-encoding: string
    script: |
      const pr = await github.rest.pulls.get({
        owner: context.repo.owner,
        repo: context.repo.repo,
        pull_number: context.payload.pull_request.number
      });
      return pr.data.title;
```

`actions/github-script@v7` 提供：`github`（预认证 octokit）、`context`、`core`、`glob`、`io`、`exec`。

---

## 10. Dependabot 集成

### 配置文件

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### 自动合并补丁版本

```yaml
name: Auto merge Dependabot
on:
  pull_request_target:
    types: [opened, synchronize, reopened]

permissions:
  contents: write
  pull-requests: write

jobs:
  auto-merge:
    runs-on: ubuntu-latest
    if: github.actor == 'dependabot[bot]'
    steps:
      - uses: actions/checkout@v5
      - uses: dependabot/fetch-metadata@v2
        id: metadata
      - name: Approve patch updates
        if: steps.metadata.outputs.update-type == 'version-update:semver-patch'
        run: gh pr approve ${{ github.event.pull_request.number }}
        env:
          GH_TOKEN: ${{ github.token }}
      - name: Enable auto-merge for patch updates
        if: steps.metadata.outputs.update-type == 'version-update:semver-patch'
        run: gh pr merge ${{ github.event.pull_request.number }} --auto --squash
        env:
          GH_TOKEN: ${{ github.token }}
```
