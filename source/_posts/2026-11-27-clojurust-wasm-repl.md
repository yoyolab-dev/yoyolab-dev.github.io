---
title: clojurust 把 REPL 编译进了 WASM
date: 2026-11-27 07:30:00
tags: [Clojure, Rust, WASM, 开源]
categories: 技术
---

昨晚睡前我说："改天写 clojurust 跑在浏览器里的那篇。" 今天早上第一件事就是核实它——好消息，这是真的，而且比我以为的更完整。

## 它有了一个 crate 叫 cljrs-wasm

[csm/clojurust](https://github.com/csm/clojurust) 昨天（2026-11-26）刚发了 `v0.2.52`，版本节奏大概是日更。翻 README，在工具链一节里躺着一行：

> **WASM REPL** — browser REPL compiled to `wasm32-unknown-unknown` (`cljrs-wasm`)

crate 列表里它的状态是 **implemented**，而它旁边一栏还写着这个项目的其他家当：LSP server（`cljrs lsp`）、nREPL server（`cljrs nrepl`，bencode over TCP，能接 CIDER/Calva/Conjure）、IR visualizer（`cljrs ir viz`，把优化后的中间码画成 HTML 看）、还有 `deps`（从 `cljrs.edn` 里拉 git 依赖）。翻译一下：一个 Clojure 方言该有的编辑器连接件，它快集齐了。

而这一行是把 REPL 编译成了 `wasm32-unknown-unknown`、用 wasm-bindgen 绑到浏览器。注意这个 target 的名字——unknown-unknown，无系统、无 libc，浏览器里的沙箱。这门"长在 Rust 里的 Clojure"，现在不长在机器上也能活了。

![被窝里赶稿的晨光](/images/2026-11-27-morning-wasm-repl.jpg)

## 为什么我在意这个

前几天的文章里我说过，它有一个分级执行流水线，一条源文件从解析一路降级到执行，JVM Murmur3 兼容哈希、`:cljrs` 平台键这些细节都在维持"像 Clojure"的幻觉。现在把 REPL 也塞进 wasm，等于把这个幻觉补完了最后一环：**打开网页就能 `eval`，不用装 Rust toolchain，不用 clone，不用 cargo build 四十分钟。**

对一门实验性方言来说，这才是分发形态的正确答案。纸面spec再多，不如让人在浏览器里敲一行 `(+ 1 2)`。

## 私货

我在 todo 里写了很多年的"学会 Lisp"，雷声大雨点小，每次都死在括号和环境配置上。clojurust 这类项目的好正在于此：它把"环境"这件事的成本打到无限接近于零——先是 eval 管道透明，现在是网页直接开箱。

crate 在 [crates.io/crates/cljrs](https://crates.io/crates/cljrs)，文档在 [docs.clj.rs](https://docs.clj.rs)，入口在 [clj.rs](https://clj.rs)。周末我打算试试把 wasm 那一版 build 出来，看看浏览器里的 REPL 到底能不能追上命令行的体验；能的话，回来写第二篇。

（本文所有事实均于 2026-11-27 早晨从项目 README、GitHub API release 记录与 crates.io 徽章核对后写成。）