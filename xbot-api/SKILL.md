---
name: xbot-api
description: 影刀RPA平台的Python自动化开发技能。当用户需要开发影刀xbot自动化脚本、使用xbot API进行网页/桌面/移动端自动化、Excel/Word操作、数据库操作、AI集成、PDF处理等功能时使用此技能。
---

# 影刀XBot API 开发技能

## 概述

影刀XBot是一个企业级RPA（机器人流程自动化）平台，提供Python API用于开发自动化脚本。此技能涵盖了xbot的核心API使用方法、代码模板和最佳实践。

**何时使用此技能：**
- 开发影刀xbot自动化流程脚本
- 使用xbot API进行网页自动化（xbot.web）
- Windows桌面应用自动化（xbot.win32）
- 移动端自动化（xbot.mobile）
- Excel/Word文档操作（xbot.excel、xbot.word）
- 数据库操作（xbot.ado）
- AI能力集成（xbot.ai - 阿里云/百度/腾讯）
- PDF文件处理（xbot.pdf）
- FTP文件传输（xbot.ftp）

## 代码模板规范

### 基础模板结构

影刀xbot模块必须遵循以下格式：

```python
# 使用提醒:
# 1. xbot包提供软件自动化、数据表格、Excel、日志、AI等功能
# 2. package包提供访问当前应用数据的功能，如获取元素、访问全局变量、获取资源文件等功能
# 3. 当此模块作为流程独立运行时执行main函数
# 4. 可视化流程中可以通过"调用模块"的指令使用此模块

import xbot
from xbot import print, sleep
from .import package
from .package import variables as glv

def main(args):
    # 在此编写自动化逻辑
    pass
```

**重要约束：**
- ❌ 不使用 `if __name__ == "__main__":`
- ✅ 必须使用 `def main(args):` 作为入口函数
- ✅ 必须导入 `xbot` 和 `package` 模块

## 核心API模块

### 1. package - 元素与资源管理

#### 元素选择器
```python
# 获取元素库中的元素
selector = package.selector('元素名称')

# 获取图像库中的图像
image_selector = package.image_selector('图像名称')
```

#### 资源文件操作
```python
# 读取文本资源
text = package.resources.get_text('data.txt', encoding='utf-8')

# 读取二进制资源
data = package.resources.get_bytes('image.png')

# 复制资源文件到本地
package.resources.copy_to('template.xlsx', 'D:\\output.xlsx')

# 添加资源文件到剪切板
package.resources.add_to_clipboard(['file1.txt', 'file2.xml'])
```

### 2. xbot.web - 网页自动化

#### 浏览器操作
```python
# 创建并打开网页
browser = xbot.web.create('https://example.com', mode='cef')

# 获取已打开的网页
browser = xbot.web.get(title='百度一下', url='baidu.com')
browser = xbot.web.get_active()  # 获取当前激活的网页

# 获取所有匹配的网页
browsers = xbot.web.get_all(url='example.com')

# 关闭所有网页
xbot.web.close_all()
```

#### 网页导航
```python
# 跳转
browser.navigate('https://newurl.com')

# 后退/前进
browser.go_back()
browser.go_forward()

# 刷新
browser.reload(ignore_cache=True)

# 获取信息
url = browser.get_url()
title = browser.get_title()
html = browser.get_html()
text = browser.get_text()
```

#### 元素操作
```python
# 查找元素
element = browser.find(package.selector('登录按钮'))
elements = browser.find_all_by_css('input[type="text"]')

# 元素交互
element.click(button='left', simulative=True)
element.input('username', simulative=True, append=False)
element.hover()

# 获取元素信息
text = element.get_text()
value = element.get_value()
attr = element.get_attribute('href')

# 下拉框操作
element.select('选项内容', mode='fuzzy')
element.select_by_index(0)
```

#### Cookie管理
```python
# 获取Cookie
cookies = xbot.web.get_cookies(mode='cef', domain='.example.com')

# 设置Cookie
xbot.web.set_cookie(
    url='https://example.com',
    name='session_id',
    value='abc123',
    expires=3600
)

# 移除Cookie
xbot.web.remove_cookie('https://example.com', 'session_id')
```

#### 对话框处理
```python
# 处理JavaScript对话框
browser.handle_javascript_dialog(dialog_result='ok', text='输入内容')

# 处理下载对话框
xbot.web.handle_save_dialog(
    file_folder='D:\\downloads',
    file_name='report.pdf'
)

# 处理上传对话框
xbot.web.handle_upload_dialog('D:\\files\\upload.pdf')
```

### 3. xbot.win32 - Windows桌面自动化

#### 窗口操作
```python
# 获取窗口
window = xbot.win32.get(title='记事本', class_name='Notepad')
window = xbot.win32.get_by_selector(package.selector('窗口元素'))

# 窗口控制
window.activate()  # 激活
window.set_state('maximize')  # 最大化/最小化/正常
window.move(x=100, y=100)  # 移动
window.resize(width=800, height=600)  # 调整大小
window.close()  # 关闭

# 等待窗口状态
window.wait_active(timeout=20)
window.wait_exists(timeout=20)
window.wait_close(timeout=20)
```

#### 元素操作
```python
# 查找元素
element = window.find(package.selector('按钮元素'))
elements = window.find_all(package.selector('列表项'))

# 元素交互
element.click(button='left', simulative=True)
element.input('输入内容')
element.check(mode='check')  # 复选框
element.select('选项')  # 下拉框
element.hover()

# 拖拽
element.drag_to(simulative=True, top=50, left=100, delay_after=1)
```

#### 鼠标键盘操作
```python
# 鼠标操作
xbot.win32.mouse_move(500, 300, move_speed='fast')
xbot.win32.mouse_click(button='left', click_type='click')
xbot.win32.mouse_wheel(wheel_direction='down', wheel_times=3)

# 键盘操作
xbot.win32.send_keys('Ctrl+C')
xbot.win32.send_keys('Hello World')

# 获取鼠标位置
pos = xbot.win32.get_mouse_position(relative_to='screen')
```

#### 图像识别
```python
# 等待图像出现
xbot.win32.Image.wait_appear(
    image_selectors=[package.image_selector('确认按钮')],
    timeout=20
)

# 点击图像
xbot.win32.Image.click(
    image_selectors=[package.image_selector('图标')],
    button='left'
)

# 图像悬停
xbot.win32.Image.hover([package.image_selector('菜单')])
```

#### 剪切板操作
```python
clipboard = xbot.win32.Clipboard

# 设置/获取文本
clipboard.set_text('复制的内容')
text = clipboard.get_text()

# 清空剪切板
clipboard.clear()

# 设置文件
clipboard.set_file(['D:\\file1.txt', 'D:\\file2.pdf'])
```

#### 截图操作
```python
# 屏幕截图
xbot.win32.Screenshot.save_screen_to_file(
    'D:\\screenshot.png',
    image_format='png'
)
xbot.win32.Screenshot.save_screen_to_clipboard()

# 窗口截图
xbot.win32.Screenshot.save_window_to_file(
    hwnd=window.handle,
    image_path='D:\\window.png',
    image_format='png'
)
```

### 4. xbot.excel - Excel操作

#### 工作簿操作
```python
# 创建Excel
workbook = xbot.excel.create(kind='office', visible=True)

# 打开Excel
workbook = xbot.excel.open('D:\\data.xlsx', kind='office', visible=True)

# 获取活动工作簿
workbook = xbot.excel.get_active_workbook(kind='office')

# 保存/另存
workbook.save()
workbook.save_as('D:\\output.xlsx')

# 关闭
workbook.close()
```

#### 工作表操作
```python
# 获取工作表
sheet = workbook.get_active_sheet()
sheet = workbook.get_sheet_by_name('Sheet1')
sheet = workbook.get_sheet_by_index(1)

# 创建工作表
workbook.create_sheet('新工作表', create_way='last')

# 激活工作表
workbook.active_sheet_by_name('Sheet1')

# 删除/复制/重命名
workbook.delete_sheet('旧工作表')
workbook.copy_sheet('源工作表', '目标工作表')
workbook.rename_sheet('旧名称', '新名称')
```

#### 单元格操作
```python
# 读取单元格
value = sheet.get_cell(1, 'A')  # 行号,列名
row = sheet.get_row(1)  # 获取整行
column = sheet.get_column('A')  # 获取整列
range_data = sheet.get_range(1, 'A', 5, 'C')  # 区域数据

# 写入单元格
sheet.set_cell(1, 'A', '标题')
sheet.set_row(1, ['A', 'B', 'C'], begin_column_name='A')
sheet.set_column('A', [1, 2, 3, 4], begin_row_num=1)
sheet.set_range(1, 'A', [[1, 2], [3, 4]])  # 二维数组

# 追加/插入行
sheet.append_row(['新', '数', '据'])
sheet.insert_row(3, ['插入', '数据'])

# 删除行列
sheet.remove_row(5)
sheet.remove_column('C')

# 清空
sheet.clear()
sheet.clear_range(1, 'A', 10, 'Z')
```

#### 其他功能
```python
# 获取行列数
row_count = sheet.get_row_count()
col_count = sheet.get_column_count()

# 获取第一个空闲行/列
free_row = sheet.get_first_free_row()
free_col = sheet.get_first_free_column()

# 选中区域
sheet.select_range(1, 'A', 10, 'C')

# 复制粘贴
sheet.copy_range(1, 'A', 5, 'C')
sheet.paste_range(1, 'D')
```

### 5. xbot.word - Word操作

#### 文档操作
```python
# 创建Word
doc = xbot.word.create(kind='office', visible=True)

# 打开Word
doc = xbot.word.open(
    'D:\\document.docx',
    kind='office',
    visible=True,
    open_password='',
    edit_password=''
)

# 保存/另存
doc.save()
doc.save_as('D:\\output.docx')

# 导出PDF
doc.export_to_pdf(
    'D:\\output.pdf',
    export_range=0,  # 0=全部, 1=当前页, 2=指定范围
    page_from=1,
    page_to=5
)

# 关闭
doc.close()
```

#### 内容操作
```python
# 获取选择区域
selection = doc.get_selection()

# 写入文本
selection.write_text('Hello World', newline=True)

# 读取文本
text = selection.read_text()

# 插入图片
selection.insert_picture(
    image_source='local',  # local/file_url/clipboard
    image_path='D:\\image.png',
    image_scale=100,  # 缩放百分比
    newline=True
)

# 插入超链接
doc.insert_hyperlink('点击这里', 'https://example.com')

# 插入表格
doc.insert_table(
    table_data=[['A', 'B'], ['1', '2']],
    table_data_format=None,
    grid=True,
    newline=True
)

# 替换文本
selection.replace_text(
    search_text='旧文本',
    replace_text='新文本',
    replace_all=True,
    case_sensitive=False
)

# 移动光标
selection.move_cursor(
    move_direction='right',  # left/right/up/down
    move_count=10,
    press_shift=False
)

# 定位光标
selection.locate_cursor_to_document('end_document')  # start_document/end_document
selection.locate_cursor_to_text('关键词')
doc.locate_cursor_to_bookmark('书签名')
```

### 6. xbot.mobile - 移动端自动化

#### 连接设备
```python
# Appium模式连接
mobile = xbot.mobile.connect(
    appium_url='http://127.0.0.1:4723/wd/hub',
    platform_name='Android',
    platform_version='11',
    device_name=' emulator-5554'
)

# 自定义名称连接
mobile = xbot.mobile.connect_by_custom_name('我的手机')

# 关闭连接
mobile.close()
```

#### 元素操作
```python
# 查找元素
element = mobile.find(package.selector('登录按钮'))
elements = mobile.find_all_by_id('com.app:id/button')
elements = mobile.find_all_by_xpath('//android.widget.Button')

# 元素交互
element.click()
element.input('输入内容')
element.longpress()
element.get_text()
element.get_attribute('text')

# 截图
element.screenshot('D:\\element.png', 'element_screenshot')
```

#### 屏幕操作
```python
# 点击/双击/长按
mobile.click(500, 300)
mobile.dblclick(500, 300)
mobile.longpress(500, 300)

# 滑动
mobile.swipe(
    start_point_x=100, start_point_y=500,
    end_point_x=100, end_point_y=100,
    swipe_time=800
)

# 系统操作
mobile.back()  # 返回
mobile.home()  # 主页
mobile.switchapp()  # 切换应用

# 屏幕方向
orientation = mobile.getoriention()
mobile.setoriention(0)  # 0=横屏, 1=竖屏

# 截图
mobile.screenshot('D:\\screen.png', filename='screenshot')
```

#### 剪切板操作
```python
# 设置/获取剪切板
mobile.set_clipboard_text('复制内容')
text = mobile.get_clipboard_text()
```

### 7. xbot.pdf - PDF操作

```python
# 提取文本
text = xbot.pdf.extract_text(
    path='D:\\document.pdf',
    from_page=1,
    to_page=10,
    password=None
)

# 提取图片
xbot.pdf.extract_images(
    path='D:\\document.pdf',
    from_page=1,
    to_page=5,
    save_to_dir='D:\\images',
    name_prefix='page'
)

# 提取页面
xbot.pdf.extract_pages(
    path='D:\\source.pdf',
    from_page=1,
    to_page=3,
    save_to='D:\\extracted.pdf'
)

# 合并PDF
xbot.pdf.merge_pdfs(
    paths=['file1.pdf', 'file2.pdf'],
    save_to='D:\\merged.pdf',
    passwords=None
)
```

### 8. xbot.ado - 数据库操作

```python
# 连接数据库
db = xbot.ado.Database
db.open('Driver={SQL Server};Server=localhost;Database=test;')

# 执行SQL
db.exec('SELECT * FROM users', timeout_seconds=20)
db.exec('UPDATE users SET name="test" WHERE id=1')

# 关闭连接
db.close()
```

### 9. xbot.ai - AI能力集成

```python
# 阿里云OCR
result = xbot.ai.AliyunAI.ocr_from_file('D:\\image.png')

# 百度OCR
result = xbot.ai.BaiduAI.ocr_from_file('D:\\image.png')

# 腾讯OCR
result = xbot.ai.TencentAI.ocr_from_file('D:\\image.png')
```

### 10. xbot.ftp - FTP操作

```python
# 连接FTP
ftp = xbot.ftp.connect(
    host='ftp.example.com',
    username='user',
    password='pass'
)

# 下载文件
ftp.download_file('D:\\local\\file.txt', 'remote/file.txt')
ftp.download_folder('D:\\local', 'remote/folder')

# 上传文件
ftp.upload_file('D:\\local\\file.txt', 'remote/')
ftp.upload_folder('D:\\local', 'remote/')

# 获取文件列表
files = ftp.get_ftp_files('remote/path')

# 断开连接
ftp.unlogin()
```

### 11. xbot.app - 数据表格与对话框

#### 数据表格操作
```python
# 获取数据表格
databook = xbot.app.databook

# 读取数据
row = databook.get_row(1)  # 获取指定行
cell = databook.get_cell(1, 'A')  # 获取单元格
range_data = databook.get_range(1, 'A', 10, 'C')  # 获取区域
row_count = databook.get_row_count()  # 获取行数

# 写入数据
databook.set_cell(1, 'A', '值')
databook.set_row(1, ['A', 'B', 'C'])
databook.set_column('A', [1, 2, 3])
databook.set_range(1, 'A', [[1, 2], [3, 4]])

# 追加/插入/删除行
databook.append_row(['新', '行'])
databook.insert_row(3, ['插入', '行'])
databook.remove_row(5)

# 清空
databook.clear()
```

#### 对话框操作
```python
# 消息对话框
xbot.app.dialog.show_alert('操作完成', title='提示')

# 确认对话框
result = xbot.app.dialog.show_confirm('确定要删除吗？', title='确认')

# 消息框
xbot.app.dialog.show_message_box(
    title='提示',
    message='操作成功',
    button='ok',
    timeout=5
)

# 数据表格对话框
xbot.app.dialog.show_workbook_dialog('数据确认', '请确认以下数据')

# 通知框
xbot.app.dialog.show_notifycation(
    message='任务完成',
    placement='rightbottom',
    level='info',
    timeout=3
)
```

### 12. xbot.xzip - 压缩解压

```python
# 压缩
xbot.xzip.zip(
    file_folder_path='D:\\source',
    zip_file_path='D:\\archive.zip',
    compress_level=5,  # 1-9
    password='secret'  # 可选
)

# 解压
xbot.xzip.unzip(
    zip_file_path='D:\\archive.zip',
    unzip_dir_path='D:\\extracted',
    password='secret'  # 可选
)
```

## 通用功能

### 日志输出
```python
from xbot import print, sleep

# 输出日志
print('普通消息')
print('重要信息')

# 延迟
sleep(3)  # 延迟3秒
```

### 全局变量
```python
from .package import variables as glv

# 设置全局变量
glv.set('username', 'admin')

# 获取全局变量
username = glv.get('username')
```

## 常见使用场景

### 场景1：网页表单自动填写
```python
import xbot
from xbot import print, sleep
from .import package
from .package import variables as glv

def main(args):
    # 打开网页
    browser = xbot.web.create('https://example.com/form')

    # 填写表单
    browser.find(package.selector('用户名')).input('admin')
    browser.find(package.selector('密码')).input('password123')
    browser.find(package.selector('登录按钮')).click()

    # 等待跳转
    sleep(2)
    print('登录成功')
```

### 场景2：Excel数据批量处理
```python
import xbot
from xbot import print, sleep
from .import package
from .package import variables as glv

def main(args):
    # 打开Excel
    workbook = xbot.excel.open('D:\\data.xlsx')
    sheet = workbook.get_active_sheet()

    # 读取数据
    row_count = sheet.get_row_count()

    # 处理每一行
    for i in range(1, row_count + 1):
        name = sheet.get_cell(i, 'A')
        # 处理逻辑...
        sheet.set_cell(i, 'C', '已处理')

    # 保存并关闭
    workbook.save()
    workbook.close()
```

### 场景3：Windows软件自动化
```python
import xbot
from xbot import print, sleep
from .import package
from .package import variables as glv

def main(args):
    # 获取记事本窗口
    window = xbot.win32.get(title='无标题 - 记事本')
    window.wait_active(timeout=10)

    # 输入文本
    window.find(package.selector('编辑区')).input('Hello XBot!')

    # 保存文件
    xbot.win32.send_keys('^s')  # Ctrl+S
    sleep(1)

    # 输入文件名
    window.find(package.selector('文件名输入框')).input('D:\\test.txt')
    window.find(package.selector('保存按钮')).click()
```

### 场景4：PDF文档处理
```python
import xbot
from xbot import print, sleep
from .import package
from .package import variables as glv

def main(args):
    # 提取PDF文本
    text = xbot.pdf.extract_text(
        path='D:\\document.pdf',
        from_page=1,
        to_page=5
    )

    # 处理文本
    print(f'提取文本长度: {len(text)}')

    # 提取图片
    xbot.pdf.extract_images(
        path='D:\\document.pdf',
        from_page=1,
        to_page=5,
        save_to_dir='D:\\images'
    )
```

## 资源说明

### references/
- `api_summary.md` - 完整API速查手册
- `package.md` - package模块详细文档

### assets/
- `default_module_template.py` - 影刀xbot标准模块模板（不含`if __name__ == "__main__"`）

## 注意事项

1. **入口函数规范**：必须使用`def main(args):`，不能使用`if __name__ == "__main__":`
2. **模块导入**：必须导入`xbot`、`print`、`sleep`、`package`和`glv`
3. **元素选择**：使用`package.selector()`获取元素库中的元素
4. **图像识别**：使用`package.image_selector()`获取图像库中的图像
5. **异常处理**：建议添加适当的异常处理逻辑
6. **资源释放**：使用完毕后记得关闭工作簿/浏览器等资源
