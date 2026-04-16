# GitHub Actions 部署模式

## 目录

1. [部署环境](#1-部署环境)
2. [GitHub Pages 部署](#2-github-pages-部署)
3. [容器发布](#3-容器发布)
4. [PyPI 受信任发布](#4-pypi-受信任发布)
5. [多阶段部署流水线](#5-多阶段部署流水线)
6. [OIDC 安全部署](#6-oidc-安全部署)

---

## 1. 部署环境

### 环境功能

- **审批机制**：要求审阅者批准后作业才能继续
- **分支限制**：限制哪些分支可触发部署
- **等待计时器**：部署前等待指定时间（最长 30 天）
- **独立机密**：环境级机密，审批后才可用
- **独立变量**：环境级变量
- **部署历史**：记录每次部署

### 基本配置

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging           # 关联 staging 环境
    steps:
      - run: deploy-to-staging.sh

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production             # 关联 production 环境
      url: ${{ steps.deploy.outputs.url }}  # 部署 URL 显示在 UI
    steps:
      - id: deploy
        run: |
          echo "url=https://app.example.com" >> $GITHUB_OUTPUT
          deploy-to-production.sh
```

### 环境配置（在 GitHub UI 中设置）

1. Settings → Environments → New environment
2. 配置保护规则：Required reviewers、Wait timer、Deployment branches
3. 配置环境机密和变量

---

## 2. GitHub Pages 部署

```yaml
name: Deploy to Pages
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write        # Pages 部署必需

concurrency:
  group: "pages"
  cancel-in-progress: false    # Pages 部署永不取消

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v4
        with:
          node-version: '20.x'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

---

## 3. 容器发布

### 推送到 GitHub Container Registry (ghcr.io)

```yaml
name: Build and Push Docker Image
on:
  push:
    branches: [main]
    tags: ['v*']

permissions:
  contents: read
  packages: write

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha

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

### Dockerfile 最佳实践

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci              # 先 COPY 再 RUN，避免存储凭据
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

---

## 4. PyPI 受信任发布

```yaml
name: Publish to PyPI
on:
  release:
    types: [created]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-python@v5
        with:
          python-version: '3.x'
      - name: Build
        run: |
          pip install build
          python -m build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  publish:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      id-token: write        # OIDC 必需（无需 API token）
    environment:
      name: pypi
    steps:
      - uses: actions/download-artifact@v5
        with:
          name: dist
          path: dist/
      - uses: pypa/gh-action-pypi-publish@release/v1
        # 无需 API 令牌，使用 OIDC 受信任发布
```

---

## 5. 多阶段部署流水线

### 完整 CI/CD 流水线

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main]

permissions: contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false

jobs:
  # 阶段 1：并行检查
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - run: npm ci
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v4
        with:
          cache: 'npm'
      - run: npm ci
      - run: npm test

  security-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v5
      - uses: github/codeql-action/init@v4
        with:
          languages: javascript-typescript
      - uses: github/codeql-action/analyze@v4

  # 阶段 2：构建
  build:
    needs: [lint, test, security-scan]
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.tag }}
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v4
        with:
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - id: version
        run: echo "tag=$(npm pkg get version)" >> $GITHUB_OUTPUT
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  # 阶段 3：部署到 staging
  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/download-artifact@v5
        with:
          name: build-output

  # 阶段 4：部署到 production（需审批）
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/download-artifact@v5
        with:
          name: build-output
      - run: deploy-to-production.sh
```

### DAG 依赖图

```
lint ─────────────────────┐
test ─────────────────────┤
security-scan ────────────┤
                          ├─→ build ──→ deploy-staging ──→ deploy-production
```

---

## 6. OIDC 安全部署

### Azure 部署（OIDC）

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          audience: api://AzureADTokenExchange

      - name: Deploy to Azure
        run: az containerapp up ...
```

### AWS 部署（OIDC）

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActions
          aws-region: us-east-1

      - name: Deploy
        run: |
          aws ecs update-service --cluster my-cluster --service my-service
```

### GCP 部署（OIDC）

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123/locations/global/workloadIdentityPools/my-pool/providers/my-provider
          service_account: my-sa@project.iam.gserviceaccount.com

      - uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: my-service
          image: gcr.io/project/image:tag
```
