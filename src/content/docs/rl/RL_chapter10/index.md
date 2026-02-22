---

title: "RL学习笔记：Actor-Critic算法"
publishDate: 2026-02-22 21:40:00
description: "简述强化学习Actor-Critic算法框架，涵盖QAC、A2C、重要性采样及确定性策略梯度（DPG）的核心推导与更新机制。"
tags: ["Reinforcement Learning", "Actor-Critic", "A2C", "DPG", "学习笔记"]
language: "中文"
---

# Actor-critic

- **Actor** 负责策略更新（Policy update），即决定在给定状态下采取什么动作。
- **Critic** 负责策略评估（Policy evaluation）或价值估计（Value estimation），用于判断 Actor 所选策略的优劣。

## QAC (Q-Actor-Critic)

$$
\begin{aligned}
&\text{Critic (价值更新):} \\
&w_{t+1} = w_t + \alpha_w [r_{t+1} + \gamma q(s_{t+1}, a_{t+1}, w_t) - q(s_t, a_t, w_t)] \nabla_w q(s_t, a_t, w_t) \\
&\text{Actor (策略更新):} \\
&\theta_{t+1} = \theta_t + \alpha_\theta \nabla_\theta \ln \pi(a_t | s_t, \theta_t) q(s_t, a_t, w_{t+1})
\end{aligned}
$$

## A2C（Advantage Actor-Critic）

- 引入基线函数（baseline）来减少方差。

$$
\begin{aligned}
\nabla_{\theta} J(\theta) &= \mathbb{E}_{S \sim \eta, A \sim \pi} \left[ \nabla_{\theta} \ln \pi(A|S, \theta_t) q_{\pi}(S, A) \right] \\
&= \mathbb{E}_{S \sim \eta, A \sim \pi} \left[ \nabla_{\theta} \ln \pi(A|S, \theta_t) (q_{\pi}(S, A) - b(S)) \right]
\end{aligned}
$$

要使上述等式成立，需满足：

$$
\mathbb{E}_{S \sim \eta, A \sim \pi} \left[ \nabla_{\theta} \ln \pi(A|S, \theta_t) b(S) \right] = 0
$$

具体推导如下：

$$
\begin{aligned}
\mathbb{E}_{S \sim \eta, A \sim \pi} \left[ \nabla_{\theta} \ln \pi(A|S, \theta_t) b(S) \right] &= \sum_{s \in \mathcal{S}} \eta(s) \sum_{a \in \mathcal{A}} \pi(a|s, \theta_t) \nabla_{\theta} \ln \pi(a|s, \theta_t) b(s) \\
&= \sum_{s \in \mathcal{S}} \eta(s) \sum_{a \in \mathcal{A}} \nabla_{\theta} \pi(a|s, \theta_t) b(s) \\
&= \sum_{s \in \mathcal{S}} \eta(s) b(s) \nabla_{\theta} \sum_{a \in \mathcal{A}} \pi(a|s, \theta_t) \\
&= \sum_{s \in \mathcal{S}} \eta(s) b(s) \nabla_{\theta} 1 = 0
\end{aligned}
$$

- 基线函数主要用于控制方差，因此目标是找到一个最优的基线函数使得方差最小化。最优的基线函数为：

$$
b^*(s) = \frac{\mathbb{E}_{A \sim \pi} [\|\nabla_{\theta} \ln \pi(A|s, \theta_t)\|^2 q(s, A)]}{\mathbb{E}_{A \sim \pi} [\|\nabla_{\theta} \ln \pi(A|s, \theta_t)\|^2]}
$$

- 由于该形式计算过于复杂，实际应用中通常会去掉权重项 $\mathbb{E}_{A \sim \pi} [\|\nabla_{\theta} \ln \pi(A|s, \theta_t)\|^2]$，即近似为：

$$
b(s) = \mathbb{E}_{A \sim \pi} [q(s, A)] = v_{\pi}(s)
$$

当 $b(s) = v_\pi(s)$ 时：

* **梯度上升算法**为：

$$
\begin{aligned}
\theta_{t+1} &= \theta_t + \alpha \mathbb{E} \left[ \nabla_\theta \ln \pi(A|S, \theta_t) [q_\pi(S, A) - v_\pi(S)] \right] \\
&\doteq \theta_t + \alpha \mathbb{E} \left[ \nabla_\theta \ln \pi(A|S, \theta_t) \delta_\pi(S, A) \right]
\end{aligned}
$$

其中：

$$
\delta_\pi(S, A) \doteq q_\pi(S, A) - v_\pi(S)
$$

该项被称为**优势函数（Advantage function）**。根据 $v_{\pi}(S)$ 的定义，它是状态 $S$ 下所有动作价值的期望。如果某个动作的 $q$ 值大于平均值 $v$，则说明该动作具备“优势”。

* 该算法的**随机版本**为：

$$
\begin{aligned}
\theta_{t+1} &= \theta_t + \alpha \nabla_\theta \ln \pi(a_t|s_t, \theta_t) [q_t(s_t, a_t) - v_t(s_t)] \\
&= \theta_t + \alpha \nabla_\theta \ln \pi(a_t|s_t, \theta_t) \delta_t(s_t, a_t)
\end{aligned}
$$

此外，该算法可以重新表示为：

$$
\begin{aligned}
\theta_{t+1} &= \theta_t + \alpha \nabla_\theta \ln \pi(a_t|s_t, \theta_t) \delta_t(s_t, a_t) \\
&= \theta_t + \alpha \frac{\nabla_\theta \pi(a_t|s_t, \theta_t)}{\pi(a_t|s_t, \theta_t)} \delta_t(s_t, a_t) \\
&= \theta_t + \underbrace{\alpha \left( \frac{\delta_t(s_t, a_t)}{\pi(a_t|s_t, \theta_t)} \right)}_{\text{步长 (step size)}} \nabla_\theta \pi(a_t|s_t, \theta_t)
\end{aligned}
$$

* 更新步长与**相对值 $\delta_t$** 成正比，而非**绝对值 $q_t$**，这在逻辑上更具合理性。
* 它依然能很好地平衡**探索与利用（Exploration and Exploitation）**。

通过 TD 误差进行近似：

$$
\delta_t = q_t(s_t, a_t) - v_t(s_t) \approx r_{t+1} + \gamma v_t(s_{t+1}) - v_t(s_t)
$$

* **这种近似是合理的**，因为：

$$
\mathbb{E}[q_\pi(S, A) - v_\pi(S)|S = s_t, A = a_t] = \mathbb{E}[R_{t+1} + \gamma v_\pi(S_{t+1}) - v_\pi(S_t)|S = s_t, A = a_t]
$$

* **优点**：只需一个神经网络来近似 $v_\pi(s)$，而不需要维护两个网络分别近似 $q_\pi(s, a)$ 和 $v_\pi(s)$。

## 重要性采样和 Off-policy

### Importance sampling technique

注意到：

$$
\mathbb{E}_{X \sim p_0}[X] = \sum_x p_0(x)x = \sum_x p_1(x) \underbrace{\frac{p_0(x)}{p_1(x)} x}_{f(x)} = \mathbb{E}_{X \sim p_1}[f(X)]
$$

* 因此，可以通过估计 $\mathbb{E}_{X \sim p_1}[f(X)]$ 来估计 $\mathbb{E}_{X \sim p_0}[X]$。
* 估计方法如下：

令：

$$
\bar{f} \doteq \frac{1}{n} \sum_{i=1}^n f(x_i), \quad \text{其中 } x_i \sim p_1
$$

那么：

$$
\mathbb{E}_{X \sim p_1}[\bar{f}] = \mathbb{E}_{X \sim p_1}[f(X)]
$$

$$
\text{var}_{X \sim p_1}[\bar{f}] = \frac{1}{n} \text{var}_{X \sim p_1}[f(X)]
$$

所以，$\bar{f}$ 是 $\mathbb{E}_{X \sim p_0}[X]$ 的良好近似：

$$
\mathbb{E}_{X \sim p_0}[X] \approx \bar{f} = \frac{1}{n} \sum_{i=1}^n f(x_i) = \frac{1}{n} \sum_{i=1}^n \frac{p_0(x_i)}{p_1(x_i)} x_i
$$

* 比例 $\frac{p_0(x_i)}{p_1(x_i)}$ 被称为**重要性权重（Importance weight）**。
* 如果 $p_1(x_i) = p_0(x_i)$，重要性权重为 1，$\bar{f}$ 退化为标准算术平均值。
* 如果 $p_0(x_i) > p_1(x_i)$，说明样本 $x_i$ 在分布 $p_0$ 中出现的频率高于 $p_1$。大于 1 的重要性权重会增强该样本在期望计算中的比重。

目标函数定义为：

$$
J(\theta) = \sum_{s \in \mathcal{S}} d_\beta(s) v_\pi(s) = \mathbb{E}_{S \sim d_\beta} [v_\pi(S)]
$$

其梯度为：

$$
\nabla_\theta J(\theta) = \mathbb{E}_{S \sim \rho, A \sim \beta} \left[ \frac{\pi(A | S, \theta)}{\beta(A | S)} \nabla_\theta \ln \pi(A | S, \theta) q_\pi(S, A) \right]
$$

离轨策略（Off-policy）梯度对基准函数 $b(s)$ 同样具有不变性：

$$
\nabla_{\theta} J(\theta) = \mathbb{E}_{S \sim \rho, A \sim \beta} \left[ \frac{\pi(A|S, \theta)}{\beta(A|S)} \nabla_{\theta} \ln \pi(A|S, \theta) (q_{\pi}(S, A) - b(S)) \right]
$$

* 为减小估计方差，同样选择基准函数 $b(S) = v_{\pi}(S)$，得到：

$$
\nabla_{\theta} J(\theta) = \mathbb{E} \left[ \frac{\pi(A|S, \theta)}{\beta(A|S)} \nabla_{\theta} \ln \pi(A|S, \theta) (q_{\pi}(S, A) - v_{\pi}(S)) \right]
$$

对应的随机梯度上升算法为：

$$
\theta_{t+1} = \theta_t + \alpha_{\theta} \frac{\pi(a_t|s_t, \theta_t)}{\beta(a_t|s_t)} \nabla_{\theta} \ln \pi(a_t|s_t, \theta_t) (q_t(s_t, a_t) - v_t(s_t))
$$

类似于同轨（On-policy）情况：

$$
q_t(s_t, a_t) - v_t(s_t) \approx r_{t+1} + \gamma v_t(s_{t+1}) - v_t(s_t) \doteq \delta_t(s_t, a_t)
$$

算法最终形式变为：

$$
\theta_{t+1} = \theta_t + \alpha_{\theta} \frac{\pi(a_t|s_t, \theta_t)}{\beta(a_t|s_t)} \nabla_{\theta} \ln \pi(a_t|s_t, \theta_t) \delta_t(s_t, a_t)
$$

重写后凸显步长关系：

$$
\theta_{t+1} = \theta_t + \alpha_{\theta} \left( \frac{\delta_t(s_t, a_t)}{\beta(a_t|s_t)} \right) \nabla_{\theta} \pi(a_t|s_t, \theta_t)
$$

## Deterministic Actor-Critic (DPG)

策略表示方式的演变：

* 在此之前，通用策略记作 $\pi(a|s, \theta) \in [0, 1]$，它通常是随机的（Stochastic）。
* 现在引入确定性策略（Deterministic policy），记作：

$$
a = \mu(s, \theta) \doteq \mu(s)
$$

* $\mu$ 是从状态空间 $\mathcal{S}$ 到动作空间 $\mathcal{A}$ 的直接映射。
* 实践中 $\mu$ 常由神经网络参数化表示，输入为 $s$，输出直接为动作 $a$，参数为 $\theta$。

目标函数的梯度为：

$$
\begin{aligned}
\nabla_{\theta} J(\theta) &= \sum_{s \in \mathcal{S}} \rho_{\mu}(s) \nabla_{\theta} \mu(s)\left.\left(\nabla_{a} q_{\mu}(s, a)\right)\right|_{a=\mu(s)} \\
&= \mathbb{E}_{S \sim \rho_{\mu}}\left[\left.\nabla_{\theta} \mu(S)\left(\nabla_{a} q_{\mu}(S, a)\right)\right|_{a=\mu(S)}\right]
\end{aligned}
$$

基于确定性策略梯度，最大化 $J(\theta)$ 的梯度上升算法为：

$$
\theta_{t+1} = \theta_t + \alpha_\theta \mathbb{E}_{S \sim \rho_\mu} \left[ \nabla_\theta \mu(S) \left( \nabla_a q_\mu(S, a) \right) |_{a=\mu(S)} \right]
$$

对应的随机梯度上升单步更新为：

$$
\theta_{t+1} = \theta_t + \alpha_\theta \nabla_\theta \mu(s_t) \left( \nabla_a q_\mu(s_t, a) \right) |_{a=\mu(s_t)}
$$

整体架构更新逻辑如下：

**TD 误差**：

$$
\delta_t = r_{t+1} + \gamma q(s_{t+1}, \mu(s_{t+1}, \theta_t), w_t) - q(s_t, a_t, w_t)
$$

**Critic (价值更新)**：

$$
w_{t+1} = w_t + \alpha_w \delta_t \nabla_w q(s_t, a_t, w_t)
$$

**Actor (策略更新)**：

$$
\theta_{t+1} = \theta_t + \alpha_\theta \nabla_\theta \mu(s_t, \theta_t) \left( \nabla_a q(s_t, a, w_{t+1}) \right) |_{a=\mu(s_t)}
$$

* 这是一种**离轨策略（Off-policy**实现。数据收集策略（行为策略 $\beta$）通常不同于当前正在优化的目标策略 $\mu$。
* 为了保证探索性，行为策略 $\beta$ 通常设定为目标策略加上噪声，即 $\beta = \mu + \text{noise}$。
* **关于 $q(s, a, w)$ 的函数近似选择**：
  * **线性函数**：$q(s, a, w) = \phi^T(s, a)w$，其中 $\phi(s, a)$ 为人工设计的特征向量。
  * **神经网络**：当使用深度神经网络近似价值和策略时，即演变为**深度确定性策略梯度（DDPG, Deep Deterministic Policy Gradient**算法。
