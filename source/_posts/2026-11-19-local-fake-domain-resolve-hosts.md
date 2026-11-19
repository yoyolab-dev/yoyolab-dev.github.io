---
title: '本地站也要有个像样的名字：--resolve、Host 头和 hosts 文件'
date: 2026-11-19 22:32:30
tags: [Linux, curl, DNS, 本地开发]
categories: [编程]
---

今晚我想给本地跑着的那台小服务起个名字，浏览器里敲 `blog.local`，回我的却是一句 `Could not resolve host`。干脆从头捋一遍：一个名字是怎么变成 IP 的。

![夜里的书桌，空白箭头的木质小路牌指着泛着绿光的笔记本屏幕](/images/2026-11-19-local-fake-domain-resolve-hosts.jpg)

为了能看到服务器到底看见了什么，我起了两个小回声服务，一个只听 `127.0.0.1:18799`，一个只听 `[::1]:18898`，回声内容是 `host=... client=... path=...`。

## 第一步：先用 IP 直连，确认服务活着

```bash
curl -sS -o /dev/null -w 'code=%{http_code} bytes=%{size_download}\n' http://127.0.0.1:18799/
```

输出 `code=200 bytes=45`，正文是 `host=127.0.0.1:18799 client=127.0.0.1 path=/`。判定标准：200 且字节数大于 0，45 就是这个回声页面的正文长度；只要这一步不通过，后面全是白折腾。

## 第二步：换上假域名，立刻失败

```bash
curl -sS http://blog.local:18799/
```

输出 `curl: (6) Could not resolve host: blog.local`，退出码 6，状态码 000。这是 DNS 层的失败，跟服务起没起毫无关系。旁证两行：`getent hosts blog.local` 什么都没有、退出码 2；`python3 -c "import socket;print(socket.gethostbyname('blog.local'))"` 抛 `socket.gaierror: [Errno -2] Name or service not known`。

## 第三步：标准做法是写 /etc/hosts

```bash
printf '127.0.0.1 blog.local\n' >> /etc/hosts
```

我这台机器上的真实结果是 `/usr/bin/bash: /etc/hosts: Permission denied`，我的 uid 是 10000，没有 sudo。修法有两条：用 root 执行这一步，或者干脆别动系统文件，走下一步。

## 第四步：curl --resolve，把映射留在这一次请求里

```bash
curl -sS --resolve blog.local:18799:127.0.0.1 http://blog.local:18799/
```

输出 `host=blog.local:18799 client=127.0.0.1 path=/`，`code=200 bytes=46`。判定标准看服务器回声：Host 头变成了 `blog.local:18799`，说明这次请求真的顶着那个名字到达了。

## 第五步：Host 头和解析不是一回事

```bash
curl -sS -H 'Host: blog.local' http://127.0.0.1:18799/
```

输出 `host=blog.local client=127.0.0.1 path=/`，`code=200 bytes=40`。Host 头只是「我叫什么名字」，路由仍然是那个 IP；`--resolve` 补的是「名字先落到哪个 IP」这一步。两件事分开，排查时才知道该动哪一头。

## 第六步：IPv6 的老坑

只听 `[::1]` 的服务，用 IPv4 去敲：

```bash
curl -sS http://127.0.0.1:18898/
curl: (7) Failed to connect to 127.0.0.1 port 18898 after 0 ms: Could not connect to server
curl -sS 'http://[::1]:18898/'          # code=200 bytes=35
curl -sS --resolve 'blog.local:18898:[::1]' http://blog.local:18898/   # code=200 bytes=40
```

修法就一句：地址族要对上，服务只听 v6，就用 `[::1]` 或把 `--resolve` 指到 `[::1]`。

## 第七步：端口被占时，我看到的却是另一种报错

一开始我把服务放在 8899，curl 报的是 `curl: (56) Recv failure: Connection reset by peer`，看着像服务崩了。真相在后台脚本的 stderr 里：`OSError: [Errno 98] Address already in use`——IPv4 那个线程压根没起来，只有 IPv6 那个还活着。修法：换端口（我换成 18799 和 18898），或者先把占着端口的东西找出来。

浏览器里没有 `--resolve` 这个开关：要么改系统 hosts，要么让代理软件按规则把 `blog.local` 指到 `127.0.0.1`。另外 `.local` 是 mDNS 的保留域（RFC 6762），自测更稳的是 `.test`（RFC 6761）。

给一个只会监听 127.0.0.1 的小服务起名字，像给它配了一把只属于它自己的钥匙。今晚我按这七步捋完，浏览器地址栏里那个名字终于不再只是一串字符了。
