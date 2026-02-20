---
title: "RL学习笔记：时序差分算法"
publishDate: 2026-02-20 20:40:00
description: "整理时序差分算法的核心思想，对比TD与MC的区别，并梳理Sarsa、n-step Sarsa与Q-learning等经典算法的更新机制与理论依据。"
tags: ["Reinforcement Learning", "TD Learning", "Sarsa", "Q-learning", "学习笔记"]
language: "中文"
---

# 时序差分算法（Temporal-Difference learning）

## TD learning of state values

- 该算法用于求解给定策略 $\pi$ 下的状态价值 (state value)。
- 本节主要介绍用于估计状态价值的基础 TD 算法（通常指 TD(0)）。

**算法所需的数据/经验：**

* $(s_0, r_1, s_1, \dots, s_t, r_{t+1}, s_{t+1}, \dots)$ 或 $\{(s_t, r_{t+1}, s_{t+1})\}_t$，这些数据是按照给定策略 $\pi$ 与环境交互生成的。

**TD 学习算法（TD learning algorithm）更新公式：**

$$
v_{t+1}(s_t) = v_t(s_t) - \alpha_t(s_t) \big[ v_t(s_t) - [r_{t+1} + \gamma v_t(s_{t+1})] \big], \quad (1)
$$

$$
v_{t+1}(s) = v_t(s), \quad \forall s \neq s_t, \quad (2)
$$

其中 $t = 0, 1, 2, \dots$。这里，$v_t(s_t)$ 是对状态价值 $v_\pi(s_t)$ 的当前估计；$\alpha_t(s_t)$ 是在时刻 $t$ 状态 $s_t$ 下的学习率。

* 在时刻 $t$，仅更新当前访问状态 $s_t$ 的价值，而未访问状态 $s \neq s_t$ 的价值保持不变。
* 在语境明确时，式 (2) 的保持步骤通常会被省略。

具体分析核心更新式：

$$
v_{t+1}(s_t) = v_t(s_t) - \alpha_t(s_t) \big[ v_t(s_t) - [r_{t+1} + \gamma v_t(s_{t+1})] \big]
$$

- **TD target** : $\bar{v}_t \doteq r_{t+1} + \gamma v_t(s_{t+1})$
- **TD error**: $\delta_t \doteq v_t(s_t) - [r_{t+1} + \gamma v_t(s_{t+1})] = v_t(s_t) - \bar{v}_t$

算法的核心思想是使当前的估计值 $v_t(s_t)$ 不断向目标值 $\bar{v}_t$ 靠拢。推导如下：

$$
\begin{aligned}
& v_{t+1}(s_t) = v_t(s_t) - \alpha_t(s_t) [v_t(s_t) - \bar{v}_t] \\
\implies & v_{t+1}(s_t) - \bar{v}_t = v_t(s_t) - \bar{v}_t - \alpha_t(s_t) [v_t(s_t) - \bar{v}_t] \\
\implies & v_{t+1}(s_t) - \bar{v}_t = [1 - \alpha_t(s_t)] [v_t(s_t) - \bar{v}_t] \\
\implies & |v_{t+1}(s_t) - \bar{v}_t| = |1 - \alpha_t(s_t)| |v_t(s_t) - \bar{v}_t|
\end{aligned}
$$

当学习率 $\alpha_t(s_t) \in (0, 1)$ 时，显然 $0 < 1 - \alpha_t(s_t) < 1$，这意味着估计值与目标值之间的距离被逐步压缩，使其不断向目标收敛。

关于 **TD error**：

$$
\delta_t = v_t(s_t) - [r_{t+1} + \gamma v_t(s_{t+1})]
$$

* 它是两个连续时间步 (time steps) 价值估计之间的差异。
* 它反映了当前估计 $v_t$ 与真实价值 $v_\pi$ 之间的偏差 (deficiency)。为了证明这一点，定义真实价值下的误差：

$$
\delta_{\pi,t} \doteq v_\pi(s_t) - [r_{t+1} + \gamma v_\pi(s_{t+1})]
$$

根据 $v_\pi(s_t)$ 的定义，对其取期望：

$$
\mathbb{E}[\delta_{\pi,t} | S_t = s_t] = v_\pi(s_t) - \mathbb{E}[R_{t+1} + \gamma v_\pi(S_{t+1}) | S_t = s_t] = 0
$$

> 由上式可知，算法的目标是在期望意义上满足贝尔曼方程。

* 如果 $v_t = v_\pi$，那么 $\delta_t$ 的期望应该为零。
* 反之，如果 $\delta_t$ 不为零，说明 $v_t$ 尚未收敛到 $v_\pi$。
* **TD error** 在本质上可视为一种创新信息 (innovation)，代表从最新经验 $(s_t, r_{t+1}, s_{t+1})$ 中获取的能够用于修正当前认知的反馈。

上述过程主要用于策略评估 (Policy Evaluation)。TD 算法的优势在于无需环境的完整模型（转移概率等），就能直接通过采样数据近似计算贝尔曼方程。

### Bellman expectation equation

策略 $\pi$ 的状态价值定义为：

$$
v_\pi(s) = \mathbb{E}[R + \gamma G | S = s], \quad s \in \mathcal{S}
$$

其中 $G$ 是折扣回报 (discounted return)。由于：

$$
\mathbb{E}[G | S = s] = \sum_{a} \pi(a|s) \sum_{s'} p(s'|s, a) v_\pi(s') = \mathbb{E}[v_\pi(S') | S = s]
$$

其中 $S'$ 是下一状态 (next state)，我们可以将方程改写为贝尔曼期望方程：

$$
v_\pi(s) = \mathbb{E}[R + \gamma v_\pi(S') | S = s], \quad s \in \mathcal{S}.
$$

我们可以借助随机近似 (Robbins-Monro) 算法的思想来求解该方程：

$$
g(v(s)) = v(s) - \mathbb{E}[R + \gamma v_\pi(S') | s] = 0
$$

由于在实际中只能获得 $R$ 和 $S'$ 的采样样本 $r$ 和 $s'$，我们得到的带噪观测值 (noisy observation) 为：

$$
\tilde{g}(v(s)) = v(s) - [r + \gamma v_\pi(s')]
$$

它可以分解为真实梯度与噪声之和：

$$
= \underbrace{\left(v(s) - \mathbb{E}[R + \gamma v_\pi(S') | s]\right)}_{g(v(s))} + \underbrace{\left(\mathbb{E}[R + \gamma v_\pi(S') | s] - [r + \gamma v_\pi(s')]\right)}_{\eta}.
$$

套用更新法则：

$$
\begin{aligned}
v_{k+1}(s) &= v_k(s) - \alpha_k \tilde{g}(v_k(s)) \\
&= v_k(s) - \alpha_k \left( v_k(s) - \left[ r_k + \gamma v_\pi(s'_k) \right] \right), \quad k=1, 2, 3, \dots
\end{aligned}
$$

相较于理论方程，实际应用的 TD 算法做了两点核心近似（修改）：

- 将 $\{s, r, s'\}$ 替换为实际轨迹采样 $\{s_t, r_{t+1}, s_{t+1}\}$。即不再需要反复在特定状态 $s$ 采样大量数据，而是访问到 $s_t$ 时就进行一次单步更新，未访问的状态保持不变。
- 将未知的真实值 $v_{\pi}(s_k')$ 替换为当前的估计值 $v_k(s_k')$，即引入了自举 (Bootstrapping) 机制。

### TD vs MC

| 特性                     | TD / Sarsa 学习 (TD/Sarsa learning)                                                                  | MC 学习 (MC learning)                                                                                                     |
|:---------------------- |:-------------------------------------------------------------------------------------------------- |:----------------------------------------------------------------------------------------------------------------------- |
| **在线/离线**              | **在线 (Online)**：TD 学习是在线的。它可以在执行单步转移收到奖励后，立即更新状态/动作价值。                                             | **离线 (Offline)**：MC 学习是离线的。它必须等待整个回合 (episode) 结束后，才能利用完整的轨迹回报进行更新。                                                     |
| **任务类型**               | **连续性任务 (Continuing tasks)**：得益于单步更新的特性，它可以无缝处理回合制任务和无限长的连续性任务。                                    | **回合制任务 (Episodic tasks)**：由于需要等待终点计算总回报，它只能处理有明确终止状态的回合制任务。                                                            |
| **自举 (Bootstrapping)** | **自举 (Bootstrapping)**：更新依赖于对下一个状态价值的当前估计（用估计更新估计），因此需要设定初始猜测值。                                    | **非自举 (Non-bootstrapping)**：直接使用实际采样的完整累积回报来估计价值，无需依赖其他状态的现有估计。                                                         |
| **估计方差 (Variance)**    | **低估计方差 (Low estimation variance)**：单步更新涉及的随机变量少。例如 Sarsa 仅引入了 $R_{t+1}, S_{t+1}, A_{t+1}$ 的单步随机性。 | **高估计方差 (High estimation variance)**：为了估计价值，需要累加整个轨迹的奖励 $R_{t+1} + \gamma R_{t+2} + \dots$ 假设回合长度为 $L$，长序列会引入巨大的随机性和方差。 |

## TD learning of action values: Sarsa

Sarsa 旨在估计给定策略 $\pi$ 的动作价值函数 $q_\pi$。算法通过以下经验序列序列进行学习：

$$
\{(s_t, a_t, r_{t+1}, s_{t+1}, a_{t+1})\}_t
$$

对于在 $t$ 时刻访问的状态-动作对 $(s_t, a_t)$，更新公式为：

$$
q_{t+1}(s_t, a_t) = q_t(s_t, a_t) - \alpha_t(s_t, a_t) \left[ q_t(s_t, a_t) - (r_{t+1} + \gamma q_t(s_{t+1}, a_{t+1})) \right]
$$

对于其他未访问的状态-动作对 $(s, a) \neq (s_t, a_t)$，其价值保持不变：

$$
q_{t+1}(s, a) = q_t(s, a)
$$

其中：

* $q_t(s_t, a_t)$：动作价值 $q_\pi(s_t, a_t)$ 的当前估计值。
* $\alpha_t(s_t, a_t)$：学习率，通常随时间衰减以保证收敛。
* $r_{t+1} + \gamma q_t(s_{t+1}, a_{t+1})$：TD 目标值。这里体现了 Sarsa “同策略”（On-policy）的特点，即严格使用当前策略 $\pi$ 实际采取的下一个动作 $a_{t+1}$ 来计算目标并更新当前值。

> 严格来说，上述单步公式属于 Sarsa 的策略评估部分。在实际控制问题中，Sarsa 算法通常代指结合了该评估公式与策略改进机制（如 $\epsilon$-greedy）的完整控制闭环。

## Expected Sarsa

Expected Sarsa 改进了标准 Sarsa 的目标计算方式：

$$
q_{t+1}(s_t, a_t) = q_t(s_t, a_t) - \alpha_t(s_t, a_t) \left[ q_t(s_t, a_t) - \left(r_{t+1} + \gamma \mathbb{E}[q_t(s_{t+1}, A)]\right) \right]
$$

$$
q_{t+1}(s, a) = q_t(s, a), \quad \forall (s, a) \neq (s_t, a_t)
$$

其中期望项展开为：

$$
\mathbb{E}[q_t(s_{t+1}, A)] = \sum_a \pi_t(a|s_{t+1}) q_t(s_{t+1}, a) \doteq v_t(s_{t+1})
$$

即在当前策略 $\pi_t$ 下，下一状态所有可能动作价值的期望值。

* **TD target 的变化**：Sarsa 依赖单次采样的特定动作 $a_{t+1}$，而 Expected Sarsa 对下一步的所有可能动作按策略概率求了期望。
* **降低方差**：虽然每个时间步需要计算所有候选动作的价值（增加了少量计算量），但它消除了选择下一步动作 $A_{t+1}$ 带来的随机性。目标值涉及的随机变量从 $\{s_t, a_t, r_{t+1}, s_{t+1}, a_{t+1}\}$ 缩减到了 $\{s_t, a_t, r_{t+1}, s_{t+1}\}$，从而有效降低了估计方差。

其本质试图求解的数学期望等式为：

$$
q_\pi(s, a) = \mathbb{E} \left[ R_{t+1} + \gamma \mathbb{E}_{A_{t+1} \sim \pi(\cdot|S_{t+1})} [q_\pi(S_{t+1}, A_{t+1})] \middle| S_t = s, A_t = a \right]
$$

即：

$$
q_\pi(s, a) = \mathbb{E} \left[ R_{t+1} + \gamma v_\pi(S_{t+1}) \middle| S_t = s, A_t = a \right]
$$

## n-step Sarsa

n-step Sarsa 是一种广义的方法，单步 Sarsa 和完整视角的 MC 都可以看作是它的极端情况。核心在于截断回报 (truncated return) 的步数差异：

$$
\begin{aligned}
\text{Sarsa (1-step)} \longleftarrow \quad G_t^{(1)} &= R_{t+1} + \gamma q_\pi(S_{t+1}, A_{t+1}) \\
G_t^{(2)} &= R_{t+1} + \gamma R_{t+2} + \gamma^2 q_\pi(S_{t+2}, A_{t+2}) \\
&\vdots \\
n\text{-step Sarsa} \longleftarrow \quad G_t^{(n)} &= R_{t+1} + \gamma R_{t+2} + \cdots + \gamma^n q_\pi(S_{t+n}, A_{t+n}) \\
&\vdots \\
\text{MC} \longleftarrow \quad G_t^{(\infty)} &= R_{t+1} + \gamma R_{t+2} + \gamma^2 R_{t+3} + \cdots
\end{aligned}
$$

它们试图拟合的理论期望：

- Sarsa (1-step):
  
  $$
  q_{\pi}(s, a) = \mathbb{E}[G_t^{(1)} | s, a] = \mathbb{E}[R_{t+1} + \gamma q_{\pi}(S_{t+1}, A_{t+1}) | s, a]
  $$
  
  

- MC learning:
  
  $$
  q_{\pi}(s, a) = \mathbb{E}[G_t^{(\infty)} | s, a] = \mathbb{E}[R_{t+1} + \gamma R_{t+2} + \gamma^2 R_{t+3} + \dots | s, a]
  $$

- n-step Sarsa:
  
  $$
  q_{\pi}(s, a) = \mathbb{E}[G_t^{(n)} | s, a] = \mathbb{E}[R_{t+1} + \gamma R_{t+2} + \dots + \gamma^n q_{\pi}(S_{t+n}, A_{t+n}) | s, a]
  $$

**$n$-step Sarsa 更新算法**：

理论上的更新公式为：

$$
q_{t+1}(s_t, a_t) = q_t(s_t, a_t) - \alpha_t(s_t, a_t) \left[ q_t(s_t, a_t) - [r_{t+1} + \gamma r_{t+2} + \dots + \gamma^n q_t(s_{t+n}, a_{t+n})] \right]
$$

* 实际执行中，由于在 $t$ 时刻环境只推进了一步，尚无法获取未来 $n$ 步的完整数据序列 $(r_{t+1}, \dots, r_{t+n}, s_{t+n}, a_{t+n})$。
* 因此，$n$-step 的实际更新需要**延迟 $n$ 个时间步**。在时刻 $t+n$，收集齐所需数据后，再回过头去更新历史状态动作对 $(s_t, a_t)$：

$$
q_{t+n}(s_t, a_t) = q_{t+n-1}(s_t, a_t) - \alpha_{t+n-1}(s_t, a_t) \left[ q_{t+n-1}(s_t, a_t) - \left[ r_{t+1} + \dots + \gamma^n q_{t+n-1}(s_{t+n}, a_{t+n}) \right] \right]
$$

简写为：

$$
q_{t+n}(s_t, a_t) = q_{t+n-1}(s_t, a_t) - \alpha_{t+n-1}(s_t, a_t) \left[ q_{t+n-1}(s_t, a_t) - Target_{n-step} \right]
$$

*(注：下标 $t$ 代表时间步的流逝，而 $n$ 代表往后看多远的视野视距。)*

* $n$-step Sarsa 是一种在 Sarsa 与 MC 之间权衡的算法：
  * **$n$ 较大时**，引入的真实奖励项较多，更接近 MC 方法。优势是偏差 (bias) 较小，受初始错误估计的影响低，但代价是累积了更多随机性，导致方差 (variance) 变大。
  * **$n$ 较小时**，更依赖于自举，接近 Sarsa。此时方差较低，但如果初始估计误差很大，则由于强依赖现存的 $q$ 值，容易引入较大的偏差。
* 与单步方法一样，$n$-step Sarsa 同样可以内嵌到策略迭代框架中，与策略改进步骤结合以寻找最优策略。

## Q-learning

Q-learning 是异策略 (off-policy) TD 控制算法的代表，其评估直接指向最优状态动作价值：

$$
\begin{align}
q_{t+1}(s_t, a_t) &= q_t(s_t, a_t) - \alpha_t(s_t, a_t) \left[ q_t(s_t, a_t) - \left[ r_{t+1} + \gamma \max_{a \in \mathcal{A}} q_t(s_{t+1}, a) \right] \right], \\
q_{t+1}(s, a) &= q_t(s, a), \quad \forall (s, a) \neq (s_t, a_t),
\end{align}
$$

它在数学上试图求解的是贝尔曼最优方程 (Bellman Optimality Equation, BOE)：

$$
q(s, a) = \mathbb{E} \left[ R_{t+1} + \gamma \max_{a'} q(S_{t+1}, a') \mid S_t = s, A_t = a \right], \quad \forall s, a.
$$

### on-policy vs off-policy

在强化学习控制问题中，我们往往涉及到两种策略：

- **Behavior policy (行动策略/行为策略)**：代理（Agent）与环境实际交互、用于生成经验样本的策略（通常具备探索性，如 $\epsilon$-greedy）。
- **Target policy (目标策略)**：算法实际评估和试图优化并最终收敛到的策略（通常是完全贪婪策略）。

根据这两种策略是否一致，可以划分为：

- **On-policy (同策略)**：目标策略与行动策略完全一致。生成样本的规则和评估的规则是同一套逻辑。
- **Off-policy (异策略)**：目标策略与行动策略分离。代理用一种策略去探索世界，同时利用这些探索数据在后台默默优化另一个（通常是更优的）策略。

> Off-policy 的核心优势：算法可以复用其他任意策略（甚至是人类专家历史操作或随机游走）产生的经验数据进行学习，数据利用效率更高。

**如何区分一个算法是 On-policy 还是 Off-policy？**
核心在于检查它在计算 TD Target 时，所使用的下一个动作 $a_{t+1}$ 的来源机制：

- **Sarsa 属于 On-policy**：它的目标依赖于下一步实际执行的动作 $a_{t+1}$，这个动作是由当前的行动策略决定的。
- **Q-learning 属于 Off-policy**：无论下一步实际上采取了什么动作去探索环境，在计算目标值时，它始终激进地假设下一步会采取当前估计下的最优动作（$\max_a$）。

### TD算法统一形式与总结

价值更新一般可以统一为以下形式：

$$
q_{t+1}(s_t, a_t) = q_t(s_t, a_t) - \alpha_t(s_t, a_t)[q_t(s_t, a_t) - \bar{q}_t],
$$

其中 $\bar{q}_t$ 为各算法对应的 TD target：

| **Algorithm**  | **Expression of TD target ($\bar{q}_t$)**                                       |
| -------------- | ------------------------------------------------------------------------------- |
| Sarsa          | $\bar{q}_t = r_{t+1} + \gamma q_t(s_{t+1}, a_{t+1})$                            |
| $n$-step Sarsa | $\bar{q}_t = r_{t+1} + \gamma r_{t+2} + \dots + \gamma^n q_t(s_{t+n}, a_{t+n})$ |
| Expected Sarsa | $\bar{q}_t = r_{t+1} + \gamma \sum_a \pi_t(a\|s_{t+1})q_t(s_{t+1},a)$           |
| Q-learning     | $\bar{q}_t = r_{t+1} + \gamma \max_a q_t(s_{t+1}, a)$                           |
| Monte Carlo    | $\bar{q}_t = r_{t+1} + \gamma r_{t+2} + \dots$                                  |

*(注：当令学习率 $\alpha_t=1$ 时，MC 形式可以直接写为 $q_{t+1}(s_t, a_t) = \bar{q}_t$，即直接用单次回报替代估计值。标准 MC 通常维护均值，等价于逐步衰减的 $\alpha$)*

各算法理论求解的底层方程总结：

| **Algorithm**  | **Equation aimed to solve**                                                                                                                 |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Sarsa          | Bellman Expectation: $q_\pi(s, a) = \mathbb{E} [R_{t+1} + \gamma q_\pi(S_{t+1}, A_{t+1}) \mid S_t = s, A_t = a]$                            |
| n-step Sarsa   | Bellman Expectation: $q_\pi(s, a) = \mathbb{E} [R_{t+1} + \gamma R_{t+2} + \dots + \gamma^n q_\pi(s_{t+n}, a_{t+n}) \mid S_t = s, A_t = a]$ |
| Expected Sarsa | Bellman Expectation: $q_\pi(s, a) = \mathbb{E} [R_{t+1} + \gamma \mathbb{E}_{A_{t+1}} [q_\pi(S_{t+1}, A_{t+1})] \mid S_t = s, A_t = a]$     |
| Q-learning     | Bellman Optimality: $q_*(s, a) = \mathbb{E} [R_{t+1} + \gamma \max_a q_*(S_{t+1}, a) \mid S_t = s, A_t = a]$                                |
| Monte Carlo    | Bellman Expectation: $q_\pi(s, a) = \mathbb{E} [R_{t+1} + \gamma R_{t+2} + \dots \mid S_t = s, A_t = a]$                                    |
