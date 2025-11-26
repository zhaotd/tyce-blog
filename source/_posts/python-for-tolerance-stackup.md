---
title: Excel 算累了？用 Python 做蒙特卡洛公差分析
date: 2025-11-26 18:01:29
tags: [Python, 公差分析, 蒙特卡洛, 质量控制, 成本优化]
categories: [斜杠技能]
cover: /img/python-for-tolerance-stackup.png
---

在设计评审会上，一个常见的场景是：为了保证最终装配精度，工程师在图纸上标注了极其严苛的公差，比如 `+/-0.02mm`。为什么？因为他们使用了最传统、最保险的**极值法（Worst Case）**进行公差分析。这种方法假设所有零件都同时处于最大或最小的尺寸极限，虽然能100%保证装配成功，但在现实世界中，这种极端情况发生的概率微乎其微。我们却为此付出了高昂的代价：精密的CNC加工、昂贵的检测设备，以及因无法达标而产生的废品。

这种思维方式，是设计上的“懒惰”，更是成本控制上的灾难。作为懂代码的机械工程师，我们必须用数据思维对这种传统模式进行一次“降维打击”。

### 抛弃极值法，拥抱概率统计

核心问题在于，机械加工的零件尺寸并非一个固定值，而是在标称值附近呈现一种统计分布，最常见的就是**正态分布（Normal Distribution）**。公差范围，本质上是这个分布的边界。例如，一个成熟的加工过程可以保证 99.73% 的零件尺寸落在 ±3σ (Sigma) 的范围内。

既然单个零件的尺寸是随机的，那么多个零件装配后的总尺寸，也必然是一个随机变量。我们要做的，不是计算那个几乎不可能发生的极值，而是计算总尺寸落在合格范围内的**概率**。这就是**蒙特卡洛模拟（Monte Carlo Simulation）**的用武之地。

它的原理很简单：通过计算机进行大量的“虚拟装配”：
1.  为每个零件的尺寸，根据其公差定义一个正态分布模型。
2.  模拟一次装配：为每个零件从其分布中随机抽取一个尺寸值。
3.  将这些随机尺寸累加，得到一个装配总尺寸。
4.  将上述过程重复十万次甚至上百万次。
5.  统计这十万个总尺寸的结果，分析其分布，从而得出最终的**理论合格率**。

### 用 Python 让数据说话

假设我们有一个由10个零件线性叠加的组件，设计要求最终总长度的公差为 `±0.8mm`。

如果用极值法，每个零件的公差必须控制在 `±0.8mm / 10 = ±0.08mm` 以内。
现在，我们尝试将每个零件的公差放宽到 `+/-0.15mm`，看看会发生什么。

下面是一段简单的 Python 代码，使用 `numpy` 库进行模拟：

```python
import numpy as np

# --- 参数定义 ---
num_simulations = 100000  # 模拟次数
num_parts = 10            # 零件数量

# 每个零件的标称尺寸和公差 (假设标称尺寸均为20mm)
part_nominal_dim = 20.0
part_tolerance = 0.15  # 放宽后的公差 +/-0.15mm

# 假设公差范围对应 3-sigma (99.73%)
# 标准差 = 公差 / 3
part_std_dev = part_tolerance / 3

# 最终装配要求的公差
assembly_nominal_dim = num_parts * part_nominal_dim
assembly_tolerance = 0.8
upper_limit = assembly_nominal_dim + assembly_tolerance
lower_limit = assembly_nominal_dim - assembly_tolerance

# --- 蒙特卡洛模拟 ---
successful_assemblies = 0
for _ in range(num_simulations):
    # 为10个零件生成随机尺寸
    actual_dims = np.random.normal(loc=part_nominal_dim, 
                                   scale=part_std_dev, 
                                   size=num_parts)
    
    # 计算装配总尺寸
    assembly_dim = np.sum(actual_dims)
    
    # 判断是否合格
    if lower_limit <= assembly_dim <= upper_limit:
        successful_assemblies += 1

# --- 输出结果 ---
pass_rate = (successful_assemblies / num_simulations) * 100
print(f"模拟 {num_simulations} 次装配...")
print(f"零件公差放宽至 ±{part_tolerance}mm")
print(f"最终装配合格率: {pass_rate:.4f}%")
```

运行这段代码，你会得到一个惊人的结果：即使每个零件的公差放宽了近一倍，最终装配的合格率依然高达 **99.9%** 以上。

### 从工程师到成本优化师

现在，你可以拿着这个数据去找你的老板或项目经理，进行一次完全不同的对话。

**传统汇报：**“为了100%保证装配，零件公差必须做到+/-0.08mm。”
**数据驱动的汇报：**“老板，我模拟了十万次装配。如果我们把零件公差从+/-0.08mm放宽到+/-0.15mm，单个零件的加工成本能从50元降到25元，而我们的最终装配合格率依然能维持在99.9%以上。为了追求那理论上0.1%的风险，我们是否值得让整个项目的零件成本翻倍？”

这就是数据驱动设计的力量。它将一个模糊的、依赖经验的公差分配问题，转化为一个清晰的、可量化的风险与成本的权衡问题。通过代码，我们不仅设计物理世界的实体，更在数字世界中预测和优化它的表现，这才是现代机械工程师的核心竞争力。