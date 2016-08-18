---
title: Markdown基础语法学习
date: 2016-05-16 09:34:01
tags: markdown
categories: markdown

---
## 优点&emsp;&emsp;

**总结 Markdown 的优点如下**：
- 纯文本，所以兼容性极强，可以用所有文本编辑器打开。
- 让你专注于文字而不是排版。
- 格式转换方便，Markdown 的文本你可以轻松转换为 html、电子书等。
- Markdown 的标记语法有极好的可读性。

## 语法介绍&emsp;&emsp;

### 标题
这是最为常用的格式，在平时常用的的文本编辑器中大多是这样实现的：输入文本、选中文本、设置标题格式。

而在 Markdown 中，你只需要在文本前面加上 `#` 即可，同理、你
还可以增加二级标题、三级标题、四级标题、五级标题和六级标题，总共六级，只需要增加  `#` 即可，标题字号相应降低。例如：
```
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题
```
### 列表
Markdown 支持有序列表和无序列表。

无序列表使用星号、加号或是减号作为列表标记：
```
*   Red
*   Green
*   Blue
```
等同于：
```
+   Red
+   Green
+   Blue
```
也等同于：
```
-   Red
-   Green
-   Blue
```
有序列表则使用数字接着一个英文句点：
```
1.  Bird
2.  McHale
3.  Parish
```
其中数字的数值大小并不影响最后的结果

### 分隔线
你可以在一行中用三个以上的星号、减号、底线来建立一个分隔线，行内不能有其他东西。你也可以在星号或是减号中间插入空格。下面每种写法都可以建立分隔线：
```
* * *

***

*****

- - -

---------------------------------------
```
### 引用
在我们写作的时候经常需要引用他人的文字，这个时候引用这个格式就很有必要了，在 Markdown 中，你只需要在你希望引用的文字前面加上 `>` 就好了，例如：
```
> 引用的文字
```
最终显示的就是：
> 引用的文字

### 粗体和斜体
Markdown 的粗体和斜体也非常简单，用两个 `*` 包含一段文本就是粗体的语法，用一个 `*` 包含一段文本就是斜体的语法。例如：
```
*这是斜体* ， **这是粗体**
```
最终显示的就是：
*这是斜体* ， **这是粗体**
### 表格
相关代码：
```
| Tables        | Are           | Cool  |
| ------------- |:-------------:| -----:|
| col 3 is      | right-aligned | $1600 |
| col 2 is      | centered      |   $12 |
| zebra stripes | are neat      |    $1 |
```
显示效果：

| Tables        | Are           | Cool  |
| ------------- |:-------------:| -----:|
| col 3 is      | right-aligned | $1600 |
| col 2 is      | centered      |   $12 |
| zebra stripes | are neat      |    $1 |
相关代码：
```
dog | bird | cat
----|------|----
foo | foo  | foo
bar | bar  | bar
baz | baz  | baz
```
显示效果：

dog | bird | cat
----|------|----
foo | foo  | foo
bar | bar  | bar
baz | baz  | baz
### 链接
- 行内链接
```
[link text](link.address.here)
```
- 行外链接

方便在文中多个地方引用相同的链接，集中管理，文本内容查看也整洁
```
[link_name][link_id]
[link_id]: http://link.address.here "注释: 要加 http:// 不然会解析为本地路径"
```
### 图片
和链接模式类似，只要在前面添加一个 `!` 叹号即可
```
![inline text](/path/to/img.jpg "Optional title")

![outline text][id]
[id]: url/to/image  "Optional title attribute"
```
### 代码
行内的一小段代码，可以使用反引号（在TAB上面的那位）来将其包起来
```
``There is a back door here.``
```
整段代码可以用三个反引号
### 结语
以上几种格式是比较常用的格式，Markdown 还有其他语法，如想了解和学习更多，可以参考[《Markdown 语法说明 (简体中文版)》](http://wowubuntu.com/markdown/#link)

><font color= Darkorange>如若觉得本文尚可，欢迎转载交流,转载请在正文明显处注明原文地址，谢谢！</font>
