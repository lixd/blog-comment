---
title: "我的第一篇文章"
subtitle: "副标题"
date: 2020-03-04T15:58:26+08:00
lastmod: 2020-03-05T15:58:26+08:00
draft: false
description: "文章的描述"
# images 是用于 Open Graph 的，也是填相对路径。
images: [../../img/hexo.png]
tags: [k8s,k3s]
categories: [cloud native]
---

前面是摘要

<!--more-->

后面是正文

> 如果去掉 `<!--more-->` 会自动截取前 70 字作为摘要。



## 官方文档

[LoveIt 使用文档](https://hugoloveit.com/zh-cn/categories/documentation/)



## 注意事项

### 图片

1. 图片默认使用本地资源，相对路径

比如，这样配置

![](../../../img/jinx.png)

最终访问的 URL 为`http://localhost:1313/img/jinx.png`

> 图片是放在 statics 目录的，hugo 编译时会移动到根目录，所以这样可以用，不过缺点是本地看 markdown 的时候加载不了。



### typeit Shortcode

支持一些 shortcode，比如下面这个 typeit ，用于呈现一个手敲代码的动画。

{{< typeit code=java >}}
public class HelloWorld {
    public static void main(String []args) {
        System.out.println("Hello World");
    }
}
{{< /typeit >}}



### music Shortcode

然后还可以贴一些音乐链接

{{< music auto="https://music.163.com/#/playlist?id=60198" >}}



### bilibili Shortcode

甚至是一个 b 站视频

 {{< bilibili BV1Sx411T7QQ >}}

> 只需要指定 bv 号即可

如果视频分为多部分，还需要指定 p

{{< bilibili id=BV1TJ411C7An p=3 >}}
