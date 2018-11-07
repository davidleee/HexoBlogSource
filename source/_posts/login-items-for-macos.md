---
title: 怎么在命令行给 macOS 设置开机启动任务
date: 2018-11-07 16:16:12
tags:
- macOS
- Launch Daemons
- Launch Agent
- plist
- launchctl
---

最近通过 docker + nginx 在 一台 Mac mini 上面配了个简易文件系统，好让其他人能方便地下载一些预先写好的配置文件，这部分就不细说了。文件系统跑起来之后，考虑到便利性，想要再添加一个开机自启动的逻辑，让这个文件系统在电脑重启之后也能自己跑起来。

在 Linux 系统下面，我们可以通过 `systemctl`  或者直接修改 *rc.local* 文件
来实现启动项的添加。但是这一套在 macOS 上面玩不转了，因为我们需要通过一个完全不一样的机制—— **Launch Daemon** 来实现这个功能。

<!--more-->

## 要做的事情
因为 macOS 的启动项是通过一个 plist 去配置的，配置一个脚本远比配置一段要执行的命令行指令要简单，所以这里采用脚本的方式去实现。

于是我们要做的事情只有两步：
1. 创建一个脚本文件去执行 docker-compose 的启动指令
2. 让这个脚本在系统启动的时候执行（不需要用户登录）

## 开始吧
### 创建脚本
先创建一个脚本文件：
```bash
vim startup.sh
```

不考虑异常情况，就是简单地进到 docker-compose.yaml 所在的目录，然后执行一下启动命令：
```shell
#!/bin/bash

cd /Users/davidleee/Desktop/docker-nginx

# make sure it runs
while [ $(docker inspect -f '{{.State.Running}}' docker-nginx_nginx_1) != "true" ]
do
  echo "Launching file-service with docker-compose..."
  docker-compose up -d
  sleep 2
done
```

> 为了避免我们执行 `docker-compose` 的时候 docker 自己还没有跑起来，所以用一个循环去检测我们的服务是不是真的启动了。
>
> 另外还要记得把上面的 `docker-nginx_nginx_1` 改成你真正的的容器名称。 

别忘了给脚本加上执行权限：

```bash
chmod +x startup.sh
```

### 配置启动项
现在我们要把上面的脚本添加到 **Launch Daemon** 里面去。

在此之前，让我们先理清一些概念。

macOS 通过一系列的 plist 文件来配置启动项，这些 plist 根据存放位置的不同而分为 **Launch Daemon** 和 **Launch Agent**。它们的区别在于，Agent 是在用户登录之后以该用户的身份去执行的任务，而 Daemon 是以根用户或 `UserName` 里指定的用户去执行的任务。

它们一般存放在这两个地方：
```bash
/Library/LaunchDaemons

/Library/LaunchAgents/
```

我们这次的任务需要用到 root 权限，所以我们将会在 LaunchDaemons 里创建一个配置文件：
```bash
sudo vim /Library/LaunchDaemons/com.file-service.plist
```

然后在里面填上以下内容：（注释部分可以去掉咯）
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" " [http://www.apple.com/DTDs/PropertyList-1.0.dtd](http://www.apple.com/DTDs/PropertyList-1.0.dtd) ">
<plist version="1.0">
  <dict>
<!-- Launch Daemon 不一定有权限访问所有需要的环境变量
	在没有权限的时候，启动项执行会失败，所以我们在这里配置一下脚本需要的环境变量 -->
    <key>EnvironmentVariables</key>
    <dict>
      <key>PATH</key>
      <string>/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:</string>
    </dict>
    <key>Label</key>
<!-- 习惯上，我们会用一个 identifier 样式的名字来作为启动项的名称 -->
    <string>com.file-service</string>
    <key>Program</key>
<!-- 要执行的脚本的绝对路径 -->
    <string>/Users/davidleee/Desktop/docker-nginx/startup.sh</string>
<!-- 这个 key 告诉系统在启动的时候执行我们的脚本
  对于 daemons 来说是系统启动之后，对于 agent 来说则是用户登录之后 -->
    <key>RunAtLoad</key>
    <true/>
<!-- 判断是按需启动我们的启动项，还是永远运行下去
	现在我们自己跑的是自己的脚本，按需启动就可以了 -->
    <key>KeepAlive</key>
    <false/>
    <key>LaunchOnlyOnce</key>        
    <true/>
<!-- 在调试脚本的时候很好用，可以指定脚本正常/错误输出的路径 -->
    <key>StandardOutPath</key>
    <string>/tmp/startup.stdout</string>
    <key>StandardErrorPath</key>
    <string>/tmp/startup.stderr</string>
    <key>UserName</key>
<!-- 执行脚本的用户 -->
    <string>davidleee</string>
  </dict>
</plist>
```

最后就是把这个 plist 加载到 launchctl 里面去了：
```bash
# `-w` 会把 plist 永久添加到 Launch Daemon 里面
sudo launchctl load -w /Library/LaunchDaemons/com.file-service.plist

# ...如果你不想让它自启动了
sudo launchctl unload -w /Library/LaunchDaemons/com.file-service.plist
```

## 写在最后
在执行完上面的 `launchctl load` 指令之后，plist 里面配置的脚本会马上被执行，你可以通过 `launchctl start` 和 `launchctl stop` 来控制它的开关，不过我们这里只是执行了一个脚本，并不会像其他应用那样长驻，所以其实也就没有“开关”一说了。

## 参考文章
* [Adding Startup Scripts to Launch Daemon on Mac OS X Sierra 10.12.6](https://medium.com/@fahimhossain_16989/adding-startup-scripts-to-launch-daemon-on-mac-os-x-sierra-10-12-6-7e0318c74de1)
* [Creating Launch Daemons and Agents](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)
