# GitHub Actions 故障排除与限制

## 目录

1. [常见错误与解决方案](#1-常见错误与解决方案)
2. [调试技巧](#2-调试技巧)
3. [系统限制汇总](#3-系统限制汇总)
4. [运行器规格参考](#4-运行器规格参考)
5. [网络要求](#5-网络要求)

---

## 1. 常见错误与解决方案

### 触发器问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 工作流不运行 | 文件不在 `.github/workflows/` | 移动文件到正确目录 |
| 工作流不运行 | 文件扩展名不是 `.yml`/`.yaml` | 重命名文件 |
| `issues` 事件不触发 | 仅从默认分支运行 | 合并到默认分支 |
| `schedule` 不触发 | 高负载时延迟 | 考虑使用外部 cron 服务 |
| 路径过滤不生效 | 超过 300 个文件差异 | 减少文件差异或移除路径过滤 |
| PR 合并冲突 | 工作流不运行 | 先解决冲突再推送 |
| `workflow_dispatch` 不可用 | 未合并到默认分支 | 先合并到默认分支 |
| fork PR 中工作流不运行 | fork 仓库缺少 `GITHUB_TOKEN` | 使用 `pull_request_target` |

### 权限问题

| 错误信息 | 原因 | 解决方案 |
|---------|------|----------|
| `Resource not accessible` | `GITHUB_TOKEN` 权限不足 | 在 `permissions` 中添加所需权限 |
| `Permission denied` | 自托管运行器文件权限 | 检查 `securityContext.fsGroup` |
| `401 Unauthorized` | Token 过期或无效 | 检查 NTP 时间同步 |

### 缓存问题

| 问题 | 解决方案 |
|------|----------|
| 缓存未命中 | 检查 `hashFiles` 路径是否正确 |
| 缓存过大 | 每仓库上限 10 GB，清理旧缓存 |
| 缓存上传失败 | 超过 200 次/分钟限制 |

### 矩阵问题

| 问题 | 解决方案 |
|------|----------|
| 超过 256 个作业 | 减少 `matrix` 组合或使用 `max-parallel` |
| 一个失败全部取消 | 设置 `fail-fast: false` |

### 自托管运行器问题

| 问题 | 解决方案 |
|------|----------|
| 资源泄漏 | 使用 `--ephemeral` 标记或配置清理钩子 |
| 作业超时 | 自托管上限 5 天/作业，检查 `timeout-minutes` |
| 注册失败 | 检查安装令牌（1 小时有效） |
| 名称过长 | 安装名称最多 45 字符 |

---

## 2. 调试技巧

### 启用调试日志

**方法 1**：设置仓库级变量
- Settings → Secrets and variables → Actions → Variables
- 添加 `ACTIONS_STEP_DEBUG` = `true`
- 添加 `ACTIONS_RUNNER_DEBUG` = `true`

**方法 2**：在 UI 中重新运行并启用调试
- 工作流运行页面 → Re-run all jobs → Enable debug logging

### 工作流命令

```bash
# 写入日志
echo "::debug::Debug message"           # 调试（默认不可见）
echo "::notice::Notice message"         # 通知
echo "::warning::Warning message"       # 警告
echo "::error::Error message"           # 错误

# 带文件注释
echo "::error file=app.js,line=10,title=Lint Error::Missing semicolon"

# 掩码敏感信息
echo "::add-mask::${{ secrets.MY_SECRET }}"

# 设置环境变量
echo "MY_VAR=value" >> $GITHUB_ENV

# 设置输出参数
echo "key=value" >> $GITHUB_OUTPUT

# 写入作业摘要
echo "## Summary" >> $GITHUB_STEP_SUMMARY
echo "- Build: success" >> $GITHUB_STEP_SUMMARY

# 分组日志
echo "::group::My Group"
echo "Grouped content"
echo "::endgroup::"
```

### 下载和分析日志

1. 工作流运行页面 → 下载日志存档
2. 解压后查看 `JOB-NAME/system.txt` 查找条件求值结果
3. 搜索 `Evaluating` 和 `Expanded` 行了解条件是否匹配

### 使用 actionlint 检测问题

```bash
# 安装
brew install actionlint    # macOS
# 或下载二进制文件

# 运行检查
actionlint .github/workflows/*.yml
```

可检测：脚本注入、YAML 语法错误、无效表达式、缺少权限等。

### 本地运行（act）

```bash
# 安装
brew install act

# 运行（使用 Docker）
act -s GITHUB_TOKEN="$(gh auth token)"

# 运行特定工作流
act push -W .github/workflows/ci.yml
```

注意：本地结果可能不是 100% 可靠。

---

## 3. 系统限制汇总

### 运行时限制

| 限制 | 阈值 |
|------|------|
| 工作流运行时间 | 35 天（含等待和审批） |
| 环境审批等待 | 30 天 |
| 作业矩阵 | 256 个作业/运行 |
| 重新运行 | 50 次/运行 |
| 触发事件速率 | 1500 事件/10秒/仓库 |
| 排队运行 | 500 运行/10秒 |

### 作业超时限制

| 运行器类型 | 上限 |
|-----------|------|
| GitHub 托管 | 6 小时/作业 |
| 自托管 | 5 天/作业 |
| `ubuntu-slim` | 15 分钟/作业 |
| 排队等待 | 24 小时 |

### 并发限制（GitHub 托管运行器）

| 计划 | 总并发 | macOS | GPU |
|------|--------|-------|-----|
| Free | 20 | 5 | - |
| Pro | 40 | 5 | - |
| Team | 60 | 5 | - |
| Enterprise | 500 | 50 | - |
| Team (大型) | 1000 | 5 | 100 |
| Enterprise (大型) | 1000 | 50 | 100 |

### 缓存限制

| 限制 | 阈值 |
|------|------|
| 每仓库缓存空间 | 10 GB |
| 未使用自动清理 | 超一周 |
| 缓存上传 | 200 次/分钟/仓库 |
| 缓存下载 | 1500 次/分钟/仓库 |
| 缓存删除 | 400 次/分钟/仓库 |
| 单个缓存大小 | 10 GB |

### API 速率限制

| 类型 | 限制 |
|------|------|
| 未认证 | 60 请求/小时 |
| 认证用户 | 5,000 请求/小时 |
| GITHUB_TOKEN (GHEC) | 15,000 请求/小时/仓库 |
| GITHUB_TOKEN (GHES) | 1,000 请求/小时/仓库 |

### 存储限制（不可提高）

| 计划 | 制品存储 | 免费分钟/月 |
|------|---------|-----------|
| Free | 500 MB | 2,000 |
| Pro | 1 GB | 3,000 |
| Team | 2 GB | 3,000 |
| Enterprise Cloud | 50 GB | 50,000 |

### 制品限制

| 限制 | 阈值 |
|------|------|
| 单个制品大小 | 无硬性限制（建议 < 2 GB） |
| 制品总大小 | 取决于计划存储空间 |
| 保留期 | 最长 90 天 |
| 制品不可变性 | v4+ 不可变（同名需先删除） |

---

## 4. 运行器规格参考

### GitHub 托管运行器规格

| 标签 | OS | CPU | 内存 | 存储 | 架构 |
|------|-----|-----|------|------|------|
| `ubuntu-slim` | Linux 24.04 | 1 | 5 GB | 14 GB | x64 |
| `ubuntu-latest` | Linux 24.04 | 4 | 16 GB | 14 GB | x64 |
| `ubuntu-24.04-arm` | Linux 24.04 | 4 | 16 GB | 14 GB | arm64 |
| `windows-latest` | Windows 2025 | 4 | 16 GB | 14 GB | x64 |
| `windows-11-arm` | Windows 11 ARM | 4 | 16 GB | 14 GB | arm64 |
| `macos-latest` | macOS 15 | 3 (M1) | 7 GB | 14 GB | arm64 |
| `macos-15-intel` | macOS 15 | 4 | 14 GB | 14 GB | Intel |

### 特殊说明

- `ubuntu-slim`：单 CPU 容器，15 分钟超时，不支持挂载文件系统/Docker-in-Docker
- Linux/macOS：无密码 `sudo`
- Windows：禁用 UAC 的管理员权限
- 公共仓库使用标准运行器免费，无分钟限制
- 专用仓库使用 2 核 8GB 规格（Linux/Windows），macOS 不变

---

## 5. 网络要求

### 通信必需域名

```
github.com
api.github.com
*.actions.githubusercontent.com
codeload.github.com               # 下载 Action
*.blob.core.windows.net           # 上传/下载制品和缓存
*.pkg.github.com                  # GitHub Packages
ghcr.io                           # GitHub Container Registry
dependabot-actions.githubapp.com  # Dependabot
```

### 网络故障排除

1. 检查 DNS 解析
2. 检查防火墙规则（出站 HTTPS 443）
3. 检查代理配置
4. 检查 IP 允许列表
5. 检查 TLS 证书链
6. 使用 `curl -v` 或 `GIT_CURL_VERBOSE=1` 调试
