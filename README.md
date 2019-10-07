# blog

## 快速开始

### 安装 Hexo

本博客由 `hexo` 驱动，主题由 `hexo-theme-butterfly` [hexo-theme-butterfly](https://jerryc.me/posts/21cfbf15/)提供,下面是安装 `Hexo` 的命令：

```bash
npm install hexo-cli -g
```

### 克隆代码

```bash
# 克隆代码到本地.
git clone https://github.com/Jarvan-IV/blog.git
```

### 安装依赖

克隆了代码之后，第一次运行需要安装所需要的相关依赖。进入 `blog` 的根目录中，执行：

```bash
cd blog

# 执行此命令安装依赖
npm install
```

### 运行

安装完依赖之后，接下来就可以直接启动运行本站点了，执行如下命令即可：

```bash
hexo s
```

当你在终端中看到如下信息时就表明已经启动成功了，可以在本地直接访问 [http://localhost:4000/](http://localhost:4000/) 看效果。

```bash
INFO  Start processing
INFO  Hexo is running at http://localhost:4000 .
```

## 写作

### 创建文章

你可以执行下列命令来创建一篇新文章，也可以直接复制 Markdown 文件到 `source/_post` 文件对应的分类目录中，然后添加该文章对应的 `Front-matter` 即可。

```bash
hexo new title
```

### Front-matter

`Front-matter` 选项中的所有内容均为**非必填**的。但我仍然建议你至少填写 `title` 和 `date` 的值。

| 配置选项   | 默认值                      | 描述                                                         |
| ---------- | --------------------------- | ------------------------------------------------------------ |
| title      | `Markdown` 的文件标题        | 文章标题，强烈建议填写此选项                                 |
| date       | 文件创建时的日期时间          | 发布时间，强烈建议填写此选项，且最好保证全局唯一             |
| author     | 根 `_config.yml` 中的 `author` | 文章作者                                                     |
| img        | `featureImages` 中的某个值   | 文章特征图，推荐使用图床(腾讯云、七牛云、又拍云等)来做图片的路径.如: `http://xxx.com/xxx.jpg` |
| top        | `true`                      | 推荐文章（文章是否置顶），如果 `top` 值为 `true`，则会作为首页推荐文章 |
| cover      | `false`                     | `v1.0.2`版本新增，表示该文章是否需要加入到首页轮播封面中 |
| coverImg   | 无                          | `v1.0.2`版本新增，表示该文章在首页轮播封面需要显示的图片路径，如果没有，则默认使用文章的特色图片 |
| password   | 无                          | 文章阅读密码，如果要对文章设置阅读验证密码的话，就可以设置 `password` 的值，该值必须是用 `SHA256` 加密后的密码，防止被他人识破。前提是在主题的 `config.yml` 中激活了 `verifyPassword` 选项 |
| toc        | `true`                      | 是否开启 TOC，可以针对某篇文章单独关闭 TOC 的功能。前提是在主题的 `config.yml` 中激活了 `toc` 选项 |
| mathjax    | `false`                     | 是否开启数学公式支持 ，本文章是否开启 `mathjax`，且需要在主题的 `_config.yml` 文件中也需要开启才行 |
| summary    | 无                          | 文章摘要，自定义的文章摘要内容，如果这个属性有值，文章卡片摘要就显示这段文字，否则程序会自动截取文章的部分内容作为摘要 |
| categories | 无                          | 文章分类，本主题的分类表示宏观上大的分类，只建议一篇文章一个分类 |
| tags       | 无                          | 文章标签，一篇文章可以多个标签                              |

> **注意**:
> 1.如果 `img` 属性不填写的话，文章特色图会根据文章标题的 `hashcode` 的值取余，然后选取主题中对应的特色图片，从而达到让所有文章都的特色图**各有特色**。
> 2.如果要对文章设置阅读验证密码的功能，不仅要在 Front-matter 中设置采用了 SHA256 加密的 password 的值，还需要在主题的 `_config.yml` 中激活了配置。有些在线的 SHA256 加密的地址，可供你使用：[开源中国在线工具](http://tool.oschina.net/encrypt?type=2)、[chahuo](http://encode.chahuo.com/)、[站长工具](http://tool.chinaz.com/tools/hash.aspx)。

以下为文章的 `Front-matter` 示例。

#### 最简示例

```yaml
---
title: typora-vue-theme主题介绍
date: 2018-09-07 09:25:00
---
```

#### 最全示例

```yaml
---
title: typora-vue-theme主题介绍
date: 2018-09-07 09:25:00
author: 赵奇
img: /source/images/xxx.jpg
top: true
cover: true
coverImg: /images/1.jpg
password: 8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92
toc: false
mathjax: false
summary: 这是你自定义的文章摘要内容，如果这个属性有值，文章卡片摘要就显示这段文字，否则程序会自动截取文章的部分内容作为摘要
categories: Markdown
tags:
  - Typora
  - Markdown
---
```
