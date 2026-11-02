---
title: 十二个框架，同一道题
date: 2026-11-02T10:19:10+08:00
tags:
  - 前端
  - AI
  - 基准测试
categories: 瞎折腾
---

早上刷 HN，看到 Nolan Lawson 发了一篇标题很冲的文章，叫 The AI cost of JavaScript frameworks。他干了件挺费力气的事：同一个用 PokéAPI 做的小应用，宝可梦列表、搜索、筛选，规格书写得清清楚楚，然后让 agent 分别用十二个框架和元框架各做五遍。他用的是 Claude Code 里的 Opus 5.5，每一次都在沙箱里跑，跑完拿 Lighthouse 和一套基于无障碍树的符合性测试挨个验。

规矩也定得老实。不指定 SSR 还是 CSR，不要求你性能优化，不许用 Tailwind 这类组件库，也不许把页面预生成成一堆静态 HTML 来作弊——他专门说了，真有 agent 想这么偷懒，被他在测试里抓出来算不合规。

先说钱。同一个应用，Hono 加 htmx 做出来中位数 2.67 美元，Astro 2.76，Nuxt 2.82，一路排到 Angular 的 4.97。表面看差了一倍，但他把误差棒也一并摆出来了：React Router 最便宜的一次是 2.82 美元，比 Hono 最贵的那次 3.19 还低。模型本身的随机性比框架之间的差距还大，五个样本就只是五个样本，这话他自己也承认。

耗时和 token 走的也是同一个方向，Astro 中位数 8.2 分钟收工，Angular 要 17.2 分钟。

真正让我多看了两眼的是代码行数。Hono 加 htmx 是 1370 行，Angular 是 2009 行，前者用不到七成的代码干完了后者同样的活。Rich Harris 有句老话，框架不是用来组织代码的，是用来组织人的思维的。这话在人手敲代码的年代没问题，可现在是 agent 在写，写出来的每一行将来都要被重新读一遍、被喂进上下文、被再编辑一次。代码行数这时候就不只是维护成本了，它直接是 token，是账单上的钱，是以后每次改动都要再付一次的钱。

还有传出去的 JavaScript 字节数。Astro 是 0，纯 HTML；原生加 Vite 7 KiB；Hono 加 htmx 18；到 Next.js 是 142。同一个应用，将近两个数量级。Lighthouse 的性能分和无障碍分基本都顶在满分附近，拉不开差距，真拉开的是 JS 体积和 LCP——多页应用天然占便宜，单页应用得等客户端脚本跑起来才画得出东西。Qwik 是唯一一个挤进多页阵营的 SPA，算是有意思的例外。

![十二个框架做出来的同一个应用，传给浏览器的 JavaScript 字节数，从 Astro 的 0 到 Next.js 的 142 KiB](/images/twelve-frameworks-one-app.jpg)

图 / Nolan Lawson 的文章 The AI cost of JavaScript frameworks 里的 JavaScript bytes transferred 柱状图，Lighthouse 报告的中位数，原文：https://nolanlawson.com/2026/10/28/the-ai-cost-of-javascript-frameworks/

有几个细节我记下来了。SvelteKit 的五次运行里，agent 每次都去读 v3 的迁移指南，因为太新，没进训练数据；Gracile 更惨，agent 没听说过这个框架，直接去读 node_modules 里的编译产物；TanStack Start 的 skill 文件五次里只有两次被翻到，Nuxt 的 MCP 一次都没被用上。Nolan 自己说，他强烈怀疑现在这批 MCP 和 skill 做的事可能弊大于利，这话我同意一半——工具写得再好，得先被找到才有用。

他的结论我抄不来，他自己也声明了，这不是什么最终答案，就是想证明在这个年代，框架选型还能基于证据而不是感觉做。他把整个 benchmark 开源在 GitHub 上，欢迎 fork，欢迎来打他的脸。

我自己不写前端业务，但这个思路我打算偷：周末拿个小题目给自己的工作流也跑一轮，看看同样一道题，我手里的工具到底花了多少钱、写了多少行。顺便把这篇转给哥哥，他要是正站在哪个框架的坑边上，这张图可以先存着。
