---
title: '开机那几秒，systemd 在屏幕上长成一棵树'
date: 2026-11-12T19:31:00+08:00
tags: [Linux, systemd, 终端, 开源, 小工具]
categories: 折腾
---

![bootvine 的开机画面：黑底终端里一棵 systemd 单元树，红色失败面板写着 nvidia-powervd 退出，下方是彩色时间线条](/images/2026-11-12-systemd-boot-tree.jpg)

今晚刷 Hacker News 首页，第一条叫 bootvine。作者 Willem Vanhulle 把它放在 Codeberg 上，十月十一日建的仓库，到今天刚一个多月，已经攒了一百二十颗星、四个 fork。

它干的事一句话说得清：把 Linux 开机时那面飞过去的日志墙，换成一棵实时生长的树。

上面这张是我从它的演示录屏里抽的一帧。左边那棵树是 systemd 的单元（unit），谁在等谁一目了然；右边红色面板写着一个叫 nvidia-powervd 的单元挂了，连原因带最后六行 journal 都摆在那儿；最下面那条彩色时间线是每个阶段和每个慢单元占了多少秒，从内核启动那一刻算起。

我第一反应是：这不就是把 systemd-analyze 画得好看点？翻完它的架构文档才发现没这么简单。它不是事后去解析日志文件，而是开机那一刻就插了进去——systemd 刚作为 PID 1 起来，它就通过 D-Bus 订阅 org.freedesktop.systemd1 的信号，订阅完立刻 ListJobs 把已经排队的活补齐，再跟着 JobNew、JobRemoved 一条条把树重画。initrd 阶段 systemd 还没起 D-Bus 守护进程，它就直接连 systemd 的私有 socket /run/systemd/private，所以能早到内核起来大概一秒就露面，连 early 的 udev 和第一波挂载都赶得上看。

树是从哪儿长出来的？systemd 每个单元都写着 After= 和 Before=，谁必须排在谁前面全在这些依赖里，它就把还没干完的单元按依赖一层层挂下去，milestone（sysinit.target、basic.target、multi-user.target、graphical.target）撑开几级枝桠。挂载和设备那几行还会把单元名换成人类能懂的描述，比如写 Initrd Root Device，而不是 /dev/disk/by-label/nixos 这种一长串。同一棵树上，菱形是当前正在干的阶段，空菱形是还没到的，方括号里转圈的是正在跑，卡住超过五秒还会多出一列倒计时。

我最喜欢的是失败面板那套讲究：一个单元如果只是因为它依赖的东西挂了才跟着失败，会在树里暗红显示，却不进面板——面板只留给真正坏掉的那个，省得你顺着依赖关系一层层往上找。密码输入也做得漂亮：LUKS 解锁的对话框浮在一棵暗下来的树上面，输错会弹回一句 Wrong passphrase, try again。关机的时候树还会整个倒过来长，根变成 poweroff.target，正在停的服务挂在必须先停的东西下面，卡住关机的那个沉在最底边。从休眠恢复更妙：内核冻结了所有进程，它也跟着停住，只在中间留一条带子显示内核自己的镜像载入进度。

要求写得也很老实：systemd 得是 246 以上且当 PID 1，要 libdbus 和 libsystemd，要虚拟终端，最好有 systemd 版的 initrd——用 initramfs-tools 那种就不行，因为里面没有它能跟随的单元。想不动自己的系统先看看，Nix 用户可以直接跑 `nix run git+https://codeberg.org/wvhulle/bootvine#sample-vm-slow-services`，开一台塞了慢服务和坏服务的样例虚拟机。

作者自己写起因的时候说得很直白：大多数发行版的开机画面是一堵滚得太快的日志墙，你从来不知道到底在发生什么。他被这个逼着写了 bootvine，灵感来自 nix-output-monitor。最新版本 0.1.3 是十一月五号发的，还修了两处挺要命的毛病——不再把 systemd 已经收走的活误报成失败，也不再在开机结束前就把状态栏抢回来。

晚上最舒服的时刻，往往就是按下电源之后盯着屏幕的那几秒。把这几秒从一堵看不清的墙换成一棵会生长的树，忽然就有点想给自己的机器也装一个了。

（配图取自 bootvine 项目的官方演示录屏，作者 Willem Vanhulle，项目以 GPL-3.0-or-later 发布：https://codeberg.org/wvhulle/bootvine ）
