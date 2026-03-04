---
title: "主题全功能深度测试指南"
date: 2026-03-03T10:00:00+08:00
lastmod: 2026-03-03T10:00:00+08:00
draft: false
description: "这是一篇包含 Hugo 原生高级功能与 主题所有核心 Shortcode 的极限测试文章，用于排查渲染问题和配置遗漏。"
tags: ["Hugo", "Shortcode", "Markdown"]
categories: ["技术测试", "前端构建"]
authors: ["Admin"]

# Blowfish 页面级显示控制
showAuthor: true
showToc: true
showBreadcrumbs: true
showPostNavLinks: true
showRelated: true
showReadingTime: true
showWordCount: true
showEdit: false
showPagination: true
invertPagination: false

# 头图配置 (Hero)
heroStyle: "background" # 可选项: basic, background, text, thumbAndText
# feature: "images/your-test-image.jpg" # 请替换为真实路径以测试头图

# 高级功能开关
math: true
mermaid: true
chart: true
---

{{< lead >}}
这是 **Lead** 组件。它通常用于文章开头的执行摘要 (Executive Summary)，在视觉上会比正文有更大的字号和更明显的行距，用于快速抓住读者眼球。
{{< /lead >}}

## 1. 复杂 Markdown 语法测试

除了基础的**粗体**、*斜体*和~~删除线~~，这里测试更复杂的 Markdown 边缘情况。

### 1.1 嵌套与混合列表

* 第一层无序列表
  * 第二层无序列表
    1. 第三层有序列表 A
    2. 第三层有序列表 B
       * 第四层无序列表（测试缩进渲染）
* 包含段落的列表项：
  
  这是同一列表项中的另一个段落。只需保持缩进即可。

### 1.2 任务列表与定义列表

- [x] 配置 Hugo 基础环境
- [x] 安装 Blowfish 主题
- [ ] 调整 `hugo.toml` 参数
- [ ] 部署到生产环境

HTML 定义列表测试：
<dl>
  <dt>Hugo</dt>
  <dd>世界上最快的构建网站的框架之一。</dd>
  <dt>Blowfish</dt>
  <dd>一个高度可定制的、基于 Tailwind CSS 的 Hugo 主题。</dd>
</dl>

### 1.3 脚注与内联 HTML

这里有一个脚注的引用[^1]，这里还有另一个[^2]。
你可以使用 `<kbd>` 标签来表示按键操作：请按 <kbd>Ctrl</kbd> + <kbd>C</kbd> 复制。

---

## 2. Blowfish 核心 Shortcode 测试

### 2.1 警告与提示 (Alerts)

Blowfish 支持多种上下文类型的警告框，用于区分信息层级：

{{< alert icon="info-circle" >}}
**默认/信息 (Info)**: 用于提供中立的补充信息。
{{< /alert >}}

{{< alert context="success" icon="check-circle" >}}
**成功 (Success)**: 操作成功或推荐的最佳实践。
{{< /alert >}}

{{< alert context="warning" icon="triangle-exclamation" >}}
**警告 (Warning)**: 需要注意的事项，可能导致非预期结果。
{{< /alert >}}

{{< alert context="danger" icon="fire" >}}
**危险 (Danger)**: 严重警告，可能导致数据丢失或系统崩溃。
{{< /alert >}}

### 2.2 徽章、按钮与图标 (Badges, Buttons & Icons)

**图标 (Icons):** 支持 FontAwesome。例如：{{< icon "github" >}} {{< icon "twitter" >}} {{< icon "envelope" >}}

**徽章 (Badges):** 用于标签或状态：{{< badge >}}v2.70.0{{< /badge >}} {{< badge >}}Stable{{< /badge >}}

**按钮 (Buttons):**
{{< button href="[https://ethanyin.cn](https://ethanyin.cn)" target="_blank" >}}访问个人站点测试{{< /button >}}

### 2.3 Github / Gitlab 仓库卡片

通过 API 实时拉取仓库信息（测试时请确保本地网络可访问外部 API）：

{{< github repo="nunocoracao/blowfish" >}}

### 2.4 打字机效果 (TypeIt)

{{< typeit >}}
$sudo apt-get update$ sudo apt-get install hugo
$ echo "Blowfish is awesome!"
{{< /typeit >}}

### 2.5 标签页选项卡 (Tabs)

用于在有限空间内展示并列的内容（例如不同操作系统的安装命令）：

{{< tabs >}}
  {{< tab "macOS" >}}
  ```bash
  brew install hugo
  ```
  {{< /tab >}}
  {{< tab "Windows" >}}
  ```powershell
  winget install Hugo.Hugo.Extended
  ```
  {{< /tab >}}
  {{< tab "Linux" >}}
  ```bash
  snap install hugo --channel=extended
  ```
  {{< /tab >}}
{{< /tabs >}}

### 2.6 时间线 (Timeline)

用于展示项目历程或个人履历：

{{< timeline >}}
  {{< timelineItem icon="rocket" header="项目启动" badge="2026-01" >}}
  确定技术栈为 Hugo + Blowfish，完成本地环境搭建。
  {{< /timelineItem >}}
  {{< timelineItem icon="code-branch" header="功能开发" badge="2026-02" >}}
  完成自定义 Shortcode 编写与 CI/CD 流程配置。
  {{< /timelineItem >}}
  {{< timelineItem icon="check" header="正式上线" badge="2026-03" >}}
  域名解析完成，全站正式对外提供访问。
  {{< /timelineItem >}}
{{< /timeline >}}

### 2.7 详情折叠 (Details)

{{< details "点击展开：详细配置说明" >}}
如果你看到了这段文字，说明 `details` shortcode 渲染正常。
这里可以放入代码块：
```json
{
  "status": "ok",
  "theme": "blowfish"
}
```
{{< /details >}}

---

## 3. 高级代码与多媒体支持

### 3.1 代码高亮与行号

测试 Hugo 原生的高亮引擎 (Chroma)，支持高亮特定行和行号显示：

```go {linenos=table,hl_lines=[3,6]}
package main

import "fmt" // 这行将被高亮

func main() {
    fmt.Println("Testing code highlighting!") // 这行也将被高亮
}
```

### 3.2 数学公式 (KaTeX/MathJax)

*请确保 Front Matter 中已开启 `math: true`。*

行内公式测试：物理学中的经典公式 $F=ma$，以及概率论中的贝叶斯定理 $P(A|B)=\frac{P(B|A)P(A)}{P(B)}$。

独立公式块测试（高斯积分）：

$$\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}$$

### 3.3 图表渲染

#### Mermaid 图表

*请确保 Front Matter 中已开启 `mermaid: true`。*

{{< mermaid >}}
sequenceDiagram
    participant User as 用户
    participant Hugo as Hugo 引擎
    participant Browser as 浏览器
    
    User->>Hugo: 执行 `hugo server`
    Hugo-->>Hugo: 解析 Markdown & 模板
    Hugo->>Browser: 热重载推流 (LiveReload)
    Browser-->>User: 渲染页面
{{< /mermaid >}}

#### Chart.js 图表

*请确保 Front Matter 中已开启 `chart: true`。*

{{< chart >}}
type: 'bar',
data: {
  labels: ['Hugo', 'Jekyll', 'Hexo', 'Gatsby'],
  datasets: [{
    label: '构建速度 (假想分数)',
    data: [100, 45, 60, 30],
    backgroundColor: [
      'rgba(54, 162, 235, 0.2)',
      'rgba(255, 99, 132, 0.2)',
      'rgba(255, 206, 86, 0.2)',
      'rgba(75, 192, 192, 0.2)'
    ],
    borderColor: [
      'rgba(54, 162, 235, 1)',
      'rgba(255, 99, 132, 1)',
      'rgba(255, 206, 86, 1)',
      'rgba(75, 192, 192, 1)'
    ],
    borderWidth: 1
  }]
}
{{< /chart >}}

---

## 4. 图库与轮播图 (Media & Galleries)

*注意：要使此部分生效，你需要将多张图片放入与本 Markdown 文件同名的文件夹中（Page Bundles 模式），或者在 `static` 目录下放置图片并修改路径。*

### 4.1 图片画廊 (Gallery)

```html
```

### 4.2 轮播图 (Carousel)

```html
```

---

## 结尾

至此，大部分 Blowfish 的高频及复杂组件均已覆盖。

[^1]: 这是第一个脚注的文本。用于对正文内容进行补充说明。
[^2]: 这是第二个脚注，支持**Markdown 语法**的嵌套渲染。