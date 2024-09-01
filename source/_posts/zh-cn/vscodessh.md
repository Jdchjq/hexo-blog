---
title: vscode ssh远程的github权限问题
date: 2024-08-12
updated:
keywords:
slug: vscodessh
cover: /image/vscodessh.png
top_image:
comments: false
maincolor:
categories:
  - 日常开发问题
tags:
  - vscode
typora-root-url: ./vscodessh
description: 解决vscode在使用ssh登录远程服务器的时候，使用 git 总是需要密码的问题。
---

# 前言

由于我在工作中经常要使用 vscode ssh 进行远程开发，在远程进行 git 操作的时候，经常会出现如下 git 权限失效的示例问题

```bash
[cjd@gz-cs-2-65 tempProject]$ git pull
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

正常使用 ssh 进行登录，只要客户端主机的 ssh key 有 gitlab 仓库权限，是可以在服务器上利用 ssh 的 forward agent 机制正常访问 gitlab 仓库的。
只需要进行如下的配置：

> 前提是你在本地已经生成了 ssh key 密钥对，并将公钥加入到 gitlab 账户下

1、设置开启 ssh 代理
在 `~/.ssh/config` 文件中配置 ssh 连接，示例如下：

```
Host 49
  HostName 172.26.2.49
  User cjd
  ForwardAgent yes
```

在终端使用 `ssh 49`，就可以连接到远程服务器。

2、启动 ssh 代理服务，将 ssh key 加入高速缓存中
对于 windows:
在 windows powershell 下执行以下命令(以管理员权限运行)，可以实现每次在系统启动时都会将 ssh key 加入到计算机缓存中

```bash
Get-Service ssh-agent | Set-Service -StartupType Automatic -PassThru | Start-Service
```

对于第一次使用，可以执行以下命令，就可以直接在当前实现，不需要重启了

```bash
start-ssh-agent.cmd
```

对于 mac:
鉴于 mac 不会经常关机，可以使用以下命令，就可以直接将 ssh key 加入 ssh agent 中

```bash
ssh-add
```

现在使用 ssh 登录远程，应该就可以正常使用 git，可以使用以下命令测试：
在本地：

```bash
ssh <user>@<ip> -T git@github.com
```

# ssh key 失效问题

前面提到的 ssh key 配置远程使用无需密钥的方法，有时候在 vscode 连接远程主机进行开发的时候，会突然失效，但是使用终端登录远程使用 git 又是正常的。

在开发的时候遇到就会非常影响开发效率

为此我总结了一些解决办法，方便我在突然遇到的时候能及时处理。

先说结论，ssh agent 突然失效的问题，主要是 vscode 处理 ssh socket 时的 bug。在重新登录远程时，复用了旧的 ssh socket 导致失效。

以下是我尝试过的一些解决办法，但是由于 vscode 的版本不同，有些方法已经失效。

1、重载扩展宿主

使用 Cmd/ctrl+shift+p 打开运行窗口，输入 restart extension host ，选择重载扩展宿主，一般这时候重新加载之后，git 就能正常使用了。

但新的版本可能会出现远程连接断联的情况，这就需要尝试其他方法。

2、配置 vscode 远程服务器

打开 vscode 设置界面，搜索以下的配置选项，配置成对应的值。

```
{
    "remote.SSH.enableAgentForwarding": true,
    "remote.SSH.useLocalServer": false,
    "remote.SSH.useExecServer": false,
    "remote.SSH.remoteServerListenOnSocket": true,
}
```

这个方式是参考 github 上的[讨论](https://github.com/microsoft/vscode/issues/168202#issuecomment-2147925134) 得到的，vscode 的 windows 版本是 1.89.0.0，亲测可以正常解决突发状况。

同时可以看到这个 github issue 还是 open 状态的，希望 vscode 可以尽快解决这个 bug
