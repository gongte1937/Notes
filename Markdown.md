# Markdown 语法参考指南

## 标题 (Headers)

```markdown
# H1 一级标题
## H2 二级标题
### H3 三级标题
#### H4 四级标题
##### H5 五级标题
###### H6 六级标题
```

## 文本样式 (Text Styling)

**粗体文本** 或 __粗体文本__

*斜体文本* 或 _斜体文本_

***粗斜体*** 或 ___粗斜体___

~~删除线文本~~

==高亮文本== (部分编辑器支持)

<u>下划线</u> (使用 HTML 标签)

## 分隔线 (Horizontal Rules)

可以使用三个或更多的星号、减号或下划线：

---

***

___

## 引用 (Blockquotes)

> 这是一级引用
>
> 可以有多行

> 这是一级引用
>> 这是嵌套的二级引用
>>> 这是嵌套的三级引用

## 列表 (Lists)

### 无序列表

* 项目 1
* 项目 2
  * 嵌套项目 2.1
  * 嵌套项目 2.2
    * 更深层嵌套 2.2.1
* 项目 3

也可以使用 `-` 或 `+`：

- 项目 A
- 项目 B

### 有序列表

1. 第一项
2. 第二项
3. 第三项
   1. 嵌套项 3.1
   2. 嵌套项 3.2
4. 第四项

## 链接 (Links)

[这是一个网页链接](https://google.com)

[带标题的链接](https://google.com "Google 首页")

直接显示 URL: <https://google.com>

[引用式链接][ref-id]

[ref-id]: https://example.com "可选的标题"

## 图片 (Images)

![图片替代文本](https://example.com/image.png)

![带标题的图片](https://example.com/image.png "图片标题")

## 代码 (Code)

### 内联代码

这是 `内联代码` 示例，使用反引号包裹。

### 代码块

使用三个反引号创建代码块：

```
npm install
npm start
```

### 带语法高亮的代码块

```javascript
const add = (a, b) => {
    return a + b;
}

console.log(add(2, 3)); // 输出: 5
```

```python
def greet(name):
    return f"Hello, {name}!"

print(greet("World"))
```

```bash
#!/bin/bash
echo "Hello, Shell!"
```

## 表格 (Tables)

| 列1标题 | 列2标题 | 列3标题 |
| ------- | ------- | ------- |
| 行1列1  | 行1列2  | 行1列3  |
| 行2列1  | 行2列2  | 行2列3  |

### 对齐方式

| 左对齐 | 居中对齐 | 右对齐 |
| :--- | :---: | ---: |
| 左   | 中    | 右   |
| 文本 | 文本  | 文本 |

## 任务列表 (Task Lists)

- [x] 已完成任务
- [x] 另一个已完成任务
- [ ] 未完成任务
- [ ] 待办事项

## 脚注 (Footnotes)

这是一个带脚注的文本[^1]。

这是另一个脚注[^note]。

[^1]: 这是第一个脚注的内容。
[^note]: 这是命名脚注的内容。

## 转义字符 (Escaping)

使用反斜杠 `\` 转义特殊字符：

\* 不是列表项

\# 不是标题

\[不是链接\]

## Obsidian 特殊语法

### 内部链接

```markdown
[[文件名]] - 链接到另一个笔记
[[文件名|显示文本]] - 带自定义显示文本的链接
[[文件名#标题]] - 链接到特定标题
```

### 标签

```markdown
#标签名
#嵌套/标签
```

### 数学公式 (LaTeX)

行内公式: $E = mc^2$

公式块:

$$
\frac{-b \pm \sqrt{b^2 - 4ac}}{2a}
$$

## 缩写 (Abbreviations)

HTML 是超文本标记语言。

*[HTML]: Hyper Text Markup Language

## 注释 (Comments)

<!-- 这是一个 HTML 注释，不会在渲染后显示 -->

## 嵌入 HTML

Markdown 支持嵌入 HTML 标签：

<div style="color: red;">
  这是红色文字
</div>

<details>
<summary>点击展开</summary>

这是折叠的内容。

</details>


