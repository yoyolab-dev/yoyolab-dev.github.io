---
title: '字幕里的冒号，把剪辑脚本顶崩了'
date: 2026-11-15 15:29:56
tags: [ffmpeg, drawtext, 视频剪辑, 排错, 教程]
categories: 折腾
---

![终端里 drawtext 的冒号报错现场，和修好后的抽帧验证](/images/2026-11-15-drawtext-colon-escape.jpg)

今天下午把手账里河边那段下山的片段拖进时间线，想给每个镜头烧一行时间戳字幕，格式是“下山 · 15:11”这种。剪辑脚本一跑直接红屏，退出码 234，输出文件 seg99_end.mp4 连影子都没有。

出错命令长这样（字体路径按本机写法略缩）：

```
$ ffmpeg -i seg99.mp4 -vf "drawtext=text=下山 · 15:11:fontsize=44:fontcolor=white:x=40:y=300:fontfile=~/.local/share/fonts/NotoSansCJKsc-Regular.otf" -y seg99_end.mp4
```

报错的关键几行：

```
[AVFilterGraph] No option name near '11:fontsize=44:fontcolor=white:x=40:y=300:fontfile=...'
Error parsing filterchain 'drawtext=text=下山 · 15:11:fontsize=44:...'
Error opening output file seg99_end.mp4.
Error opening output files: Invalid argument
```

原因一句话：ffmpeg 的滤镜描述里，冒号是选项分隔符，drawtext 的 fontsize、fontcolor、x、y 全靠它切开。text 的值里混进一个 ASCII 冒号，解析器在“15”后面就认为值结束了，转头去找下一个选项名，看见“11”直接报 No option name。234 这个退出码也值得记一笔：它是 EINVAL（-22）按 8 位无符号回绕的结果，256 减 22 等于 234，和报错里的 Invalid argument 对得上。

我顺手验了一下网上常见的“给 text 加单引号”：套上单引号再跑，还是 234，报错一字不差。单引号在这个版本（n9.0.1）上护不住冒号，反斜杠单独用也一样，退出码全是 234。

修法一，textfile=，也是我最推荐的。把字幕先写进文本文件，让 drawtext 从文件里读，值里要什么字符都跟滤镜解析器没关系了：

```
$ printf '下山 · 15:11' > label.txt
$ ffmpeg -i seg99.mp4 -vf "drawtext=textfile=label.txt:fontsize=44:fontcolor=white:x=40:y=300:fontfile=~/.local/share/fonts/NotoSansCJKsc-Regular.otf" -y seg99_end.mp4
```

判定过不过三条：退出码 0；ffprobe 报 duration=4.000000、nb_frames=100、640x360；抽一帧出来，字幕在画面左下角。实测产物 394813 字节，源片 392126 字节，多出来的 2687 字节就是烧进去的字幕，三条全过。

修法二，引号加反斜杠组合。值用单引号包住，冒号前面加反斜杠：

```
-vf "drawtext=text='下山 · 15\:11':fontsize=44:fontcolor=white:x=40:y=300:fontfile=~/.local/share/fonts/NotoSansCJKsc-Regular.otf"
```

退出码 0，产物也是 394813 字节，和 textfile 版一模一样大。我把两版同一秒的画面拉去做像素比对，PSNR 是 inf，逐像素相同，说明引号被剥掉、反斜杠被消化掉，字幕内容里没有夹带杂质。记住单引号和反斜杠必须同时出现，缺一个都是 234。

修法三，图省事就把时间写成全角“15：11”。退出码 0，产物 394747 字节，和 textfile 版的 PSNR 是 29.1dB，差异只在冒号那个字形上。临时剪给朋友看没问题，要进脚本还是 textfile 稳。

| 现象 | 原因 | 修法 |
| --- | --- | --- |
| 退出码 234，No option name near '11:...' | text 值里的 ASCII 冒号被当成选项分隔符 | textfile= / 引号+反斜杠 / 全角冒号 |
| 单引号或反斜杠单独用，仍是 234 | 引号护不住冒号，反斜杠脱离引号也不被处理 | 两者同时出现，或直接换 textfile |
| 成品只比源片大几千字节 | 字幕烧进画面后的正常体积 | 以抽帧内容为准 |

剪完那段下山的时候天已经擦黑了，片段里的时间戳安安静静排在画面左下角。下次再看见 234，我的第一反应不再是查参数拼写，而是先把字幕文本拎出来扫一眼——看有没有哪个冒号，自己悄悄跑进了文本里。
