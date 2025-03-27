---
title: openpyxl
date: 2024-12-09 15:23:49
tags: python
categories: Posts
---

# openpyxl

## 1. 安装 OpenPyXL

```bash
pip install openpyxl
```

## 2. 基本操作

### 2.1 创建 Excel 文件

```python
from openpyxl import Workbook

# 创建一个新的工作簿
wb = Workbook()

# 激活默认工作表
ws = wb.active
ws.title = "Sheet1"  # 重命名工作表

# 写入数据
ws["A1"] = "Hello"
ws["B1"] = 123

# 保存文件
wb.save("example.xlsx")
```

### 2.2 读取 Excel 文件

```python
from openpyxl import load_workbook

# 加载已有的工作簿
wb = load_workbook("example.xlsx")

# 获取默认的工作表
ws = wb.active

# 读取单元格数据
print(ws["A1"].value)  # 输出: Hello
print(ws["B1"].value)  # 输出: 123

# 使用行列索引读取数据
print(ws.cell(row=1, column=2).value)  # 输出: 123
```

### 2.3 修改 Excel 文件

```python
from openpyxl import load_workbook

# 加载已有工作簿
wb = load_workbook("example.xlsx")
ws = wb.active

# 修改单元格数据
ws["A2"] = "World"
ws["B2"] = 456

# 保存修改
wb.save("example_modified.xlsx")
```

## 3. 进阶操作

### 3.1 操作多个工作表

```python
# 创建多个工作表
ws1 = wb.create_sheet("Sheet2")  # 创建新工作表
ws2 = wb.create_sheet("Sheet3", 0)  # 将工作表插入到第1个位置

# 获取工作表
print(wb.sheetnames)  # 输出所有工作表名

# 删除工作表
wb.remove(ws1)
```

### 3.2 迭代读取数据

```python
# 按行迭代
for row in ws.iter_rows(min_row=1, max_row=2, min_col=1, max_col=2):
    for cell in row:
        print(cell.value)

# 按列迭代
for col in ws.iter_cols(min_row=1, max_row=2, min_col=1, max_col=2):
    for cell in col:
        print(cell.value)
```

### 3.3 合并和拆分单元格

```python
# 合并单元格
ws.merge_cells("A1:C1")  # 合并A1到C1
ws["A1"] = "Merged Cells"

# 拆分单元格
ws.unmerge_cells("A1:C1")
```

### 3.4 设置单元格样式

```python

from openpyxl.styles import Font, Alignment, PatternFill

# 设置字体样式
ws["A1"].font = Font(name="Arial", size=14, bold=True, italic=True, color="FF0000")

# 设置对齐方式
ws["A1"].alignment = Alignment(horizontal="center", vertical="center")

# 设置背景颜色
ws["A1"].fill = PatternFill(fill_type="solid", start_color="FFFF00", end_color="FFFF00")
```

### 3.5 插入图表

```python
from openpyxl.chart import BarChart, Reference

# 准备数据
ws.append(["Year", "Sales"])
ws.append([2021, 100])
ws.append([2022, 200])
ws.append([2023, 300])

# 创建图表
chart = BarChart()
data = Reference(ws, min_col=2, min_row=1, max_col=2, max_row=4)
chart.add_data(data, titles_from_data=True)
ws.add_chart(chart, "E5")  # 将图表插入到E5单元格
```

## 4. 高级功能

### 4.1 使用公式

```python
ws["A3"] = "=SUM(B1:B2)"  # 插入SUM公式
```

### 4.2 数据验证

```python
from openpyxl.worksheet.datavalidation import DataValidation

# 创建数据验证规则
dv = DataValidation(type="list", formula1='"Option1,Option2,Option3"', allow_blank=True)

# 应用数据验证到单元格
ws.add_data_validation(dv)
dv.add(ws["A1"])
```

### 4.3 操作超大文件

```python
openpyxl 支持只加载部分数据，适用于处理大文件。

wb = load_workbook("large_file.xlsx", read_only=True)
ws = wb.active

for row in ws.iter_rows(values_only=True):
    print(row)
```

## 5. 常见问题

### 5.1 只加载数据，不加载公式的计算结果

```python
wb = load_workbook("example.xlsx", data_only=True)
ws = wb.active
print(ws["A3"].value)  # 输出公式计算后的结果，而不是公式
```

### 5.2 保护工作表

```python
ws.protection.sheet = True  # 启用保护
ws.protection.password = "mypassword"  # 设置密码
```

### 5.3 设置列宽和行高

```python
ws.column_dimensions["A"].width = 20  # 设置列宽
ws.row_dimensions[1.height = 30  # 设置行高
```
