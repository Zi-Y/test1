# TMLR 审稿意见

**论文**：Prune the Supervision, Not the Windows: Decision Units for Data Pruning in Time-Series Forecasting（匿名投稿，正文 8 页 + 参考文献 + 附录 A–D，共 14 页）

**审稿人领域**：数据剪枝 / 数据选择 / 鲁棒训练；长期时间序列预测（LTSF）

---

## 1. 论文摘要（审稿人复述）

论文指出：在步长为 1 的滑动窗口训练中，序列里每个目标值（论文称为 *element*，即标量 $y_{t,d}$）每个 epoch 会作为目标被大约 $T$ 个不同窗口监督一次，一个窗口的损失由 $T\times D$ 个平方误差项组成（式 1）。以往面向预测的数据剪枝沿用了图像分类的做法，以"窗口/样本"为剪枝单位。作者固定打分方式和预算，只改变*决策单位*（decision unit），即一次决策同时保留或丢弃的一组损失项（式 2）。比较了五种单位：window（$T\times D$ 项）、time column（$D$ 项）、channel row（$T$ 项）、occurrence（单个损失项），以及跨窗口的 element（同一目标在所有覆盖它的窗口中的 $c(t,d)$ 个 occurrence）。

打分（§3.3）：先跑一次普通全数据训练，在训练进行到 10% epoch 时，对所有训练窗口做一次前向（关闭 dropout），记录每一项的平方残差；单位得分为其所含项残差的均值（element 见式 3）。在预算 $r$ 下，每种单位屏蔽得分最高的 $\lfloor rm+1/2\rfloor$ 个成员（"high end"），对照组则屏蔽得分最低的成员（"low end"）。屏蔽方式是把损失权重置 0 并重新归一化（式 4），所有窗口和输入值都保留，因此主实验**不缩短 epoch**。

实验（§4–§7）：iTransformer、DLinear、TimeMixer 三个骨干；ETTh1/h2/m1/m2、weather、solar、ECL 七个数据集；$L=96$，$T\in\{96,192,336,720\}$；5 个随机种子；采用 TSLib 官方配方。统计推断以"数据集"为单位：7 个数据集上的配对 t 检验、符号计数、±0.3 点的单侧检验，以及跨五种单位的 Friedman 检验。

主要结果：(i) 在 $r=10\%$ 下屏蔽 high end，按骨干平均后，每种单位在全部 7 个数据集上的 test MAE 都下降，幅度从 2.03%（window）到 4.72%（occurrence）；屏蔽 low end 从不带来收益。(ii) window 收益最小，channel row 和 occurrence 在每个数据集上都优于 window。(iii) 收益集中在误差分布中部：中位数 |e| 下降 4.42%–12.45%，到 95 分位基本消失；CVaR@90% 无显著变化；MSE 平均下降 1.73%–2.81%，但在跨数据集的检验下不显著。(iv) §7 只在 iTransformer 上做了分析：分数几乎就是"其余序列无法解释的误差"的重新标注（Spearman 0.998）；按长度和时段匹配的随机掩码几乎没有收益；10% 时刻读取的早期分数接近更昂贵的分数（MC-dropout 略好）；element mask 的 MAE 优于 Selective Learning；用插值修复反而有害；在 element mask 之上按等间距再删去 30% 窗口，可以节省 30% 的优化步数而不显著变差。作者推荐默认使用 channel row，只关心典型误差时使用 occurrence，任何情况下都不要屏蔽 low end。

---

## 2. 主要主张与证据充分性

| # | 主张（原文位置） | 证据 | 评估 |
|---|---|---|---|
| C1 | 屏蔽 high-end 10% 在每种单位、全部 7 个数据集上降低 test MAE（摘要；§5；表 1） | 表 1 均值行及 "7/7"；我逐项复核了表 1 的若干均值（如 window MAE −2.03、occ MAE −4.72、chan.row MSE −2.81、element MSE −2.09），与各数据集数值一致；骨干行与表 4 一致 | **数值本身充分**。但"全部 7 个数据集"依赖三骨干平均：TimeMixer 的 window 在 ETTh1 上为 +0.14，element 在 ECL 上为 +0.39（表 6）。正文对此有交代，摘要没有说明 |
| C2 | 屏蔽 low end 从不带来收益（§5，图 1） | 只给出了骨干平均柱（+0.09 到 +0.49）和单侧检验；28/30 个单骨干情形 | **基本充分**，但没有给出 low end 的逐数据集表，±0.3 点的等效边界也缺少依据（见 W4） |
| C3 | 决策单位很重要，window 收益最小（§5，Friedman p=0.002） | 表 1、表 2；channel row/occurrence 相对 window 在 7/7 个数据集上更好 | **证据有混杂**：在固定"项数"预算下，更细的单位按构造就会移除更多高残差质量，结论接近同义反复（见 W2）。而且只有一个预算 r=10% |
| C4 | 收益来自"屏蔽了哪些项"，而不是屏蔽本身（§5 末尾，§7.1） | low-end 对照；iTransformer 上的匹配随机掩码（−0.16%） | **部分支持**。随机对照只在 iTransformer + element 上做过。更关键的是缺少鲁棒损失（L1/Huber/截断损失）这类最直接的替代解释（见 W1） |
| C5 | 收益在误差分布中部，尾部（最差 10%）"不发生可测变化"（§6，表 2） | 分位数曲线；CVaR 的 Friedman p=0.27 | **"无变化"是靠不显著推出来的**，属于 absence of evidence。occurrence 的 CVaR 为 +0.47，只在 3/7 个数据集上改善（见 W4） |
| C6 | 推荐默认使用 channel row（§6 "Which unit to use"） | MAE 7/7；MSE 最低，但不显著 | **证据弱**：推荐依据的是不显著差异，而在唯一的高维数据集 ECL 上 channel row 的 MSE 上升（+0.05） |
| C7 | 分数≈"其余序列无法解释的误差"（§7.1） | 5 折掩码实验，Spearman 0.998；strict hold-out 中位数 0.990 | **解释力有限**：两者都在训练早期读取，高相关几乎是预期之中的；它也不能区分噪声和"难但可学"的目标（见 W6） |
| C8 | 早期分数接近昂贵分数（§7.2，表 3） | MC-dropout 在 MSE 上显著更好（p=0.045）；TDDS"无法区分" | **部分支持**。"无法区分"并不等于等效，而 MC-dropout 只多 5 次前向，成本几乎可以忽略 |
| C9 | element mask 的 MAE 优于 Selective Learning（§7.2） | 表 3，6 个数据集，p=0.013/0.040 | **需要核查公平性**：SL 被强制改成 10% 预算，并由作者复现（见 W6） |
| C10 | 屏蔽优于修复（§7.2） | 插值处理 MSE +4.32% | **对照不对等**：插值同时改变了输入和目标（见 W6） |
| C11 | 可以按等间距修剪窗口来节省优化步数，且无需打分（§7.3） | 只有正文中的几个 p 值，**没有任何表或图**；"三种按分数选窗的规则"没有定义 | **证据不充分**（见 W3） |

---

## 3. 优点

1. **问题提得清楚，而且确实是时间序列特有的问题。** 论文点出步长为 1 的滑动窗口中"每个目标被监督约 T 次"，把剪枝决策从"样本"推广到"损失项集合"，这个视角简洁有用。element 单位（跨窗口聚合同一目标）只在重叠窗口设定中存在，概念上有新意（§3.1–3.2，式 2）。
2. **受控设计。** 固定分数和预算，只改变单位，并设置 high-end 与 low-end 的对称对照（§3.3），因此"屏蔽本身 vs 屏蔽什么"至少可以初步区分。
3. **统计推断意识好于该领域的平均水平。** 以数据集为推断单位（§4 "Inference" 明确指出 horizon 与骨干间不独立），报告符号计数、Friedman 检验和不显著的结果（如 MSE、CVaR），没有只挑显著的报。
4. **超出均值指标的误差分布分析。** §6 的分位数剖面（图 2、图 3）和 CVaR@90% 揭示了"收益在中部、尾部无收益"，这对实践者很重要，也诚实地解释了 MSE 收益为什么小。
5. **数值内部一致性高。** 我抽查了表 1 的均值行、数据集数、骨干行与表 4–6 的对应关系，正文 §5–§7 引用的差值（如 1.43/2.69 点、0.91 点、0.24 点、2.03/1.76 点、7.13 点）也与表格一致，没有发现计算错误。
6. **有若干有价值的控制实验**：长度和时段匹配的随机掩码（§7.1）、strict hold-out、多种分数比较（表 3）、修复 vs 屏蔽，都是该方向少见的消融。
7. 篇幅紧凑，结论有节制（例如 §7.1 写道 "neither says that a target is unpredictable in principle"）。

---

## 4. 缺点与修改建议（按重要性排序）

### W1【最关键】缺少"鲁棒损失/损失截断"这一最直接的替代解释与基线，核心因果解释没有建立

- **问题**：屏蔽残差最大的 10% 损失项，本质上是对 MSE 做**修剪（trimmed）M-估计**，属于经典的鲁棒回归手段（least trimmed squares；Rousseeuw, 1984；深度学习中的 iterative trimmed loss，见 Shen and Sanghavi, 2019；生成任务中的 loss truncation，见 Kang and Hashimoto, 2020，论文已引用但没有作为基线比较）。论文观察到的整套现象，正是"训练目标从条件均值向更鲁棒的位置估计（更接近条件中位数）移动"的典型特征：
  - 收益最大在 **median |e|**（−4.42% 至 −12.45%），到 q95 基本消失（表 2、图 2a）；
  - **MAE 收益显著，MSE 不显著**，RMSE 收益约为 MAE 的四分之一（图 2b）；
  - 在偏态、零膨胀的 **solar** 上，occurrence 让 MAE 降 8.81%，MSE 却 **升 0.91%**；time column 的 MAE −7.83%，MSE +0.66%（表 1）。条件分布偏斜时，均值和中位数分离，这正是这种模式会出现的地方；
  - 收益最大的是**线性模型 DLinear 在 ETTh2/ETTm2 上**（MSE −8.7% 至 −17.8%，表 5），而线性最小二乘对离群点最敏感。
- **为何重要**：如果用全数据加 L1/Huber 损失训练（零额外成本、无需打分）就能得到同等或更大的 MAE 收益，那么论文的主要贡献（"选哪些监督项""单位很重要"）就会降格为"MSE 对离群点敏感"这一已知事实。C4（"收益来自屏蔽了哪些项"）目前只排除了"屏蔽本身"，没有排除"任何鲁棒化都行"。
- **建议的补充实验**（三个骨干、全部数据集）：
  1. 全数据 + **L1 损失**；全数据 + **Huber 损失**（$\delta$ 在验证集上选）；
  2. **动态逐批截断**：每个 mini-batch 内丢掉 occurrence 级残差最高的 10%，即 small-loss trick（Han et al., 2018）和 loss truncation 的直接类比，无需打分运行；
  3. **软重加权**：按残差分位数做连续降权，以区分"硬掩码"和"降低影响"；
  4. **mask + L1**：如果用 L1 训练后掩码收益大幅缩小，说明二者在利用同一机制。
  
  **可证伪预测**：如果论文的解释成立（关键在于"其余序列无法解释的目标"这一*身份*），那么 (a) 在 L1/Huber 训练下掩码仍应有显著增益；(b) 掩码的收益不应能被逐批截断完全复现。另外请报告预测的偏差（mean signed error）和每个数据集训练残差的峰度，检验收益与残差厚尾程度的相关性。

### W2【关键】单位比较被"移除的残差质量 / 定位纯度"混杂，且只有一个预算点

- **问题**：§3.3 用相同*项数*（约 $r$ 份损失项）作为预算。但单位越粗，它的均值分数就越会被大量低残差项稀释。occurrence 恰好移除残差最高的 10% 项，而 window 移除的是均值最高的 10% 窗口，其中包含大量正常项。因此在相同项数下，单位越细，移除的**高残差总量（residual mass）**按构造就越大。"window gains least"（§5）和"finer units gain more"在很大程度上是这个构造的结果，而不是"单位"本身的独立效应。§5 中 "Deciding per target gives up part of the gain" 也可以用同一逻辑解释：element 混入了被模型预测良好的 occurrence。
- **证据位置**：§3.3 "so every unit removes about the same share r of the loss terms"；主文全部结果都只在 $r=10\%$ 下得到（§3.3 "The main comparison uses r = 10%"）。
- **为何重要**：论文标题和贡献 1 的核心就是"单位很重要"。如果所有单位在"移除残差质量 vs 收益"的坐标上落在同一条曲线上，那么正确的结论应是"收益由移除的高残差质量决定，细单位只是更高效地定位了它"。这个结论依然有用，但与现在的表述不同。此外，数据剪枝文献几乎都会报告随剪枝率变化的曲线（Paul et al., 2021；Sorscher et al., 2022；Ayed and Hayou, 2023），单点比较无法回答排序是否依赖预算。
- **建议**：
  1. **预算扫描**：$r\in\{1,2,5,10,20,30,50\}\%$，在每个单位上同时报告 high end、low end 和随机掩码；
  2. 对每个单位报告实际移除的**项占比**和**残差质量占比**（element 在序列两端 occurrence 较少，项占比不严格等于 $r$），并画出"收益 vs 移除的残差质量"散点图；
  3. **按残差质量匹配预算**：让各单位移除相同的总残差，再比较；
  4. **合成/半合成控制实验**：在真实序列中注入结构已知的异常，例如单点离群（occurrence/element 结构）、单通道片段异常（channel row 结构）、跨通道同时刻异常（time column 结构），再检验与异常结构匹配的单位是否胜出。这能把经验观察变成机制性理解，也能直接检验"单位重要"的含义。

### W3【关键】"数据剪枝"的定位与实际方法、实际收益不符；效率部分（§7.3）证据缺失

- **问题**：
  - 主实验中的掩码**不减少任何计算**（§3.4 "it does not shorten an epoch"），反而多了一次 10% epoch 的打分运行和一次全训练集前向，总成本 ≥ 全数据训练。主流数据剪枝（Paul et al., 2021；Sorscher et al., 2022；Qin et al., 2024 的 InfoBatch；Okanovic et al., 2024）的首要目的是在同等精度下省计算，而本文主要方法实际上是一种**损失重加权/鲁棒训练**方法，以提高精度为目标。
  - §2 "Scores and their limits" 把"分类中保留难样本，而这里应丢弃最难项"表述为一种对立，但这是**体制不匹配**：本文剪枝率很低（10%），目标是带噪的回归，损失是 MSE；分类文献在数据稀缺或剪枝率高时同样建议保留易样本（Sorscher et al., 2022），CCS（Zheng et al., 2023）会先剔除最难的一部分样本，Moderate coreset（Xia et al., 2023）选择分数居中的样本。论文没有引用 CCS 与 Moderate，这条"对立"应改写为"连续"。
  - §7.3 是唯一涉及节省计算的部分，但**没有任何表或图**。"None of the three rules that select windows by their score" 中的三种规则没有定义；节省的只有"30% 的优化步数"，没有 wall-clock 数据；也没有与 InfoBatch、随机重采样（Okanovic et al., 2024）、You et al. (2024) 的 loss-weighted sampler 或 Liu et al. (2025) 的早期丢弃高损失样本进行比较。另外，步长为 1 的窗口高度重叠，等间距删窗本质上就是增大 stride。这一点很合理，但作为发现需要量化（例如在 10%–90% 的删窗率上画曲线）。
- **为何重要**：TMLR 的标准 (a) 要求主张与证据一致。标题 "Data Pruning" 会让读者期待效率收益，而论文主要提供的是精度收益；§7.3 的主张没有呈现证据。
- **建议**：
  1. 在标题、摘要、引言中明确把方法定位为"监督剪枝/损失掩码 = 鲁棒训练"，并在 related work 中补充鲁棒回归、trimmed loss、self-paced learning（Kumar et al., 2010）和 small-loss 选择等脉络；
  2. 给 §7.3 补一张完整的表或图：删窗率 10%–90% × {等间距, 随机, 三种分数规则}，每个数据集的结果、wall-clock 时间和总 FLOPs（包括打分成本），并加入 InfoBatch 与 You et al. (2024) 作为基线；
  3. 在 window 单位上加入真正"删除输入"的经典剪枝基线（EL2N/GraNd、forgetting、TDDS、random）作为与先前做法的直接比较。目前的 window 列只是"按本文分数屏蔽窗口损失"，不完全等同于已有方法。

### W4【重要】统计报告不完整："无变化"的主张、等效边界、方差与多重比较

- **问题**：
  1. **没有任何种子方差、置信区间或绝对数值**。全文只报告相对变化的均值，读者无法判断 ±0.3 点或 0.24 点的差异相对种子噪声有多大，也无法核对全数据基线是否与 TSLib 公开结果一致（基线偏弱会放大相对收益）。
  2. **等效边界 ±0.3 点**（§4）没有依据。它是相对百分比的差值，不同数据集、不同指标的噪声水平差别很大。
  3. **"The worst 10% of errors does not change measurably"（§6）** 依据的是不显著（Friedman p=0.27，以及与全数据比较不显著）。这是 absence of evidence，并且没有做等效检验（对 low end 做了，对 CVaR 却没做）。而 occurrence 的 CVaR@90% 均值为 +0.47%、只在 3/7 个数据集上改善，time column 为 +0.22%（表 2）；在没有 ECL 的表 7 中，occurrence 只有 3/6。也就是说，对最细的单位，尾部有变差的迹象。
  4. **多重比较不做校正**（§4 已声明）。全文有几十个检验，不少关键主张的 p 值落在 0.01–0.05（例如 "p ≤ 0.022 for each unit"、p=0.036、p=0.040、p=0.045）。经 Holm（Holm, 1979）等校正后，部分结论可能不再成立。
  5. **骨干平均掩盖异质性**：DLinear 的收益远大于另外两个骨干（MAE 均值 −3.57 至 −6.77 vs iTransformer −1.29 至 −3.69），主导了"所有骨干平均"的结论。TimeMixer 上 element 的 MSE 只在 3/7 个数据集上改善（表 6），iTransformer 上 MAE 最好的单位是 element 而不是 occurrence（表 4）。
  6. **每个 (骨干, 数据集, horizon) 只有一次打分运行**（附录 A "one ordinary full-data run"），5 个种子共享同一掩码，掩码本身的随机性没有计入方差。
  7. **"Runs that diverge are excluded"（§4）**：没有报告被排除的数量、分布在哪些单位或数据集、以及配对是否一并剔除。如果某种单位更容易发散，就会产生幸存者偏差。
- **建议**：报告每个配置的绝对 MSE/MAE 与跨种子标准差（附录即可）；用 TOST（Lakens, 2017）对 CVaR/尾部做等效检验，并说明边界的来源（例如以全数据跨种子标准差为尺度）；采用分层/混合效应模型（数据集为随机效应，骨干 × 单位为固定效应），或至少按骨干分别检验；对主张族做 Holm 校正，并在正文标出哪些主张在校正后仍成立；用不同种子的打分运行重复生成掩码并报告掩码重叠率（Jaccard）；报告发散次数。关于在多数据集上比较多种方法的推断，可参考 Demšar (2006) 和 Benavoli et al. (2016)。

### W5【重要】时间序列特有的混杂因素没有分析：非平稳/分布漂移、通道尺度、horizon 依赖、输入侧污染

- **问题**：
  1. **非平稳与时间位置**：ETT 等数据的训练段与测试段存在已知漂移。如果高早期残差的目标集中在训练段的特定时期（例如早期的旧体制，或某段异常期），那么掩码的作用可能是"丢掉与测试体制不符的旧数据"（类似近期加权），而不是"丢掉无法解释的噪声"。论文没有展示被掩码项在时间轴、通道和时段（例如 solar 的日夜）上的分布。
  2. **通道尺度**：分数是归一化尺度上的原始平方残差（§3.3），没有按通道标准化。噪声大的通道会占据高分，channel row 和 element 可能集中在少数通道，在 ECL（321 通道）上尤其如此。这可能解释为什么细单位在 ECL 上让 MSE 上升（表 1：+0.05 至 +1.75）。
  3. **horizon 依赖性没有报告**：论文的核心论点是"每个目标被监督 T 次"，但所有结果都按四个 horizon 平均。occurrence 与 element 的区别、window 被稀释的程度，理论上都应随 $T$（96→720）变化。这是最直接、最能支撑或证伪论点的分析，却缺失了。$L=96$ 也是固定的。
  4. **被掩码的目标仍在输入中**（§3.4 "a masked element still enters up to L windows as input"）。iTransformer 等模型使用实例归一化（RevIN 类，Kim et al., 2022），输入中的异常会扭曲归一化统计量。论文没有讨论这种输入侧影响，也没有做"掩码目标 + 修复输入"的分解。
- **建议**：(a) 给出被掩码 element 的时间直方图、通道分布和时段分布，并与"删除最旧的 10% 训练段"或"按时间近期加权"基线比较；(b) 做按通道标准化分数（例如残差除以通道残差方差）的消融；(c) 在附录给出按 horizon 拆分的表 1/表 2，并画出"单位间差距 vs $T$"；(d) 做 $L\in\{96, 336, 512\}$ 的敏感性分析；(e) 做 2×2 分解：目标 {保留, 掩码} × 输入 {原值, 插值}。

### W6【重要】§7 的范围、解释与对照公平性

- **问题**：
  1. **范围与排除**：§7 只使用 iTransformer，排除了 ECL，分解实验还排除了 solar（§7 首段 "the decomposition below also leaves out solar"），都没有给出理由。strict hold-out 用的是"nine combinations"，没有说明是哪九个。
  2. **Spearman 0.998 几乎是预期结果**：分数和"unexplained error"都在训练 10% 时读取（§7.1 "read at the same early point of training"）。这时模型对单个目标的记忆很弱，训练项残差和留出项残差自然几乎相同。这说明分数不是记忆效应，但**不能区分**"不可约噪声/异常"与"难但后期可学"的目标（RHO-Loss 的区分正是要在收敛的参考模型上读取 holdout loss，见 Mindermann et al., 2022）。所以"the plain early score already ranks targets by the error that the rest of the series leaves unexplained"（§2）这一解释，其信息量比措辞暗示的要小。
  3. **MC-dropout 显著更好且成本极低**：表 3 中 MC-dropout 的 MSE 优于本文分数（p=0.045），而成本只多 5 次推理前向，相对一次训练可以忽略。这与"cheap early score"的推荐有张力。另外 DLinear 没有 dropout，所以 MC-dropout 不能通用，这一点应讨论。
  4. **Selective Learning（SL）的比较公平性**：SL 被改造成 10% 名义预算（附录 B "calibrated until the realised share of masked elements matches the 10% budget"），是作者复现的实现，原超参数是否调过不清楚。强制统一预算可能不利于 SL。
  5. **"Masking beats repairing" 的对照不对等**：插值处理同时改变了输入和目标（附录 B "an interpolated value enters both"），而掩码只改变目标。差异可能来自输入被修改，而不是"修复 vs 屏蔽"。
  6. **"same elements, values interpolated" 的 MAE 差异 p=0.058**，但正文写的是 "trails the mask on all six datasets"。两者不矛盾，但应同时给出符号检验的 p 值（6/6 时双侧 1/32）。
- **建议**：说明排除 ECL/solar 的原因，或补全这两个数据集；在收敛点（或用独立训练到收敛的参考模型）重做 unexplained/explained 分解，并将 RHO-Loss 式的 "training loss − holdout loss" 作为一个分数纳入表 3；在 SL 比较中同时报告"原文阈值/原始预算"版本，并与 Fu et al. (2025) 报告的数值对照；增加"只修复目标、输入保持原值"的处理；在三个骨干上都做匹配随机掩码（目前只在 iTransformer element 上做过）。

### W7【中等】外推范围有限，"默认 channel row"的推荐缺乏支撑

- **问题**：三个骨干都是监督式 LTSF 模型，缺少 PatchTST（Nie et al., 2023）这类通道独立的 patch Transformer，也缺少时间序列基础模型的微调场景（论文 §2 引用了 Wu et al., 2026，与基础模型相关）。数据集缺少 TSLib 标配中的 **Traffic**（862 通道）。而 ECL 作为唯一的高维数据集恰好出现了 MSE 方向反转（表 1：channel row/occ/element 为 +0.05/+1.51/+1.75），因此"细单位更好"在高维数据上是否成立正是未决问题。§6 推荐 channel row 的依据是 MAE 7/7 和不显著的 MSE 优势，在 ECL 和 TimeMixer 上 channel row 的 MSE 也不稳定（表 6 ECL +0.68）。
- **建议**：加入 Traffic，以及至少一个 patch 类骨干和一个基础模型微调场景；在推荐处明确写出条件，例如"在低维数据上"，并把推荐和其置信度绑定。

### W8【中等】可复现性细节不足

- 缺少：每个骨干/数据集的 epoch 数（"10% of full epochs" 在 TSLib 默认 10 个 epoch 下约为 1 个 epoch，需明确）、早停 patience、学习率调度、batch size；式 (4) 在 mini-batch 中是**按批归一化**还是全局常数归一化（window 单位会改变每批的有效项数，归一化方式会改变梯度尺度）；TSLib 版本/commit，以及测试加载器是否使用 `drop_last`（TFB 基准指出这会影响 LTSF 结果的可比性，见 Qiu et al., 2024）；总运行数和算力开销；代码是否开放。
- 建议补一个算法框（打分 → 掩码 → 训练）和一个超参数表。

### W9【中等】部分陈述无法核实或与表格存在张力

1. §5："the occurrence is ahead on every dataset of every backbone"。按上下文指的是 median |e|，但正文没有按骨干、按数据集给出 median |e|，因此**无法核实**。而在 MAE 上，iTransformer 的 element 在 solar（−7.57 vs −5.64）和 ECL（−2.01 vs −1.86）上优于 occurrence，按骨干平均也更好（−3.69 vs −3.67，表 4），这与"按目标决策会损失部分收益"的一般表述存在张力。建议在附录补充分骨干的 median |e| 表，并修改措辞。
2. 摘要 "lowers test MAE at every unit and on all seven datasets" 应写明"按三个骨干平均"。
3. 摘要和结论中的 "MSE ... improves on average ... but not significantly" 与图 1 标题 "MSE moves in the same direction but less" 的措辞基本一致，但图 1 只画了 MAE，建议把 MSE 也画出来。
4. §7.3 "Equal spacing ... level at this rate (within ±0.3 points, p=0.002)" 与 "removing 30% ... +0.30 points against the mask alone"：+0.30 已经在 ±0.3 边界上，需要澄清这两个比较的对象分别是什么。

### W10【次要但影响阅读】写作与呈现

- 摘要信息密度过高，数字太多。建议压缩到 3–4 个核心发现，其余放进正文。
- "points"（两个相对百分比之差）与 "%"（相对变化）混用，读者容易混淆。建议统一说明，或改用绝对差。
- 没有 Limitations 小节（TMLR 鼓励写）。
- 表格只有相对变化，建议在附录给出绝对值表。

---

## 5. 领域层面的分析

### 5.1 研究脉络与定位

| 维度 | 已有工作（代表） | 本文位置 |
|---|---|---|
| 决策粒度 | 序列级（Prayogo et al., 2020；Zhu et al., 2024）→ 窗口/样本级（You et al., 2024；Liu et al., 2025；Wu et al., 2026）→ 时间点/通道（Hounie et al., 2024；Wang et al., 2025）→ 时间步掩码（Fu et al., 2025） | 系统比较窗口以下的五种粒度，并提出跨窗口的 element |
| 目的 | 效率（InfoBatch；Okanovic et al., 2024；Paul et al., 2021）vs 鲁棒/精度（Co-teaching；loss truncation；RHO-Loss） | 主实验实际属于"鲁棒/精度"，§7.3 才涉及效率 |
| 分数 | 静态早期（EL2N/GraNd）、训练轨迹（forgetting、TDDS）、参考模型（RHO-Loss、Rho-1）、异常估计器（RobustTSF、SL） | 静态早期残差均值；MC-dropout 略优 |
| 保留哪端 | 分类中丰富数据保留难样本（Sorscher et al., 2022）；高剪枝率/噪声下先去最难（CCS；Moderate） | 丢最难（high end） |

### 5.2 隐藏联系

1. **掩码 high end ≈ 修剪 M-估计**（least trimmed squares；Shen and Sanghavi, 2019）。occurrence 单位是逐项 trimmed MSE，element 是"按目标"的离群剔除，window 是"按样本"的修剪。从这个角度看，论文发现"细单位更好"等价于"离群点的定位精度越高越好"，这是鲁棒统计里熟悉的结论（W2）。
2. **"其余序列无法解释的误差" ≈ RHO-Loss 的 irreducible loss**，区别在于本文在训练早期读取（W6）。
3. **element 单位 ≈ 异常感知训练**（RobustTSF、Selective Learning）。本文的贡献是表明，不需要单独的异常检测器，早期残差就足够了。
4. **等间距删窗 ≈ 增大 stride**。在高度重叠的窗口中，这几乎不损失信息，与"重复随机采样是强基线"（Okanovic et al., 2024）的观察一致。

### 5.3 悖论与其消解

- **"分类保留难样本，预测丢弃难样本"**：这不是领域间的根本对立，而是由 (i) 低剪枝率、(ii) 回归目标中存在异常或噪声、(iii) MSE 对离群点的二次敏感性共同决定的。可证伪的预测：在 L1 训练下、在干净的合成数据上，或在高剪枝率下，"丢弃 high end"的优势会缩小甚至反转。
- **"典型误差改善、尾部不改善"**：这对风险敏感的应用（如电网峰值、极端天气）是一个实际代价。掩码"无法解释的目标"可能让模型更少学到罕见事件，需要在尾部指标上做专门评估（W4）。

### 5.4 研究空白（本文可以，或后续工作应当填补）

1. 一个关于"何时单位重要"的理论或半合成刻画：异常的时空结构 × 单位结构。
2. 在重叠窗口下，真正节省计算的剪枝：梯度相关性与有效样本量。
3. 兼顾尾部风险的监督剪枝：例如只剪"中部偏高"的项，或做分位数感知的预算。
4. 扩展到基础模型的微调和预训练语料的 token/patch 级选择，与 Rho-1 相呼应。

---

## 6. 给作者的问题

1. 在全数据训练中直接使用 L1 或 Huber 损失，MAE/MSE 相对于 MSE 训练如何变化？与 high-end 掩码相比如何？两者叠加呢？
2. 在 $r\in\{1,5,20,30\}\%$ 下，五种单位的排序是否保持？各单位在 $r=10\%$ 下实际移除了多少比例的损失项和残差质量？
3. 被掩码的 element 在训练段时间轴上如何分布？是否集中在特定时期、通道或时段（例如 solar 的日出日落）？
4. "10% of full epochs" 在各骨干/数据集上具体是第几个 epoch？式 (4) 的归一化在实现中是按 batch 还是全局？
5. 被排除的发散运行有多少？分布在哪些单位或数据集？
6. §7 为什么排除 ECL，分解实验又为什么排除 solar？"nine combinations" 具体是哪些？
7. 既然 MC-dropout 在 MSE 上显著更好且成本极低，为什么推荐早期残差分数？对 DLinear 这类没有 dropout 的模型有什么替代？
8. §7.3 中"三种按分数选窗的规则"分别是什么？30% 删窗节省了多少 wall-clock 时间（包括打分成本）？
9. 能否给出按 horizon 拆分的结果？element 与 occurrence 的差距是否随 $T$ 增大？
10. Selective Learning 使用的是官方实现吗？在其原始阈值下的结果如何，与 Fu et al. (2025) 报告的数值是否一致？
11. 全数据基线的绝对 MSE/MAE 是否与 TSLib 或原论文报告的数值一致？跨 5 个种子的标准差是多少？

---

## 7. 次要问题（排版、笔误、符号）

1. **符号冲突**：$r_i(t,d)$ 表示平方残差（§3.3），$r$ 又表示预算（"At a budget r"）。建议把预算改为 $\rho$ 或 $b$。
2. $c(t,d)=|\mathcal{W}(t)|$ 与 $d$ 无关，可直接记为 $c(t)$。"every element is supervised between 96 and 720 times" 只对远离序列两端的内部元素成立；前 $L$ 步只作为输入，$c=0$。
3. §3.3 "each unit masks its ⌊rm+1/2⌋ highest-scoring members"：对 element 而言，成员的项数不同，"about the same share" 应给出实际数值。
4. 图 1 只画了 MAE，标题却提到 MSE，建议补充 MSE 子图或误差棒。
5. 表 1、2、4–7 都没有不确定性（标准差/CI）。
6. 表 6 中 ECL 的 MSE 一栏把 +0.02 加粗为"最佳"，技术上没错，但建议注明"均未改善"。
7. 参考文献格式不统一：TimeMixer（Wang et al., 2024）缺少 arXiv 编号，其他条目都有。
8. "points" 的定义只在 §4 出现一次，建议在表注中重复说明。
9. 附录 D 写 "the median, MAE, and the 90th percentile fall at every unit"，但表 7 中 window 的 q90 只在 5/6 个数据集上下降。应写明是"均值下降"。
10. 摘要中 "by 1.7% to 2.8%" 与正文 "1.73% to 2.81%" 精度不一致（小问题）。
11. 建议增加 Limitations 小节和 Broader Impact 讨论（尾部风险）。

---

## 8. TMLR 标准下的评估与总体推荐

**标准 (a) 主张是否有准确、令人信服、清晰的证据支持**：**部分满足**。
- 描述性的经验发现（C1、C2，以及表 1/表 2 中的数值）报告得严谨，内部一致，统计设计优于平均水平。
- 但论文的核心*解释性*主张，即"决策单位本身重要"（C3）和"收益来自屏蔽'无法解释'的目标"（C4/C7），受到两个未排除的强替代解释威胁：**鲁棒损失效应（W1）**和**移除残差质量的构造性差异（W2）**。此外，"尾部不变"（C5）是由不显著推出的，§7.3 的效率主张（C11）没有呈现证据，"推荐 channel row"（C6）依据的是不显著的差异。

**标准 (b) TMLR 读者是否会感兴趣**：**满足**。重叠窗口下"每个目标被监督 T 次"的视角、element 单位，以及对误差分布的细致刻画，对时间序列和数据选择两个社区都有价值。即便最终结论被重新表述为"细粒度鲁棒修剪"，也是有用的实证贡献。

**总体推荐：Major revision（大修）**

接收所需的最低修改：(1) 加入 L1/Huber/逐批截断基线（W1）；(2) 预算扫描，加上按残差质量匹配的单位比较或合成注入实验（W2）；(3) 给 §7.3 补充完整证据，或删去相应主张，并相应调整"数据剪枝"的定位（W3）；(4) 补充方差和绝对数值、尾部的等效检验、发散运行报告（W4）；(5) 按 horizon 拆分结果，并分析被掩码项的时间/通道分布（W5）。

**置信度**：4/5。我熟悉数据剪枝和 LTSF 文献，也逐表核对了论文数值；但没有运行代码，也不能独立核实 2025–2026 年的部分参考文献。

---

## 9. 参考文献（Harvard 格式；仅列审稿中引用、且审稿人确认存在的文献）

- Ayed, F. and Hayou, S. (2023) 'Data pruning and neural scaling laws: fundamental limitations of score-based algorithms', *Transactions on Machine Learning Research*.
- Benavoli, A., Corani, G. and Mangili, F. (2016) 'Should we really use post-hoc tests based on mean-ranks?', *Journal of Machine Learning Research*, 17(5), pp. 1–10.
- Demšar, J. (2006) 'Statistical comparisons of classifiers over multiple data sets', *Journal of Machine Learning Research*, 7, pp. 1–30.
- Han, B., Yao, Q., Yu, X., Niu, G., Xu, M., Hu, W., Tsang, I. and Sugiyama, M. (2018) 'Co-teaching: Robust training of deep neural networks with extremely noisy labels', *Advances in Neural Information Processing Systems*, 31.
- Holm, S. (1979) 'A simple sequentially rejective multiple test procedure', *Scandinavian Journal of Statistics*, 6(2), pp. 65–70.
- Huber, P.J. (1964) 'Robust estimation of a location parameter', *The Annals of Mathematical Statistics*, 35(1), pp. 73–101.
- Kang, D. and Hashimoto, T. (2020) 'Improved natural language generation via loss truncation', *Proceedings of ACL 2020*.
- Kim, T., Kim, J., Tae, Y., Park, C., Choi, J.-H. and Choo, J. (2022) 'Reversible instance normalization for accurate time-series forecasting against distribution shift', *International Conference on Learning Representations*.
- Kumar, M.P., Packer, B. and Koller, D. (2010) 'Self-paced learning for latent variable models', *Advances in Neural Information Processing Systems*, 23.
- Lakens, D. (2017) 'Equivalence tests: A practical primer for t tests, correlations, and meta-analyses', *Social Psychological and Personality Science*, 8(4), pp. 355–362.
- Mindermann, S. et al. (2022) 'Prioritized training on points that are learnable, worth learning, and not yet learnt', *International Conference on Machine Learning*.
- Nie, Y., Nguyen, N.H., Sinthong, P. and Kalagnanam, J. (2023) 'A time series is worth 64 words: Long-term forecasting with transformers', *International Conference on Learning Representations*.
- Okanovic, P. et al. (2024) 'Repeated random sampling for minimizing the time-to-accuracy of learning', *International Conference on Learning Representations*.
- Paul, M., Ganguli, S. and Dziugaite, G.K. (2021) 'Deep learning on a data diet: Finding important examples early in training', *Advances in Neural Information Processing Systems*, 34.
- Qin, Z. et al. (2024) 'InfoBatch: Lossless training speed up by unbiased dynamic data pruning', *International Conference on Learning Representations*.
- Qiu, X. et al. (2024) 'TFB: Towards comprehensive and fair benchmarking of time series forecasting methods', *Proceedings of the VLDB Endowment*, 17(9).
- Rousseeuw, P.J. (1984) 'Least median of squares regression', *Journal of the American Statistical Association*, 79(388), pp. 871–880.
- Shen, Y. and Sanghavi, S. (2019) 'Learning with bad training data via iterative trimmed loss minimization', *International Conference on Machine Learning*.
- Sorscher, B., Geirhos, R., Shekhar, S., Ganguli, S. and Morcos, A.S. (2022) 'Beyond neural scaling laws: beating power law scaling via data pruning', *Advances in Neural Information Processing Systems*, 35.
- Toneva, M. et al. (2019) 'An empirical study of example forgetting during deep neural network learning', *International Conference on Learning Representations*.
- Xia, X., Liu, J., Yu, J., Shen, X., Han, B. and Liu, T. (2023) 'Moderate coreset: A universal method of data selection for real-world data-efficient deep learning', *International Conference on Learning Representations*.
- Zheng, H., Liu, R., Lai, F. and Prakash, A. (2023) 'Coverage-centric coreset selection for high pruning rates', *International Conference on Learning Representations*.
- 以下文献引自被审论文本身的参考文献列表，审稿人未逐一独立核实其 2025–2026 年版本：Fu et al. (2025) Selective Learning；Cheng et al. (2024) RobustTSF；Zhang et al. (2024) TDDS；Hounie et al. (2024)；Liu et al. (2025)；You et al. (2024)；Wang et al. (2025)；Wu et al. (2026)；Prayogo et al. (2020)；Zhu et al. (2024)。
