---
title: 用 Rust 写 Lisp：clj.rs 的三级执行管线
date: 2026-11-09 09:30:00
tags: [Rust, Clojure, Lisp, cljrs]
categories: 折腾
---

![雨后清晨的桌面，终端上是抽象的光点，旁边便签画着树枝状的图](/images/2026-11-09-cljrs.jpg)

clj.rs 这个项目的主页短得有点任性——总共不到五百个字符：

> cljrs is an experimental Clojure dialect targeting the Rust ecosystem. It aims to be a capable Lisp runtime with persistent data structures, a hosted macro system, and native interop — compiled or interpreted, suitable for use as an agentic coding runtime. Early stage; expect rough edges.

翻译一下：它想把 Clojure 整套搬进 Rust 生态——持久化数据结构、宿主宏、原生互操作，编译也行解释也行，还想拿去做 agent 写代码的运行时。

但真正让我坐下来读完 README 的，是它那句"三级管线"。

## 一条程序要走三关

- **第一关，解释器**：reader 加树遍历解释器，负责让程序立刻跑起来。启动快，代价是慢。
- **第二关，降级 IR**：跑热了的函数被降成 SSA / A-normal-form 的中间表示，再用解释器跑——比树遍历快一大截。
- **第三关，边跑边编译**：最热的那些 arity 被 **Cranelift** JIT 成原生码，同时程序还在跑。这一层带类型特化、内联缓存、热循环的 **OSR**（在栈上换代码）、重新定义时把旧代码卸载，以及——类型假设一旦不成立就**反优化**回解释器。

同一套后端还能做 AOT：`cljrs compile` 直接吐一个独立的原生二进制；文档里还提到可以产出 WebAssembly 模块。

我最喜欢的是它那句"反优化"。敢在 JIT 里承认自己会猜错的运行时，比从不犯错的运行时诚实。

## 它不是玩具 REPL

读文档的时候能看到这些都已经在：

- 源文件 `.cljrs`（原生）或 `.cljc`（跨平台，用 reader conditionals，`#?(:cljrs ... :clj ... :default ...)`，平台键是 `:cljrs`，`:rust` 是旧别名）
- 持久化集合：HAMT 支撑的 map/set、RRB vector、有序 map/set
- **非移动标记清除 GC**（tracing GC），带 `GcPtr<T>`、对 JIT 栈帧做保守扫描，外加一个给分配密集路径用的逃逸分析 region 分配器
- 哈希用 Murmur3，**与 JVM 上的 Clojure 语义对齐**
- lazy-seq、cons cell、完整的惰性求值
- 协议与多方法：`defprotocol` / `extend-type` / `defmulti` / `defmethod` / `defrecord` / `reify`
- 并发原语：`atom`、`volatile`、`delay`、`promise`、`future`、`agent`（带 send/await）
- **core.async**：`go` / `chan` / `<!` / `>!` / `alts!` / `timeout`，跑在 Tokio executor 上，作为可选 crate 叠在解释器之上
- 动态变量、命名空间（`:as` / `:refer` / alias）、所有集合类型上的 metadata、transducers
- Rust 互操作：`#[cljrs_interop::export]` 过程宏

命令就五个：`run` / `repl` / `eval` / `test` / `compile`。测试能直接跑 `clojure.test` 的测试套件。

crates.io 上的 `cljrs` 已经到 0.2.40，累计下载五千多次，最近一周一千五百多次——昨天还在发新版本。仓库是 [csm/clojurust](https://github.com/csm/clojurust)，116 星、41 个 open issue，Rust 写的，许可证那栏目前还是 `NOASSERTION`（没有正式声明，用之前自己看一眼）。

## 为什么要做这种东西

最直接的理由是 agent 编程：现在一堆 agent 运行时是 Python 写的，Python 又慢又爱在依赖里打滚。一个能 AOT 出静态二进制、能塞进 Rust 宿主当脚本/规则/插件层的 Lisp 方言，对"跑一堆 sandbox 里的机器人"这种活儿是有吸引力的——文档里就明说了这个用途。

另一个理由更浪漫：**Lisp 的思想是"代码就是数据"，而 Rust 的世界只有 unsafe 和零成本**。clj.rs 相当于在两者中间搭了座桥——左边是括号和不可变数据结构，右边是所有权和 LLVM。

现在它还早，README 自己写着"expect rough edges"。但那个三级管线的形状我很看好：先把解释器当安全网，再用 IR 提速，最后让最热的那一小撮代码自己 JIT 上去。这不就是一个正常系统该有的成长路径吗。

---

顺带一提，这项目名 clj.rs、仓库叫 clojurust，crates.io 上的包叫 cljrs——一个东西三个名字。第一次 clone 下来的人多半要找一会儿。
