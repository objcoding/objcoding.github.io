---
layout: post
title: "使用 Hexo + Gitee 快速搭建属于自己的博客"
categories: Hexo
tags: blog 博客 hexo
author: 张乘辉
---

* content
{:toc}
程序员总会有一些技术文章的输出总结，很多人会选择各大第三方博客平台，但某些第三方博客平台的 UI 简直惨不忍睹，广告巨多，且对 md 格式支持得不够好等等，基于这些原因，我们有足够的理由搭建一套属于我们自己的博客。

我早在 17 年的时候就已经在 GitHub Pages 搭建了一套属于自己的博客，当时使用的 GitHub Pages 官方支持的 JekyII 工具进行部署，体验真的不是很好。后来的 Hexo 比它优秀太多了。现在 Gitee 也推出了自己的 Gitee Pages，由于是 Gitee Pages 的服务器是在国内的，因此访问速度非常快，而 GitHub 在国内的访问速度实在是惨不忍睹，于是我使用了 Hexo 在 Gtiee 搭建部署了另一个博客，下面我将搭建的整个过程总结写下来，也许能够帮助正在使用 Hexo 搭建博客的你。







## 安装 Hexo

Hexo 是一套基于 NodeJS 的博客框架，以 MarkDown 的写文方式，快速生成属于自己的静态博客系统。在使用 Hexo 之前，我们需要在系统中安装 NodeJS（以下使用 MacOS 系统环境）：

```bash
brew install node
```

安装好之后，使用命令 `npm --version`，若显示有版本信息，说明 NodeJs 已经安装成功。

安装 Hexo 工具：

```bash
npm -g install hexo-cli
```

随后运行 `hexo --version`，若出现以下信息：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213135450.png)

则说明 Hexo 已经安装成功。



## 初始化博客

安装好 Hexo 之后，接着我们使用 Hexo 生成博客源码文件，首先创建博客源码文件存放的目录：

```bash
mkdir -p ~/blog && cd ~/blog
```

博客的目录地址可以是任意目录，这里我放在用户根目录。

初始化博客：

```bash
hexo init ./
```

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213140015.png)

这个步骤可能会因为网络情况，耗时可能会很长，过程中如果出现了 Err 字样的错误问题，说明初始化出错了，这是你需要将目录中所有的文件删除后再试试，科学上网也许能解决这个问题。当然你也可以试试以下这个方法：

```bash
npm config set registry https://registry.npm.taobao.org
```

表示初始化成功的提示：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213140506.png)

在博客源码文件目录下（一定要在当前目录），运行：

```bash
hexo s
```

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213140918.png)

Hexo 会在本地启动一个 Http 服务器。按照提示，我们访问 http://localhost:4000：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213141115.png)

如果能够向上面这样正常打开，则说明博客已经在本地启动成功了！



## 编写文章

可以快速地使用以下命令创建一篇文章：

```bash
hexo new post 'post name'
```

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213141819.png)

如上，我们创建了一篇名为《test-post》的文章，hexo 会将他文章输出到 `{blog_path}/source/_posts/`，也因此我们知道了 Hexo 管理下的博客所有文章都会被放在 `source/_posts` 目录中。

我们使用 Finder 打开目录：

```bash
open ./source/_posts
```

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213142136.png)

这时我们就可以使用 md 编辑器写文章了，这里我强烈推荐 Typora，在我心中是最强 md 编辑器没有之一！

文章开头的格式大致有以下几个选项：

```
---
title: 文章标题
date: 文章的编写日期，格式：Y-m-d H:i
tags: 文章标签，格式：[标签1, 标签2, 标签3，……]
categories: 文章分类，格式：[分类名1, 分类名2, 分类名3, ……]
description: 文章描述
---
```

其中 title 是必要的，其他可省略。

Hexo 的文章模版会放在 ./scaffolds 目录中：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213145501.png)

Hexo 默认有三个 模版，刚刚我们执行的 `hexo new post` 命令使用的默认即 post.md，我们可以使用其它模版：

```bash
hexo new page
```

写好文章之后，运行 `hexo s`：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213142654.png)



## 自定义主题

如果官方自带的主题不符合你的胃口，你可以去 Hexo 主题库 `https://hexo.io/themes/` 寻觅你喜欢的主题，这里我使用 nexT 主题进行演示（经过我对大量主题的使用对比，nexT 是一款非常优秀的主题）：

```
git clone https://github.com/theme-next/hexo-theme-next themes/next
```

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213143514.png)

这里可能需要小等一会。

下载好之后，会保存在 `./themes` 目录中，该目录也是存在自定义主题的地方：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213143804.png)

启用主题：

```bash
vim _config.yml
```

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213143716.png)

将 theme 选项配置为我们刚下好的主题名称（`./themes` 目录中的主题目录名称）。

再使用 `hexo s` 启动博客本地服务器：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213144343.png)

此时可以看到博客主题已经换成 nexT 主题了。

这里需要特别说明一下，在 Hexo 中，一共有两个 `_config.yaml`：

1、./_config.yaml

为 Hexo 博客的配置文件，定义主题样式的根基，博客名称、文章展示规则、主题切换配置、插件配置、部署配置等等都在这里设置。

1、./themes/xxx/_config.yaml

为主题的配置，不同的主题有不同的选项，交由主题设计者自行设置选项，nexT 主题的配置选项特别多，从这方面也可以看出 nexT 主题功能丰富的一面。



## 安装插件

Hexo 最强大的地方在于它的插件体系，它具备丰富的插件，比如访客统计插件，博客本地搜索插件、文章字数统计插件等等。如果你想要扩展你的博客功能，可以去 Hexo 的插件库`https://hexo.io/plugins/`搜索一下，说不定有意外的惊喜。

安装完插件之后，记得在 `./_config.yaml`中配置一下：

```yaml
plugins:
  plugin-name:
  	xxx:
  	xxxx:
```



## 发布博客

前面我们操作了这么多，都只是在本地可以访问我们的博客，因此我们需要将我们的博客发布到具备公网 IP 的服务器上面，但我们可以使用 GitHub Pages 或者 Gitee Pages 进行托管，免去了花钱购买服务器的步骤。以下使用 Gitee Pages 演示。

在发布博客之前，我们需要将博客内容生成一份静态资源文件，使用以下命令：

```bash
hexo generate # 或者 hexo g
```

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213150353.png)

静态文件生成好之后会保存到 `./public` 目录中。

在 Gitee 中，创建一个与 Gitee ID 同名的仓库，例如我的 Gitee 博客仓库 `https://gitee.com/objcoding/objcoding`

还需要在 `./_config.yaml` 中配置部署信息：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213150836.png)

运行以下命令：

```bash
hexo deploy # 或者 hexo d
```

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213151102.png)

这时直接将 `./public` 目录中的文件提交到配置的仓库中。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213151233.png)

这时我们在仓库中找到：服务 -> Gitee Pages：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201213151351.png)

部署好之后，会提示你的 Gitee Pages 的地址，例如我的博客 Gitee Pages 地址是：https://objcoding.gitee.io/。

这时别人就可以通过该链接访问你的博客了。

同理，如果你不想使用 GitHub Pages 或者 Gitee Pages，我需要使用自己买的服务器进行搭建，你只需要将 `./public`目录中的内容拷贝到服务器中，服务器再安装一个 Nginx 服务器配置到该目录即可（`./public` 目录中已经有 index.html）。

这里需要提醒一下大家，我们刚刚只是将  `./public` 中的静态文件 push 到 gitee 仓库中，其它目录是没有任何备份的，需要你自行对其它源码内容进行备份，要不然你的电脑磁盘出问题，文章源文件就丢失了。此时你可以将博客目录也当作一个 Git 仓库，在 Gitee 中创建一个仓库保存即可。

附上我的博客地址：

GitHub Pages 地址: https://objcoding.com/

Gitee Pages 地址: https://objcoding.gitee.io/

云服务博客地址: https://objcoding.cn/