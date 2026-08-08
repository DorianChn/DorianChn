# Skill：用Python自动生成Word文档

> 从 Hermes Agent 的 `docx` skill 蒸馏而来，改用纯 Python，零基础友好

## 这个skill能做什么

用Python自动生成、编辑Word文档（.docx）—— 报告、简历、批量信函、表格报告，一行代码都不用手动敲Word。

## 使用场景

- **批量生成报告**：每天的数据分析结果自动输出Word
- **自动填表**：几十份合同/邀请函，换个名字就生成一份
- **简历生成**：结构化数据→格式化简历
- **作业/论文**：自动排版标题、正文、表格

## 前置要求

```bash
# 安装 python-docx
pip install python-docx
```

## 快速开始

```python
from docx import Document

# 1. 创建文档
doc = Document()

# 2. 加标题
doc.add_heading('我的第一个文档', level=0)

# 3. 加正文
doc.add_paragraph('这是用Python自动生成的。')

# 4. 保存
doc.save('output.docx')
print('✅ 生成成功：output.docx')
```

运行后，当前目录会出现一个 `output.docx`，双击就能打开。

## 完整代码

### 1. 基础文档（标题+正文+列表+图片）

```python
from docx import Document
from docx.shared import Inches, Pt, RGBColor
from docx.enum.text import WD_ALIGN_PARAGRAPH

doc = Document()

# === 标题 ===
doc.add_heading('数据分析报告', level=0)

# === 小标题 ===
doc.add_heading('一、数据概况', level=1)
doc.add_heading('1.1 数据来源', level=2)

# === 正文（带格式） ===
p = doc.add_paragraph()
run = p.add_run('本次分析的数据来自用户行为日志，')
run.font.size = Pt(12)  # 字号
run.font.color.rgb = RGBColor(0x33, 0x33, 0x33)  # 颜色
run = p.add_run('共采集10000条记录。')
run.bold = True  # 加粗

# === 列表 ===
doc.add_heading('1.2 主要指标', level=2)
doc.add_paragraph('日活跃用户数（DAU）', style='List Bullet')
doc.add_paragraph('用户平均停留时长', style='List Bullet')
doc.add_paragraph('转化率（CVR）', style='List Bullet')

# === 有序列表 ===
doc.add_paragraph('第一步：数据清洗', style='List Number')
doc.add_paragraph('第二步：特征工程', style='List Number')
doc.add_paragraph('第三步：模型训练', style='List Number')

# === 图片 ===
# doc.add_picture('chart.png', width=Inches(5))  # 取消注释即可用
# 最后一张图片右对齐
# last_p = doc.paragraphs[-1]
# last_p.alignment = WD_ALIGN_PARAGRAPH.CENTER

doc.save('基础报告.docx')
print('✅ 已生成：基础报告.docx')
```

### 2. 表格（核心功能，最常用）

```python
from docx import Document
from docx.shared import Cm
from docx.enum.table import WD_TABLE_ALIGNMENT

doc = Document()
doc.add_heading('销售数据表', level=1)

# 准备数据
headers = ['月份', '销售额', '利润', '增长率']
data = [
    ['1月', '¥12,000', '¥3,600', '12%'],
    ['2月', '¥15,000', '¥4,500', '25%'],
    ['3月', '¥18,000', '¥5,400', '20%'],
    ['4月', '¥22,000', '¥6,600', '22%'],
]

# 创建表格（行数=数据+表头，列数=4）
table = doc.add_table(rows=1 + len(data), cols=4)
table.style = 'Light Grid Accent 1'  # 内置样式
table.alignment = WD_TABLE_ALIGNMENT.CENTER  # 表格居中

# 写表头
for i, header in enumerate(headers):
    cell = table.rows[0].cells[i]
    cell.text = header
    # 表头加粗
    for paragraph in cell.paragraphs:
        for run in paragraph.runs:
            run.bold = True

# 写数据
for row_idx, row_data in enumerate(data):
    for col_idx, value in enumerate(row_data):
        table.rows[row_idx + 1].cells[col_idx].text = value

# 设置列宽
for row in table.rows:
    row.cells[0].width = Cm(3)
    row.cells[1].width = Cm(4)
    row.cells[2].width = Cm(4)
    row.cells[3].width = Cm(4)

doc.save('表格示例.docx')
print('✅ 已生成：表格示例.docx')
```

### 3. 批量生成（核心场景：模板+数据）

```python
from docx import Document
import os

# 假设这是数据库查出来的数据
students = [
    {'name': '张三', 'score': 95, 'rank': 'A'},
    {'name': '李四', 'score': 82, 'rank': 'B'},
    {'name': '王五', 'score': 88, 'rank': 'B'},
]

# 创建输出目录
os.makedirs('成绩单', exist_ok=True)

for student in students:
    doc = Document()

    # 标题
    doc.add_heading('学生成绩通知单', level=0)

    # 正文（用学生数据填充）
    doc.add_paragraph(f'{student["name"]} 同学：')
    doc.add_paragraph(
        f'本学期你的总评成绩为 {student["score"]} 分，'
        f'等级为 {student["rank"]}。'
    )
    doc.add_paragraph('请继续努力！')

    # 落款
    doc.add_paragraph('教务处')
    doc.add_paragraph('2026年8月')

    # 用学生名字命名文件
    doc.save(f'成绩单/{student["name"]}成绩单.docx')
    print(f'✅ 已生成：{student["name"]}成绩单.docx')

print('🎉 全部生成完成！')
```

### 4. 页面设置 + 页眉页脚

```python
from docx import Document
from docx.shared import Cm, Pt
from docx.enum.section import WD_ORIENT

doc = Document()

# 页面设置
section = doc.sections[0]
section.page_width = Cm(21)     # A4宽
section.page_height = Cm(29.7)  # A4高
section.top_margin = Cm(2.54)   # 上边距
section.bottom_margin = Cm(2.54)
section.left_margin = Cm(3.18)
section.right_margin = Cm(3.18)

# 页眉
header = section.header
hp = header.paragraphs[0]
hp.text = '机密文件 - 请勿外传'
hp.alignment = 1  # 居中（0=左, 1=中, 2=右）

# 页脚
footer = section.footer
fp = footer.paragraphs[0]
fp.text = '第 1 页'
fp.alignment = 1

doc.add_heading('正式文档', level=1)
doc.add_paragraph('这是设置了页边距和页眉页脚的文档。')

doc.save('正式文档.docx')
print('✅ 已生成：正式文档.docx')
```

## 常见问题

### Q1: 安装python-docx失败？
```bash
# 试试换源
pip install python-docx -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### Q2: 表格里的中文显示乱码？
python-docx 默认支持中文，不需要额外设置。如果打开Word显示乱码，检查Word的编码设置。

### Q3: 怎么设置字体？
```python
from docx.shared import Pt
run = paragraph.add_run('文字')
run.font.name = '微软雅黑'
run.font.size = Pt(12)
```

### Q4: 怎么加粗/斜体/下划线？
```python
run.bold = True       # 加粗
run.italic = True     # 斜体
run.underline = True  # 下划线
```

### Q5: 怎么改表格背景色？
```python
from docx.oxml.ns import qn
shading = cell._element.get_or_add_tcPr()
shading_elem = shading.makeelement(qn('w:shd'), {
    qn('w:fill'): 'D9E2F3',  # 浅蓝色
    qn('w:val'): 'clear',
})
shading.append(shading_elem)
```

### Q6: 怎么读取已有文档？
```python
doc = Document('已有文件.docx')
for p in doc.paragraphs:
    print(p.text)
for table in doc.tables:
    for row in table.rows:
        for cell in row.cells:
            print(cell.text)
```

## 进阶用法

### 合并多个文档
```python
from docx import Document

def merge_docx(file_list, output_path):
    """合并多个docx文件"""
    merged = Document()
    for file in file_list:
        doc = Document(file)
        for element in doc.element.body:
            merged.element.body.append(element)
    merged.save(output_path)
```

### 模板替换（自动填合同）
```python
from docx import Document

def fill_template(template_path, data, output_path):
    """用dict替换文档中的 {占位符}"""
    doc = Document(template_path)
    for p in doc.paragraphs:
        for key, value in data.items():
            if f'{{{key}}}' in p.text:
                p.text = p.text.replace(f'{{{key}}}', str(value))
    doc.save(output_path)

# 用法：模板里写 {姓名}、{日期}，然后传dict
data = {'姓名': '张三', '日期': '2026-08-08'}
fill_template('合同模板.docx', data, '张三合同.docx')
```

## 参考资源

- [python-docx官方文档](https://python-docx.readthedocs.io/)
- [Hermes Agent 原版 docx skill](https://github.com/NousResearch/hermes-agent/tree/main/skills/productivity/docx)
- 内置样式列表：`docx.styles.BUILTIN_STYLES`