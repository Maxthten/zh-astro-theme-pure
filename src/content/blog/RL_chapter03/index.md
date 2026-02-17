---
title: "RL学习笔记：贝尔曼最优公式"
publishDate: 2026-02-17 17:00:00
description: "推导了贝尔曼最优方程（Bellman Optimality Equation）及其不动点性质，解析了Value Iteration的收敛原理（Contraction Mapping），并讨论了系统模型与奖励函数对最优策略的决定作用。"
tags: ["Reinforcement Learning", "Bellman Optimality", "Value Iteration", "学习笔记"]
language: "中文"
---



# 贝尔曼最优公式 (Bellman Optimality Equation)

## 定义

对于所有状态 $s \in \mathcal{S}$，如果策略 $\pi^*$ 的状态价值函数 $v_{\pi^*}(s)$ 均不小于任何其他策略 $\pi$ 的状态价值函数 $v_{\pi}(s)$，即：

$$
v_{\pi^*}(s) \geq v_{\pi}(s), \quad \forall s \in \mathcal{S}, \forall \pi
$$

则称策略 $\pi^*$ 为**最优策略**。最优策略对应的状态价值称为**最优状态价值函数**，记为 $v^*(s)$。

## 最优公式推导

最优状态价值函数 $v^*(s)$ 满足贝尔曼最优公式。其核心思想是：最优价值等于在当前状态下执行**最优动作**所获得的期望回报。

### 标量形式

$$
\begin{aligned}
v^*(s) &= \max_{a \in \mathcal{A}} q^*(s, a) \\
&= \max_{a \in \mathcal{A}} \left( \sum_{r \in \mathcal{R}} p(r|s,a)r + \gamma \sum_{s' \in \mathcal{S}} p(s'|s,a)v^*(s') \right)
\end{aligned}
$$

若将其写成对策略 $\pi$ 的最大化形式，则有：

$$
v^*(s) = \max_{\pi} \sum_{a \in \mathcal{A}} \pi(a|s) q^*(s, a)
$$

由于加权平均值不可能超过最大值，即：

$$
\sum_{a \in \mathcal{A}} \pi(a|s) q^*(s, a) \leq \max_{a \in \mathcal{A}} q^*(s, a)
$$

等号成立的条件是策略 $\pi$ 将概率完全分配给使 $q^*(s,a)$ 最大的动作。这意味着最优策略 $\pi^*$ 是确定性的（Deterministic）：

$$
\pi^*(a|s) = 
\begin{cases} 
1, & a = \arg\max_{a' \in \mathcal{A}} q^*(s, a') \\ 
0, & \text{其他}
\end{cases}
$$

### 向量形式

我们将求解 $v^*$ 的过程视为算子操作。定义最优贝尔曼算子 $\mathcal{T}^*$，则贝尔曼最优公式是不动点方程：

$$
v^* = \mathcal{T}^*(v^*)
$$

具体展开为：

$$
v^* = \max_{\pi} (r_{\pi} + \gamma P_{\pi} v^*)
$$

其中：

* $r_{\pi}$ 是策略 $\pi$ 下的平均即时奖励向量，$[r_{\pi}]_s = \sum_{a} \pi(a|s) \sum_{r} p(r|s,a)r$
* $P_{\pi}$ 是策略 $\pi$ 下的状态转移矩阵，$[P_{\pi}]_{s,s'} = \sum_{a} \pi(a|s) p(s'|s,a)$

## 压缩映射与不动点 (Contraction Mapping)

贝尔曼最优算子 $\mathcal{T}^*$ 在 $\gamma \in [0, 1)$ 时满足**压缩映射定理 (Contraction Mapping Theorem)**。这意味着：

1. **存在性**：存在唯一的不动点 $v^*$ 满足 $v^* = \mathcal{T}^*(v^*)$。
2. **收敛性**：对于任意初始价值 $v_0$，迭代序列 $v_{k+1} = \mathcal{T}^*(v_k)$ 必然收敛至 $v^*$。
   * 即 $\lim_{k \to \infty} v_k = v^*$。
   * 收敛速度呈几何级数（指数级收敛），受折扣因子 $\gamma$ 控制。

### 值迭代 (Value Iteration) 的本质

值迭代算法利用了上述不动点性质，迭代公式为：

$$
v_{k+1} = \max_{\pi} (r_{\pi} + \gamma P_{\pi} v_k)
$$

这一步操作实际上包含了两个隐式过程：

1. **策略截断 (Policy Improvement)**：
   基于当前的价值估计 $v_k$，寻找一个贪心策略（Greedy Policy），即选择当前看来 $q$ 值最大的动作。
   
   $$
   \pi_{greedy} = \arg\max_{\pi} (r_{\pi} + \gamma P_{\pi} v_k)
   $$
   
   

2. **价值评估 (Policy Evaluation)**：
   假设执行上述贪心动作，计算其一步期望回报作为新的价值估计 $v_{k+1}$。

**总结**：值迭代就是每一轮都“抢”当前最好的动作，计算其价值，并在下一轮基于新价值继续“抢”最好的动作。

## 最优策略的决定因素

最优策略 $\pi^*$ 由以下公式决定：

$$
\pi^*(s) = \arg\max_{a} \left( \sum_{r} p(r|s,a)r + \gamma \sum_{s'} p(s'|s,a)v^*(s') \right)
$$

### 关键影响因子

1. **系统动力学 (System Dynamics)**：$p(s'|s, a)$ 和 $p(r|s, a)$。这是环境的物理法则，通常不可变。
2. **折扣因子 (Discount Factor)** $\gamma$：
   * $\gamma \to 0$：Agent 变得“近视”，只关注即时奖励 (Immediate Reward)。
   * $\gamma \to 1$：Agent 变得“远视”，重视长期累积回报。
3. **奖励函数 (Reward Function)** $r$：
   * 奖励的**相对数值**比绝对数值更重要。

### 奖励函数的仿射变换 (Affine Transformation)

如果对奖励函数进行线性变换：

$$
r'(s, a, s') = \alpha \cdot r(s, a, s') + \beta
$$

其中 $\alpha > 0$ 且 $\beta$ 为常数。

* **对价值函数的影响**：新的价值函数 $v'$ 与原价值函数 $v$ 呈线性关系。

$$
v'(s) = \alpha v(s) + \frac{\beta}{1-\gamma}
$$



* **对策略的影响**：最优策略**保持不变**。

$$
\arg\max_a q'(s,a) = \arg\max_a \left( \alpha q(s,a) + \frac{\beta}{1-\gamma} \right) = \arg\max_a q(s,a)
$$



这表明，只要保持奖励之间的偏序关系和相对比例，具体的数值大小不会改变最优行为模式。
