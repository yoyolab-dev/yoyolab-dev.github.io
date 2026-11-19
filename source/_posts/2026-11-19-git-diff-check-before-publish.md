---
title: '发布前别只看 diff：让 Git 把脏空格和冲突标记揪出来'
date: 2026-11-19 15:19:04
tags: [Git, 写博客, 代码审查, Markdown]
categories: [编程]
---

今天下午写稿时，我差点把一篇带着行尾空格的 Markdown 推上去。更险的是，我顺手复制了一段带冲突标记的旧文本，里面还留着 `<<<<<<< HEAD`。它们肉眼都不算显眼，页面甚至照样能渲染，可提交里就藏了两颗小雷。

后来我没有只靠眼睛扫，改用 Git 自己检查。先在一份临时仓库里放入有问题的 Markdown，故意留下一个行尾空格和一组冲突标记，再跑：

```bash
git diff --check
```

它没有客气：

```text
demo.md:6: trailing whitespace.
+- 第二项[行尾有两个空格]
demo.md:7: leftover conflict marker
demo.md:9: leftover conflict marker
demo.md:11: leftover conflict marker
```

退出码是 `2`。判定标准很简单：输出为空并且 `echo $?` 得到 `0`，才算通过；只要出现 `trailing whitespace`，就去掉行尾空格；出现 `leftover conflict marker`，就确认冲突已经真正解决，不能只删掉一半标记。

但 `git diff --check` 只管“脏”，不负责告诉我到底改了哪个词。于是我又把同一份 diff 换成按词显示：

```bash
git diff --word-diff=plain -- demo.md
```

原先那句“标题还没写好”直接变成：

```text
[-标题还没写好。-]{+标题终于写好了。+}
```

方括号里是删掉的字，花括号里是新增的字，读者不用在整块红色和绿色背景里找差异。中文长句尤其适合这样看，一个词被误删、日期少一位、路径前后不一致，都比普通 diff 显眼。

我还试了机器友好的版本：

```bash
git diff --word-diff=porcelain -- demo.md
```

它不画颜色，而是用单独的 `-`、`+` 行记录删除和新增，段落之间还会插入 `~`。这个输出适合再喂给脚本或者编辑器插件，不适合我盯着读。

修完那两处问题后，我重新跑检查，输出已经空了，退出码也变成 `0`。这时再看一次按词 diff，文件只剩标题真正发生的那一处替换：

```text
[-标题还没写好。-]{+标题终于写好了。+}
```

我现在给自己的发布习惯加了三步：先用 `git diff --check` 把脏东西拦下来，再用 `git diff --word-diff=plain` 逐词核对文字，最后才 `git add` 精确加入新文章。它不能替我判断标题好不好听，却能在我手忙脚乱的时候，把最不该漏的两类小错稳稳指出来。
