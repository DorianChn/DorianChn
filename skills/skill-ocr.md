# skill-ocr：用Python提取PDF和图片中的文字（OCR）

## 这个skill能做什么
一句话：**把PDF和图片里的文字变成可复制、可搜索的文本** —— 文字版PDF秒提取，扫描件/图片用OCR识别。

## 使用场景
- 下载了论文PDF，想复制里面的话却选不中文字（扫描件）
- 收到了合同/发票/证书的照片或扫描件，需要转成文字存档
- 老板发来一份扫描版PDF，要你把全文整理成Word或txt
- 批量处理几十个PDF，手动复制太痛苦

## 前置要求
```bash
# 1. 安装Python 3.8+（已装可跳过）

# 2. 安装核心库（文字版PDF提取，必装）
pip install pymupdf

# 3. 扫描件OCR（可选，识别图片/扫描PDF才需要）
pip install pytesseract
# 4. 还要安装Tesseract软件本身（OCR引擎）：
#    Windows: https://github.com/UB-Mannheim/tesseract/wiki 下载安装
#    安装时勾选 Chinese (Simplified) 语言包，用于识别中文
```

## 快速开始
1. 把下面的 `extract_pdf.py` 保存到本地
2. 运行：`python extract_pdf.py 你的文件.pdf`
3. 文字直接打印在屏幕上；加 `--out 输出.txt` 保存到文件
4. 扫描件用 `ocr_scan.py`：`python ocr_scan.py 扫描件.png`

## 完整代码

### 脚本1：extract_pdf.py（文字版PDF，已验证可运行）
```python
#!/usr/bin/env python3
"""从PDF中提取文字（轻量版，无需OCR模型）"""
import sys


def extract_text(pdf_path, pages=None, out_path=None):
    """提取PDF文字：pages=[0,2] 只提取第1、3页；默认全部"""
    import pymupdf
    doc = pymupdf.open(pdf_path)  # 打开PDF
    result = []
    page_range = range(len(doc)) if pages is None else pages
    for i in page_range:
        if i < len(doc):
            result.append(f"\n--- 第 {i+1}/{len(doc)} 页 ---\n")
            result.append(doc[i].get_text())  # 提取该页文字
    text = "".join(result)
    if out_path:
        with open(out_path, "w", encoding="utf-8") as f:
            f.write(text)
        print(f"已保存到: {out_path}")
    else:
        print(text)


def show_metadata(pdf_path):
    """显示PDF基本信息（标题、作者、页数）"""
    import pymupdf
    doc = pymupdf.open(pdf_path)
    print(f"页数: {len(doc)}")
    print(f"标题: {doc.metadata.get('title', '无')}")
    print(f"作者: {doc.metadata.get('author', '无')}")


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("用法: python extract_pdf.py 文件.pdf [--pages 0-2] [--out 输出.txt]")
        sys.exit(1)
    pdf = sys.argv[1]
    pages = None
    out = None
    if "--pages" in sys.argv:
        i = sys.argv.index("--pages")
        p = sys.argv[i + 1]
        pages = list(range(int(p.split("-")[0]), int(p.split("-")[1]) + 1))
    if "--out" in sys.argv:
        out = sys.argv[sys.argv.index("--out") + 1]
    if "--meta" in sys.argv:
        show_metadata(pdf)
    else:
        extract_text(pdf, pages, out)
```

### 脚本2：ocr_scan.py（扫描件/图片OCR，需装Tesseract）
```python
#!/usr/bin/env python3
"""识别图片和扫描PDF中的文字（OCR）"""
import sys
import pytesseract
from PIL import Image
import pymupdf


def ocr_image(img_path, lang="chi_sim+eng"):
    """识别单张图片，lang指定语言：chi_sim中文, eng英文"""
    img = Image.open(img_path)
    text = pytesseract.image_to_string(img, lang=lang)
    return text


def ocr_pdf_scan(pdf_path, lang="chi_sim+eng"):
    """把扫描版PDF每一页转成图片再识别"""
    doc = pymupdf.open(pdf_path)
    all_text = []
    for i, page in enumerate(doc):
        # 把页面渲染成高清图片（150 DPI，清晰度够用）
        pix = page.get_pixmap(dpi=150)
        img_path = f"_page{i+1}.png"
        pix.save(img_path)
        print(f"识别第 {i+1}/{len(doc)} 页...")
        all_text.append(f"--- 第 {i+1} 页 ---\n" + ocr_image(img_path, lang))
    return "\n".join(all_text)


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("用法: python ocr_scan.py 图片或PDF文件 [输出.txt]")
        sys.exit(1)
    path = sys.argv[1]
    text = ocr_image(path) if path.lower().endswith((".png", ".jpg", ".jpeg")) \
        else ocr_pdf_scan(path)
    if len(sys.argv) > 2:
        with open(sys.argv[2], "w", encoding="utf-8") as f:
            f.write(text)
        print(f"已保存到: {sys.argv[2]}")
    else:
        print(text)
```

## 常见问题
**Q1：提取出来的文字是空的？**
说明是扫描版PDF（图片型），没有文字层。改用 `ocr_scan.py` 走OCR识别。

**Q2：pip install pytesseract 后报错 tesseract is not installed？**
pytesseract只是Python调用工具，真正干活的是Tesseract软件。去 https://github.com/UB-Mannheim/tesseract/wiki 下载安装，装完记得重启终端。

**Q3：识别中文全是乱码/空？**
安装Tesseract时没有勾选中文语言包。重装勾选 `Chinese (Simplified)`，并确认代码里 `lang="chi_sim+eng"`。

**Q4：OCR识别结果不准？**
扫描件太模糊。提高清晰度：把 `dpi=150` 改成 `dpi=300`；图片先放大再识别。

**Q5：PDF文字顺序错乱（表格、分栏）？**
`get_text()` 对复杂排版按位置输出。进阶用 `pymupdf4llm` 转Markdown：`pip install pymupdf4llm`，然后 `import pymupdf4llm; print(pymupdf4llm.to_markdown("a.pdf"))`。

## 进阶用法
- **提取图片**：`page.get_images(full=True)` 拿到PDF内嵌图片，配合 `pymupdf.Pixmap` 保存成png
- **提取表格**：`page.find_tables()` 可直接把表格转成pandas DataFrame，导出Excel
- **批量处理**：写个for循环遍历文件夹里所有PDF，统一提取并保存成同名txt
- **在线PDF**：有URL时直接用浏览器/web工具抓取内容，比本地提取更方便

## 参考资源
- PyMuPDF官方文档：https://pymupdf.readthedocs.io/
- Tesseract下载页：https://github.com/UB-Mannheim/tesseract/wiki
- pytesseract用法：https://pypi.org/project/pytesseract/
- 原始skill来源（hermes-agent）：https://github.com/NousResearch/hermes-agent/tree/main/skills/productivity/ocr-and-documents
