---
name: obsidian
description: Obsidian vault management via CLI. Use when the user asks to create, read, update, delete, search, or organize notes in Obsidian, manage properties/tags/links/backlinks/tasks, query bases, or perform any Obsidian vault operations through the command line.
---

# Obsidian CLI

通过 `obsidian <命令> [选项]` 与 Obsidian 仓库交互。所有命令均支持 `vault=<仓库名>` 指定目标仓库。

## 仓库选择

```
obsidian vaults verbose          # 列出所有仓库及路径
obsidian vault                   # 显示当前仓库信息
obsidian vault info=path         # 仅返回仓库路径
obsidian version                 # 显示 Obsidian 版本
```

默认使用 Obsidian 当前激活的仓库；建议始终显式指定 `vault=<仓库名>`。

## 文件操作

### 创建

```
obsidian vault=<仓库> create name="笔记名" content="内容"
obsidian vault=<仓库> create path="文件夹/笔记.md" content="内容" template="模板名" overwrite open newtab
```

### 读取

```
obsidian vault=<仓库> read file="笔记名"
obsidian vault=<仓库> read path="folder/note.md"
obsidian vault=<仓库> random:read [folder=<文件夹>]    # 读取随机笔记
```

### 编辑

```
obsidian vault=<仓库> append file="笔记名" content="追加内容"          # 追加（自动换行）
obsidian vault=<仓库> append file="笔记名" content="行内内容" inline   # 追加（不换行）
obsidian vault=<仓库> prepend file="笔记名" content="前置内容"         # 前置
```

### 移动 / 重命名 / 删除

```
obsidian vault=<仓库> rename file="旧名" name="新名"
obsidian vault=<仓库> move file="笔记名" to="目标文件夹"
obsidian vault=<仓库> delete file="笔记名"              # 移到回收站
obsidian vault=<仓库> delete file="笔记名" permanent    # 永久删除
```

### 文件信息

```
obsidian vault=<仓库> file file="笔记名"                    # 文件详情（路径、大小、时间戳）
obsidian vault=<仓库> files [folder=<路径>] [ext=<扩展名>] [total]
obsidian vault=<仓库> folders [folder=<路径>] [total]
obsidian vault=<仓库> folder path="文件夹" [info=files|folders|size]
obsidian vault=<仓库> wordcount file="笔记名" [words] [characters]
```

### 打开

```
obsidian vault=<仓库> open file="笔记名" [newtab]
obsidian vault=<仓库> random [folder=<路径>] [newtab]    # 打开随机笔记
obsidian vault=<仓库> recents [total]                    # 列出最近打开的文件
```

## 搜索

```
obsidian vault=<仓库> search query="关键词" [path=<文件夹>] [limit=<n>] [case] [format=text|json] [total]
obsidian vault=<仓库> search:context query="关键词" [path=<文件夹>] [limit=<n>] [case] [format=text|json]
obsidian vault=<仓库> search:open query="关键词"          # 在 Obsidian 中打开搜索面板
```

- `search` 返回匹配的文件列表
- `search:context` 额外返回匹配行的上下文
- `search:open` 在 Obsidian GUI 中打开搜索
- `total` 仅返回匹配数量

## 属性（Frontmatter）

```
obsidian vault=<仓库> properties [file="笔记名"] [total] [counts] [sort=count] [format=yaml|json|tsv]
obsidian vault=<仓库> properties active                       # 当前活动文件的属性
obsidian vault=<仓库> property:read name="属性名" file="笔记名"
obsidian vault=<仓库> property:set name="属性名" value="值" [type=text|list|number|checkbox|date|datetime] file="笔记名"
obsidian vault=<仓库> property:remove name="属性名" file="笔记名"
```

## 标签

```
obsidian vault=<仓库> tags [file="笔记名"] [total] [counts] [sort=count] [format=json|tsv|csv]
obsidian vault=<仓库> tags active           # 当前活动文件的标签
obsidian vault=<仓库> tag name="标签名" [total] [verbose]   # 单个标签详情
```

## 链接与图谱

```
obsidian vault=<仓库> links file="笔记名" [total]                  # 出链
obsidian vault=<仓库> backlinks file="笔记名" [counts] [total] [format=json|tsv|csv]   # 反向链接
obsidian vault=<仓库> orphans [total] [all]                        # 孤儿文件（无入链）
obsidian vault=<仓库> deadends [total] [all]                       # 死端文件（无出链）
obsidian vault=<仓库> unresolved [total] [counts] [verbose] [format=json|tsv|csv]      # 未解析链接
```

## 任务

```
obsidian vault=<仓库> tasks [file="笔记名"] [todo] [done] [status="<字符>"] [verbose] [format=json|tsv|csv] [total]
obsidian vault=<仓库> tasks active              # 当前活动文件的任务
obsidian vault=<仓库> tasks daily               # 日记中的任务
obsidian vault=<仓库> task file="笔记名" line=<行号> toggle
obsidian vault=<仓库> task file="笔记名" line=<行号> done
obsidian vault=<仓库> task file="笔记名" line=<行号> todo
obsidian vault=<仓库> task file="笔记名" line=<行号> status="<字符>"
```

## 大纲

```
obsidian vault=<仓库> outline file="笔记名" [format=tree|md|json] [total]
```

## Bases（数据库插件）

```
obsidian vault=<仓库> bases                                    # 列出所有 base 文件
obsidian vault=<仓库> base:views                               # 列出当前 base 文件的视图
obsidian vault=<仓库> base:query file="base文件" [view=<视图名>] [format=json|csv|tsv|md|paths]
obsidian vault=<仓库> base:create file="base文件" name="新文件名" [content="内容"] [view=<视图名>] [open] [newtab]
```

## 书签与别名

```
obsidian vault=<仓库> bookmarks [total] [verbose] [format=json|tsv|csv]
obsidian vault=<仓库> bookmark file="文件路径" [title="标题"]
obsidian vault=<仓库> bookmark folder="文件夹路径"
obsidian vault=<仓库> bookmark search="搜索查询"
obsidian vault=<仓库> bookmark url="URL" [title="标题"]
obsidian vault=<仓库> aliases [file="笔记名"] [total] [verbose]
obsidian vault=<仓库> aliases active                            # 当前活动文件的别名
```

## 模板

```
obsidian vault=<仓库> templates [total]
obsidian vault=<仓库> template:read name="模板名" [resolve] [title="标题"]
obsidian vault=<仓库> template:insert name="模板名"
```

- `resolve` 解析模板变量（如 `{{title}}`、`{{date}}`）
- `title` 指定变量解析时使用的标题

## 插件与主题

### 插件

```
obsidian vault=<仓库> plugins [filter=core|community] [versions] [format=json|tsv|csv]
obsidian vault=<仓库> plugins:enabled [filter=core|community] [versions] [format=json|tsv|csv]
obsidian vault=<仓库> plugin id="plugin-id"                     # 获取插件详情
obsidian vault=<仓库> plugin:install id="plugin-id" [enable]
obsidian vault=<仓库> plugin:enable id="plugin-id" [filter=core|community]
obsidian vault=<仓库> plugin:disable id="plugin-id" [filter=core|community]
obsidian vault=<仓库> plugin:uninstall id="plugin-id"
obsidian vault=<仓库> plugin:reload id="plugin-id"              # 重载插件（开发者用）
obsidian vault=<仓库> plugins:restrict [on|off]                 # 切换受限模式
```

### 主题

```
obsidian vault=<仓库> themes [versions]
obsidian vault=<仓库> theme [name="主题名"]                     # 获取主题详情
obsidian vault=<仓库> theme:set name="主题名"                   # 设置主题
obsidian vault=<仓库> theme:install name="主题名" [enable]
obsidian vault=<仓库> theme:uninstall name="主题名"
```

### CSS 片段

```
obsidian vault=<仓库> snippets                                 # 列出所有 CSS 片段
obsidian vault=<仓库> snippets:enabled                         # 列出已启用的片段
obsidian vault=<仓库> snippet:enable name="片段名"
obsidian vault=<仓库> snippet:disable name="片段名"
```

## 命令与快捷键

```
obsidian vault=<仓库> commands [filter=<前缀>]                 # 列出可用命令
obsidian vault=<仓库> command id="command-id"                  # 执行指定命令
obsidian vault=<仓库> hotkeys [total] [verbose] [all] [format=json|tsv|csv]
obsidian vault=<仓库> hotkey id="command-id" [verbose]         # 获取单个命令快捷键
```

## 历史版本

```
obsidian vault=<仓库> history file="笔记名"                    # 列出文件历史版本
obsidian vault=<仓库> history:list                             # 列出有历史记录的文件
obsidian vault=<仓库> history:read file="笔记名" version=<n>   # 读取指定版本
obsidian vault=<仓库> history:restore file="笔记名" version=<n> # 恢复到指定版本
obsidian vault=<仓库> history:open file="笔记名"               # 打开文件恢复界面
```

## 标签页与工作区

```
obsidian vault=<仓库> tabs [ids]                               # 列出打开的标签页
obsidian vault=<仓库> tab:open [group=<组ID>] [file=<路径>] [view=<类型>]  # 打开新标签页
obsidian vault=<仓库> workspace [ids]                          # 显示工作区树
```

## 版本对比

```
obsidian vault=<仓库> diff file="笔记名"                       # 列出版本列表
obsidian vault=<仓库> diff file="笔记名" from=<n> to=<n>       # 对比两个版本
obsidian vault=<仓库> diff file="笔记名" filter=local|sync     # 按来源筛选版本
```

## 高级功能

### JavaScript 执行

```
obsidian vault=<仓库> eval code="<JavaScript 代码>"
```

在 Obsidian 上下文中执行任意 JS，返回结果。可用于操作 app、vault、workspace 等 API。

### 开发者工具

```
obsidian devtools                                             # 切换开发者工具
obsidian vault=<仓库> dev:console [clear] [limit=<n>] [level=log|warn|error|info|debug]
obsidian vault=<仓库> dev:errors [clear]
obsidian vault=<仓库> dev:debug [on|off]                      # CDP 调试器
obsidian vault=<仓库> dev:cdp method="<CDP方法>" [params=<JSON>]
obsidian vault=<仓库> dev:dom selector="<CSS选择器>" [total] [text] [inner] [all] [attr=<名>] [css=<属性>]
obsidian vault=<仓库> dev:css selector="<CSS选择器>" [prop=<属性名>]
obsidian vault=<仓库> dev:screenshot [path=<文件名>]
obsidian vault=<仓库> dev:mobile [on|off]                     # 移动端模拟
```

### 应用控制

```
obsidian vault=<仓库> reload                                  # 重载仓库
obsidian restart                                              # 重启 Obsidian
```

## 使用备注

- `file=` 按名称解析（类似 wikilink），`path=` 为精确文件路径
- 大多数命令省略 file/path 时默认操作当前活动文件
- 含空格的值需用引号包裹：`name="我的笔记"`
- content 中使用 `\n` 表示换行，`\t` 表示制表符
- 查看具体命令帮助：`obsidian help <命令>`
