---
title: "RL学习笔记：策略梯度方法"
publishDate: 2026-02-22 16:30:00
description: "总结强化学习中策略梯度方法的核心思想，涵盖目标函数的定义、对数求导技巧与梯度定理的推导过程，并梳理REINFORCE算法及其参数更新机制。"
tags: ["Reinforcement Learning", "Policy Gradient", "REINFORCE", "学习笔记"]
language: "中文"
---

# 策略梯度方法

在策略梯度方法中，策略被以一种参数函数的形式呈现。

$$
\pi(a|s,\theta)
$$

其中 $\theta \in \mathbb{R}^m$ 是参数向量。

* 该函数可以是一个神经网络，其输入为状态 $s$，输出是采取每个动作的概率，参数为 $\theta$。
* 当状态空间较大时，表格表示法（tabular representation）在存储和泛化方面的效率较低，函数近似能有效解决这一问题。
* 函数表示通常也写作 $\pi(a, s, \theta)$、$\pi_\theta(a|s)$ 或 $\pi_\theta(a, s)$。

### 基本思想

设定一个目标函数（例如 $J(\theta)$）来评估策略的表现。

使用梯度上升的方法来更新参数，寻找最优策略：

$$
\theta_{t+1} = \theta_t + \alpha \nabla_\theta J(\theta_t)
$$

## 目标函数 (Metrics)

### 1. 状态价值的加权平均

$$
\bar{v}_{\pi} = \sum_{s \in \mathcal{S}} d(s) v_{\pi}(s)
$$

* $\bar{v}_{\pi}$ 是一个加权平均值。
* $d(s) \ge 0$ 是状态 $s$ 的权重，可以理解为状态出现的概率分布。
* 期望形式表示为：$\bar{v}_{\pi} = \mathbb{E}[v_{\pi}(S)]$。

其向量形式为：

$$
\bar{v}_{\pi} = d^T v_{\pi}
$$

其中 $v_{\pi} \in \mathbb{R}^{|\mathcal{S}|}$ 且 $d \in \mathbb{R}^{|\mathcal{S}|}$。

在幕式（Episodic）任务中，目标函数可以定义为从初始状态出发的期望回报：

$$
\begin{aligned}
J(\theta) &= \mathbb{E} \left[ \sum_{t=0}^{\infty} \gamma^t R_{t+1} \right] \\
&= \sum_{s \in \mathcal{S}} d_0(s) v_\pi(s)
\end{aligned}
$$

**关于权重 $d(s)$ 的设定：**

* **与策略 $\pi$ 无关：** 在幕式任务中，通常将 $d$ 设定为初始状态分布 $d_0$。例如，认为所有状态同等重要时 $d_0(s) = 1/|\mathcal{S}|$，或者只关心特定初始状态 $s_0$ 时 $d_0(s_0) = 1$。
* **与策略 $\pi$ 有关：** 在持续性（Continuing）任务中，$d$ 依赖于策略 $\pi$，通常选择稳态分布 $d_\pi$。稳态分布满足 $d_{\pi}^T P_{\pi} = d_{\pi}^T$，其中 $P_{\pi}$ 是状态转移矩阵。

### 2. 平均单步奖励 (Average Reward)

$$
\bar{r}_\pi = \sum_{s \in \mathcal{S}} d_\pi(s) r_\pi(s) = \mathbb{E}[r_\pi(S)]
$$

其中状态 $S \sim d_\pi$。状态 $s$ 下的期望即时奖励为：

$$
r_\pi(s) = \sum_{a \in \mathcal{A}} \pi(a|s) r(s, a)
$$

* 权重 $d_\pi$ 是平稳分布。
* $\bar{r}_\pi$ 是单步即时奖励的加权平均值。

平均奖励也可以定义为长期奖励的极限形式：

$$
\begin{aligned}
\lim_{n \to \infty} \frac{1}{n} \mathbb{E} \left[ \sum_{k=1}^{n} R_{t+k} \mid S_t = s_0 \right] &= \sum_{s \in \mathcal{S}} d_\pi(s) r_\pi(s) = \bar{r}_\pi
\end{aligned}
$$

此时初始状态 $s_0$ 的影响在极限下被消除，两种定义方式等价。

**指标对比：**

* 上述指标均依赖于策略 $\pi$，因此本质上都是参数 $\theta$ 的函数。
* 直观来看，$\bar{r}_\pi$ 更加关注即时奖励，而 $\bar{v}_\pi$ 则更关注长期回报。
* 在包含折扣因子 $\gamma$ 的情况下，二者存在数学联系：$\bar{r}_\pi = (1 - \gamma) \bar{v}_\pi$。

## 策略梯度定理 (Gradients of the Metrics)

无论目标函数是 $\bar{v}_{\pi}$ 还是 $\bar{r}_{\pi}$，其梯度都可以统一表示为以下形式（成比例）：

$$
\nabla_{\theta} J(\theta) \propto \sum_{s \in \mathcal{S}} \eta(s) \sum_{a \in \mathcal{A}} \nabla_{\theta} \pi(a|s, \theta) q_{\pi}(s, a)
$$

其中 $\eta$ 是状态的分布权重。

### 梯度的推导与期望转化

**1. 核心目标：** 将复杂的“状态-动作双重求和”形式，转化为允许通过数据采样来近似计算的“数学期望”形式。

**2. 对数求导技巧 (Log-Derivative Trick)：** 根据微积分法则，对策略函数取对数并求梯度：

$$
\nabla_{\theta} \ln \pi(a|s, \theta) = \frac{1}{\pi(a|s, \theta)} \nabla_{\theta} \pi(a|s, \theta)
$$

移项后得到：

$$
\nabla_{\theta} \pi(a|s, \theta) = \pi(a|s, \theta) \nabla_{\theta} \ln \pi(a|s, \theta)
$$

这一步“无中生有”地构造出了概率项 $\pi(a|s, \theta)$，是转化为期望定义的先决条件。

**3. 公式代入：** 将上述结果代入梯度公式：

$$
\nabla_{\theta} J(\theta) \propto \sum_{s \in \mathcal{S}} \eta(s) \sum_{a \in \mathcal{A}} \pi(a|s, \theta) \nabla_{\theta} \ln \pi(a|s, \theta) q_{\pi}(s, a)
$$

**4. 转化为数学期望：** 上式包含两层概率加权求和（外层基于状态分布 $\eta(s)$，内层基于动作概率 $\pi$），等价于对随机变量 $S$ 和 $A$ 的期望：

$$
\nabla_{\theta} J(\theta) = \mathbb{E}[\nabla_{\theta} \ln \pi(A|S, \theta) q_{\pi}(S, A)]
$$

**5. 实际意义：** 完成转化后，算法无需遍历所有状态和动作。智能体只需在环境中依据当前策略采样，收集到的轨迹数据自然符合该期望分布，从而可以通过单步采样来近似计算梯度。

如果利用采样来近似梯度，单步更新方向为：

$$
\nabla_{\theta} J \approx \nabla_{\theta} \ln \pi(a|s, \theta) q_{\pi}(s, a)
$$

## 策略函数的参数化 (Softmax)

为了保证概率性质 $\pi(a|s, \theta) > 0$ 且概率和为 1，通常使用 Softmax 函数将实数偏好映射为概率。

对于向量 $x = [x_1, \dots, x_n]^T$：

$$
z_i = \frac{e^{x_i}}{\sum_{j=1}^n e^{x_j}}
$$

其中 $z_i \in (0, 1)$ 且 $\sum_{i=1}^n z_i = 1$。

应用到策略函数中：

$$
\pi(a|s, \theta) = \frac{e^{h(s,a,\theta)}}{\sum_{a' \in \mathcal{A}} e^{h(s,a',\theta)}}
$$

其中 $h(s, a, \theta)$ 是动作偏好函数，可由神经网络参数化。



## REINFORCE 与策略优化算法

利用真实梯度最大化目标函数：

$$
\begin{aligned}
\theta_{t+1} &= \theta_t + \alpha \nabla_{\theta} J(\theta) \\
&= \theta_t + \alpha \mathbb{E}[\nabla_{\theta} \ln \pi(A|S, \theta_t) q_{\pi}(S, A)]
\end{aligned}
$$

在实际应用中，使用随机梯度进行更新：

$$
\theta_{t+1} = \theta_t + \alpha \nabla_{\theta} \ln \pi(a_t|s_t, \theta_t) q_{\pi}(s_t, a_t)
$$

由于真实的动作价值函数 $q_\pi$ 是未知的，需要对其进行近似估算：

* **REINFORCE 算法：** 使用蒙特卡洛方法，将完整的轨迹回报 $G_t$ 作为 $q_\pi(s_t, a_t)$ 的无偏估计进行梯度更新。
* **Actor-Critic 算法：** 结合时间差分（TD）算法，训练一个价值函数网络来近似估算 $q_\pi(s_t, a_t)$。

结合对数求导公式的反向展开 $\nabla_{\theta} \ln \pi = \frac{\nabla_{\theta} \pi}{\pi}$，参数更新规则可以直观地改写为：

$$
\begin{aligned}
\theta_{t+1} &= \theta_t + \alpha \nabla_{\theta} \ln \pi(a_t | s_t, \theta_t) q_t(s_t, a_t) \\
&= \theta_t + \alpha \underbrace{\left( \frac{q_t(s_t, a_t)}{\pi(a_t | s_t, \theta_t)} \right)}_{\beta_t} \nabla_{\theta} \pi(a_t | s_t, \theta_t)
\end{aligned}
$$

即：

$$
\theta_{t+1} = \theta_t + \alpha \beta_t \nabla_{\theta} \pi(a_t | s_t, \theta_t)
$$

这里的系数 $\beta_t$ 能够很好地平衡探索与利用：它与回报 $q_t$ 成正比（鼓励高回报动作），与动作概率 $\pi$ 成反比（赋予罕见动作更大的更新步长，从而鼓励探索）。
