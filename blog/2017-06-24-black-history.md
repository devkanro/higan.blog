---
slug: black-history
title: 旧博客文章存档
authors: higan
tags: [ Life of coder ]
---

作为 MSP 福利，为期一年的 MSDN 订阅到期了，所附带的 Azure 的每月的信用额度也没有了，所以之前跑在 Azure
上面的博客就挂掉了。但是看到推酷上面之前爬过我的文章，所以好歹还有一份存稿，所以这篇文章就把之前所有旧博客的文章做一个索引、备份。
但是 Azure 过期，微软居然没有给什么邮件通知之类的，也是很蛋疼的，导致我的所有的博客文章都没了。当然我没有即时备份也是有责任...OTZ。
<!--truncate-->
# 旧博客文章索引

对这些文章按照主题进行索引。

## UWP 控件开发系列

这个系列算是比较古老的系列了，说是控件开发其实到现在都是一直在讲布局系统。如果要对 XAML 的布局体系进行了解的话，这个系列绝对可以帮助你了解
XAML 中各个 UI 元素是如何随心所欲的进行布局的。

### [UWP下控件开发01 - 布局面板的布局](http://blog.higan.me/uwp-control-development-01/)

本文主要对一个面板的测量与布局方法进行一些基础的说明。

### [UWP下控件开发02 - 进阶瀑布流布局](http://blog.higan.me/uwp-control-development-02/)

本文主要通过实现一个瀑布流的布局的例子来对面板的布局做进一步的讲解。

### [UWP下控件开发03 - 照片墙面板概念](http://blog.higan.me/uwp-control-development-03/)

这是一个挖坑的文章，简要的介绍了 OneDrive 的照片墙的布局，还有很多后续工作并没有在本文中讲述，但是其实这个面板已经写出来了，不过有些复杂，就懒得写文章了。

### [UWP下控件开发04 - 虚拟化面板实现](http://blog.higan.me/uwp-control-development-04/)

这也是一个挖坑的文章，用最简单的实现来构建一个支持 UI 虚拟化的面板，但是其中的效率问题是很严重的，但是在之后的文章中有介绍如何更高效的实现
UI 虚拟化。

### [UWP下控件开发05 - 自适应交错布局](http://blog.higan.me/uwp-control-development-05/)

看到这篇文章想到了一些黑历史，当时还在 OXL 当实习生的时候，为了实现需求就写了这么一个面板，顺便水了博客。但是由于某些不可描述的原因，最后还是没有用上，只用了最基础
ItemsWrapGrid。

### [UWP下控件开发06 - 完善虚拟化方案](http://www.tuicool.com/articles/rm22e2U)

本文是对系列第四篇文章的一个完善，详细了介绍了一个支持虚拟化的面板所需要做的工作，基本上就是简单的实现了一下 WPF
上的虚拟化面板基类所做的事情。

### [UWP下控件开发07 - 虚拟化与瀑布流](http://www.tuicool.com/articles/JbmeqmU)

本文是对上一篇文章的一个补充，将上一篇文章中的虚拟化面板基类进行拓展，支持瀑布流的布局方式。

## UWP 风向标系列

我写博客比较喜欢写一些大家基本都没有写过的东西，尤其是 UWP 风向标系列。这个系列基本都是一些很多人都忽略的一些新功能与一些最新的API。说起这个系列也是（黑）槽（历）点（史）满满...

### [UWP风向标 - 向 Windows 10 商店通用应用中添加矢量图标](http://www.tuicool.com/articles/b6Vviu)

由于 UWP 的特性，需要在不同的分辨率的设备上都保持良好的显示效果，所以这里对比了好几种矢量图标在 UWP 上的实践，并给出了最佳实践。

### [UWP风向标 - 为 Windows 10 商店通用应用构建图标字体](http://www.tuicool.com/articles/VJvIvau)

结合上一篇文章，这篇文章介绍了如何自己生成一个图标字体以供 UWP 使用，对比了好几个生成字体的工具，另外还附带介绍了彩色图标的实践与应用。

### [UWP风向标 - 使用全新的 Composition API 来呈现可视元素与演示动画](http://www.tuicool.com/articles/zymqeiJ)

当时还是 Composition API
刚刚出现，现在感觉好久没有关注了，也不知道到了怎样的地步了。这篇文章干货不多，其实主要是为了介绍 [CompositionHelper](https://github.com/higankanshi/CompositionHelper)。

### [UWP风向标 - Composition API 在 SDK 10586 中的变更](http://www.tuicool.com/articles/zymqeiJ)

本文主要介绍了一些 Composition API 在 10586 版本中的变更，也没多少干货，纯粹水博客。

## 玩转 .NET 线程调度模型系列

这个系列只有两篇文章，主要是为了 [Meta.Vlc](https://github.com/higankanshi/Meta.Vlc) 的不卡顿的播放器做准备的。

### [玩转 .NET 线程调度模型 - Dispatcher](http://www.tuicool.com/articles/zq2Yfq)

本文是为了后面的一篇文章做铺垫，简单的介绍了 .NET 的线程模型，基本都是一些大家都知道的东西。

### [玩转 .NET 线程调度模型 - 多线程窗体与控件](http://www.tuicool.com/articles/vEN3Qbu)

这篇文章是本系列的核心，介绍了如何创建一个运行在不同线程的窗体，更是介绍了如何创建一个运行在同一个窗体但是不同线程的控件，得益于这篇文章的研究，Meta.Vlc
终于可以解决了由于开发人员的低效率的 XAML 代码导致的视频卡顿了。

## 遇见 Kanro 系列

这个系列只有一篇的存档，其实还有三篇并没有公开，因为那个时候 Kanro 还在写，所以变数很大，就没有公开，现在看起来有些后悔了。等
Kanro 完善起来，到时候我会再重新的介绍一下 Kanro 这个东西。

### [Kanro - RESTful 后端框架（01）- 遇见 Kanro](http://www.tuicool.com/articles/M3miE3v)

主要介绍了 Kanro 的思想，与特色，感觉大家都不太看好 Kanro，但是，本来嘛，搭轮子本来就是要自己爽就好了2333。

## 杂项

一些随笔，没有形成系列的文章就放在这里了，随意看看就好了。

### [Azure 架设 TFS 服务器出现 401 解决方案](http://www.tuicool.com/articles/uqMVN3q)

当时记得找了很多地方，最后在一个感觉就像是假的技术文章找到了解决方案，也是挺讽刺的

### [Ghost 博客多说评论插件样式](http://www.tuicool.com/articles/vQfyAfu)

多说已经倒了...很伤心没有评论插件了。

### [如何快速地为控件添加条纹背景(striped background)](http://www.tuicool.com/articles/mUNFRri)

来自好友的投稿，有且仅有的一篇由其他作者发布的文章。

# 博客的进展

懒癌晚期患者，下次更新还不知道是什么时候，而且也没有什么好玩的技术值得分享和探讨的...看看什么时候热情来了，说不定会继续水几篇博客。