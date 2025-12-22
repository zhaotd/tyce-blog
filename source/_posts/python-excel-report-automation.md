---
title: 机械工程师用 Python 自动生成周报/月报
date: 2025-12-21 19:21:59
tags: [Python, 办公自动化, 数据分析, 效率工具]
categories: [斜杠技能应用]
cover: /img/python-excel-report-automation.png
---

## 痛点：数据泥潭中的工程师

在非标自动化项目中，工程师每周都要面对一个经典且耗时的“脏活”：整合项目数据。BOM清单（物料、供应商、交期）在一个Excel里，装配调试进度表在另一个Excel里，采购订单状态可能又在第三个表格中。手动复制粘贴、核对数据、生成格式统一的周报，动辄消耗数小时。这不仅挤占了本应用于核心设计、调试和问题解决的时间，更关键的是，**人工处理极易引入错误**——一个错位的行，就可能导致采购漏项或进度误判。

这种重复劳动，正是自动化脚本发挥价值的绝佳场景。我们的目标不是成为Excel专家，而是让工具替我们完成繁琐的数据搬运和格式化工作。

## 解决方案：用 Python 搭建自动化数据流水线

我们将使用两个核心库：`pandas` 负责数据的“粗加工”（读取、清洗、计算），`openpyxl` 或 `xlsxwriter` 负责“精加工”（生成格式规范的报告）。整个流程如同一条自动化产线。

### 第一步：数据读取与清洗 (Pandas)

首先，使用 `pandas` 读取多个来源的 Excel 文件。假设我们有 `bom.xlsx` 和 `schedule.xlsx`。

```python
import pandas as pd

# 读取数据
df_bom = pd.read_excel('bom.xlsx', sheet_name='Sheet1')
df_schedule = pd.read_excel('schedule.xlsx')

# 数据清洗是关键步骤，例如：
# 1. 统一日期格式
df_bom['预计到货日'] = pd.to_datetime(df_bom['预计到货日'], errors='coerce')
# 2. 处理缺失值（如供应商信息为空的关键物料）
critical_bom = df_bom[df_bom['关键等级'] == 'A']
missing_supplier = critical_bom[critical_bom['供应商'].isna()]
if not missing_supplier.empty:
    print("警告：以下A类物料供应商信息缺失：")
    print(missing_supplier[['物料编码', '名称']])
# 3. 数据合并：通过“项目号”或“物料编码”关联BOM与进度
df_merged = pd.merge(df_bom, df_schedule, on='项目号', how='left')
```

**工程思维**：这里的清洗逻辑直接对应质量管理。`errors='coerce'` 将无效日期转为空值，防止后续计算崩溃；检查A类物料信息完整性，是风险管控的前置动作。

### 第二步：核心数据分析与计算

基于合并后的数据，我们可以进行有意义的聚合计算，为周报提供核心内容。

```python
# 计算项目总体进度（假设进度表中有关键节点完成状态）
completion_rate = df_schedule['完成状态'].value_counts(normalize=True).get('已完成', 0) * 100

# 统计未来一周待到货的关键物料
from datetime import datetime, timedelta
next_week = datetime.now() + timedelta(days=7)
upcoming_critical = df_bom[
    (df_bom['关键等级'] == 'A') &
    (df_bom['预计到货日'] <= next_week) &
    (df_bom['到货状态'] != '已收货')
]

# 按供应商统计采购金额，用于风险分散分析
supplier_summary = df_bom.groupby('供应商')['总价(元)'].sum().sort_values(ascending=False)
```

### 第三步：生成格式化周报 (OpenPyXL/XlsxWriter)

`pandas` 的 `to_excel` 功能简单，但格式丑陋。我们需要 `openpyxl` 来制作一份具有专业外观的报告，包含标题、表格、甚至图表。

```python
from openpyxl import Workbook
from openpyxl.chart import BarChart, Reference
from openpyxl.styles import Font, Alignment, Border, Side, PatternFill

# 创建 workbook 和 sheet
wb = Workbook()
ws = wb.active
ws.title = "项目周报"

# 1. 写入标题和摘要信息
ws['A1'] = "自动化项目X周报"
ws['A1'].font = Font(size=14, bold=True)
ws['A3'] = f"总体进度：{completion_rate:.1f}%"
ws['A4'] = f"未来一周待到货A类物料数：{len(upcoming_critical)}项"

# 2. 写入待办事项表格
ws['A6'] = "本周重点关注（A类物料待到货）"
table_start_row = 7
upcoming_critical.to_excel(writer, sheet_name='周报', startrow=table_start_row-1, index=False)
# 对表格进行格式化：加粗表头、添加边框、设置对齐方式
for row in ws.iter_rows(min_row=table_start_row, max_row=table_start_row+len(upcoming_critical), min_col=1, max_col=len(upcoming_critical.columns)):
    for cell in row:
        cell.border = Border(left=Side(style='thin'), right=Side(style='thin'), top=Side(style='thin'), bottom=Side(style='thin'))
        cell.alignment = Alignment(horizontal='center', vertical='center')

# 3. 插入图表（供应商采购金额分布）
chart = BarChart()
chart.title = "供应商采购金额 Top 5"
chart.x_axis.title = "供应商"
chart.y_axis.title = "金额 (元)"
# 准备图表数据（取前5名）
chart_data = Reference(ws, min_col=supplier_summary.index, min_row=..., max_row=...) # 需根据实际数据位置调整
chart.add_data(chart_data, titles_from_data=True)
ws.add_chart(chart, "J2")

# 保存文件
report_name = f"Project_X_Weekly_Report_{datetime.now().strftime('%Y%m%d')}.xlsx"
wb.save(report_name)
print(f"周报已生成：{report_name}")
```

**关于库的选择**：`openpyxl` 适合读写和编辑现有 `.xlsx` 文件，功能全面；`xlsxwriter` 则专注于写入和格式化，性能更优，图表功能更强。对于纯生成的报告，`xlsxwriter` 是更专业的选择。

## 结论：让 AI 与代码处理表格，让人专注于决策与创造

通过上述流程，我们将数小时的手工劳动压缩为一次脚本执行（可能只需几秒）。生成的标准周报，数据准确、格式统一、重点突出。

这背后的理念，正是现代工程师效率进化的核心：**将重复性、规则明确的“操作工”工作交给代码和AI，而工程师自身则聚焦于那些更需要经验、创造力和判断力的“工艺师”工作**。例如，从周报中识别出“某单一供应商占比过高”的供应链风险，或是根据进度滞后推断出设计环节的潜在问题，并提出解决方案。

自动化脚本的价值不止于节省时间，更在于**提升决策质量**。它消除了人为失误的噪音，让基于数据的决策更加清晰可靠。在非标自动化这个充满不确定性的领域，拥有一套可靠的数据自动化流水线，就如同为你的项目团队增加了一位永不疲倦、绝对精准的助理工程师。