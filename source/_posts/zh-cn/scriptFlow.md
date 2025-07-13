---
title: 低代码工作流脚本平台- windmill
date: 2025-06-22 18:30:22
updated: 
keywords: 
slug: scriptFlow
cover: /image/windmill.png
top_image: 
comments: false
maincolor: 
categories:
  - 后端开发
tags:
  - 工作研究
typora-root-url: ./scriptFlow
---

最近发现工作中有很多写脚本的需求
其中有一个需求，背景是：
运营侧需要给一批本地图片数据使用目前的模型检测切图逻辑来切图，得到的结果送往标注同事去处理。
由于操作走生产环境那一套 AI 识别过程，会消耗整个识别流程的服务资源。
所以给到我希望能直接对这批数据按线上流程的逻辑来过模型检测，这样就不需要走剩下的流程就能直接拿到切图结果。

然而，还有一些时不时的新需求，比如特定的数据查询和报表导出，在系统上做一些批量处理的创建操作等等...
需求太零散，每次也就只能临时写脚本去处理。

同时也催生一个问题，面对这些零散的需求，没有前端、产品的设计参与，直接由开发做到脚本集成，那么有没有一种平台能够帮助搭建这些脚本，支持管理和丰富的自动化运行场景呢？

答案是，有的

一些主流的低代码工作流平台，能支持运行多种语言的脚本，不需要考虑运行环境。同时还支持集成多种触发器。

我选中的是 [windmill](https://www.windmill.dev/docs/intro) 平台，支持免费的本地部署，使用 docker 或 k8s 运行脚本。
主要优点如下：
1、通过低代码方式配置 UI 界面显示输入输出。
2、支持多种触发器，如定时触发、通过 webhook 网络请求触发、邮件触发等。
3、启动速度快，支持丰富的开发语言如 ts、python、go、php、c#、sql 等。
4、支持多个脚本组合起来形成工作流

下面是我在安装和使用过程遇到的一些问题总结

## 安装

相关的自管理本地安装方式在这：[传送门](https://www.windmill.dev/docs/advanced/self_host)
由于我遇到的脚本需求不是很频繁，因此使用 docker 部署的方式，在单服务器上就足够多任务执行。除此之外还可以使用 k8s 的安装方式

通过 Docker compose build 运行成功后，访问服务面板，并根据文档提示登录进去就能看到管理主页了

![](windmill_homepage.png)

> 这里我设置 caddy 的映射端口为 5501，避免使用默认的 80 端口

## 写脚本

在愉快写脚本过程中发现几个问题，经过几天摸索总结如下：

### 问题 1：脚本如何拉取私有仓库代码

脚本总会需要引用自己公司的私有仓库代码，避免重复造轮子。我使用的是 go 作为脚本开发语言，这里给出我总结的在 mac 环境和 linux 环境下的配置

前提：你的 docker 运行的机器，要有能访问私有仓库的 ssh 权限

#### mac 环境

打开 docker-compose.yaml 文件，找到 windmill_worker 配置，只需要在环境变量和挂载卷加几项配置即可

```yaml
windmill_worker:
    image: ${WM_IMAGE}
    pull_policy: always
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "1"
          memory: 2048M
          # for GB, use syntax '2Gi'
    restart: unless-stopped
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - MODE=worker
      - WORKER_GROUP=default
      # 添加以下Go 相关配置
      - GOPRIVATE=git.xxxxx.com  # 这里替换成你的私有仓库
      - GOPROXY=https://goproxy.cn,direct
      - SSH_AUTH_SOCK=/tmp/ssh/ssh-agent.sock   # 修改worker 的ssh agent为挂载后的agent.sock
    depends_on:
      db:
        condition: service_healthy
    # to mount the worker folder to debug, KEEP_JOB_DIR=true and mount /tmp/windmill
    volumes:
      # mount the docker socket to allow to run docker containers from within the workers
      - /var/run/docker.sock:/var/run/docker.sock
      - worker_dependency_cache:/tmp/windmill/cache
      - worker_logs:/tmp/windmill/logs
      # git
      - ~/.gitconfig:/root/.gitconfig
      # monut mac ssh-agent
      - /run/host-services/ssh-auth.sock:/tmp/ssh/ssh-agent.sock
    logging: *default-logging
```

注意到，我在 environment 和 volumes 配置中加了一些新东西
1、配置 GOPRIVATE 为私有仓库的地址，配置 GOPROXY 为国内可访问的 github 仓库代理
2、将 SSH_AUTH_SOCK 变量重新定义为自定义的 ssh sock 地址。

> 这一步操作是为了使 docker worker 内部能直接使用宿主机的 ssh 代理，通过宿主机的 ssh 权限去访问私有仓库代码。

3、挂载宿主机的 git 配置，主要是为了能获取到 git 拉取仓库的方式，使用 ssh 而不是 http.

```
[user]
	name = xxxxx
	email = xxxxx
[http]
	proxy = http://127.0.0.1:7890
[https]
	proxy = http://127.0.0.1:7890
[url "ssh://git@git.xxxxx.com/"]
	insteadOf = https://git.xxxxx.com/
```

宿主机的 ~/.gitconfig 文件应该上上述样子，主要是配置将 ssh:// 替换 https://
如果你的宿主机没有配置，则在宿主机上执行 `git config --global url."ssh://git@git.xxxxx.com/".insteadOf "https://git.xxxxx.com/"`
其中 git.xxxxx.com 替换成你的私有仓库地址

4、挂载 mac 的 ssh-agent sock 到 docker worker 中
你会发现，为什么挂载的是 /run/host-services/ssh-auth.sock 路径？
这是 mac 开放用于 ssh 挂载的路径，实际的 ssh-agent.sock 并不能支持挂载

> 这里提一下，为什么直接挂载 ssh-agent，而不是将 ~/.ssh/ 目录挂载进 worker 实例。
> 直接挂载 ssh-agent 明显安全性更高

#### linux 环境

windmill_worker 配置如下：

```yaml
windmill_worker:
    image: ${WM_IMAGE}
    pull_policy: always
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "1"
          memory: 2048M
          # for GB, use syntax '2Gi'
    restart: unless-stopped
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - MODE=worker
      - WORKER_GROUP=default
      # 添加以下Go 相关配置
      - GOPRIVATE=git.xxxxx.com  # 这里替换成你的私有仓库
      - GOPROXY=https://goproxy.cn,direct
      - SSH_AUTH_SOCK=$SSH_AUTH_SOCK   # 修改worker 的ssh agent为挂载后的agent.sock
    depends_on:
      db:
        condition: service_healthy
    # to mount the worker folder to debug, KEEP_JOB_DIR=true and mount /tmp/windmill
    volumes:
      # mount the docker socket to allow to run docker containers from within the workers
      - /var/run/docker.sock:/var/run/docker.sock
      - worker_dependency_cache:/tmp/windmill/cache
      - worker_logs:/tmp/windmill/logs
      # git
      - ~/.gitconfig:/root/.gitconfig
      # monut mac ssh-agent
      - $SSH_AUTH_SOCK:$SSH_AUTH_SOCK
    logging: *default-logging
```

和 mac 的配置差不多，只有 ssh agent 的配置发生变化


### 问题2：如何在本地IDE写脚本

直接参考[官方文档](https://www.windmill.dev/docs/cli_local_dev/vscode-extension)，通过 安装 vscode 插件，配置好即可运行


#### 本地安装windmill 工具

```
npm install -g windmill-cli
export HEADERS=header_key:header_value,header_key2:header_value2
wmill --version
```
如果输出版本，则安装成功

配置你的远程 windmill 服务和workspace
```
wmill workspace add [workspace_name] [workspace_id] [remote]
```

找一个开发文件夹，使用 `wmill sync pull` 拉取你的自建windmill 服务上的脚本，最后你写完的代码可以使用 `wmill sync push` 推送保存到 windmill 服务

另外，这个开发文件夹，还可以初始化成git 仓库，这样就能推送到git仓库，甚至可以设置git push 后，执行webhook 自动执行 wmill sync push。
这个功能比较简单就不展开说了

#### 配置vscode

在插件市场搜索安装windmill 插件，然后根据提示，cmd+shift+p 打开设置，搜索 Windmill: Configure remote, workspace and token
进行以下相关配置：

![](vscode_config.png)

这里我配置了好几次都无法成功访问到windmill 服务，后来我在settings.json 中把windmill 相关配置全清了，再重新配置就OK了


#### 编写和运行脚本
可参考文档：[传送门](https://www.windmill.dev/docs/advanced/local_development)
vscode 打开前面创建的开发文件夹，里面应该有从windmill 拉取的所有配置文件。
创建本地脚本：
```
wmill script bootstrap u/work/newScript.go go
```
这里的 u/work/newScript.go 路径是我windmill 的脚本路径，可以根据需要改成自己的脚本路径
这样就能新建一个脚本，打开newScript.go文件编写就行

运行脚本时，在vscode 打开这个脚本文件，使用快捷键 cmd+enter 就能打开执行预览窗，操作脚本的运行。
> 如果快捷键不行，打开配置搜索 Windmill: Run preview 也能运行

![](vscode.png)

可以看到，在vscode 也能操作和页面一样的动作来测试和运行脚本。

测试没问题后，直接操作 `wmill sync push` 推送即可



## 总结

本文介绍了从安装配置到使用的过程，以及一些问题的解决。
在使用体验下来，windmill 确实能提供很大的便利性，只要做好脚本提交到平台，把脚本连接发送给运营同事，他们提交一些输入就能触发脚本。
后续重复的使用也不需要开发同事介入