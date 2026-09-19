# Jev：把智能变成可编程的决策

> 从 System One Model、Noul / Choice / Score，到并行概率判断、RLCD、在线强化学习与决策蒸馏的一套统一理解

**发布日期：2026-09-20**

Jev 最值得关注的地方，不只是“更快的模型”，而是一种不同的智能接口：把生成式模型擅长的开放式语言能力，收缩成软件可以直接消费的、类型明确的概率化决策。

TypeSafe AI 在 2026 年 9 月 15 日发布 Jev，将其称为第一款 System One Model。官方给出的定位是：输入非结构化或结构化状态，输出预先定义类型中的概率化决策；模型不依赖逐 token 的自回归文本生成，而是面向大量可并行的窄判断进行优化。官方同时公布了新的 model architecture、parallel sampler 和 Reinforcement Learning for Calibrated Decisions（RLCD），但尚未公开完整架构、训练数据与 RLCD 的具体算法。

这篇文章试图做两件事：

1. 基于官方文档，把 Jev 已知的接口、行为与展示案例整理成一个清晰的工程模型；
2. 在明确区分“官方事实”和“架构推演”的前提下，进一步推导一种可能的训练哲学：Jev 也许可以被理解为把昂贵的 System-2 决策过程，通过决策蒸馏和在线概率塑形，压缩成高速的 System-1 决策函数。

---

## 1. Jev 应该怎样理解

传统 LLM 的典型路径是：

~~~
state / prompt
    ↓
reasoning
    ↓
autoregressive token generation
    ↓
text / JSON / tool call
    ↓
parse + validate
    ↓
software action
~~~

Jev 的理想路径则更接近：

~~~
state
  +
decision specification
    ↓
semantic decision model
    ↓
typed probability distribution
    ↓
ordinary code
    ↓
action
~~~

它不以“写出一个回答”为第一目标，而以“在已定义的决策空间里判断什么更成立”为第一目标。

TypeSafe 将这种能力描述成面向软件的 intelligence primitive。软件先定义问题和合法输出空间，模型负责处理其中无法用确定规则表达的语义判断，随后普通代码根据概率、阈值和业务规则决定下一步。

因此，Jev 的关键不是“让模型更听话地生成 JSON”，而是改变模型与软件之间的接口边界。

---

## 2. Noul、Choice、Score：三个表面接口，一个共同核心

Jev 当前公开的三个 primitive 是 Noul、Choice 和 Score。

### 2.1 Noul：二元判断

Noul 接收一个 yes/no 问题，返回“yes 成立”的概率：

$$
P(\mathrm{yes}\mid s,q)
$$

例如：

~~~
state:
用户已经修改代码，但还没有运行测试。

question:
当前是否应该运行测试？
~~~

返回可以是：

~~~
noul = 0.96
~~~

这里 0.96 不是一段语言，而是 API 中直接可消费的数值。

### 2.2 Choice：有限决策空间

Choice 让软件提前定义候选项：

~~~
inspect
edit
run_test
finish
~~~

模型返回所有选项上的概率分布，并选择概率最大的候选：

$$
\sum_i P(c_i)=1
$$

以及：

$$
c^*=\arg\max_i P(c_i)
$$

官方文档说明，一个 Choice 当前最多支持 255 个 options，并且 Choice 内各 option 的概率和为 1。

### 2.3 Score：有序决策空间

Score 不是让模型凭空生成一个任意实数，而是先定义若干具有语义描述的有序 level，例如：

~~~
0 = Cosmetic
1 = Feature degraded but workaround exists
2 = Blocking, no workaround
~~~

模型对每个 level 给出概率：

$$
P(L_0),P(L_1),P(L_2)
$$

最终 score 是各 level 位置的概率期望：

$$
Score=\sum_i iP(L_i)
$$

例如：

$$
0\times0+1\times0.7+2\times0.3=1.3
$$

因此 Noul、Choice、Score 都可以看作“对一个有限语义空间进行概率判断”，差别主要发生在外围解释方式。

---

## 3. 一个统一抽象：fθ(s, q, c)

为了更简洁地理解 Jev，可以把三个 primitive 进一步压缩成一个统一函数：

$$
\boxed{
f_\theta(s,q,c)\rightarrow r
}
$$

其中：

- $s$：当前状态 state；
- $q$：完整的 decision specification，包括问题、决策空间、候选含义与约束；
- $c$：当前唯一待评估的 candidate；
- $r$：对这个 candidate 的 scalar judgment，可理解为 logit、energy 或 probability-like score。

这里有一个重要定义：**完整候选空间已经被包含在 q 中。**

因此不必单独再写：

$$
P(c_i\mid s,q,C)
$$

因为可以定义：

$$
q'=(q,C)
$$

于是统一为：

$$
P(c_i\mid s,q')
$$

这种写法还有一个好处：即使 candidate 本身相同，只要决策空间发生变化，q 就会变化，因此模型仍然可以感知候选之间的相互关系。

例如：

~~~
q1:
下一步做什么？
choices = [run_test, finish]

q2:
下一步做什么？
choices = [inspect_logs, run_test, finish]
~~~

虽然 c 都是 run_test，但：

$$
q_1\neq q_2
$$

所以完全允许：

$$
f(s,q_1,\mathrm{run\_test})
\neq
f(s,q_2,\mathrm{run\_test})
$$

### 3.1 用这一抽象重新解释三个 primitive

Noul 可以理解为：

$$
q=\{\mathrm{True},\mathrm{False}\}
$$

只需显式返回：

$$
P(\mathrm{True})
$$

Choice 可以理解为对每个候选并行求：

$$
z_i=f_\theta(s,q,c_i)
$$

然后在当前 Choice 内做归一化：

$$
p_i=\mathrm{Normalize}(z_1,\ldots,z_n)_i
$$

具体是否使用标准 softmax，TypeSafe 尚未公开。

Score 则可以看作 ordered Choice，再由 runtime 计算期望值。

由此，一个可能的最小内部抽象是：

$$
\boxed{
\mathrm{SemanticDecision}(s,q,c)\rightarrow scalar
}
$$

Noul、Choice、Score 只是它在 API 层面的不同组合。

---

## 4. Schema 为什么能稳定：智能存在于概率分配，不存在于输出类型选择

Jev 的 schema 稳定性不能理解成“模型被训练得非常听话，所以几乎总能按 JSON 返回”。

更准确的理解是：**可能输出的类型与合法取值空间在调用前就已被软件定义。**

对于普通 LLM Structured Outputs，底层仍然是：

~~~
semantic intelligence
    ↓
token generator
    ↓
constrained decoding
    ↓
structured result
~~~

而 Jev 的接口目标更接近：

~~~
semantic intelligence
    ↓
decision distribution
    ↓
typed result
~~~

模型拥有自由度的是：

~~~
run_test 应该是 0.82 还是 0.36？
~~~

而不是：

~~~
我要不要创造一个 schema 中不存在的动作？
~~~

TypeSafe 将 schema matching 描述为保证性质，并把“不会产生类型错误”作为 System One Models 的核心特征。

需要特别区分：这里的强保证是**类型与输出空间层面的保证**，不等价于语义判断永远正确。Jev 完全可能稳定地输出合法 Choice，同时选错合法选项。

---

## 5. 并行到底来自模型还是 GPU

TypeSafe 明确表示：

- 一个请求中的多个 questions 会并行评估；
- 单个 Choice 中的多个 options 同样适合并行；
- 增加 questions 对延迟影响很小，但仍增加 token 和计算成本；
- Jev 使用了新的 model architecture 与 parallel sampler。

但从机器学习系统角度看，“GPU 能并行处理很多独立计算”本身并不是独特能力。现代神经网络训练和推理天然依赖大规模矩阵并行。

Jev 更关键的变化可能是：**它把问题重新表述成一种没有 autoregressive output dependency 的计算。**

传统 LLM：

$$
t_{n+1}\sim P(t_{n+1}\mid t_{\le n})
$$

输出 token 之间存在时间依赖：

~~~
token1
  ↓
token2
  ↓
token3
  ↓
...
~~~

Jev-like 决策模型则可以写成：

$$
z_{j,k}=f_\theta(s,q_j,c_{j,k})
$$

其中：

- j 是 question 维度；
- k 是 candidate 维度。

这些维度天然可以向 GPU 的 batch / tensor 维度展开。

因此更合理的判断是：

> GPU 提供了并行计算的基础能力；Jev 的模型与任务设计消除了大量必须串行生成的输出时间维度，使 GPU 的并行能力真正能够覆盖“智能输出本身”。

真正值得关注的架构问题是：Jev 是否能够对长 state 做共享计算，例如：

$$
H_s=\mathrm{Encode}(s)
$$

只计算一次，然后让大量 question / candidate 基于同一个 $H_s$ 并行评分。

如果做到了这一点，它就不仅是“把许多 classifier call 放进 batch”，而是一种为 massively parallel dynamic decisions 优化过的计算图。

TypeSafe 尚未公开足够细节来确认这一点。

---

## 6. 真正的能力核心：让 fθ(s, q, c) 足够强

如果把 schema、runtime 和 GPU 并行都剥离，Jev 最难复制的部分很可能落在：

$$
\boxed{
f_\theta(s,q,c)
}
$$

这个函数本身。

它必须同时具备：

1. **广泛语义理解**：理解 state、question 和 candidate 的自然语言含义；
2. **组合判断**：处理否定、约束、隐含后果和多条件关系；
3. **比较能力**：候选之间彼此竞争时维持正确排序；
4. **动态 label 理解**：候选不是训练时固定类别，而是运行时用自然语言定义；
5. **不确定性识别**：证据不足、语义模糊或 OOD 时降低置信；
6. **概率校准**：高置信判断在统计上确实更可靠。

一个普通 embedding similarity 模型也可以极快地计算：

$$
f(s,q,c)
$$

但它未必能理解：

> 用户要求修复 bug，同时禁止改变 public API；当前 patch 修好了问题，却改变了返回类型。

这类任务需要接近强 LLM 的语言、知识与组合判断能力。

因此 Jev 真正有价值的目标不是“快速 classifier”，而更接近：

> **frontier-level semantic judgment + classifier-like output constraint + machine-scale parallelism。**

---

## 7. 概率并不要求每个样本存在一个精确的“正确数字”

看到：

$$
p=0.73
$$

很容易产生一个问题：

> 谁能证明正确概率恰好是 0.73？

在大多数现实任务中，这个问题没有单样本答案。

概率的意义通常来自群体统计性质。例如，一个模型长期把某类事件预测为 0.8，如果其中约 80% 最终成立，那么这个模型在该区域是 calibrated 的。

经典概率学习也不要求训练数据提供“正确概率 = 0.73”。如果最终 outcome 是：

$$
y\in\{0,1\}
$$

就可以通过 proper scoring rules，例如 log loss 或 Brier score，让模型在大量样本上自动形成统计意义上的概率。

但对于 Jev 的 System One 定位，还有一种更贴近其产品哲学的训练视角：**概率未必首先被当成一个必须精确命中的点，而可以先被塑造成一个正确的决策方向与合理置信区域。**

这会自然引向在线强化学习。

---

## 8. 一个更有解释力的训练猜想：在线概率塑形

以下部分是本文基于 Jev 公开目标进行的训练方式推演，并非 TypeSafe 已公开的 RLCD 实现。

设：

$$
x=(s,q,c)
$$

Jev 当前输出：

$$
p_t=f_{\theta_t}(x)
$$

强 Teacher、领域专家或 review model 看到当前 state、decision specification、candidate 和 Jev 给出的 $p_t$ 后，不需要给出一个精确的目标概率。

它只需要判断：

~~~
这个概率明显太低
这个概率偏低
这个概率差不多
这个概率偏高
这个概率明显太高
~~~

可以写成：

$$
d_t\in\{-2,-1,0,+1,+2\}
$$

或者更抽象地：

$$
R_T(x,p_t)
$$

Jev 的训练目标是不断提高 Teacher 对当前概率判断的 reward：

$$
\max_\theta
E[R_T(x,f_\theta(x))]
$$

### 8.1 Teacher 不需要知道 p*

假设某个资深工程师无法回答：

> “这个 patch 可以 finish 的正确概率是不是 67.3%？”

但他可能非常容易判断：

~~~
Jev = 0.98
→ 太高，还没有跑 integration tests

Jev = 0.20
→ 太低，主体实现已经完成

Jev = 0.65
→ 大致合理
~~~

这说明 Teacher 未必拥有显式的：

$$
p^*=0.673
$$

但它可能拥有一个稳定的局部方向函数：

$$
D(x,p)\rightarrow\{\uparrow,\approx,\downarrow\}
$$

只要 Teacher 在长期期望意义上能够做到：

$$
p<p^* \Rightarrow E[D]>0
$$

$$
p>p^* \Rightarrow E[D]<0
$$

在线 stochastic updates 就可能把模型推向一个稳定区域。

### 8.2 Region supervision，而不是 point supervision

更自然的目标因此可能不是：

$$
f_\theta(x)=p^*
$$

而是：

$$
\boxed{
f_\theta(x)\in A(x)
}
$$

其中 $A(x)$ 是一个可接受概率区域。

例如：

~~~
0.10 → strongly up
0.35 → up
0.58 → almost
0.66 → almost
0.73 → almost
0.90 → down
0.99 → strongly down
~~~

训练数据从头到尾都不需要写：

~~~
正确答案 = 0.6714
~~~

却已经为模型定义了一个非常明确的 attraction region。

这与真实决策尤其匹配，因为工程系统通常关心：

- 0.9 是否比 0.6 更可信；
- 是否已经越过自动执行阈值；
- 是否应该继续检查；
- 是否应该交给人工 review。

它通常不关心 0.7134 和 0.7141 的差别。

---

## 9. 蒸馏：把强模型或顶尖专家的决策习惯压进 Jev

Jev 的另一个自然训练来源是 knowledge distillation / policy distillation。

经典 knowledge distillation 的基本思想是：用更强、更昂贵的 Teacher 模型产生 richer supervision，再把这种能力压进更小、更便宜的 Student。

Policy Distillation 则进一步证明，可以把一个昂贵强化学习 Agent 的策略压缩到更小、更高效的网络。

把这个思想映射到 Jev：

$$
\pi_T(c\mid s,q)
$$

表示一个强 Teacher 在当前状态与完整决策问题下选择 candidate $c$ 的倾向。

Jev 学习：

$$
\boxed{
\pi_\theta(c\mid s,q)
\approx
\pi_T(c\mid s,q)
}
$$

这里 Teacher 可以是：

- 顶尖 LLM + 高 reasoning effort；
- 多模型 ensemble；
- 领域专家；
- 实际环境 rollout；
- formal verifier；
- 上述几种来源的组合。

### 9.1 Teacher 的“脑子”可以被视为一个环境

这是一个有用的哲学抽象。

顶尖工程师看到：

~~~
repository state
+
user goal
+
current patch
~~~

然后决定：

~~~
inspect
edit
run_tests
finish
~~~

可以把这个过程抽象成：

$$
\pi_H(action\mid state)
$$

这个人脑里未必真的显式存储：

~~~
run_tests = 0.72
finish = 0.11
...
~~~

但如果能从大量相似决策、不同专家、不同 reasoning trajectory 中采样，就可以形成经验 decision distribution。

在线训练进一步允许 Student 直接把自己的 $p_t$ 暴露给 Teacher，让 Teacher 判断“应该更高还是更低”。

于是训练不只是传统离线蒸馏：

~~~
Teacher 给 soft target
Student 拟合
~~~

而更像：

~~~
Student 给当前判断
    ↓
Teacher / environment 评价当前判断
    ↓
reward / direction
    ↓
Student 更新
    ↓
再次交互
~~~

这更接近 preference-based online RL 或 bandit-style feedback。

---

## 10. System 2 → System 1：把昂贵推理变成摊销后的直觉

TypeSafe 使用“System One”这个名称，本身就很适合用 amortized reasoning 理解。

强 Teacher 面对一个问题可能执行：

~~~
理解状态
→ 检查约束
→ 比较候选
→ 推演后果
→ 得出判断
~~~

这是一段昂贵的 System-2 reasoning。

如果训练阶段已经观察过海量这样的决策：

$$
(s_1,q_1)\rightarrow c_1
$$

$$
(s_2,q_2)\rightarrow c_2
$$

$$
...
$$

那么 Student 可以直接学习：

$$
(s,q)\rightarrow decision
$$

推理时不再显式重走完整 reasoning trajectory，而是直接调用已经压进参数的判断函数。

这可以理解成：

$$
\boxed{
\mathrm{System\ 2}
\xrightarrow{\mathrm{distillation}}
\mathrm{System\ 1}
}
$$

或者：

$$
\boxed{
\mathrm{expensive\ reasoning}
\rightarrow
\mathrm{amortized\ decision}
}
$$

资深工程师看到“代码改完但没有跑测试”时，往往可以几乎立即说“先跑测试”。这并不意味着他没有推理能力，而是过去的大量推理已经形成了高度压缩的专业直觉。

Jev 试图工程化的，很可能正是这类“高速智能判断”。

---

## 11. 决策概率与成功概率必须区分

训练 Jev-like 模型时，有两个很容易混淆的概率。

### 11.1 Teacher policy probability

$$
\pi_T(c\mid s,q)
$$

它表示：

> 高质量 Teacher 有多大倾向选择 c。

例如：

~~~
80% 的强工程师会选择 run_tests
~~~

### 11.2 Success probability

$$
Q(s,c)=P(success\mid s,q,c)
$$

它表示：

> 如果真实采取 c，最终成功的概率是多少。

例如：

~~~
采取 run_tests 后，任务最终完成概率是 95%
~~~

两者可以相关，但不等价。

一个更强的数据体系会把它们结合起来：

~~~
Strong Teacher
    ↓
proposes decision
    ↓
real environment executes
    ↓
success / failure / new state
    ↓
feedback corrects teacher signal
~~~

这样训练数据同时包含：

- 强模型或专家的决策知识；
- 环境对该决策真实后果的反馈。

---

## 12. “谁保证 y 正确”：真值必须来自任务结构

训练算法不会凭空制造真理。

如果 supervision 来自一个会犯错的 Judge，模型最多只能逼近这个 Judge 的行为分布。

因此应区分三类任务。

### 12.1 Deterministic / verifiable truth

例如：

- 代码是否编译；
- 测试是否通过；
- 文件是否存在；
- checksum 是否匹配；
- SQL 查询是否满足条件；
- 数学 proof 是否通过 formal checker。

这里可以构造接近确定性的：

$$
y=g(x)\in\{0,1\}
$$

最优策略是尽可能让真实环境或 verifier 产生 reward。

### 12.2 Stochastic truth

例如：

- 明天下雨；
- 用户是否会流失；
- 机器未来一天是否故障。

这里真实答案本身就是概率：

$$
P(y=1\mid x)
$$

calibration 是合理目标。

### 12.3 Semantic / normative judgment

例如：

- 这个回答是否足够有帮助；
- 当前实现是否“优雅”；
- 客服是否应该升级人工；
- 任务是否在语义上已经完成。

这里可能不存在独立于判定协议的唯一真值。

模型更准确地学习：

$$
P(
\mathrm{expert\ protocol\ says\ yes}
\mid x
)
$$

因此，高质量系统不应该把所有判断都交给 probabilistic model，而应该尽可能把可验证部分交给代码与 verifier，只保留真正模糊的语义部分给 Jev-like 模型。

---

## 13. 一个更实用的目标：方向、修正、收敛

从真实决策系统的角度，Jev 最有启发性的思想可能不是“每一步都输出完美概率”，而是：

$$
\boxed{
\mathrm{Direction}
\rightarrow
\mathrm{Correction}
\rightarrow
\mathrm{Convergence}
}
$$

现实世界中的复杂任务很少是：

~~~
initial state
    ↓
一次完美预测
    ↓
final answer
~~~

更多时候是：

~~~
state0
  ↓
decision0
  ↓
action0
  ↓
observe
  ↓
state1
  ↓
decision1
  ↓
action1
  ↓
observe
  ↓
...
~~~

即：

$$
s_t\rightarrow a_t\rightarrow s_{t+1}
$$

在这种闭环里，系统不需要第一次就精确知道整个未来轨迹，它需要的是：

> 当前这一小步足够可能位于正确方向，并且能够快速观察结果、重新判断和持续修正。

这可以把 Jev 理解成一个“智能方向场”。

在每个状态上，它给不同动作或判断分配相对权重：

~~~
这个方向很好
这个方向一般
这个方向风险很高
这个方向证据不足
~~~

Agent 不断跟随局部高质量决策，再从新状态重新采样。

---

## 14. 为什么 Jev 的展示案例都很适合这种哲学

### 14.1 Doom：高频闭环控制

Doom demo 的价值并不在于一次性规划完整游戏，而在于快速重复：

~~~
observe
→ decide
→ act
→ observe
→ decide
→ act
~~~

局部决策可以有误差，因为下一次观察会立即重新修正。

### 14.2 Wikiracing：局部导航

每次只需要回答：

> 当前页面上的哪个真实链接更可能把我带向目标？

可以近似理解成：

$$
Q(s,a)=P(\mathrm{reach\ target}\mid s,a)
$$

选择一个链接后获得新页面，再重新判断。

### 14.3 Smart Home：speculative fan-out

软件可以一次并行询问大量可能有用的问题：

- 用户想控制哪个房间；
- 什么设备；
- 什么动作；
- 是否需要进一步解析；
- 是否属于可直接执行的命令。

代码随后只读取真正需要的判断。

### 14.4 Skill Selection：Harness routing

给定 user request 和大量 Skills：

$$
f(s,q,c_i)
$$

可以被理解为：

> Skill i 对解决当前任务有多相关？

Jev 先做高速筛选，再让主 Agent 只看到更小的相关上下文。

### 14.5 Workflow Evals：把业务 policy 编译成很多窄判断

TypeSafe 的公开 workflow evals 涵盖：

- Security Incidents；
- Agent Trace Observability；
- Invoice Processing；
- Customer Service。

共同模式是：

~~~
structured state
    ↓
many narrow semantic judgments
    ↓
ordinary deterministic code
    ↓
business action
~~~

这比把整个业务政策压成一个 prompt 并要求模型端到端解决，更接近传统软件可控的执行结构。

---

## 15. Jev 对 Agent Harness 的真正启示

在 Codex、Claude Code、OpenCode 等 Agent Harness 中，主模型并不一定应该承担每一个智能判断。

一个完整 Harness 可能包含：

~~~
                         Main LLM
                   planning / coding
                           │
                           ▼
                  Harness / Runtime
                           │
      ┌────────────┬───────┼───────────┬────────────┐
      ▼            ▼       ▼           ▼            ▼
  Tool Router   Skill   Permission   Verifier    Evaluator
               Router      Gate
      │            │       │           │            │
      └────────────┴───────┼───────────┴────────────┘
                           ▼
                     typed decisions
~~~

其中非常多的内部问题都是 System-One-shaped：

- 当前应该加载哪个 Skill？
- 当前 tool call 是否符合用户授权？
- 这一步应该 inspect、edit、test 还是 finish？
- 当前结果是否足够值得进一步验证？
- 这段 agent trace 是否需要人工 review？
- 哪些 context 对当前 turn 真正相关？
- 当前 action 的风险等级是多少？

这些判断未必值得再启动一次完整的长推理生成。

因此 Jev 的潜在价值不是替代主 LLM，而是把 Harness 中大量“智能 if statement”拆出来，形成一个低延迟、高并发、概率可见的 decision layer。

---

## 16. 一个 Jev-like 模型可能怎样准备数据

如果从本文的统一抽象出发，一个可能的数据体系可以分成四类。

### 16.1 Semantic decision data

统一成：

$$
(s,q,c)
$$

其中 q 包含完整问题与 decision space。

目标是让模型学会：

$$
f_\theta(s,q,c)
$$

的基本语义兼容性。

### 16.2 Teacher distillation data

让强模型或专家执行昂贵 reasoning，再收集它们的决策。

可以保留：

- 最终 choice；
- 多次 rollout 的选择频率；
- 多 Teacher disagreement；
- reasoning 产生的外部可验证结果。

### 16.3 Online confidence-shaping data

把当前 Jev probability 暴露给 Teacher：

$$
p_t=f_{\theta_t}(s,q,c)
$$

Teacher 只反馈：

~~~
strongly lower
lower
approximately right
higher
strongly higher
~~~

这类数据训练的是 decision confidence 的方向与区域，而不是单点概率。

### 16.4 Environment / verifier data

凡是可以真实执行或验证的决策，都让环境参与：

~~~
Teacher recommends finish
    ↓
run tests
    ↓
FAIL
    ↓
finish receives negative evidence
~~~

这可以防止 Student 只忠实复制 Teacher 的系统性错误。

---

## 17. 概率质量应该如何评价

如果 Jev-like 模型真的把概率暴露给软件，评测至少需要五个维度。

### 17.1 Decision accuracy / ranking

正确 candidate 是否稳定得到更高分。

### 17.2 Calibration

预测约 0.8 的判断，长期正确率是否也接近 0.8。

可靠性图、ECE、Brier decomposition 等都可以辅助分析。

### 17.3 Candidate perturbation robustness

改变候选顺序、增加 distractor、改写候选描述之后，合理决策是否稳定。

### 17.4 OOD / ambiguity behavior

输入证据不足、候选不完整、任务超出训练分布时，模型是否主动降低置信，而不是继续给出尖锐分布。

### 17.5 Closed-loop outcome

最终最重要的不是单次 classification，而是：

> 把模型放入真实 workflow / Agent Loop 后，任务完成率、事故率、人工升级率、成本和延迟是否改善。

这正是 Jev 更适合用 workflow-level evaluation 而不是传统聊天 benchmark 评价的原因。

---

## 18. Jev 当前最大的未知量

截至 2026 年 9 月 20 日，TypeSafe 已公开：

- Jev 是首个 System One Model；
- 三类 primitive：Noul、Choice、Score；
- 并行 questions / options；
- 新 model architecture；
- parallel sampler；
- RLCD；
- typed outputs 和概率化决策；
- workflow evals 与若干 demo。

但以下关键技术仍未公开：

1. Jev 的基础模型结构；
2. state 是否以及如何共享编码；
3. dynamic candidate scoring 的具体计算图；
4. Choice 使用何种 normalization / sampler；
5. RLCD 的 reward function；
6. RLCD 是否使用在线 Teacher feedback；
7. 是否以及如何使用 LLM / expert distillation；
8. 训练数据来自哪里；
9. calibration 是训练内生得到，还是还叠加了 post-hoc calibration；
10. parallel sampler 到底贡献了多少 latency / throughput 优势。

因此本文关于 $f_\theta(s,q,c)$、在线概率塑形、region supervision 和 System-2 → System-1 蒸馏的部分，应当被理解为**基于公开行为与机器学习原理的架构假设**，而不是对 TypeSafe 内部实现的事实陈述。

---

## 19. 最终理解

如果把 Jev 的产品语言、demo 和 API 全部抽象掉，可以得到一个非常简洁的模型：

$$
\boxed{
f_\theta(s,q,c)\rightarrow p
}
$$

其中：

- state 提供当前世界；
- q 定义完整决策问题和合法决策空间；
- c 是某个唯一 candidate；
- p 表示模型在当前 decision specification 下对 c 的支持程度。

多个 question 和 candidate 可以在 tensor 维度并行展开；Noul、Choice、Score 都可以由这一核心判断能力构造。

真正困难的部分不是“如何让 GPU 同时算很多数字”，而是如何让这个函数拥有：

$$
\boxed{
\mathrm{strong\ semantic\ intelligence}
+
\mathrm{dynamic\ decision\ understanding}
+
\mathrm{uncertainty\ awareness}
+
\mathrm{calibration}
}
$$

从训练哲学上，它又可以进一步理解为：

$$
\boxed{
\mathrm{expensive\ System\ 2\ decisions}
\xrightarrow{\mathrm{distillation\ +\ online\ shaping}}
\mathrm{fast\ System\ 1\ judgments}
}
$$

Teacher 不一定需要知道一个精确的“正确概率”；它只需要长期能够判断当前 decision confidence 应该更高、更低还是已经合理。大量交互可以把这种方向性 supervision 压进参数，使模型第一次 forward 就更容易落在合理区域。

最终，Jev 所代表的思想不是要求一次判断拥有完美的全局答案，而是让软件拥有一种廉价、快速、可组合的局部智能：

$$
\boxed{
\mathrm{Direction}
\rightarrow
\mathrm{Correction}
\rightarrow
\mathrm{Convergence}
}
$$

对于可以持续观察、重新决策和修正的 Agent / workflow，这种智能形式可能比“每一步都重新调用一个完整生成式模型完成端到端推理”更适合成为基础设施。

---

## 参考资料

### TypeSafe / Jev

- TypeSafe AI, **Introducing System One Models & Jev**  
  https://typesafe.ai/blog/introducing-system-one-models-and-jev

- TypeSafe AI, **System One Workflow Evals**  
  https://evals.typesafe.ai/

- TypeSafe AI Docs, **Noul**  
  https://docs.typesafe.ai/primitives/noul

- TypeSafe AI Docs, **Choice**  
  https://docs.typesafe.ai/primitives/choice

- TypeSafe AI Docs, **Score**  
  https://docs.typesafe.ai/primitives/score

- TypeSafe AI Docs, **State**  
  https://docs.typesafe.ai/concepts/state

### Distillation / Calibration

- Geoffrey Hinton, Oriol Vinyals, Jeff Dean, **Distilling the Knowledge in a Neural Network**  
  https://arxiv.org/abs/1503.02531

- Andrei A. Rusu et al., **Policy Distillation**  
  https://arxiv.org/abs/1511.06295

- Chuan Guo et al., **On Calibration of Modern Neural Networks**  
  https://arxiv.org/abs/1706.04599

- scikit-learn, **Probability calibration**  
  https://scikit-learn.org/stable/modules/calibration.html
