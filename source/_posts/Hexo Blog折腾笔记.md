---
title: Hexo Blog折腾笔记
date: 2019-03-12 00:01
tags: 
- hexo
---

## 环境安装
安装git
安装nodejs
配置npm镜像
```bash
# 淘宝npm镜像官网：https://npm.taobao.org/
# 镜像地址：https://registry.npm.taobao.org
$ npm config set registry <https://mirror.url>
```
安装hexo-cli
```bash
$ npm install -g hexo-cli
```

<!-- more -->

## 建站
创建blog（基本目录结构文件）
```bash
$ hexo init <folder>
$ cd <folder>
# 安装package.json指定的包
$ npm install
```

## 配置
> 当前已经可以使用`hexo server`命令运行博客查看效果

### 配置hexo
修改hexo配置文件`<blog_root>/_config.yml`
```yml
title: #网站标题
subtitle: #网站副标题
author: #作者名字
```

### 配置主题
> 使用hexo主题：Theme-next，文档地址：[https://theme-next.org/](https://theme-next.org/)

#### theme-next安装
下载next主题文件：[https://github.com/theme-next/hexo-theme-next/archive/master.zip](https://github.com/theme-next/hexo-theme-next/archive/master.zip)，或使用git：`git clone https://github.com/theme-next/hexo-theme-next themes/next`

将文件解压，重命名为 hexo-next，拷贝至`<blog_root>/themes`目录下
修改hexo配置文件`<blog_root>/_config.yml`，使用next主题:

```yml
theme: hexo-next
```

#### 切换主题模式
主题模式由Muse切换为Gemini，修改hexo-next配置文件 `<next_root>/_config.yml`
```yml
# scheme: Muse
scheme: Gemini
```

#### 修改摘录(excerpt)方式
修改hexo-next配置文件 `<next_root>/_config.yml`
```yml
excerpt_description: false # 使用 front-matter 的 description 字段作为简介显示在博客列表页（当字段为空时显示完整博客），关闭
# Automatically Excerpt. Not recommend.
# Use <!-- more --> in the post to control excerpt accurately.
auto_excerpt:
  enable: true # 自动摘录，开启，暂时没发现bug
  length: 150
# Automatically scroll page to section which is under <!-- more --> mark.
scroll_to_more: true
```

#### 修改“Edited on”展示策略
修改hexo-next配置文件 `<next_root>/_config.yml`
```yml
post_meta:
  updated_at:
    # If true, show updated date label only if `updated date` different from `created date` (post edited in another day than was created).
    # If false show anyway, but if post edited in same day, show only edited time.
    another_day: false
```

#### 添加about和tags
创建about和tags页面
```bash
$ cd <blog_root>
$ hexo new page <page_name>
```
生成`<hexo_root>/source/about/index.md`和`<hexo_root>/source/tags/index.md`文件，在about下的`index.md`中添加个人信息；修改tags下的`index.md`，在 Front-matter 中添加 `type: tags`

修改hexo-next根目录下的配置文件 `<next_root>/_config.yml`，菜单栏配置格式：`Key: /link/ || icon`

| key | link | icon |
| - | - | - |
| 名称 | uri， 即菜单项对应页面链接：http://[home_page]/[link] | 使用的FontAwesome图标名称 |

修改为
```yml
menu:
  tags: /tags/ || tags
  about: /about/ || user
```

#### 搜索服务
> 使用`Local Search`的搜索服务

安装hexo插件
```bash
$ npm install hexo-generator-searchdb --save
```
修改hexo配置文件`<blog_root>/_config.yml`：
```yml
# Configration for Theme-Next
search:
  path: search.xml
  field: post
  format: html
  limit: 20
```
修改hexo-next配置文件 `<next_root>/_config.yml`
```yml
local_search:
  enable: true
  # if auto, trigger search by changing input
  # if manual, trigger search by pressing enter key or search button
  trigger: auto
```

#### 侧边栏头像
头像文件保存至`<hexo_root/source/uploads/avatar.png>`
修改hexo-next配置文件 `<next_root>/_config.yml`
```yml
avatar:
  url: /uploads/avatar.png
  # If true, the avatar would be dispalyed in circle.
  rounded: true
  # If true, the avatar would be rotated with the cursor.
  rotated: true
```

#### 侧边栏社交信息
修改hexo-next配置文件 `<next_root>/_config.yml`，格式与前面相同：
```yml
social:
  GitHub: https://github.com/xyty007 || github
  E-Mail: xyty2012@outlook.com || envelope
  知乎: https://www.zhihu.com/people/initial-75 || book # FontAwesome的知乎图标为纯白色，不能用
social_icons:
  icons_only: true #只显示图标，不显示文字
```

## 保存和部署
> 将博客部署到gitpage，参考文档(官网)：[https://pages.github.com/](https://pages.github.com/)
> 创建符合gitpage命名的repo，使用source分支存放博客源码，使用master分支存放hexo生成的页面
> (gitpage默认使用master发布；所有的repo都可以使用gitpage，只是需要手动开启，另外url会有区别)

### 保存
创建存放博客的仓库`<git_repo>`
向项目中添加`.gitignore`文件，提取自 [hexo.site](https://github.com/hexojs/site)
将项目代码push到source分区

### 部署
使用git部署时，每次deploy会使用生成的新文件强制覆盖远端`<git_repo>`的master分支中的旧文件
安装git部署插件
```bash
$ npm install hexo-deployer-git --save
```
修改hexo配置文件`<blog_root>/_config.yml`
```yml
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: <git_repo>
  branch: master
```
运行命令部署
```bash
$ hexo deploy #或 hexo d
```

## 开启评论功能(基于next主题)
> 评论功能使用Gitalk服务实现

创建github验证应用：[Register a new OAuth application](https://github.com/settings/applications/new)，需要填写的项目如下：

| 项目 | 描述 |
| - | - |
| Application name | 应用名称，会在登录评论的登录验证界面展示 |
| Homepage URL | 可以填博客主页 |
| Application description | 应用简介 |
| Authorization callback URL | 必须填该博客主页 |

应用创建完成后，会获得Client ID和Client Secret

使用存放博客的`<git_repo>`存储评论数据，修改hexo-next配置文件 `<next_root>/_config.yml`如下(admin_user可以和github_id相同)
```yml
gitalk:
  enable: true
  github_id: xyty007 # Github repo owner
  repo: <git_repo> # Repository name to store issues.
  client_id: xxxxxxxxxxxxxxxxxxxx # Github Application Client ID
  client_secret: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx # Github Application Client Secret
  admin_user: xyty007 # GitHub repo owner and collaborators, only these guys can initialize github issues
```
## 个人博客站收录
> 参考：
> [Hexo个人博客站点被百度谷歌收录](https://blog.csdn.net/qq_32454537/article/details/79482914)
> [给Hexo搭建的博客增加百度谷歌搜索引擎验证](https://zhuxin.tech/2017/10/20/给%20Hexo20%搭建的博客增加百度谷歌搜索引擎验证/)
> [Hexo插件之百度主动提交链接](https://monsterlin.github.io/2017/01/04/Hexo插件之百度主动提交链接/)

### 谷歌
访问 [Google Search Console](https://search.google.com/search-console) “添加资源”，支持两种方式： ![请选择资源类型(使用gitpage薅羊毛的各位请选择“网址前缀”)](https://ghamster.gitee.io/ihservice/Hexo_Blog折腾笔记/请选择资源类型.png)

验证方法可选：
- HTML标记：修改hexo-next配置文件 `<next_root>/_config.yml`
```yml
# Google Webmaster tools verification setting
# See: https://www.google.com/webmasters
google_site_verification: uL1SKjfKIZMEUOU77rLDMH7JfjC_Gz1JOA(your code)
```
- HTML文件：将谷歌验证文件拷贝至`<hexo_root>/source`下，并在头部添加`layout: false`，避免被渲染

添加sitemap
- 安装sitemap插件
```bash
$ npm install hexo-generator-sitemap --save
```
- `hexo clean`，`hexo g`生成`sitemap.xml`，部署博客
- 在`Google Search Console`的`站点地图`栏添加`sitemap.xml`文件

### 百度
与谷歌类似，访问 [百度搜索资源平台](https://ziyuan.baidu.com/site/index) ,添加网站并验证--*百度通过HTML标记验证翻车，大概是因为gitpage封了百度爬虫*

(*同样的原因，这条基本没用*)添加sitemap插件命令：`npm install hexo-generator-baidu-sitemap --save`，生成的文件名为`baidusitemap.xml`

百度提供了三种链接提交方式：主动推送（实时）、自动推送和sitemap，设置路径：[用户中心->站点管理->（自己的站点）->链接提交](https://ziyuan.baidu.com/linksubmit/index)，next主题支持了第二种，在配置文件`<next_root>/_config.yml`中设置：
```yml
# Enable baidu push so that the blog will push the url to baidu automatically which is very helpful for SEO.
baidu_push: true
```

## 踩坑
这里不定期更新一下最近遇到的问题～

### 乱码
*现象*：菜单栏、“Read more >>”、Archrives页面……等任何由next主题中的文字(包括英文)都显示为乱码
*解决方式*： 在hexo配置文件中将`language`设置为`en`或`zh-CN`，不要使用缺省值

### 图标
*现象*： 侧边栏Font Awesome图标不显示，控制台显示`lib`目录下找不到`font-awesome.min.css`文件
*原因*： 生成博客时，插件未将Font Awesome文件打包到合适位置，查看`<next_root>/layout/_partials/head/head.swig`文件，相关代码如下：
```swig
{% set font_awesome_uri = url_for(theme.vendors._internal + '/font-awesome/css/font-awesome.min.css?v=4.6.2') %}
{% if theme.vendors.fontawesome %}
  {% set font_awesome_uri = theme.vendors.fontawesome %}
{% endif %}
```
可以修改hexo-next配置文件 `<next_root>/_config.yml`，使用cdn提供的Font Awesome
```yml
vendors:
  # Internal version: 4.6.2
  # See: https://fontawesome.com
  # Example:
  # fontawesome: //cdn.jsdelivr.net/npm/font-awesome@4/css/font-awesome.min.css
  # fontawesome: //cdnjs.cloudflare.com/ajax/libs/font-awesome/4.6.2/css/font-awesome.min.css
  fontawesome: https://cdn.bootcss.com/font-awesome/4.6.2/css/font-awesome.min.css
```
> 参考： [the icons are gone?](https://github.com/theme-next/hexo-filter-optimize/issues/2)

### 图片与图床
> Hexo官方描述：[资源文件夹](https://hexo.io/zh-cn/docs/asset-folders)
> 主流图床：[markdown博客图床上传的艰辛之路](https://wdd.js.org/the-hard-way-of-markdown-insert-images.html)

Hexo给出的方案最大的问题在于实时显示和可传播性：
- 标签就不说了，离了Hexo直接玩完
- 使用markdown的标准格式`![]()`时，与通用的markdown链接解析逻辑不兼容。比如`_post`文件夹下有一篇名为`test.md`的博客，Hexo默认会生成`test`文件夹存放图片资源，博客中图片的引用链接必须直接写成`xxx.png`，而不是markdown标准的`/test/xxx.png`或`./test/xxx.png`

最好的解决方式是搭建图床，但是自建图床太贵，严重违背了“一个子儿不花薅羊毛”的初衷，公共图床又怕哪天就没了...

**所以，我盯上了gitee，没错！就是国内的码云！访问速度快，存储容量高！重点是可靠免费！**

在gitee创建repo，创建一个`index.html`文件（内容不限，标准的html文档就可以），样例代码：
```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <title>博客图床</title>
    </head>
    <body>
        <div>
            <a href="https://ghamster0.github.io/">博客地址</a>
            <a href="https://gitee.com/Ghamster/IHService">仓库地址</a>
    </body>
</html>
```
选择*服务->Gitee Pages*，开启Gitee Pages服务
上传图片文件，更新服务（唯一的美中不足，免费版需要手动更新）后便可通过url访问，路径格式为`<gitee page主页地址>/<文件在仓库中的路径>`。如用户`Ghamster`的仓库`HService`下，存放文件`Hexo_Blog折腾笔记/请选择资源类型.png`，则图片链接为：[https://ghamster.gitee.io/ihservice/Hexo_Blog折腾笔记/请选择资源类型.png](https://ghamster.gitee.io/ihservice/Hexo_Blog折腾笔记/请选择资源类型.png)
![图床首页](https://ghamster.gitee.io/ihservice/Hexo_Blog折腾笔记/图床首页.png)
![图床文件页](https://ghamster.gitee.io/ihservice/Hexo_Blog折腾笔记/图床文件页.png)

**懒人通道**
显然手动填写每张图片的url太过繁琐，所以在此通过编写脚本简化这一工作，代码地址：[https://github.com/Ghamster0/Blog-Tools](https://github.com/Ghamster0/Blog-Tools)
使用方式：
- 将图片放到博客repo中，可以将所有图片存放到一个默认位置，如`<hexo_root>/source/images`,也可以在`_post`下为每个`.md`文件创建单独的文件夹
- 写作时，使用相对链接引用，如`./folder_for_title/xxx.png`或`../images/xxx.png`
- 运行脚本，将图片拷贝到图床仓库，并自动修改链接

## 常用指令
```bash
$ hexo init [folder] # 新建一个网站。如果没有设置 folder ，Hexo 默认在目前的文件夹建立网站
$ hexo new [layout] <title> # 新建一篇文章。如果没有设置 layout ，默认使用 _config.yml 中的 default_layout 参数代替
$ hexo generate # 生成静态文件，该命令可以简写为 $ hexo g
$ hexo clean # 清除缓存文件 (db.json) 和已生成的静态文件 (public)。 在某些情况（尤其是更换主题后），如果发现您对站点的更改无论如何也不生效，
$ hexo render <file1> [file2] ... # 渲染文件
$ hexo server # 启动服务器，http://localhost:4000/
$ hexo deploy # 部署网站，该命令可以简写为 $ hexo d您可能需要运行该命令
$ hexo list <type> # 列出网站资料
$ hexo publish [layout] # <filename> 发表草稿
$ hexo version # 显示 Hexo 版本
```