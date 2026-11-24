---
title: '新建的文件为什么是 644：umask 这个藏起来的默认值'
date: 2026-11-24 21:20:15
tags: [Linux, Shell, 权限, 教程]
categories: [技术教程]
---

今晚我又撞见一次"UNPROTECTED PRIVATE KEY FILE"——密钥明明在那儿，ssh-keygen 就是不肯读。问题不在命令，在文件刚被创建时就被悄悄定下的那串数字。于是我在一只临时目录里把 `umask` 从头到尾摸了一遍。

### 先看默认值：0022

```bash
umask
```

输出是 `0022`。新建一个空文件和一个目录，再问系统它们各是什么权限：

```bash
touch demo.txt
mkdir demo
stat -c '%a %n' demo.txt demo
```

得到 `644 demo.txt` 和 `755 demo`。判定标准很简单：普通用户新建的普通文件是 644，目录是 755，这就是 Linux 上最常见的默认姿态。

### umask 是减法，不是加法

umask 里的位表示"要从基础权限里拿掉哪些位"。文件的底子是 666，目录的底子是 777，umask 负责往回减：

```bash
(umask 0077; touch s077.txt; mkdir d077; stat -c '%a %n' s077.txt d077)
```

输出 `600 s077.txt` 和 `700 d077`，因为 077 把属主之外的所有位都拿掉了，只剩属主自己。再试一次 `umask 002`：

```bash
(umask 0002; touch s002.txt; mkdir d002; stat -c '%a %n' s002.txt d002)
```

输出是 `664 s002.txt` 和 `775 d002`——只拿掉"其他人"的写权限，同组的人还能一起改。

### 它管新文件，不管旧文件

umask 只在文件被创建的那一刻起作用。已有的文件想改权限，得用 `chmod`，而 `chmod` 明确写出的数字不受 umask 管：

```bash
touch old.txt && chmod 600 old.txt && stat -c '%a %n' old.txt
```

输出 `600 old.txt`，哪怕当时 umask 是 022 也照样 600。写密钥、存密码这类脚本，我现在的习惯是先 `umask 077`，再用 `install -m 600` 把文件放到位，两道保险。

### 两个容易踩空的地方

第一，umask 是 shell 的状态，进得去出不来。在子 shell 里改，父 shell 纹丝不动：

```bash
(umask 0077; touch in_sub.txt; stat -c '%a %n' in_sub.txt)
umask
```

第一行输出 `600 in_sub.txt`，第二行还是 `0022`。反过来，子进程会继承父进程的 umask：`umask 077` 之后 `bash -c 'touch child.txt'` 拿到的文件是 600。所以脚本里想改，必须写在脚本开头、当前 shell 那一层。

第二，`mkdir -m 755` 和 `cp` 各有各的脾气。`mkdir -m` 指定的权限会直接盖过 umask，`umask 077` 下建出来的目录仍是 755；而 `cp` 新建文件时会拿源文件的权限去和 umask 相遇——644 的源文件在 `umask 077` 下落地成 600。解 tar 包也是一样：默认解出来的目录跟着 umask 走（实测 700），加 `-p` 才保留归档里原本的 755。

### Git 只认那个 x 位

Git 不存 644 还是 600，它只记有没有执行位。给文件加上 `chmod +x` 之后：

```bash
git diff --summary
```

会打印 `mode change 100644 => 100755 a.txt`。判定标准是看 `git ls-files -s` 里的那串 `100644` / `100755`，644 和 600 在 Git 眼里没有区别。怕本机权限和服务器不一致时，`git config core.fileMode false` 可以让 Git 别再管这事——但这只是让 Git 闭嘴，文件本身的权限该修还得修。

### 最疼的一次，是 SSH 私钥

我在临时目录里生成一把 ed25519 密钥，故意把权限放开到 644，再让 ssh-keygen 读它：

```bash
chmod 644 key_600
ssh-keygen -y -f key_600
```

屏幕上是一整块警告：`WARNING: UNPROTECTED PRIVATE KEY FILE! Permissions 0644 for 'key_600' are too open.`，退出码 255。放宽到 640——同组可读——也一样被拒。只有 600 才肯好好干活，退出码 0。

所以那串看着不起眼的四位数字，其实是系统里最便宜的一道保险。你新建一个文件时顺手带上的权限，决定了下一个拿到它的人能做什么。

我今晚把写密钥的脚本都翻了一遍，补上了 `umask 077`。顺手改掉的那些，凌晨都不会再叫醒我。
