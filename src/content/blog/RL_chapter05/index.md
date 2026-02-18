---
title: "RL学习笔记：蒙特卡洛方法"
publishDate: 2026-02-18 20:00:00
description: "深入解析强化学习中的蒙特卡洛方法（Monte Carlo Methods），涵盖MC Basic与Exploring Starts采样机制。探讨了广义策略迭代（GPI）框架，并详细推导epsilon-Greedy策略如何平衡探索与利用，实现无模型场景下的策略优化。"
tags: ["Reinforcement Learning", "Monte Carlo Methods", "GPI", "Epsilon-Greedy", "学习笔记"]
language: "中文"
---

# 蒙特卡洛方法 (Monte Carlo Methods)

## 最简单的 MC-based RL 算法：MC Basic

* **核心流程**：从某个 $(s, a)$ 出发，遵循一个策略 $\pi_k$，产生一个 episode。

* **回报计算**：
  
  * 这个 episode 得到的 (discounted) return 记作 $g(s,a)$。
  * $g(s,a)$ 是 $G_t$ 的一个采样，即 $q_{\pi_k}(s, a) = \mathbb{E}[G_t | S_t = s, A_t = a]$ 的采样。

* **大数定律估计**：如果有许多 episode 产生了一组 $\{g^{(j)}(s, a)\}$，则：

$$
q_{\pi_k}(s, a) = \mathbb{E}[G_t | S_t = s, A_t = a] \approx \frac{1}{N} \sum_{i=1}^{N} g^{(i)}(s, a)
$$



* **核心思想**：当没有模型 (Model) 的时候必须要有数据，没有数据的时候必须要有模式，即 Experience。

* **定位**：MC Basic 是 Policy Iteration 算法的一个变种，去除了基于模型的部分。

* **缺点**：MC Basic 过于低效，实际中很少直接使用。

### Episode 长度的考量

* **过短**：只有足够近的 state 才能找到最优策略。
* **逐渐加长**：随着长度增加，慢慢可以找到最优策略。
* **结论**：Episode 必须足够长，但不需要无限长。



## MC Exploring Starts

### 采样序列示例

$$
s_1 \xrightarrow{a_2} s_2 \xrightarrow{a_4} s_1 \xrightarrow{a_2} s_2 \xrightarrow{a_3} s_5 \xrightarrow{a_1} \dots
$$

### Visit 的定义

在一个 episode 中，每一个 state-action pair (状态-动作对) 出现一次，称其为一个 **visit**。
基于上述序列的拆解示例：

$$
\begin{aligned}
    s_1 \xrightarrow{a_2} s_2 \xrightarrow{a_4} s_1 \xrightarrow{a_2} s_2 \xrightarrow{a_3} s_5 \xrightarrow{a_1} \dots & \quad [\text{原始 episode}] \\
    s_2 \xrightarrow{a_4} s_1 \xrightarrow{a_2} s_2 \xrightarrow{a_3} s_5 \xrightarrow{a_1} \dots & \quad [\text{从 } (s_2, a_4) \text{ 开始的 episode}] \\
    s_1 \xrightarrow{a_2} s_2 \xrightarrow{a_3} s_5 \xrightarrow{a_1} \dots & \quad [\text{从 } (s_1, a_2) \text{ 开始的 episode}] \\
    s_2 \xrightarrow{a_3} s_5 \xrightarrow{a_1} \dots & \quad [\text{从 } (s_2, a_3) \text{ 开始的 episode}] \\
    s_5 \xrightarrow{a_1} \dots & \quad [\text{从 } (s_5, a_1) \text{ 开始的 episode}]
\end{aligned}
$$

### 数据效率 (Data Efficiency) 方法

* **First-visit**：每个 state-action pair 只用其**第一次**出现的时候来估计。
* **Every-visit**：每个 state-action pair 只要出现一次就重新估计。



## Generalized Policy Iteration (GPI)

关于更新时机的考量：

* 等到所有的 episode 都收集完成之后，对其求平均再估计。
* 或者，得到一个 episode 之后就开始估计，每一次都重新估计。

**GPI 的核心概念**：

* GPI 不是一种特殊的算法，而是一个统称。
* 它体现了在 **Policy Evaluation** (策略评估) 和 **Policy Improvement** (策略提升) 之间不断切换的过程。
* 许多 Model-based 和 Model-free 的算法都在其框架之中。



## MC-$\epsilon$-Greedy

### Soft Policies

* 定义：一个 policy 对每一个 action 都有非零的概率选择，叫做 soft policy。
* 作用：不再需要通过“从每一个 state-action 出发产生大量 episode”来确保覆盖率，从而去除了 Exploring Starts 的强假设。

### $\epsilon$-Greedy 策略

公式如下：

$$
\pi(a|s) = 
\begin{cases} 
    1 - \dfrac{\epsilon}{|\mathcal{A}(s)|}(|\mathcal{A}(s)| - 1), & \text{对于贪婪动作 (greedy action), 即 } a = a^* \\[15pt]
    \dfrac{\epsilon}{|\mathcal{A}(s)|}, & \text{对于其他 } |\mathcal{A}(s)| - 1 \text{ 个动作}
\end{cases}
$$

其中 $\epsilon \in [0, 1]$，$|\mathcal{A}(s)|$ 是该状态下可选动作的总数。

**Exploitation 与 Exploration 的平衡**：

* 当 $\epsilon = 0$：变成 Greedy (完全利用)。
* 当 $\epsilon = 1$：变成均匀分布 (完全探索)。

### 策略提升 (Policy Improvement)

目标是最大化动作价值函数：

$$
\pi_{k+1}(s) = \arg \max_{\pi \in \Pi_{\varepsilon}} \sum_{a} \pi(a|s) q_{\pi_k}(s, a)
$$

由此导出的更新规则：

$$
\pi_{k+1}(a|s) = \begin{cases} 
1 - \frac{|\mathcal{A}(s)|-1}{|\mathcal{A}(s)|}\varepsilon, & a = a_k^* \\ 
\frac{1}{|\mathcal{A}(s)|}\varepsilon, & a \neq a_k^* \end{cases}
$$

**结论**：通过引入 $\epsilon$-Greedy，不再需要 exploring starts 条件 (即允许从所有的状态出发的假设)。
