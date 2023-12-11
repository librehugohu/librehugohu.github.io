---
title: Markdown 语法指南
date: 2023-01-01
tags:
  - Markdown
  - Example
  - Guide
authors:
  - name: Hugo HU
    link: https://github.com/librehugohu
    image: https://github.com/librehugohu.png
excludeSearch: true
math: true
---

这篇文章提供了一些基础的 Markdown 语法样例，这些可以在 Hugo 的内容文件中使用。

<!--more-->

## 基础语法

### 标题

```
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题
```

## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题

```text
*这段文字将是斜体*
_这也将是斜体_

**这段文字将是粗体**
__这也将是粗体__

_你 **可以** 组合它们_
```

*这段文字将是斜体*
_这也将是斜体_

**这段文字将是粗体**
__这也将是粗体__

_你 **可以** 组合它们_

### 列表

#### 无序列表

* 项目 1
* 项目 2
  * 项目 2a
  * 项目 2b

#### 有序列表

1. 项目 1
2. 项目 2
3. 项目 3
   1. 项目 3a
   2. 项目 3b

### 图片

```markdown
![GitHub Logo](https://github.githubassets.com/images/modules/logos_page/GitHub-Mark.png)
```

![GitHub Logo](https://github.githubassets.com/images/modules/logos_page/GitHub-Mark.png)

### 链接

```markdown
[Hugo](https://gohugo.io)
```

[Hugo](https://gohugo.io)

### 块引用

```markdown
牛顿曾说：

> 如果我看得更远，那是因为我站在巨人的肩膀上。
```

> 如果我看得更远，那是因为我站在巨人的肩膀上。

### 行内代码

```markdown
行内 `代码` 有 `反引号` 包围。
```

行内 `代码` 有 `反引号` 包围。

### 代码块

#### 语法高亮

````markdown
```go
func main() {
    fmt.Println("Hello World")
}
```
````

```go
func main() {
    fmt.Println("Hello World")
}
```

### 表格

```markdown
| Syntax    | Description |
| --------- | ----------- |
| Header    | Title       |
| Paragraph | Text        |
```

| Syntax    | Description |
| --------- | ----------- |
| Header    | Title       |
| Paragraph | Text        |

### Latex

$$F(\omega) = \int_{-\infty}^{\infty} f(t) e^{-j\omega t} \, dt$$

### Mermaid

```mermaid
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
```

### Shortcodes

#### Callout

```markdown
{{< callout emoji="🌐" >}}
  Hugo can be used to create a wide variety of websites, including blogs, portfolios, documentation sites, and more.
{{< /callout >}}

{{< callout type="info" >}}
  Please visit GitHub to see the latest releases.
{{< /callout >}}

{{< callout type="warning" >}}
  A **callout** is a short piece of text intended to attract attention.
{{< /callout >}}

{{< callout type="error" >}}
  Something went wrong and it's going to explode.
{{< /callout >}}
```

#### Card

```markdown
{{< cards >}}
  {{< card link="../callout" title="Callout" icon="warning" >}}
  {{< card link="/" title="No Icon" >}}
{{< /cards >}}

{{< cards >}}
  {{< card link="/" title="Image Card" image="https://source.unsplash.com/featured/800x600?landscape" subtitle="Unsplash Landscape" >}}
  {{< card link="/" title="Local Image" image="/images/card-image-unprocessed.jpg" subtitle="Raw image under static directory." >}}
  {{< card link="/" title="Local Image" image="images/space.jpg" subtitle="Image under assets directory, processed by Hugo." method="Resize" options="600x q80 webp" >}}
{{< /cards >}}
```

#### Detail

```markdown
{{% details title="Click me to reveal" closed="true" %}}


This is the content of the details.

Markdown is **supported**.

{{% /details %}}
```

#### FileTree

```markdown
{{< filetree/container >}}
  {{< filetree/folder name="content" >}}
    {{< filetree/file name="_index.md" >}}
    {{< filetree/folder name="docs" state="closed" >}}
      {{< filetree/file name="_index.md" >}}
      {{< filetree/file name="introduction.md" >}}
      {{< filetree/file name="introduction.fr.md" >}}
    {{< /filetree/folder >}}
  {{< /filetree/folder >}}
  {{< filetree/file name="hugo.toml" >}}
{{< /filetree/container >}}
```

#### Step

```markdown
{{% steps %}}

### Step 1

This is the first step.

### Step 2

This is the second step.

{{% /steps %}}
```

#### Tab

```markdown
{{< tabs items="JSON,YAML,TOML" defaultIndex="1" >}}

  {{< tab >}}**JSON**: JavaScript Object Notation (JSON) is a standard text-based format for representing structured data based on JavaScript object syntax.{{< /tab >}}
  {{< tab >}}**YAML**: YAML is a human-readable data serialization language.{{< /tab >}}
  {{< tab >}}**TOML**: TOML aims to be a minimal configuration file format that's easy to read due to obvious semantics.{{< /tab >}}

{{< /tabs >}}
```

## 参考

- [Markdown Syntax](https://www.markdownguide.org/basic-syntax/)
- [Hugo Markdown](https://gohugo.io/content-management/formats/#markdown)