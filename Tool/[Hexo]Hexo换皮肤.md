---
title: Hexo+Github配置与主题
date: 2017-12-01 19:17:23
tags:
- Hexo

archives: Hexo
---

### 1. 基本配置

#### 1.1 语言设置

每个主题都会配置几种界面显示语言，修改语言只要编辑站点配置文件，找到 `language` 字段，并将其值更改为你所需要的语言(例如，简体中文)：
```
language: zh-Hans
```

#### 1.2 网站标题，作者

打开站点配置文件，修改这些值：
```
title: SmartSi #博客标题
subtitle: #博客副标题
description: #博客描述
author: sjf0115 #博客作者
```

注意：
```
配置文件要符合英文标点符号使用规范: 冒号后必须空格，否则会编译错误
```

#### 1.3 域名与文章链接

```
# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: http://sjf0115.github.io #你的博客网址
root: / #博客跟目录，如果你的博客在网址的二级目录下，在这里填上
permalink: :year/:month/:day/:title/ # 文章链接
permalink_defaults:
```

### 2. 安装与启用主题

最简单的安装方法是克隆整个仓库，在这里我们使用的是`NexT`主题：
```
cd hexo
git clone https://github.com/theme-next/hexo-theme-next themes/next
```
或者你可以看到其他详细的[安装说明](https://github.com/theme-next/hexo-theme-next/blob/master/docs/INSTALLATION.md)

安装后，我们要启用我们安装的主题，与所有`Hexo`主题启用的模式一样。 当克隆/下载完成后，打开站点配置文件， 找到 `theme` 字段，并将其值更改为 `next` 。
```
theme: next
```

### 3. 主题风格

`NexT` 主题目前提供了3中风格类似，但是又有点不同的主题风格，可以通过修改 `主题配置文件` 中的 `Scheme` 值来启用其中一种风格，例如我的博客用的是 `Mist` 风格，只要把另外两个用#注释掉即可:
```
# Schemes
#scheme: Muse
scheme: Mist
#scheme: Pisces
```

### 4. 设置 RSS

`NexT` 中 `RSS` 有三个设置选项，满足特定的使用场景。 更改 `主题配置文件`，设定 `rss` 字段的值：
- false：禁用 `RSS`，不在页面上显示 `RSS` 连接。
- 留空：使用 Hexo 生成的 `Feed` 链接。 你可以需要先安装 `hexo-generator-feed` 插件。
- 具体的链接地址：适用于已经烧制过 Feed 的情形。

### 5. 导航栏添加标签菜单

新建标签页面，并在菜单中显示标签链接。标签页面将展示站点的所有标签，若你的所有文章都未包含标签，此页面将是空的。

(1) 在终端窗口下，定位到 `Hexo` 站点目录下。使用如下命令新建一名为 `tags` 页面:
```
hexo new page "tags"
```
(2) 编辑刚新建的页面，将页面的类型设置为 `tags` ，主题将自动为这个页面显示标签云。页面内容如下：
```
title: 标签
date: 2017-12-22 12:39:04
type: "tags"
```
(3) 在菜单中添加链接。编辑 `主题配置文件` ，添加 `tags` 到 `menu` 中，如下:
```
  menu:
    home: /
    archives: /archives
    tags: /tags
```
(4) 使用时在你的文章中添加如下代码：
```
---
title: title name
date: 2017-12-12-22 12:39:04
tags:
  - first tag
  - second tag
---
```

### 6. 添加分类页面

新建分类页面，并在菜单中显示分类链接。分类页面将展示站点的所有分类，若你的所有文章都未包含分类，此页面将是空的。

(1) 在终端窗口下，定位到 `Hexo` 站点目录下。使用 `hexo new page` 新建一个页面，命名为 `categories` ：
```
hexo new page categories
```
(2) 编辑刚新建的页面，将页面的 `type` 设置为 `categories` ，主题将自动为这个页面显示分类。页面内容如下：
```
---
title: 分类
date: 2014-12-22 12:39:04
type: "categories"
---
```
(3) 在菜单中添加链接。编辑 `主题配置文件` ， 添加 `categories` 到 `menu` 中，如下:
```
menu:
  home: /
  archives: /archives
  categories: /categories
```
(4) 使用时在你的文章中添加如下代码：
```
---
title: title name
date: 2017-12-12-22 12:39:04
type: "categories"
---
```
### 7. 侧边栏社交链接

侧栏社交链接的修改包含两个部分，第一是链接，第二是链接图标。 两者配置均在 `主题配置文件` 中。

(1) 链接放置在 `social` 字段下，一行一个链接。其键值格式是 `显示文本: 链接地址 || 图标`：
```
social:
  GitHub: https://github.com/sjf0115 || github
  E-Mail: mailto:1203745031@qq.com || envelope
  CSDN: http://blog.csdn.net/sunnyyoona
```
备注:
```
如果没有指定图标（带或不带分隔符），则会加载默认图标。
```

(2) 设定链接的图标，对应的字段是 `social_icons`。其键值格式是 匹配键: Font Awesome 图标名称， 匹配键 与上一步所配置的链接的 显示文本 相同（大小写严格匹配），图标名称 是 Font Awesome 图标的名字（不必带 fa- 前缀）。 enable 选项用于控制是否显示图标，你可以设置成 false 来去掉图标。



















### 3. 插件

现在，在NexT配置中，您可以找到每个移动到外部存储库的模块的依赖关系，这些存储库可以通过主要组织链接找到。
