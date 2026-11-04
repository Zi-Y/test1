# 理论部分建议（v4：经三轮独立数学复核修订）

**论文**：*Prune the Supervision, Not the Windows: Decision Units for Data Pruning in Time-Series Forecasting*（投 TMLR）
**配套文件**：
- `FINAL_META_REVIEW.md`（审稿问题编号 S1–S3、A1–A8、#1–#38）
- `EXPERIMENT_PLAN.md`（实验编号 E0–E17）
- `theory_checks/`：
  - 第一、二轮复核：`check_T1_T3.md`、`check_T4_T8.md`
  - 第三轮复核：`round3_T1_T2.md`、`round3_T3_T8.md`、`round3_T4_T7.md`

  所有修正的推导细节、反例和确定性数值核对都在这些报告里。

**本文目标**：给出能**直接支撑论文主张、并把审稿人的主要批评转化为论文贡献**的理论。每条结果都写明成立条件和证明要点，以及它预测什么。所有预测都留给作者在**真实数据**上检验。本文不包含任何仿真结果；文中的数值常数都来自确定性公式。

---

## 记号（v4 统一，避免 v3 中的符号冲突）

| 记号 | 含义 |
|---|---|
| $N_{\text{win}}=N-L-T+1$ | 训练窗口数（论文记为 $n$；本文不再用 $n$ 表示其他量） |
| $\Omega=\{(i,t,d)\}$，$\lvert\Omega\rvert=N_{\text{win}}TD$ | 全部损失项（occurrence） |
| $h=t-i-L\in\{0,\dots,T-1\}$ | 步长位置（lead），**不再用 $k$** |
| $c(t)=\lvert\mathcal W(t)\rvert$ | 覆盖目标时刻 $t$ 的窗口数，与 $d$ 无关；只对 $c(t)\ge1$ 的时刻定义 element |
| $e_\omega=\hat y_\omega-y_\omega$ | 残差（**全文统一**这一符号；T7 的 GD 递推中会特别注明） |
| $\mathcal P$、$B$ | 单元（划分）、块 |
| $N_{\mathcal P}$ | 单元 $\mathcal P$ 的块数（论文中的"成员数"$m$） |
| $m_{\max}$ | 单元的最大块大小 |
| $m_P^{\text{el}}$ | 一个块覆盖的**不同 element** 个数 |
| $r$ | 预算。**论文的预算是成员（块）比例**：屏蔽 $b(r)=\lfloor rN_{\mathcal P}+\tfrac12\rfloor$ 个块 |
| $f_0$ | 打分模型（全数据运行在 10% epoch 时的检查点） |
| $b_0(x)=f_0(x)-f^*(x)$ | 打分模型在输入 $x$ 处的误差 |
| $s$ | T4 中的抽样间隔 |
| $K_{\text{gd}}$ | T7 中的 GD 步数 |

---

## v4 相对 v3 的修订（第三轮复核结论；都经我核对后采纳）

| # | 位置 | v3 的问题 | v4 的处理 | 来源 |
|---|---|---|---|---|
| 1 | T1 含义 2；§4 | ❌ "没有一个单元在所有结构上都最优" | 对任意 $S\subseteq\Omega$ 都有 $\kappa_{\text{occ}}(S)\equiv1$，所以 occurrence 在 κ 意义下处处最优。改为：**除 occurrence 外**没有单元处处 κ=1；occurrence 在**收益**上并非处处最优（DLinear 在 ETTh2、ETTm2 上 channel row 更好），这是 κ 之外的经验事实 | R3-A |
| 2 | T1 对齐指数 | ⚠️ 按 element 定义的 $A_r$ 使 $\kappa_{\text{occ}}=\kappa_{\text{elem}}\equiv1$，加上 κ 关于细化单调、在 $1/r$ 处饱和，几乎是空检验 | 改为用 **occurrence 级**的 top-r 集合 $S_r$；取较小的 $r$ 或用归一化形式；只比较不可比的单元对；逐骨干计算 | R3-A |
| 3 | T1 | 推论漏了 element 与 window 的先后同样由数据决定；κ 只对 element 集合定义 | 把 κ、ρ 推广到任意 $S\subseteq\Omega$，写出单调性和上下界；推论改用 $M_{\mathcal P}(s):=\max$ 的定义 | R3-A |
| 4 | T1 原则 6 | ⚠️ "实际项数差异可以忽略"对 element 不成立 | element 的实际屏蔽项数可达名义值的 $(N_{\text{win}}+T-1)/N_{\text{win}}$ 倍（ETTh、T=720 时为 **1.092**），所有比较都要按**实际项数**对齐 | R3-A、R3-C |
| 5 | T2 | ⚠️ η² 的 $1/m$ 基线对预测残差无效（共享目标噪声会给出 $\rho^2+(1-\rho^2)/T$；随机游走加完美预测器下 $\eta^2_{\text{elem}}=0.448$） | 主度量改为**质量捕获比** $M_{\mathcal P}(s)/M_{\text{occ}}(s)$ 和 **top-r 重叠率**；η² 降为辅助量，并须用基于模型的零假设 | R3-A |
| 6 | T2 步长剖面 | ❌ 条件写错了对象 | 给出**精确恒等式**：逐步长均值中 β 精确抵消，偏差只来自序列两端各 $T-1$ 个时刻；别名斜率的紧界改为 $4B_\beta/(N_{\text{win}}-1)$ | R3-A |
| 7 | T2 | 遗漏：time column 同样完全暴露于步长效应；序列两端的 element 只在近步长或远步长上被看到 | 新增"各单元对 φ 的暴露"表；预测 element 掩码在 φ 递增时会**偏向训练段末尾** | R3-A |
| 8 | T2 双重足迹 | ⚠️ 通道混合时 time、chan 的 κ 也是 1；输入足迹的幅度约为 $M/L$，有 RevIN 时有界；$\kappa_{\text{elem}}$ 在各 T 下为 1.99/2.99/4.49/8.49 | 全部改正，并注明输入足迹**很可能根本进不了高分集合** | R3-A |
| 9 | §0.3 | ⚠️ ETTh2、ETTm2 的"上下文侧"解读不成立：起点带上 $\kappa_{\text{chan}}=\kappa_{\text{elem}}$；三骨干平均值由 DLinear 主导 | 改为**逐骨干**解读；ETTh2、ETTm2 一行降为"未解释"；solar 一行保留为假设（三个骨干上一致） | R3-A |
| 10 | T3 命题 5 | ❌ v3 把条件改成加在 $2\delta-\delta^2$ 上，**改反了** | 恢复为 $\mathrm{Cov}_{w_1}(\delta,q)\le0$（它本身就充分），并给出精确分解 | R3-B |
| 11 | T3a | ❌ "solar 上 CVaR +0.47"引错（这是七个数据集的平均值）；⚠️ 阶梯模型的"⇔ 重尾"解读不对 | 改正引用；删去"重尾"解读，改用经验 δ(u) 和精确分解 | R3-B |
| 12 | 命题 0 | ⚠️ 推论列举不全；命名易与 NFL 定理混淆 | 改名为"注记 0（总体 MSE 下界）"；补上"优化不精确""按验证集选 epoch""统计噪声"三种来源 | R3-B |
| 13 | 命题 6′ 与"三种效应" | ⚠️ 截断比例应写成 $r(x)$；效应相互重叠，并遗漏了两类机制；可检验量混入了 $b_0$ | 改为"多类机制"（非可加）；补上 x 重加权和打分方差的继承；可检验量改用全数据最终模型的残差，并按 $b_0$ 的符号分组 | R3-B |
| 14 | 命题 7 | ❌ 掩码运行从头训练，"每一步扰动都很小"不成立 | **改用精确结果**：线性 LS 中掩码解 − 全数据解 $=(X_K^\top X_K)^{-1}X_S^\top e_S(\theta^*)$，由**收敛点**残差决定，与训练轨迹无关；按 batch 归一化本质上是重加权 | R3-B，我推导 |
| 15 | 命题 8′ | ❌ "掩码偏差为 0""本质区别是 ψ 形状"对论文的一步掩码不成立 | 区分理想掩码、**一步掩码**和不动点掩码三种对象。一步掩码的 IF 为 $\psi_c/(1-2\alpha)+k\,\mathrm{IF}_{f_0}$；打分模型若是 LS 型则 IF 无界；偏差约为 $s(b_0)\approx k\,b_0$。E1 的预测改为**条件形式** | R3-B |
| 16 | 命题 16 | ⚠️ 条件和适用范围 | 下界只需对称，上界需要对称且单峰，并要求窗口内概率质量为正；**只严格适用于 occurrence**；0.377 只在 $b_0$ 小且同方差时成立，异方差时局部斜率为 $k(z)=2z\varphi(z)/(2\Phi(z)-1)$；删去 MC-dropout 的解释；P15 改用更干净的统计量 | R3-B |
| 17 | T8 | ❌ Sorscher 统一的第 3 条与命题 6′、16 矛盾；⚠️ "差 30%"的量级会误导 | 重写统一表述（改变目标的三种情形，并补上数据量这一变量）；明确 1.30 是估计量方差，对测试 MSE 的贡献是 $O(p/N)$；补上 RC (1980) 的等价性条款 | R3-B |
| 18 | T4 | ❌ "系统抽样相对分层抽样增益不大" | 新增**命题 9a**：$s\mid T$ 时系统抽样的误差只来自序列两端，$EV_{sy}/EV_{st}\approx T/N_{\text{win}}$（0.011–0.085），**小一到两个数量级**；论文的 50% 删窗正属此类 | R3-C |
| 19 | T4 | ⚠️ 若干条件需要补全 | §7.3 的掩码损失是比率估计量，需先线性化；周期混叠只适用于"保留 1/k"，gcd=1 的补救会排除 50%，改为推荐每个 epoch 重抽起点；相位偏差需要设定错误；"P 整除 T 时抵消"只对加性、目标侧的周期成立；连续删除使 $(\ell-T+1)D$ 个 element 失去监督；连续段长度是**重尾分布的均值**，应报告 $P(\text{段长}\ge T)$ | R3-C |
| 20 | T5 定理 10 | ❌ "当且仅当 $r\ge\rho$"用项数口径，而论文按块数取整；⚠️ A1 没有写进条件；(ii) 应逐步长陈述 | 改为**定理 10′**：块数口径；A1 作为条件 (0)；(ii′) 逐步长满列秩；用排序条件 (i\*) 取代泄漏条件 (i) | R3-C |
| 21 | T5 | ❌ "窗口单元失效的第二个机制"、ECL 的结论、P14 | **删除**。(i) 只是充分条件；截距泄漏是均匀的，不改变排序（ECL 规模下有确定性反例）。P14 改写为注入实验中检验 (i\*) | R3-C |
| 22 | T5 推论 11 | ⚠️ | 写成**期望**（实现值的标准差为 1–2 个百分点）；"能 ✓"只是目标侧、A1 下的结论；T=720 时 channel row 需要 97.3% | R3-C |
| 23 | T5 输入侧 | ⚠️ 新问题：$M$ 大时，打分**认不出**输入被污染的行（$Mw_j\to0$） | 写明 38.2% 只是**必要**预算，high-end 掩码不会把这些行选出来；补全矩阵衰减公式的条件；E10 的条件改为逐特征方向陈述 | R3-C |
| 24 | T6 | ❌ "element 得分近似高斯，$\pi_c<1\%$"：同一 element 的各项共享目标噪声 | $\pi_c$ 介于约 0.8% 与 7.6% 之间。新预测：**即使没有异常，Tukey 预算也可能屏蔽数个百分点**。同号情形补上 $1/(1-r)^2$；补上 $r^*\in\{0,\rho\}$ 的严格证明；补上 individual 设置下 $p/N_K$ 不小的提醒 | R3-C |
| 25 | T7 | ❌ "$P_\perp y$ 是论文 unexplained error 的精确对应" | $P_\perp y$ 是**样本内**残差；论文的 unexplained error 是 iTransformer 上的**交叉拟合**残差。DLinear 的闭式解 $(I-H_{SS})^{-1}e_S$ 只是**类比**，不能"复现 §7.1"；截距的预条件系数是 2；非零初值时暂态项用 $y-Xw_0$；只减均值的 RevIN 使函数类受限 | R3-C |

---

## 0. 结论先行

### 0.1 一句话

这篇论文**最适合加的理论不是泛化界，而是四类结构性结果**：
1. **组合结构**（T1）：五种单元都是 $\Omega$ 的划分，由此得到细化序、移除质量的单调性，以及附带损伤系数 κ；
2. **统计结构**（T3）：论文的掩码是"**按打分模型残差的一步截断 LS**"，由此给出总体 MSE 下界、分位数恒等式、**掩码模型被拉向打分模型**（命题 16），以及它与 L1、Huber 的真实关系；
3. **抽样结构**（T4）：系统抽样与覆盖引理，**$s\mid T$ 时等间距删窗的误差只来自序列两端**；
4. **线性模型中的精确结果**（T5、T7，DLinear 本身是线性模型）："单元 × 结构 × 预算"律（定理 10′），以及掩码解偏移的精确公式。

### 0.2 优先级

| 优先级 | 编号 | 理论 | 放在论文哪里 | 回应的审稿问题 | 工作量 | 真实数据上检验的成本 |
|---|---|---|---|---|---|---|
| ★★★ | **T1** | 划分、细化序、移除质量单调性、κ、质量捕获比 | 新增 §3.5 | **S2**、A6 | 2–3 天 | 零 |
| ★★★ | **T3** | 注记 0；分位数恒等式；命题 6′；掩码偏移的精确公式（命题 7′）；一步掩码与 L1/Huber（命题 8″）；被拉向打分模型（命题 16） | §5、§6 | **S1**、#11、#33、#36，以及尾部变差 | 约 1–1.5 周 | 零（E1 除外） |
| ★★★ | **T2** | 质量捕获比与 top-r 重叠率；步长剖面恒等式；对 φ 的暴露；双重足迹 | §3.6 或 §5 | A3、element 与 occurrence 的差距、A6 | 3–5 天 | 零 |
| ★★ | **T4** | 系统抽样（含命题 9a）、覆盖引理、周期混叠 | §7.3 | **S3** | 3–5 天 | E7 |
| ★★ | **T5** | 定理 10′ 及推论（目标侧、A1 下）；输入侧的局限 | 新增 §8，证明放附录 | **S2**、A2 | 2–3 周 | 注入实验、E3 |
| ★ | T6 | 风险形状、$r^*\in\{0,\rho\}$、Tukey 预算的行为 | T5 的推论 | A2、自适应预算 | 1–2 天 | E0.9、E3 |
| ★ | T7 | 早期得分 = 不可解释部分 + 慢信号；DLinear 的闭式类比 | §7.1 | **A4** | 1–2 周 | 近乎零 |
| ☆ | T8 | 截断何时有益；与 Sorscher 的关系（启发式） | Discussion | A7 | 1 天 | 零 |

### 0.3 与现有数据的初步对照（**只是回看论文中的表，属于假设，待检验**）

由于打分是对每个"骨干 × 数据集 × 步长"分别做的，**解读必须逐骨干进行**，不能用表 1 的三骨干平均值。

| 数据集 | 观察（MAE，表 4–6） | 可能的解读（假设） |
|---|---|---|
| solar | 三个骨干上，time col.、occ.、element 都远好于 chan. row 和 window（表 1 平均：−7.83、−8.81、−8.43 对 −3.41、−2.76） | 困难是**通道间同时刻对齐**的（日出、日落）。κ 表第 2 行：occ、elem、time 的 κ 为 1，chan、win 的 κ 为 $T$。三个骨干方向一致，可以作为一个较强的假设 |
| ETTh2、ETTm2 | DLinear 上两者都是 chan. row 最好（ETTh2：−12.09 对 occ. −10.81；ETTm2：−13.46 对 −11.71）；TimeMixer 上 ETTh2 是 chan. row 最好（−3.84 对 −3.23），ETTm2 是 occ. 最好；iTransformer 上两者都是 occ. ≈ element > chan. row（表 4–6） | **目前没有理论解释**。v3 的"上下文侧"解读不成立：起点带上 $\kappa_{\text{chan}}=\kappa_{\text{elem}}$，两者无法区分。可能与 DLinear 特有的机制有关，有待研究 |
| ETTm1、weather | 三个骨干上 occ. 都最好 | 与目标侧困难一致；但目标侧结构下 $\kappa_{\text{occ}}=\kappa_{\text{elem}}=1$，**解释不了** occ. 与 element 的差距（例如 −4.57 对 −3.73）。这个差距需要 T2 中的步长、上下文效应来解释 |

---

## 1. 原则

1. **理论服务于主张**：不加与实验无关的泛化界。
2. **每条理论都给出可证伪的预测，并在真实数据上检验**；"定理的推论"（数据不可能违反的结论）不能列为可证伪预测，只能作为实现检查。
3. **假设写在明处，主张不过度**。三轮复核中发现的错误几乎都属于两类：**漏写条件，以及把近似当成精确**。写作时，每个命题都要单列"成立条件"。
4. **优先使用精确、可完整证明的结果**：组合恒等式、分位数恒等式、线性 LS 的闭式结果、经典鲁棒统计结论。
5. **把审稿人的批评写成定理**（例如把 S2 写成 T1 的推论）。
6. **所有跨单元比较都按实际屏蔽的项数对齐**。论文按块数取整，element 的块大小不等，实际项数可比名义值多出 $O(T/N_{\text{win}})$ 的比例（ETTh、T=720 时为 +9.2%）。
7. **引用经典结论之前先核对原文**：Cochran (1946) 的准确条件、Ruppert & Carroll (1980) 的全文、Huber 关于极小极大偏差的出处（见 §8）。

---

## 2. 理论骨架

```
T1 组合：单元 = Ω 的划分 → 细化序、移除质量单调性、κ_P(S)、ρ_P(S)
   │                      （κ_occ ≡ 1；κ 关于细化单调）
   ├─► T2 统计描述：质量捕获比、top-r 重叠率、步长剖面恒等式、对 φ 的暴露、双重足迹
   └─► T5 线性律（DLinear、目标侧、A1 下）：定理 10′（块数口径；排序条件 (i*)）
          ├─ 推论 11：期望所需预算 1−(1−ε)^{m_P^el}
          ├─ 输入侧：矩阵衰减；打分认不出输入被污染的行
          └─► T6 风险形状、r* ∈ {0, ρ}、Tukey 预算的行为
T3 一步截断 LS：注记 0；分位数恒等式；命题 6′；命题 7′（掩码偏移的精确式）；
   命题 8″（一步掩码 vs L1/Huber）；命题 16（被拉向打分模型）
T4 抽样：Cochran（附条件）、命题 9a（s | T 时只有边界误差）、覆盖引理、周期混叠
T7 早期得分 = 不可解释部分 + 慢信号；DLinear 的交叉拟合闭式类比
T8 截断何时有益（命题 15）；与 Sorscher 的关系（启发式）
```

---

## 3. 逐项方案

### T1. 单元作为划分（★★★）

**放在哪里**：新增 §3.5 "Units as partitions"。

**定义**：五种单元都是 $\Omega$ 的划分（element 只包含 $c(t)\ge1$ 的时刻，所以不含空块）：

| 单元 | 块 | 块大小 | 块数 $N_{\mathcal P}$ | 每块覆盖的不同 element 数 $m_P^{\text{el}}$ |
|---|---|---|---|---|
| window | $B_i$ | $TD$ | $N_{\text{win}}$ | $TD$ |
| time column | $B_{i,t}$ | $D$ | $N_{\text{win}}T$ | $D$ |
| channel row | $B_{i,d}$ | $T$ | $N_{\text{win}}D$ | $T$ |
| occurrence | $\{\omega\}$ | 1 | $N_{\text{win}}TD$ | 1 |
| element | $B_{t,d}$ | $c(t)\le T$ | $(N_{\text{win}}+T-1)D$ | 1 |

**细化序与 Hasse 图**（设 $N_{\text{win}}\ge2$）：
- occurrence ⪯ time column ⪯ window；occurrence ⪯ channel row ⪯ window；occurrence ⪯ element。
- 当 $T\ge2$ 时，element 与 window、channel row 都不可比；当 $T\ge2$ 且 $D\ge2$ 时，element 还与 time column 不可比，并且 **time column 与 channel row 不可比**。
- 交运算：window ∧ element = time ∧ element = chan ∧ element = time ∧ chan = occurrence。并运算：time ∨ chan = window。
- 当 $T\ge2$ 时，window ∨ element 是只含一个块的划分 $\{\Omega\}$，它不在五种单元之中。所以**这五种单元不构成划分格的子格**，图只能当作偏序图画。
- 准确表述：**在五种单元中**，element 是唯一不细于 window 的单元。

**引理 1（细化单调性）**：若 $\mathcal P\preceq\mathcal Q$，则任何由 $\mathcal Q$ 块组成的并也是由 $\mathcal P$ 块组成的并，所以对任意集合函数，在相同容量下的最大值关于细化单调。∎

**引理 2（块均值前缀，在其前缀大小处精确最优）**：把块按打分时刻的块均值 $\bar e^2_B$ 降序排列，取前 $k$ 个块，记其并集大小为 $s_k$。在所有"$\mathcal P$ 块的并、且大小 $\le s_k$"的集合中，这个前缀使打分时刻的移除质量 $F(S)=\sum_{\omega\in S}e^2_\omega$ 精确最大。
*证明*：这是 0-1 背包问题，其 LP 松弛的最优解按价值密度贪心装入；容量恰好为 $s_k$ 时贪心解不含分数项，所以 LP 最优 = 前缀价值 ≥ 整数最优。∎
- 注意：结论**只在前缀大小处成立**（非前缀容量下贪心可能次优），而论文的规则恰好取前缀；
- 它说的是"在实际项数下**最大化打分时刻的移除质量**"，**不涉及测试误差**；
- 对等大小的单元，这几乎就是定义本身，只对 element 有实质内容。

**推论（S2 的形式化）**：定义 $M_{\mathcal P}(s):=\max\{F(S):S\text{ 为 }\mathcal P\text{ 块之并},\ |S|\le s\}$。由引理 1，对**所有** $s$ 都有
$$M_{\text{occ}}(s)\ge M_{\text{time}}(s)\ge M_{\text{win}}(s),\qquad M_{\text{occ}}(s)\ge M_{\text{chan}}(s)\ge M_{\text{win}}(s),\qquad M_{\text{occ}}(s)\ge M_{\text{elem}}(s).$$
由引理 2，当 $s$ 是 $\mathcal P$ 的前缀大小时，$M_{\mathcal P}(s)$ 就是论文中 $\mathcal P$ 掩码移除的质量。
- 理论只给出偏序：**element 与 window、time column、channel row 之间的先后都由数据决定**。例如若质量全部集中在一个窗口里，window 能取到全部质量，而 element 在同样项数下只能取到 $1/T$。
- **收益 ≠ 质量**：例如 DLinear 在 ETTh2 上 channel row（−12.09）胜过 occurrence（−10.81），而 occurrence 是移除质量最大的单元。这正是 E4（按质量匹配预算）的动机；比较时要按实际项数对齐（原则 6）。

**命题 3（κ 与 ρ）**：对任意 $S\subseteq\Omega$，定义
$$\kappa_{\mathcal P}(S)=\frac{|\bigcup\{B\in\mathcal P:B\cap S\ne\emptyset\}|}{|S|},\qquad \rho_{\mathcal P}(S)=\frac{|\bigcup\{B\in\mathcal P:B\cap S\ne\emptyset\}|}{|\Omega|}.$$
性质：
- $1\le\kappa_{\mathcal P}(S)\le|\Omega|/|S|$；
- **$\kappa_{\text{occ}}\equiv1$**；
- **单调性**：$\mathcal P\preceq\mathcal Q\Rightarrow\kappa_{\mathcal P}(S)\le\kappa_{\mathcal Q}(S)$。

$\rho_{\mathcal P}(S)$ 是"能够完全移除 $S$ 的**最小**项数比例"。它只是必要条件，是否充分取决于排序（见 T5）。注意：论文的预算按块数计，对 element 而言，按块数计的阈值与 ρ 相差 $(N_{\text{win}}+T-1)/N_{\text{win}}$ 倍（见定理 10′）。

**κ 表**（$S=\text{occ}(A)$，$A$ 为目标集合）：

| 结构 $A$ | occ. | element | time col. | chan. row | window | 适用范围 |
|---|---|---|---|---|---|---|
| 单点 $(t,d)$ | 1 | 1 | $D$ | $T$ | $TD$ | 所有 $c(t)\ge1$ 的 $t$（含序列两端） |
| 跨通道同时刻 $\{t\}\times[D]$ | 1 | 1 | 1 | $T$ | $T$ | 同上 |
| 单通道片段（长度 $\ell$） | 1 | 1 | $D$ | $\frac{\ell+T-1}{\ell}$ | $\frac{(\ell+T-1)D}{\ell}$ | occ.、element、time 三列对所有位置成立；chan.、window 两列只对内部片段成立（两端的通式见 R3-A 报告第 7 节，端点值可能更大也可能更小） |
| 全通道片段 | 1 | 1 | 1 | $\frac{\ell+T-1}{\ell}$ | $\frac{\ell+T-1}{\ell}$ | 同上 |

- κ 是纯组合量，表中数值没有附加条件。
- 但"高残差集合**恰好是**某个片段"只在 A1（输入未被污染）下成立。现实中，片段也会出现在输入里，高残差集中在片段起点之后（以及终点之后）的"带"上；这需要模型是水平等变的，例如每个步长的权重和为 1，或者带 RevIN 类归一化。**在最简的起点带模型下 $\kappa_{\text{chan}}=\kappa_{\text{elem}}=2T/(T+1)$，κ 无法区分这两个单元。**

**含义**：
1. 单点困难下 window 的 κ 为 $TD$（ETT、T=96 时为 672），这与 "window gains least" **一致**；但 κ 只刻画可达性，不是收益的完整机制（例如 channel row 的 κ=T，比 time column 的 D 大得多，却在 DLinear 的 ETTh2、ETTm2 上最好）。
2. **除 occurrence 外**，没有一个单元对所有结构的 κ 都为 1：
   - element 对目标侧结构为 1，对输入足迹为 $(L+T-1)/L$；
   - channel row 对输入足迹为 1，对单点为 $T$；
   - time column 对跨通道同时刻为 1。

   occurrence 在 κ 意义下处处最优，但**在收益上并非处处最优**，这是 κ 之外的经验事实。可能的原因包括粗单元的平均降低了打分噪声、掩码的方差代价、命题 16 的拉动等，都有待检验。
3. **对齐诊断（修正版，零成本）**：
   - 取 **occurrence 级**的 top-r 集合 $S_r$（建议用较小的 $r$，例如 0.5%–2%，以避免饱和）；
   - 报告 $\kappa_{\mathcal P}(S_r)$，或归一化形式 $(\kappa-1)/(1/r-1)$；
   - 只把不可比的单元对（time 对 chan、element 对 chan、element 对 time）作为检验对象；
   - **逐骨干**计算。

---

### T2. 各单元看到了什么（★★★）

**放在哪里**：§3.6 或 §5。

**主度量（取代 v3 的 η²）**，都可以零成本计算：
1. **质量捕获比**：$\mathrm{MCR}_{\mathcal P}(s)=M_{\mathcal P}(s)/M_{\text{occ}}(s)\in[0,1]$，在实际项数 $s$ 处计算；
2. **top-r 重叠率**：occurrence 掩码中有多少比例被 $\mathcal P$ 的掩码覆盖。

两者都直接与掩码选择相关，并与 T1 衔接。

**辅助度量 η²**（相关比，$SS_B/SS_T$）：
- 它关于细化单调，$\eta^2_{\text{occ}}=1$；
- **但"无结构基线约 $1/m$"只在 i.i.d. 下成立**，对预测残差无效：同一 element 的各项共享目标新息（高斯下精确给出 $\rho^2+(1-\rho^2)/T$）；随机游走加完美预测器时 $\eta^2_{\text{elem}}=0.448$，而 $1/T=0.0104$；
- 如果使用 η²，必须用**基于模型的零假设**做归一化，并逐骨干计算。

**v1 可加模型不可识别**（保留这个结论）：在 $e^2=\mu+\alpha(t,d)+\beta(i,d)+\gamma(i,t)+\phi(h)+\eta$ 中，φ 和 $\bar\alpha(t)$、$\bar\beta(i)$ 都会被 γ 吸收，并且存在 APC 式的线性混淆。**不能报告"各效应的方差份额"。**

**步长剖面的精确恒等式（取代 v3 的 O(T/n) 说法）**：定义 $\hat\phi(h)$ 为步长 $h$ 的原始平均平方残差。在**不含 γ**（或 γ 沿每条对角线均值为 0）的可加模型下，
$$\hat\phi(h)-\hat\phi(0)=\phi(h)-\phi(0)+\frac{1}{N_{\text{win}}}\Big[\sum_{j<h}\bar\alpha(L+N_{\text{win}}+j)-\sum_{j<h}\bar\alpha(L+j)\Big]+(\bar\eta_h-\bar\eta_0).$$
- **β 精确抵消**，因为每个步长都恰好看到全部 $N_{\text{win}}$ 个窗口；
- 偏差只来自序列两端各 $T-1$ 个时刻的 $\bar\alpha$，上界为 $(T-1)R_\partial/N_{\text{win}}$，其中 $R_\partial$ 是 $\bar\alpha$ 在这些端点时刻的极差；
- 稳健性检查（零成本）：去掉两端各 $T-1$ 个时刻后重算。

**拟合可加模型时的识别宽度**：若真实的 β 与拟合的 β 都满足 $\|\beta(\cdot,d)\|_\infty\le B_\beta$，别名斜率的紧界为 $|a|\le4B_\beta/(N_{\text{win}}-1)$，所以 φ 在 $[0,T)$ 上的不确定宽度为 $4B_\beta(T-1)/(N_{\text{win}}-1)$（ETTh、T=720 时约为 $0.37B_\beta$，**不能说可以忽略**）。

**各单元对步长效应 φ 的暴露（新增）**：

| 单元 | 块内的步长构成 | 对 φ 的暴露 |
|---|---|---|
| occurrence | 单一步长 | 完全暴露 |
| **time column** | 单一步长 | **完全暴露**（与 occurrence 相同） |
| channel row、window | 全部 $T$ 个步长 | φ 抵消 |
| element（内部） | 全部 $T$ 个步长 | φ 抵消 |
| element（训练段开头 $T-1$ 个时刻） | 只有近步长 | 得分被低估（φ 递增时） |
| element（训练段末尾 $T-1$ 个时刻） | 只有远步长 | 得分被**高估**（φ 递增时） |

**可检验的推论**：
- (i) 若 φ 递增，occurrence 和 time column 的掩码都会偏向远步长。这时它们在效果上部分等价于**下调**远步长的权重。注意方向：Hounie et al. (2024) 的 loss shaping 是通过对偶变量**上调**超出约束的步长的权重，两者方向相反，只在"都是按步长加权"的意义上类似。E5.1 的按步长设上限应**同时**用于 occurrence 和 time column。
- (ii) element 掩码会**系统性地偏向训练段末尾**的端点元素。ETTh、T=720 时，端点元素约占全部元素的 16.8%，所以不可忽略。建议零成本统计 element 掩码中端点元素的占比。
- "occurrence 的 top-r 在远步长被屏蔽得更多"成立的条件是：非 φ 部分在各步长上**同分布**（不需要可交换）；φ 递增是经验事实，需要先验证。

**双重足迹（修正版）**：一个被污染的 $y_{t,d}$ 留下两处痕迹。
- **目标足迹**：$c(t)$ 个 occurrence，幅度约为 $M$（若打分模型已部分拟合了污染，会更小）。
- **输入足迹**（通道独立模型；输入中含 $t$ 的 $L$ 个窗口 × $T$ 个步长；内部情形）：$\kappa_{\text{chan}}=1$，$\kappa_{\text{time}}=D$，$\kappa_{\text{win}}=D$，$\kappa_{\text{elem}}=(L+T-1)/L$（$L=96$ 时，$T=96、192、336、720$ 分别对应 1.99、2.99、4.49、8.49）。
- **通道混合模型**：输入足迹是 $L$ 个完整窗口，$\kappa_{\text{win}}=\kappa_{\text{time}}=\kappa_{\text{chan}}=1$，$\kappa_{\text{elem}}$ 不变。
- **幅度提醒**：线性预测器每个步长的权重和约为 1、分散在 $L$ 个滞后上时，$|W_{hj}|\sim1/L$，输入足迹的平方残差约为目标足迹的 $1/L^2$；有 RevIN 类归一化时，幅度对 $M$ 有界。所以**输入足迹很可能根本进不了高分集合**，上面这些 κ 只有在它确实是高残差集合时才有意义。
- 论文 §4 对骨干的描述是：iTransformer mixes channels，TimeMixer mixes **scales**。TimeMixer 在 TSLib 中是否通道独立，**需要作者核对实际配置**。表 4 的结果（iTransformer 上 window 最差、element 最好）也不支持"通道混合 → window 更好"的推测，所以 v3 中那句话已删除。

---

### T3. 统计结构：论文的掩码是"按打分模型残差的一步截断 LS"（★★★）

**适用范围**：以下 T3b、T3d、T3e 中关于"截断窗口"的结论**只对 occurrence 单元严格成立**，因为只有它是对单个平方残差取全局阈值。其他单元按块均值选择：例如 element 的保留条件是 $|y-\bar y_{t,d}|\le\sqrt{\tau-V_{t,d}}$，中心是跨窗口的平均预测，半径因元素而异。

#### 注记 0（总体 MSE 下界；v3 中称为命题 0）

设掩码运行与全数据运行使用**同一个模型类** $\mathcal F$；训练损失（式 (1)，各项等权）与测试 MSE 对同一个（窗口化后的）联合分布 $P$ 取平均；$EY^2<\infty$。令 $R(f)=E_P(f-Y)^2$。则对任意掩码权重 $w\in\{0,1\}$（可以依赖 $Y$ 和一个固定的打分模型，$Ew>0$），掩码目标的任一最小化者 $f_w\in\mathcal F$ 都满足 $R(f_w)\ge\inf_{\mathcal F}R$。∎

**推论**：论文观察到的 MSE 平均变化（−1.73% 至 −2.81%，在数据集间**不显著**，p=0.06–0.13；部分数据集上为正）只可能来自以下来源：
- (a) 有限样本估计效应；
- (b) 训练段与测试段分布不同；
- (c) 优化不精确或隐式正则（非凸、Adam、有限 epoch）；
- (d) 按验证损失选 epoch（数据依赖的模型选择）；
- (e) 统计噪声。

#### T3a. 分位数恒等式

记全数据与掩码运行的测试绝对误差分位函数分别为 $q$ 与 $q_{\text{mask}}$（同一测试集），$\delta(u)=1-q_{\text{mask}}(u)/q(u)\le1$。则精确地有
$$\tfrac{\Delta\text{MAE}}{\text{MAE}}=-E_{w_1}[\delta],\ w_1\propto q;\qquad \tfrac{\Delta\text{MSE}}{\text{MSE}}=-E_{w_2}[2\delta-\delta^2],\ w_2\propto q^2 .$$
δ 比较的是**边缘分布的分位**，不是"同一批点改善了多少"。

**命题 5（修正）**：设 $E_{w_1}[q]>0$。若 $\mathrm{Cov}_{w_1}(\delta,q)\le0$，等价地 $E_{w_2}[\delta]\le E_{w_1}[\delta]$，则
$$-\Delta\text{MSE}_{\text{rel}}=2E_{w_2}[\delta]-E_{w_2}[\delta^2]\ \le\ 2E_{w_2}[\delta]\ \le\ 2E_{w_1}[\delta]=2(-\Delta\text{MAE}_{\text{rel}}).$$
"δ 关于 $u$ 非增"是一个充分条件（由 Chebyshev 关联不等式直接得到）。v3 把条件改为加在 $2\delta-\delta^2$ 上是**改反了**：两个条件互不包含，而原条件本身就充分。

**它只说明 MSE 的相对改善不超过 MAE 的两倍，推不出"MSE 动得比 MAE 少"。**
- 阶梯模型的反例：$|e|\sim U[0,1]$，收益只在主体 90%，两者之比为 1.71。
- 阶梯模型 $\delta=\delta_0\mathbf 1\{u\le u^*\}$ 下，两者之比为 $(2-\delta_0)W_2(u^*)/W_1(u^*)$。比值是否小于 1 同时取决于 $u^*$ 和尾部形状：例如高斯误差在 $u^*<0.705$ 时也小于 1；**不能简单地说"⇔ 重尾"**。论文图 2 的收益延续到 q90，在 $u^*\approx0.9$ 时连 Laplace 误差也预测比值大于 1。所以阶梯模型**解释不了**论文中的现象，起作用的是 δ 随分位数的**渐变衰减**。

**论文中的用法**：
- 逐次运行（单次运行的完整测试误差分布）计算精确分解 $2E_{w_2}[\delta]-E_{w_2}[\delta^2]$，并按分位区间画出对 ΔMAE 和 ΔMSE 的贡献（回应 #36），再按论文的方式平均；
- 尾部 δ 为负时可能出现"MSE 升、MAE 降"。例如表 1 中 solar 的 occurrence：MAE −8.81%，MSE +0.91%。（**注意**：表 2 的 CVaR +0.47 是七个数据集的平均值，不是 solar 的值。）

#### T3b. 命题 6′：截断目标的位置偏移

设在 $x$ 处的截断集合为 $R(x)$，$P(R\mid x)=r(x)$（全局阈值下各 $x$ 的截断比例不同），模型足够灵活且设定正确，测试的条件分布与训练相同。则：
1. 掩码 LS 的目标为 $m(x)=E[Y\mid x]-\dfrac{E[(Y-E[Y|x])\mathbf 1_R\mid x]}{1-r(x)}$。**偏移方向由截掉部分的一阶矩决定；偏度的符号不能决定方向**（反例：取值 20、−2、0，概率 0.005、0.2、0.795；按三阶矩是右偏的，但中位数 > 均值，截断后均值上移）。
2. 总体 MSE 的增量为 $E_x[(m(x)-E[Y|x])^2]\ge0$。
3. 若 $m(x)$ 位于条件均值与条件中位数之间，则总体 MAE 不增。∎

**掩码影响 MSE 的机制**（彼此相互作用，**不是可加分解**）：
- (a) 移除污染或降低影响；
- (b) 信息损失（在重尾噪声下符号可能反转，见 T8）；
- (c) 相对**测试**条件均值的位置偏移；
- (d) **x 重加权**：设定错误时会改变最佳逼近；
- (e) 继承打分模型的估计误差（方差项，见 T8；偏差项，见命题 16）。

**可检验量**：被屏蔽项的带符号残差均值，如果用**打分模型**的残差计算，会混入 $b_0$，分不开"噪声非对称"与"打分模型有偏"。应改用**全数据最终模型**（最好换一个随机种子）的残差，并按 $b_0$ 的符号分组报告；同时注明符号约定 $e=\hat y-y$。

#### T3c. 命题 7′（取代 v3 的命题 7）：线性 LS 中掩码解偏移的精确公式

设某个步长的回归是线性 LS（DLinear 就是这种情形），全数据解为 $\theta^*$，保留行为 $K$，屏蔽行为 $S$，并且 $X_K$ 满列秩。则
$$\theta_K-\theta^*=(X_K^\top X_K)^{-1}X_S^\top e_S(\theta^*),\qquad e_S(\theta^*)=X_S\theta^*-y_S .$$
*证明*：由全数据法方程 $X^\top(X\theta^*-y)=0$ 得 $X_K^\top(y_K-X_K\theta^*)=X_S^\top e_S(\theta^*)$，再由掩码法方程 $X_K^\top(X_K\theta_K-y_K)=0$ 相减即得。（位置模型下的检验：$\theta_K-\bar y=\frac1{n_K}\sum_S(\bar y-y_S)$，正确。）∎

**含义**：
1. 掩码的效果**由全数据解在被屏蔽项上的收敛残差决定**，与训练轨迹无关（只要优化收敛到最小化者），也与式 (4) 的全局归一化常数无关。
2. low end 的项如果在**收敛点**上残差也很小，那么它几乎不移动解。这对 low end 的零结果给出了一个**精确**的解释，但前提要检验：low end 是按**打分时刻**的残差选出来的，它们在 $\theta^*$ 处是否仍然很小，可以用 DLinear 的闭式解零成本验证。这个结论**不能**代替随机对照（E2）。
3. high end 的偏移同样由它们在 $\theta^*$ 处的大残差决定，与 T5 的分析一致。
4. 关于按 batch 归一化：如果实现是按 batch 内的 $\sum w$ 归一化，因子 $1/(1-r_b)$ 与 batch 的梯度相关，本质上是一种**重加权**，不是纯步长缩放。window 单元使 $\mathrm{Var}(r_b)$ 达到最大值 $r(1-r)/B$（Bhatia–Davis 不等式）。**论文应写明实现采用哪种归一化。**

#### T3d. 命题 8″：理想掩码、一步掩码、不动点掩码与 L1、Huber（取代 v3 的命题 8′）

**模型**：$Y=\mu(x)+\xi+a$，$a$ 为稀疏污染，$P(a\ne0)=\varepsilon$。以位置模型为例说明，高斯 $\xi$，r=10%（两侧各 5%，$c=1.645\sigma$）。

**三种掩码估计量必须分开**：
1. **理想掩码**（恰好覆盖污染支撑，并且与 $(x,\xi)$ 独立）：偏差为 0，方差约为 $\sigma^2/((1-\varepsilon)n)$（线性模型、同方差时为 $\sigma^2(Exx^\top)^{-1}/((1-\varepsilon)n)$）。
2. **一步掩码（论文的做法）**：按打分模型 $f_0$ 的残差截断后，用 LS 重新拟合。影响函数为
$$\mathrm{IF}(x)=\frac{\psi_c(x)}{1-2\alpha}+k\,\mathrm{IF}_{f_0}(x),\qquad \psi_c(u)=u\mathbf 1\{|u|\le c\},\quad k=\frac{2cf(c)}{1-2\alpha}\approx0.377 .$$
   - 打分模型是 LS 型（包括早停的 GD）时，$\mathrm{IF}_{f_0}(x)=x$ 无界，**一步掩码不是 B-稳健的**：离群点先把打分中心推偏，截断窗口随之平移，掩码估计按比例继承这个偏移。这与命题 16 是同一个机制。
   - **单侧污染下的偏差约为 $s(b_0)\approx k\,b_0$**，其中 $b_0$ 是打分模型吸收的污染（总体 LS 打分时 $b_0=\varepsilon\bar M$）。确定性计算：ε=1%、M=10 时掩码偏差为 0.035，L1 为 0.013，Huber 为 0.017。一阶近似下，**掩码偏差在 $\bar M\gtrsim3.3\sigma$ 时超过 L1，在 $\bar M\gtrsim4.3\sigma$ 时超过 Huber。**
   - **对称污染下**的渐近方差（以 σ²/n 为单位，ε=1%）：M=5 时论文掩码为 1.33，**比不加掩码的 LS（1.24）还差**；M=10 时为 1.43（LS 为 1.99，Huber 为 1.09，L1 为 1.60）。
3. **不动点掩码**（反复重新打分直到收敛，类似 LTS 的 C-step）：ψ 为硬拒绝、重下降型，IF 有界；但在高斯下渐近方差约为 $1.78\sigma^2$（效率约 56%）。

**L1、Huber 的偏差系数**（单侧远端污染，高斯 ξ）：
- L1 约为 $\varepsilon/(2f(0))\approx1.25\varepsilon$；
- Huber（$c=1.345$）约为 $\varepsilon c/((1-\varepsilon)(2\Phi(c)-1))\approx1.64\varepsilon$；
- 两者都与 $M$ 无关。在对称单峰 F 的 ε-污染邻域中，中位数是平移等变泛函里**最大偏差最小**的，见 Huber (1981) 或 Huber & Ronchetti (2009) 关于 minimax bias 的章节（写入论文前请核对章节号）。

**效率**：Huber（$c=1.345$）在**无污染**高斯下效率为 95%，ε=1%、5% 时渐近方差约为 1.09、1.26；L1 为 $\pi/(2(1-\varepsilon)^2)$。

**测试集被非对称污染时**：总体 MAE 的最小化者是被污染分布的中位数。预测干净中心会多出约 $\varepsilon^2/(4f_\xi(\mathrm{med}))$ 的 MAE（一阶近似；ε=0.1 时低估约 11%）；总体 MSE 的最小化者是被污染分布的均值，预测干净中心的代价约为 $(\varepsilon\bar M)^2$。

**对 E1 的理论预测（条件形式）**：
- **若**打分模型在 10% epoch 时尚未吸收污染（$b_0\approx0$），掩码偏差约为 0，掩码在非对称污染下有偏差优势；**否则**偏差约为 $k\,b_0$，大幅污染时可能**不如** L1 和 Huber。这个前提可以零成本检验：看打分模型在**未屏蔽**项上残差的带符号均值是否接近 0。
- 对称污染、轻尾噪声下，**理论预测 Huber 优于论文的掩码**（不是持平）。
- 两种补救可以写进讨论：(i) 用稳健的打分中心。以 α 与 1−α 分位数的中点作初值时，一步截断与截断均值渐近等价（Ruppert & Carroll, 1980），IF 有界。(ii) 再做一次重新打分，继承的偏差大约每轮乘以 $k$，代价是向不动点的高方差靠近。

#### T3e. 命题 16：掩码模型被拉向打分模型（附完整条件）

**设定**：occurrence 单元；在 $x$ 处 $Y=f^*(x)+\xi$；全局阈值 $c=\sqrt\tau$；打分误差为 $b_0(x)$；模型类包含 $x\mapsto E[Y\mid x,\text{保留}]$。

**命题 16**：
1. 若 $P(|\xi-b_0(x)|\le c)>0$，掩码 LS 在 $x$ 处的总体目标为 $f^*(x)+s(b_0(x))$，其中 $s(b)=E[\xi\mid|\xi-b|\le c]$。
2. **下界** $s(b)\ge0$（$b\ge0$）只需要 $\xi$ 关于 0 对称；**上界** $s(b)\le b$ 还需要**单峰**（$|v|$ 越大 $f(v)$ 越小）。两者对所有 $b\ge0$ 成立，包括 $b\ge c$；$b<0$ 时对称成立。反例：双峰分布会使 $s(b)>b$；有界支撑可能使窗口内概率质量为 0，此时 $s$ 无定义。
3. 局部斜率为 $s'(0)=\dfrac{2cf(c)}{1-2\alpha}$（需要 $f$ 在 $\pm c$ 处连续）。同方差高斯、两侧各截断 5% 时为 **0.377**。**异方差时**局部斜率为 $k(z)=\dfrac{2z\varphi(z)}{2\Phi(z)-1}$，$z=c/\sigma(x)$；$z$ = 0.25、1、1.645、2.5、3.5 时分别约为 0.98、0.71、0.38、0.09、0.006。**噪声大的区域几乎完全继承打分模型的偏差。**
4. 若测试的条件分布等于训练的干净分布，且 ξ 对称、均值存在，则总体 MSE 增加 $E_x[s(b_0(x))^2]$（与注记 0 一致）。∎

**含义（方向性，需检验）**：
- 它是命题 6′ 中位置偏移的一个具体来源：截断窗口以 $f_0(x)$ 而不是 $E[Y\mid x]$ 为中心。即使噪声对称、没有污染也会出现。
- **峰值处**（$b_0$ 大）继承的比例反而更高：高斯、$c=1.645$ 时，$b$=1、2、4 对应的 $s(b)/b$ 为 0.42、0.52、0.67。但保留概率趋于 0（0.74、0.36、0.009），这时该处的预测实际由参数共享和插值决定，"灵活模型"的结论不可靠。所以"尾部变差"只是一个方向性的假设。
- v3 中用 MC-dropout 解释表 3 的说法已删除：MC 平均预测约等于关闭 dropout 时的预测，$b_0$ 几乎不变。
- **可检验预测 P15（修正）**：在测试点上，把 $(f_{\text{mask}}-f_{\text{full}})$ 对 $(f_0-f_{\text{full}})$ 做回归，预测斜率为正（$|b_0|$ 小时约为 $k$，噪声大的区域更大）。参照的全数据模型 $f_{\text{full}}$ 要用**另一个随机种子**，以消除"全数据运行就是打分运行的延续"带来的混杂。

---

### T4. 系统抽样与覆盖：§7.3 的理论（★★）

**程式化模型（明写）**：$r_i(t,d)=g(t,d)+\psi(h)+\zeta_{i,t,d}$，其中目标难度 $\bar g(t)=\frac1D\sum_dg(t,d)$ 在时间上独立同分布，$\psi$ 为步长效应，$\zeta$ 为独立的 occurrence 噪声。这时逐窗口损失 $\ell_i$ 的自相关在滞后 $1\le u\le T-1$ 上与 $1-u/T$ 成正比（有 ζ 时在 0 处有一个跳跃，即 nugget，凸性不变），在 $u\ge T$ 时为 0。**步长 × 目标的交互，或者上下文效应，都会破坏这个结构。**

**Cochran (1946) 定理**：超总体均值、方差为常数，自相关非负、非增、凸 ⇒ 系统抽样（随机起点）≤ 分层抽样 ≤ 简单随机抽样（期望方差意义下）。**写入论文前，必须对照原文（以及 Cochran, 1977, 第 8 章）核对条件的准确范围。** 三角形自相关满足这些条件。

**命题 9a（新增，替换 v3 中"增益可能不大"的错误说法）**：在上述模型下，若间隔 $s$ 整除 $T$，则系统样本中每个内部时刻**恰好**被覆盖 $T/s$ 次。所以系统样本均值对每个 $\bar g(t)$ 的权重与总体均值完全相同，误差只来自序列两端（约 $2T$ 个边界时刻）。以 $\ell_i$ 的方差为单位，$EV_{sy}=O(1/N_{\text{win}}^2)$ 且与 $T$ 无关，而 $EV_{st}=O(s/(N_{\text{win}}T))$，所以
$$EV_{sy}/EV_{st}\approx T/N_{\text{win}}$$
（ETTh 规模下精确计算为：T=96 时 0.011，T=720 时 0.085）。也就是说，**系统抽样比分层抽样好一到两个数量级**；$s\nmid T$ 时两者同阶。论文 §7.3 的 50% 删除（$s=2$）在所有 $T\in\{96,192,336,720\}$ 上都满足 $s\mid T$。

**命题 9 的其余部分（附条件）**：
1. **比例**：利用恒等式 $\bar\ell_K-\bar\ell=-\frac{n_R}{n_K}(\bar\ell_R-\bar\ell)$，删除比例或保留比例为 $1/s$ 时都可以套用定理；30%、70% 不在定理范围内。另外，§7.3 是在 element 掩码**之上**剪窗，掩码损失 $\sum_KN_i/\sum_KW_i$ 是**比率估计量**，应对线性化后的 $z_i=N_i-\hat RW_i$ 套用定理。§7.3 用的是 iTransformer；关于"训练结果"的推论只能作为启发式。
2. **周期混叠**（只对"**保留** 1/s"的情形）：$s=P$ 时系统样本总落在同一相位，误差不随样本量减小。在覆盖层面，固定步长时只能看到 $1/\gcd(s,P)$ 的相位；**只有在模型设定错误时**才会因此产生相位偏差（若输入已携带相位信息，例如 $L\ge P$，则不会）。
   - 论文的 50% 和 90% 删除，在 P=24、96、144 上都有 gcd=2；
   - 补救"取 $\gcd(s,P)=1$"会排除 50%，所以**推荐的补救是每个 epoch 重新随机抽取起点**。
3. **"P 整除 T 时周期成分抵消"**只对**加性、只依赖目标时刻**的周期难度成立，步长 × 相位的交互不会抵消。$P\mid T$ 的情况：日周期 P=24 对四个 $T$ 都成立；周周期 P=168 只对 T=336 成立；ETTm 的 P=96 对 T=336、720 不成立；weather、solar 的 P=144 只对 T=720 成立。
4. **覆盖引理（已穷举验证）**：
   - 整数间隔 $s$：每个内部 element 的覆盖数为 $\lfloor T/s\rfloor$ 或 $\lceil T/s\rceil$（$s>T$ 时也成立）；
   - **非整数比例 $f$**（用 Beatty 序列 $\lfloor(j+\theta)/f\rfloor$ 保留）：覆盖数为 $\lfloor fT\rfloor$ 或 $\lceil fT\rceil$，覆盖方差 ≤ 1/4，这就覆盖了 30% 和 70%；
   - 随机保留：覆盖方差约为 $Tf(1-f)$；
   - 连续删除长度 $\ell\ge T$ 的一段：$(\ell-T+1)$ 个**时刻**，即 $(\ell-T+1)D$ 个 element，完全失去监督。
5. **按得分选窗**：这是确定性选择，带来的是**偏差**（偏向规则所保留的那一侧）和**覆盖空洞**，不是方差。高分窗口连续段的**平均**长度约为 $\frac{Q(u)}{\varphi(u)}\frac{\sqrt T}{E[(Z_1-Z_2)^+]}$（高斯难度下 top-10% 约为 $1.01\sqrt T$，精确到 0.06%）。但段长分布**重尾**（中位数只有约 1–2），均值由少数长段贡献。论文应报告经验分位数，或直接报告 $P(\text{段长}\ge T)$，后者对应覆盖空洞。

---

### T5. 线性模型中的"单元 × 结构 × 预算"律（★★，修正版）

**放在哪里**：新增 §8，主文写陈述，附录给证明。对 DLinear 是精确的，因为它的函数类就是对原始窗口做仿射回归；对其他骨干只作为预测。

**设定与条件**：
- **(0) A1：输入未被污染**，即污染只影响目标角色。**这是定理的条件**：没有 A1 时，$H$、$\gamma$、$B_0$ 都依赖 $M$，证明的每一步都不成立。
- 每个步长 $h$ 是一个独立的 LS 回归。观测目标为 $y_t=y^*_t+Mu_t\mathbf 1\{t\in A\}$，$|u_t|=1$。
- 打分模型是 $y$ 的**线性平滑器** $\hat y=Hy+(\text{与 }y\text{ 无关的项})$：OLS，或者从固定初值、以固定步长和步数做的 GD（DLinear 的 GD 应写成预条件形式，见 T7）。Adam 不在此列，只能近似。
- 定义 $B_0=\|(H-I)y^*\|_\infty$（加上与 $y$ 无关的项）；对每个块，定义 $q_B=|B|^{-1}\|((H-I)u_A)_B\|_2^2$。

**定理 10′（块数口径）**：记 $b(r)=\lfloor rN_{\mathcal P}+\tfrac12\rfloor$，$N_A$ 为与 $\text{occ}(A)$ 相交的块数。假设
- (0) A1；
- (ii′) 对每个步长 $h$ 以及规则可能产生的每个保留集，第 $h$ 个回归的保留设计矩阵 $X_{K_h}$ 满列秩（**逐步长陈述**；occurrence 掩码可能删光某些步长的大量行）；
- (iii) 对所有可能的剩余污染配置，$X_K^\top u_R\ne0$；
- (iv) 测试协方差 $\Sigma\succ0$。

则当 $M\to\infty$ 时：
- **(a)** 若 $b(r)<N_A$，掩码估计的超额风险为 $\Theta(M^2)$。**这一方向不需要任何排序条件**：至少有一个被触及的块留了下来，其中至少有一个污染项。
- **(b)** 若 $b(r)\ge N_A$，并且排序条件
  $$\text{(i*)}\qquad \min_{B\cap\text{occ}(A)\ne\emptyset}q_B\ >\ \max_{B\cap\text{occ}(A)=\emptyset}q_B$$
  成立，则超额风险对 $M$ 一致有界（可能的保留集只有有限个）。
- v3 的泄漏条件 (i) $\gamma_A+\sqrt{m_{\max}}\gamma_C<1$ 是 (i\*) 的**充分条件**（因为 $q_{\text{touched}}\ge(1-\gamma_A)^2/m_{\max}>\gamma_C^2\ge q_{\text{clean}}$），但不是必要条件。
- 对等大小的块，$b(r)\ge N_A\iff r\ge\rho_{\mathcal P}-\frac{1}{2N_{\mathcal P}}$，约等于 $r\ge\rho_{\mathcal P}$。
- **对 element**，按块数计的阈值是 $\rho_{\text{elem}}\cdot N_{\text{win}}/(N_{\text{win}}+T-1)$（ETTh、T=720 时为 $0.916\rho$）。

*证明要点*：
- 污染项满足 $|e_\omega|\ge M(1-|(Hu_A)_\omega|)-B_0$，干净项满足 $|e_\omega|\le B_0+M|(Hu_A)_\omega|$。所以当 $M\to\infty$ 时，块得分除以 $M^2$ 收敛到 $q_B$，(i\*) 保证被触及的块全部排在前面；
- 其余部分见 R3-C 报告的 T5-3 至 T5-6。∎

**v3 中"窗口单元失效的第二个机制"已删除**：截距造成的泄漏对所有块几乎均匀，不改变排序。ECL 规模下有确定性反例：(i) 不成立，但 (i\*) 成立，窗口单元在 r=10% 下中和了污染。在随机污染模型下，截距泄漏要破坏 (i)，需要 $\varepsilon>1/\sqrt{m_{\max}}$；而此时几乎所有窗口都已经被触及，窗口单元早已因为预算不足而失败。

**推论 11（期望所需预算）**：每个 element 独立地以概率 ε 被污染时，一个块被触及的概率**精确**为 $1-(1-\varepsilon)^{m_P^{\text{el}}}$，所以被触及块比例的**期望**等于它。实现值很集中：ETTh1、T=96 时，channel row 的标准差约为 1.9 个百分点，window 约为 1.4 个百分点。

| 单元 | $m_P^{\text{el}}$ | 期望所需预算（ε=0.5%，D=7，T=96） | 目标侧、A1 下 r=10% 能否中和 |
|---|---|---|---|
| occurrence、element | 1 | 0.5% | 能 |
| time column | 7 | 3.45% | 能 |
| channel row | 96 | 38.2%（T=720 时 97.3%） | 不能 |
| window | 672 | 96.6% | 不能 |

**重要脚注**：这张表**只是目标侧、A1 下**的结论。随机 element 污染**必然也污染输入**，而输入侧所需的预算至少是 38.2%（见下），所以**在 r=10% 下，没有任何单元能完全中和这种污染**。

**推论 12**：
- 跨通道同时刻事件：time column 的块全是污染项，κ=1，与 occurrence、element 等效 ✅。
- 起点带：在最简的 0/1 模型下，$\kappa_{\text{chan}}=\kappa_{\text{elem}}$，并且两者块均值的多重集也相同（关于反对角线对称）。channel row 的优势需要额外假设"**残差幅度只随窗口 $i$ 变化、在同一行内与 $t$ 无关**"，这需要在真实数据上检验。

**输入侧污染（去掉 A1）**：
- **衰减**：在以下条件下，OLS 系数的概率极限为 $(\Sigma_x+\Sigma_\delta)^{-1}\mathrm{Cov}(x,y^*)$（设定正确时为 $(\Sigma_x+\Sigma_\delta)^{-1}\Sigma_xw^*$），其中 $\Sigma_\delta=\varepsilon M^2I$（符号对称时），或 $\varepsilon(1-\varepsilon)M^2I$（同号、均值由截距吸收时）（Fuller, 1987）。条件为：点尖峰在时间上独立；与信号和目标噪声独立；模型有截距；$N_{\text{win}}\to\infty$（**这与定理 10′ 的"样本固定、$M\to\infty$"是不同的渐近，不要混用**）。在特征方向 $j$ 上的衰减因子为 $\lambda_j/(\lambda_j+\varepsilon M^2)$，特征值小的方向衰减最严重。
- **所需预算**：通道独立时，一个 channel row 的输入含尖峰的概率为 $1-(1-\varepsilon)^L$（L=96、ε=0.5% 时为 38.2%）；通道混合时为 $1-(1-\varepsilon)^{LD}$。
- **更根本的问题（新增）**：打分模型同样在被污染的输入上训练。输入在滞后 $j$ 处含尖峰的行，其残差中的尖峰项为 $Mw_j$，在 $M\to\infty$ 时趋于 0。所以 **$M$ 越大，基于残差的得分越认不出输入被污染的行**。38.2% 只是**必要**预算，high-end 掩码并不会把这些行选出来。
- element 只能以 $(L+T-1)/L$ 倍的附带损伤间接覆盖输入足迹，而且前提是这些项确实进入了高分集合。
- **E10 的条件**应逐特征方向陈述："$\varepsilon M^2\gtrsim\lambda_j$，其中 $j$ 为 $w^*$ 主要分布的特征方向"。

**P14（修正）**：在**注入已知 $A$** 的实验中，用闭式 OLS 计算每个块的 $q_B$，检验 (i\*)。没有注入时 $A$ 未知，无法检验。

---

### T6. 风险形状与预算（★）

**零均值污染**（剩余比例 $\varphi=(1-r/\rho)_+$，块内污染数均匀）：
$$R(r)\approx\frac{p}{N}\cdot\frac{\sigma^2+M^2\varepsilon\varphi/(1-r)}{1-r}.$$
**同号污染**（与 $X$ 无关、进入截距）：偏差项约为 $M^2\varepsilon^2\varphi^2/(1-r)^2$。

**命题 13（含严格证明）**：在零均值模型下，记 $g(r)=\dfrac{\sigma^2}{1-r}+\dfrac{M^2\varepsilon(1-r/\rho)}{(1-r)^2}$（$r\in[0,\rho]$）。则 $g'(r)=N(r)/(\rho(1-r)^3)$，其中 $N(r)=\sigma^2\rho(1-r)-M^2\varepsilon(1+r-2\rho)$ 关于 $r$ 线性递减。所以 $g$ 在 $(0,\rho)$ 内部没有极小值；$r>\rho$ 时 $g=\sigma^2/(1-r)$ 递增。因此**全局最优点只能是 $r=0$ 或 $r=\rho$**，并且
$$R(\rho)<R(0)\iff M^2>\frac{\sigma^2\rho}{\varepsilon(1-\rho)} .$$
- 数值：ETT 上的窗口单元，ε=0.01% 时 $\rho=6.5\%$，需要 $M\gtrsim26.4\sigma$；ε=0.5% 时 $\rho=96.6\%$，需要 $M\gtrsim74.9\sigma$（屏蔽 96.6% 本身就没有意义）。
- 注意：$\rho>1/2$ 时，在 $[0,2\rho-1]$ 上污染项随 $r$ **递增**，因为屏蔽让分母缩小得比移除污染更快。
- $p/N$ 渐近式是否可用，取决于 DLinear 的设置：shared 设置下 $p/N_K\approx0.05$，可以用；individual 设置下，$\rho=96.5\%$ 时 $p/N_K\approx0.33$，精确因子约为 1.5，渐近式会明显低估。

**Tukey 自适应预算**：超出 fence 的比例为 $\hat\varepsilon=(1-\varepsilon)\pi_c+\varepsilon P_{\text{cont}}(s>\text{fence})$。
- ε<1/4 只是**必要**条件；ε 较大时分位数本身会移动。
- 即使在最理想的高斯情形，$\pi_c=0.35\%$，也**不满足** $\pi_c\ll\varepsilon=0.5\%$。
- **element 得分不是近似高斯的**：同一 element 的各项**共享目标噪声**，所以 $\pi_c$ 介于约 0.8%（模型误差占主导）与 7.6%（目标噪声占主导，此时得分就是缩放的 χ²₁）之间，再加上尺度混合会更大。
- "上偏"只在分离条件成立时才成立；一般地 $\hat\varepsilon-\varepsilon=(1-\varepsilon)\pi_c-\varepsilon P_{\text{cont}}(s\le\text{fence})$，符号不定。
- **新预测（P12）**：即使没有任何异常，Tukey 预算也可能屏蔽数个百分点的 element。可以用 §7.3 中自适应预算的实际屏蔽比例直接检验。

---

### T7. 早期得分（★，修正版）

**GD 递推**：本节记 $\tilde e=y-\hat y$（与全文的 $e$ 符号相反，平方之后没有区别）。设 $0<\eta\le1/\lambda_{\max}$，则
$$\tilde e_{K_{\text{gd}}}=\sum_{\lambda_j>0}(1-\eta\lambda_j)^{K_{\text{gd}}}\langle u_j,y-Xw_0\rangle u_j+P_\perp y .$$
非零初值不改变 $P_\perp y$（因为 $Xw_0$ 在列空间中），但**暂态项**要用 $y-Xw_0$。按 $1/L$ 初始化时，早期得分中的慢信号是相对于"窗口均值预测器"的残差。

**命题 14（谱截断）**：早期得分 = **样本内**不可解释部分 $P_\perp y$ + 尚未学到的慢信号 + 指数小的余项。记 $\Delta\ge\|\tilde e_{K_{\text{gd}}}-P_\perp y\|_\infty$，则：
- 按 $|e|$ 排序时，只要参照序列第 $b$ 与第 $b+1$ 名之间的间隔大于 $2\Delta$，top-b 集合就不变；
- 按平方残差的块均值排序时，块均值间隔大于 $2\Delta(2\|P_\perp y\|_\infty+\Delta)$ 是充分条件。

**DLinear**：
- 函数类等于对原始窗口做仿射回归；
- GD 是预条件 GD，预条件矩阵为 $\mathrm{blockdiag}(P,2)$，其中 $P=(I-A)^\top(I-A)+A^\top A\succ0$，截距的系数是 2。步长条件应改用 $\lambda_{\max}$ of $\tilde X\tilde P\tilde X^\top/N$；
- 若使用只减均值的 RevIN，函数类会变成"行和为 1"的受限仿射类，闭式解要相应修改（TSLib 的 DLinear 是否使用 RevIN，需要核对版本）；
- **TSLib 默认用 Adam**，所以早期动力学的谱刻画只是近似。

**与 §7.1 的关系（修正）**：
- $P_\perp y$ 是**样本内**残差，那次拟合见过该监督。论文 §7.1 的 unexplained error 定义为**交叉拟合**残差（在没见过该监督的那次运行上读取），并且是在 **iTransformer** 上计算的。
- 在 DLinear 上，与之对应的是交叉拟合残差的闭式解 $e^{cv}_S=(I-H_{SS})^{-1}e_S$（逐步长使用，要求 $I-H_{SS}$ 可逆）。普通 GD 下，"训练 $K_{\text{gd}}$ 步后的交叉拟合残差"也有闭式解。
- 所以 DLinear 的计算给出的是 §7.1 分解在**线性骨干上的类比**，可以回答"在 DLinear 上、在收敛点上，这个结论是否成立"，**不能复现 §7.1**。

**附带假设**：线性 LS 的影响函数无界，而早停的过参数化网络对标签噪声有一定的隐式鲁棒性（Li, Soltanolkotabi & Oymak, 2020；Liu et al., 2020）。所以线性骨干可能从掩码中获益最多，这与表 5 一致。**只作为假设。**

---

### T8. 截断何时有益；与 Sorscher 的关系（☆，修正版）

**命题 15（位置模型；对称 F；两侧各截断 α，$c=F^{-1}(1-\alpha)$）**：

(a) **按分位数截断**（截断均值；在回归中对应 Koenker & Bassett (1978) 基于回归分位数的截断 LS，其渐近性质由 Ruppert & Carroll (1980) 给出）：
$$V_\alpha(F)=\frac{\int_{-c}^{c}x^2\,dF+2\alpha c^2}{(1-2\alpha)^2},$$
高斯、r=10% 时为 1.026。

(b) **按初步估计 $\hat\theta_0$ 的残差截断**（论文的做法）：要求 F 对称、$\hat\theta_0$ 为 $\sqrt n$ 相合、$f$ 在 $\pm c$ 处连续。此时样本分位数 $\hat c$ 的一阶贡献为 0，影响函数为
$$\mathrm{IF}=\frac{\psi_c(x)}{1-2\alpha}+k\,\mathrm{IF}_{\hat\theta_0}(x).$$
也就是把截断均值 IF 中的 Winsor 项**换成**了 $k\,\mathrm{IF}_{\hat\theta_0}$。以 α 与 1−α 分位数的中点作初值时恰好还原截断均值，这对应 Ruppert & Carroll (1980) 的等价性结论。
- 以样本均值作初值时，渐近方差约为 **1.30σ²**。这是本文自己算的数，**不是 Ruppert & Carroll 给出的结果**。
- **量级提醒**：这是**估计量的渐近方差**，它对测试 MSE 的贡献是 $O(p/N)$，在大样本下相对 σ² 可以忽略。不能说成"测试 MSE 差 30%"。
- Ruppert & Carroll (1980) 的结论目前只在摘要层面核实过；**写入论文前必须阅读全文并准确转述**。

(c) **重尾**时截断可以优于 LS（以 LS 为单位）：
- Laplace：(a) 为 0.827，(b) 为 0.795；
- t3：(a) 为 0.589，(b) 为 0.571；
- ν ≤ 2 时 LS 的方差为无穷，由于 $k>0$，(b) 也为无穷，而 (a) 仍然有限。

(d) **打分模型不相合**（论文的情形）：$k$ 换成 $s'(b_0)$，$\hat c$ 的一阶贡献不再为 0，另外还有命题 16 的 $O(1)$ 偏差项。

**结论**：
- 轻尾、无污染、对称噪声下，论文的掩码**预计受损**，主要来自 $O(1)$ 的偏差项（命题 16），方差项只是 $O(p/N)$；
- 重尾或有污染时可能获益；
- 这只对 occurrence 严格成立；
- 论文表 1 中七个数据集、五个单元的 MAE **全部改善**。按这一理论，要么没有一个数据集是"干净轻尾"的，要么位置模型的分析漏掉了主导效应。论文应如实说明这一点。

**与 Sorscher et al. (2022) 的关系（启发式，修正）**：
- **改变估计目标的情形有三种**：噪声非对称（命题 6′）、打分模型有偏（命题 16）、模型设定错误且屏蔽比例随 $x$ 变化（x 重加权）。在设定正确、噪声对称、打分模型无偏时，屏蔽不改变目标，只改变方差。
- Sorscher 的核心变量是**数据是否充足**（数据少时保留易例，数据多时保留难例），以及打分指标的质量；论文 p.3 自己引用的正是这一点。任何"统一"都必须包含数据量这一维。
- 一种启发式的对应：Sorscher 中的"难"是 $x$ 的性质（margin 小，信息量大）；论文的早期得分大致是 $b_0(x)+\xi$。其中 $b_0$ 是 $x$ 的性质，对应"难但可学"；$\xi$ 是噪声，在高斯下不带信息。剪枝方向取决于得分主要由哪一部分驱动：主要由 $b_0$ 驱动时，屏蔽会丢掉信息，并引起命题 16 的拉动；主要由重尾 ξ 或污染驱动时，屏蔽有益。论文 §7.1 发现得分几乎等于不可解释误差，支持后一种情形。**这一段只能标为启发式。**

---

## 4. 不建议加的理论

| 方向 | 原因 |
|---|---|
| 依赖数据下的泛化界 | 结果松，与主张关系弱，容易被挑错 |
| 深度网络的 NTK 或平均场分析 | 工作量巨大，与实际训练相差太远 |
| 证明"早期得分对测试误差最优"或"某单元对测试误差最优" | 都不成立。T1 只在 κ 和移除质量意义下给出 occurrence 的最优性，这与测试误差无关（数据中 occurrence 并非处处最好） |
| 用 v1 的可加模型报告"各效应的方差份额" | 不可识别（T2） |
| 用"无结构"的 $1/m$ 作为 η² 的基线 | 对预测残差无效（T2） |
| 子模优化框架下的"最优掩码" | 移除质量是模块化函数，前缀已经精确最优（引理 2） |

---

## 5. 放在论文哪里

```
§3 Pruning the supervision
   3.5 Units as partitions        ← T1（引理 1、2，推论，命题 3 + κ 表 + 偏序图）
   3.6 What each unit sees        ← T2（质量捕获比、步长剖面恒等式、对 φ 的暴露、双重足迹）
§5 Which end, which unit
   + "What masking changes, exactly" ← 命题 7′（线性 LS 的精确偏移式），引出随机对照 E2
   + "Is it just a robust loss?"     ← 命题 8″（一步掩码 vs L1/Huber，条件形式）+ E1
   + "Gain vs removed mass"          ← T1 推论 + E4
§6 Where the gain lives
   改写 "Why MSE moves less"          ← 注记 0 + T3a（恒等式与精确分解）+ 命题 6′ + 命题 16
§7 A closer look
   7.1 + 早期得分                     ← T7（谱截断；DLinear 上 §7.1 的闭式类比）
   7.3 + 抽样与覆盖                   ← T4（命题 9a、覆盖引理、周期混叠与每 epoch 重抽）
§8 A linear analysis（新增）          ← 定理 10′、推论 11、12、输入侧局限 + T6
§9 Discussion                        ← T8（启发式），以及局限
附录 E：全部证明；附录 F：理论预测在真实数据上的检验汇总
```

---

## 6. 可检验内容

### 6.1 实现检查（定理推论，数据不可能违反，只用来查实现错误）

- **C1**：在相同实际项数下，移除质量满足细化偏序（T1）。
- **C2**：DLinear 上命题 7′ 的偏移式与实际重新拟合的结果一致。
- **C3**：分位数恒等式在单次运行上精确复现 ΔMAE 和 ΔMSE。

### 6.2 可证伪预测（全部在真实数据上检验）

| # | 预测 | 来源 | 检验方式 | 成本 |
|---|---|---|---|---|
| P2 | 基于 occurrence 级 $S_r$ 的对齐诊断：在不可比的单元对（time 对 chan、element 对 chan）中，κ 较小的单元收益较大（逐骨干） | T1 | E0.4 | 零 |
| P3 | 质量捕获比 $\mathrm{MCR}_{\mathcal P}$ 越高，该单元收益越接近 occurrence（逐骨干） | T2 | E0.4 | 零 |
| P4 | φ 递增时，occurrence 和 time column 的掩码偏向远步长；element 掩码偏向训练段末尾的端点元素；按步长设上限后（同时用于 occurrence 和 time column），由此带来的收益差消失 | T2 | E0.5、E5.1 | 零或低 |
| P5 | 逐次运行的精确分解显示 ΔMSE 主要由高分位决定；"MSE 动得少"对应 δ(u) 的渐变衰减 | T3a | E0.12 | 零 |
| P6 | "MSE 升、MAE 降"出现在被屏蔽项带符号残差均值（用另一种子的全数据模型、按 $b_0$ 符号分组计算）明显不为零的数据集上 | 命题 6′ | E0 | 零 |
| P7 | 若打分模型在未屏蔽项上的带符号残差均值约为 0，掩码在非对称污染下不劣于 L1 和 Huber；否则在大幅污染下劣于两者；对称污染下 Huber 优于掩码 | 命题 8″ | **E1** | P0 |
| P8 | $s\mid T$ 时等间距删窗相对随机删窗的优势远大于 $s\nmid T$ 时；按得分选窗的 $P(\text{段长}\ge T)$ 明显大于 0 | T4 | E7、E0.5⑤ | 低 |
| P9 | 在真实数据中注入点异常时（目标侧），细单元在 $r\approx\varepsilon$ 处饱和，channel row 和 window 需要 $1-(1-\varepsilon)^{m_P^{\text{el}}}$；由于输入侧污染，所有单元都不能完全恢复干净性能 | T5 | 注入实验 | 低 |
| P10 | 注入跨通道事件时，time column ≈ occurrence；注入水平漂移时，高残差集中在起点带和终点带 | T5 | 注入实验 | 低 |
| P11 | （弱预测，检验力低）在条件尾部较重的数据集上 high-end 收益更大；尾部重量要用按局部尺度标准化的条件残差估计 | T8 | 尾部指数 + 表 1 | 零 |
| P12 | Tukey 预算在没有明显异常的数据集上也会屏蔽数个百分点的 element；E3 中 high end 的风险曲线在 0 或 $\rho_{\mathcal P}$ 处取最优 | T6 | E0.9、E3 | 低 |
| P13 | 在 DLinear 上，10% 得分与闭式交叉拟合残差 $(I-H_{SS})^{-1}e_S$ 的秩相关，以及它在收敛点上是否仍然很高（这是 §7.1 的线性类比） | T7 | 闭式计算 | 零 |
| P14 | 注入已知 $A$ 时，用闭式 OLS 计算的 $q_B$ 满足 (i\*) 的单元，才能在 $b(r)\ge N_A$ 时中和污染 | T5 | 注入实验 | 低 |
| P15 | 测试点上 $(f_{\text{mask}}-f_{\text{full}})$ 对 $(f_0-f_{\text{full}})$ 的回归斜率为正（$f_{\text{full}}$ 用另一个种子） | 命题 16 | 已有预测 | 零 |

---

## 7. 实施顺序

| 周 | 理论工作 | 同步的零成本或低成本检验 |
|---|---|---|
| 1 | T1；注记 0；命题 7′ | C1、C2；E0.4（质量捕获比、对齐诊断） |
| 2 | T2；T3a；命题 6′ | 步长剖面恒等式；端点元素占比；E0.12 |
| 3 | 命题 8″；命题 16；T4（命题 9a、覆盖、混叠） | E1 启动；E7；$\ell_i$ 自相关检验 |
| 4–5 | 定理 10′；推论 11、12；输入侧局限；T6 | 注入实验；E3；E0.9 |
| 6 | T7（DLinear 闭式类比）；T8 | P13、P15 |
| 7 | 统一记号；写附录证明；**核对经典文献原文（见 §8）；由第二位作者独立核对全部证明** | — |

---

## 8. 风险清单与写入论文前必须完成的核对

| 风险或待核对项 | 处理 |
|---|---|
| **Cochran (1946)** 的准确条件（凸性的下标范围、各个不等号各需要哪些条件） | 阅读原文和 Cochran (1977) 第 8 章，逐字核对后再写 |
| **Ruppert & Carroll (1980)** 只在摘要层面核实过 | 阅读全文，准确转述其等价性结论，并说明 1.30 是本文自己的计算 |
| **Huber 关于 minimax bias 的出处** | 核对 Huber (1981) 或 Huber & Ronchetti (2009) 中的章节号 |
| TSLib 配置细节：TimeMixer 是否通道独立；DLinear 是否使用 RevIN；初值；式 (4) 按 batch 还是全局归一化 | 对照论文实际使用的代码版本 |
| 定理 10′ 依赖 A1 与线性平滑器 | 在正文中把 (0) A1 写进条件，并单独讨论去掉 A1 的情形 |
| 理论预测被真实数据否定 | 如实报告，把它写成边界条件 |
| 理论太多冲淡主线 | 主文只放陈述和直觉，证明全部放附录 |
| 新的错误 | 三轮复核中，每一轮都找到了上一轮引入的错误（例如 v3 中命题 5 的"修正"本身就是错的）。**写入论文的最终版本必须由一位熟悉鲁棒统计和抽样理论的合作者，再做一次完整的独立复核** |

---

## 参考文献（Harvard；只列确信存在的文献；标注"需核对"的必须查原文）

- Arpit, D. et al. (2017) 'A closer look at memorization in deep networks', *Proceedings of ICML*.
- Cochran, W.G. (1946) 'Relative accuracy of systematic and stratified random samples for a certain class of populations', *The Annals of Mathematical Statistics*, 17(2), pp. 164–177. （条件需核对原文）
- Cochran, W.G. (1977) *Sampling Techniques*. 3rd edn. New York: Wiley.
- Fienberg, S.E. and Mason, W.M. (1979) 'Identification and estimation of age-period-cohort models in the analysis of discrete archival data', *Sociological Methodology*, 10, pp. 1–67.
- Fuller, W.A. (1987) *Measurement Error Models*. New York: Wiley.
- Hampel, F.R., Ronchetti, E.M., Rousseeuw, P.J. and Stahel, W.A. (1986) *Robust Statistics: The Approach Based on Influence Functions*. New York: Wiley.
- Holford, T.R. (1983) 'The estimation of age, period and cohort effects for vital rates', *Biometrics*, 39(2), pp. 311–324.
- Hounie, I., Porras-Valenzuela, J. and Ribeiro, A. (2024) 'Loss shaping constraints for long-term time series forecasting', *ICML*.
- Huber, P.J. (1964) 'Robust estimation of a location parameter', *The Annals of Mathematical Statistics*, 35(1), pp. 73–101.
- Huber, P.J. (1981) *Robust Statistics*. New York: Wiley. （minimax bias 的章节需核对）
- Huber, P.J. and Ronchetti, E.M. (2009) *Robust Statistics*. 2nd edn. Hoboken, NJ: Wiley.
- Koenker, R. and Bassett, G. (1978) 'Regression quantiles', *Econometrica*, 46(1), pp. 33–50.
- Koenker, R. (2005) *Quantile Regression*. Cambridge: Cambridge University Press.
- Li, M., Soltanolkotabi, M. and Oymak, S. (2020) 'Gradient descent with early stopping is provably robust to label noise for overparameterized neural networks', *Proceedings of AISTATS*.
- Liu, S., Niles-Weed, J., Razavian, N. and Fernandez-Granda, C. (2020) 'Early-learning regularization prevents memorization of noisy labels', *NeurIPS*.
- Madow, W.G. and Madow, L.H. (1944) 'On the theory of systematic sampling, I', *The Annals of Mathematical Statistics*, 15(1), pp. 1–24.
- Mindermann, S. et al. (2022) 'Prioritized training on points that are learnable, worth learning, and not yet learnt', *ICML*.
- Rousseeuw, P.J. (1984) 'Least median of squares regression', *Journal of the American Statistical Association*, 79(388), pp. 871–880.
- Ruppert, D. and Carroll, R.J. (1980) 'Trimmed least squares estimation in the linear model', *Journal of the American Statistical Association*, 75(372), pp. 828–838. （只在摘要层面核实，需读全文）
- Shen, Y. and Sanghavi, S. (2019) 'Learning with bad training data via iterative trimmed loss minimization', *ICML*.
- Sorscher, B., Geirhos, R., Shekhar, S., Ganguli, S. and Morcos, A.S. (2022) 'Beyond neural scaling laws: beating power law scaling via data pruning', *NeurIPS*.
- Zeng, A., Chen, M., Zhang, L. and Xu, Q. (2023) 'Are Transformers effective for time series forecasting?', *AAAI*.
