---
title: 解决 Copilot 登录无限循环的问题
date: 2024-07-03
updated:
keywords:
slug: enablecopilot
cover: /medias/ast.jpeg
top_image:
comments: false
maincolor:
categories:
  - ai 技术
tags:
  - copilot
typora-root-url: ./enableCopilot
description: 如何解决 copilot 被墙的问题
---

## 背景

众所周知，Copilot 的响应速度比 ChatGPT 快很多，并且 Copilot 底层使用的是 GPT4，效果也不差。工作中如果有 Copilot 的帮助就方便很多了。但是 Copilot 在中国大陆地区无法使用，有时开了代理仍然还是无法使用，访问[Microsoft Copilot](https://copilot.microsoft.com/)总是陷入无限登录的"月读"循环。

![](img2342390.png)

最近终于闲下来，研究如何解决这个无法访问的问题。

> 前提，访问 Copilot 必须要用梯子访问，因为 copilot 在中国地区不开放。

## 原因 1：账号地区不正常

在 edge 浏览器登录你的微软账号

如果你无法访问[国际版 bing](<[Bing](https://cn.bing.com/?cc=jp)>)，那么你应该考虑设置一下你的微软账号所在地区，将其改为非中国区域。
点击管理账户，在网页中登录你的微软账号

![](img23545656.png)

点击个人信息，如果弹出验证手机号，则可以关掉弹窗。

点击国家或地区，选择美国或者其他非中国区域，点击保存。

![](img2366943.png)

在 edge 浏览器退出重新登录你的微软账号，再访问[Microsoft Copilot](https://copilot.microsoft.com/)看看是否解决。

## 原因 2：没有正确使用代理

由于 Copilot 在中国大陆不开放，因此需要使用 🪜 才能访问。

但有时候你可能会发现，只有在全局代理模式下才能访问 Copilot，这是因为你的梯子没有将 Copilot 的网址加入代理规则中。

下面我们来看看如何将 Copilot 添加到代理规则中

1、找到并打开你的订阅配置文件
我使用的是 clashx，从“配置”选项中找到打开配置文件夹，找到你目前正在使用的配置文件，使用 vscode 或者其他编辑工具打开该文件

![](img237993470.png)

![](img23363465.png)

2、在相关位置下添加规则

打开配置文件，搜索 `microsoft`，跳到最后一个匹配项，就能看到微软相关域名的代理设置。

方法 1：在第 1428 行，直接将 microsoft.com 改成 Proxy 规则，这样在规则模式下也能命中 copilot.microsoft.com

方法 2：在 microsoft.com 规则前面添加一行

```

- DOMAIN-SUFFIX,copilot.microsoft.com,Proxy

```

![](img2384563456.png)

上述两种方式都能让你在规则模式下访问 copilot

保存文件后，记得将你的配置文件更新频率设置长一些（设置为一年两年都行），否则下次自动更新配置文件，就会将你的更改覆盖掉。

## 总结

经过上述方式，终于可以在办公的时候随便使用 Copilot 了
