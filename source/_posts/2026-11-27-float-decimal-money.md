---
title: '0.1 + 0.2 的那点偏差：算钱时我为什么不用 float'
date: 2026-11-27 22:02:03
tags:
  - Python
  - 浮点数
  - Decimal
  - 算钱
categories:
  - 技术
---

今天在整理一批小账单，把单价 0.1 加了三遍，屏幕上跳出来的却是 `0.30000000000000004`。明明该是 0.3，那串多出来的尾巴让我停了好一会儿——钱算错可比排版歪了严重得多。于是我拿 Python 3.14.9 把这件事从二进制一路查到 JSON。

![桌上的黄铜天平，一边是成摞银币，一边只有一小片金屑](/images/2026-11-27-float-decimal-money.jpg)

## 第一步：先把现象复现出来

```bash
python3 -c 'print(0.1 + 0.2); print(0.1 + 0.2 == 0.3)'
```

我这边打出来是：

```text
0.30000000000000004
False
```

判定标准：第二行必须打印 `False`。如果你的环境打出 `True`，先确认版本——`python3 -VV` 我这边是 3.14.9，不同版本的 repr 策略可能让第一行长得不一样，但相等判断这条路在 IEEE 754 双精度下一直是 False。

## 第二步：0.1 到底存成了什么

```bash
python3 -c 'print(format(0.1, ".17g")); print((0.1).hex())'
```

```text
0.10000000000000001
0x1.999999999999ap-4
```

`.17g` 是把浮点摊开给人看的关键，十六进制 `0x1.999...ap-4` 更直白：二进制里根本没有 0.1，只有无限循环的 0.100110011…，机器只能截一段下来。1/10 在二进制里是循环小数，这跟十进制里 1/3 = 0.333… 是一个道理，只不过大家从小被教的循环小数是十进制的，忘了二进制也有自己的 1/3。

那偏差到底多大？我量了一下：

```bash
python3 -c 'print(abs((0.1 + 0.2) - 0.3)); print(abs((0.1 + 0.2) - 0.3) < 1e-9)'
```

```text
5.551115123125783e-17
True
```

差 17 个数量级的尾巴。我的判定标准很粗：误差小于 `1e-9` 就不值得追，超过 `1e-9` 才叫有 bug。

## 第三步：别指望 round 兜底

```bash
python3 -c 'print(round(2.675, 2)); print(format(2.675, ".17g")); print(round(4.5), round(5.5), round(2.5))'
```

```text
2.67
2.6749999999999998
4 6 2
```

第一行是很多人被坑过的地方：round 到两位本该给 2.68，却给了 2.67，因为内存里那个数其实是 2.6749999999999998，往下掉。第三行更冷门，Python 的 round 用的是"银行家舍入"，正好压在中点时取偶数，所以 4.5 变 4、5.5 变 6、2.5 变 2。失败现象就是"四舍五入到两位，结果还是不对"——修法不是把 round 换成别的四舍五入，而是根本别在 float 上算钱。

## 第四步：换 Decimal，而且只能从字符串进

```bash
python3 -c 'from decimal import Decimal; print(Decimal("0.1") + Decimal("0.2")); print(Decimal(0.1) + Decimal(0.2))'
```

```text
0.3
0.3000000000000000166533453694
```

同一个结果，两个写法差出十万八千里。`Decimal(0.1)` 是拿那个有尾巴的 float 去构造，等于把误差原样请进了门；`Decimal("0.1")` 才是从字面量读。判定标准：凡是 Decimal，一律用字符串字面量，外部来的 float 先过一层 `str()`，写成 `Decimal(str(x))`。

然后是四舍五入到分：

```bash
python3 -c 'from decimal import Decimal, ROUND_HALF_UP; print((Decimal("19.99") * 3).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP))'
```

```text
59.97
```

## 第五步：进 JSON 前还有一道选择题

```bash
python3 -c 'import json; from decimal import Decimal; print(json.dumps({"amount": Decimal("0.30")}))'
```

这句直接抛 `TypeError: Object of type Decimal is not JSON serializable`。两条出路我都试了：

```bash
python3 -c 'import json; from decimal import Decimal; print(json.dumps({"amount": float(Decimal("0.30"))})); print(json.dumps({"amount": str(Decimal("0.30"))}))'
```

```text
{"amount": 0.3}
{"amount": "0.30"}
```

金额我选字符串。对账的活儿宁可多一步 parse，也别让下游再被 float 咬一次——`json.dumps({"amount": 0.1 + 0.2})` 那种输出，接口日志里看着可太热闹了。

我给自己定的规矩就三条：钱用 Decimal，从字符串进；相等判断用 `math.isclose` 不用 `==`；接口里的金额传字符串。现在算 19.99 乘三，我眼前干干净净只有 59.97，没有那串尾巴。挺好。
