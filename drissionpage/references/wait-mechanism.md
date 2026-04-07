# 等待机制详解

## 目录
1. [等待概述](#等待概述)
2. [页面等待方法](#页面等待方法)
3. [元素等待方法](#元素等待方法)
4. [浏览器等待方法](#浏览器等待方法)
5. [通用等待方法](#通用等待方法)

## 等待概述

DrissionPage 内置智能等待机制，避免使用 `time.sleep()` 的硬编码等待。

所有等待方法都有 `timeout` 和 `raise_err` 参数：
- `timeout`: 超时时间（秒），默认使用页面设置
- `raise_err`: 超时是否抛异常，默认根据 `Settings` 设置

## 页面等待方法

### 加载等待

```python
# 等待页面开始加载
tab.wait.load_start()
tab.wait.load_start(timeout=5)

# 等待文档加载完成
tab.wait.doc_loaded()
```

### 元素等待

```python
# 等待元素加载到DOM
tab.wait.eles_loaded('#div1')
tab.wait.eles_loaded(['#div1', '#div2'])  # 多个
tab.wait.eles_loaded('#div1', any_one=True)  # 任一个即可

# 等待元素显示
tab.wait.ele_displayed('#div1')
tab.wait.ele_displayed(ele)  # 传入元素对象

# 等待元素隐藏
tab.wait.ele_hidden('#div1')

# 等待元素被删除
tab.wait.ele_deleted('#div1')
```

### URL/标题等待

```python
# 等待URL变化
tab.wait.url_change('keyword')      # URL包含keyword
tab.wait.url_change('keyword', exclude=True)  # URL不含keyword

# 等待标题变化
tab.wait.title_change('keyword')
tab.wait.title_change('keyword', exclude=True)
```

### 下载等待

```python
# 等待下载开始
tab.wait.download_begin()
tab.wait.download_begin(cancel_it=True)  # 取消该下载

# 等待所有下载完成
tab.wait.downloads_done()
tab.wait.downloads_done(timeout=60, cancel_failed=True)
```

### 弹窗等待

```python
# 等待alert关闭
tab.wait.alert_closed()
```

### 文件上传等待

```python
tab.set.upload_files('demo.txt')
btn.click()
tab.wait.upload_paths_inputted()  # 等待路径填入
```

## 元素等待方法

通过元素的 `wait` 属性调用：

```python
ele = tab.ele('#button')
```

### 显示/隐藏等待

```python
# 等待显示
ele.wait.displayed()
ele.wait.displayed(timeout=3)

# 等待隐藏
ele.wait.hidden()
```

### 删除等待

```python
# 等待从DOM删除
ele.wait.deleted()
```

### 尺寸等待

```python
# 等待元素有大小
ele.wait.has_rect()
```

### 覆盖等待

```python
# 等待被覆盖
ele.wait.covered()

# 等待不被覆盖
ele.wait.not_covered()
```

### 可用状态等待

```python
# 等待可用
ele.wait.enabled()

# 等待不可用
ele.wait.disabled()

# 等待消失或不可用
ele.wait.disappeared()
```

### 运动等待

```python
# 等待停止运动
ele.wait.stop_moving()
ele.wait.stop_moving(gap=0.1)  # 检测间隔

# 等待可点击（包含停止运动）
ele.wait.clickable()
ele.wait.clickable(wait_moved=False)
```

## 浏览器等待方法

通过 `browser.wait` 调用：

```python
from DrissionPage import Chromium
browser = Chromium()

# 等待新标签页
browser.wait.new_tab()
browser.wait.new_tab(curr_tab=tab)  # 指定当前tab

# 等待下载开始
browser.wait.download_begin()

# 等待所有下载完成
browser.wait.downloads_done()
```

## 通用等待方法

所有对象都支持的通用等待：

```python
# 固定等待
browser.wait(1)        # 等待1秒
tab.wait(0.5)          # 等待0.5秒
ele.wait(2)            # 等待2秒

# 随机等待
tab.wait(1, 3)         # 等待1-3秒随机值
tab.wait(0.5, 2.5)     # 等待0.5-2.5秒随机值
```

## 超时设置

### 全局设置

```python
# ChromiumOptions
co.set_timeouts(base=10, page_load=30, script=30)

# 运行时设置
tab.set.timeouts(base=10)
browser.set.timeouts(base=10)
```

### 单次设置

```python
tab.wait.ele_displayed('#id', timeout=5)
tab.ele('#id', timeout=5)  # 查找元素超时
```

## 错误处理

```python
from DrissionPage.errors import WaitTimeoutError

# 返回False或抛异常
result = tab.wait.ele_displayed('#id', raise_err=False)  # 返回False
tab.wait.ele_displayed('#id', raise_err=True)   # 抛WaitTimeoutError
```

## 最佳实践

1. **点击前等待可点击**
```python
ele.wait.clickable()
ele.click()
```

2. **页面跳转后等待加载**
```python
ele.click()
tab.wait.load_start()
tab.wait.doc_loaded()
```

3. **动态内容等待元素出现**
```python
tab.wait.eles_loaded('#result', timeout=10)
```

4. **避免硬编码sleep**
```python
# 不推荐
time.sleep(2)
ele.click()

# 推荐
ele.wait.clickable()
ele.click()
```
