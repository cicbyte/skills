# 元素定位语法详解

## 目录
1. [基本定位语法](#基本定位语法)
2. [简化写法](#简化写法)
3. [组合定位](#组合定位)
4. [相对定位](#相对定位)
5. [查找方法](#查找方法)

## 基本定位语法

### 属性定位 `@`

| 写法 | 匹配方式 | 说明 |
|------|----------|------|
| `@attr` | 存在匹配 | 属性存在即匹配 |
| `@attr=value` | 精确匹配 | 属性值完全等于 |
| `@attr:value` | 模糊匹配 | 属性值包含 |
| `@attr^value` | 开头匹配 | 属性值以...开头 |
| `@attr$value` | 结尾匹配 | 属性值以...结尾 |

否定匹配 `@!`：

| 写法 | 说明 |
|------|------|
| `@!attr` | 属性不存在 |
| `@!attr=value` | 属性值不等于 |

### 文本定位 `text`

| 写法 | 匹配方式 | 示例 |
|------|----------|------|
| `text` | 模糊匹配 | `'text登录'` |
| `text=value` | 精确匹配 | `'text=登录'` |
| `text:value` | 模糊匹配 | `'text:登录'` |
| `text^value` | 开头匹配 | `'text^登'` |
| `text$value` | 结尾匹配 | `'text$录'` |

**注意**：组合使用时 `text` 改为 `@text()`

### 标签定位 `tag`

```python
tab.ele('tag:div')      # 查找div元素
tab.ele('tag:input')    # 查找input元素
tab.ele('tag:a')        # 查找a标签

# 组合使用
tab.ele('@tag()=div@class=btn')
```

### XPath 定位 `xpath`

```python
tab.ele('xpath://div[@id="content"]')
tab.ele('xpath://a[contains(text(),"更多")]')
tab.ele('x://div[@class="item"][1]')  # 简化写法
```

### CSS选择器定位 `css`

```python
tab.ele('css=div#content.item')
tab.ele('css=div.class1.class2')
tab.ele('c:div#content')  # 简化写法
```

## 简化写法

| 原写法 | 简化 | 说明 |
|--------|------|------|
| `@id=value` | `#value` | ID选择器 |
| `@class=value` | `.value` | 类选择器 |
| `tag:xxx` | `t:xxx` | 标签 |
| `text=xxx` | `tx=xxx` | 精确文本 |
| `text:xxx` | `tx:xxx` | 模糊文本 |
| `xpath=xxx` | `x:xxx` | XPath |
| `css=xxx` | `c:xxx` | CSS选择器 |

**注意**：`#` 和 `.` 只能单独使用，不能组合

## 组合定位

### AND 关系 `@@`

多个条件同时满足：

```python
# tag为input，type为text，name为user
tab.ele('tag:input@@type=text@@name=user')

# 类包含btn，同时文本包含提交
tab.ele('.btn@@text:提交')

# input且type=text或type=password
tab.ele('tag:input@@type=text@|type=password')
```

### OR 关系 `@|`

满足任一条件：

```python
# class为a或class为b
tab.ele('tag:div@|class=a@|class=b')

# 文本为登录或文本为注册
tab.ele('@|text=登录@|text=注册')
```

### 混合使用

```python
# AND和OR混合
tab.ele('tag:div@@class=container@|class=wrapper')
```

## 相对定位

基于已获取元素进行相对定位：

### 父子关系

```python
ele.parent()           # 父元素
ele.parent(2)          # 第2级父元素
ele.child()            # 第1个子元素
ele.child(2)           # 第2个子元素
ele.children()         # 所有子元素
ele.children('tag:a')  # 所有a标签子元素
```

### 兄弟关系

```python
ele.next()             # 后一个兄弟
ele.next(2)            # 后第2个兄弟
ele.next('tag:div')    # 后面第一个div兄弟
ele.nexts()            # 后面所有兄弟
ele.nexts('tag:div')   # 后面所有div兄弟

ele.prev()             # 前一个兄弟
ele.prev(2)            # 前第2个兄弟
ele.prev('tag:div')    # 前面第一个div兄弟
ele.prevs()            # 前面所有兄弟
ele.prevs('tag:div')   # 前面所有div兄弟
```

### 文档顺序（不限于兄弟）

```python
ele.after()            # 文档中下一个元素
ele.after('tag:div')   # 文档中下一个div
ele.afters()           # 文档中后面所有

ele.before()           # 文档中上一个元素
ele.before('tag:div')  # 文档中上一个div
ele.befores()          # 文档中前面所有
```

## 查找方法

### 单元素 `ele()`

```python
ele = tab.ele('#id')           # 定位符
ele = tab.ele('xpath://div')   # xpath
ele = tab.ele(locator)         # 定位元组

# 超时设置
ele = tab.ele('#id', timeout=5)
```

### 多元素 `eles()`

```python
eles = tab.eles('tag:a')       # 返回列表
eles = tab.eles('.item')       # 返回所有匹配

# 判断是否存在
if tab.eles('#id'):
    print('存在')
```

### 在元素内查找

```python
form = tab.ele('#form')
input = form.ele('tag:input')  # 在form内查找
links = form.eles('tag:a')     # 在form内查找所有a
```

### 简化调用

```python
# 直接用()代替ele()
tab('#id')        # 等同于 tab.ele('#id')
tab('.class')     # 等同于 tab.ele('.class')
ele('#child')     # 等同于 ele.ele('#child')
```
