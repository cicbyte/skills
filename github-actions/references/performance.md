# GitHub Actions 性能优化

## 目录

1. [缓存策略](#1-缓存策略)
2. [并发控制](#2-并发控制)
3. [作业并行化与依赖](#3-作业并行化与依赖)
4. [矩阵优化](#4-矩阵优化)
5. [检出优化](#5-检出优化)
6. [成本优化](#6-成本优化)
7. [制品管理](#7-制品管理)

---

## 1. 缓存策略

GitHub 托管运行器每次在干净映像中启动，缓存是提升速度的关键。

### 方式一：setup action 内置缓存（推荐）

```yaml
# Node.js — 支持 npm/yarn/pnpm
- uses: actions/setup-node@v4
  with:
    node-version: '20.x'
    cache: 'npm'

# Python — 自动搜索 requirements.txt / Pipfile.lock / poetry.lock
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'

# Java/Maven
- uses: actions/setup-java@v4
  with:
    java-version: '17'
    distribution: 'temurin'
    cache: maven

# Java/Gradle — 推荐使用 gradle/actions
- uses: gradle/actions/setup-gradle@v4
  # 默认启用缓存

# Go — 默认启用缓存，搜索 go.sum
- uses: actions/setup-go@v5
  with:
    go-version: '1.22'

# Ruby — 使用 bundler-cache
- uses: ruby/setup-ruby@v1
  with:
    ruby-version: '3.3'
    bundler-cache: true

# .NET
- uses: actions/setup-dotnet@v4
  with:
    dotnet-version: '8.x'
    cache: true
    cache-dependency-path: '**/packages.lock.json'
```

### 方式二：actions/cache 精细控制

```yaml
# 基本缓存
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-

# 多路径缓存
- uses: actions/cache@v4
  with:
    path: |
      ~/.cargo/registry
      ~/.cargo/git
      target
    key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

# 条件缓存保存（仅在缓存未命中时保存）
- uses: actions/cache@v4
  id: cache
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
- name: Install dependencies
  if: steps.cache.outputs.cache-hit != 'true'
  run: npm ci
```

### 缓存限制

- 每个仓库最多 10 GB
- 超一周未使用的缓存自动清理
- 缓存上传：200 次/分钟/仓库
- 缓存下载：1500 次/分钟/仓库

---

## 2. 并发控制

### CI 工作流 — 取消旧运行

```yaml
# 同一分支的新提交取消旧运行
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### CD/部署工作流 — 队列等待

```yaml
# 生产部署永不取消，排队等待
concurrency:
  group: "deploy-production"
  cancel-in-progress: false
```

### 高级模式

```yaml
# 仅取消开发分支，保留发布分支
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ !contains(github.ref, 'release/') }}

# PR 使用 head_ref，其他使用 run_id
concurrency:
  group: ${{ github.head_ref || github.run_id }}
  cancel-in-progress: true
```

---

## 3. 作业并行化与依赖

### DAG 依赖图

```yaml
jobs:
  lint:                     # 独立任务，并行运行
    runs-on: ubuntu-latest
  build:                    # 独立任务，并行运行
    runs-on: ubuntu-latest
  test:                     # 依赖 build
    needs: build
    runs-on: ubuntu-latest
  security-scan:            # 依赖 build
    needs: build
    runs-on: ubuntu-latest
  deploy:                   # 依赖所有前置任务
    needs: [lint, test, security-scan]
    runs-on: ubuntu-latest
```

### 作业间传递数据

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.tag }}
    steps:
      - id: version
        run: echo "tag=v1.2.3" >> $GITHUB_OUTPUT
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v5
        with:
          name: build-output
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```

---

## 4. 矩阵优化

### 基本矩阵

```yaml
strategy:
  matrix:
    node-version: [18.x, 20.x, 22.x]
    os: [ubuntu-latest, windows-latest]
  fail-fast: false        # 不让一个失败取消其他
  max-parallel: 4         # 限制并行数
```

### 排除和包含

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
    python-version: ["3.9", "3.11", "3.13", "pypy3.10"]
    exclude:
      - os: macos-latest
        python-version: "3.11"
      - os: windows-latest
        python-version: "3.11"
    include:
      - os: ubuntu-latest
        python-version: "3.13"
        experimental: true
```

### 条件步骤（仅特定矩阵执行）

```yaml
steps:
  - name: Run experimental tests
    if: matrix.experimental == true
    run: npm run test:experimental
```

### 矩阵限制

- 每次运行最多 256 个作业
- 使用 `max-parallel` 控制并发
- 公共仓库矩阵无分钟限制

---

## 5. 检出优化

```yaml
# 默认（克隆完整历史，最慢）
- uses: actions/checkout@v5

# 浅克隆（仅最新提交，最快）
- uses: actions/checkout@v5
  with:
    fetch-depth: 1

# 需要 Git 历史时（如语义化版本）
- uses: actions/checkout@v5
  with:
    fetch-depth: 0

# LFS 支持
- uses: actions/checkout@v5
  with:
    lfs: true

# 子模块
- uses: actions/checkout@v5
  with:
    submodules: recursive

# 稀疏检出（大幅减少下载量）
- uses: actions/checkout@v5
  with:
    sparse-checkout: |
      src
      package.json
      package-lock.json
    sparse-checkout-cone-mode: true
```

---

## 6. 成本优化

### 分钟倍率

| 运行器 | 倍率 | 每分钟成本 |
|--------|------|-----------|
| Linux | 1x | $0.008 |
| Windows | 2x | $0.016 |
| macOS | 10x | $0.08 |
| ARM Linux | 1x | $0.008 |

### 优化策略

1. **优先 Linux**：仅在必要时使用 Windows/macOS
2. **使用缓存**：减少依赖下载时间
3. **合理使用并发控制**：避免重复运行浪费分钟
4. **公共仓库免费**：标准 GitHub 托管运行器对公共仓库无限制
5. **使用 `ubuntu-slim`**：单 CPU 容器，适合轻量任务
6. **设置 `timeout-minutes`**：避免僵尸作业消耗分钟

### 免费额度

| 计划 | 免费分钟/月 | 存储空间 | 缓存空间 |
|------|-----------|---------|---------|
| Free | 2,000 | 500 MB | 10 GB |
| Pro | 3,000 | 1 GB | 10 GB |
| Team | 3,000 | 2 GB | 10 GB |
| Enterprise | 50,000 | 50 GB | 10 GB |

---

## 7. 制品管理

### 上传和下载

```yaml
# 上传（支持排除模式）
- uses: actions/upload-artifact@v4
  if: ${{ always() }}           # 即使失败也上传
  with:
    name: build-output
    path: |
      dist
      !dist/**/*.md             # 排除 Markdown
    retention-days: 7           # 自动清理（不超过仓库限制）

# 下载单个制品
- uses: actions/download-artifact@v5
  with:
    name: build-output

# 下载所有制品
- uses: actions/download-artifact@v5
```

### 制品 vs 缓存

| 特性 | 缓存 | 制品 |
|------|------|------|
| 用途 | 复用不频繁更改的文件（依赖项） | 保存作业生成的文件（构建产物、日志） |
| 生命周期 | 跨工作流运行共享 | 工作流运行结束后可查看 |
| 典型内容 | npm/pip/Maven 依赖 | 二进制文件、测试结果、代码覆盖率 |
| 不可变性 | 可覆盖 | v4+ 不可变，同名需先删除 |
