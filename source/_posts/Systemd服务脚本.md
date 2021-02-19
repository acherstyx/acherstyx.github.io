---
title: Systemd服务脚本
date: 2020-07-07 02:13:11
updated: 2020-07-07 02:13:11
tags:
 - Linux
categories:
 - Linux
---

## 序

现在大多数发行版已经切换到了使用systemd进行服务管理，原本的service已经逐渐被废弃。

这次来整理下systemd下服务脚本的写法。

<!--more-->

## 路径

systemctl的脚本存储在以下两个路径中：

- `/usr/lib/systemd/system`： 系统服务，运行具有管理员权限。

- `/usr/lib/systemd/user `：用户服务，只有在用户登陆之后才开始运行。

一般自定义的服务都以`*.service`结尾，存放在`/usr/lib/systemd/system`中即可，另一个使用比较少。

## 脚本结构

一个服务脚本由三个部分组成：`[Unit]`、`[Service]`和`[Install]`。

- `[Unit]`：主要包含服务的说明。
- `[Service]`：定义了服务的主要内容（核心）。
- `[Install]`：

```service
[Unit]
...

[Service]  
...

[Install]   
...
```

## 参数字段[^five]

### [Unit]

```service
[Unit]
Description=test	# 对于服务的说明文本
After=network.target	# 表示服务需要在哪些服务之后启动
Before=xxx.service		# 表示服务需要在哪些服务之前启动
Requires=xxx.service	# 表明服务之间的强依赖关系
Wants=xxx.service		# 表示服务之间的弱依赖关系
```

**Requires - 强依赖**

在当前服务启动的时候，Requires列表下的服务也会被一起启动；如果Requires列表下某一个服务被停止，或者启动失败了，这一个服务也会被同时停止。所以Requires会产生很强的依赖关系。

Requires不能保证服务之间执行的顺序，当前服务和Requires中的服务可能同时被激活运行。

> i.e. If a unit foo.service requires a unit bar.service as configured with Requires= and no ordering is configured with After= or Before=, then both units will be started simultaneously and without any delay between them if foo.service is activated.[^one]
>
> 在foo.service中定义了Requires=bar.service，表明foo.service依赖于bar.service，但是在启动的时候foo.service和bar.service完全可能同时开始运行，中间没有任何延迟。

所以如果foo.service中定义了Requires=bar.service，启动的过程应该如下：

1. 检查`bar.service`是否启动。
2. 如果`bar.service`没有启动，则启动bar.service。
3. 如果`bar.service`已经启动（或经由2启动了），则可以启动`foo.service`。
4. 如果在`foo.service`运行过程中`bar.service`终止了，则同时终止`foo.service`。
5. 反过来`bar.service`终止了，对`foo.service`没有影响。

**Wants - 弱依赖**

Wants的行为介于Requires和After之间。在一个服务被启动的时候，Wants列表下的服务也会被启动，但和Requires的区别是Wants下的服务即使启动失败、或者运行中出现了问题，也不会反过来影响当前服务的继续执行，因此被作为弱依赖。

**After和Before**

两者可以保证服务之间启动的顺序性，定义之后可以保证服务在某一个服务启动之后/之前启动。和Requires不同的是，After不会产生依赖关系，所以虽然在启动的时候要求After列表中的服务先启动，但是之后即使After列表中某一个服务停止了，当前服务也依然会继续运行。

注意在服务终止的时候，会依照相反的顺序终止。

> i.e. if a unit is configured with After= on another unit, the former is stopped before the latter if both are shut down.
>
> 如果foo.service中设定了After=bar.service，则在两个一起被终止的时候，foo.service会先于bar.service终止。

比阿另外如果两个服务存在着After或Before中任意的关系，在一个服务被shutdown而同时另一个服务在start的时候，shutdown永远先执行。而当没有After或Before的时候，会是同时的。

同样的在foo.service中定义了After=bar.service，形式化描述这一个启动过程应该是这样的：

1. 检查`foo.service`是否启动。
2. 如果`foo.service`启动了，则可以启动`bar.service`。
3. 如果`foo.service`没有启动 ，则一直等待。
4. 如果在`bar.service`或`foo.service`都启动了之后，某一方出现问题终止了，则都不会影响另一方。

### [Service]

```service
[Service]
Type=forking	# 代表了服务的运行模式
User=user		# 用户
Group=user		# 用户组
KillMode=control-group	# 指定systemd在停止服务时该怎么做
PIDFile=/usr/local/test/test.pid	# 提供pid文件的绝对路径
Restart=always		# 重启规则
RestartSec=10		# 重启之前等待的秒数
ExecStart=/usr/local/test/bin/startup.sh	# 执行内容
PrivateTmp=true		# 分配独立的临时空间
```

**Type - 运行模式**[^three]

运行模式的关键词和效果主要有以下这些：

| Type    | 效果                                                         |
| ------- | ------------------------------------------------------------ |
| simple  | 默认值。使用这一类型会使得systemd认为你的服务不会fork，所以认为服务是立即执行的，当前进程的结束即代表了整个服务程序结束运行，会识别为退出状态。所以这一类型适合于那些只有单进程，会持续运行直至主进程退出的程序。[^four] |
| forking | 一般服务都是使用这一个类型。这一模式下systemd会认为当前的服务会进行fork，所以主进程的退出不一定就意味着整个服务已经结束，因为服务可能作为子进程在后台正常运行着。使用这一类型应该同时指定`PIDFile=`，以便systemd能够跟踪主进程。 |
| oneshot | 适用于只需要执行一次的服务、随后立即退出的服务。另外为了让systemd在服务退出之后仍然认为服务处于激活状态，需要同时设置，`RemainAfterExit=yes`。 |
| notify  | 和simple相同，除了约定在服务就绪之后会向systemd发送一个信号，通知的实现由 libsystemd-daemon.so 提供。 |
| dbus    | 使用这一方式启动的时候，systemd通过指定的`BusName`是否出现在了`DBus`总线上来判断服务是否就绪。 |

**User和Group - 执行身份**

指定服务运行的用户和用户组。

**KillMode - 如何停止**

（待补充）

**Restart - 重启规则**

决定了服务以怎样的模式重新启动。

| Restart     | 效果                                           |
| ----------- | ---------------------------------------------- |
| no          | 默认。不会重启。                               |
| on-success  | 只有在服务正常退出（return 0）时才重启。       |
| on-failure  | 在程序以任何非正常的方式退出时重启。           |
| on-abnormal | 只有在被信号终止或超时时，才重启。             |
| on-abort    | 只有在收到没有被捕捉到的信号而终止时，才重启。 |
| on-watchdog | 只有在超时退出时，才重启。                     |
| always      | 不管什么原因，重启就对了。                     |

**各种Exec*字段 - 执行程序**

定义在服务的各个阶段分别执行什么指令，有以下这些：

| Exec*         | 执行阶段   |
| ------------- | ---------- |
| ExecStart     | 启动服务   |
| ExecReload    | 重启服务   |
| ExecStop      | 停止服务   |
| ExecStartPre  | 启动服务前 |
| ExecStartPost | 启动服务后 |
| ExecStopPost  | 停止服务后 |

### [Install]

```service
[Install]   
WantedBy=multi-user.target
```

**WantedBy**[^six]

用于表示服务所属的target，从而配置开机启动。一般常用的只有`multi-user.target`和`graphical.target`（依赖于`multi-user.target`）。

## 相关参考

[^one]:关于`After=`和`Requires=`区别的讨论：[In systemd, what's the difference between After= and Requires=? - Server Fault](https://serverfault.com/questions/812584/in-systemd-whats-the-difference-between-after-and-requires)
[^two]: 在一个服务因为依赖出问题而停止了之后，如何在依赖恢复后恢复运行：[systemd unit stops when required unit is stopped, but then does not start again when required unit starts - Server Fault](https://serverfault.com/questions/914181/systemd-unit-stops-when-required-unit-is-stopped-but-then-does-not-start-again)
[^three]: 啥都有的Arch Wiki：[systemd (简体中文) - ArchWiki](https://wiki.archlinux.org/index.php/Systemd_(简体中文))
[^four]: 关于simple和forking模式的区别：[redhat enterprise linux - systemd forking vs simple? - Super User](https://superuser.com/questions/1274901/systemd-forking-vs-simple/1274913)
[^five]: 一篇系统性介绍的文章：[Systemd 入门教程：实战篇](https://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-part-two.html)
[^six]: 一篇介绍了些参数的文章：[Centos7 自定义systemctl服务脚本](https://www.cnblogs.com/wang-yc/p/8876155.html)