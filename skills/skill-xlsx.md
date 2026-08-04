# Excel 文件处理（Python 版）

## 这个 skill 能做什么
用 Python 自动读写 Excel 文件——创建表格、读取数据、修改内容、生成报表，完全不需要手动打开 Excel。

## 使用场景
- 学生：整理成绩单、课表、实验数据
- 打工人：处理报表、批量修改数据、生成统计表
- 开发者：爬虫数据导出、API 结果存储、自动化报表

## 前置要求
```bash
# 只需要装一个库
pip install openpyxl
```

## 快速开始

### 1. 创建 Excel 文件
```python
import openpyxl

# 创建工作簿
wb = openpyxl.Workbook()
ws = wb.active
ws.title = "学生成绩"

# 写表头
ws["A1"] = "姓名"
ws["B1"] = "语文"
ws["C1"] = "数学"
ws["D1"] = "英语"
ws["E1"] = "总分"

# 写数据
data = [
    ["张三", 85, 92, 78],
    ["李四", 90, 88, 95],
    ["王五", 76, 85, 82],
]
for row_idx, row_data in enumerate(data, start=2):
    for col_idx, value in enumerate(row_data, start=1):
        ws.cell(row=row_idx, column=col_idx, value=value)
    # 计算总分（也可以用 Excel 公式）
    ws.cell(row=row_idx, column=5).value = sum(row_data[1:])

# 保存
wb.save("学生成绩.xlsx")
print("✅ 文件已生成：学生成绩.xlsx")
```

### 2. 读取 Excel 文件
```python
import openpyxl

wb = openpyxl.load_workbook("学生成绩.xlsx")
ws = wb.active

print(f"工作表名：{ws.title}")
print(f"行数：{ws.max_row}，列数：{ws.max_column}")

# 读取所有数据
for row in ws.iter_rows(min_row=1, values_only=True):
    print(row)

# 读取特定单元格
print(f"\nA1 的值：{ws['A1'].value}")
print(f"B2 的值：{ws['B2'].value}")
```

### 3. 修改已有文件
```python
import openpyxl

wb = openpyxl.load_workbook("学生成绩.xlsx")
ws = wb.active

# 修改某个单元格
ws["B2"] = 95  # 把张三的语文改成 95

# 新增一行
ws.append(["赵六", 88, 91, 87, 266])

# 删除一行（删除第3行，王五）
ws.delete_rows(4)

wb.save("学生成绩_修改版.xlsx")
print("✅ 修改完成")
```

## 完整代码：成绩单生成器

```python
"""
成绩单生成器 - 实际可运行
功能：从字典数据生成 Excel 成绩单，带格式
"""
import openpyxl
from openpyxl.styles import Font, Alignment, Border, Side, PatternFill
from openpyxl.utils import get_column_letter


def create_score_report(data, filename="成绩报告.xlsx"):
    """从数据字典生成格式化的 Excel 成绩单"""
    
    wb = openpyxl.Workbook()
    ws = wb.active
    ws.title = "成绩单"
    
    # ========== 样式设置 ==========
    header_font = Font(name="微软雅黑", bold=True, size=12, color="FFFFFF")
    header_fill = PatternFill(start_color="4472C4", end_color="4472C4", fill_type="solid")
    header_align = Alignment(horizontal="center", vertical="center")
    
    cell_font = Font(name="微软雅黑", size=11)
    cell_align = Alignment(horizontal="center", vertical="center")
    
    thin_border = Border(
        left=Side(style="thin"),
        right=Side(style="thin"),
        top=Side(style="thin"),
        bottom=Side(style="thin"),
    )
    
    # ========== 写表头 ==========
    headers = ["姓名", "语文", "数学", "英语", "总分", "平均分"]
    for col, header in enumerate(headers, start=1):
        cell = ws.cell(row=1, column=col, value=header)
        cell.font = header_font
        cell.fill = header_fill
        cell.alignment = header_align
        cell.border = thin_border
    
    # ========== 写数据 ==========
    for row_idx, student in enumerate(data, start=2):
        name = student["name"]
        scores = student["scores"]  # [语文, 数学, 英语]
        total = sum(scores)
        avg = round(total / len(scores), 1)
        
        row_values = [name] + scores + [total, avg]
        for col, value in enumerate(row_values, start=1):
            cell = ws.cell(row=row_idx, column=col, value=value)
            cell.font = cell_font
            cell.alignment = cell_align
            cell.border = thin_border
    
    # ========== 设置列宽 ==========
    col_widths = [12, 10, 10, 10, 10, 10]
    for i, width in enumerate(col_widths, start=1):
        ws.column_dimensions[get_column_letter(i)].width = width
    
    # ========== 保存 ==========
    wb.save(filename)
    print(f"✅ 成绩单已生成：{filename}")
    print(f"  共 {len(data)} 名学生")


# ========== 使用示例 ==========
if __name__ == "__main__":
    students = [
        {"name": "张三", "scores": [85, 92, 78]},
        {"name": "李四", "scores": [90, 88, 95]},
        {"name": "王五", "scores": [76, 85, 82]},
        {"name": "赵六", "scores": [93, 87, 91]},
        {"name": "孙七", "scores": [68, 72, 88]},
    ]
    create_score_report(students)
```

## 进阶用法

### 在 Excel 中使用公式
```python
from openpyxl import Workbook
from openpyxl.utils import get_column_letter

wb = Workbook()
ws = wb.active

# 写入数据
ws["A1"] = "商品"
ws["B1"] = "单价"
ws["C1"] = "数量"
ws["D1"] = "金额"

data = [["苹果", 5.5, 10], ["香蕉", 3.0, 8], ["橘子", 4.5, 12]]
for row_idx, row_data in enumerate(data, start=2):
    for col_idx, value in enumerate(row_data, start=1):
        ws.cell(row=row_idx, column=col_idx, value=value)
    # 用公式计算金额 = 单价 * 数量
    ws.cell(row=row_idx, column=4).value = f"=B{row_idx}*C{row_idx}"

# 合计行
ws.cell(row=len(data)+2, column=3, value="合计")
ws.cell(row=len(data)+2, column=4).value = f"=SUM(D2:D{len(data)+1})"

wb.save("商品清单.xlsx")
print("公式已写入，用 Excel 打开会自动计算")
```

### 合并单元格
```python
from openpyxl import Workbook
from openpyxl.styles import Alignment

wb = Workbook()
ws = wb.active

# 合并并写标题
ws.merge_cells("A1:D1")
cell = ws["A1"]
cell.value = "2024年上学期成绩表"
cell.alignment = Alignment(horizontal="center", vertical="center")

wb.save("合并单元格示例.xlsx")
```

### 读取 CSV 转 Excel
```python
import csv
import openpyxl

# 读取 CSV
with open("data.csv", "r", encoding="utf-8") as f:
    reader = csv.reader(f)
    rows = list(reader)

# 写入 Excel
wb = openpyxl.Workbook()
ws = wb.active
for row_idx, row_data in enumerate(rows, start=1):
    for col_idx, value in enumerate(row_data, start=1):
        ws.cell(row=row_idx, column=col_idx, value=value)

wb.save("data.xlsx")
print(f"✅ CSV 已转为 Excel，共 {len(rows)} 行")
```

## 常见问题

**Q: openpyxl 能打开 .xls 文件吗？**
A: 不能。openpyxl 只支持 .xlsx/.xlsm/.xltx 格式。老版 .xls 文件需要用 `xlrd` 库读取。

**Q: 为什么我打开文件后中文乱码？**
A: openpyxl 默认支持 UTF-8，一般不会乱码。如果从 CSV 读数据，确保 CSV 编码是 UTF-8：`open("file.csv", encoding="utf-8")`。

**Q: 如何让单元格自动适应内容宽度？**
A: openpyxl 没有自动列宽功能，需要手动设置：
```python
ws.column_dimensions["A"].width = 20
```
或者遍历单元格计算最大宽度。

**Q: 写入公式后 Excel 打开没显示结果？**
A: openpyxl 写入公式后不会自动计算。用 Excel 打开后按 `Ctrl+Alt+F9` 重新计算，或者用 `data_only=True` 读取已计算过的值：
```python
wb = openpyxl.load_workbook("file.xlsx", data_only=True)
```

**Q: 如何给单元格添加颜色？**
A: 使用 PatternFill：
```python
from openpyxl.styles import PatternFill
red_fill = PatternFill(start_color="FF0000", end_color="FF0000", fill_type="solid")
cell.fill = red_fill
```

## 参考资源
- [openpyxl 官方文档](https://openpyxl.readthedocs.io/)
- [openpyxl 样式指南](https://openpyxl.readthedocs.io/en/stable/styles.html)