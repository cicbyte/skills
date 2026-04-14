# Obsidian CLI 常用工作流

## 1. 创建结构化笔记

```bash
obsidian vault=<仓库> create name="我的笔记" content="---\ntitle: 我的笔记\ndate: 2026-04-14\ntags: [笔记, 工作]\n---\n\n# 我的笔记\n\n正文内容"
```

## 2. 搜索并阅读

```bash
# 搜索包含关键词的文件
obsidian vault=<仓库> search query="关键词" format=json
# 带上下文搜索（显示匹配行）
obsidian vault=<仓库> search:context query="关键词"
# 读取笔记
obsidian vault=<仓库> read file="笔记名"
```

## 3. 追加内容到已有笔记

```bash
# 追加（自动换行）
obsidian vault=<仓库> append file="日记" content="## 新的一节\n新内容"
# 行内追加（不换行，适合列表项）
obsidian vault=<仓库> append file="清单" content="- 新项目" inline
```

## 4. 管理任务

```bash
# 列出所有未完成任务（含文件名和行号）
obsidian vault=<仓库> tasks todo verbose
# 切换任务状态
obsidian vault=<仓库> task file="项目计划" line=5 toggle
# 标记完成
obsidian vault=<仓库> task file="项目计划" line=5 done
```

## 5. 分析仓库图谱

```bash
# 孤儿笔记（无入链）
obsidian vault=<仓库> orphans
# 死端笔记（无出链）
obsidian vault=<仓库> deadends
# 未解析/损坏的链接
obsidian vault=<仓库> unresolved verbose
# 查看某笔记的反向链接
obsidian vault=<仓库> backlinks file="笔记名" counts
```

## 6. 批量整理

```bash
# 移动笔记到文件夹
obsidian vault=<仓库> move file="笔记名" to="归档/2026"
# 重命名笔记
obsidian vault=<仓库> rename file="旧名" name="新名"
# 列出文件夹内容
obsidian vault=<仓库> files folder="归档"
```

## 7. 查询数据库（Base）

```bash
# 列出所有 base
obsidian vault=<仓库> bases
# 查询 base
obsidian vault=<仓库> base:query file="我的数据库" format=json
# 从 base 创建条目
obsidian vault=<仓库> base:create file="我的数据库" name="新条目" content="字段1: 值1"
```

## 8. 使用模板

```bash
# 列出模板
obsidian vault=<仓库> templates
# 读取模板（解析变量）
obsidian vault=<仓库> template:read name="日记模板" resolve title="今日标题"
# 用模板创建笔记
obsidian vault=<仓库> create name="新日记" template="日记模板"
```

## 9. 管理属性

```bash
# 查看笔记属性
obsidian vault=<仓库> property:read name="status" file="笔记名"
# 设置属性
obsidian vault=<仓库> property:set name="status" value="已完成" file="笔记名"
# 设置列表属性
obsidian vault=<仓库> property:set name="tags" value="标签1,标签2" type=list file="笔记名"
# 删除属性
obsidian vault=<仓库> property:remove name="draft" file="笔记名"
```

## 10. 版本恢复

```bash
# 查看文件历史
obsidian vault=<仓库> history file="笔记名"
# 读取历史版本
obsidian vault=<仓库> history:read file="笔记名" version=1
# 恢复到指定版本
obsidian vault=<仓库> history:restore file="笔记名" version=1
```

## 提示

- 始终指定 `vault=<仓库名>` 避免歧义
- 需要程序化解析输出时使用 `format=json`
- 仅需数量时使用 `total` 标志
- 需要查看匹配行时用 `search:context` 代替 `search`
- 中文笔记名使用 `file=` 加显示名称即可
