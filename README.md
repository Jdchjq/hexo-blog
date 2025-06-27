# 介绍

这是我的博客，基于 hexo 的 anzhiyu 主题构建的，原始主题来自于一个北京的产品设计大佬[HEO](https://blog.zhheo.com/)自己搭建的闭源主题。

# 安装

## 安装 nodejs

```
# Download and install nvm:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
# in lieu of restarting the shell
\. "$HOME/.nvm/nvm.sh"
# Download and install Node.js:
nvm install 22
# Verify the Node.js version:
node -v # Should print "v22.17.0".
nvm current # Should print "v22.17.0".
npm install -g yarn
```

## 安装依赖

```
yarn install
```

## 本地运行

```
yarn build && yarn server
```

# 使用

## 写文章

在 source/\_posts/zh-ch 目录下创建新的 md 文章，同时创建一个同名文件夹，用于放置文章的图片。

文章的封面图一般放在 image 目录下，这些图片路径等配置都可以在文章内容的开头配置里调整。目前这样放置是能正常显示的。

关于多语言，还未深入研究如何兼顾多语言的显示，随着语言的切换能否找到对应的文章。

关于使用 hexo new 自动创建文章，目前自动创建文章所在目录是 source/\_posts/undefined，还未找到解决办法

## 本地调试

`hexo s` 或 `hexo server` 本地运行起来，使用浏览器访问查看效果。

## 发布到线上服务器

> 可以先将代码提交到 github 保存，这样后面在其他设备上开发也能拉取代码。github 与下面的部署无关。

1、执行 `hexo generate` 对本地文件生成静态文件，生成的目标目录是 public。
2、执行 `hexo deploy` 执行部署脚本，当运行成功后，线上博客就更新到最新了。
3、如果线上博客没有更新，执行`hexo clean` 再执行 `hexo generate` 看下能否重新生成最新的静态文件，再继续 delpoy

> 自动发布参考了 Heo 的[部署教程](https://blog.zhheo.com/p/49b7a68d.html)

原理：通过在服务器上部署 git 裸仓库，然后本地将代码推送到裸仓库上，会触发裸仓库的自动脚本 post-receive，脚本做了将收到的最新静态文件拷贝到 nginx 目录下，实现更新。

其中，需要本地的 ssh key 能访问服务器的 git 用户，服务器创建了 git 用户和正常用户。

裸仓库就等同 github、gitlab 那样的一个中央仓库，大家有权限的都可以往中央仓库推送和拉取代码，可以创建多个 git 代码库。
