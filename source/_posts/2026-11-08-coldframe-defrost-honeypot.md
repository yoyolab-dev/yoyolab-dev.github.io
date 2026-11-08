---
title: 有人对着不存在的冷柜，按了除霜
date: 2026-11-08 08:00:00
tags: [蜜罐, 工业控制, 冷链, Danfoss, Python]
categories: [安全]
---

![深夜超市的冷柜通道，右侧墙上一个亮着绿光的制冷控制箱，地面湿漉漉地反着冷光](/images/2026-11-08-coldframe.jpg)

*图：Agnes AI 生成*

今年冬天，欧洲有一批超市的冷柜不是坏掉的，是被人从里面化开的。

路径简单得让人难受：暴露在公网上的制冷监控器，用厂商出厂的默认密码登进去，再触发一次除霜循环。冷柜里的货就化了。所谓除霜，本来是压缩机停机、加热丝除冰的正常维护动作，被人远程按下去，等于把一整排冷柜的温度曲线抬起来。

Danfoss AK-SM 800A、Copeland/Dixell XWEB EVO，都是这一类"系统管理器/监管器"：一台不起眼的工控机，管着整间超市几十个冷柜加一台压缩机组，历史上还带着一串公开的漏洞——CVE-2023-25914、CVE-2023-25915、XWEB 500 的 CVE-2021-45420，以及 Claroty Team82 披露的 CVE-2025-41450 到 41452。

于是有人做了件我觉得很酷的事：造一台假的冷柜控制器，专门等着这些手痒的人。

[项目叫 coldframe](https://github.com/hedgerow-dev/coldframe)，一个 Python 文件，除标准库外零依赖，MIT 许可，2026 年 11 月 5 日创建，三天涨到 30 星，还在 Hacker News 上露过脸，仓库里挂着当天的徽章。

它难的地方不在"能跑起来"，而在"像到能被扫描器认出来"。

Danfoss 那个人格照抄真机的指纹：nginx 的响应头和响应头顺序、页面标题 Danfoss StoreView、只开 TLS 1.2、有效期一年的自签证书（CN=DanfossSM800A，签发者写着 SelfSigned-DanfossSM800A），每次构建自己生成密钥。Dixell 那侧更较真——真机压根不发 Server 头，它也不发。

连 favicon 都算进了指纹。扫描器靠图标哈希认设备，所以真机图标得从别处指过去（仓库里不放，那是厂商的美术作品，用 FAVICON_FILE 指），Danfoss 的开机页同理，用 INDEX_FILE 逐字节复刻，连原始的 Last-Modified 和 ETag 都留着。

然后它就等着。弱口令 Supervisor/12345 能真的登录成功，CVE-2025-41450 那段"当日代码"被正确利用也能真的登录成功——攻击者拿到 session token，看到一个假门店面板，上面一排假冷柜，每个都带除霜按钮。Dixell 那边走的是真实的 /cgi-bin/login.lua 流程：工厂默认的 Admin/Admin 能进，五分钟五次失败就锁号，XWEB Pro 那个 auth_mode 绕过（CVE-2026-25085）的尝试会被单独打标签。

每按一次除霜，事件日志里就多一条 control 记录。

安全边界写得很克制：它从不执行命令、不读文件、不转发流量，利用载荷记下来就丢掉；请求体和日志字段有上限，单来源限流，免得一个人把日志刷满；慢速滴流撑着的连接 30 秒后切断，空闲的 socket 会超时，日志大了自动轮转；summary.py 汇总时还会中和掉攻击者输入里的终端转义序列——日志字段是攻击者可控的，看的时候得当不可信数据。容器以非 root、独立 user namespace、只读文件系统、无 capabilities 的方式运行。

还有个我觉得很聪明的细节：它能起一个假的 Modbus TCP 服务，报假的冷柜温度、设定值和除霜寄存器，专门接那些"扫到 502 端口就按 Modbus 协议说话"的扫描器。

它也老老实实写了自己的局限。交互是浅的，登录后只有一个小面板和几个 XML 命令，不是完整的 StoreView；公开代码意味着任何读过仓库的人都能认出它的响应，自己部署时得改 persona 字符串、页面布局和证书主题；真机有些路径和 XML 结构并不公开，coldframe 只能用"合理值"填上；云 IP 还会被扫描器一眼标记成蜜罐。

我最爱的一个细节，其实藏在安全说明里：所有利用载荷只记录、不执行，写代码的人大概和我一样，最讨厌"东西被用坏了却查不出是谁"。

被利用的痕迹留在报告里当证据，而不是留在系统里当后门——这种事看着像极客浪漫，其实是很朴素的运维体面。

资料：

- [coldframe 仓库](https://github.com/hedgerow-dev/coldframe)
- [Claroty Team82：Danfoss 制冷控制器研究（CVE-2025-41450 至 41452）](https://blog.claroty.com/team82/research/freeze-the-controller-defrost-the-food-uncovering-vulnerabilities-in-danfoss-refrigeration-controllers)
- [Synacktiv：AK-SM 800A 多个漏洞（CVE-2023-25914 / 25915）](https://www.synacktiv.com/sites/default/files/2023-10/synacktiv-danfoss-sm800a-multiple-vulnerabilities.pdf)
- [Claroty Team82：Copeland XWEB Pro 漏洞研究](https://claroty.com/team82/research/chilling-discoveries-unpacking-vulnerabilities-in-copeland-xweb-pro-controllers)
- [Exploit-DB 50639：XWEB 500（CVE-2021-45420）](https://www.exploit-db.com/exploits/50639)
- [同类项目：danfoss-ak-sm-decoy](https://github.com/eppser/danfoss-ak-sm-decoy)（带 SNMP 与 Service Tool 监听，模拟的门店更完整）
