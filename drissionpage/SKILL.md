---
name: drissionpage
description: DrissionPage 4.x Python 浏览器自动化框架指南。使用场景：(1) 使用 DrissionPage 库编写浏览器自动化脚本 (2) 元素定位、页面交互、网络监听 (3) 表单填写、文件上传下载、截图录像 (4) 标签页管理、iframe 操作、动作链 (5) 代码中包含 Chromium、ChromiumOptions、SessionPage、WebPage 等导入。触发关键词：DrissionPage、浏览器自动化、Chromium、元素定位、页面交互、爬虫、网页测试
---

# DrissionPage 4.x 浏览器自动化指南

基于 Chromium 的 Python 浏览器自动化框架，融合浏览器控制和数据包监听功能。

## 核心架构

```
Chromium (浏览器对象)
    └── Tab (标签页对象)
            ├── ele() / eles() (元素查找)
            ├── actions (动作链)
            ├── listen (网络监听)
            └── wait (智能等待)
```

## 快速开始

### 基本使用

```python
from DrissionPage import Chromium

browser = Chromium()          # 连接/启动浏览器
tab = browser.latest_tab      # 获取标签页
tab.get('https://example.com')  # 访问网页

# 元素查找与交互
tab.ele('#username').input('user')
tab.ele('#password').input('pass')
tab.ele('@type=submit').click()
```

### 启动配置

```python
from DrissionPage import Chromium, ChromiumOptions

co = ChromiumOptions()
co.headless()           # 无头模式
co.incognito()          # 无痕模式
co.no_imgs(True)        # 不加载图片
co.mute(True)           # 静音
co.set_proxy('http://127.0.0.1:7890')  # 代理
co.auto_port(True)      # 自动分配端口（多进程安全）

browser = Chromium(co)
```

## 元素定位语法

### 基本定位符

| 语法 | 说明 | 示例 |
|------|------|------|
| `#id` | ID选择器 | `tab.ele('#kw')` |
| `.class` | 类选择器 | `tab.ele('.btn')` |
| `tag:div` | 标签 | `tab.ele('tag:input')` |
| `text=文本` | 精确文本 | `tab.ele('text=登录')` |
| `text:文本` | 模糊文本 | `tab.ele('text:登录')` |
| `@attr=value` | 属性 | `tab.ele('@type=submit')` |
| `xpath=//div` | XPath | `tab.ele('xpath://div[@id="x"]')` |
| `css=div#id` | CSS选择器 | `tab.ele('css=div.class')` |

### 简化写法

| 原写法 | 简化 | 示例 |
|--------|------|------|
| `@id=value` | `#value` | `tab.ele('#username')` |
| `@class=value` | `.value` | `tab.ele('.btn-primary')` |
| `tag:div` | `t:div` | `tab.ele('t:a')` |
| `text=xxx` | `tx=xxx` | `tab.ele('tx=提交')` |
| `xpath=xxx` | `x:xxx` | `tab.ele('x://div')` |

### 组合定位

```python
# 多属性同时匹配
tab.ele('tag:input@@type=text@@name=user')  # AND关系
tab.ele('tag:div@|class=a@|class=b')        # OR关系

# 链式查找
tab.ele('#form').ele('tag:input')           # 内部查找

# 相对定位
ele.next()          # 下一个兄弟
ele.prev()          # 上一个兄弟
ele.parent()        # 父元素
ele.child()         # 第一个子元素
ele.children()      # 所有子元素
```

### 批量获取

```python
eles = tab.eles('tag:a')          # 获取所有链接
eles.get.texts()                  # 获取所有文本列表
eles.get.attrs('href')            # 获取所有href属性
eles.get.links()                  # 获取所有链接
```

## 元素交互

### 点击操作

```python
ele.click()              # 左键点击（智能处理遮挡）
ele.click(by_js=True)    # JS点击（无视遮挡）
ele.click(by_js=None)    # 优先模拟，被遮挡则JS
ele.click.right()        # 右键点击
ele.click.middle()       # 中键点击
ele.click.multi(2)       # 双击
ele.click.at(50, -50)    # 带偏移点击（相对左上角）
```

### 输入操作

```python
ele.input('文本')                # 输入文本
ele.input('文本\n')              # 输入并回车
ele.input('文本', clear=True)    # 清空后输入
ele.clear()                      # 清空内容

# 组合键
from DrissionPage.common import Keys
ele.input(Keys.CTRL_A)           # 全选
ele.input((Keys.CTRL, 'a'))      # Ctrl+A
ele.input((Keys.CTRL, 'v'))      # 粘贴
```

### 其他交互

```python
ele.hover()              # 悬停
ele.drag(100, 50)        # 拖拽（偏移量）
ele.drag_to(target)      # 拖拽到目标元素
ele.focus()              # 获取焦点
ele.check()              # 选中checkbox
ele.check(uncheck=True)  # 取消选中

# 下拉选择
select = tab.ele('t:select')
select.select.by_text('选项文本')
select.select.by_value('value')
select.select.by_index(1)  # 从1开始
```

## 页面操作

### 导航

```python
tab.get(url)             # 访问网址
tab.back()               # 后退
tab.forward()            # 前进
tab.refresh()            # 刷新
tab.refresh(ignore_cache=True)  # 强制刷新
tab.stop_loading()       # 停止加载
```

### 窗口管理

```python
tab.set.window.max()     # 最大化
tab.set.window.mini()    # 最小化
tab.set.window.full()    # 全屏
tab.set.window.size(800, 600)  # 设置大小
tab.set.window.location(100, 100)  # 设置位置
```

### 滚动

```python
tab.scroll.down(200)     # 向下滚动
tab.scroll.up(100)       # 向上滚动
tab.scroll.to_top()      # 滚到顶部
tab.scroll.to_bottom()   # 滚到底部
tab.scroll.to_see(ele)   # 滚动到元素可见
```

### 执行脚本

```python
tab.run_js('alert(1)')                    # 执行JS
tab.run_js('return document.title')       # 返回结果
tab.run_js('arguments[0].click()', ele)   # 传入元素
ele.run_js('this.style.color="red"')      # 元素执行JS
```

## 智能等待

### 页面等待

```python
tab.wait.load_start()       # 等待开始加载
tab.wait.doc_loaded()       # 等待文档加载完成
tab.wait.eles_loaded('#id') # 等待元素加载
tab.wait.ele_displayed('#id')  # 等待元素显示
tab.wait.ele_hidden('#id')  # 等待元素隐藏
tab.wait.url_change('keyword')  # 等待URL变化
tab.wait.title_change('keyword') # 等待标题变化
tab.wait(1)                 # 强制等待1秒
```

### 元素等待

```python
ele.wait.displayed()        # 等待显示
ele.wait.hidden()           # 等待隐藏
ele.wait.deleted()          # 等待删除
ele.wait.clickable()        # 等待可点击
ele.wait.not_covered()      # 等待不被遮挡
ele.wait.stop_moving()      # 等待停止移动
```

## 动作链

模拟鼠标键盘操作，支持链式调用。

```python
from DrissionPage.common import Keys

# 鼠标操作
tab.actions.move_to('#input').click().input('文本')
tab.actions.hold('#ele').right(100).release()  # 拖拽
tab.actions.scroll(delta_y=200)  # 滚轮

# 键盘操作
tab.actions.key_down(Keys.CTRL).type('a').key_up(Keys.CTRL)
tab.actions.type(Keys.CTRL_A)  # 组合键简写

# 方向移动
tab.actions.up(50).down(50).left(50).right(50)
```

## 网络监听

抓取浏览器数据包，适用于动态加载内容。

```python
# 等待并获取
tab.listen.start('api/data')  # 开始监听
tab.ele('@text=下一页').click()
res = tab.listen.wait()       # 等待数据包
print(res.response.body)      # 获取响应体
tab.listen.stop()             # 停止监听

# 实时获取
tab.listen.start('api/data')
for packet in tab.listen.steps(count=5):
    print(packet.url)
    # 执行操作...

# 过滤设置
tab.listen.start(targets='api/', method='GET', is_regex=False)
```

## 标签页管理

```python
# 获取标签页
tab = browser.latest_tab           # 最新激活的
tab = browser.get_tab(index=0)     # 按索引
tab = browser.get_tab(title='xxx') # 按标题

# 创建和关闭
new_tab = browser.new_tab(url)     # 新建并访问
tab.close()                        # 关闭当前
tab.close(others=True)             # 关闭其他

# 切换
browser.latest_tab.set.activate()  # 激活标签页
```

## iframe 操作

```python
# 进入iframe
frame = tab.get_frame('#iframe-id')
frame.ele('#button').click()

# 简化写法
tab('#iframe-id').ele('#button').click()
```

## 文件操作

### 上传

```python
# 方式1：直接输入路径
tab.ele('type=file').input(r'C:\file.txt')

# 方式2：点击触发
tab.set.upload_files(r'C:\file.txt')
tab.ele('@type=button').click()
tab.wait.upload_paths_inputted()

# 多文件
tab.ele('type=file').input(r'C:\file1.txt\nC:\file2.txt')
```

### 下载

```python
# 浏览器下载
tab.set.download_path(r'D:\downloads')
ele.click.to_download(save_path=r'D:\file.zip')

# download方法（requests封装）
result = tab.download(url, save_path)
mission = tab.download.add(url, save_path)  # 并发下载
mission.wait()  # 等待完成
```

## 截图和录像

```python
# 截图
tab.get_screenshot(path='screenshot.png')
ele.get_screenshot(path='ele.png')

# 录像
tab.screencast.start(path='video.webm')
# ... 操作 ...
tab.screencast.stop()
```

## 获取信息

### 页面信息

```python
tab.url           # URL
tab.title         # 标题
tab.html          # HTML源码
tab.cookies()     # 获取cookies
```

### 元素信息

```python
ele.tag           # 标签名
ele.text          # 文本（已格式化）
ele.raw_text      # 原始文本
ele.html          # outerHTML
ele.inner_html    # innerHTML
ele.attrs         # 所有属性字典
ele.attr('href')  # 某个属性
ele.value         # value值
ele.link          # href或src

# 位置大小
ele.rect.size           # 大小 (width, height)
ele.rect.location       # 页面坐标
ele.rect.viewport_location  # 视口坐标

# 状态
ele.states.is_displayed    # 是否显示
ele.states.is_enabled      # 是否可用
ele.states.is_checked      # 是否选中
ele.states.is_in_viewport  # 是否在视口
```

## 异常处理

```python
from DrissionPage.errors import ElementNotFoundError, WaitTimeoutError

try:
    ele = tab.ele('#notexist', timeout=2)
except ElementNotFoundError:
    print('元素未找到')

try:
    tab.wait.ele_displayed('#ele', timeout=5)
except WaitTimeoutError:
    print('等待超时')
```

## 最佳实践

1. **优先使用智能等待**而非 `time.sleep()`
2. **链式操作**提高代码简洁性：`tab.ele('#id').click()`
3. **相对定位**处理动态元素：`ele.next()`
4. **网络监听**获取动态数据，比解析DOM更可靠
5. **by_js点击**处理被遮挡元素
6. **auto_port**避免多实例端口冲突

## 详细参考

更多功能详见：
- [元素定位详细语法](references/locator-syntax.md)
- [等待机制详解](references/wait-mechanism.md)
- [动作链完整API](references/actions-api.md)
