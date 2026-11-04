# TMLR 审稿报告

**论文**：*Prune the Supervision, Not the Windows: Decision Units for Data Pruning in Time-Series Forecasting*（匿名投稿，14 页）
**审稿人**：独立审稿人（sonnet_high）
**说明**：以下批评均基于论文正文、表格、图和附录。凡是我的推断或假设，均标注为"推断"或"假设"。外部文献只列出我确信存在的。论文自己引用的 2026 年预印本我无法独立核实，因此不依赖它们。

---

## 0. 论文摘要（用自己的话）

滑窗训练（stride=1）下，每个目标值 $y_{t,d}$ 在每个 epoch 中被 $c(t,d)\le T$ 个不同窗口各监督一次。一个窗口的损失是 $T\times D$ 个平方误差项之和（Eq. 1）。作者认为，现有时间序列"数据剪枝"沿用了分类中"样本 = 决策单元"的做法，把整个窗口作为剪枝单位。

论文固定评分和预算，只改变"决策单元"。共五种单元（Eq. 2）：

- 窗口 $i$：$T\times D$ 项。
- 时间列 $(i,t)$：$D$ 项。
- 通道行 $(i,d)$：$T$ 项。
- 单次出现 $(i,t,d)$：1 项。
- 元素 $(t,d)$：跨所有覆盖它的窗口的 $c(t,d)$ 项。

评分方法如下（Sec. 3.3）：

- 在一次普通的全数据训练过程中，训练到 10% 的 epoch 时，对所有训练窗口做一次评估（关闭 dropout、不更新参数），记录每个 loss 项的平方残差。
- 单元得分 = 其所含项的平均残差。
- 每个单元屏蔽（mask）得分最高的 10%（"high end"），或最低的 10%（"low end"）。
- 屏蔽通过 Eq. (4) 的 0/1 权重实现，所有输入数据保留，因此不缩短 epoch。

实验设置：3 个骨干（iTransformer、DLinear、TimeMixer）、7 个数据集（ETTh1/h2/m1/m2、weather、solar、ECL）、4 个预测步长 {96,192,336,720}、5 个随机种子，$L=96$，使用 TSLib 官方配方。推断以数据集为单位（paired t 检验、符号翻转检验、Friedman 检验）。

主要结果：

1. 屏蔽 high end 在所有单元、所有 7 个数据集上降低测试 MAE，平均幅度从窗口的 2.03% 到 occurrence 的 4.72%。屏蔽 low end 从不带来增益（+0.09% 到 +0.49%）。
2. 窗口增益最小。通道行和 occurrence 在每个数据集上优于窗口。
3. 增益集中在误差分布的"主体"（|e| 中位数下降 4.4% 到 12.5%），向 95 分位数衰减。最差 10% 误差的均值（CVaR@90%）无可测变化。因此 MSE 的改善较小，且单元之间在 MSE 上不可区分。
4. 作者推荐通道行作为默认单元。
5. 在 iTransformer 上（Sec. 7）：
   - 该得分与"该元素被屏蔽后的 unexplained error"几乎同序（Spearman 0.998）。
   - 与时间段和时刻匹配的随机屏蔽几乎无增益。
   - 早期单次评估的得分与 MC-dropout、TDDS 等更贵的得分相当。
   - 元素屏蔽在 MAE 上优于 Selective Learning（Fu et al., 2025）。
   - 屏蔽优于用插值"修复"被选元素。
   - 在元素屏蔽之上，按等间距剪窗口可省 30% 的优化步，MSE 变化 +0.30 点。

---

## 1. 主要主张与证据充分性

| # | 主张（出处） | 证据 | 我的评估 |
|---|---|---|---|
| C1 | 只有 high end 值得屏蔽；low end 从不帮助（Abstract, Sec. 5） | Table 1、Fig. 1：high end 的 MAE 收益为 −2.0% 到 −4.7%，low end 为 +0.09% 到 +0.49% | **部分充分**。缺少"随机屏蔽"这一关键对照（只有 Sec. 7.1 在 iTransformer 上对元素单元做了匹配的随机屏蔽）。"low end 无效"在 MSE 训练下几乎是预期结果（见 W2）。"收益来自屏蔽了哪些项而非屏蔽本身"这一表述，需要随机对照才成立。 |
| C2 | high end 屏蔽在所有单元、所有 7 个数据集上降低 MAE | Table 1（MAE 列 7/7） | **数值上被支持**（我核对了 Table 1 均值，如 occ. MAE: (2.40+5.49+4.57+6.26+4.21+8.81+1.31)/7=4.72 ✓）。**统计上被高估**：数据集并不独立（ETTh1/ETTm1、ETTh2/ETTm2 是同一原始序列的不同采样率），有效 n 约 5，且无多重比较校正（W4）。 |
| C3 | 窗口增益最小；单元选择重要（Friedman p=0.002） | Table 2 | **关联得到支持，但"单元"解释被混淆**：更细的单元同时意味着更"尖锐"的选择、被屏蔽的损失质量更大，二者无法分开（W1）。 |
| C4 | 增益在误差分布主体，而非尾部；最差 10% 误差"无可测变化" | Table 2、Fig. 2a、Fig. 3 | **分位数曲线有说服力，"无可测变化"属于缺乏证据，不等于等价**。CVaR 对 full-data 的变化只给了均值，无 CI/等价检验；occ. 为 +0.47，只有 3/7 数据集改善（W5）。 |
| C5 | MSE 较小幅度的改善，且不显著，源于 MSE 对大误差加权 | Table 2、Fig. 2b | 解释合理（推断），但**MSE 是训练目标和社区主指标**，摘要以 MAE 为头条结果有选择性（W3）。同时有更直接的替代解释：损失与指标不匹配（见 W2）。 |
| C6 | 推荐通道行为默认单元 | Sec. 5 末段 | **证据薄弱**。其 MAE、中位数均劣于 occurrence，MSE 上"最低"但单元间 p=0.52，不显著。Sec. 7 的所有机理分析均未针对通道行（W7）。 |
| C7 | 得分≈"其余序列无法解释的误差"（Spearman 0.998） | Sec. 7.1 | **定义含糊，缺少平凡基线**。没有展示 naive 得分（如幅值或局部方差）是否也同样相关（W6）。 |
| C8 | 一次性早期得分近似昂贵得分；优于 Selective Learning | Table 3、Sec. 7.2 | **仅 iTransformer、6 个数据集；对 SL 的复现保真度无法判断**（nominal rate 经"校准"）。EL2N/GraNd 等未比较；"我方得分"本身就是回归版 EL2N（W8）。 |
| C9 | 屏蔽优于修复（插值） | Table 3 末行 | 插值方式很特定（沿通道线性插值），是弱对照；不是对"修复"的充分检验。 |
| C10 | 窗口剪枝可用等间距，无需得分，省 30% 优化步（Sec. 7.3） | 文字描述，无表格/图 | **证据呈现不足**：没有表和图，没有 wall-clock，没有 per-dataset 数据（W9）。 |

---

## 2. 优点

1. **问题切入点有价值。** 在滑窗训练中，"一个目标被 $T$ 个窗口重复监督"，因此剪枝单元不必等于样本。Eq. (1)–(2) 的形式化清晰，概念上也干净：窗口剪枝"移除每个 $T\times D$ 个元素的一次出现，却没有真正移除任何元素"（Sec. 3.2）。
2. **统计推断的态度严谨，比多数同类工作更讲究。**
   - 明确承认横跨步长和骨干的非独立性，以数据集为推断单位。
   - 使用精确符号翻转检验（并指出 7 个数据集时 p 下限为 1/64，该计算正确）。
   - 对"无增益"类主张使用单侧等价式检验（±0.3 点）。
   - 对多个单元使用 Friedman 检验，并如实声明不做多重性校正。
3. **报告了误差分布而不仅是均值**（中位数、q75/q90/q95、CVaR@90%、RMSE）。这个分解很有信息量：它揭示了 MAE 与 MSE 行为的差异。
4. **有"匹配随机屏蔽"控制**（Sec. 7.1，匹配段长和时刻），这是一个相当严格的对照，值得肯定。
5. **有"不受益"的结果和诚实的局限表述。** 例如 ECL 上细粒度单元使 MSE 变差（Table 1），并给出去掉 ECL 的附录 D；承认得分只对"此模型类/此回望/此训练阶段"成立。
6. **逐骨干的附录表格**（Tables 4–6、Fig. 3）使读者可以核对聚合数据。我抽查了 Table 1 与 Table 3 的聚合，与逐骨干表一致。

---

## 3. 缺点与修改建议（按重要性排序）

### W1（关键）：未将"决策单元"的效应与"选择尖锐度/被移除损失质量"区分开，核心结论被混淆

**问题。** 预算以"被屏蔽项数占比 $r=10\%$"定义（Sec. 3.3）。但在相同的项数预算下，不同单元移除的损失质量差异极大：

- occurrence 单元精确选出最大的 10% 个别项。
- 窗口/通道行/时间列单元用均值得分，混入大量低残差项，所选单元里平均只有部分项是"真正高残差"的。
- 元素单元用的是 $c(t,d)$ 项的均值。

因此 Table 2 中 occ. > element ≈ chan. row ≈ time col. > window 的次序，几乎就是"单元越细，选中的项越极端，被移除的损失质量越大"的单调排序，而不一定是"决策单元"本身的效应。论文在 Sec. 5 对元素的解释（"a target with a high mean residual also has occurrences that the model predicts well"，以及"gives up about a quarter of the occurrence's gain"）实际上印证了这个理解，但没有量化。

**为何重要。** 标题和贡献 1/2 的核心命题是"unit 决定收益"。若在"被移除损失质量/被移除项的平均残差"的相同条件下各单元表现相同，则结论应改写为"选择精度"，而不是"决策单元"。同时，"窗口剪枝最差"这一结论对已发表的窗口级方法的含义也会不同。

**建议（补充实验）。**
1. 对每个单元和每个预算，报告被屏蔽项占总损失（$\sum r$）的份额，以及被屏蔽项残差的分位数分布。
2. 做 **matched-mass** 对比：为每个单元调预算 $r_u$，使被移除的损失质量（或被屏蔽项的平均残差）相等，再比较 MAE/MSE。
3. 画"测试 MAE 增益 vs. 被移除损失质量"散点图，所有单元 × 骨干 × 数据集在同一图上。若各单元落在同一条曲线上，则"unit"是代理变量。
4. 增加一个 **oracle-ish 对照**：与 occurrence 屏蔽使用同一组 $K$ 个项，但强制按窗口/通道行聚合出"最接近的"集合（覆盖同一批项），以隔离聚合带来的"附带损伤"（co-masked 低残差项）。

### W2（关键）：缺少最直接的替代解释和基线——这很可能就是"修剪损失 / 鲁棒损失"，而且训练损失为 MSE、评估指标为 MAE

**问题。** 方法本质：以一个固定的 0/1 权重屏蔽残差最高的 10% 项。这就是 **trimmed / least-trimmed-squares 损失**的一个静态版本，与 loss truncation（Kang and Hashimoto, 2020）、small-loss 样本选择（Han et al., 2018）、iterative trimmed loss（Shen and Sanghavi, 2019）、RobustTSF（论文已引用）同类。论文没有把它与鲁棒回归/鲁棒损失联系起来，也没有这些基线。

关键观察：

- 增益集中在中位数/MAE（中位数 −4.4% 到 −12.5%，MAE −2.0% 到 −4.7%）。
- 增益在尾部消失（q95 ≈ 0；CVaR ≈ 0）。
- MSE 增益小且不显著。

这正是"用 MSE 训练但对大残差降权 → 预测更接近条件中位数 → 对 MAE/中位数有利、对 MSE/尾部无利"的预期签名。所以论文的发现（"pruning 帮助典型误差而非尾部"）也可以被一个**与"剪枝"无关**的机制解释：训练损失与评估指标不一致。TSLib 的默认训练损失为 MSE（我对 TSLib 配方的了解如此；论文只写"official per-dataset recipes"，未明确写出，建议核实并写明）。

**"low end 无效"也是预期的，而非"悖论"。** 在 MSE 损失下，梯度与残差成正比。屏蔽最低残差的 10% 项，对梯度几乎没有贡献（被移除的损失质量很小），因此几乎等于"什么都没做"。Fig. 1 中 low end 的 +0.09% 到 +0.49% 因此是"几乎无扰动"，并非"屏蔽没用"的证据。论文据此得出的"gain comes from which loss terms are masked, not from masking as such"（Sec. 5）需要的是**随机屏蔽**对照，而不是 low end。

**建议（补充实验，必须做）。**
1. 增加 **随机屏蔽**对照（在每个单元，同一预算 $r=10\%$ 下随机选，5 个种子），作为主表（Table 1/2）的基线行。
2. 增加 **L1 / Huber / quantile(0.5) 损失训练**的全数据基线，并在该损失下重复"high-end 屏蔽"。如果 L1/Huber 全数据训练已经复现了"MAE↓、中位数↓、尾部≈、MSE↔"的模式，则论文需要大幅度改写；如果屏蔽在 L1 下仍有额外增益，则论文的结论才真正独立于损失不匹配。
3. 增加 **在线 trimmed loss**（每个 mini-batch 动态丢弃残差最大的 10% 项，零额外开销、无需评分运行）。若它与静态掩码相当或更好，则"早期评分一次"的价值和整个流水线开销要重新论证。
4. 增加 **soft 权重基线**（如 Gaussian NLL 的异方差权重，Kendall and Gal, 2017；或残差的逆权重）。这检验"硬屏蔽"是否必要。
5. 在相关工作和引言中引入鲁棒统计文献（Huber, 1964; Rousseeuw, 1984）。

### W3（高）：评估指标的选取与头条表述有倾向性；MSE 结果明显弱于 MAE 结果

**证据。**
- 摘要头条为 MAE。MSE 是 TSF 社区主指标，也是训练目标。
- Table 1：MSE 的"Datasets better"为 window 7/7、time col 6/7、chan row 6/7、occ. 5/7、element 6/7；DLinear 上窗口 MSE 仅 3/7（Table 5）；TimeMixer 上元素 MSE 仅 3/7（Table 6）。
- ECL 上 occurrence 和 element 的 MSE 为 +1.51% 和 +1.75%；solar 上 occurrence 为 +0.91%。
- Table 2：MSE 的跨单元 Friedman p=0.525，且作者自己写"none of these changes is significant across datasets (p = 0.06 to 0.13)"。

**为何重要。** TMLR 看重主张与证据一致。"屏蔽 high end 改善预测"在 MSE 上并未被统计证实，仅在 MAE/中位数上被证实。这一点在摘要里有提及，但头条、结论和推荐（Sec. 8）均向 MAE 倾斜。

**建议。**
- 在摘要和结论中把"MAE 显著、MSE 不显著、且在 ECL/solar 上方向相反"与头条并列，而不是作为次要句。
- 报告 MSE 的 per-dataset 置信区间。
- 解释为什么选 MAE 作为主要指标（若是预先设定的，请说明；若不是，应承认它是在看到结果之后选择的）。

### W4（高）：统计推断——有效样本量被高估、缺少方差报告、多重比较未校正

**证据与问题。**
1. **数据集并不独立。** ETTh1/ETTm1 和 ETTh2/ETTm2 是同一原始电力变压器序列的两种采样率（论文只称它们为"standard datasets"，在 Sec. 4 描述为 hourly vs 15 minutes）。论文声称"以数据集为单位"消除了依赖，但这四个数据集只是 2 个独立源。独立单元约为 5 个（ETT-1、ETT-2、weather、solar、ECL）。因此符号翻转检验的最小 p 值、"7/7"和 Friedman 的 $p=0.002$ 都被高估了显著性。
2. **没有任何种子方差。** 全文表格只报告 5 个种子平均后的百分比变化，没有标准差/CI。许多单元-数据集的差异在 0.1%–1% 量级（如 ETTh1 窗口 −0.11%；weather 窗口 −0.08%/−0.11%）。读者无法判断这些在种子噪声之内还是之外。尤其是每次比较用的是"同种子"配对，但 mask 运行与 full 运行的数据顺序并不相同，种子配对并不等于噪声配对。
3. **多重比较。** 论文声明"p-values without multiplicity adjustment"，但全文有几十个检验（单元 × 指标 × 骨干 × 比较 × 附录 D 版本）。作者在不同位置使用 p=0.045、p=0.04、p=0.036、p=0.027 作为"优于"的证据（Sec. 7.2、7.3），这类边缘结果在合理的校正下会消失。
4. **±0.3 点的等价边界无依据。** 它决定"low end never helps"的结论，但没有说明如何选定、是否在看到数据之前设定。
5. **发散运行被排除（Sec. 4："Runs that diverge are excluded"）**，但没有报告数量、涉及哪些配置，是仅排除该运行还是整个配对。若发散更频繁地发生在 full 或 masked 一侧，则会产生选择偏差。
6. 用"百分比变化的均值"聚合：ETTh2/ETTm2 上 DLinear 的 MSE 变化达 −8.7% 到 −17.8%，这些大数值会主导算术平均与 t 检验。建议使用对数比率并做 bootstrap，或使用秩/中位数汇总。

**建议。**
- 把 ETTh1/ETTm1 和 ETTh2/ETTm2 合并为"配对数据集"再做推断，或做 block-bootstrap，或明确承认有效 n≈5。
- 增加更多独立数据集（Traffic、Exchange、ILI、PEMS 等；至少 Traffic 在 TSLib 中是标准的大规模数据集，却缺席）。
- 报告每个单元-数据集-骨干单元格的种子 SD 和 95% CI（附录中可给出）。
- 说明等价边界的来源，并显示结论对 ±0.2/±0.5 的敏感性。
- 报告被排除的发散运行数量和位置。
- 使用 Holm 或 Benjamini–Hochberg 校正，至少对主表中的主张族做校正；或使用 Demšar（2006）/Benavoli et al.（2017）推荐的多数据集比较方法。

### W5（高）：对"最差 10% 误差无变化"的主张属于"缺乏证据即无证据"，并且可能掩盖分布漂移下的代价

**证据。** Table 2 的 CVaR@90% 行：窗口 −0.27 (4)、时间列 +0.22 (5)、通道行 −0.52 (4)、occ. +0.47 (3)、元素 +0.18 (5)；Friedman p=0.273，该检验比较的是**单元之间**，不是单元相对 full-data。文中说"no unit differs measurably from the full-data run"，但没有给出该检验、CI 或等价检验。摘要写"does not change measurably"。

**问题。**
- 在 n≈7、百分比噪声较大时，检验功效低，"不显著"≠"无变化"。
- 去掉高残差训练目标，直觉上会使与之相似的测试时极端事件预测更差。tail 指标恰好是"屏蔽是否伤害鲁棒性"的探针；作者的 CVaR 变化为 +0.47 对 −0.52，混合。
- 这些高残差训练项可能对应于**非平稳性/分布漂移/季节性断点**（假设）。这些是测试时也会出现的 regime。论文没有区分"噪声/异常"（应屏蔽）与"可学习的 regime 变化"（不应屏蔽）。
- 屏蔽项在时间上的分布未呈现：如果被屏蔽元素集中在训练集末端（与测试期最近），则"去掉最近的 regime"可能带来风险。

**建议。**
- 对 CVaR@90%、q95、q99 报告相对 full-data 的 CI 和等价检验。
- 增加按 regime 的评估：把测试集按"与训练时被屏蔽区域的相似度"或"水平/方差漂移"分层。
- 画出被屏蔽元素在训练时间轴上的密度图（每个数据集）。
- 增加一个带有人为注入异常/outlier 的 sanity 实验（受控设置），检验屏蔽是否确实在去除"噪声"而非"信息"。

### W6（高）：Sec. 7.1 的机理主张（"score ≈ unexplained error"）定义含糊，缺少平凡基线，且论证力度被高估

**证据。** Sec. 7.1："the score is almost a relabelling of the unexplained error (median Spearman correlation 0.998) and only weakly related to the explained error (0.22)"。

**问题。**
1. "unexplained error"和"explained error"没有公式定义，只有文字，且措辞含糊（"the drop from the untrained residual to it is its explained error"）。读者无法复现或检验。
2. 在被评估的早期（10% epoch，若 TSLib 配方为 10 个 epoch，则 ≈ 1 个 epoch——**这是推断，论文没有给出 epoch 数**），模型尚未拟合，残差几乎由目标自身的变异（幅值、局部方差、季节偏离）决定。因此"score 与 held-out 残差 Spearman 0.998"可能仅表明二者都主要反映**目标本身的难度/幅度**。需要平凡得分（如 $|y_{t,d}-\bar y|^2$、局部方差、季节性朴素残差）来检验：若它们同样得到 ~0.99 相关和同样的屏蔽增益，则"模型残差"并未增加信息，"什么被移除"的答案就是"高幅值/高变异性目标"。
3. 跨数百万高度自相关的元素计算 Spearman，再取 20 个组合的中位数，其"0.998"几乎没有推断意义，且这种高度依赖的数据下"中位数"没有给出置信度。
4. 被保留的输入泄漏：被屏蔽的元素仍出现在后续窗口的输入中（作者承认）。作者用"严格 hold-out（同时扣留输入和目标）"给出 0.990，这是个好的补强，但仍只涵盖 9 个组合。
5. 论文自己承认这一得分"belongs to this model class, lookback, and stage of training; neither says that a target is unpredictable in principle"，但标题和摘要仍采用"the error that the rest of the series leaves unexplained"这种几乎是本体论的措辞。

**建议。**
- 给出公式化定义。
- 增加平凡得分基线（幅值、局部方差、季节朴素残差、固定 persistence 残差）：它们与本得分的 rank correlation 如何？用它们作为得分屏蔽的增益是多少？
- 将"解释"弱化为经验观察。

### W7（中高）：推荐默认单元（通道行）与机理分析（元素）与"最优单元"（occurrence）三者不一致

**证据。**
- Sec. 5："As a default we recommend the channel row: it lowers MAE on every dataset (3.46%), has the lowest MSE of the five units, though not significantly so"。
- Table 1：通道行 MAE −3.46%，劣于 occurrence（−4.72%）和 element（−3.81%）；MSE −2.81%，只是在均值上最低，单元间 p=0.52。
- Table 2：通道行在中位数 (−7.24)、MAE、q90 上均劣于 occurrence。仅在 CVaR 与 MSE 上最低，而这两项单元间均不显著（p=0.273、0.525）。
- Sec. 7 全部围绕"元素"（既不是最佳也不是推荐的单元），只在 iTransformer 上。

**问题。** 推荐建立在不显著差异上；且未解释通道行相对元素/occurrence 的机制优势（假设：通道行不是单项极端噪声，而是"该通道在该窗口整体难"，有更好的稳定性，但论文没有验证）。读者按推荐会得到一个没有被深入分析的单元。另外，通道行相对不同骨干的适用性不同（DLinear 通道独立 vs. iTransformer 通道混合）；Table 5 中 DLinear 的通道行确实最强（MAE −5.73，MSE −4.66 为最佳），而 TimeMixer/iTransformer 上并不是（iTransformer MAE 上元素/occ. 优于它）。

**建议。**
- 要么把推荐改为条件性（依赖骨干/指标），要么在 Sec. 7 对通道行重复关键分析（随机匹配控制、与 SL 比较、得分对比）。
- 增加对"通道独立骨干 vs 通道混合骨干"的单元选择的分析。

### W8（中高）：与已有数据剪枝/选择方法的比较不足，"我方得分"实际上是回归版 EL2N；对 Selective Learning 的复现保真度不明

**证据。**
- Sec. 3.3 的得分：训练早期的平方残差，对单元内的项取平均。这在形式上就是 EL2N（Paul et al., 2021）在回归中的对应物（EL2N 是早期训练的误差范数，且 Paul et al. 对多个初始化取平均）。论文在 Related Work 提到 Paul et al.，但没有说明"我方得分 = 回归版 EL2N"，也没有把 EL2N/GraNd 作为命名基线。
- Table 3 的比较仅含：MC-dropout、TDDS（改编）、variance across occurrences、average loss、RobustTSF residual、Selective Learning（两种形式）。**没有** forgetting score、CCS、Moderate、InfoBatch、GraNd、random（作为元素掩码）、herding/k-center 等引言和相关工作列出的方法。
- Appendix B：TDDS 被改编为"the variance of the successive differences of [the average loss] trajectory"，与原 TDDS（分类概率的时间双深度）差别很大；"forgetting"在回归里无直接对应。改编是否保真，作者未论证。
- Selective Learning：Appendix B 说"nominal rates calibrated until the realised share of masked elements matches the 10% budget within half a point"。这改变了 SL 的原始超参设置；SL 的结果也可能对其辅助估计器敏感。

**同时，"窗口"这一列并不是"先前方法"**。论文的窗口单元使用的是**同一个得分**，而不是 You et al. (2024)、Liu et al. (2025) 等真实的窗口级方法。因此"先前数据选择方法（窗口）增益最少"的主张是对作者自己实现的泛化。

**建议。**
- 明确声明"评分 = 回归 EL2N"，并增加 GraNd 等价物（梯度范数）、forgetting events 的回归版（"误差突然增大"次数）、InfoBatch 风格（随机丢弃低损失样本并重新缩放）作为基线。
- 至少把一个真实的窗口级时间序列方法（如 Liu et al. 2025 的早期高损失丢弃，或 You et al. 2024 的采样器）按原样跑一遍，与你的窗口单元比较。
- 提供 SL 复现的细节（超参、mask rate、是否使用官方代码）。若可能，给出 SL 在其原论文设置下的数字，以证明复现与原文一致。
- 在 Sec. 7.2 扩展到 DLinear 和 TimeMixer，以及 ECL。

### W9（中高）："数据剪枝"的叙事与证据不匹配：没有任何计算节省，成本被低估，目标函数被重新定义

**证据与问题。**
1. 作者承认"a mask does not shorten an epoch"（Sec. 3.4）。因此主实验（Sec. 5、6）无计算节省；还额外需要一次评分运行（10% 的 epoch 加上对所有训练窗口的一次完整评估）。总开销大于 full-data 基线。论文并未报告 wall-clock 总时间。因此这些结果是一个"训练质量/鲁棒性"结果，而不是"数据剪枝"结果。
2. 传统数据剪枝（Paul et al., 2021; Sorscher et al., 2022; Qin et al., 2024 等）的成功标准是：**在剪掉数据以节省计算的同时保持性能**。按这个标准，low end 屏蔽（+0.09% 到 +0.49% MAE）恰恰是"几乎无损"的，是候选的剪枝方向，而论文反过来把它当作负面结果（"never helps"，"should be avoided at every unit"）。目标被重新定义为"最大化精度"，没有声明。
3. 唯一的效率结果是 Sec. 7.3：等间距窗口剪枝（30%）省 30% 的优化步，MSE +0.30 点（p=0.58，不显著，且"data cannot show it is zero"）。但这是文字描述，没有表格或图，没有 wall-clock，没有各数据集的明细，也没有给出"选窗口的三种得分规则"是哪三个、结果如何。并且这一结果与论文标题"Prune the Supervision, Not the Windows"相左：真正省计算的恰是剪窗口。
4. 等间距窗口剪枝本身（stride>1 的子采样）在重叠度高的数据上近乎无损，并不新（推断：这与"stride=1 滑窗高度冗余"的常识一致）。

**建议。**
- 把论文定位为"在滑窗训练中的监督屏蔽（supervision masking / loss trimming）"，或者增加真正的效率实验（如在固定计算预算下，masking + window subsampling vs. baselines 的 pareto 曲线，含 wall-clock）。
- 报告每个方法的总成本（评分运行 + 评估 pass + 训练）。
- 用 Sec. 7.3 的完整表格取代文字叙述（各数据集 × 剪枝率 × 规则）。
- 在低端屏蔽的"近乎无损"结果上，增加更高 $r$ 的扫描（如 30–70%），验证其是否确实是一个可用的"压缩"方向。

### W10（中）：仅测试了 $r=10\%$（主实验），没有剪枝比例扫描；"high end 值得、low end 无效"是否对 $r$ 稳健未知

**证据。** Sec. 3.3："The main comparison uses r = 10%"。Sec. 7.3 仅对窗口剪枝做了 10% 到 90% 的扫描。

**问题。**
- 屏蔽比例直接决定"trimmed loss"是否过度。在 $r$ 较大时，high end 屏蔽可能先升后降（过度丢弃信息），单元的次序也可能随 $r$ 反转（尤其是窗口单元在较大 $r$ 时会因覆盖面不同而改变）。
- 数据剪枝文献（Sorscher et al., 2022; Ayed and Hayou, 2023）的关键发现就是"最优策略依赖于剪枝率"，而这里只给出一个点。
- 在 10% 下，所有单元之间的差异（2%–5%）可能只是同一条曲线的不同位置。

**建议。** 对 $r\in\{1,2,5,10,20,30,50\}\%$ 做完整的单元 × 端点扫描（至少 2 个骨干 × 4 个数据集），报告 MAE/MSE 的曲线；同时报告 Tukey fence 自适应预算（Sec. 3.3）实际屏蔽的比例（正文未给出）。

### W11（中）：基线调参与实验覆盖范围——全部使用 TSLib 官方配方、$L=96$，可能存在欠拟合/过拟合，使"屏蔽"充当正则化

**证据。**
- Sec. 4：$L=96$，"official per-dataset recipes of TSLib"。
- DLinear 在 ETTh2/ETTm2 上 MSE 提升 8.7% 到 17.8%（Sec. 5）；Table 5：ETTh2 窗口 MSE −10.92%、ETTm2 窗口 −12.00%（连"窗口"也如此）。

**问题。**
- 屏蔽 10% 的项带来 10–17% 的 MSE 改善，对一个已调好的基线来说幅度惊人。ETTh2/ETTm2 是已知的高噪声、有强水平漂移、易过拟合的数据集（我对这些数据集的一般印象；论文没有讨论）。更合理的解释（假设）是：基线本身过拟合/次优，**任何**正则化（早停、权重衰减、更小学习率、dropout、随机丢弃损失项）都会有帮助。这需要随机屏蔽和调参后的基线来排除。
- 单一 $L=96$ 偏低，DLinear 等线性模型在更长回望（如 336/512）下通常更强；回望长度影响 $c(t,d)$ 之外的冗余结构。
- 缺失骨干：PatchTST、TimesNet 等常见基线；缺失数据集：Traffic、Exchange、ILI；更广义的，缺失预训练基础模型（Chronos、TimesFM 等）的数据选择设置，那里"数据剪枝"实际有价值（Wu et al.，2026 已被论文引用，说明该方向存在）。
- 论文没有报告任何 full-data 基线的**绝对** MSE/MAE，因此无法与已发表数值核对，也无法判断基线是否合理。Qiu et al.（2024）曾指出 TSF 基准中评估细节（如 drop_last）会影响结果，这类细节也应说明。

**建议。**
- 在附录加全数据基线的绝对数值表（含 SD）。
- 对基线做至少轻量的调参（权重衰减、学习率、dropout），并比较"调参后基线 vs 屏蔽"。
- 增加 $L\in\{96, 336, 512\}$ 的敏感性。
- 加入 PatchTST 或 TimesNet 中至少一个，并说明为何不包含 Traffic。

### W12（中）：Sec. 7 的范围过窄且部分结论依赖于特定实现

**证据。** Sec. 7 仅 iTransformer，且去掉 ECL（和 solar，对分解部分）；Table 3 仅 6 个数据集。

**问题。**
- 关键的机理结论（随机屏蔽几乎无增益、得分≈unexplained、SL 比较、屏蔽优于修复）均只在一个骨干上。DLinear/TimeMixer 的 mask 增益不同（Table 2 中 DLinear 贡献了大部分 MAE 增益），无从知道机理是否一般。
- "屏蔽优于修复"：对照使用沿通道的线性插值（Appendix B）。插值会引入平滑伪影，破坏频谱特征，因此这是一个相对弱的"修复"策略。更合理的是用模型自身预测、季节性朴素、或 STL 分解值来替换。
- 匹配随机屏蔽只在元素单元上做（Sec. 7.1）。

**建议。** 在 DLinear（通道独立、线性）和 TimeMixer 上至少复现随机匹配控制和 SL 比较；增加更强的"修复"对照。

### W13（中低）：负面/边缘现象未被解释

- ECL 上精细单元使 MSE 变差（Table 1：occ. +1.51%，element +1.75%）；论文称"ECL is the only high-dimensional dataset"（Sec. 5），但同文 Appendix A 又指出 solar 有 137 个通道，且缺席的 Traffic 有更多通道（推断自 TSLib 常识）。该说法自相矛盾或至少不准确。
- 在 solar 上 occurrence 的 MSE 为 +0.91%，而 MAE 为 −8.81%，说明屏蔽使模型在 MAE 上大幅提升，同时在 MSE 上变差，与"trimmed loss → 中位数回归"的解释（W2）一致，论文没有讨论。
- 在 solar 上，夜间值恒为 0（我对该数据集的认识；论文未讨论），残差分布含结构性零；"ties are broken by array index"（Appendix A）对 low-end 的平局会产生时间顺序偏差。建议给出 solar 上被屏蔽位置的可视化。
- 个别数据集上 DLinear 的 window MSE 变差（ETTh1 +0.09%、weather +0.12%、solar +2.25%；Table 5），而摘要仍写"MSE therefore improves on average"。

### W14（中低）：训练目标/实现细节不清，影响可复现性

- Eq. (4) 的分母是整个训练集上的 $\sum w$，而实际训练是 mini-batch。每个 batch 是按 batch 内 $\sum w$ 归一化，还是按固定的 $nTD$，还是全局常数？这将影响有效学习率和不同单元之间的梯度尺度（被屏蔽比例在 batch 之间差异，尤其对窗口单元）。
- "10% of the full number of epochs"：TSLib 配方的 epoch 数、是否使用 early stopping patience、"full number" 是否指最大 epoch 数，论文未写。若实际为 1 个 epoch，则评分的信号量和稳定性需要讨论。
- 评分运行与 masked 运行是否共用种子？mask 是否随种子变化？
- 没有代码链接（匿名评审阶段可提供匿名仓库）。没有给出总计算时间和硬件。

### W15（低中）：写作、呈现与定位

- 摘要极为稠密，包含十余个数字和概念（"unexplained error"、"occurrence"、"time column"、"decision unit"），读者难以在首读时抓住主线。
- 术语过多：time column / channel row / occurrence / element / high end / low end / typical error / largest errors。其中 "typical error"（中位数、MAE、q90）的归类有些随意（MAE 是平均而不是"典型"，且 MAE 并不"几乎不受最大误差影响"，对比 q90 的 −1% 到 −2.7%）。
- 引言称 Selective Learning 是"the one prior method that masks targets"；而 loss truncation、小损失选择、RobustTSF 的残差滤波（尽管单位不同）、Rho-1（token 级）均有类似"按项屏蔽"精神。这个"唯一"的主张需要更谨慎。
- Sec. 7 开头说"We therefore use it [element] to ask what the mask removes ..."，没有解释为什么在此用元素而不是推荐的通道行。
- 没有局限性/更广影响小节（TMLR 虽不强制，但应有）。

---

## 4. 领域层面的分析

### 4.1 研究脉络

| 脉络 | 代表工作 | 与本文的关系 |
|---|---|---|
| 分类中基于得分的数据剪枝 | EL2N/GraNd（Paul et al., 2021）；forgetting（Toneva et al., 2019）；TDDS（Zhang et al., 2024）；动态不确定性（He et al., 2024）；CCS（Zheng et al., 2023）；Moderate（Xia et al., 2023）；InfoBatch（Qin et al., 2024） | 本文的得分就是回归版 EL2N；这些方法的"单元 = 样本"假设正是本文挑战的对象 |
| 剪枝缩放律 | Sorscher et al.（2022）；Ayed and Hayou（2023） | 在"数据丰富"时应保留难样本；本文发现相反（去掉难项有益）。这是一个值得讨论的悖论（见 4.3） |
| 鲁棒回归/损失截断 | Huber（1964）；Rousseeuw（1984）的 LTS；Han et al.（2018）；Kang and Hashimoto（2020）；Shen and Sanghavi（2019） | **与本文方法最接近的先例**，论文基本未涉及（仅引 Han 和 Kang） |
| 时间序列数据选择 | You et al. (2024)、Liu et al. (2025)、Chen et al. (2026)、Li et al. (2026)、Wu et al. (2026)（均为论文引用，我无法独立核实） | 多数在窗口级操作 |
| 时间序列异常/鲁棒训练 | RobustTSF（Cheng et al., 2024）；Selective Learning（Fu et al., 2025） | 直接的方法邻居 |
| 误差度量与评估 | Hewamalage et al.（2023）；Qiu et al.（2024） | 评估陷阱与基准实现细节 |

### 4.2 隐藏联系

1. **滑窗 = 带权重的数据复制。** 窗口重叠使每个元素的"有效权重"等于 $c(t,d)$，在序列两端更小。作者定义了 $c(t,d)$，但没有分析"边缘元素"（前 $T$ 个目标时间）是否因 $c$ 小、均值方差大而被不成比例地选入 top-10%（评分噪声更高）。这是一个可检验的混淆。
2. **"监督屏蔽"与"噪声标签学习"同构。** 在预测里，$y$ 既是输入又是标签；被屏蔽的元素若是"标签噪声"，其仍作为输入留在后续窗口里（Sec. 3.4），带噪声的输入与带噪声的目标被区别对待。这与小损失准则（Co-teaching）下"噪声标签但样本特征保留"的设置相似。论文没有利用这个联系。
3. **损失-指标不匹配**（见 W2）：MSE 训练 + MAE 评估，使"修剪高残差"天然提高 MAE。
4. **分布漂移/非平稳性。** 高残差区域往往对应 regime 变化；训练集内的高残差并不等同于"不可预测"；论文提到"neither says that a target is unpredictable in principle"，但没有做 regime 层面的验证。可联系可逆实例归一化（Kim et al., 2022）这类针对漂移的处理，检验这些方法已经吸收了部分漂移后，屏蔽增益是否变小。

### 4.3 悖论与隐藏假设

- **悖论：** Sorscher et al.（2022）的理论预测，数据丰富时应保留难样本；滑窗数据极度冗余（"丰富"），但本文发现去掉难项有益。最自然的调和是：理论假设标签无噪声，而这里高残差项里有不可约噪声（"noisy label"解释）。本文应讨论，并用受控噪声（注入已知方差的噪声到训练目标）验证。
- **隐藏假设 1：** 训练集与测试集的噪声/异常结构相似。若测试期的 regime 分布与训练不同，屏蔽的收益可能反转。
- **隐藏假设 2：** 在 10% epoch 的残差排序对最终模型稳定。论文只用 Spearman 0.998（与 held-out 残差）佐证，没有显示与"不同种子的评分运行"的排序稳定性。
- **隐藏假设 3：** 评估指标按标准化尺度（TSLib 标准化）计算，通道间尺度已被均衡；在原始尺度上单元选择的结论可能不同。

### 4.4 研究空白

1. 在**不节省计算**的屏蔽之外，真正"可节省计算"的时间序列数据剪枝（在 Pareto 前沿上的）比较仍然缺失，本文也没有补上。
2. 剪枝比例×单元×骨干的完整曲线。
3. 预训练基础模型语境下的粒度选择（序列、片段、patch、点）。
4. 区分"噪声/异常/可学习难样本"的可识别性与受控实验。

---

## 5. 给作者的问题

1. 在相同预算下，各单元实际移除的损失质量（$\sum r$ 的份额）分别是多少？按质量匹配后，单元排序还成立吗？
2. 随机屏蔽（各单元、$r=10\%$）的结果是什么？L1/Huber 全数据训练 + 同样的屏蔽呢？在线 trimmed loss 呢？
3. 全数据基线在 L1/Huber 下的 MAE/中位数提升有多大？它是否已复现"中位数大幅改善、尾部不变"的模式？
4. 评分发生在第几个 epoch？TSLib 配方的 epoch 数、patience 是多少？评分运行与 masked 运行是否使用同一种子？
5. Eq. (4) 的归一化在实现中是 batch 内 $\sum w$，还是全局常数？
6. "unexplained error"和"explained error"的精确公式是什么？平凡得分（幅值、局部方差）的相关性和屏蔽收益如何？
7. 被排除的"发散"运行有多少个，分别在 full 还是 masked 一侧？
8. 为什么 Traffic 被排除？为什么称 ECL 是"唯一的高维数据集"而 solar 有 137 个通道？
9. 对 ETTh1/ETTm1、ETTh2/ETTm2 这种同源数据集，你们如何处理推断中的依赖？
10. 全数据基线的绝对 MSE/MAE 是多少？与 TSLib 公布的数值是否一致？
11. 为何推荐通道行？在 DLinear 以外的骨干上，通道行是否仍具优势（Tables 4、6 中并不明显）？
12. Sec. 7.3 的"三种按得分选窗口的规则"是什么？各数据集、各剪枝率下的表格能否提供？总 wall-clock 是否节省？
13. 屏蔽元素在训练时间轴上的分布如何？是否集中在训练集末端或某个 regime？
14. 对 Selective Learning 的复现是否使用官方代码？其原论文设置下的结果是否与你们的复现一致？

---

## 6. 次要问题（排版、笔误、符号）

1. 摘要过长（约 20 行）且数字密集，建议精简到核心三条结论。
2. Eq. (2) 中的集合表示 "$\{(i,t,d)\}_{t\in\mathcal T_i, d}$" 对"时间列/通道行"的下标写法不够清晰（逗号与下标混用），建议明确写出被固定与被遍历的指标。
3. Sec. 3.1："every element is supervised between 96 and 720 times per epoch"。严格说，只对远离两端的元素成立；前 $T$ 个目标时间的元素次数更少，且前 $L$ 个时间点从不作为目标（无法被屏蔽）。请在正文里一并说明。
4. Sec. 3.3："each unit masks its $\lfloor rm+1/2\rfloor$ highest-scoring members, so every unit removes about the same share"，请说明"about"的偏差范围（尤其元素单元在边缘处 $c(t,d)$ 不同）。
5. Table 1 的"Datasets better"含义（相对 full-data 改善的数据集数）已在 caption 说明，但同一表中有"mean"行含 7 个数据集的平均，再下又是"按骨干平均"的行，层次不够清楚；建议用分隔线或分表。
6. Table 3：caption 提到 "mean over six datasets"，建议在表内标明 ECL 和 solar 的处理（Sec. 7 开头说分解部分也去掉 solar，但表格用了 solar），避免混淆。
7. Table 2 的 "Friedman p" 列：说明检验基于 7 个数据集的 5 个单元，且每个比较是单独检验，已在 Sec. 4 声明；但表中没有指出哪些单元对在事后比较中显著，建议补充 Nemenyi 或类似的事后检验。
8. Fig. 1 的图示（右上到左下的蓝色小方块）对 occurrence 与 element 的可视区分不够；建议给每个单元的示例标注被屏蔽项数。
9. Appendix C 的第一句称 "Tables 4 to 6 split Table 1 by backbone" 正确，但 Fig. 3 的纵轴范围对 DLinear（约 −16%）与其他不同，建议采用共同纵轴或在图注中提示。
10. 参考文献大量为 2026 年预印本，且多数仅给出 arXiv 号；请确保正式引用信息正确，并在核心论点处引用经同行评审的文献。
11. "The window gains least"（Sec. 5）段落使用 "p = 0.001, p = 0.008"，但这些 p 来自 t 检验，请注明是哪一种检验以及单/双侧。
12. 中文读者提示：Sec. 5 的 "points" 与 "%" 的区分（百分点 vs. 百分比）较多，建议统一标注。

---

## 7. TMLR 两个标准下的评估

**标准 (a)：主张是否有准确、令人信服、清晰的证据支持？ —— 部分满足。**

- 优点：论文在统计细节上比同类更讲究，数字内部一致（我核对的聚合均值正确），主张措辞相对谨慎。
- 缺口：
  1. 核心"决策单元"主张被"选择尖锐度/被移除质量"混淆（W1）。
  2. 缺少随机屏蔽、L1/Huber、在线 trimmed loss 等必要基线，"low end 无效"在 MSE 训练下近乎预期（W2）。
  3. 有效样本量被高估，没有种子方差，多重比较未校正（W4）。
  4. "最差 10% 无变化"没有等价检验（W5）。
  5. 机理主张缺少平凡基线（W6）。
  6. "数据剪枝"的叙事与无计算节省的实际不符（W9）。
  7. 只有单一 $r$、单一 $L$、三个骨干（W10、W11）。

**标准 (b)：是否有 TMLR 读者会感兴趣的内容？ —— 是。**
"滑窗训练下剪枝单元可以小于样本"是一个干净而实用的观察，对时间序列预测训练和数据选择社区都有意义；误差分布层面的分析（增益集中在主体、不在尾部）也有启发性。若补上 W2 的基线并厘清 W1，这将是一个有分量的 TMLR 论文。

**总体推荐：Major revision（大修）。**
我认为论文有价值且方向正确，但当前证据不足以支持其核心结论的因果解释（"unit matters"、"which terms are masked, not masking as such"、"score ≈ unexplained error"）。W1 和 W2 属于需要新增实验才能回答的关键问题；若作者补做且结论保持，我愿意支持接收。

**置信度：4/5**（对 W1、W2、W4 的判断较有把握；对 TSLib 默认损失/epoch 数以及 ETT/solar 数据集特性的论述属于基于一般知识的推断，已在文中标明，作者可核实并反驳）。

---

## 参考文献（Harvard）

Ayed, F. and Hayou, S. (2023) 'Data pruning and neural scaling laws: fundamental limitations of score-based algorithms', *Transactions on Machine Learning Research*.

Benavoli, A., Corani, G., Demšar, J. and Zaffalon, M. (2017) 'Time for a change: a tutorial for comparing multiple classifiers through Bayesian analysis', *Journal of Machine Learning Research*, 18(77), pp. 1–36.

Cheng, H., Wen, Q., Liu, Y. and Sun, L. (2024) 'RobustTSF: towards theory and design of robust time series forecasting with anomalies', *International Conference on Learning Representations*.

Demšar, J. (2006) 'Statistical comparisons of classifiers over multiple data sets', *Journal of Machine Learning Research*, 7, pp. 1–30.

Fu, Y. et al. (2025) 'Selective learning for deep time series forecasting', *Advances in Neural Information Processing Systems*. (引自论文参考文献，我未独立核实。)

Han, B. et al. (2018) 'Co-teaching: robust training of deep neural networks with extremely noisy labels', *Advances in Neural Information Processing Systems*, 31.

Hewamalage, H., Ackermann, K. and Bergmeir, C. (2023) 'Forecast evaluation for data scientists: common pitfalls and best practices', *Data Mining and Knowledge Discovery*, 37, pp. 788–832.

Huber, P.J. (1964) 'Robust estimation of a location parameter', *Annals of Mathematical Statistics*, 35(1), pp. 73–101.

Kang, D. and Hashimoto, T. (2020) 'Improved natural language generation via loss truncation', *Proceedings of the 58th Annual Meeting of the Association for Computational Linguistics*.

Kendall, A. and Gal, Y. (2017) 'What uncertainties do we need in Bayesian deep learning for computer vision?', *Advances in Neural Information Processing Systems*, 30.

Kim, T., Kim, J., Tae, Y., Park, C., Choi, J.-H. and Choo, J. (2022) 'Reversible instance normalization for accurate time-series forecasting against distribution shift', *International Conference on Learning Representations*.

Paul, M., Ganguli, S. and Dziugaite, G.K. (2021) 'Deep learning on a data diet: finding important examples early in training', *Advances in Neural Information Processing Systems*, 34.

Qin, Z. et al. (2024) 'InfoBatch: lossless training speed up by unbiased dynamic data pruning', *International Conference on Learning Representations*.

Qiu, X. et al. (2024) 'TFB: towards comprehensive and fair benchmarking of time series forecasting methods', *Proceedings of the VLDB Endowment*, 17(9).

Rousseeuw, P.J. (1984) 'Least median of squares regression', *Journal of the American Statistical Association*, 79(388), pp. 871–880.

Shen, Y. and Sanghavi, S. (2019) 'Learning with bad training data via iterative trimmed loss minimization', *Proceedings of the 36th International Conference on Machine Learning*.

Sorscher, B., Geirhos, R., Shekhar, S., Ganguli, S. and Morcos, A.S. (2022) 'Beyond neural scaling laws: beating power law scaling via data pruning', *Advances in Neural Information Processing Systems*, 35.

Toneva, M. et al. (2019) 'An empirical study of example forgetting during deep neural network learning', *International Conference on Learning Representations*.

Xia, X. et al. (2023) 'Moderate coreset: a universal method of data selection for real-world data-efficient deep learning', *International Conference on Learning Representations*.

Zhang, X., Du, J., Li, Y., Xie, W. and Zhou, J.T. (2024) 'Spanning training progress: temporal dual-depth scoring (TDDS) for enhanced dataset pruning', *IEEE/CVF Conference on Computer Vision and Pattern Recognition*.

Zheng, H., Liu, R., Lai, F. and Prakash, A. (2023) 'Coverage-centric coreset selection for high pruning rates', *International Conference on Learning Representations*.

*备注：正文中引用的 You et al. (2024)、Liu et al. (2025)、Chen et al. (2026)、Li et al. (2026)、Wu et al. (2026) 等均出自被审论文的参考文献，我未能独立核实，因此未列入上表。*
