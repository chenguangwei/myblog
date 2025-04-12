<div align="center">
<h1>hugo LoveIt 个人开源博客</h1>

![Hugo](https://img.shields.io/badge/Hugo-EBB851?style=for-the-badge&logo=hugo&logoColor=#FF4088)    ![Markdown](https://img.shields.io/badge/Markdown-34BA91?style=for-the-badge&logo=Markdown&logoColor=#FF4088)

<strong>一个 Hugo + Markdown 的个人博客项目，开箱即用</strong>



本博客主题修改自[hugo-theme-LoveItn](https://github.com/dillonzq/LoveItn)

[**在线预览（忽略站内广告）**](https://myblog.yilutongxing.cn) • [**部署配置教程**](#部署配置教程) • [**Twitter关注我**](https://x.com/cgw1017) 
</div>

### 功能特性



## 目录

- [部署配置教程](#部署配置教程)
  * [本地启动](#本地启动)
  * [全局配置](#全局配置)
- [定制化](#定制化)
  * [配置作者卡片和相关文章](#配置作者卡片和相关文章)
    + [相关文章](#相关文章)
    + [作者卡片](#作者卡片)
  * [评论功能](#评论功能)
    + [Utterance](#utterance)
- [菜单和对应的目录](#菜单和对应的目录)
- [添加文章](#添加文章)
- [编译和发布](#编译和发布)
- [配置自动发布流程](#配置自动发布流程)
- [自定义域名（可选）](#自定义域名（可选）)
  * [购买域名](#购买域名)
  * [解析域名](#解析域名)
  * [配置域名](#配置域名)
  * [自动更改自定义域名](#自动更改自定义域名)


### 部署配置教程

#### 本地启动

下载本项目，进入项目目录，就可以直接启动项目看看效果了。

使用下面的命令启动

```shell
hugo server
```

之后，看到下面这样的输出，说明启动成功了。

```shell
......
Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
Press Ctrl+C to stop
```

现在在浏览器打开 `http://localhost:1313`，就能看到效果了。当然了，刚开始肯定只有基础样式，没有内容。


启动之后，之后做的修改会自动热部署，不用重启应用，除非有些配置的修改。什么配置呢，就是那种你发现改了，刷新页面了，但是没起作用，很多时候就是需要重启了。

`Ctrl+C`停止应用，然后`hugo server`重新启动。

#### 全局配置

首先将主题目录下的 `config.toml`文件复制到项目根目录下，这个配置文件就用来配置网站的全局配置了，比如host、标题、菜单、默认语言等等。

我用 hugo-theme-den 这个主题的配置做一个介绍。

以下是 Hugo 配置文件中各项设置的名称、值和说明：

| 名称                             | 值                                                  | 说明                                    |
| ------------------------------ | -------------------------------------------------- | ------------------------------------- |
| baseURL                        | "https://myblog.yilutongxing.cn"                           | 网站的基础 URL，引用的本地图片、css等，将会以这个地址作为 host |
| title                          | "古时的风筝"                                            | 网站的标题                                 |
| theme                          | "hugo-theme-den"                                   | 网站所使用的主题                              |
| enableRobotsTXT                | true                                               | 生成 robots.txt 文件，用于控制搜索引擎爬虫的访问        |
| enableEmoji                    | true                                               | 启用 Emoji 表情支持                         |
| hasCJKLanguage                 | true                                               | 检测 CJK（中文、日文、韩文）语言用于字数统计等功能           |
| preserveTaxonomyNames          | true                                               | 不将标签名转为小写                             |
| rssLimit                       | 20                                                 | 限制 RSS 订阅中的条目数量                       |
| disablePathToLower             | true                                               | 禁止将路径转为小写                             |
| paginate                       | 20                                                 | 每页显示的条目数量（用于归档、标签和分类）                 |
| paginatePath                   | "page"                                             | 分页路径的前缀                               |
| PygmentsCodeFences             | true                                               | 启用代码块的语法高亮                            |
| PygmentsUseClasses             | true                                               | 需要用于 shhighlight shortcode            |
| disqusShortname                | ""                                                 | Disqus 评论系统的 shortname                |
| googleAnalytics                | ""                                                 | Google Analytics 的跟踪 ID               |
| defaultContentLanguage         | "zh-cn"                                            | 默认使用的语言                               |
| defaultContentLanguageInSubdir | false                                              | 默认语言是否在子目录中                           |
| permalinks.posts               | "/:year/:month/:day/:slug/"                        | 文章的永久链接格式                             |
| permalinks.categories          | "/category/:slug/"                                 | 分类的永久链接格式                             |
| permalinks.tags                | "/tag/:slug/"                                      | 标签的永久链接格式                             |
| permalinks.pages               | "/:slug/"                                          | 页面的永久链接格式                             |
| author.name                    | "风筝"                                               | 作者的名称                                 |
| sitemap.changefreq             | "weekly"                                           | sitemap.xml 文件的更新频率                   |
| sitemap.priority               | 0.5                                                | sitemap.xml 文件的优先级                    |
| sitemap.filename               | "sitemap.xml"                                      | sitemap.xml 文件的文件名                    |
| params.since                   | "2023"                                             | 网站创建时间                                |
| params.rssFullContent          | true                                               | 在 RSS 订阅中使用完整内容而不是摘要                  |
| params.keywords                | ["Hugo", "theme","编程",'java','ChatGPT',"程序员",'开发'] | 网站的关键词                                |
| params.description             | "一个程序员的个人博客"                                       | 网站的描述                                 |
| params.logoTitle               | " "                                                | 在左上角显示的 Logo 标题                       |
| params.siteLogoImage           | ""                                                 | 在 Logo 标题旁边显示的 Logo 图片                |
| params.headerImage             | "images/background.png"                            | 头部背景图片                                |
| params.showAuthorCard          | true                                               | 是否在文章下方显示作者信息                         |
| params.comments                | true                                               | 是否启用评论功能                              |
| params.showMenuLanguages       | true                                               | 是否显示多                                 |



