---
title: "RL学习笔记：基本概念"
publishDate: 2026-02-15 19:45:00
description: "整理了强化学习中的State、Action、Reward等核心定义，以及马尔可夫决策过程（MDP）的组成要素。"
tags: ["Reinforcement Learning", "MDP", "学习笔记", "数学基础"]
language: "中文"
---

# 基本概念

## 一、核心定义

1. **State (状态)**
   Agent 相对于环境的一个状态（Status）。在网格世界中，通常被视为 Agent 所在的坐标位置。例如 $S_1$ 可表示为向量坐标：
   $$S_1 = \begin{pmatrix} x \\ y \end{pmatrix}$$

2. **State space (状态空间)**
   所有可能状态的集合，记为 $\mathcal{S}$。例如：$S=\left\{s_{i}\right\}_{i=1}^{9}$。本质上就是一个集合（Set）。

3. **Action (动作)**
   对于每一个状态 (State)，Agent 可以采取的行动。例如在网格世界中可能有五个：上、下、左、右、不动。

4. **Action space (动作空间)**
   针对某个特定状态 $s_i$，所有可能采取的动作集合，记为 $\mathcal{A}(s_{i}) = \left\{a_{i}\right\}_{i=1}^{5}$。
   *注意：Action 往往依赖于 State，即 $\mathcal{A}$ 是 $s$ 的函数。*

5. **State transition (状态转移)**
   采取某个行动后，Agent 从当前状态转移到另一个状态的过程，记为 $S_{1}\stackrel{a_2}{\longrightarrow}S_{2}$。
   这定义了 Agent 与环境交互的机制。在虚拟环境中可任意定义，但在现实世界中必须遵循客观物理规律。

6. **State transition probability (状态转移概率)**
   用概率描述状态转移的不确定性。例如在 $S_1$ 选择 $a_2$，转移到 $S_2$ 的概率：
   
   $$
   p(s'|s, a) \Rightarrow \begin{cases} p(s_{2}|s_{1},a_{2})=1 \\ p(s_{i}|s_{1},a_{2})=0, & \forall i\neq2 \end{cases}
   $$
   
   上例为确定性环境，当然也可能是随机环境。

7. **Policy (策略)**
   指导 Agent 在特定 State 下应该采取什么 Action 的规则。可以视为一个函数或映射 $\pi$。
   例如一个确定性策略（Deterministic Policy）：
   
   $$
   \pi(a|s) \Rightarrow \begin{cases} \pi(a_{2}|s_{1})=1 \\ \pi(a_{i}|s_{1})=0, & \forall i\neq2 \end{cases}
   $$
   
   随机策略（Stochastic Policy）同理，$\pi$ 即为选择该动作的概率。

8. **Reward (奖励)**
   Agent 采取动作后，环境反馈的一个**标量实数**。
   
   * 正数通常代表奖励（鼓励行为）；
   * 负数通常代表惩罚（抑制行为）。
   
   Reward 是人机交互（Human-Machine Interface）的关键手段，用于引导 Agent 表现出我们预期的行为。数学表达：
   
   $$
   p(r=-1|s_{1},a_{1})=1 \quad \text{and} \quad p(r\neq-1|s_{1},a_{1})=0
   $$

9. **Trajectory (轨迹)**
   一条完整的 State-Action-Reward 链。即：在某 State 采取某 Action，得到 Reward 并转移到下一 State，如此循环。
   
   $$
   S_{1} \xrightarrow[r=0]{a_3} S_{4} \xrightarrow[r=-1]{a_3} S_{7} \xrightarrow[r=0]{a_2} S_{8} \xrightarrow[r=+1]{a_2} S_{9}
   $$

10. **Return (回报)**
    一个 Trajectory 中所有 Reward 的总和。不同的 Policy 会导致不同的 Return。

11. **Discounted Return (折扣回报)**
    对于无限运行的 Trajectory，直接求和会导致 Return 无穷大（发散）。引入折扣因子 $\gamma \in [0,1)$：
    
    $$
    \text{Return} = 0+0+1+1+\dots = \infty \quad (\text{发散})
    $$
    
    引入 $\gamma$ 后：
    
    $$
    G_t = R_{t+1} + \gamma R_{t+2} + \gamma^2 R_{t+3} + \dots = \sum_{k=0}^{\infty} \gamma^k R_{t+k+1}
    $$
    
    举例：
    
    $$
    \text{Discounted Return} = \gamma^{3}(1+\gamma+\gamma^{2}+\ldots) = \frac{\gamma^{3}}{1-\gamma} \quad (\text{收敛})
    $$
    
    * **$\gamma$ 的作用**：决定 Agent 的“视野”。$\gamma$ 越小越短视（注重眼前利益），$\gamma$ 越大越远视（注重长期利益）。

12. **Episode (回合)**
    Agent 根据 Policy 与环境交互，直到达到**终止状态 (Terminal State)** 停止，这段完整的轨迹称为一个 Episode (或 Trial)。
    
    * **Episodic Tasks**: 有终止状态，任务会结束。
    * **Continuing Tasks**: 没有终止状态，任务无限进行。

---

## 二、MDP (马尔可夫决策过程) 要素

### 1. Sets (集合)

* **State**: 状态集合 $\mathcal{S}$
* **Action**: 动作集合 $\mathcal{A}(s)$，其中 $s \in \mathcal{S}$
* **Reward**: 奖励集合 $\mathcal{R}(s,a)$

### 2. Probability Distribution (概率分布/动力学)

* **State transition probability**: $p(s'|s,a)$
* **Reward probability**: $p(r|s,a)$

### 3. Policy (策略)

* Agent 的决策机制：$\pi(a|s)$

### 4. MDP Property (性质)

**Memoryless (无记忆性 / 马尔可夫性)**:
下一时刻的状态和奖励，仅取决于当前时刻的状态和动作，与之前的历史无关。

$$
p(s_{t+1}, r_{t+1} | s_t, a_t, s_{t-1}, \dots) = p(s_{t+1}, r_{t+1} | s_t, a_t)
$$

### 5. MDP vs Markov Process

* **Markov Process (马尔可夫过程)**: 只有 State 和 Transition Probability。观察者只能被动接受环境按概率发生的演变，无法干预。
* **MDP (马尔可夫决策过程)**: 增加了 **Decision (决策/动作)**。
    状态的转移不仅取决于当前状态，还取决于 Agent 采取的 **Action**。Agent 可以通过选择不同的 Action 来改变未来状态分布的概率，从而主动影响结果。
