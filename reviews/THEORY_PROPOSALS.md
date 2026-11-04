# 理论部分建议：可以在哪里加、加什么、怎么证、怎么验证

**论文**：*Prune the Supervision, Not the Windows: Decision Units for Data Pruning in Time-Series Forecasting*（投 TMLR）
**配套文件**：`FINAL_META_REVIEW.md`（问题编号 S1–S3、A1–A8、#1–#38），`EXPERIMENT_PLAN.md`（实验编号 E0–E17）
**本文目标**：找出能**直接支撑论文主张、并把审稿人的主要批评转化为论文贡献**的理论。每条理论都给出精确的命题、证明思路、它预测什么，以及如何用已有数据（多数零成本）或计划中的实验来验证。

> 记号约定：为避免与预算 $r$ 冲突（这也是审稿意见 C1 提出的问题），下文把残差记为 $e_\omega=\hat y_\omega-y_\omega$，平方残差记为 $e_\omega^2$，**预算仍记为 $r$**。另外，$c(t,d)$ 与 $d$ 无关，记为 $c(t)$。

---

## 0. 结论先行

### 0.1 一句话

这篇论文**最适合加的理论不是泛化界，而是三类"结构性"结果**：
1. **组合结构**：把五种决策单元形式化为损失项集合 $\Omega$ 的**划分（partition）**，由此得到"细化序"、"移除残差质量的单调性"和"附带损伤系数 $\kappa$"；
2. **统计结构**：把掩码看作**固定掩码的截断最小二乘（trimmed LS）**，推导"收益在主体、MSE 动得少、何时 MSE 变差"，以及"掩码与 L1/Huber 各自估计什么"；
3. **线性模型下的精确定理**：DLinear 本身是线性模型，所以可以在线性模型中证明一条 **"单元 × 污染结构 × 预算"律**：单元 $P$ 能中和污染 $A$，当且仅当 $r\ge\kappa_P(A)\cdot\varepsilon$。

这三类结果都可以严格证明，而且都能用论文已有的数据或零成本的分析来检验。

### 0.2 最值得做的五项（按"收益 / 工作量"排序）

| 优先级 | 编号 | 理论 | 放在论文哪里 | 回应的审稿问题 | 工作量 | 验证成本 |
|---|---|---|---|---|---|---|
| ★★★ | **T1** | 单元 = 划分；细化序；移除质量单调性；**附带损伤系数 $\kappa_P$** | 新增 §3.5 | **S2**（残差质量混杂）、A6（单元推荐） | 2–3 天 | 零（用已有得分计算） |
| ★★★ | **T3** | **分位数恒等式**（MAE = ∫q，MSE = ∫q²）；偏态下的位置偏移；low end 近乎无效的上界；**掩码与 L1/Huber 各自估计什么** | 替换 §6 的 "Why MSE moves less"；§5 | **S1**（鲁棒损失）、#11/#33/#36（尾部）、low end 对照 | 约 1 周 | 零（用已有测试误差） |
| ★★★ | **T2** | **残差方差分解**：目标效应 / 上下文效应 / 步长效应；不同单元各自"看到"哪一项 | 新增 §3.6 或 §5 | A3（lead time）、element 与 occurrence 的差距、A6 | 3–5 天 | 零（对已有得分张量做 ANOVA） |
| ★★ | **T5** | **线性模型中的"单元 × 结构 × 预算"律**（旗舰定理） | 新增 §8（主文写定理，附录给证明） | **S2**、S1、A2（预算）、"窗口收益最小" | 2–3 周 | E6（合成注入）+ E3（预算扫描） |
| ★★ | **T4** | **系统抽样理论**：为什么等间距删窗 ≥ 随机 ≥ 按得分选窗 | §7.3 | **S3**（§7.3 证据不足） | 3–5 天 | E7 |
| ★ | T6 | 最优预算 $r^*_P\approx\kappa_P\varepsilon$；Tukey fence 是 $\varepsilon$ 的估计量 | T5 的推论 | A2、自适应预算 | 1–2 天（T5 完成后） | E3 |
| ★ | T7 | 早期得分的学习动力学（线性 GD 闭式解）：早期残差何时能排出"不可解释"的目标 | §7.1 | **A4**（0.998 同义反复） | 1–2 周 | DLinear 闭式 OLS 残差（近乎零成本） |
| ☆ | T8 | 与分类剪枝理论（Sorscher）的统一解释 | Discussion | A7 | 1 天 | E6（无异常的合成数据） |

**最小可行组合**：T1 + T2 + T3。约 2 周，几乎零额外算力，能把 S1、S2 的批评转化为论文的分析贡献。
**推荐组合**：T1–T5。约 5–6 周。理论部分在主文中约占 3–4 页，证明放附录。

### 0.3 这些理论与论文已有数据的初步吻合情况（**只是回看现有数字，待验证，不是结论**）

用表 1（三个骨干平均后的 MAE 变化）回看 T1 和 T2 的预测：

| 数据集 | window | time col. | chan. row | occ. | element | 理论解读（假设） |
|---|---|---|---|---|---|---|
| solar | −2.76 | **−7.83** | −3.41 | **−8.81** | **−8.43** | time column ≈ occurrence ≈ element，远好于 channel row。这符合"**跨通道同时刻**的困难"（各电站共享日出、日落时刻），因为此时 $\kappa_{\text{time}}=1$，而 $\kappa_{\text{chan}}=T$ |
| ETTh2 | −3.84 | −3.31 | **−5.99** | −5.49 | −3.94 | channel row 最好，element 明显更差。这符合"**上下文侧（输入侧）、单通道片段**"的困难：某个窗口某个通道的整条预测都偏 |
| ETTm2 | −4.69 | −5.17 | **−6.51** | −6.26 | −4.94 | 同上 |
| ETTm1 | −0.94 | −2.57 | −2.41 | **−4.57** | −3.73 | occurrence > element > 其他。这符合"**目标侧**困难为主" |
| weather | −1.09 | −2.89 | −2.79 | **−4.21** | −3.23 | 同上 |

- 这个"数据集依赖的单元排序"正是 T1 和 T2 能够解释的结构。论文现在只把它当作噪声，并给出一个"默认用 channel row"的建议，而这个建议恰恰受到 6/8 份评审的批评。
- 只要用零成本的得分张量方差分解（T2）去检验，就能判断这个解释是否成立。如果成立，论文就能把"推荐 channel row"升级为**"先测出困难结构，再选单元"的有理论依据的规则**。

---

## 1. 给 TMLR 论文加理论的原则（先定规矩）

1. **理论必须服务于论文的主张。** TMLR 的第一条标准是 claims 要有证据支撑。装饰性的定理（与实验无关的泛化界）不加分，反而增加被挑错的风险。
2. **每条理论都要给出可证伪的预测，并且用数据验证。** "命题 → 预测 → 在表 X 或实验 EY 中检验"形成闭环，这会显著增强论文的可信度。
3. **假设写在明处，主张不要过度。** 线性模型的结果写成"在线性模型中 / 在程式化模型中"，不外推到深度网络；深度网络部分只写"预测"，然后用实验检验。
4. **优先用简单、精确、可以完整证明的结果**（组合恒等式、分位数恒等式、经典鲁棒统计结论、线性最小二乘），避免松散的上界。
5. **把审稿人的批评变成定理。** 例如"残差质量混杂"（S2）不要回避，而是写成 T1 的引理："在相同项数预算下，更细的单元必然移除更多残差质量"。这是论文的结构性质，承认之后，问题就变成了"收益是否只由残差质量决定"（E4）。
6. **记号统一**：顺带修正审稿意见 C1、C2 指出的 $r$ 符号冲突和 $c(t,d)$ 的写法。

---

## 2. 理论骨架：各部分如何连在一起

```
                 ┌──────────────────────────────────────────────┐
                 │ T1 组合结构：单元 = Ω 的划分                  │
                 │   · 细化序（Hasse 图）                        │
                 │   · 移除质量单调性（引理）                     │
                 │   · 附带损伤系数 κ_P(A)（命题 + 表）           │
                 └───────────────┬──────────────────────────────┘
                                 │ κ 决定"需要多大预算"
        ┌────────────────────────┼─────────────────────────────┐
        ▼                        ▼                             ▼
┌───────────────────┐  ┌──────────────────────────┐  ┌────────────────────────┐
│ T2 残差方差分解     │  │ T5 线性模型定理（旗舰）    │  │ T4 系统抽样（§7.3）      │
│ 目标/上下文/步长    │  │ 单元 P 中和污染 A         │  │ 等间距 ≥ 随机 ≥ 按得分    │
│ → 哪个单元"看见"   │  │ ⇔ r ≥ κ_P(A)·ε           │  │ （Cochran 1946 定理）     │
│   哪种困难          │  │ + 推论 T6：r* ≈ κε        │  └────────────────────────┘
└─────────┬─────────┘  │ + T7：早期得分何时可靠     │
          │            └────────────┬─────────────┘
          ▼                         ▼
┌──────────────────────────────────────────────────────────────┐
│ T3 统计结构：固定掩码 = 截断 LS                                 │
│  · 分位数恒等式 → "收益在主体，MSE 动得少"、solar 上 MSE 反而变差 │
│  · 偏态下的位置偏移 → 什么时候 MSE 会变差                       │
│  · low end 上界 → 为什么 low end 不是有效对照                   │
│  · 掩码与 L1/Huber 各自估计什么 → E1 的理论假设                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. 逐项详细方案

### T1. 单元作为划分：细化序、移除质量单调性、附带损伤系数（★★★）

**放在哪里**：新增 §3.5 "Units as partitions"，约 0.75 页。证明只有几行，可以直接写在正文。

**动机**：
- S2（5 家族中 3–4 个指出的"残差质量混杂"）；
- "window gains least" 需要机制；
- A6（单元推荐缺乏依据）。

**形式化**：

- 损失项集合 $\Omega=\{(i,t,d): i\in[n],\ t\in\mathcal T_i,\ d\in[D]\}$，$|\Omega|=nTD$。
- **决策单元**是 $\Omega$ 的一个划分 $\mathcal P$（五种单元**都是**划分，这一点可以直接验证）：

| 单元 | 块 | 块大小 |
|---|---|---|
| window | $B_i=\{(i,t,d):t\in\mathcal T_i,d\}$ | $TD$ |
| time column | $B_{i,t}=\{(i,t,d):d\}$ | $D$ |
| channel row | $B_{i,d}=\{(i,t,d):t\in\mathcal T_i\}$ | $T$ |
| occurrence | $\{\omega\}$ | 1 |
| element | $B_{t,d}=\{(i,t,d):i\in\mathcal W(t)\}$ | $c(t)$ |

- **细化序**：如果 $\mathcal P$ 的每个块都包含在 $\mathcal Q$ 的某个块中，就说 $\mathcal P$ 细于 $\mathcal Q$，记作 $\mathcal P\preceq\mathcal Q$。于是有：
  - occurrence $\preceq$ time column $\preceq$ window；
  - occurrence $\preceq$ channel row $\preceq$ window；
  - occurrence $\preceq$ element；
  - **element 与 window、time column、channel row 不可比**；
  - **window ∧ element = occurrence**（一个窗口块与一个 element 块的交集恰好是一个损失项）。

  这给出一张 Hasse 图，建议放进图 1。它把论文的核心观点"element 是唯一跨窗口的单元"精确表达为：**element 是 occurrence 之上唯一不在 window 之下的粗化。**

- **可行掩码**：$\mathcal M_{\mathcal P}(b)=\{S\subseteq\Omega:\ S\text{ 是 }\mathcal P\text{ 中若干块的并},\ |S|\le b\}$。

**引理 1（细化单调性）**：若 $\mathcal P\preceq\mathcal Q$，则 $\mathcal M_{\mathcal Q}(b)\subseteq\mathcal M_{\mathcal P}(b)$。于是对任意集合函数 $F$，都有 $\max_{\mathcal M_{\mathcal P}(b)}F\ \ge\ \max_{\mathcal M_{\mathcal Q}(b)}F$。
*证明*：$\mathcal Q$ 的每个块都是若干 $\mathcal P$ 块的并。∎

**引理 2（论文的打分规则在质量意义下是最优的）**：取 $F(S)=\sum_{\omega\in S}e_\omega^2$（移除的残差质量）。如果 $\mathcal P$ 的所有块等大（window、time column、channel row、occurrence 都满足），那么"按块内均值取前 $\lfloor b/m\rfloor$ 个块"就最大化了 $F$。对 element 单元，块大小只在序列两端不等，误差不超过一个块的质量。
*证明*：块数固定、块等大时，最大化块和之和等价于取块和最大的那些块，也就是块均值最大的那些块。∎

**推论（把 S2 写成定理）**：在相同项数预算下，论文实际移除的残差质量满足
$$M_{\text{occ}}\ \ge\ M_{\text{time}},\ M_{\text{chan}}\ \ge\ M_{\text{win}},\qquad M_{\text{occ}}\ \gtrsim\ M_{\text{elem}}.$$
**理论只给出偏序，element 与 time column、channel row 之间的先后由数据决定。**
- 这与表 1 的均值排序一致：occurrence 最大，window 最小。
- 但逐数据集来看，收益并不总是服从质量排序。例如 ETTh2 上 time column 的 −3.31 小于 window 的 −3.84。这说明"收益 ≠ 质量"，正好为 E4（按质量匹配预算）提供了动机。

**命题 3（附带损伤系数 $\kappa$）**：对一组目标 $A$（element 的集合），记它的全部 occurrence 为 $\text{occ}(A)$。定义
$$\kappa_{\mathcal P}(A)=\frac{\big|\bigcup\{B\in\mathcal P: B\cap \text{occ}(A)\neq\emptyset\}\big|}{|\text{occ}(A)|}$$
也就是用单元 $\mathcal P$ **完全移除** $A$ 的监督时，被一起屏蔽的项数与目标项数之比（≥1，等于 1 表示没有附带损伤）。对内部的 $A$，可以精确算出：

| 困难或污染结构 $A$ | occurrence | element | time column | channel row | window |
|---|---|---|---|---|---|
| 单个点 $(t,d)$ | 1 | 1 | $D$ | $T$ | $TD$ |
| 跨通道同时刻事件 $\{t\}\times[D]$ | 1 | 1 | **1** | $T$ | $T$ |
| 单通道长度 $\ell$ 的片段 | 1 | 1 | $D$ | $\frac{\ell+T-1}{\ell}\to 1$ | $\frac{(\ell+T-1)D}{\ell}$ |
| 全通道长度 $\ell$ 的片段（regime） | 1 | 1 | **1** | $\frac{\ell+T-1}{\ell}$ | $\frac{\ell+T-1}{\ell}\to1$ |

*证明*：直接计数。以单点为例，覆盖它的窗口有 $T$ 个，对应 $T$ 个 time column（各 $D$ 项）、$T$ 个 channel row（各 $T$ 项），或 $T$ 个窗口（各 $TD$ 项）。∎

**含义**：
1. 对单点困难，窗口的附带损伤系数是 $TD$（ETT、T=96 时为 672；ECL 时为 30816），这就是"window gains least"的定量机制。
2. **没有一个单元在所有结构上都最优**（除了 occurrence 和 element，它们对目标侧结构总是 1）。最优单元取决于**困难的结构**。
3. **对齐指数**（零成本诊断）：取早期得分最高的 r% 个 element 作为 $A$，对每个单元算 $\kappa_{\mathcal P}(A)$。预测是：在 7 个数据集上，$\kappa$ 的单元排序与收益的单元排序呈正相关（Spearman）。可以先用 §0.3 中 solar、ETTh2 的模式做定性检验。

**验证（零成本）**：
- 每个数据集、每个单元的移除质量（E0.4）；
- $\kappa_{\mathcal P}(A_r)$；
- "收益 vs 质量"散点图和"收益 vs $\kappa$"秩相关。

**风险**：$\kappa$ 只刻画"完全移除某组目标的代价"，没有刻画"移除后收益多大"。所以它解释的是**可达性**，不是**效果**，要配合 T2 和 T5。

---

### T2. 残差方差分解：不同单元各自"看见"哪种困难（★★★）

**放在哪里**：新增 §3.6 或 §5 的一个段落 "What each unit sees"，约 0.5–0.75 页。

**动机**：
- A3（lead time 混杂，4/4 家族）；
- element 比 occurrence 差约 0.91 点 MAE（论文 §5 只给了一句话的解释）；
- A6（单元推荐）；
- Fable·high 提出的"值不值得学取决于上下文"。

**模型**：对平方残差张量（打分时刻）做可加分解：
$$e^2_i(t,d)=\mu+\underbrace{\alpha(t,d)}_{\text{目标效应}}+\underbrace{\beta(i,d)}_{\text{上下文效应（窗口×通道）}}+\underbrace{\gamma(i,t)}_{\text{窗口×时刻效应}}+\underbrace{\phi(t-i-L)}_{\text{步长效应}}+\eta_i(t,d)$$

- **目标效应 $\alpha$**：这个目标值本身难（异常值、尖峰、不可预测）。它出现在覆盖该目标的全部 $c(t)$ 个窗口中。
- **上下文效应 $\beta$**：窗口 $i$ 在通道 $d$ 上的**输入**异常或处于 regime 转换处，导致整条预测都偏。它出现在该行的全部 $T$ 个步长上。
- **窗口×时刻效应 $\gamma$**：某个窗口在某个时刻的跨通道联合偏差。
- **步长效应 $\phi$**：远步长的残差系统性更大。

**命题 4（各单元得分是哪些效应的平均）**：在上述可加模型下，各单元的块均值得分满足（内部块；$\bar\cdot$ 表示在块内平均）：

| 单元 | 得分 ≈ | 主要"看见" |
|---|---|---|
| element $(t,d)$ | $\alpha(t,d)+\overline{\beta}_{i\in\mathcal W(t)}+\overline{\gamma}+\bar\phi$ | **$\alpha$**（上下文和步长效应被 $T$ 个窗口平均掉了） |
| channel row $(i,d)$ | $\beta(i,d)+\overline{\alpha}_{t\in\mathcal T_i}+\overline{\gamma}+\bar\phi$ | **$\beta$**，加上时间平滑后的 $\alpha$ |
| time column $(i,t)$ | $\gamma(i,t)+\phi(t-i-L)+\overline{\alpha(t,\cdot)}+\overline{\beta(i,\cdot)}$ | **$\gamma$、$\phi$**，以及跨通道平均后的 $\alpha$ |
| window $i$ | $\overline{\beta(i,\cdot)}+\overline{\alpha}+\overline{\gamma}+\bar\phi$ | 被稀释得最严重的混合 |
| occurrence | 全部 | 全部（包括 $\phi$） |

*证明*：对可加模型在块内取平均，与块正交的效应会被平均成常数或平滑项。∎

**由此得到的可检验预测**：

1. **element 与 occurrence 的差距由 $\beta$、$\gamma$、$\phi$ 的方差份额决定。** 若 $\mathrm{Var}(\alpha)$ 占主导（组内相关系数 ICC 接近 1），两者几乎相同；若上下文或步长效应大，occurrence 会明显更好。这为 §5 的 "Deciding per target gives up part of the gain" 给出了定量机制。
2. **lead time 混杂的精确形式**：occurrence 的 top-r 在步长 $k$ 处被屏蔽的比例是 $P(\alpha+\beta+\gamma+\phi(k)+\eta>\tau)$，随 $\phi(k)$ 单调增加。如果 $\mathrm{Var}(\phi)$ 相对其他效应不可忽略，那么 occurrence 掩码就**部分等价于按步长重新加权**（Hounie et al., 2024）。E5.1（按步长设预算上限的掩码）正好把这一部分剥离出来。
3. **数据集依赖的单元排序**（见 §0.3）：
   - 目标侧（$\alpha$）占主导：element ≈ occurrence；
   - 上下文侧（$\beta$）占主导：channel row ≈ occurrence，element 落后（ETTh2、ETTm2 的模式）；
   - 跨通道同时刻（$\alpha$ 在 $d$ 方向高度相关，或 $\gamma$ 大）：time column ≈ occurrence（solar 的模式）。
4. **一个点异常有"双重足迹"**（这是很少被注意到的 TS 特有机制）：
   - 被污染的 $y_{t,d}$ 作为**目标**，产生大小约为 $M$ 的 $\alpha$ 足迹（$T$ 项，在 $(i,t)$ 平面上是一条对角带）；
   - 它作为**输入**进入 $L$ 个窗口，又产生大小约为 $M|W_{\cdot j}|$ 的 $\beta$ 足迹（$L\times T$ 项）。
   - element 只能处理目标足迹，channel row 和 window 能处理输入足迹，occurrence 两者都能处理。**这可能正是 occurrence 平均最好的原因。**

**验证（零成本）**：
- 对每个数据集、骨干和 horizon 的得分张量做交叉随机效应方差分量估计（Searle et al., 1992），或者更简单的逐因子平方和分解；
- 报告 $\alpha,\beta,\gamma,\phi$ 的方差份额；
- 检验份额与"各单元相对 occurrence 的差距"在数据集间的秩相关。

**风险**：可加模型是近似的，因为平方残差有交互项。所以建议把它写成**描述性分解**，而不是因果模型；份额的估计受序列两端影响，只用内部块。

**可能衍生的方法**（可选，审慎使用）：**按 T2 份额自动选单元**，或者使用"element ∪ channel row"的混合掩码，同时覆盖目标侧和上下文侧。这会扩大论文范围。如果要做，应在留出的数据集上验证，而不是在同一批数据上挑选。

---

### T3. 统计结构：固定掩码 = 截断最小二乘（★★★）

**放在哪里**：
- **替换** §6 的 "Why MSE moves less"（审稿人认为它用 RMSE 论证 MSE，属于代数关系，C16）；
- §5 加入 low end 命题；
- §2 或新的 §5 段落加入"与 L1/Huber 的关系"。

总计约 1–1.25 页。

**动机**：
- **S1**（鲁棒损失替代解释，8/8 份评审）；
- #11、#33、#36（尾部"无变化"、ΔMSE 分解）；
- low end 作为对照是否有效（SH、FM、FH）。

#### T3a. 分位数恒等式：收益为什么在主体、MSE 为什么动得少（**精确，无近似**）

设测试绝对误差 $|e|$ 的分位函数为 $q(u)$，$u\in(0,1)$。那么
$$\text{MAE}=\int_0^1 q(u)\,du,\qquad \text{MSE}=\int_0^1 q(u)^2\,du .$$
定义逐分位的相对改善 $\delta(u)=1-q'(u)/q(u)$（$q'$ 为掩码后的分位函数）。则**精确地**有
$$\frac{\Delta\text{MAE}}{\text{MAE}}=-\,\mathbb E_{w_1}[\delta],\quad w_1(u)\propto q(u);\qquad
\frac{\Delta\text{MSE}}{\text{MSE}}=-\,\mathbb E_{w_2}[2\delta-\delta^2],\quad w_2(u)\propto q(u)^2 .$$

**命题 5**：若 $\delta(u)$ 关于 $u$ 不增（"收益在主体"），则由于 $w_2$ 相对 $w_1$ 是似然比递增的重加权（$w_2/w_1\propto q$ 递增），有 $\mathbb E_{w_2}[\delta]\le\mathbb E_{w_1}[\delta]$（Chebyshev 关联不等式）。也就是说，**MSE 天然把权重放在收益为零的尾部分位上**。尾部只要有少量变差（$\delta<0$），就可能让 MSE 上升而 MAE 下降。
- 这精确解释了 solar 上 occurrence 的 MAE −8.81% 而 MSE +0.91% 的现象：其 CVaR 为 +0.47，说明尾部略微变差。

**价值**：
- 这是**恒等式，不需要任何模型假设**，审稿人无从反驳；
- 它把"收益在主体"（表 2、图 2a）与"MSE 动得少、甚至变差"联系成**同一个现象**；
- 它直接给出 Haiku 审稿人要求的"按分位区间分解 ΔMSE"（#36）。

**验证（零成本）**：
- 用已保存的测试误差（或重新推理一次）估计 $q,q'$；
- 画出各分位区间对 ΔMAE 和 ΔMSE 的贡献；
- 用恒等式从分位剖面"预测"表 2 的 MAE 和 MSE 变化，并与实测对比（理论上应完全吻合，这同时也是对实现的检查）。

#### T3b. 偏态下的位置偏移：什么时候 MSE 会变差

**命题 6（总体层面）**：设某目标的条件分布 $F$ 右偏（中位数 < 均值），掩码在总体层面去掉其上尾 $R=\{Y>F^{-1}(1-r)\}$。截断后的最小二乘解是截断均值 $m_r=\mathbb E[Y\mid Y\notin R]<\mathbb E[Y]$。在未截断的测试分布上：
- **MSE 的超额 = $(\mathbb E Y-m_r)^2>0$**；
- 只要 $m_r$ 仍不低于中位数，**MAE 就会下降**，因为 $m\mapsto\mathbb E|Y-m|$ 是凸函数，最小值在中位数处。

*证明*：最小二乘解是条件均值；MSE 的分解是偏差平方加方差；MAE 关于 $m$ 是凸的。∎

**解读**：掩码对 MSE 有**两条通道**：
1. **影响降低 / 方差通道**：在重尾或被污染的噪声下，降低估计方差，**使 MSE 下降**；
2. **位置偏移通道**：只在偏态下出现，**使 MSE 上升**。

净效应的符号取决于偏态强度。
- **预测**：MSE 上升而 MAE 下降的数据集（solar，以及 ECL 上的细单元），其训练残差应该是明显偏态的。solar 有大量零值且右偏，与预测一致。
- **验证（零成本）**：报告每个数据集训练残差的偏度，以及它与"MSE 变化 − MAE 变化"的相关。

#### T3c. low end 近乎无效的上界（主动回应"low end 不是有效对照"）

记全梯度 $\nabla\mathcal L=\frac{2}{|\Omega|}\sum_\omega e_\omega\nabla f_\omega$，掩码 $S$（$|S|=r|\Omega|$）后按式 (4) 重新归一化的梯度为 $\nabla\mathcal L_S$。则
$$\nabla\mathcal L_S-\nabla\mathcal L=\underbrace{\tfrac{r}{1-r}\nabla\mathcal L}_{\text{对所有单元和两端都相同的统一缩放}}-\ \tfrac{2}{(1-r)|\Omega|}\sum_{\omega\in S}e_\omega\nabla f_\omega ,$$
$$\Big\|\tfrac{2}{(1-r)|\Omega|}\sum_{\omega\in S}e_\omega\nabla f_\omega\Big\|\ \le\ \tfrac{2rG}{1-r}\cdot\overline{|e|}_S,\qquad G=\max_\omega\|\nabla f_\omega\|.$$

**命题 7**：
1. 式 (4) 的重新归一化对所有单元和两端**完全相同**（等价于把学习率乘以 $1/(1-r)$），因此**不可能**解释单元之间的差异。这顺带排除了一个审稿人可能提出的机械性混杂。
2. low end 的"选择性"扰动受 $\overline{|e|}_{S_{\text{low}}}\le\sqrt{q_r(e^2)}$ 控制，相比 high end 的 $\overline{|e|}_{S_{\text{high}}}$ 小几个数量级。**所以 low end 在构造上近乎无效**，它的零结果并不能证明"收益来自屏蔽了哪些项"，**必须用随机对照**（E2）。论文应主动写出这一点，而不是等审稿人指出。

**验证（零成本）**：报告 high、low、random 三种掩码移除的 $\sum|e|$ 和 $\sum e^2$ 占比（E0.4），以及比值 $\overline{|e|}_{\text{high}}/\overline{|e|}_{\text{low}}$。

#### T3d. 掩码与 L1/Huber 各自估计什么（为 E1 提供理论假设，**可能成为论文最强的卖点**）

**模型**：$Y=\mu(x)+\xi+a$。其中 $\xi$ 是干净噪声，$a$ 是稀疏污染，$P(a\neq0)=\varepsilon$，幅度大。

**命题 8**（经典结果的组合：Huber, 1964；Koenker, 2005）：
1. 掩码恰好覆盖污染支撑时（理想掩码），**掩码后的 LS 估计的是干净条件均值** $\mu(x)+\mathbb E\xi$，渐近方差约为 $\mathrm{Var}(\xi)/(1-\varepsilon)$。
2. **L1（LAD）估计的是被污染后的条件中位数**，约为 $\mu(x)+\mathrm{med}(\xi)+O(\varepsilon)$。它对污染稳健，但目标是中位数；渐近方差约为 $1/(4f(\mathrm{med})^2)$。
3. 若 $\xi$ 是高斯噪声，则 L1 的方差是 LS 的 $\pi/2\approx1.57$ 倍（中位数的相对效率是 $2/\pi$）。

**推论（E1 的理论预测）**：
- **干净噪声近似对称、近似高斯，且污染稀疏而大时**：掩码准确的话，**MSE 和 MAE 都应优于 L1**。这是决策树里的"情况 A"，也是论文的最佳叙事。
- **干净噪声偏态时**：L1 在 MAE 上有结构性优势，但 MSE 会变差（因为位置偏移到中位数）；掩码保留均值目标，MSE 更好。这是"情况 B"。
- Huber 介于两者之间。

所以论文可以**先写出理论预测，再用 E1 检验**，而不是被动地补一个基线。审稿人会把这看作理论与实验的闭环。

**风险**：真实掩码不是理想掩码，它由早期得分决定，可能误删干净项或漏掉污染。所以应写成"理想掩码下的结论 + 掩码准确性条件见 T5、T7"。

---

### T4. 系统抽样：为什么等间距删窗有效，而按得分选窗不行（★★）

**放在哪里**：§7.3，约 0.5 页。这也正是 S3 要求补证据的地方。

**设定**：逐窗口损失序列 $\ell_i=\frac1{TD}\sum_{t\in\mathcal T_i,d}e^2(t,d)$。在"目标效应为主、且在时间上近似白噪声"的程式化模型下（$e^2(t,d)\approx\alpha(t,d)$），$\ell_i$ 是长度为 $T$ 的白噪声滑动平均，所以
$$\mathrm{Corr}(\ell_i,\ell_{i+k})=\rho_k=(1-|k|/T)_+ .$$
这个 $\rho_k$ **非负、递减、凸**（它是两个仿射函数的最大值）。

**定理（Cochran, 1946；Cochran, 1977, Ch. 8）**：对自相关满足 $\rho_k\ge\rho_{k+1}\ge0$ 且二阶差分 $\rho_{k+1}+\rho_{k-1}-2\rho_k\ge0$ 的总体，在期望意义下，**系统抽样（等间距，随机起点）的方差 ≤ 分层随机抽样 ≤ 简单随机抽样**。

**命题 9（应用到删窗）**：
1. 以"剪枝后目标函数与全数据目标函数之差的期望平方"作为度量，**等间距删窗 ≤ 随机删窗**。这为 §7.3 中"等间距在 50%、70% 时领先（p=0.04、0.006）"提供了理论解释，而且预测保留率越低，优势越明显。
2. **覆盖引理（精确）**：步长为 $s\le T$ 的等间距保留下，每个内部 element 恰好被 $\lfloor T/s\rfloor$ 或 $\lceil T/s\rceil$ 个窗口监督，覆盖方差为 0，并且每个 element 在 $T/s$ 个步长位置（模 $s$ 的一个剩余类）上被监督。随机保留比例 $f$ 时，覆盖服从超几何分布，方差约为 $Tf(1-f)$。**移除一段长度 $\ell\ge T$ 的连续窗口**，则会让 $\ell-T+1$ 个 element 完全失去监督。
3. **按得分选窗**：由于 $\rho_k$ 很高，得分最高的窗口会聚成长度约为 $\Theta(T)$ 的连续段，相当于**整群抽样**，在正自相关下是效率最低的设计。这解释了 §7.3 的"三条按得分的规则都不优于随机"，并与 Fable·high 的"窗口掩码相当于整段删除"相互印证。

**预测**：
- 等间距相对随机的优势随 $T$（重叠程度）增大而增大；当 $s>T$（目标不再重叠）时，优势只剩下序列本身的自相关部分。
- 按得分选窗形成的连续段长度分布与 $T$ 成正比（E0.5⑤ 可以直接验证）。

**验证**：E7（删窗率 × 规则 × horizon）以及 E0.5⑤（游程长度）。

---

### T5. 线性模型中的"单元 × 污染结构 × 预算"律（★★，旗舰定理）

**放在哪里**：新增 §8 "A linear analysis"（或放在 §5 之后）。主文约 1–1.5 页，写定理陈述和推论；附录 E 给出完整证明。

**为什么可行**：DLinear 是**线性**模型（趋势和季节分量各接一个线性层），所以这里的结论对三个骨干之一是**精确**的，对另外两个是"程式化预测"。

**设定**（为清晰起见写单通道，多通道与通道独立的 DLinear 同理）：
- 对每个步长 $k\in[T]$，有一个独立的最小二乘问题：行 $i$ 的特征是 $x_i\in\mathbb R^{L}$（输入窗口），目标是 $y_{t(i,k)}$。
- **污染**：$y_t=y^*_t+a_t$，其中 $a_t=M u_t$（$|u_t|=1$）当 $t\in A$，否则为 0。被污染的项占全部损失项的比例为 $\varepsilon$。
- **假设 A1（理想化）**：污染只影响**监督角色**，输入不受影响。附注里讨论去掉这条假设后的情况（见下）。
- **假设 A2（打分时刻有界）**：打分时的预测满足 $|\hat y_\omega-y^*_\omega|\le B$ 对所有 $\omega$ 成立。
- **假设 A3（非退化）**：未被屏蔽的污染行满足 $X^\top a\neq0$（一般位置条件）。

**定理 10**：设单元 $\mathcal P$、预算 $r$，掩码按 §3.3 的块均值规则选取。若
$$M>B\,(1+\sqrt{m_{\max}})\quad(\text{分离条件，}m_{\max}\text{ 为最大块大小}),$$
则当 $M\to\infty$ 时：
$$\text{掩码估计的超额风险有界（与 }M\text{ 无关）}\iff r\ \ge\ \kappa_{\mathcal P}(A)\cdot\varepsilon ;$$
否则超额风险以 $\Theta(M^2)$ 发散。
- 全数据 LS（不掩码）的超额风险总是 $\Theta(M^2)$。
- 满足条件时，掩码后的估计恰好等于"在干净行上的 LS"，只付出约 $1/(1-r)$ 的方差代价。

*证明思路*：
1. **分离**：在 A2 下，含有污染项的块的均值至少是 $(M-B)^2/m$，干净块的均值至多是 $B^2$。在分离条件下，所有含污染的块都排在干净块之前。
2. **覆盖**：由 $\kappa$ 的定义，含污染的块共有 $\kappa_{\mathcal P}(A)\cdot\varepsilon|\Omega|$ 项。预算足够时全部被屏蔽，剩下的就是干净行上的 LS，与 $M$ 无关；预算不足时，至少有 $1-\frac{r}{\kappa\varepsilon}$ 比例的污染项保留下来。
3. **偏差**：LS 的偏差为 $(X^\top X)^{-1}X^\top a_{\text{remain}}$，在 A3 下关于 $M$ 线性增长。∎

**推论 11（点异常，各单元需要的预算）**：对间隔至少为 $T$ 的点异常，代入 T1 的表格得到：

| 单元 | 中和所需预算 | 例：$\varepsilon=0.5\%$，ETT（$D=7$，$T=96$） |
|---|---|---|
| occurrence、element | $\varepsilon$ | 0.5% ✓ |
| time column | $D\varepsilon$ | 3.5% ✓ |
| channel row | $T\varepsilon$ | 48% ✗（在 r=10% 下不够） |
| window | $TD\varepsilon$ | 336% ✗（**在任何预算下都不可能**） |

这给出 "window gains least" 的**定量律**。同时，它预测 E3 的预算曲线会出现**单元依赖的饱和点**：细单元在 $r\approx\varepsilon$ 时饱和，window 一直在上升。

**推论 12（同一定理，不同结构）**：
- 跨通道同时刻事件：time column 与 occurrence 等价（$\kappa=1$）。
- 单通道长片段：channel row 接近最优。

这与 T2 的方差分解和 §0.3 中 solar、ETTh2 的模式**是同一件事的两面**：T2 是统计描述，T5 是机制和风险律。

**附注（去掉 A1：输入侧污染）**：被污染的 $y_t$ 还作为输入进入 $L$ 个窗口，带来变量含误差（errors-in-variables）偏差，导致系数衰减（Fuller, 1987）。
- **element 掩码不能消除这部分偏差**，因为被屏蔽的目标仍然作为输入出现（§3.4 原文："a masked element still enters up to L windows as input"）。
- 屏蔽**输入受污染的行**（channel row 或 window 单元能捕捉，对应 T2 中的 $\beta$ 足迹）才能消除它。

这给出两个推论：
1. occurrence（同时覆盖两种足迹）平均最好；
2. "屏蔽 vs 插值修复"的比较之所以不干净（A5），正是因为插值同时修复了输入侧，而屏蔽只处理目标侧。E10 的 2×2 分解有了理论依据。

**验证**：
- **E6（合成注入）是这条定理的直接实验**：分别注入点、跨通道、单通道片段、regime 四类结构，测量各单元的饱和预算，并与 $\kappa\varepsilon$ 比较。
- E3（预算扫描）检验真实数据上的饱和点。

**风险与诚实声明**：
- 重叠窗口使各行相关，这在固定设计下不影响偏差项，但方差项需要用有效样本量来讨论（写进附注即可）；
- 真实数据中的"困难"不一定是加性污染（可能是难但可学的信号），这时定理不适用，见 T8；
- 深度网络部分只写"预测"，交给实验检验。

---

### T6. 最优预算与 Tukey 自适应预算（★，T5 的推论）

**命题 13**：在 T5 的模型下，单元 $\mathcal P$ 的风险关于 $r$ 的形状是：
- $r<\kappa_{\mathcal P}\varepsilon$ 时，偏差项约为 $M^2\big(1-\frac{r}{\kappa\varepsilon}\big)^2\cdot c_X$；
- $r\ge\kappa\varepsilon$ 时，只剩方差项，约为 $\sigma^2 p/((1-r)n_{\text{eff}})$，外加误删干净的高杠杆行带来的信息损失。

所以 $r^*_{\mathcal P}\approx\kappa_{\mathcal P}(A)\varepsilon$，超过之后风险缓慢上升。

**推论**：对 element 单元（$\kappa=1$），**Tukey 上界（Q3 + 1.5 IQR）就是污染比例 $\varepsilon$ 的一个估计量**（在污染与干净残差分离时）。这为 §3.3 和 §7.3 中的自适应预算提供了理论依据。目前论文只是"用了"这个预算，没有解释为什么。

**预测**：
- E3 中 high end 曲线先升后平或缓降，拐点在 $\kappa\varepsilon$ 附近；
- **low end 在较大预算（如 40%）下会变成有害**，因为删掉的是干净的信息项。这是可以证伪的。

---

### T7. 早期得分的学习动力学：早期残差何时能排出"不可解释"的目标（★）

**放在哪里**：§7.1。替换或补充 Spearman 0.998 的论证（A4：8/8 份评审都认为它近乎同义反复）。

**设定**：线性最小二乘，从 0 初始化做梯度下降，步长 $\eta$。设 Gram 矩阵 $\frac1nXX^\top=\sum_j\lambda_ju_ju_j^\top$。第 $k$ 步时训练残差为
$$e_k=\sum_{j:\lambda_j>0}(1-\eta\lambda_j)^k\langle u_j,y\rangle u_j\ +\ P_\perp y .$$
- **收敛残差 $P_\perp y$** 是该模型类**无论如何也无法用输入线性解释**的部分。在 DLinear 中，这正是论文所说"the error that the rest of the series leaves unexplained"的精确对应。
- **早期残差** = 不可解释部分 + **尚未学到的慢方向**（$\lambda_j$ 小的方向）。

**命题 14**：
$$\|e_k-P_\perp y\|_\infty\le(1-\eta\lambda_{\min}^+)^k\,\|P_\parallel y\|_2 .$$
若 $P_\perp y$ 在第 $\lfloor r|\Omega|\rfloor$ 名与下一名之间的间隔大于 $2(1-\eta\lambda_{\min}^+)^k\|P_\parallel y\|_2$，那么第 $k$ 步早期得分的 top-r 集合与"不可解释残差"的 top-r 集合**完全相同**。

*证明*：在特征基下展开，然后用一致上界加上排序稳定性。∎

**含义**：
1. 孤立尖峰几乎完全落在 $P_\perp$ 中（它不是过去输入的线性函数），因此**很早就被识别出来**；而"难但可学"的慢信号会在早期混入得分，这正是 RHO-Loss 要区分的情形（Mindermann et al., 2022）。
2. 线性模型（$p=L\ll n$）不会插值，所以 $P_\perp y\ne0$，**晚期打分仍然有效**。高容量网络最终会记住污染，晚期残差失去分离度（Arpit et al., 2017；Liu et al., 2020），**所以需要早期打分**。

   **预测**：打分时刻敏感性（E8）在 DLinear 上很小，在 iTransformer、TimeMixer 上较大。这也解释了表 3 中"整个训练过程的平均损失"不如早期得分。
3. **零成本的非同义反复验证**：对 DLinear，可以**闭式求解**每个数据集、每个 horizon 的 OLS，得到精确的 $P_\perp y$，再与 10% 时刻的得分计算 Spearman 相关。这避免了 §7.1 那种"两个早期量相关"的循环论证。

**附带解释（为什么 DLinear 收益最大）**：线性 LS 的影响函数无界，而早停的过参数化网络对标签噪声有一定的隐式鲁棒性（Li, Soltanolkotabi & Oymak, 2020）。所以**理论预测线性骨干从掩码中获益最多**，与表 5（DLinear 的 MAE 均值 −3.57 至 −6.77，三个骨干中最大）一致。这一点作为"假设"写，不要写成"定理"。

---

### T8. 与分类剪枝理论的统一（☆，写进 Discussion）

- Sorscher et al. (2022) 的师生模型说明：在**无噪声**且数据充足时应保留难样本。本文的结论是去掉最难的项。
- 统一解释：在 T5 的模型中，**当 $\varepsilon=0$（没有污染）时**，屏蔽 high end 只会移除干净且高杠杆的信息行，增加方差，**理论预测不会有收益，甚至有害**；**当 $\varepsilon>0$ 时**，难项主要是污染，去掉它们是最优的。
- 所以"极性"不取决于任务是分类还是回归，而取决于**难度的来源**（信息还是污染）。这把审稿意见 A7（"opposite end"的叙述有误导性）转化为一个可检验的预测：**在没有注入异常的干净合成数据上，high end 掩码应该没有收益**（E6 中加一个 $\varepsilon=0$ 的对照，成本几乎为零）。

---

## 4. 不建议加的理论（以及原因）

| 方向 | 不建议的原因 |
|---|---|
| 依赖数据下的泛化界（β-mixing Rademacher 等） | 结果很松，与论文主张（单元比较、收益分布）关系弱；证明复杂，容易被挑错；TMLR 审稿人会质疑"这个界说明了什么" |
| 深度网络的 NTK 或平均场分析 | 工作量巨大，与 iTransformer、TimeMixer 的实际训练相差太远，可信度低 |
| 证明"早期得分最优"或"某单元最优" | 不成立（T1 表明不存在对所有结构都最优的单元），这类主张会被反例推翻 |
| 信息论界（互信息 / 数据处理不等式） | 难以与具体指标（MAE 分位数）联系起来，预测力弱 |
| 子模优化框架下的"最优掩码" | 移除质量是**模块化**函数，贪心已经最优（引理 2），子模框架是多余的。只有加入"覆盖约束"时才有意义，可以作为未来工作 |
| 统计推断方法本身的理论（检验的有效性） | 属于方法学，放附录说明假设即可，不算理论贡献 |

---

## 5. 放在论文哪里：修改后的结构建议

```
§3 Pruning the supervision
   3.1–3.4（原有，改记号：残差记为 e，预算仍记为 r，c(t)）
   3.5 Units as partitions               ← T1（引理 1、2，命题 3 + κ 表 + Hasse 图）
   3.6 What each unit sees               ← T2（方差分解，命题 4）
§4 Experimental protocol（原有）
§5 Which end to prune, and at which unit
   + "Why the low end is inert"          ← T3c（命题 7）→ 引出随机对照（E2）
   + "Is it just a robust loss?"         ← T3d（命题 8）+ E1 结果
   + "Gain vs removed mass and κ"        ← T1 的推论 + E4
§6 Where the gain lives
   "Why MSE moves less" 改写为           ← T3a（分位数恒等式，命题 5）+ T3b（偏态，命题 6）
§7 A closer look
   7.1 + 早期得分的动力学                 ← T7（命题 14；DLinear 闭式验证）
   7.3 + 系统抽样                         ← T4（命题 9）
§8 A linear analysis（新增）              ← T5（定理 10，推论 11、12）+ T6（命题 13）
§9 Discussion                            ← T8（与分类剪枝的统一）
附录 E 证明；附录 F 理论预测的经验检验汇总
```

**篇幅**：主文新增约 3–4 页（TMLR 没有硬性页数限制，但应保持可读），附录新增约 4–6 页证明。

---

## 6. 理论给出的可证伪预测（建议在论文中集中列成一张表）

| # | 预测 | 来源 | 检验方式 | 成本 |
|---|---|---|---|---|
| P1 | 移除的残差质量满足细化偏序：occ ≥ time col、chan row ≥ window | T1 引理 1、2 | E0.4 | 零 |
| P2 | 各单元的收益排序与 $\kappa_{\mathcal P}(A_r)$ 的排序在数据集间正相关 | T1 命题 3 | E0.4 + 表 1 | 零 |
| P3 | element 与 occurrence 的差距随 $\beta,\gamma,\phi$ 的方差份额增大而增大 | T2 命题 4 | 得分张量 ANOVA | 零 |
| P4 | occurrence 掩码集中在远步长的程度与 $\mathrm{Var}(\phi)$ 份额一致；按步长设上限后，差距中由 $\phi$ 贡献的部分消失 | T2 | E0.5①、E5.1 | 零或低 |
| P5 | 用分位剖面通过恒等式可以精确复现表 2 的 MAE 和 MSE 变化；ΔMSE 主要来自高分位 | T3a | E0.12 | 零 |
| P6 | "MSE 升、MAE 降"只出现在训练残差偏态的数据集上 | T3b | 残差偏度 | 零 |
| P7 | 干净噪声近似高斯时，掩码在 MSE 和 MAE 上都优于 L1；偏态时 L1 的 MAE 更好但 MSE 更差 | T3d | **E1** | P0 实验 |
| P8 | 等间距删窗相对随机的优势随 $T$ 增大而增大；按得分选窗形成长度约为 $\Theta(T)$ 的连续段 | T4 | E7、E0.5⑤ | 低 |
| P9 | 注入点异常时，细单元在 $r\approx\varepsilon$ 饱和，window 需要 $r\approx TD\varepsilon$ | T5 推论 11 | **E6** | 低 |
| P10 | 注入跨通道事件时 time column ≈ occurrence；注入单通道长片段时 channel row ≈ occurrence | T5 推论 12 | E6 | 低 |
| P11 | 在 $\varepsilon=0$ 的干净合成数据上，high end 掩码没有收益或有害 | T8 | E6 | 几乎零 |
| P12 | low end 在 r=40% 时有害；high end 曲线在 $\kappa\varepsilon$ 附近出现拐点 | T6 | E3 | P1 实验 |
| P13 | DLinear 上 10% 得分与闭式 OLS 残差高度一致，且对打分时刻不敏感；iTransformer 上更敏感 | T7 | DLinear 闭式解 + E8 | 零或低 |

**如果部分预测不成立**：
- 如实报告，并把失败写成边界条件。例如 P7 失败（L1 全面占优），就按 `FINAL_META_REVIEW.md` §6.2 决策树中的"情况 E"重新定位；T1–T4 的结构性结论不受影响。
- 关键是：**理论给出预测、实验检验预测**，这种结构本身就满足 TMLR 的第一条标准，即使有个别预测失败也是如此。

---

## 7. 实施顺序与工作量

| 周 | 理论工作 | 同步需要的分析或实验 |
|---|---|---|
| 1 | T1（定义、引理 1–2、命题 3、κ 表、Hasse 图）；T3c（low end 上界） | E0.4（移除质量、κ）；E0.5（掩码结构） |
| 2 | T2（方差分解）；T3a（分位数恒等式）；T3b（偏态命题） | 得分张量 ANOVA；E0.12（分位剖面）；残差偏度 |
| 3 | T3d（掩码与 L1/Huber）；T4（系统抽样） | E1 启动；E7 |
| 4–5 | T5（定理 10 的完整证明，含输入侧附注）；T6 | E6（合成注入，直接检验 T5）；E3 |
| 6 | T7（GD 动力学）；T8（Discussion） | DLinear 闭式 OLS；E8 |
| 7 | 统一记号、写附录证明、自查 | 内部模拟审稿（可以再派几个独立审稿代理只审理论部分） |

**分工建议**：T1、T3a、T3c、T4 的证明都只有几行，由一位作者完成即可。T5、T7 需要更仔细的推导（尤其是分离条件、A3 的一般位置条件和重叠窗口的方差项），建议由熟悉线性回归和鲁棒统计的作者负责，并让另一位作者独立核对。

---

## 8. 风险清单

| 风险 | 应对 |
|---|---|
| 审稿人认为 T1 "太平凡" | 强调它的价值不在证明难度，而在**给出可检验的定量预测**（κ 表 → P2、P9、P10），以及把 S2 的批评转化为明确的结构性质 |
| T2 的可加模型拟合差 | 写成描述性分解；报告残余方差占比；只用内部块 |
| T5 的假设 A1（输入不受污染）被质疑 | 主动在附注中去掉 A1 进行讨论，并把它转化为 E10 的动机 |
| T3d 的预测（P7）被 E1 否定 | 理论部分仍然成立（它说的是"何时"掩码占优），只调整论文的实践建议 |
| 理论太多冲淡主线 | 主文只放命题陈述和一句直觉解释，证明全部放附录；每个命题都配一个"在数据中检验"的图或表 |

---

## 参考文献（Harvard；只列本文新引用、且确信存在的文献）

- Arpit, D. et al. (2017) 'A closer look at memorization in deep networks', *Proceedings of ICML*.
- Cochran, W.G. (1946) 'Relative accuracy of systematic and stratified random samples for a certain class of populations', *The Annals of Mathematical Statistics*, 17(2), pp. 164–177.
- Cochran, W.G. (1977) *Sampling Techniques*. 3rd edn. New York: Wiley.
- Fuller, W.A. (1987) *Measurement Error Models*. New York: Wiley.
- Hampel, F.R., Ronchetti, E.M., Rousseeuw, P.J. and Stahel, W.A. (1986) *Robust Statistics: The Approach Based on Influence Functions*. New York: Wiley.
- Hounie, I., Porras-Valenzuela, J. and Ribeiro, A. (2024) 'Loss shaping constraints for long-term time series forecasting', *ICML*.
- Huber, P.J. (1964) 'Robust estimation of a location parameter', *The Annals of Mathematical Statistics*, 35(1), pp. 73–101.
- Koenker, R. (2005) *Quantile Regression*. Cambridge: Cambridge University Press.
- Li, M., Soltanolkotabi, M. and Oymak, S. (2020) 'Gradient descent with early stopping is provably robust to label noise for overparameterized neural networks', *Proceedings of AISTATS*.
- Liu, S., Niles-Weed, J., Razavian, N. and Fernandez-Granda, C. (2020) 'Early-learning regularization prevents memorization of noisy labels', *NeurIPS*.
- Madow, W.G. and Madow, L.H. (1944) 'On the theory of systematic sampling, I', *The Annals of Mathematical Statistics*, 15(1), pp. 1–24.
- Mindermann, S. et al. (2022) 'Prioritized training on points that are learnable, worth learning, and not yet learnt', *ICML*.
- Rousseeuw, P.J. (1984) 'Least median of squares regression', *Journal of the American Statistical Association*, 79(388), pp. 871–880.
- Rousseeuw, P.J. and Leroy, A.M. (1987) *Robust Regression and Outlier Detection*. New York: Wiley.
- Searle, S.R., Casella, G. and McCulloch, C.E. (1992) *Variance Components*. New York: Wiley.
- Shen, Y. and Sanghavi, S. (2019) 'Learning with bad training data via iterative trimmed loss minimization', *ICML*.
- Sorscher, B., Geirhos, R., Shekhar, S., Ganguli, S. and Morcos, A.S. (2022) 'Beyond neural scaling laws: beating power law scaling via data pruning', *NeurIPS*.
- Zeng, A., Chen, M., Zhang, L. and Xu, Q. (2023) 'Are Transformers effective for time series forecasting?', *AAAI*.
