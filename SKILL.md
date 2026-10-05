---
title: 从扫描版+zh_pages 翻译生成中文电子书 PDF
agent_created: true
description: 已有俄文/外文扫描版 PDF 和每页中文译文（zh_pages/pg_NNN.txt）时，用 reportlab + PyMuPDF 重排成一本嵌入中文字体、可阅读的电子书 PDF。
version: 1.0.0
---

## 触发条件
- 用户已有 OCR 后的外文扫描版 PDF，以及按页拆分的中文译文 `zh_pages/pg_NNN.txt`。
- 之前的双语 PDF 把中文做成了**隐形可搜索层**（`render_mode=3`），用户打开只能看到原文扫描图，要求“有中文”。
- 目标：生成一本**直接可读**的中文电子书 PDF（保留原书插图/线图/照片）。

## 依赖
- Python 3.x
- `pip install reportlab PyMuPDF`
- Windows 中文字体，推荐 `C:/Windows/Fonts/msyh.ttc` + `msyhbd.ttc`（微软雅黑 + 粗体）。

## 核心步骤

### 1. 识别图版页
两种方式取并集：
- OCR 阶段标记的 `[插图 位于PDF第X页]` → 得到原书页码集合 `FIG_MARKERS`。
- 中文译文中残留的处理备注，如 `[本页为驱逐舰照片页]` → 得到另一集合 `BRACKET`。
- 图版页 = `FIG_MARKERS | BRACKET`。

### 2. 注册中文字体（reportlab）
```python
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
pdfmetrics.registerFont(TTFont("CN", r"C:/Windows/Fonts/msyh.ttc", subfontIndex=0))
pdfmetrics.registerFont(TTFont("CN-Bold", r"C:/Windows/Fonts/msyhbd.ttc", subfontIndex=0))
pdfmetrics.registerFontFamily("CN", normal="CN", bold="CN-Bold", italic="CN", boldItalic="CN-Bold")
```

### 3. 中文段落样式关键
- 正文样式必须设置 `wordWrap='CJK'`，否则 reportlab 不会在汉字之间自动换行。
- 建议两端对齐 `alignment=TA_JUSTIFY`、行高 leading ≈ 1.6×字号。

### 4. 从原扫描 PDF 取图
```python
import fitz
doc = fitz.open(src_pdf)
page = doc[pg - 1]  # pg 为原书 1 起页码
pix = page.get_pixmap(matrix=fitz.Matrix(dpi/72, dpi/72), alpha=False)
pix.save(f"fig_{pg:03d}.jpg", output="jpeg", jpg_quality=85)
```
- 建议 `dpi=150`，`jpg_quality=85`，在清晰度与文件大小间平衡。
- 按页宽缩放后嵌入：`width=content_width`，保持宽高比，最大高度约 600pt。

### 5. 正文页处理（合并连续行）
`zh_pages/pg_NNN.txt` 通常每行是一个段落，但一个段落可能被截断成多行。建议：
- 连续的非空普通行合并成一个 `Paragraph`（CJK 不需要空格）。
- 遇到空行、标题、图版页时先 flush 当前段落。

### 6. 标题识别规则
- 一级标题：`第[一二三四五六七八九十百零0-9]+[章節节]` 或 `附录/序言/引言/目录/参考书目/缩略语/出版说明/后记`。
- 二级标题：行首为数字+点，如 `1.`、`.1.`、`1、` 等。

### 7. 图版页渲染
- 先放扫描图，再放中文图注。
- **不要把图注里的“第...章”误判为正文标题**；图版页上的文字统一用图注样式（小字号、灰色）。
- 剔除方括号处理备注，如 `[本页为封面，含船舶照片插图]`。

### 8. 输出与验证
- 输出文件示例：`ocr_zh_ebook.pdf`。
- 验证项：
  - 页数合理（通常远少于原页数，因为文字可连续流动）。
  - 提取文本含大量 CJK 字符，且字体为 MicrosoftYaHei（TrueType 子集嵌入）。
  - 无 `[本页为` 等处理备注残留。
  - 图片数量与图版页数一致。

## 关键坑
1. **隐形层不等于可见**：PyMuPDF `render_mode=3` 适合搜索，不适合阅读。电子书必须使用普通渲染。
2. **CJK 换行**：reportlab 默认按空格换行，中文必须 `wordWrap='CJK'`。
3. **TTC 字体索引**：`msyh.ttc` 是字体集合，注册时必须加 `subfontIndex=0`。
4. **图版页标题污染**：图注常含“第一章 ……”，若按标题样式渲染会造成满屏重复大标题。
5. **PyMuPDF JPEG 参数名**：保存 JPEG 用 `jpg_quality=`，不是 `jpeg_quality`。

## 参考脚本
- `D:/ocr_work/build_ebook.py`（本项目实际使用的重排脚本）。
