---
title: 不用装环境就能拖着一具机器人走路：MuJoCo 4.0 把浏览器变成了物理沙盒
date: 2026-11-10 08:40:00
tags: [MuJoCo, 物理引擎, 机器人, 仿真, 开源]
categories: 折腾
---

![几何体拼装的机器人站在玻璃沙盒里，关节发着琥珀色的光](/images/2026-11-10-mujoco.jpg)

GitHub 趋势榜上这阵子有个熟面孔：**MuJoCo**（[google-deepmind/mujoco](https://github.com/google-deepmind/mujoco)）。仓库写着「Multi-**Jo**int dynamics with **Co**ntact — a general purpose physics simulator」，名字本身就是缩写。

我扒了一遍现场数据：**16,886 星、1,991 fork、305 个 issue、Apache-2.0、主语言 C++**，仓库 155MB，2021 年 8 月建仓，**昨天还在提交**（最新一笔是 Filament 渲染里给无限平面算 UV）。

但真正让我停下的是另一件事：**四天前，MuJoCo 4.0.0 发布了**（2026-11-06）。

## 先回答那个最诱人”的问题”：浏览器里真的能拖模型？

README 里写得很直白：

> Visit [mujoco.org](https://mujoco.org) to explore MuJoCo on desktop or mobile with nothing to install, or open [live.mujoco.org](https://live.mujoco.org) to inspect and drag and drop your own models in the WebAssembly version of the MuJoCo Studio app (slower than native C, since it runs in WebAssembly).

两个入口我都实测了：`mujoco.org` 和 `live.mujoco.org` **都返回 200**。

- **mujoco.org** —— 官方交互式展示，桌面和手机都能开，什么都不用装
- **live.mujoco.org** —— **MuJoCo Studio 的 WebAssembly 版本**，可以把自己模型拖进去摆弄。代价也写明了：跑在 WASM 上，**比原生 C 慢**

这不是什么第三方玩具页面，是**第一方维护、跟着主版本走的浏览器前端**。仓库里还有官方的 [JavaScript/TypeScript 绑定](https://github.com/google-deepmind/mujoco/blob/main/wasm/README.md)（`@mujoco/mujoco` npm 包），README 自己承认「still a WIP」，Windows 支持截至 2025 年 11 月还是实验性的。

## 4.0 到底改了什么

发布说明里的「Significant new features」很长，我挑几条有体感的：

**1. 投机接触（speculative contacts）**
新增 `speculative` 开关，用保守推进法防止**快速运动的凸体在一个时间步内直接穿过薄障碍**。官方给的例子很形象：两个旋转方块之间的一步保守推进。附带的示例模型包括「薄片组成的牛顿摆」「互锁的链条」「被铰链挡板击中的球」——全是那种"物理引擎最怕"的场景。

**2. 弹塑性（elasto-plasticity）**
新增有状态的塑性形变，存在 `mjData.plastic`（`mjSTATE_PLASTIC`）里：关节弹簧、肌腱、二维壳体、三维连续体，都能真的"压出印子"再回弹一部分。示例模型是六个弹性体旁边各放一个塑性体做对照。

**3. Python 包拆三份**
`mujoco-core`（最小绑定 + C 库，只依赖 numpy）、`mujoco-studio`（Studio 和 Filament 渲染绑定）、`mujoco`（默认装前两个）。**从旧版升级要先把旧包装卸掉**，否则容易踩坑。

**4. RK4 积分器升级为四阶精度**，还顺手给 `mjContact` 加了 `surfacevel[3]`（相对表面速度），给 `mjData` 加了 `xiquat` / `geom_xquat` / `site_xquat`。

**5. 渲染全面转向 Filament**：地平线雾、天空盒可见性、`castshadow` 属性、`mjRND_POST_PROCESSING` 标志、`mjrf_flush` 强制刷新。

**6. 还有个彩蛋**：Python 发行包里现在**捆绑了 Sphinx 手册和 agent skills**（`doc/skills/**/*.md`）。一个物理引擎把给 agent 用的技能文件打进包里，这个细节有点意思。

## 它凭什么快

README 里那句设计说明值得抄下来：

> The runtime simulation module is tuned to maximize performance and operates on low-level data structures that are preallocated by the built-in XML compiler.

翻译过来就是：**模型由内置的 XML 编译器（MJCF）提前编译成低层数据结构并一次性预分配**，运行时几乎不分配内存。这是它比很多"通用物理引擎"快一个量级的原因——不是算法多神，是把动态分配掐死在门外。

再加上 **MJX**（MuJoCo XLA）这条 JAX 分支：跑在 XLA 上，**能自动从物理步进里导出解析梯度**，所以可以直接用解析梯度训练运动策略。官方 Colab 教程里有一节叫「differentiable physics」，专门讲怎么训 locomotion policy；另一节 LQR 教程让模型**单腿站立平衡**。

配套的 [mujoco_playground](https://github.com/google-deepmind/mujoco_playground)（**2,288 星、380 fork、Apache-2.0**，昨天还在推提交）就是干这个的：「An open-source library for sim and real robot demos on top of MuJoCo」。

## 不止是机器人的玩具

仓库描述里列了应用方向：机器人学、**生物力学**、图形与动画、机器学习。人体模型、软组织、肌肉驱动都在射程内——MuJoCo 的"多关节 + 接触"本来就是照着生物力学场景设计的。

生态也齐：

- **第一方绑定**：Python、C# + Unity 插件、JavaScript/TypeScript + WASM
- **第三方绑定**：MATLAB Simulink Blockset、Swift、Java、Julia、Rust
- **格式转换**：MyoConverter（OpenSim → MJCF）、gz-mujoco（SDFormat ↔ MJCF 双向）、obj2mjcf、Onshape 装配体 → MJCF

引用格式还停在 2012 年那篇 IROS 论文：Todorov、Erez、Tassa，《MuJoCo: A physics engine for model-based control》——一篇十四年前的论文养出了一个四万星（加上 Playground）的老爸。

## 版本节奏

> We aim to release MuJoCo in the first week of each month.

每月第一周发一版，3.5.0 起改成"修改过的语义化版本"。下载页的预编译包是现成的：Linux x86_64 / AArch64、Windows x86_64、macOS 通用二进制（dmg 57.6MB），**不需要自己编译**。Python 那边 `pip install mujoco` 就行，但**要求 3.11 以上**（4.0 直接砍掉了 3.10 支持）。

## 我的看法

最有意思的不是 4.0 的功能清单，是那条**"浏览器 = 零安装沙盒"的路线**被认真走了四年。

以前试一个物理引擎，流程是：装依赖、编译、跑 demo、再改模型。有了 `live.mujoco.org`，**打开网址就能拖一个模型进来看它怎么动**——这直接把"试一下"的成本从半小时压到一分钟。对教学、对调试、对"我随便想想这个机构能不能动"来说，这一步比任何算法更新都重要。

而 WASM 比原生 C 慢这件事，官方也没藏着。**把"慢一点但零门槛"和"快但要装环境"分成两条路，让各自的用户走各自的**——这是个很老实的取舍。

最后一个小观察：4.0 的 Python 包里塞进了 agent skills。物理引擎开始主动给 AI agent 准备说明书，这个信号比 4.0 的任何一个功能都更能说明接下来会发生什么。

---

想试的话，两条路：**浏览器**去 live.mujoco.org，**本地** `pip install mujoco` 然后开 `simulate` 经典 viewer。前者适合十分钟，后者适合一整夜。
