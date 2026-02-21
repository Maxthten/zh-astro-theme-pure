---
title: "RL学习笔记：值函数近似"
publishDate: 2026-02-21 21:15:00
description: "总结强化学习中值函数近似的核心概念，涵盖线性与非线性近似、状态分布假设与梯度优化方法，并梳理DQN及经验回放机制。"
tags: ["Reinforcement Learning", "Value Function Approximation", "DQN", "学习笔记"]
language: "中文"
---

# 值函数近似 (Value Function Approximation)

## 线性函数形式

$$
\hat{v}(s, w) = as + b = \underbrace{[s, 1]}_{\phi^T(s)} \underbrace{\begin{bmatrix} a \\ b \end{bmatrix}}_{w} = \phi^T(s)w
$$

其中：

- $w$ 是参数向量。
- $\phi(s)$ 是状态 $s$ 的特征向量。
- $\hat{v}(s, w)$ 关于 $w$ 是线性的。

## 非线性函数形式

$$
\hat{v}(s, w) = as^2 + bs + c = \underbrace{[s^2, s, 1]}_{\phi^T(s)} \underbrace{\begin{bmatrix} a \\ b \\ c \end{bmatrix}}_{w} = \phi^T(s)w
$$

在这种情况下：

- $w$ 和 $\phi(s)$ 的维度增加了，数值拟合可能会更准确。
- 虽然 $\hat{v}(s, w)$ 关于状态 $s$ 是非线性的，但它关于参数 $w$ 依然是线性的。非线性特征被包含在 $\phi(s)$ 的映射中。

## 状态价值估计 (State Value Estimation)

目标函数（Objective Function）：

$$
J(w) = \mathbb{E}[(v_\pi(S) - \hat{v}(S, w))^2]
$$

- 核心目标是找到最优的参数 $w$ 来最小化该目标函数。
- $S$ 是一个随机变量，其概率分布主要有以下两种考量：

### 平均分布 (Uniform Distribution)

$$
J(w) = \frac{1}{|\mathcal{S}|} \sum_{s \in \mathcal{S}} (v_\pi(s) - \hat{v}(s, w))^2
$$

- 平均分布平等对待所有状态。但实际强化学习中，某些状态的访问频率更高且更关键，因此这种分布往往不适用。

### 稳态分布 (Stationary Distribution)

稳态分布描述了马尔可夫过程的长期行为（Long-run behavior）。其中 $\{d_{\pi}(s)\}_{s \in \mathcal{S}}$ 代表了状态分布的集合，满足 $d_{\pi}(s) \geq 0$ 且 $\sum_{s \in \mathcal{S}} d_{\pi}(s) = 1$。

$$
J(w) = \sum_{s \in \mathcal{S}} d_{\pi}(s)(v_{\pi}(s) - \hat{v}(s, w))^2
$$

- $d_{\pi}(s)$ 代表了在策略 $\pi$ 下处于特定状态的概率平稳值。使用稳态分布可以使得在常访问状态上的拟合误差更小。
- 稳态分布满足公式：

$$
d_{\pi}^T = d_{\pi}^T P_{\pi}
$$

其中 $P_{\pi}$ 为贝尔曼方程中的状态转移矩阵。

## 优化方法

使用梯度下降法更新参数：

$$
w_{k+1} = w_k - \alpha_k \nabla_w J(w_k)
$$

真实梯度的推导过程如下：

$$
\begin{aligned}
\nabla_w J(w) &= \nabla_w \mathbb{E}[(v_\pi(S) - \hat{v}(S, w))^2] \\
&= \mathbb{E}[\nabla_w (v_\pi(S) - \hat{v}(S, w))^2] \\
&= 2\mathbb{E}[(v_\pi(S) - \hat{v}(S, w))(-\nabla_w \hat{v}(S, w))] \\
&= -2\mathbb{E}[(v_\pi(S) - \hat{v}(S, w))\nabla_w \hat{v}(S, w)]
\end{aligned}
$$

在实际应用中，通常采用随机梯度下降（SGD）：

$$
w_{t+1} = w_t + \alpha_t (v_\pi(s_t) - \hat{v}(s_t, w_t)) \nabla_w \hat{v}(s_t, w_t)
$$

其中 $s_t$ 是 $S$ 的一个采样。为了表达简洁，常数 $2$ 被吸收到学习率 $\alpha_t$ 中。由于真实的 $v_{\pi}(s_t)$ 未知，我们需要用估计值来替代它：

- **基于蒙特卡洛 (Monte Carlo)**：使用一个回合中的折扣回报 $g_t$ 来近似 $v_{\pi}(s_t)$。

$$
w_{t+1} = w_t + \alpha_t (g_t - \hat{v}(s_t, w_t)) \nabla_w \hat{v}(s_t, w_t)
$$

- **基于时序差分 (Temporal Difference, TD)**：目标值 $r_{t+1}+\gamma\hat{v}(s_{t+1},w_t)$ 被视为 $v_{\pi}(s_t)$ 的近似。

$$
w_{t+1} = w_t + \alpha_t [r_{t+1} + \gamma \hat{v}(s_{t+1}, w_t) - \hat{v}(s_t, w_t)] \nabla_w \hat{v}(s_t, w_t)
$$

## TD-Linear 算法

在 $\hat{v}(s, w) = \phi^T(s)w$ 的线性情况下，梯度为：

$$
\nabla_w \hat{v}(s, w) = \phi(s)
$$

将梯度代入 TD 算法：

$$
w_{t+1} = w_t + \alpha_t [r_{t+1} + \gamma \phi^T(s_{t+1}) w_t - \phi^T(s_t) w_t] \phi(s_t)
$$

这就是带线性函数逼近的 TD 学习算法，简称为 **TD-Linear**。

### 线性近似的求导解析

在强化学习的线性近似中，$\hat{v}(s, w)$ 是一个标量（预测的状态价值），而 $w$ 是一个向量（权重参数）。

1. **拆解线性表达式**
   对于列向量 $\phi(s) = [\phi_1, \dots, \phi_n]^T$ 和 $w = [w_1, \dots, w_n]^T$，内积为：
   
   $$
   \hat{v}(s, w) = \sum_{i=1}^{n} \phi_i w_i
   $$

2. **对向量求导**
   $\nabla_w \hat{v}(s, w)$ 的本质是对标量函数按向量 $w$ 的每一个分量求偏导：
   
   $$
   \frac{\partial}{\partial w_i} (\phi_1 w_1 + \dots + \phi_n w_n) = \phi_i
   $$
   
   拼合后即得到 $\nabla_w \hat{v}(s, w) = \phi(s)$。

### 表格表示 (Tabular Representation)

表格法是线性函数逼近的一个特例。
设定状态 $s$ 的特征向量为独热编码（One-hot）向量：

$$
\phi(s) = e_s \in \mathbb{R}^{|\mathcal{S}|}
$$

此时：

$$
\hat{v}(s, w) = e_s^T w = w(s)
$$

即 $w(s)$ 提取了向量 $w$ 中对应状态 $s$ 的第 $s$ 个分量。

## 动作价值函数近似

### Sarsa with Function Approximation

$$
w_{t+1} = w_t + \alpha_t \left[ r_{t+1} + \gamma \hat{q}(s_{t+1}, a_{t+1}, w_t) - \hat{q}(s_t, a_t, w_t) \right] \nabla_w \hat{q}(s_t, a_t, w_t)
$$

### Q-learning with Function Approximation

$$
w_{t+1} = w_t + \alpha_t \left[ r_{t+1} + \gamma \max_{a \in \mathcal{A}(s_{t+1})} \hat{q}(s_{t+1}, a, w_t) - \hat{q}(s_t, a_t, w_t) \right] \nabla_w \hat{q}(s_t, a_t, w_t)
$$

## 深度 Q 网络 (Deep Q-Network, DQN)

DQN 使用神经网络来逼近非线性的 Q 函数。

**损失函数 (Loss Function)**：

$$
J(w) = \mathbb{E} \left[ \left( R + \gamma \max_{a \in \mathcal{A}(S')} \hat{q}(S', a, w) - \hat{q}(S, A, w) \right)^2 \right]
$$

这实际上是在最小化贝尔曼最优误差 (Bellman Optimality Error)。定义目标值 $y$ 为：

$$
y \doteq R + \gamma \max_{a \in \mathcal{A}(S')} \hat{q}(S', a, w)
$$

为了保证训练的稳定性，防止目标值随着网络更新不断移动，DQN 引入了双网络架构：

- **主网络 (Main Network)**：$\hat{q}(S, A, w)$，负责当前动作的评估与参数实时更新。
- **目标网络 (Target Network)**：$\hat{q}(S', A, w_T)$，提供稳定的目标值 $y$。

引入目标网络后的损失函数变为：

$$
J(w) = \mathbb{E} \left[ \left( R + \gamma \max_{a \in \mathcal{A}(S')} \hat{q}(S', a, w_T) - \hat{q}(S, A, w) \right)^2 \right]
$$

在运算时，假定 $w_T$ 是常数（即不参与梯度计算），梯度下降仅更新 $w$：

$$
\nabla_w J(w) = -2\mathbb{E} \left[ \left( R + \gamma \max_{a \in \mathcal{A}(S')} \hat{q}(S', a, w_T) - \hat{q}(S, A, w) \right) \nabla_w \hat{q}(S, A, w) \right]
$$

*注：主网络的参数 $w$ 每隔一段时间会复制给目标网络 $w_T$。*

### 经验回放 (Experience Replay)

- **动机**：强化学习中收集的连续数据存在强相关性，直接用于训练容易导致网络不稳定。
- **机制**：将智能体与环境交互产生的数据以 $(s, a, r, s')$ 的元组形式存入一个回放缓冲区（Replay Buffer）$\mathcal{B}$。
- **采样**：在训练时，从缓冲区中抽取一批随机采样（Mini-batch）。这种抽取过程通常采用均匀分布（Uniform Distribution），从而打破数据之间的时间相关性，并显著提高数据的利用率。
