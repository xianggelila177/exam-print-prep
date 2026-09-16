---
name: "exam-print-prep"
description: "把试卷、讲义、习题的截图或拍照批量做成可直接打印的稿件——自动裁掉手机截图的黑边和页码条、锐化文字、拉白底色，并按\"无答案版/有答案解析版\"分成两个文件夹，各出一份合并 PDF，最后打成压缩包。当用户发来试卷/讲义/习题的截图或压缩包，并说\"去黑边\"\"方便打印\"\"打印出来\"\"处理一下发给学生\"\"分成有答案和没答案两份\"\"裁一下边\"\"图片太糊打印不清楚\"之类的话时，必须使用此 skill。即使用户只是丢来一个截图 zip 说\"帮我弄成能打印的\"，也应触发。"
---

# 试卷截图 → 打印稿

老师和家长常见的场景：在手机或 iPad 上看到一份试卷/讲义，一页页截图存下来，
结果每张图上下都是播放器黑边和"3/10"这样的页码条，直接打印会浪费墨、纸面歪、
字还发虚。这个 skill 把这堆截图变成能直接塞进打印机的稿件。

绝大多数这类资料会同时存在**原卷**和**带答案解析版**两套。学生要做的是原卷，
老师批改要的是解析版，所以默认就得分成两个文件夹——用户往往不会明说，但这是
他们真正想要的。

## 准备：把脚本落地

所有像素活儿都由一个脚本完成。**每个新 session 第一次用时，先把本文末尾
「附：prep_print.py 全文」里的代码原样写到 `/tmp/prep_print.py`**（用 Write 工具，
或 bash heredoc），后面所有命令都调它。它依赖 Pillow 和 numpy，一般已装；
缺了就 `pip install pillow numpy --break-system-packages`。

## 工作流

三步走，不要跳过第一步。

### 第 1 步：survey — 先看清楚手里是什么

```bash
python3 /tmp/prep_print.py survey <图片/目录/zip...> --sheet /tmp/sheet.png
```

它产出**两张图**加一份裁剪诊断 JSON，两张都要用 Read 工具看：

- `/tmp/sheet.png` — 缩略图联表，看版式、找重复页、判断有没有答案解析。
- `/tmp/sheet_footers.png` — 页脚拼贴，把每页底部按原始分辨率放大后竖着排开。
  分组判断几乎全靠页脚的"第 X 页/共 N 页"，而这行字在缩略图里必然糊掉，
  所以单独出这张。**先看它**，页码往往一眼就把分组问题解决了。

这是整个流程唯一需要"用眼睛"的地方，它决定后面所有判断。看的时候回答三个问题：

1. **分几套？** 看页脚拼贴里的"第 X 页/共 N 页"。共 8 页的和共 10 页的显然是
   两套；同一套内页码应当 1→N 连续。缺页、重复页也在这里暴露。
   注意别被截图右上角"3/10"这种阅读器计数器骗了——它是当次滑动的进度，
   同一套里可能中途变化，页脚才是文档自己的页码。
2. **哪套带答案？** 带答案的版本会出现【答案】【详解】【解析】【小问详析】
   这类标记，或者选项后面直接跟着正确选项。带解析的那套通常页数更多。
3. **裁剪对不对？** 诊断 JSON 里 `cropped_ratio` 接近 1.414 说明裁出来正好是
   A4 纸面；`kept_fraction` 特别小（比如 <0.3）说明裁过头了，这多半发生在
   深色底的讲义上——这时候用 `--thresh 100` 甚至 `--no-crop` 重来。

页脚在纸面上的位置各家排版不同，默认取底部 12%。如果拼贴里没看到页码，
用 `--footer-frac 0.2` 多取一点再看。

如果只有一套（用户只截了原卷，或只有解析版），别硬凑两个文件夹，
就出一个，然后在回复里说明只识别到一套。

### 第 2 步：build — 每套单独跑一次

```bash
python3 /tmp/prep_print.py build <这一套的图...> --out "无答案-试卷"
python3 /tmp/prep_print.py build <那一套的图...> --out "有答案-解析"
```

传图片时按页序传，脚本对同一批文件会按文件名里的数字自然排序。输出目录名
就是 PDF 名，所以目录名直接用中文的、用户一眼能懂的名字，例如
`无答案-试卷` / `有答案-解析`，或者带上科目 `化学-无答案`。

默认参数是给"手机截图的电子版试卷"调的：放大到 2480px 宽（A4@300dpi）、
USM 锐化 130%、对比 1.18。跑完看返回 JSON 里的 `dark_edge_warnings`——
非空说明某页四边还有暗像素，黑边没裁干净，得回头调 `--thresh`。

### 第 3 步：pack — 打成一个压缩包

```bash
python3 /tmp/prep_print.py pack "无答案-试卷" "有答案-解析" --zip "化学试卷_打印版.zip"
```

zip 里保留两个文件夹的结构，每个文件夹里是 `01.jpg…NN.jpg` 加一份合并 PDF。
PDF 是给"直接点打印"的人用的，JPG 是给要挑页、要重排的人用的，两个都留着。
用户如果明说"给我一个 PDF 就行"，就别打包，直接交 PDF——多一个 zip 是噪音。

注意：有些环境里 `zip` 命令行工具无法直接往挂载目录写（它要先建临时文件），
脚本的 `pack` 用 Python 的 zipfile，不受这个限制。如果你另外用了 `zip` 命令
遇到 "Operation not permitted"，改成先在 `/tmp` 打包再 `cp` 过去。

## 参数怎么调

用户抱怨什么，就调什么。别一上来堆参数，默认值对干净的电子版截图已经够好了。

| 用户说 | 调什么 |
|---|---|
| 字还是虚 / 打印出来发灰 | `--sharpen 180 --contrast 1.3` |
| 锐化过头了，笔画有白边毛刺 | `--sharpen 80 --radius 2` |
| 底色发灰、有阴影（多见于拍照） | `--whitepoint 235`，更脏就 `225` |
| 想省墨 / 黑白打印机 | `--grayscale` |
| 打印出来纸边缘有黑线 | `--margin 8`，再不行 `--thresh 180` |
| 图被裁掉了内容 | `--thresh 100`；还不行 `--no-crop` |
| 文件太大发不出去 | `--target-width 1654`（A4@200dpi）`--quality 85` |
| 就要图片不要 PDF | `--no-pdf` |

`--whitepoint` 是拍照场景的关键：它把设定亮度以上的像素一把推成纯白。
定多少不要拍脑袋——先测一下纸面实际有多灰（`np.percentile(灰度图, 95)`），
测出来 225 就设 225。纸面上的浅灰阴影会消失，但注意如果卷子上有铅笔写的
浅色答案，会一起被抹掉，所以只在用户确实抱怨灰底、且卷面没有手写内容时才用。

手机拍的照片比截图软，`--sharpen 170 --contrast 1.25` 通常比默认值更合适。

## 交付

用 `present_files` 把 zip（或 PDF）递给用户，然后用两三句话说明：分了哪两套、
各多少页、做了什么处理。不要长篇复述流程——他们要的是能打印的文件，
不是工作报告。

如果 survey 阶段发现了不确定的地方，在交付时点出来让用户核对，别默默替他们
决定。最常见的三种：解析版页脚写"共15页"但只截到第10页；同一页截了两次；
文件名说的科目和卷面实际科目对不上。这些都值得单独一句话提醒。

## 边界情况

- **拍的照片而不是截图**：纸面可能是倾斜的、四角带桌面背景。亮度扫描只能裁出
  外接矩形，裁不掉倾斜。先如实告诉用户"这几张是斜的，裁边只能裁到矩形"，
  再问要不要接受，别假装处理干净了。
- **重复页**：同一页截了两次很常见。先用 md5 查完全相同的文件，再看联表里
  相邻两格是不是同一页，处理前剔掉。
- **横屏截图 / 双页拼在一张图上**：一张图里有两页时，先用 `--no-crop` 出图，
  再按中缝从中间切开，然后当两页处理。
- **图很多（>50 张）**：survey 的联表会很长，用 `--cols 8` 让它更紧凑，
  必要时分批看。
- **卷面有大块深色照片、又要省墨**：`--grayscale` 之外，可以另写几行把
  大面积深色区域调淡到约 45% 墨量，同时用腐蚀操作把细笔画排除在外，
  保证文字仍是纯黑。用户明确说"省墨"时这一步值得做。

---

## 附：prep_print.py 全文

把下面的代码原样保存为 `/tmp/prep_print.py`。

```python
#!/usr/bin/env python3
"""
prep_print.py — 把试卷/讲义的截图或拍照，批量处理成可直接打印的稿件。

三个子命令，对应工作流的三步：

  survey   给一批图生成缩略图联表 + 页脚拼贴 + 每张图的裁剪诊断，
           供 Codex 用眼睛看一遍，判断分组和裁剪是否正确。
  build    裁黑边 → 升采样到目标 DPI → 锐化 → 对比增强 → 出 JPG + 合并 PDF。
  pack     把若干输出目录打成一个 zip。

设计要点：
- 裁剪用"行/列亮度扫描"找纸面，能同时干掉播放器黑边、状态栏、页码条。
- 所有增强参数都可从命令行调，因为不同来源的截图脏的程度差很多。
- 脚本只做确定性的像素活儿；"哪几页是一套""哪套带答案"这类判断留给 Codex。
"""

import argparse
import json
import os
import re
import sys
import tempfile
import zipfile

import numpy as np
from PIL import Image, ImageEnhance, ImageFilter

Image.MAX_IMAGE_PIXELS = None
EXTS = (".jpg", ".jpeg", ".png", ".webp", ".bmp", ".tif", ".tiff")


# ---------------------------------------------------------------- 工具

def natural_key(p):
    """按文件名里的数字自然排序，保证 2.jpg 排在 10.jpg 前面。"""
    return [int(t) if t.isdigit() else t.lower()
            for t in re.split(r"(\d+)", os.path.basename(p))]


def collect(paths):
    """接受文件、目录或 zip，统一返回排好序的图片路径列表。"""
    out = []
    for p in paths:
        if os.path.isdir(p):
            for root, _, files in os.walk(p):
                if "__MACOSX" in root:
                    continue
                for f in files:
                    if f.lower().endswith(EXTS) and not f.startswith("._"):
                        out.append(os.path.join(root, f))
        elif p.lower().endswith(".zip"):
            # 解到临时目录，因为上传目录通常是只读的。
            dest = tempfile.mkdtemp(prefix="prep_print_")
            with zipfile.ZipFile(p) as z:
                z.extractall(dest)
            out += collect([dest])
        elif p.lower().endswith(EXTS):
            out.append(p)
    return sorted(set(out), key=natural_key)


def find_page(im, thresh=150, margin=0):
    """
    找纸面的外接框。

    思路：纸是亮的，播放器黑边/状态栏/页码条是暗的。先按行求平均亮度，
    留下亮的行；再在这些行里按列求平均亮度，留下亮的列。这比"逐像素找非黑"
    稳，因为页码条上的白字不会把整行的平均亮度拉上去。

    返回 (left, top, right, bottom)；找不到纸面就返回整图。
    """
    a = np.asarray(im.convert("L"), dtype=np.float32)
    rows = np.where(a.mean(axis=1) > thresh)[0]
    if rows.size == 0:
        return (0, 0, im.width, im.height)
    t, b = int(rows[0]), int(rows[-1]) + 1
    cols = np.where(a[t:b].mean(axis=0) > thresh)[0]
    if cols.size == 0:
        return (0, t, im.width, b)
    l, r = int(cols[0]), int(cols[-1]) + 1
    if margin:
        l, t = max(0, l + margin), max(0, t + margin)
        r, b = min(im.width, r - margin), min(im.height, b - margin)
    return (l, t, r, b)


def enhance(im, target_w, sharpen, contrast, radius, threshold, grayscale, whitepoint):
    """升采样 → 提白 → USM 锐化 → 对比增强。

    顺序有讲究：先放大再锐化，笔画边缘才干净；先锐化再提对比，
    否则对比度会把锐化产生的振铃一起放大。
    """
    if target_w and im.width < target_w:
        scale = target_w / im.width
        im = im.resize((round(im.width * scale), round(im.height * scale)), Image.LANCZOS)
    if whitepoint < 255:
        # 把接近白的底色一把推到纯白，专治拍照/低质量截图的灰底。
        a = np.asarray(im.convert("RGB"), dtype=np.float32)
        a = np.clip(a * (255.0 / whitepoint), 0, 255)
        im = Image.fromarray(a.astype(np.uint8))
    if sharpen:
        im = im.filter(ImageFilter.UnsharpMask(radius=radius, percent=sharpen,
                                               threshold=threshold))
    if contrast != 1.0:
        im = ImageEnhance.Contrast(im).enhance(contrast)
    if grayscale:
        im = im.convert("L").convert("RGB")
    return im


# ---------------------------------------------------------------- survey

def cmd_survey(args):
    files = collect(args.inputs)
    if not files:
        sys.exit("没有找到图片")
    d = os.path.dirname(os.path.abspath(args.sheet))
    os.makedirs(d, exist_ok=True)

    cols = args.cols
    rows = (len(files) + cols - 1) // cols
    cw, ch = args.cell_width, args.cell_height
    sheet = Image.new("RGB", (cols * cw, rows * ch), "white")
    report = []

    for i, f in enumerate(files):
        im = Image.open(f)
        box = find_page(im, args.thresh)
        crop = im.crop(box)
        report.append({
            "index": i,
            "file": f,
            "size": list(im.size),
            "crop_box": list(box),
            "cropped_size": [box[2] - box[0], box[3] - box[1]],
            "cropped_ratio": round((box[3] - box[1]) / max(1, box[2] - box[0]), 3),
            "kept_fraction": round(((box[2] - box[0]) * (box[3] - box[1]))
                                   / (im.width * im.height), 3),
        })
        sheet.paste(crop.convert("RGB").resize((cw, ch), Image.LANCZOS),
                    ((i % cols) * cw, (i // cols) * ch))

    sheet.save(args.sheet)

    # 页脚拼贴：把每页底部一条按原始分辨率竖着贴成一列。
    # 分组判断全靠页脚的"第X页/共N页"，而缩略图联表里这行字必然糊掉——
    # 所以单独出一张放大的页脚图，保证读得出来。
    strip_path = None
    if not args.no_footer:
        crops = []
        for f, r in zip(files, report):
            im = Image.open(f).crop(tuple(r["crop_box"])).convert("RGB")
            fh = max(40, int(im.height * args.footer_frac))
            c = im.crop((0, im.height - fh, im.width, im.height))
            if args.footer_zoom != 1.0:  # 页码字号很小，放大后才读得准
                c = c.resize((round(c.width * args.footer_zoom),
                              round(c.height * args.footer_zoom)), Image.LANCZOS)
            crops.append(c)
        sw = max(c.width for c in crops)
        gap = 6
        strip = Image.new("RGB", (sw, sum(c.height for c in crops) + gap * len(crops)), "gray")
        y = 0
        for c in crops:
            strip.paste(c, (0, y))
            y += c.height + gap
        strip_path = os.path.splitext(args.sheet)[0] + "_footers.png"
        strip.save(strip_path)

    print(json.dumps({"count": len(files), "sheet": args.sheet,
                      "footer_strip": strip_path, "pages": report},
                     ensure_ascii=False, indent=2))


# ---------------------------------------------------------------- build

def cmd_build(args):
    files = collect(args.inputs)
    if not files:
        sys.exit("没有找到图片")
    os.makedirs(args.out, exist_ok=True)

    pages, manifest = [], []
    for i, f in enumerate(files, 1):
        im = Image.open(f).convert("RGB")
        before = im.size
        if not args.no_crop:
            im = im.crop(find_page(im, args.thresh, args.margin))
        im = enhance(im, args.target_width, args.sharpen, args.contrast,
                     args.radius, args.threshold, args.grayscale, args.whitepoint)
        name = f"{i:02d}.jpg"
        im.save(os.path.join(args.out, name), "JPEG", quality=args.quality,
                dpi=(args.dpi, args.dpi), optimize=True)
        pages.append(im)
        manifest.append({"page": i, "source": os.path.basename(f),
                         "before": list(before), "after": list(im.size), "out": name})

    pdf = None
    if not args.no_pdf:
        stem = args.pdf_name or os.path.basename(args.out.rstrip("/"))
        pdf = os.path.join(args.out, stem + ".pdf")
        pages[0].save(pdf, "PDF", resolution=float(args.dpi),
                      save_all=True, append_images=pages[1:])

    # 自检：处理完的图四边应该都是白的，否则说明黑边没裁干净。
    warn = []
    for m, im in zip(manifest, pages):
        a = np.asarray(im.convert("L"), dtype=np.float32)
        edges = [a[0].mean(), a[-1].mean(), a[:, 0].mean(), a[:, -1].mean()]
        if min(edges) < args.edge_min:
            warn.append({"page": m["page"], "edge_means": [round(e) for e in edges]})

    print(json.dumps({"out": args.out, "pages": len(pages), "pdf": pdf,
                      "manifest": manifest, "dark_edge_warnings": warn},
                     ensure_ascii=False, indent=2))


# ---------------------------------------------------------------- pack

def cmd_pack(args):
    d = os.path.dirname(os.path.abspath(args.zip))
    os.makedirs(d, exist_ok=True)
    n = 0
    with zipfile.ZipFile(args.zip, "w", zipfile.ZIP_DEFLATED) as z:
        for folder in args.dirs:
            base = os.path.basename(folder.rstrip("/"))
            for root, _, files in os.walk(folder):
                for f in sorted(files, key=natural_key):
                    full = os.path.join(root, f)
                    z.write(full, os.path.join(base, os.path.relpath(full, folder)))
                    n += 1
    print(json.dumps({"zip": args.zip, "files": n,
                      "size_mb": round(os.path.getsize(args.zip) / 1e6, 1)},
                     ensure_ascii=False))


# ---------------------------------------------------------------- CLI

def main():
    ap = argparse.ArgumentParser(description="试卷/讲义截图 → 打印稿")
    sub = ap.add_subparsers(dest="cmd", required=True)

    s = sub.add_parser("survey", help="生成缩略图联表 + 页脚拼贴 + 裁剪诊断")
    s.add_argument("inputs", nargs="+", help="图片/目录/zip")
    s.add_argument("--sheet", default="sheet.png")
    s.add_argument("--cols", type=int, default=6)
    s.add_argument("--cell-width", type=int, default=420)
    s.add_argument("--cell-height", type=int, default=594)
    s.add_argument("--thresh", type=int, default=150)
    s.add_argument("--footer-frac", type=float, default=0.12,
                   help="页脚拼贴取纸面底部多少比例，默认 12%%")
    s.add_argument("--footer-zoom", type=float, default=1.5)
    s.add_argument("--no-footer", action="store_true")
    s.set_defaults(func=cmd_survey)

    b = sub.add_parser("build", help="裁边+锐化+出 JPG 和 PDF")
    b.add_argument("inputs", nargs="+")
    b.add_argument("--out", required=True, help="输出目录，目录名会成为 PDF 名")
    b.add_argument("--pdf-name", default=None)
    b.add_argument("--target-width", type=int, default=2480,
                   help="目标像素宽，2480≈A4@300dpi")
    b.add_argument("--dpi", type=int, default=300)
    b.add_argument("--sharpen", type=int, default=130, help="USM 强度百分比，0=不锐化")
    b.add_argument("--radius", type=float, default=3.0)
    b.add_argument("--threshold", type=int, default=2)
    b.add_argument("--contrast", type=float, default=1.18)
    b.add_argument("--whitepoint", type=int, default=255,
                   help="<255 时把该亮度以上推成纯白，治灰底")
    b.add_argument("--grayscale", action="store_true", help="转灰度，黑白打印更省墨")
    b.add_argument("--quality", type=int, default=95)
    b.add_argument("--margin", type=int, default=0, help="裁剪框再向内收 N 像素")
    b.add_argument("--thresh", type=int, default=150)
    b.add_argument("--edge-min", type=int, default=200, help="四边平均亮度低于此值就报警")
    b.add_argument("--no-crop", action="store_true")
    b.add_argument("--no-pdf", action="store_true")
    b.set_defaults(func=cmd_build)

    p = sub.add_parser("pack", help="打包成 zip")
    p.add_argument("dirs", nargs="+")
    p.add_argument("--zip", required=True)
    p.set_defaults(func=cmd_pack)

    args = ap.parse_args()
    args.func(args)


if __name__ == "__main__":
    main()
```

