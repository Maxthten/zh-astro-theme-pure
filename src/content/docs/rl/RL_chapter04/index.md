---
title: "RL学习笔记：值迭代与策略迭代"
publishDate: 2026-02-18 10:30:00
description: "深入解析值迭代（Value Iteration）与策略迭代（Policy Iteration）的核心算法流程，推导策略更新与值更新的数学形式。探讨了截断策略迭代（Truncated Policy Iteration）如何通过调整评估步数，在统一视角下连接这两种经典算法。"
tags: ["Reinforcement Learning", "Value Iteration", "Policy Iteration", "Truncated Policy Iteration", "学习笔记"]
language: "中文"

---

# 值迭代与策略迭代

## 值迭代 (Value Iteration)

值迭代通过迭代更新值函数 $v_k$ 来逼近最优值函数 $v_*$。其核心迭代公式为：

$$
v_{k+1} = f(v_k) = \max_{\pi} (r_{\pi} + \gamma P_{\pi} v_k), \quad k = 1, 2, 3 \dots
$$

该过程可分解为两个步骤：

1. **策略更新 (Policy Update)**：
   
   $$
   \pi_{k+1} = \arg\max_{\pi} (r_{\pi} + \gamma P_{\pi} v_k)
   $$

2. **值更新 (Value Update)**：
   
   $$
   v_{k+1} = r_{\pi_{k+1}} + \gamma P_{\pi_{k+1}} v_k
   $$

*注：此处的 $v_k$ 是第 $k$ 次迭代的估值向量，而非最终的 state value。*

### 1. 策略更新 (Policy Update)

$$
\pi_{k+1} = \arg\max_{\pi} (r_{\pi} + \gamma P_{\pi} v_k)
$$

其逐元素形式 (Elementwise form) 为：

$$
\pi_{k+1}(s) = \arg\max_{\pi} \sum_{a} \pi(a|s) \underbrace{\left( \sum_{r} p(r|s,a)r + \gamma \sum_{s'} p(s'|s,a)v_k(s') \right)}_{q_k(s,a)}, \quad s \in \mathcal{S}
$$

由此得到的 $\pi_{k+1}$ 为贪婪策略 (Greedy Policy)，即在状态 $s$ 下选择使 $q_k(s,a)$ 最大的动作 $a_k^*(s)$：

$$
a_k^*(s) = \arg\max_{a} q_k(a, s)
$$

$$
\pi_{k+1}(a|s) = \begin{cases} 1, & a = a_k^*(s) \\ 0, & a \neq a_k^*(s) \end{cases}
$$

### 2. 值更新 (Value Update)

$$
v_{k+1} = r_{\pi_{k+1}} + \gamma P_{\pi_{k+1}} v_k
$$

其逐元素形式为：

$$
v_{k+1}(s) = \sum_{a} \pi_{k+1}(a|s) \underbrace{ \left( \sum_{r} p(r|s,a)r + \gamma \sum_{s'} p(s'|s,a)v_k(s') \right) }_{q_k(s,a)}, \quad s \in \mathcal{S}
$$

由于 $\pi_{k+1}$ 是贪婪策略，上式等价于：

$$
v_{k+1}(s) = \max_{a} q_k(a, s)
$$

---

## 策略迭代 (Policy Iteration)

策略迭代包含以下步骤：

1. **初始化**：给定一个随机的初始策略 $\pi_0$。

2. **策略评估 (Policy Evaluation, PE)**：计算当前策略的状态价值 $v_{\pi_k}$。
   
   $$
   v_{\pi_k} = r_{\pi_k} + \gamma P_{\pi_k} v_{\pi_k}
   $$

3. **策略提升 (Policy Improvement, PI)**：基于当前价值生成更好的策略。
   
   $$
   \pi_{k+1} = \arg \max_{\pi} (r_\pi + \gamma P_\pi v_{\pi_k})
   $$

### 1. 策略评估 (Policy Evaluation)

求解 $v_{\pi_k}$ 通常采用迭代法：

* **矩阵-向量形式**：
  
  $$
  v_{\pi_k}^{(j+1)} = r_{\pi_k} + \gamma P_{\pi_k} v_{\pi_k}^{(j)}, \quad j = 0, 1, 2, \dots
  $$

* **逐元素形式**：
  
  $$
  v_{\pi_k}^{(j+1)}(s) = \sum_{a} \pi_k(a|s) \left( \sum_{r} p(r|s, a)r + \gamma \sum_{s'} p(s'|s, a)v_{\pi_k}^{(j)}(s') \right), \quad s \in \mathcal{S}
  $$
  
  

停止条件：当 $j \to \infty$ 或 $\| v_{\pi_k}^{(j+1)} - v_{\pi_k}^{(j)} \|$ 足夠小时停止迭代。

### 2. 策略提升 (Policy Improvement)

* **矩阵-向量形式**：
  
  $$
  \pi_{k+1} = \arg \max_{\pi} (r_\pi + \gamma P_\pi v_{\pi_k})
  $$

* **逐元素形式**：
  
  $$
  \pi_{k+1}(s) = \arg \max_{\pi} \sum_{a} \pi(a|s) \underbrace{\left( \sum_{r} p(r|s, a)r + \gamma \sum_{s'} p(s'|s, a)v_{\pi_k}(s') \right)}_{q_{\pi_k}(s, a)}, \quad s \in \mathcal{S}
  $$

令 $a_k^*(s) = \arg \max_{a} q_{\pi_k}(a, s)$，更新策略为确定性贪婪策略：

$$
\pi_{k+1}(a|s) = \begin{cases} 1, & a = a_k^*(s) \\ 0, & a \neq a_k^*(s) \end{cases}
$$

---

## 截断策略迭代 (Truncated Policy Iteration)

我们可以从“策略评估的步数”这一角度来统一值迭代和策略迭代：

$$
\begin{alignat*}{2}
\text{Policy Iteration: } & \pi_0 \xrightarrow{PE} v_{\pi_0} \xrightarrow{PI} \pi_1 \xrightarrow{PE} v_{\pi_1} \xrightarrow{PI} \pi_2 \dots \\
\text{Value Iteration: }  & \phantom{\pi_0 \xrightarrow{PE}} v_0 \xrightarrow{PU} \pi_1' \xrightarrow{VU} v_1 \xrightarrow{PU} \pi_2' \dots
\end{alignat*}
$$

* **Policy Iteration**：$\text{P} \to \text{vvvv...} \to \text{P} \to \text{vvvv...}$ (评估至收敛)
* **Value Iteration**：$\text{P} \to \text{v} \to \text{P} \to \text{v} \to \text{P} \to \text{v}$ (评估仅做一步)

**统一视角下的算法对比：**

$$
\begin{array}{rll}
    & v_{\pi_1}^{(0)} = v_0 & \text{初始值} \\
    \text{Value Iteration} \leftarrow v_1 \longleftarrow & v_{\pi_1}^{(1)} = r_{\pi_1} + \gamma P_{\pi_1} v_{\pi_1}^{(0)} & \text{(只迭代 1 次)} \\
    & v_{\pi_1}^{(2)} = r_{\pi_1} + \gamma P_{\pi_1} v_{\pi_1}^{(1)} & \\
    & \quad \vdots & \\
    \text{Truncated Policy Iteration} \leftarrow \bar{v}_1 \longleftarrow & v_{\pi_1}^{(j)} = r_{\pi_1} + \gamma P_{\pi_1} v_{\pi_1}^{(j-1)} & \text{(迭代 j 次)} \\
    & \quad \vdots & \\
    \text{Policy Iteration} \leftarrow v_{\pi_1} \longleftarrow & v_{\pi_1}^{(\infty)} = r_{\pi_1} + \gamma P_{\pi_1} v_{\pi_1}^{(\infty)} & \text{(迭代至收敛)}
\end{array}
$$

**注意**：标准的 Policy Iteration 要求在每一步都精确求解 $v_{\pi_k}$（即 $j \to \infty$），这在实际计算中往往不可行或效率低下。因此，实际应用中使用的通常是 **Truncated Policy Iteration**，即限制评估步数 $j$。
