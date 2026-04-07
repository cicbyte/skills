# package 模块详细文档

```
!描述：package模块用于获取各类元素（网页元素、软件元素及图像元素）及对资源文件进行读写等操作
```

## 详情

package模块内部一共由三类对象组成：元素选择器（selector）、图像选择器（image_selector）和资源文件对象（resources）

## selector()

根据元素名称返回元素选择器对象。元素选择器指的是在元素库中捕获的网页元素或软件元素。可以通过`def selector(name)`方法来获取指定元素。

**def selector(name)**

### 输入参数

**name**：选择器名称，即元素库中元素的名称。

### 返回值

**Selector**：元素选择器对象，即通过元素名称所指定的元素

### 示例1

```python
from .package import *
def main(args):
    # 这里"百度一下"指的是元素库中名为"百度一下"的网页元素
    selector = package.selector('百度一下')
    print(selector)
```

**该示例执行逻辑：** 获取名为"百度一下"的网页元素--> 打印输出该元素对象

## image_selector()

根据图像库中图像元素的名称返回图像选择器对象

**def image_selector(name)**

### 参数

**name**：图像选择器名称，即图像库中图像元素的名称

### 返回值

**ImageSelector**：图像选择器对象，即通过图像元素名称所指定的元素

### 示例1

```python
from .package import *
def main(args):
    selector = package.image_selector('图像1') #这里"图像1"指的是图像库中名称为"图像1"的图像元素
```

**该示例执行逻辑：** 获取名为"图像1"的图像元素--> 打印输出该图像元素

## resources

资源文件对象。通过调用`package.resources`获取资源文件对象以便对资源文件进行读写操作。

### 方法

#### get_text()

获取资源文件的文本内容

**def get_text(self, filename, encoding='utf-8')**

##### 输入参数

**filename**：资源文件名

**encoding**：读取文件时的编码格式，若不传入，则默认以 `utf-8` 的格式读取

##### 返回值

**str**：返回资源文件的文本内容

##### 示例1

以 utf-8 的编码格式读取资源文件 123.txt 的文本内容

```python
from . import package
def main(args):
    text = package.resources.get_text('123.txt')
    print(text)
```

**该示例执行逻辑：** 读取资源文件名为 "123.txt" 的文件内容--> 打印输出该内容

#### get_bytes()

获取资源文件的二进制信息

**get_bytes(self, filename)**

##### 输入参数

**filename**：资源文件名

##### 返回值

**byte**：返回资源文件的二进制信息

##### 示例1

```python
from . import package
def main(args):
    value = package.resources.get_bytes('123.txt') #输出结果将以二进制的形式返回
    print(value)
```

**该示例执行逻辑：** 读取资源文件名为 "123.txt" 的二进制文件内容--> 打印输出该二进制内容

#### copy_to()

将资源文件的内容拷贝到目标文件中

**copy_to(self, filename, dest_filename)**

##### 输入参数

**filename**：资源文件名

**dest_filename**：资源文件内容最终拷贝到的位置（为文件绝对路径，如 "D:\123.txt"）

##### 返回值

无

##### 示例1

```python
from . import package
def main(args):
    package.resources.copy_to('123.txt', 'D:\\aaa.txt')
```

**该示例执行逻辑：** 将名为 "123.txt" 的资源文件拷贝到本地D盘并重命名为 "aaa.txt"

#### add_to_clipboard()

将资源文件添加到剪切板

**add_to_clipboard(self, filenames)**

##### 输入参数

**filenames**：资源文件名列表（一定要注意！）, 如['123.txt', '234.xml']

##### 返回值

无

##### 示例1

```python
from . import package
def main(args):
    package.resources.add_to_clipboard(['123.txt', '234.xml'])
```

**该示例执行逻辑：** 将名为 "123.txt" 、 "234.xml" 的资源文件拷贝到剪切板中
