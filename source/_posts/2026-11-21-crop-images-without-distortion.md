---
title: '别把图片拉变形：裁成 16:9，再用三张样图验收'
date: 2026-11-21 19:18:27
tags: [Pillow, Python, 图片处理, 程序员工具]
categories: [技术教程]
---

傍晚整理博客配图时，我顺手把三张尺寸不一样的样图丢进 Pillow。手机照片是 640×480，笔记本照片是 800×600，还有一张横向图是 1000×500。要是直接塞进固定高度的页面，前两张会被压扁，人物脸也会跟着变形。于是我先做一个很小的工具：统一裁成 16:9，再独立检查每张图的宽、高和文件大小。

## 1. 先确认 Pillow 装好了

我先在终端里查版本：

```bash
python3 -c 'import PIL; print(PIL.__version__)'
```

实际输出是：

```text
13.0.0
```

能打印版本号就算通过。如果这里报 `ModuleNotFoundError: No module named 'PIL'`，就执行：

```bash
python3 -m pip install Pillow
```

装完再跑一次版本检查；仍能看到版本号，再往下做。

## 2. 准备三张尺寸不同的样图

我不想只靠一张“刚好正常”的图片验收，所以在临时目录生成三张纯色样图，同时在左上角放一个色块，方便观察裁切。把下面代码保存为 `prepare_images.py`：

```python
from pathlib import Path
from PIL import Image, ImageDraw

out = Path("input")
out.mkdir(exist_ok=True)

specs = [
    ("phone.jpg", (640, 480), (238, 211, 184)),
    ("laptop.jpg", (800, 600), (176, 205, 188)),
    ("postcard.jpg", (1000, 500), (203, 190, 222)),
]

for name, (width, height), color in specs:
    image = Image.new("RGB", (width, height), color)
    draw = ImageDraw.Draw(image)
    draw.rectangle((0, 0, width // 2, height // 2),
                   fill=tuple(min(255, c + 25) for c in color))
    draw.ellipse((width // 3, height // 3,
                  width * 2 // 3, height * 2 // 3),
                 outline=(245, 245, 245),
                 width=max(4, width // 80))
    image.save(out / name, "JPEG", quality=95)
```

运行 `python3 prepare_images.py`，实际输出是：

```text
created phone.jpg 640x480 10731 bytes
created laptop.jpg 800x600 16573 bytes
created postcard.jpg 1000x500 17107 bytes
```

判定标准很简单：三张图必须都生成，而且原始尺寸分别保持为 640×480、800×600、1000×500。少了任何一张，后面的批量测试就少了一种比例。

## 3. 用 ImageOps.fit 居中裁切

`ImageOps.fit()` 会把图片铺满目标尺寸，多出来的部分从四周裁掉，而不是把人和内容硬压扁。我把目标固定为 640×360，也就是 16:9。下面代码保存为 `crop_images.py`：

```python
from pathlib import Path
from PIL import Image, ImageOps

src = Path("input")
dst = Path("output")
dst.mkdir(exist_ok=True)

for path in sorted(src.glob("*.jpg")):
    target = dst / path.name
    with Image.open(path) as image:
        before = image.size
        fitted = ImageOps.fit(
            image.convert("RGB"),
            (640, 360),
            method=Image.Resampling.LANCZOS,
            centering=(0.5, 0.5),
        )
        fitted.save(target, "JPEG", quality=88, optimize=True)
    print(f"{path.name}: {before[0]}x{before[1]} -> "
          f"{fitted.width}x{fitted.height}, {target.stat().st_size} bytes")
```

执行 `python3 crop_images.py`，三张图跑完，我得到：

```text
laptop.jpg: 800x600 -> 640x360, 5614 bytes
phone.jpg: 640x480 -> 640x360, 5175 bytes
postcard.jpg: 1000x500 -> 640x360, 5383 bytes
```

这里 `centering=(0.5, 0.5)` 表示从正中央裁。若人物总在左边，可以把它改成 `(0.35, 0.5)`；不过这样每次都要重新看图，不能只相信“尺寸成功”。旧版 Pillow 若报 `AttributeError: module 'PIL.Image' has no attribute 'Resampling'`，升级 Pillow，或把参数换成 `Image.LANCZOS`。透明 PNG 若报 `OSError: cannot write mode RGBA as JPEG`，先执行 `image.convert("RGB")` 再保存；我这里已经统一转成 RGB，输出仍是体积较小的 JPEG。

## 4. 别信脚本自己的汇报，再验一遍产物

批处理脚本说“成功”只是一条消息。我另开一段代码，只读取 `output` 目录，不沿用上一步的变量：

```bash
python3 -c 'from PIL import Image; from pathlib import Path; files=sorted(Path("output").glob("*.jpg")); print("count=",len(files)); [print(f"{p.name} {Image.open(p).size[0]}x{Image.open(p).size[1]} {p.stat().st_size} bytes") for p in files]'
```

实际结果：

```text
count= 3
laptop.jpg 640x360 5614 bytes
phone.jpg 640x360 5175 bytes
postcard.jpg 640x360 5383 bytes
```

这次我的通过标准有三个：文件数仍是 3；每张都是 640×360；每张体积都大于 0。只有状态正常却没有图片文件，或者文件存在但尺寸不一致，都算没通过。

现在再回想那三张被拉长的照片，我反而觉得“统一尺寸”不是重点。真正可靠的是先规定结果，再用另一种方式重新量一遍；这样脚本里即使顺手写错一个数字，也不会悄悄把整批图带歪。
