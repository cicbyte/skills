# GitHub Actions 自定义 Action 开发指南

## 目录

1. [Action 三种类型](#1-action-三种类型)
2. [action.yml 元数据文件](#2-actionyml-元数据文件)
3. [JavaScript/TypeScript Action](#3-javascripttypescript-action)
4. [Docker 容器 Action](#4-docker-容器-action)
5. [复合操作 (Composite Action)](#5-复合操作-composite-action)
6. [环境文件](#6-环境文件)
7. [发布到市场](#7-发布到市场)
8. [Action 选择指南](#8-action-选择指南)

---

## 1. Action 三种类型

| 类型 | 运行方式 | 跨平台 | 速度 | 适用场景 |
|------|---------|--------|------|----------|
| JavaScript | 直接在运行器上 | 是 | 快 | 通用逻辑、API 调用、数据转换 |
| Docker 容器 | 在容器中 | 仅 Linux | 慢 | 需要特定环境/依赖 |
| 复合操作 | 合并多个步骤 | 是 | 快 | 简单步骤组合，无需编译 |

---

## 2. action.yml 元数据文件

每个 Action 必须有 `action.yml` 或 `action.yaml` 文件。

### 完整示例

```yaml
name: 'My Action Name'
description: 'What this action does'
author: 'Your Name or Org'
branding:
  icon: 'check-circle'    # 图标名（来自 Feather Icons）
  color: 'green'           # 背景色

inputs:
  my-input:
    description: 'Description of the input'
    required: true
    default: 'default-value'
  optional-input:
    description: 'An optional input'
    required: false
  github-token:
    description: 'GitHub token'
    required: false
    default: ${{ github.token }}

outputs:
  my-output:
    description: 'Description of the output'

runs:
  using: 'node20'          # node16/node20/composite/docker
  main: 'dist/index.js'    # JS 入口

# 仅 Docker 类型需要
# runs:
#   using: 'docker'
#   image: 'Dockerfile'
#   args:
#     - ${{ inputs.my-input }}
```

---

## 3. JavaScript/TypeScript Action

### 项目结构

```
my-action/
├── action.yml
├── package.json
├── tsconfig.json
├── webpack.config.js
├── src/
│   └── index.ts
└── dist/
    └── index.js            # 打包后产物（提交到仓库）
```

### package.json

```json
{
  "name": "my-action",
  "version": "1.0.0",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc && ncc build src/index.ts -o dist",
    "test": "jest",
    "lint": "eslint src/**/*.ts",
    "package": "ncc build src/index.ts -o dist --source-map"
  },
  "dependencies": {
    "@actions/core": "^1.10.0",
    "@actions/github": "^6.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "@vercel/ncc": "^0.38.0",
    "eslint": "^8.0.0",
    "jest": "^29.0.0",
    "ts-jest": "^29.0.0",
    "typescript": "^5.0.0"
  }
}
```

### TypeScript 源码 (src/index.ts)

```typescript
import * as core from '@actions/core';
import * as github from '@actions/github';

async function run(): Promise<void> {
  try {
    // 获取输入
    const myInput = core.getInput('my-input', { required: true });
    const optionalInput = core.getInput('optional-input');

    // 验证输入
    if (!myInput) {
      core.warning('my-input is empty, using default');
    }

    // 分组日志
    core.startGroup('Processing');
    core.info(`Input: ${myInput}`);

    // 处理逻辑...
    const result = processInput(myInput);
    core.endGroup();

    // 设置输出
    core.setOutput('my-output', result);

    // 写入作业摘要
    core.summary
      .addHeading('Action Summary')
      .addTable([
        [{ data: 'Input', header: true }, { data: 'Output', header: true }],
        [{ data: myInput }, { data: result }]
      ])
      .write();

  } catch (error) {
    if (error instanceof Error) {
      core.setFailed(`Action failed: ${error.message}`);
    }
  }
}

function processInput(input: string): string {
  // 实现你的逻辑
  return input.toUpperCase();
}

run();
```

### 关键工具包 API

```typescript
// @actions/core
core.getInput(name, options?)        // 获取输入
core.setOutput(name, value)          // 设置输出
core.setFailed(message)              // 标记步骤失败
core.warning(message)                // 输出警告（不失败）
core.info(message)                   // 输出信息
core.debug(message)                  // 输出调试信息
core.isDebug()                       // 是否启用调试
core.startGroup(name)                // 开始日志分组
core.endGroup()                      // 结束日志分组
core.setSecret(secret)               // 标记为机密（日志中屏蔽）
core.saveState(name, value)          // 保存状态

// @actions/github
github.context                       // 工作流上下文
github.getOctokit(token?)            // 获取 Octokit 客户端
```

### Octokit 客户端配置（重试 + 限流）

```typescript
import * as github from '@actions/github';
import { retry } from '@octokit/plugin-retry';
import { throttling } from '@octokit/plugin-throttling';

const OctokitWithPlugins = github.GitHub.plugin(retry, throttling);
const octokit = new OctokitWithPlugins({
  auth: token,
  retry: { do: true, retryAfter: 30, maxRetries: 5 },
  throttle: {
    onRateLimit: (retryAfter, options) => {
      if (options.request.retryCount === 0) return true;
    },
    onSecondaryRateLimit: (retryAfter, options) => {
      if (options.request.retryCount === 0) return true;
    },
  },
});
```

---

## 4. Docker 容器 Action

### 项目结构

```
my-docker-action/
├── action.yml
├── Dockerfile
└── entrypoint.sh
```

### Dockerfile

```dockerfile
FROM debian:9.5-slim

# 不要使用 USER 指令（Docker action 必须以 root 运行）
# 不要使用 WORKDIR（GitHub 自动设置 GITHUB_WORKSPACE）

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

### entrypoint.sh

```bash
#!/bin/sh -l
# $1 对应 action.yml 中的第一个 arg

echo "Hello $1"
time=$(date)
echo "time=$time" >> $GITHUB_OUTPUT
```

### action.yml

```yaml
name: 'Hello World Docker'
inputs:
  who-to-greet:
    description: 'Who to greet'
    required: true
    default: 'World'
outputs:
  time:
    description: 'The greeting time'
runs:
  using: 'docker'
  image: 'Dockerfile'
  args:
    - ${{ inputs.who-to-greet }}
```

### 注意事项

- 必须以 root 运行（不要使用 USER 指令）
- 建议基于 Debian（与 GitHub 运行器一致）
- `GITHUB_WORKSPACE` 自动映射到 `/github/workspace`
- entrypoint.sh 必须有执行权限
- 仅支持 Linux 运行器

---

## 5. 复合操作 (Composite Action)

### 项目结构

```
.github/actions/my-composite/
└── action.yml
```

### action.yml

```yaml
name: 'My Composite Action'
description: 'Combine multiple steps into one'
inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20.x'
  build-command:
    description: 'Build command to run'
    required: false
    default: 'npm run build'
outputs:
  artifact-path:
    description: 'Path to build artifacts'
    value: ${{ steps.set-output.outputs.path }}

runs:
  using: 'composite'
  steps:
    - uses: actions/checkout@v5

    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - name: Install dependencies
      run: npm ci
      shell: bash            # 必须！

    - name: Build
      id: set-output
      run: |
        ${{ inputs.build-command }}
        echo "path=dist" >> $GITHUB_OUTPUT
      shell: bash            # 必须！

    - name: Save cache
      if: steps.cache.outputs.cache-hit != 'true'
      uses: actions/cache@v4
      with:
        path: ~/.npm
        key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
```

### 复合操作限制

- 每个 `run` 步骤**必须**指定 `shell`
- 不支持 `env` 顶层字段
- 不能创建/使用服务容器
- 不能使用 `uses` 引用同一仓库中的其他复合操作（需用 JS Action）
- 不能使用 `GITHUB_ENV` 在步骤间传递环境变量

---

## 6. 环境文件

### GITHUB_OUTPUT

```bash
# 单行值
echo "key=value" >> $GITHUB_OUTPUT

# 多行值
echo "key<<EOF" >> $GITHUB_OUTPUT
echo "line 1"
echo "line 2"
echo "EOF"
```

### GITHUB_ENV

```bash
# 设置环境变量
echo "MY_VAR=value" >> $GITHUB_ENV

# 追加 PATH
echo "PATH=$PATH:/new/path" >> $GITHUB_ENV

# 多行值
echo "MY_VAR<<EOF" >> $GITHUB_ENV
echo "line 1"
echo "line 2"
echo "EOF"
```

### GITHUB_STEP_SUMMARY

```bash
# Markdown 内容
echo "## Build Results" >> $GITHUB_STEP_SUMMARY
echo "" >> $GITHUB_STEP_SUMMARY
echo "| Metric | Value |" >> $GITHUB_STEP_SUMMARY
echo "|--------|-------|" >> $GITHUB_STEP_SUMMARY
echo "| Time | 5m 30s |" >> $GITHUB_STEP_SUMMARY
```

### GITHUB_PATH

```bash
# 预置 PATH（在所有后续步骤中可用）
echo "/custom/path/bin" >> $GITHUB_PATH
```

---

## 7. 发布到市场

### 语义化版本控制

```
v1.0.0       # 主版本.次版本.补丁版本
v2.1.3       # 补丁：向后兼容的 Bug 修复
v2.2.0       # 次版本：向后兼容的新功能
v3.0.0       # 主版本：不兼容的变更
v1.0.0-beta.1  # 预发布版本
```

### 发布步骤

1. 添加 `branding` 到 `action.yml`
2. 创建 Git 标签：`git tag v1.0.0 && git push --tags`
3. 创建 GitHub Release（自动发布到市场）
4. 建议使用**标签保护规则**防止标签被篡改

### 用户引用方式

```yaml
# 推荐：语义化版本标签
uses: owner/action@v2

# 更安全：提交 SHA
uses: owner/action@abc123sha

# 不推荐：分支名
uses: owner/action@main
```

---

## 8. Action 选择指南

```
需要特定环境/工具链？
├── 是 → Docker 容器 Action（仅 Linux）
└── 否 → 需要复杂逻辑/API 调用？
    ├── 是 → JavaScript/TypeScript Action
    └── 否 → 复合操作（Composite Action）
```

| 场景 | 推荐类型 |
|------|---------|
| 简单的步骤组合 | 复合操作 |
| API 调用、数据转换 | TypeScript Action |
| 需要 Python/Go 等非 JS 运行时 | Docker 容器 Action |
| 需要特定系统依赖 | Docker 容器 Action |
| 需要发布到市场 | TypeScript 或 Docker Action |
| 仅内部使用、快速开发 | 复合操作 |
