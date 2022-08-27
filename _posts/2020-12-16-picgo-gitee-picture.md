---
layout: post
title: "使用 PicGo + Gitee 搭建免费图床"
categories: PicGo Gitee
tags: PicGo Gitee 图床 Typora
author: 张乘辉
---

* content
{:toc}
前面我写了一篇 「[使用 Hexo + Gitee 快速搭建属于自己的博客](https://mp.weixin.qq.com/s/2VDKw4rfm73W8xEaTuqnYQ)」，很多人问起如果使用 md 写文章，图片如何快速地插入 md 文件中，我们都知道，md 格式与富文本格式不一样，md 的需要插入图片的访问地址，如果图片在本地，那么可以使用图片的本地存放地址，但如果你将 md 文件发给别人之后，图片的链接就失效了，这时我们就需要将图片存放在一个大家都能访问的地方，将这个地方的链接插入 md 文件即可，这就是图床。

但问题又来了，每次插入一张图片，我们总是要先将图片上传到图床，获取链接之后再将链接插入到 md 文件中，这个过程过于繁琐，且每次插入都在做重复的工作，今天我就跟大家分享一下，我是如何使用 PicGo 图床工具高效地解决上面的问题。









首先我们需要在本地安装 PicGo，PicGo 下载链接：https://github.com/Molunerfinn/PicGo/releases/

PicGo 本身支持很多图床，比如阿里云、七牛等等，但这些都需要钱，使用 GitHub 虽然免费但是访问速度太慢。

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201215221059.png)

这时我们又想到了 Gitee，但 PicGo 本身不支持，需要安装第三方图床插件，于是我们打开插件设置，搜索 gitee，安装 gitee 插件：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201215220920.png)

PicGo 更多插件可以在这里找到：https://github.com/PicGo/Awesome-PicGo

登录 Gitee，然后创建一个仓库，接着在个人设置中生成一个私人令牌，紧接着我们回到 PicGo，在 Gitee 图床设置栏中找到 Gitee 图床，进行相关设置：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201215171502.png)

- owner：你的 Gitee ID；
- repo：你刚刚创建的那个用于保存图片的仓库名称；
- path：你需要将图片保存到仓库哪个目录中，如果在根目录就不需要填写；
- token：刚刚在个人设置中生成的私人令牌；
- message：默认即可。

设置好之后保存，并且设置为默认图床。这时我们就可以使用 PicGo 将图片上传到 Gitee 仓库中并且返回图片链接了：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201215223720.png)

但是每次都要在这个页面进行上传操作，不是很方便，我们可以设置一个上传快捷键：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201215171636.png)

这样，你截图之后，再通过快捷键即可将图片上传到 Gitee 了，然后你就可以通过粘贴快捷键，快速地将图片以 md 图片链接的形式粘贴到你的文中：

```
![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/20201215220920.png)
```

可以作一下对比：

 md：截图 -> 快捷键上传图片 -> 粘贴图片链接

富文本：截图 -> 粘贴

通过 PicGo 图床工具，我们几乎可以做到与平时我们复制粘贴图片那样方便。

如果此时你使用的 md 编辑器是 typora，还在 typora 中设置 PicGo 自动上传图片：

![](https://raw.githubusercontent.com/objcoding/md-picture/master/img/image-20201215233937471.png)

如上图的设置，截图之后，直接在 typora 编辑器中粘贴即可自动将图片上传至 Gitee，并且自动包装成 md 图片链接的形式。通过这个设置，与我们平时复制粘贴图片的方式就没有任何区别了！

PicGo + Typora + Gitee，简直就是程序员写文章的三大利器！

