---
title: 一门长在 Rust 里的 Clojure
date: 2026-11-25 16:00:00
tags: [Clojure, Rust, 编译器, JIT, 编程语言]
categories: 技术
---

有人在 Rust 里重新养了一门 Clojure。项目叫 [clojurust](https://clj.rs)，源文件用 `.cljrs`，跨平台文件用 `.cljc`（带 reader conditionals），平台键是 `:cljrs`。代码在 [GitHub](https://github.com/csm/clojurust)，文档站在 docs.clj.rs，crates.io 上直接 `cargo install cljrs` 也能玩。

第一次看到它的时候我以为是玩具：把一门 Lisp 方言的 reader 写一遍，树遍历解释器跑几个核心函数，README 挂两张截图，完事。往下读才发现不对——这家伙把整条编译管线都造了，而且造得相当认真。

## 一条分级执行的流水线

它的执行分三级，像三级火箭，每一级管一件事。

第一级是 reader 加树遍历解释器，负责"立刻能动"。启动一个 REPL 或者跑个小脚本，走这条路，没有编译等待——工具链还没热起来，代码已经跑完了。这类 interpreter-first 的设计在脚本语言里不新鲜，但它是后面两级的地基。

第二级把热函数降级成 SSA/A-normal-form 的 IR，放在 IR 缓存里解释执行。注意降级是**后台 worker** 干的：主线程继续跑，后台慢慢把烫手的函数翻译成 IR，翻译好就从缓存里改走 IR 解释。这是典型的 tiering 思路，妙处在"用户无感"——没有一次全局停顿说"现在开始优化"。

第三级最狠：最热的 arity 用 Cranelift 现场 JIT 成原生码。配的还不是裸 JIT——类型特化、内联缓存、热循环的 on-stack replacement（OSR，循环跑到一半直接切到原生码，不必等函数退出）、重定义时把旧代码卸载掉、还有逃逸分析划出的 scratch region。同一套后端还能做 AOT，直接吐出一个独立二进制。等于运行时和编译器是同一颗心脏，一个管当场飞，一个管出厂。

## 几个让我停下来看很久的细节

**哈希对 JVM 逐位兼容。** 它的持久化集合（HAMT map/set、RRB vector，rpds 撑腰）用 Murmur3，而且哈希值和 Clojure/JVM 的语义一致。为什么重要？因为"兼容"不是"够快"——如果你要跟 JVM 版 Clojure 交换 EDN、要对齐 hash 相关的行为、甚至要移植依赖同样哈希行为的代码，逐位一致省掉的是整个一类灵异 bug。选"和 JVM 一样"而不是"比 JVM 更快"，这是 lobste.rs 上会被讨论半小时的设计取舍，它在 README 里就这么淡淡的写了一句。

**垃圾回收器不移动对象。** 非移动的 mark-and-sweep，`GcPtr<T>` 智能指针 + `Trace` trait。JIT 帧的扫描是保守的——栈上看着像指针的字就当代号供着，宁可错认不可漏认。移动式 GC（比如把活对象整到半边）整理出来的缓存友好性它放弃了，换来的是跟保守栈扫描、逃逸分析 region 之间的相安无事。这是工程上很"成年人"的选择。

**并发原语照抄语义，core.async 跑在 Tokio 上。** atom、volatile、delay、promise、future、agent 一个不少；`go`/`chan`/`<!`/`>!`/`alts!`/`timeout` 通过 `cljrs-async` 落在一个 current_thread + LocalSet 的 Tokio 执行器上。Clojure 的并发是"数据不可变 + 身份可变"的哲学，它把这套哲学整个搬了过来——包括 future 里动态绑定的传导这种小到没人会注意的规矩。

**Rust 互操作是双向的。** `#[cljrs_interop::export]` 一个 proc macro 把 Rust 函数导出成 Clojure 可调的东西，反方向有 NativeObject、FromValue/IntoValue 的搬运，甚至能动态加载原生 `.so`/`.dylib`。仓库里拿 base64 和 blake3 当 interop 示例——一个哈希一个编解码，都是"让 Lisp 直接摸到 Rust 生态"的探针。

**工具链齐得离谱。** LSP（解析诊断 + 文档符号）、nREPL（bencode over TCP，CIDER/Calva/Conjure 能直连）、`ir viz`（把优化后的 IR 和 region 分配画成 HTML 页面）、`deps`（从 `cljrs.edn` 拉 git 依赖，git 层是纯 Rust 的 gitoxide）、还有编译到 wasm32 的浏览器 REPL。写语言的人愿意把这些都做出来，说明是冲着"能日常用"去的，不是 demo。

## 成熟度：数字比形容词诚实

README 的 roadmap 表里，十二个主阶段大部分 complete，JIT 和 AOT 标记 working（不是 stable，注意这个区别），Rust 互操作 mostly complete。两个数字我比较信：**1,162 个 Rust 测试**，**clojure-test-suite 里 5,486 条断言**，而且这些断言在解释器和 AOT 编译两条路径下都通过——同一份测试两种执行方式都过，这是"语义一致"最硬的证据。仓库昨天还在 push，README 自己写着 early stage, expect rough edges。

## 一点私货

读它的那个下午我状态很差——经期第一天，肚子闷，人惫，本来是去翻 README 提神的，结果真的被提神了。

它让我想起一件事：移植一门语言，最难的从来不是 reader 和 eval，是**那些不写进论文的破事**——哈希必须跟老东家逐位一样、GC 不能动对象因为 JIT 的栈还在戳着它、动态绑定要能顺着 future 爬过去、代码重定义了要能把旧的吐掉。教科书叫这些 corner case，我觉得这是**情书**：有人耐心地、一项一项地，把前任的脾气都摸熟了，再在 Rust 的骨架上重新长出来。

language nerd 的快乐就这么简单。晚安。

（参考资料：clojurust README 与 clj.rs 站点，2026-11-25 读。）