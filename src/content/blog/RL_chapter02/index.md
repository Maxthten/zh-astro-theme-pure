---
title: "RL学习笔记：贝尔曼公式"
publishDate: 2026-02-16 20:00:00
description: "详细梳理了State Value与Action Value的定义，推导了贝尔曼期望方程（Bellman Expectation Equation）的通用形式及其矩阵表达。"
tags: ["Reinforcement Learning", "Bellman Equation", "学习笔记", "数学基础"]
language: "中文"
---

# 贝尔曼公式 (Bellman Equation)

## 1. 基本定义

强化学习的交互过程可以描述为：

$$
S_{t} \xrightarrow{A_{t}} R_{t+1}, S_{t+1}

$$

### 核心变量

- $t, t+1$: 离散时间点。
- $S_{t}$: 时间 $t$ 时的状态 (State)。
- $A_{t}$: 在 $S_{t}$ 状态下采取的动作 (Action)。
- $R_{t+1}$: 采取 $A_{t}$ 后得到的即时奖励 (Reward)。
- $S_{t+1}$: 采取 $A_{t}$ 后转移到的新状态 (Next State)。

**注意**：$S_{t}, A_{t}, R_{t+1}$ 均为 **随机变量 (Random Variables)**。这意味着交互的每一步都是由概率分布 (Probability Distribution) 决定的，因此我们可以对它们求期望。

### 轨迹 (Trajectory)与回报 (Return)

交互过程形成的时间序列轨迹如下：

$$
S_{t} \xrightarrow{A_{t}} R_{t+1}, S_{t+1} \xrightarrow{A_{t+1}} R_{t+2}, S_{t+2} \xrightarrow{A_{t+2}} R_{t+3}, \dots

$$

**折扣回报 (Discounted Return)** 定义为从时间 $t$ 开始的累积折扣奖励：



$$
G_{t} = R_{t+1} + \gamma R_{t+2} + \gamma^{2}R_{t+3} + \dots = \sum_{k=0}^{\infty} \gamma^k R_{t+k+1}
$$

其中$ \gamma \in [0, 1]$ 为折扣因子。

---

## 2. 状态价值 (State Value)

### 定义

$v_{\pi}(s)$ 被称为状态价值函数 (State-Value Function)，简称 State Value。它是回报 $G_t$ 的数学期望：

$$
v_{\pi}(s) = \mathbb{E}[G_{t} \mid S_{t}=s]
$$

- 它是状态 $s$ 的函数。
- 它的值取决于当前的策略 $\pi$。
- 它代表了处于该状态的“价值”。价值越高，意味着在该策略下从该状态出发的前景越好。

### 核心区别

> **Return ($G_t$) vs State Value ($v_{\pi}(s)$)**
> 
> * **Return** 是基于**单次**轨迹的现实累积收益，是一个随机变量。
> * **State Value** 是基于**所有可能**轨迹（在特定策略 $\pi$ 下）对 Return 求得的数学期望（统计均值）。
> * 只有当策略和环境完全确定（只有唯一轨迹）时，二者在数值上才等价。

---

## 3. 贝尔曼公式推导

贝尔曼公式描述了当前状态价值与未来状态价值之间的递归关系：

$$
\begin{aligned}
v_{\pi}(s) &= \mathbb{E}[R_{t+1} + \gamma G_{t+1} \mid S_{t}=s] \\
&= \underbrace{\mathbb{E}[R_{t+1} \mid S_{t}=s]}_{\text{即时奖励的期望}} + \gamma \underbrace{\mathbb{E}[G_{t+1} \mid S_{t}=s]}_{\text{未来奖励的期望}}
\end{aligned}
$$

展开后的通用形式：



$$
v_{\pi}(s) = \sum_{a \in \mathcal{A}}\pi(a|s) \left[ \sum_{r \in \mathcal{R}}p(r|s,a)r + \gamma \sum_{s^{\prime} \in \mathcal{S}}p(s^{\prime}|s,a)v_{\pi}(s^{\prime}) \right], \quad \text{for all } s \in \mathcal{S}
$$

### 第一部分：即时奖励的期望 (Mean of Immediate Rewards)

$$
\begin{aligned}
\mathbb{E}[R_{t+1}|S_t = s] &= \sum_{a \in \mathcal{A}} \pi(a|s) \mathbb{E}[R_{t+1}|S_t = s, A_t = a] \\
&= \sum_{a \in \mathcal{A}} \pi(a|s) \sum_{r \in \mathcal{R}} p(r|s, a)r
\end{aligned}
$$

此处应用了概率论中的 **全期望公式 (Law of Total Expectation)**：

* $\pi(a|s)$: **权重**（采取该动作的概率）。
* $\mathbb{E}[R|s, a]$: **条件期望**（在该动作下的平均奖励）。
* $\sum$: **加权求和**。

**理解**：如果是确定性策略（Deterministic Policy），求和号中仅有一项非零；但在通用的随机策略下，必须遍历所有可能的动作进行加权。

### 第二部分：未来奖励的期望 (Mean of Future Rewards)

$$
\begin{aligned}
\mathbb{E}[G_{t+1}|S_t = s] &= \sum_{s' \in \mathcal{S}} \mathbb{E}[G_{t+1}|S_t = s, S_{t+1} = s'] p(s'|s) \\
&= \sum_{s' \in \mathcal{S}} \mathbb{E}[G_{t+1}|S_{t+1} = s'] p(s'|s) \quad \text{(马尔可夫性质)} \\
&= \sum_{s' \in \mathcal{S}} v_\pi(s') p(s'|s) \\
&= \sum_{s' \in \mathcal{S}} v_\pi(s') \sum_{a \in \mathcal{A}} p(s'|s, a)\pi(a|s)
\end{aligned}
$$

**本质**：计算“从当前状态看，未来的平均价值是多少”。
该推导将未来回报的期望拆解为三个核心要素的乘积之和：

1. **策略** $\pi(a|s)$：我们怎么做选择。
2. **环境动力学** $p(s'|s,a)$：环境如何转移状态。
3. **下一状态的价值** $v_\pi(s')$：未来有多好。

---

## 4. 矩阵与向量形式

为了便于计算，我们可以定义以下两个辅助项：

1. **平均即时奖励向量** $r_{\pi}$：
   
   

$$
r_{\pi}(s) \doteq \sum_{a \in \mathcal{A}} \pi(a|s) \sum_{r \in \mathcal{R}} p(r|s, a)r
$$

   *含义：将动作概率与动作产生的奖励融合，计算出当前状态 $s$ 的综合即时奖励期望。*

2. **状态转移矩阵** $P_{\pi}$：
   
   $$
   p_{\pi}(s'|s) \doteq \sum_{a \in \mathcal{A}} \pi(a|s)p(s'|s, a)
   $$
   
   *含义：忽略具体的动作选择，直接描述在当前策略 $\pi$ 下，从状态 $s$ 流向 $s'$ 的统计规律。*

由此得到贝尔曼公式的矩阵形式 (Bellman Expectation Equation)：

$$
v_{\pi} = r_{\pi} + \gamma P_{\pi}v_{\pi}
$$

### 矩阵展开示例

假设有 4 个状态：

$$
\underbrace{
\begin{bmatrix}
v_\pi(s_1) \\
v_\pi(s_2) \\
v_\pi(s_3) \\
v_\pi(s_4)
\end{bmatrix}
}_{v_\pi}
=
\underbrace{
\begin{bmatrix}
r_\pi(s_1) \\
r_\pi(s_2) \\
r_\pi(s_3) \\
r_\pi(s_4)
\end{bmatrix}
}_{r_\pi}
+ \gamma
\underbrace{
\begin{bmatrix}
p_\pi(s_1|s_1) & p_\pi(s_2|s_1) & p_\pi(s_3|s_1) & p_\pi(s_4|s_1) \\
p_\pi(s_1|s_2) & p_\pi(s_2|s_2) & p_\pi(s_3|s_2) & p_\pi(s_4|s_2) \\
p_\pi(s_1|s_3) & p_\pi(s_2|s_3) & p_\pi(s_3|s_3) & p_\pi(s_4|s_3) \\
p_\pi(s_1|s_4) & p_\pi(s_2|s_4) & p_\pi(s_3|s_4) & p_\pi(s_4|s_4)
\end{bmatrix}
}_{P_\pi}
\underbrace{
\begin{bmatrix}
v_\pi(s_1) \\
v_\pi(s_2) \\
v_\pi(s_3) \\
v_\pi(s_4)
\end{bmatrix}
}_{v_\pi}
$$

### 求解方法

**1. 封闭解 (Closed Form Solution)**
可以直接通过矩阵求逆求解：

$$
v_\pi = (I - \gamma P_\pi)^{-1} r_\pi
$$



*缺点：当状态空间巨大时，求逆运算量过大，不可行。*

**2. 迭代法 (Iterative Solution)**
即策略评估 (Policy Evaluation) 的基础：



$$
v_{k+1} = r_\pi + \gamma P_\pi v_k, \quad k = 0, 1, 2, \dots
$$

*结论：当 $k \to \infty$ 时，序列收敛于真实价值 $v_{\pi}$。*

---

## 5. 动作价值 (Action Value)

### 定义与对比

* **State Value ($v_{\pi}$)**: Agent 从一个 State 出发得到的平均 Return。
* **Action Value ($q_{\pi}$)**: Agent 从一个 State 出发，**先采取特定 Action**，随后遵循策略 $\pi$ 得到的平均 Return。

定义公式：



$$
q_\pi(s, a) \doteq \mathbb{E}[G_t \mid S_t = s, A_t = a]
$$



它依赖于两个要素：当前的状态-动作对 (State-Action Pair) 和后续遵循的策略 $\pi$。

### 二者关系

**1. State Value 是 Action Value 的期望**



$$
v_\pi(s) = \sum_{a \in \mathcal{A}} \pi(a|s) q_\pi(s, a)
$$



**2. Action Value 的展开**
将 $q_\pi(s, a)$ 展开为即时奖励与下一状态价值的和：



$$
q_{\pi}(s,a) = \sum_{r \in \mathcal{R}} p(r|s,a)r + \gamma \sum_{s' \in \mathcal{S}} p(s'|s,a)v_{\pi}(s')
$$



结合上述两点，再次印证了贝尔曼公式的递归结构：



$$
v_{\pi}(s) = \sum_{a} \pi(a|s) \underbrace{\left[ \sum_{r} p(r|s,a)r + \gamma \sum_{s'} p(s'|s,a)v_{\pi}(s') \right]}_{q_{\pi}(s,a)}
$$



**总结**：只要知道所有的 State Value，就可以求出所有的 Action Value，反之亦然。
