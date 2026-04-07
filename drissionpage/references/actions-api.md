# 动作链完整API

## 目录
1. [基本使用](#基本使用)
2. [鼠标移动](#鼠标移动)
3. [鼠标点击](#鼠标点击)
4. [滚轮操作](#滚轮操作)
5. [键盘操作](#键盘操作)
6. [拖拽操作](#拖拽操作)
7. [等待操作](#等待操作)
8. [属性](#属性)

## 基本使用

### 使用内置actions属性

```python
tab = Chromium().latest_tab
tab.actions.move_to('#input').click().input('text')
```

### 创建独立对象

```python
from DrissionPage.common import Actions

tab = Chromium().latest_tab
actions = Actions(tab)
actions.move_to('#input').click().input('text')
```

**区别**：内置属性会等待页面加载完成，独立对象不会。

### 操作方式

```python
# 链式操作
tab.actions.move_to(ele).click().type('text')

# 分步操作
tab.actions.move_to(ele)
tab.actions.click()
tab.actions.type('text')
```

## 鼠标移动

### move_to() - 移动到目标

```python
# 移动到元素中点
tab.actions.move_to('#div1')
tab.actions.move_to(ele)

# 移动到绝对坐标
tab.actions.move_to((100, 200))

# 带偏移量（相对元素左上角）
tab.actions.move_to('#div1', offset_x=10, offset_y=20)

# 设置移动时间
tab.actions.move_to('#div1', duration=1)  # 1秒移动
tab.actions.move_to('#div1', duration=0)  # 瞬间移动
```

### move() - 相对移动

```python
# 相对当前位置移动
tab.actions.move(100, 50)   # 右100，下50
tab.actions.move(-50, -30)  # 左50，上30
tab.actions.move(100, 0, duration=0.5)
```

### 方向移动

```python
tab.actions.up(50)     # 向上50像素
tab.actions.down(100)  # 向下100像素
tab.actions.left(30)   # 向左30像素
tab.actions.right(80)  # 向右80像素
```

## 鼠标点击

### 左键点击

```python
# 点击当前位置
tab.actions.click()

# 点击元素
tab.actions.click('#div1')
tab.actions.click(ele)

# 多次点击
tab.actions.click(times=2)  # 双击
```

### 右键点击

```python
tab.actions.r_click()
tab.actions.r_click('#div1')
tab.actions.r_click(times=2)
```

### 中键点击

```python
tab.actions.m_click()
tab.actions.m_click('#div1')
```

### 按住/释放

```python
# 左键
tab.actions.hold()          # 按住当前位置
tab.actions.hold('#div1')   # 按住元素
tab.actions.release()       # 释放
tab.actions.release('#div2') # 在元素上释放

# 右键
tab.actions.r_hold()
tab.actions.r_release()

# 中键
tab.actions.m_hold()
tab.actions.m_release()
```

## 滚轮操作

```python
# 向下滚动
tab.actions.scroll(delta_y=200)

# 向上滚动
tab.actions.scroll(delta_y=-200)

# 水平滚动
tab.actions.scroll(delta_x=100)

# 在元素上滚动
tab.actions.scroll(delta_y=200, on_ele='#div1')
```

## 键盘操作

### 按键按下/抬起

```python
from DrissionPage.common import Keys

tab.actions.key_down(Keys.CTRL)
tab.actions.key_up(Keys.CTRL)

# 或用字符串
tab.actions.key_down('ENTER')
tab.actions.key_up('ENTER')
```

### 输入文本

```python
# input() - 直接输入
tab.actions.input('Hello World')

# type() - 模拟按键输入
tab.actions.type('Hello')

# 输入到元素
tab.actions.click('#input').input('text')
```

### 组合键

```python
from DrissionPage.common import Keys

# 方式1：分开写
tab.actions.key_down(Keys.CTRL).type('a').key_up(Keys.CTRL)

# 方式2：元组
tab.actions.type((Keys.CTRL, 'a'))

# 方式3：内置组合键
tab.actions.type(Keys.CTRL_A)   # 全选
tab.actions.type(Keys.CTRL_C)   # 复制
tab.actions.type(Keys.CTRL_V)   # 粘贴
tab.actions.type(Keys.CTRL_X)   # 剪切
tab.actions.type(Keys.CTRL_Z)   # 撤销
tab.actions.type(Keys.CTRL_Y)   # 重做
```

### 多段输入

```python
# 输入多段文本
tab.actions.type(('ab', 'cd'))

# 组合输入
tab.actions.type((Keys.CTRL_A, 'new text'))
```

## 拖拽操作

### 拖拽元素

```python
# 按住 -> 移动 -> 释放
tab.actions.hold('#div1').right(300).release()

# 拖到另一个元素
tab.actions.hold('#div1').release('#div2')
```

## 拖入文件

```python
# 拖入文件
tab.actions.drag_in('#dropzone', files='C:/file.txt')
tab.actions.drag_in('#dropzone', files=['file1.txt', 'file2.txt'])

# 拖入文本
tab.actions.drag_in('#dropzone', text='some text')
```

## 等待操作

```python
# 固定等待
tab.actions.wait(1)       # 等待1秒

# 随机等待
tab.actions.wait(1, 3)    # 等待1-3秒随机
```

## 属性

```python
# 当前坐标
x = tab.actions.curr_x
y = tab.actions.curr_y

# 所属页面
page = tab.actions.owner
```

## 常用示例

### 复制粘贴

```python
from DrissionPage.common import Keys

# 全选复制
tab.actions.click('#input').type(Keys.CTRL_A).type(Keys.CTRL_C)

# 粘贴
tab.actions.click('#input2').type(Keys.CTRL_V)
```

### 拖拽排序

```python
# 拖拽元素到目标位置
tab.actions.hold('#item1').release('#item2')
```

### 画布绘制

```python
# 模拟绘制
tab.actions.move_to('#canvas').hold().move(100, 0).move(0, 100).move(-100, 0).move(0, -100).release()
```

### 右键菜单

```python
tab.actions.r_click('#element')
tab.actions.wait(0.5)  # 等待菜单出现
tab.actions.click('@text=复制')
```

### 滚动加载更多

```python
for _ in range(5):
    tab.actions.scroll(delta_y=500)
    tab.wait(1)
```
