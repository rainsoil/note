# Hexo  安装教程

## 1.  安装前的准备

### 1. git安装

####  window

直接 git官网下载window的git 安装包,就可以了

#### linux

>  yum install git



### 2. node 安装

####   window

去node 官网下载安装包

####  linux

>   yum install nodejs



## 2. 安装Hexo

###  2.1 安装hexo

git和mode安装好了之后,就可以安装hexo了,首先你要先创建一个文件夹blog,然后cd 到这个文件夹下

输入命令 

>```bash
> npm install -g hexo-cli
>```
>
>  也可以使用淘宝的npm源
>
> ```bash
> npm install -g cnpm --registry=https://registry.npm.taobao.org
> ```
>
> 

使用hexo -v 查看一下版本

###  2.2 初始化项目

```bash
hexo init  myblog
```

这个myblog可以自己取什么名字都行，然后

```bash
cd  myblog
npm install 
```

执行成功之后,指定文件夹目录下有:

- node_modules: 依赖包
- public : 存在生成的页面
- scaffolds: 生成文章的一些模板
- themes: 主题
- _config.yml: hexo的主配置文件

```
hexo g
hexo server
```

启动成功后,在浏览器输入 http://localhost:4000  就可以看到博客的内容了 

## 3. 基本配置

在文件根目录下的_config.yml 文件,就是整个hexo框架的配置文件了,可以在里面修改大部分的配置,详情如下:

#### 网站

| 参数        | 描述                               |
| ----------- | ---------------------------------- |
| title       | 网站标题                           |
| subtitle    | 网站副标题                         |
| description | 网站描述                           |
| author      | 你的名字                           |
| language    | 网站使用的语言 (zh-Hans为简体中文) |
| timezone    | 网站时区,默认为电脑使用的时区      |

####  网址

| 参数               | 描述                     |
| ------------------ | ------------------------ |
| url                | 网址                     |
| root               | 网址根目录               |
| permalink          | 文章的永久链接格式       |
| permalink_defaults | 永久链接中各部分的默认值 |

在这里,你需要把url 改成你的网站的域名

permalink,也就是你生成某个文章的url格式

比如我新建一个文章叫new.md. 那么这个时候他生成的地址就是 http://yousite.com/2019/08/31/new

再往下翻,中间这些默认就好了

```bash
theme: landscape

# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: <repository url>
  branch: [branch]
  	
```

theme 就是选择什么主题,也就是在theme这个文件下,在官网上提供了很多的主题,默认给你安装的是landscape 这个主题,当你需要更换主题的时候,只需要把主题的文件放在theme 这个文件夹下,在修改这个参数就可以了 

接下来就是deploy ,网站的部署,repo就仓库的缩写, branch 是选择仓库的哪个分支,我这里选择的是部署到服务器上,就介绍这个了 

#### Front-matter

Front-matter是 文件最上面用--- 分割的区域, 用来指定个别文件的变量,举例来说:

```bash

---
title: Hello World
date: 2013/7/13 20:46:25
---
```

下面是预定义的参数,你可以再模板中使用这些参数并加以利用

| 参数       | 描述               |
| ---------- | ------------------ |
| layout     | 布局               |
| title      | 标题               |
| date       | 建立日期           |
| updated    | 更新日期           |
| comments   | 开启文章的评论功能 |
| tags       | 标签               |
| categories | 分类               |
| permalink  | 覆盖文章网址       |

其中,分类和标签是需要区别一下的,分类具有顺序性和层次性,而标签没有

```markdown
categories:
 - 后端
 - java
 说明 后端为一级分类,java为二类
tags:
- 架构
- 源码
只是说明有两个标签而已
```

#### layout（布局）

当你每一次使用代码

> hexo new  paper

其实他默认的就是post 这个布局, 也就是在resource下的 _post 里面

Hexo其实有三种默认的布局: post、page、draft,他们分别对应不同的路径,而自定义的其他布局都和 post相同，都将存储到 source/_post 文件夹中

| 布局  | 路径          |
| ----- | ------------- |
| post  | source/_posts |
| page  | source        |
| draft | source/_draft |

而new这个命令其实是:

> hexo new [layout] <title>

只不过这个 layout 默认的是post罢了

#####  Page

如果你想另起一页,那么可以使用

> hexo new page newPage

系统会自动给你在source 文件下创建一个 newPage的文件夹,以及newPage 文件夹的index.md 文件,这样你访问的newPage 对应的连接就是 http://xxx.xxx/newPage

#####  draft

draft是草稿的意思,也就是如果你想写文章, 但是又不希望被看到,那么可以

>  hexo  new draft newPage

这样会在 source/_draft 中新建一个newPage.md 文件,如果你的草稿文章在写的过程中需要预览一下,可以使用

> hexo  server --draft

在本地端口中开启服务预览,

如果你的草稿文件写完了,想要发布到post 中

>  hexo publish draft newPage

就会自动的把newPage.md 发送到 post中了

### 添加分类

执行下面的命令

> hexo new page categories

这个命令会新建一个分类页面,会在站点的目录下的resource 文件下新建一个 categories 文件夹

![](http://files.luyanan.com//img/20190831151353.png)

打开里面的index.md 文件,修改为

```markdown
---
title: 分类
date: 2019-08-30 15:10:24
type: categories
---

```

###  添加标签

执行

> hexo new page tags

执行这个命令,会在source目录下生成上图中的tags 目录,修改里面的index.md文件为

```markdown
---
title: 标签
date: 2019-08-30 15:10:10
type: tags
---

```



##   4.  添加主题

到这一步的时候,如果你觉得默认的主题不好看，可以在官网的主题当中, 选择你喜欢的主题进行修改. 我这里使用的是NexT 主题

在myblog 目录下,执行

> ```
> git clone https://github.com/theme-next/hexo-theme-next themes/next
> ```

#### 菜单配置

 这里需要在next的目录下打开_config,yml 文件,修改

```yml
menu:
  home: / || home
  about: /about/ || user
  tags: /tags/ || tags
  categories: /categories/ || th
  archives: /archives/ || archive
  #schedule: /schedule/ || calendar
  #sitemap: /sitemap.xml || sitemap
  commonweal: /404/ || heartbeat
```

打开菜单的配置就可以了

####  添加博客头像

打开主题文件夹下的 _config.yml  配置文件

找到avatar

去掉 avatar 漆面的#. 然后将值设置成头像的地址就可以了

```
# Sidebar Avatar
# in theme directory(source/images): /images/avatar.gif
# in site  directory(source/uploads): /uploads/avatar.gif
avatar: /images/avatar.jpg
```

然后刷新页面,就可以看到

 ![](http://files.luyanan.com//img/20190831153215.png)

##### 头像圆角并旋转

打开主题文件夹的 source\css_common\components\sidebar 目录下的 sidebar-author.styl 文件，然后把下面的代码添加进去即可

```css
.site-author-image {
  display: block;
  margin: 0 auto;
  padding: $site-author-image-padding;
  max-width: $site-author-image-width;
  height: $site-author-image-height;
  border: $site-author-image-border-width solid $site-author-image-border-color;
  /* 头像圆形 */
  border-radius: 80px;
  -webkit-border-radius: 80px;
  -moz-border-radius: 80px;
  box-shadow: inset 0 -1px 0 #333sf;
  /* 设置循环动画 [animation: (play)动画名称 (2s)动画播放时长单位秒或微秒 (ase-out)动画播放的速度曲线为以低速结束 
    (1s)等待1秒然后开始动画 (1)动画播放次数(infinite为循环播放) ]*/
 
  /* 鼠标经过头像旋转360度 */
  -webkit-transition: -webkit-transform 1.0s ease-out;
  -moz-transition: -moz-transform 1.0s ease-out;
  transition: transform 1.0s ease-out;
}
img:hover {
  /* 鼠标经过停止头像旋转 
  -webkit-animation-play-state:paused;
  animation-play-state:paused;*/
  /* 鼠标经过头像旋转360度 */
  -webkit-transform: rotateZ(360deg);
  -moz-transform: rotateZ(360deg);
   transform: rotateZ(360deg); -webkit-transform: rotateZ(360deg); -moz-transform: rotateZ(360deg); -o-transform: rotateZ(360deg);
}
/* Z 轴旋转动画 */
@-webkit-keyframes play {
  0% {
    -webkit-transform: rotateZ(0deg);
  }
  100% {
    -webkit-transform: rotateZ(-360deg);
  }
}
@-moz-keyframes play {
  0% {
    -moz-transform: rotateZ(0deg);
  }
  100% {
    -moz-transform: rotateZ(-360deg);
  }
}
@keyframes play {
  0% {
     transform: rotateZ(0deg); -webkit-transform: rotateZ(0deg); -moz-transform: rotateZ(0deg); -o-transform: rotateZ(0deg);
  }
  100% {
     transform: rotateZ(-360deg); -webkit-transform: rotateZ(-360deg); -moz-transform: rotateZ(-360deg); -o-transform: rotateZ(-360deg);
  }
}
```

然后,刷新页面就可以看到效果了

#### RSS 设置

按照 hexo插件.需要在站点目录下进行安装

>  npm install --save hexo-generator-feed

安装完成后在站点目录下的 _config.yml 配置文件的文末添加下面这段代码：

```
# Extensions
## Plugins: http://hexo.io/plugins/
plugins: hexo-generate-feed
```

在主题文件下的 _config.yml 配置文件中,找到 rss. 在后面加上 /atom.xml  . 

```
# Set rss to false to disable feed link.
# Leave rss as empty to use site's feed link.
# Set rss to specific value if you have burned your feed already.
rss: /atom.xml

```

####  底部设置,

hexo 默认的底部设置为  由 hexo 提供强力支持,那么如何修改成自己的备案号呢?

在 主题文件下的_config.yml 配置文件中, 找到 footer, 修改为

```yml
footer:
  # Specify the date when the site was setup.
  # If not defined, current year will be used.
  since: 2019

  # Icon between year and copyright info.
  icon: user

  # If not defined, will be used `author` from Hexo main config.
  copyright: 晋ICP备17009011号-2
  # -------------------------------------------------------------
  # Hexo link (Powered by Hexo).
  powered: false

  theme:
    # Theme & scheme info link (Theme - NexT.scheme).
    enable: false
    # Version info of NexT after scheme info (vX.X.X).
    version: false
```

####  社交图标

在主题文件夹下的 _config.yml 配置文件中，搜索 Social。在需要设置图标的前面，把#删掉，链接改为你自己的链接。

```yml
# ---------------------------------------------------------------
# Sidebar Settings
# ---------------------------------------------------------------

# Social Links.
# Usage: `Key: permalink || icon`
# Key is the link label showing to end users.
# Value before `||` delimeter is the target permalink.
# Value after `||` delimeter is the name of FontAwesome icon. If icon (with or without delimeter) is not specified, globe icon will be loaded.
social:
  GitHub: https://github.com/luyanan0718 || github
  E-Mail: luyanan0718@163.com || envelope
  Gitee : https://gitee.com/luyanan || gitee
  #Google: https://plus.google.com/yourname || google
  #Twitter: https://twitter.com/yourname || twitter
  #FB Page: https://www.facebook.com/yourname || facebook
  #VK Group: https://vk.com/yourname || vk
  #StackOverflow: https://stackoverflow.com/yourname || stack-overflow
  #YouTube: https://youtube.com/yourname || youtube
  #Instagram: https://instagram.com/yourname || instagram
  #Skype: skype:yourname?call|chat || skype

```

####  动态背景

在主题配置文件中找到canvas_nest: false，把它改为canvas_nest: true就行了（注意分号后面要加一个空格

> 
>
> canvas_nest: true

####   主题文章阴影效果

- 具体实现方法：

- 打开\themes\next\source\css\_custom\custom.styl

  在里面加入

  ```css
  // 主页文章添加阴影效果
   .post {
     margin-top: 60px;
     margin-bottom: 60px;
     padding: 25px;
     -webkit-box-shadow: 0 0 5px rgba(202, 203, 203, .5);
     -moz-box-shadow: 0 0 5px rgba(202, 203, 204, .5);
    }
  ```

  

####  网站底部字数统计

- 具体方法实现：

- 在站点目录运行如下代码：

  > npm install hexo-wordcount --save

然后在/themes/next/layout/_partials/footer.swig文件尾部加上：

```html
<div class="theme-info">
  <div class="powered-by"></div>
  <span class="post-count">博客全站共{{ totalcount(site) }}字</span></div>
```

####  实现统计功能

- 具体实现方法：

- 在站点目录下运行以下命令：

> npm install hexo-wordcount  --save

然后在主题目录的配置文件中，修改以下参数的值为true

```yml
# Post wordcount display settings# Dependencies: https://github.com/willin/hexo-wordcountpost_wordcount:
  item_text: true
  wordcount: true
  min2read: true
```

#### 文末显示微信公众号

在主题目录下打开 _config.yml 配置文件，定位到 wechat_subscriber，修改相关配置，如下图所示：

```
# Wechat Subscriber
wechat_subscriber:
enabled: true
qcode: 二维码地址
description: 希望您可以扫一扫这个二维码，关注我的公众号。
```

#### 打赏功能设置

主题目录下打开 _config.yml 配置文件中，定位到 reward，把微信收款码和支付宝收款码存放到网络图库中。然后如下图所示修改相关配置，animation 这个字段是设置字体跳动，可以根据个人需求设置。

```yml
# Reward
reward_comment: 如果觉得文章对您有帮助请我喝杯咖啡吧
wechatpay: http://files.luyanan.com//img/20190831111607.png
alipay: http://files.luyanan.com//img/20190831111548.jpg
#bitcoin: /images/bitcoin.png
```

####  友情链接设置

在主题目录下的 _config.yml 配置文件中，搜索 links_title，然后根据自己的需求自己添加修改

```
# Blog rolls
links_icon: link
links_title: Links
links_layout: block
#links_layout: inline
links:
  #Title: http://example.com/
  #Gitee: https://gitee.com/luyanan
```

links_icon：显示在标题前的图标。

links_title：标题。

links_layout：block 一行一个，inline 一行多个。

links：要显示的链接以及名称

####  站内搜索

在站点目录安装

> npm install hexo-generator-search  --save
>
> npm install hexo-generator-searchdb --save

在站点目录的 _config.yml 配置文件中文末添加下面这段代码

```yml
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```

编辑主题目录的 _config.yml 配置文件,设置 local_search 为 true
