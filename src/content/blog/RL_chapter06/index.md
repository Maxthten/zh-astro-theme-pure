---
title: "RL学习笔记：随机近似与随机梯度下降"
publishDate: 2026-02-19 21:50:00
description: "梳理随机近似理论与Robbins-Monro算法，推导随机梯度下降（SGD）的演变过程与收敛特性，并对比BGD、MBGD与SGD的采样差异。"
tags: ["Stochastic Approximation", "SGD", "Robbins-Monro", "Optimization", "学习笔记"]
language: "中文"
---



# 随机近似理论与随机梯度下降

## 均值估计 (Mean Estimation)

首先，考虑样本均值的估计问题：

$$
w_{k+1} \doteq \frac{1}{k} \sum_{i=1}^{k} x_{i}, \quad k=1,2, \dots
$$

对于上一步的均值，我们有：

$$
w_{k} = \frac{1}{k-1} \sum_{i=1}^{k-1} x_{i}, \quad k=2,3, \dots
$$

通过代数变换，可以将全量计算转化为递推形式：

$$
w_{k+1} = \frac{1}{k} \sum_{i=1}^{k} x_{i} = \frac{1}{k}\left(\sum_{i=1}^{k-1} x_{i}+x_{k}\right) = \frac{1}{k}\left((k-1) w_{k}+x_{k}\right) = w_{k}-\frac{1}{k}\left(w_{k}-x_{k}\right)
$$

即：

$$
w_{k+1} = w_k - \frac{1}{k}(w_k - x_k)
$$

* 这是一种迭代算法，无需保存并重复计算所有历史数据。
* 在迭代初期，由于样本量不足，估计值可能不够精确；但随着样本量 $k$ 的增加，结果会逐渐逼近真实均值。

由此可以引出一个更广义的迭代等式：

$$
w_{k+1} = w_{k} - \alpha_{k}(w_{k} - x_{k})
$$

* 当步长序列 $\{\alpha_k\}$ 满足一定条件时，估计值 $w_k$ 依然会收敛于期望 $\mathbb{E}[X]$。
* 这种形式可以被视为随机近似（SA）算法的一种特例，同时也是随机梯度下降（SGD）算法的基础形态。

## Robbins-Monro (RM) 算法

### 随机近似 (Stochastic Approximation, SA)

* SA 是一大类依赖随机迭代来求解方程根或最优化问题的算法。
* SA 的核心优势在于**黑盒求解**：无需知晓目标方程的解析表达式或全局性质，仅依赖带噪声的观测数据即可进行参数更新。

### RM 算法定义

寻找函数极值的一个必要条件是梯度为零，即 $g(w) \doteq \nabla_{w} J(w) = 0$。同理，对于 $g(w) = c$ 的情况，可以通过移项转化为求根问题。SGD 正是一种特殊的 RM 算法。

RM 算法通过以下迭代格式求解 $g(w) = 0$：

$$
w_{k+1} = w_k - a_k \tilde{g}(w_k, \eta_k), \quad k = 1, 2, 3, \dots
$$

其中：

* $w_k$ 是第 $k$ 次迭代对根的估计值。
* $\tilde{g}(w_k, \eta_k) = g(w_k) + \eta_k$ 是第 $k$ 次带有随机噪声 $\eta_k$ 的观测值。
* $a_k$ 是控制步长的正系数。

### 收敛性定理与条件解析

若以下条件成立：

(a) $0 < c_1 \leq \nabla_w g(w) \leq c_2$，对于所有 $w$；
(b) $\sum_{k=1}^{\infty} a_k = \infty$ 且 $\sum_{k=1}^{\infty} a_k^2 < \infty$；
(c) $\mathbb{E}[\eta_k | \mathcal{H}_k] = 0$ 且 $\mathbb{E}[\eta_k^2 | \mathcal{H}_k] < \infty$；

其中 $\mathcal{H}_k = \{w_k, w_{k-1}, \dots\}$ 表示直到时刻 $k$ 的历史记录，那么序列 $w_k$ 将**以概率 1**（Almost surely）收敛于满足 $g(w^*) = 0$ 的根 $w^*$。

* **条件 (a)**：要求函数 $g(w)$ 的导数（或梯度）有严格的上下界。这保证了函数足够平滑，且其斜率不会趋于零或无穷大，从而确保更新方向能稳定且持续地指向目标解 $w^*$。
* **条件 (b)**：经典的步长（学习率）约束。$\sum a_k = \infty$ 保证了步长总和无限大，算法有能力跨越任意初始距离到达目标点；$\sum a_k^2 < \infty$ 则保证步长衰减得足够快，使得算法最终能够收敛，避免在根节点附近永久震荡。
* **条件 (c)**：对随机噪声性质的约束。给定历史序列的条件期望为零（构成鞅差分序列），说明噪声的估计是无偏的；条件方差有限则限制了单步探索时的波动幅度，防止算法发散。

## 随机梯度下降 (SGD)

SGD 主要用于解决期望风险最小化问题：

$$
\min_{w} \quad J(w) = \mathbb{E}[f(w, X)]
$$

* $w$ 是待优化的参数。
* $X$ 是随机变量，期望是关于 $X$ 的分布计算的。
* 函数 $f(\cdot)$ 输出标量，参数 $w$ 和输入 $X$ 可以是标量或向量。

### 梯度下降演变

**1. 梯度下降 (Gradient Descent, GD)**

$$
w_{k+1} = w_k - \alpha_k \nabla_w \mathbb{E}[f(w_k, X)] = w_k - \alpha_k \mathbb{E}[\nabla_w f(w_k, X)]
$$

**2. 批量梯度下降 (Batch Gradient Descent, BGD)**

通过有限样本的经验均值近似数学期望：

$$
\mathbb{E}[\nabla_w f(w_k, X)] \approx \frac{1}{n} \sum_{i=1}^n \nabla_w f(w_k, x_i)
$$

$$
w_{k+1} = w_k - \alpha_k \frac{1}{n} \sum_{i=1}^n \nabla_w f(w_k, x_i)
$$

**3. 随机梯度下降 (Stochastic Gradient Descent, SGD)**

$$
w_{k+1} = w_k - \alpha_k \nabla_w f(w_k, x_k)
$$

* 与 GD 相比：将真实梯度 $\mathbb{E}[\nabla_w f(w_k, X)]$ 替换为单样本的随机梯度 $\nabla_w f(w_k, x_k)$。
* 与 BGD 相比：相当于将批量大小设定为 $n = 1$。

根据 RM 算法的理论，如果满足海森矩阵正定有界（$0 < c_1 \le \nabla^2_w f(w, X) \le c_2$）、学习率满足 Robbins-Monro 序列条件，且样本序列独立同分布（i.i.d.），则 SGD 同样会以概率 1 收敛于最优解。

### SGD 梯度的相对误差与随机性

引入相对误差 $\delta_k$ 来衡量随机梯度偏离真实梯度的程度：

$$
\delta_k \doteq \frac{|\nabla_w f(w_k, x_k) - \mathbb{E}[\nabla_w f(w_k, X)]|}{|\mathbb{E}[\nabla_w f(w_k, X)]|}
$$

在最优解 $w^*$ 处，满足 $\mathbb{E}[\nabla_w f(w^*, X)] = 0$。将其代入分母，并应用积分中值定理：

$$
\delta_k = \frac{|\nabla_w f(w_k, x_k) - \mathbb{E}[\nabla_w f(w_k, X)]|}{|\mathbb{E}[\nabla_w f(w_k, X)] - \mathbb{E}[\nabla_w f(w^*, X)]|} = \frac{|\nabla_w f(w_k, x_k) - \mathbb{E}[\nabla_w f(w_k, X)]|}{|\mathbb{E}[\nabla_w^2 f(\tilde{w}_k, X)(w_k - w^*)]|}
$$

由于存在强凸性假设 $\nabla_w^2 f \ge c > 0$，对分母放缩可得：

$$
\begin{aligned}
|\mathbb{E}[\nabla_w^2 f(\tilde{w}_k, X)(w_k - w^*)]| &= |\mathbb{E}[\nabla_w^2 f(\tilde{w}_k, X)] \cdot (w_k - w^*)| \\
&= |\mathbb{E}[\nabla_w^2 f(\tilde{w}_k, X)]| \cdot |w_k - w^*| \ge c|w_k - w^*|
\end{aligned}
$$

进而得到相对误差的上限：

$$
\delta_k \leq \frac{|\overbrace{\nabla_w f(w_k, x_k)}^{\text{随机梯度}} - \overbrace{\mathbb{E}[\nabla_w f(w_k, X)]}^{\text{真实梯度}}|}{\underbrace{c|w_k - w^*|}_{\text{距最优解的距离}}}
$$

该不等式严格揭示了 SGD 的一种重要收敛模式：

* 相对误差 $\delta_k$ 与距最优解的距离 $|w_k - w^*|$ 成反比。
* 当 $w_k$ 距离最优解较远时，分母较大，$\delta_k$ 较小。此时 SGD 的更新方向非常接近真实梯度，表现出与 GD 高度相似的下降轨迹。
* 当 $w_k$ 逐渐逼近最优解 $w^*$ 时，分母趋于零，相对误差 $\delta_k$ 会显著增大。这意味着在最优解邻域内，噪声的干扰占据主导，导致算法在收敛末期表现出较强的随机震荡（这也是为何 SGD 需要配合学习率衰减的原因）。

## BGD, MBGD, 和 SGD 采样对比

> 这种随机采样策略与截断（truncated）方法有异曲同工之妙。

假设我们需要最小化 $J(w) = \mathbb{E}[f(w, X)]$，并拥有一组 $X$ 的随机样本 $\{x_i\}_{i=1}^n$。三种算法的迭代公式对比：

$$
w_{k+1} = w_k - \alpha_k \frac{1}{n} \sum_{i=1}^n \nabla_w f(w_k, x_i) \quad \text{(BGD)}
$$

$$
w_{k+1} = w_k - \alpha_k \frac{1}{m} \sum_{j \in \mathcal{I}_k} \nabla_w f(w_k, x_j) \quad \text{(MBGD)}
$$

$$
w_{k+1} = w_k - \alpha_k \nabla_w f(w_k, x_k) \quad \text{(SGD)}
$$

* **BGD**：每次迭代计算完整的 $n$ 个样本。当 $n$ 足够大时，更新方向极其接近真实期望梯度。
* **MBGD**：每次迭代从全局样本中抽取大小为 $m$ 的子集 $\mathcal{I}_k$。该集合是通过 $m$ 次独立同分布（i.i.d.）采样获得的。
* **SGD**：每次迭代仅在时刻 $k$ 随机抽取单一标本 $x_k$。

> **核心差异注意**：即使当 $m=n$ 时，MBGD 与 BGD 也并不等价。因为 MBGD 的 $m$ 次随机采样通常是有放回的（可能抽取到重复的样本），而 BGD 则是严格遍历所有不重复的全局样本数据。
