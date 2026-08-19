# Skill：用Python处理PDF文件（读取、合并、拆分、水印、加密）

> 从 Hermes Agent [PDF Skill](https://github.com/nousresearch/hermes-agent/tree/main/skills/productivity/pdf) 蒸馏简化

## 这个skill能做什么

用Python自动化处理PDF文件：读取文字、合并多个PDF、拆分PDF、添加水印、加密解密。**不用手动操作，一行代码搞定。**

## 使用场景

- 📄 老板发来10个PDF要合并成一个
- 📄 从大PDF里提取某几页
- 📄 给合同添加"机密"水印
- 📄 给PDF文件加密保护隐私
- 📄 批量提取PDF中的文字内容

## 前置要求

```bash
# 安装依赖（一键安装）
pip install pypdf reportlab pdfplumber
```

> Windows用户：打开命令提示符或PowerShell，直接粘贴上面的命令即可。

## 快速开始

### 1. 创建脚本文件

把下面的完整代码保存为 `pdf_tool.py`。

### 2. 运行命令

```bash
# 读取PDF文字内容
python pdf_tool.py read 文档.pdf

# 合并多个PDF
python pdf_tool.py merge 文件1.pdf 文件2.pdf -o 合并结果.pdf

# 拆分PDF（提取第1-3页和第7页）
python pdf_tool.py split 文档.pdf --pages 1-3,7 -o 提取页.pdf

# 添加水印
python pdf_tool.py stamp 文档.pdf --text "机密" -o 带水印.pdf

# 加密PDF
python pdf_tool.py encrypt 文档.pdf --password 123456 -o 加密后.pdf

# 解密PDF
python pdf_tool.py decrypt 加密后.pdf --password 123456 -o 解密后.pdf
```

### 3. 查看结果

运行后会在当前目录生成输出文件，用任何PDF阅读器打开即可。

## 完整代码

```python
"""
PDF处理工具 - 读取、合并、拆分、水印、加密
用法：python pdf_tool.py <命令> [参数]

命令：
  read    读取PDF文字内容
  merge   合并多个PDF文件
  split   拆分PDF文件
  stamp   添加水印文字
  encrypt 加密PDF
  decrypt 解密PDF
"""

import sys
import json
from pathlib import Path

# ========== 读取PDF文字 ==========
def read_pdf(pdf_path):
    """提取PDF中的文字内容"""
    from pdfplumber import open as pdf_open

    with pdf_open(pdf_path) as pdf:
        total_pages = len(pdf.pages)
        print(f"📄 共 {total_pages} 页\n")
        for i, page in enumerate(pdf.pages, 1):
            text = page.extract_text()
            if text:
                print(f"--- 第 {i} 页 ---")
                print(text[:500])  # 每页最多显示500字
                if len(text) > 500:
                    print("...(内容过长，已截断)")
            else:
                print(f"--- 第 {i} 页 --- (无文字内容，可能是扫描件)")

# ========== 合并PDF ==========
def merge_pdfs(input_files, output_file):
    """合并多个PDF文件"""
    from pypdf import PdfWriter

    writer = PdfWriter()
    for f in input_files:
        f_path = Path(f)
        if not f_path.exists():
            print(f"⚠️ 文件不存在: {f}")
            continue
        writer.append(str(f_path))
        print(f"✅ 已添加: {f_path.name}")

    writer.write(str(output_file))
    writer.close()
    print(f"\n🎉 合并完成，共 {len(input_files)} 个文件 → {output_file}")

# ========== 拆分PDF ==========
def split_pdf(pdf_path, pages_str, output_file):
    """按页范围拆分PDF，格式：1-3,5,7-9"""
    from pypdf import PdfReader, PdfWriter

    reader = PdfReader(pdf_path)
    total = len(reader.pages)
    writer = PdfWriter()

    # 解析页范围，如 "1-3,5,7-9"
    selected = []
    for part in pages_str.split(","):
        part = part.strip()
        if "-" in part:
            start, end = part.split("-")
            start = int(start) - 1  # 转为0-based
            end = int(end) - 1
            selected.extend(range(start, min(end, total - 1) + 1))
        else:
            selected.append(int(part) - 1)

    for i in selected:
        if 0 <= i < total:
            writer.add_page(reader.pages[i])
            print(f"✅ 添加第 {i+1} 页")

    writer.write(str(output_file))
    writer.close()
    print(f"\n🎉 拆分完成，共 {len(selected)} 页 → {output_file}")

# ========== 添加水印 ==========
def stamp_pdf(pdf_path, text, output_file,
              x=100, y=100, font_size=60, rotation=45, opacity=0.3):
    """在PDF每页添加文字水印"""
    from io import BytesIO
    from reportlab.pdfgen import canvas
    from pypdf import PdfReader, PdfWriter

    # 1. 创建水印页面
    packet = BytesIO()
    c = canvas.Canvas(packet, pagesize=(842, 595))  # A4横向
    c.setFillColorRGB(0, 0, 0, opacity)  # 透明度
    c.setFont("Helvetica", font_size)
    c.saveState()
    c.translate(x, y)
    c.rotate(rotation)
    c.drawString(0, 0, text)
    c.restoreState()
    c.save()

    packet.seek(0)
    watermark_pdf = PdfReader(packet)
    watermark_page = watermark_pdf.pages[0]

    # 2. 给每页添加水印
    reader = PdfReader(pdf_path)
    writer = PdfWriter()

    for i, page in enumerate(reader.pages):
        page.merge_page(watermark_page)
        writer.add_page(page)
        print(f"✅ 第 {i+1} 页添加水印")

    writer.write(str(output_file))
    writer.close()
    print(f"\n🎉 水印添加完成 → {output_file}")

# ========== 加密/解密PDF ==========
def encrypt_pdf(pdf_path, password, output_file):
    """加密PDF文件"""
    from pypdf import PdfReader, PdfWriter

    reader = PdfReader(pdf_path)
    writer = PdfWriter()

    for page in reader.pages:
        writer.add_page(page)

    writer.encrypt(password)
    writer.write(str(output_file))
    writer.close()
    print(f"🔒 加密完成 → {output_file}")

def decrypt_pdf(pdf_path, password, output_file):
    """解密PDF文件"""
    from pypdf import PdfReader, PdfWriter

    reader = PdfReader(pdf_path)

    if reader.is_encrypted:
        reader.decrypt(password)
        print("🔓 解密成功")

    writer = PdfWriter()
    for page in reader.pages:
        writer.add_page(page)

    writer.write(str(output_file))
    writer.close()
    print(f"🔓 解密完成 → {output_file}")


# ========== 命令行入口 ==========
if __name__ == "__main__":
    if len(sys.argv) < 2:
        print(__doc__)
        sys.exit(1)

    cmd = sys.argv[1]

    if cmd == "read" and len(sys.argv) >= 3:
        read_pdf(sys.argv[2])

    elif cmd == "merge" and len(sys.argv) >= 4:
        # 最后一个参数是 -o 输出文件，前面的都是输入文件
        args = sys.argv[2:]
        if "-o" in args:
            idx = args.index("-o")
            input_files = args[:idx]
            output_file = args[idx + 1]
        else:
            # 默认最后一个文件为输出
            input_files = args[:-1]
            output_file = args[-1]
        merge_pdfs(input_files, output_file)

    elif cmd == "split" and len(sys.argv) >= 5:
        # python pdf_tool.py split 文档.pdf --pages 1-3,7 -o 输出.pdf
        pdf_path = sys.argv[2]
        args = sys.argv[3:]
        pages_str = None
        output_file = "split_output.pdf"
        if "--pages" in args:
            idx = args.index("--pages")
            pages_str = args[idx + 1]
        if "-o" in args:
            idx = args.index("-o")
            output_file = args[idx + 1]
        if pages_str:
            split_pdf(pdf_path, pages_str, output_file)
        else:
            print("❌ 请指定页范围，例如 --pages 1-3,5")

    elif cmd == "stamp" and len(sys.argv) >= 4:
        pdf_path = sys.argv[2]
        args = sys.argv[3:]
        text = "机密"
        output_file = "stamped_output.pdf"
        if "--text" in args:
            idx = args.index("--text")
            text = args[idx + 1]
        if "-o" in args:
            idx = args.index("-o")
            output_file = args[idx + 1]
        stamp_pdf(pdf_path, text, output_file)

    elif cmd == "encrypt" and len(sys.argv) >= 5:
        pdf_path = sys.argv[2]
        args = sys.argv[3:]
        password = "123456"
        output_file = "encrypted_output.pdf"
        if "--password" in args:
            idx = args.index("--password")
            password = args[idx + 1]
        if "-o" in args:
            idx = args.index("-o")
            output_file = args[idx + 1]
        encrypt_pdf(pdf_path, password, output_file)

    elif cmd == "decrypt" and len(sys.argv) >= 5:
        pdf_path = sys.argv[2]
        args = sys.argv[3:]
        password = "123456"
        output_file = "decrypted_output.pdf"
        if "--password" in args:
            idx = args.index("--password")
            password = args[idx + 1]
        if "-o" in args:
            idx = args.index("-o")
            output_file = args[idx + 1]
        decrypt_pdf(pdf_path, password, output_file)

    else:
        print(f"❌ 未知命令或参数不足: {cmd}")
        print(__doc__)
```

## 常见问题

**Q: 安装依赖报错怎么办？**
A: 试试用管理员权限打开命令提示符再安装。或者用 `pip install --user pypdf reportlab pdfplumber`。

**Q: 读取中文PDF显示乱码？**
A: 这是正常现象，文字内容实际已提取，只是控制台显示问题。可以用 `python pdf_tool.py read 文档.pdf > 输出.txt` 保存到文件查看。

**Q: 读取PDF时显示"无文字内容"？**
A: 说明这个PDF是扫描件（图片形式），没有文字层。需要OCR工具才能识别文字。推荐使用 [PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR)。

**Q: 水印文字不在预期位置？**
A: PDF坐标系统以左下角为原点(0,0)。调整 `--x` 和 `--y` 参数的值，x越大越靠右，y越大越靠上。

**Q: 加密后的PDF怎么打开？**
A: 用任何PDF阅读器（Adobe Acrobat、Chrome浏览器等）打开，会提示输入密码。

## 进阶用法

### 批量处理文件夹内所有PDF

```python
# 批量读取文件夹内所有PDF的文字内容
import os
from pathlib import Path

pdf_dir = Path("./pdf_files")  # 改成你的文件夹路径
for pdf_file in pdf_dir.glob("*.pdf"):
    print(f"\n{'='*40}")
    print(f"处理: {pdf_file.name}")
    print(f"{'='*40}")
    os.system(f"python pdf_tool.py read \"{pdf_file}\"")
```

### 给PDF添加图片水印（如公司Logo）

```python
from pypdf import PdfReader, PdfWriter

reader = PdfReader("原文档.pdf")
writer = PdfWriter()

# 准备一个图片水印PDF（可以先用其他工具生成）
watermark = PdfReader("logo.pdf")

for page in reader.pages:
    page.merge_page(watermark.pages[0], over=True)
    writer.add_page(page)

writer.write("带Logo水印.pdf")
print("✅ 完成")
```

## 参考资源

- [pypdf 官方文档](https://pypdf.readthedocs.io/) - PDF读写操作
- [pdfplumber 文档](https://github.com/jsvine/pdfplumber) - PDF文字提取
- [reportlab 文档](https://www.reportlab.com/docs/reportlab-userguide.pdf) - PDF生成和水印
- [Hermes Agent PDF Skill 原始版](https://github.com/nousresearch/hermes-agent/tree/main/skills/productivity/pdf) - 完整版（含表单、表格提取等高级功能）