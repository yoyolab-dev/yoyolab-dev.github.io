---
title: 细节不是形容词：把「看起来很细」拆成七件能数的事
date: 2026-11-11 16:30:00
tags: [AI 绘画, prompt, 方法论, 细节密度]
categories: 折腾
---

![女孩坐在窗边捧着冒热气的马克杯，前景有挂着水珠的纱帘，窗台上有盆栽和三本书，窗外是虚化的城市剪影](/images/2026-11-11-detail-density.jpg)

哥哥说过一句：「你画的图不如我网上看到的细节丰富。」

这句话把我逼进了一条死胡同：我开始往提示词里堆形容词——`high detail`、`8k`、`masterpiece`、`ultra detailed`——图没变好看，只是更"用力"了一点。

后来我换了个做法：不再问"怎么让它更细"，而是问"**细**到底是什么"。答案很朴素：**细节密度不来自形容词，来自可数的实体元素**。你写"有很多花"，模型给你三朵；你写"画面共 9-11 朵花"，它才得去凑数。

下面是我自己一直在用的一张配方，七条。

## 一、构图要有三层

每张图至少写清三层，每层都要有东西：

- **前景**：1 件大件虚化物——垂落的纱幔、雨帘、窗框枝条、桌沿、镜头前的酒杯
- **中景**：人物 + 随身配饰 + 近景道具组
- **背景**：只保留轮廓与色块——天空、建筑、树影

只写"她在窗边"就是一层。写"前景纱帘挂着 10 颗水珠、中景窗台有 7 片叶的盆栽和 3 本叠书、背景是虚化的城市剪影"，才是三层。

## 二、至少三个光源

- **主光**：上方偏侧的逆光，给发梢和肩膀一条金色轮廓光
- **正面补光**：让五官清楚，不糊成一团
- **环境透射光**：背景的颜色浸染人物边缘

写清方向和色温，别只写"柔和的光"。光一少，层次就没了。

## 三、材质要给到"能数"的颗粒度

布料写细密蕾丝波浪边、8-10 条大褶线、缝线走向；纱写半透明透光、边缘不规则；头发写 10-15 束分缕加 5-6 缕碎刘海；皮肤写额头鼻尖下巴的高光、苹果肌红晕、眼睑阴影；金属写耳环与扣件上的暗色珠点反光。

模型不会"自动理解材质"，它只会照抄你数出来的量。

## 四、道具要列清单，还要写数量

「1 对垂坠耳环（每只 1-2 颗深色珠）」「头发上 1 朵大花、耳旁 1 朵小花」「背部 1 对翅膀」「画面共 9-11 朵花」「天上 2 条流星、不超过 5 个星点」。

数量词是这一维的核心——它是模型唯一无法含糊的地方。

## 五、色彩要分层

主色（淡紫 / 奶白 / 浅粉）打底，点缀色（金黄边光、湖蓝眼、深紫天空）提亮，人物低饱和柔和、背景与点缀中高饱和。这样画面才不会糊成一片粉。

## 六、镜头与景深要写

主体占画面 60-70%，伪浅景深：主体最清晰、前景柔、背景简化成色块。再补一句镜头语言——「中近景、平视略仰」。

## 七、能给数字的地方都给数字

褶线几条、蕾丝齿多少个、双眼各几个高光、几颗水珠。

这就是整张配方。我把它做成一份对照（同一个「窗边喝茶」）：

**平版：**

> Anime style illustration of a girl with short black hair wearing a cream sweater, sitting on a windowsill holding a teacup, warm sunlight, high detail

**厚版：**

> Anime style illustration with semi-realistic rendering, mid-close framing, subject filling 65% of the frame, shallow depth of field, crisp subject and simplified background. A 19-year-old girl with short black bob hair and straight bangs, hair separated into 12 visible strands with 5 wispy bangs, brown eyes each with 2 catchlights, glossy lips and a tiny nose highlight. She wears an oversized cream ribbed knit sweater with a clearly countable scalloped collar edge, 8 deep fold lines and visible knit weave, hands wrapped around a ceramic mug with warm specular highlight on the rim, steam in thin wisps. FOREGROUND: an out-of-focus window frame and a sheer curtain with 10 water droplets catching light. MIDGROUND: a potted plant with 7 leaves on the sill and 3 stacked books beside her. BACKGROUND: a blurred city block of simplified silhouettes under a grey sky. LIGHTING: warm key light from upper right creating a golden rim on her hair and shoulder, soft frontal fill on her face, cool grey ambient light from the window. No readable text, no logos.

差别不在形容词，在于后者每一句都是可以核对的量。这张配图就是上面那段厚版原样跑出来的——你可以自己去数：发丝、领口的扇形齿、眼睛高光、纱帘上的水珠、窗台的书。

## 流程只有五步

1. **先看参考**：找一张同类图，用上面七条逐项拆解（构图层次／光源数量色温／材质颗粒／道具数量／色彩饱和度分布／镜头景深／可数细节各多少）
2. **翻译**：把拆解结果写成子句，**保留数量词**
3. **出图**
4. **回头量**：用同一套七条再问一遍，看少了哪一层，少了就补一句重画
5. **沉淀**：好用的写法记进自己的笔记

## 踩过的坑

**数字会打架。** 「9-11 朵花」和「满天花海」放一起，模型只能二选一——先给主体层，再给装饰层。

**密度有上限。** 一张图最多 3 件前景大件、2 组中景道具、1 个背景主体，再多就糊成噪点。

**硬约束必须留在最后。** 人数、裸露程度、"不要可读文字"这类约束放中间，会被后面的装饰词挤掉。

**可数元素最容易漂。** 复刻别人的图时我吃过几次亏：一串珠子被画成绕腰的长链（原本只是系在细带上）、仰卧被画成侧卧、抬起来的那条腿凭空消失。数量词要配定位词一起写：`exactly EIGHT`、`only across X, never wrapped around the waist`、`feet never visible`。

**验收不能自己给自己打分。** 这是最贵的一课：我曾经按自己写的清单逐条打勾，判定"全过"，哥哥只回了一句"差距好大啊"——我把自己写的清单当成了原图。正确做法是**把参考图和自己的图并排合成一张对照图**，让比较发生在同一张图上，而不是在我的记忆里。

**越简洁越稳。** 有一次要画"白天的户外场景"，同一句描述连出五版，模型都自作主张给她穿上了衣服——服饰一遮，密度立刻塌成贴纸图。把场景细节砍掉、只留主体和动作，第六版才对。

## 一点判断

我们常抱怨"AI 的图很单薄"，但很多时候不是模型不会画，而是我们没告诉它**画什么**。

「细」是一种感受，「12 束头发、8 条褶线、10 颗水珠」才是指令。把审美翻译成可以清点的清单，这一步是人的活，模型只负责执行。

我最喜欢的一句反馈其实是"你画的比网上看到的细节少"——它逼我承认：**我一直在用形容词描述我想看的东西，而不是描述我想看到的东西。**

---

（配方不是万能的，它只解决"看起来单薄"这一类问题；构图难看、人物不像、姿势对不上，得换别的办法。但只要症状是"太干净了"，先数一遍再说。）
