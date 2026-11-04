# TMLR 审稿意见：《Prune the Supervision, Not the Windows: Decision Units for Data Pruning in Time-Series Forecasting》

审稿人：独立审稿人（Fable, medium）
审稿依据：论文 PDF 全文 14 页（正文 §1–§8、参考文献、附录 A–D，Tables 1–7，Figures 1–3）。以下所有引用均注明页码/章节/表格；未在论文中找到的内容不作断言。

---

## 1. 论文摘要（审稿人复述）

论文研究滑动窗口（stride = 1）训练范式下的"数据剪枝"。作者的核心观察是：在分类任务中剪枝的最小单位是样本，而在预测任务中一个窗口的损失由 T×D 个独立可屏蔽的损失项组成，且同一目标值 y_{t,d} 会被约 T 个重叠窗口各监督一次（§1 p.1；§3.1 p.3–4，Eq. 1）。于是作者提出把"决策单位（decision unit）"作为唯一自变量，固定打分方式（在全量训练跑到 10% epoch 时，对所有训练窗口做一次无梯度前向，记录每个损失项的平方残差 r_i(t,d)，§3.3 p.4）和预算（r = 10% 的损失项），比较五种单位：window（T×D 项）、time column（D 项）、channel row（T 项）、occurrence（1 项）、element（跨窗口的同一目标的全部 c(t,d) 个出现，Eq. 2–3）。剪枝的实现不是删除窗口，而是把被选中损失项的权重置 0（Eq. 4，§3.4 p.4），因此不缩短 epoch。

实验：3 个骨干（iTransformer、DLinear、TimeMixer）× 7 个数据集（ETTh1/h2/m1/m2、weather、solar、ECL）× 4 个 horizon × 5 个种子，以数据集为推断单位（n = 7）做配对 t 检验和 Friedman 检验（§4 p.5）。主要发现（§5–§6，Table 1–2，Fig. 1–2）：(i) 屏蔽高残差端（high end）在所有单位都降低测试 MAE（window −2.03% 到 occurrence −4.72%），屏蔽低残差端从不帮忙；(ii) window 单位增益最小；(iii) 增益集中在误差分布的中位数附近，向 95 分位衰减，CVaR@90% 无可测变化，因此 MSE 改善较小且跨数据集不显著。§7 在 iTransformer 上（去掉 ECL，部分分析再去掉 solar）分析：分数与"其余序列无法解释的误差"高度相关（Spearman 0.998）；10% epoch 的均值残差分数与 MC-dropout、TDDS 等更贵的分数相当，优于 Selective Learning（Fu et al., 2025）；屏蔽优于插值修复；在 element mask 之上按等间距删 30% 窗口可省优化步数而 MSE 变化不显著（+0.30 点，p = 0.58）。作者推荐默认用 channel row。

---

## 2. 主要主张与证据充分性

| # | 主张（出处） | 证据 | 评估 |
|---|---|---|---|
| C1 | 屏蔽高残差 10% 在每个单位、每个数据集上都降低 MAE（摘要；§5 p.5；Table 1 "Datasets better 7/7"） | Table 1 平均值 + 配对 t 检验 p ≤ 0.022 | **部分充分**。平均效应有统计支持；但 "7/7" 中包含 ETTh1 window −0.11%、ECL window −0.78% 这类极小效应，而全文**没有任何方差/标准差/置信区间**（见缺点 2），"每个数据集都改善"的个体层面主张无法评估。 |
| C2 | 屏蔽低残差端从不帮忙（摘要；§5 p.5） | 平均在 −0.05% 到 +0.51% 之间；one-sided test at 0.3 点 | **基本充分，但过强**。作者自己承认 TimeMixer 两例区间过宽（p.5）；"never" 应改为"没有可测增益"。此外该结果在理论上几乎是平凡的（低残差项梯度本来就小，见缺点 4）。 |
| C3 | window 单位增益最小，channel row 和 occurrence 在每个数据集上优于 window（§5 p.5–6；Friedman p = 0.002） | Table 1、Table 2 | **充分**（在 r = 10%、分数固定的前提下）。但只在一个预算下验证（缺点 5）。 |
| C4 | 增益位于误差分布主体，尾部无变化，因此 MSE 变化小（§6；Table 2；Fig. 2） | 分位数曲线、CVaR@90%、Friedman | **充分且是论文最有信息量的部分**。但作者没有给出这一现象的最直接解释——截断高残差项的 MSE 训练在统计上等价于一种 robust/trimmed loss，自然使预测从条件均值偏向条件中位数（缺点 1）。 |
| C5 | 分数衡量的是"其余序列无法解释的误差"（§7.1；Spearman 0.998） | 5-fold mask-out，在 10% epoch 处读取 | **证据薄弱/近似套套逻辑**。在训练 10% 时，模型对单个目标的拟合几乎尚未发生，屏蔽掉该目标的监督与否对其残差几乎没有影响，相关性 0.998 很大程度上是必然的（缺点 7）。 |
| C6 | 早期一次读取的分数接近更贵的分数，优于 Selective Learning（§7.2；Table 3） | 6 个数据集的均值 + t 检验 | **部分充分**。MC-dropout 实际上在 MSE 上显著更好（p = 0.045）；Selective Learning 被强制校准到 10% 预算，并非原方法设置（缺点 8）。 |
| C7 | 屏蔽优于修复（§7.2 末尾） | 插值替换 vs 屏蔽 | **充分但解释受限**：插值同时改变了输入（附录 B 明确说明），不是对同一变量的干净对比。 |
| C8 | 可在 element mask 之上等间距删 30% 窗口而不损失（§7.3） | +0.30 MSE 点，p = 0.58，n = 6 | **不充分**。n = 6 的不显著不等于等价；文中提到的"three rules that select windows by their score"从未定义（缺点 9）。 |

---

## 3. 优点

1. **问题提法清晰且确实是时间序列特有的**。把"决策单位"从样本分离出来、并指出 stride-1 滑窗下每个目标被约 T 个窗口重复监督（§1、§3.1），是一个有价值的视角；element 单位（Eq. 2 最后一行、Eq. 3）确实是分类设定下不存在的单位。
2. **控制变量的设计是干净的**：五个单位共用同一个打分跑、同一个残差、同一个预算，只改变 Eq. 4 中 w(i,t,d) 的分组（§3.3–3.4）。这是 TMLR 偏好的"问一个精确的问题并回答它"的风格。
3. **实验规模与推断方式较为严谨**：3 骨干 × 7 数据集 × 4 horizon × 5 种子，且明确指出 horizon 与骨干间不独立而以数据集为推断单位（§4 "Inference"），使用 Friedman 检验比较单位，报告未经多重校正的 p 值并说明数据集数量——这比该领域大多数论文诚实。
4. **分位数分析（§6、Fig. 2a、Fig. 3）**是一个很好的诊断工具，解释了为什么 MAE 和 MSE 的结论不一致，并避免了只报 MSE 的常见误导。
5. **负结果被如实报告**：ECL 上 finer units 使 MSE 升高（Table 1）、CVaR 不变、MC-dropout 在 MSE 上更好、窗口按分数选择不优于随机（§7.3）等都没有被隐藏。
6. 写作总体精炼，图 1 对五个单位的可视化很直观。

---

## 4. 缺点与修改建议（按重要性排序）

### 缺点 1（最关键）：缺少最直接的替代解释与对应基线——这其实是一个 robust / trimmed loss，而非"数据剪枝"

**问题描述。** 在固定 mask 下屏蔽残差最高的 10% 的损失项，训练目标 Eq. 4 在统计意义上就是一种 *trimmed squared loss*（least trimmed squares 的固定版本，Rousseeuw, 1984；loss truncation，Kang & Hashimoto, 2020——后者作者已引用但只作为"相关设定"而非基线）。众所周知，截断大残差的平方损失会使估计量从条件均值向条件中位数靠拢，其后果恰恰就是论文 §6 观察到的现象：MAE / 中位数误差显著下降、尾部与 MSE 几乎不变（Table 2、Fig. 2）。换言之，论文的主要经验现象（C1、C4）有一个比"决策单位"更简单的解释：**改变了损失函数的鲁棒性**。在 TSLib 生态中，用 L1 / Huber loss 训练并报 MAE 是一个普通的、零成本的做法，而论文没有任何一个损失函数基线。

**位置/证据。** §3.4 Eq. 4；§6 "Why MSE moves less" (p.7)；§2 "Below the window" 段承认与 loss truncation、Co-teaching 的联系，但 §7.2/Table 3 的所有对比都是"同一单位下不同分数"，没有"不同损失函数"。

**为何重要。** 若 L1 或 Huber 损失训练（或在线 trimmed loss，每个 batch 丢掉最高 10% 的项，不需要任何打分跑和固定 mask）能带来相同或更大的 MAE 增益，则 (a) 论文"数据剪枝"的框架与 Sec. 7 对分数的分析失去大部分意义；(b) occurrence 单位的优势可能只是"更精细的 trimming"。这直接关系 TMLR 标准 (a)：主张是否被证据支持——目前"增益来自单位选择"这一因果主张与"增益来自损失鲁棒化"无法区分。

**具体建议 / 补充实验。**
1. 增加基线：(i) 全量数据 + L1 loss；(ii) 全量数据 + Huber loss（δ 取若干值）；(iii) **在线 trimmed loss**：每个 mini-batch 内按当前残差屏蔽最高 r 比例的项（occurrence 级别）和最高 r 比例的窗口（window 级别），无需打分跑。报告与 Table 1/2 相同的指标和分位数曲线。
2. 若固定 mask 仍优于在线 trimming，这将是论文真正的新结论（"读一次的早期分数优于逐步截断"），值得放进摘要；若不优，则应重写框架，把贡献定位为"trimming 的粒度分析"。
3. 在 §6 显式讨论 trimmed-loss / 中位数回归的解释，并把它与"仅屏蔽离群目标"的解释区分开（可用缺点 10 的诊断）。

### 缺点 2（关键）：没有任何方差、置信区间或绝对数值，"7/7 数据集改善"等个体主张无法核验；且剔除了发散的运行

**问题描述。** 全文（Tables 1–7）只报告相对于全量跑的百分比变化的均值，没有种子间标准差、没有置信区间，也**没有任何绝对 MSE/MAE 数值**。读者无法 (a) 判断 −0.11%（ETTh1 window，Table 1）这种效应是否在种子噪声内；(b) 核验全量基线是否达到 TSLib 公开的水平（例如 DLinear 在 ETTh2 上 −17.76% MSE，Table 5，这么大的相对增益是否来自一个偏弱的基线？）。此外 §4 p.5 "Runs that diverge are excluded" 没有说明剔除了多少、哪些配置、是否在 masked 与 full 之间不对称——这是潜在的幸存者偏差。

**为何重要。** TMLR 对"主张与证据一致"要求很高。"on all seven datasets" 出现在摘要、§1、§5 多处，但在没有方差的情况下，这些 "7/7" 计数对小效应是没有信息量的。

**建议。** (1) 在附录中给出每个 backbone × dataset × horizon 的绝对 MSE/MAE（全量与每种单位）及 5 个种子的标准差；(2) Table 1 的每个单元格附上 95% CI 或种子 std；(3) 报告被剔除的发散运行的完整清单及剔除规则，并做敏感性分析（把发散运行以最差值计入）；(4) 相对变化取平均对少数大幅变化极为敏感（DLinear ETTh2 −17.76% 对 "Mean" 行的拉动），建议同时报告中位数或在绝对值上做平均。

### 缺点 3（关键）：对"数据剪枝"文献读者而言，本方法不剪数据、不省计算，框架名实不符

**问题描述。** 标题与 §1 把工作定位为 data pruning，但 §3.4 明确"Every window and every input value stays in the training set"，"a mask keeps every window, it does not shorten an epoch"，而且还需要一次额外的全量前向打分（§3.3；虽然 §4 说打分跑就是全量跑本身，但仍然需要先跑完全量再跑 masked）。因此总成本 ≥ 全量训练 + 一次打分前向 + 再训练一次。数据剪枝文献（Paul et al., 2021; Toneva et al., 2019; Qin et al., 2024 InfoBatch; Okanovic et al., 2024）的根本动机是以更少数据/计算达到同等精度，本文在此维度上是负收益。§7.3 的省步数实验只有 iTransformer、6 个数据集、一个 mask 配置。

**为何重要。** 影响 TMLR 标准 (b)（读者是否会感兴趣）以及对相关工作的公平定位：读者按标题期望的是效率-精度权衡，得到的是一个损失重加权方法。

**建议。** (1) 把定位改为 "supervision masking / loss-term selection"，在摘要和 §1 明确声明不省计算；(2) 报告端到端的训练时间/FLOPs（全量 vs 打分+masked）；(3) 若要保留"pruning"的名义，需要在 §7.3 之外给出完整的 预算–精度–计算 三维权衡曲线。

### 缺点 4：低残差端"从不帮忙"在当前设定下近乎平凡，且与分类文献的对比有误导性

**问题描述。** §2 "Scores and their limits" 和 §5 把"在本设定下丢掉低分项有害、丢掉高分项有益"作为与分类文献（Sorscher et al., 2022：丢掉容易样本）相反的发现。但本文的 mask 仅移除 10% 的损失项，且不删除任何窗口；平方残差最小的 10% 项对梯度的贡献本来接近零（∂r/∂ŷ ∝ 残差），移除它们几乎不改变优化，结果"+0.1% 到 +0.5%"正是"无影响 + 噪声"。分类文献中丢弃容易样本有益的前提是大剪枝比例（30%–80%）与数据冗余，在 10% 的预算下根本不可比。

**建议。** (1) 把这一"发现"降级为一致性检查，不要写成"with the opposite polarity of classification"的结论；(2) 若要真正与分类文献对话，需要在 r ∈ {30%, 50%, 70%} 下重复低端/高端实验（见缺点 5）。

### 缺点 5：只有一个预算（r = 10%），单位排序与"window 最差"的结论缺乏预算稳健性

**问题描述。** §3.3 "The main comparison uses r = 10%"；仅 §7.3 用了 Tukey 上围栏的自适应预算（而且只用于 element）。数据剪枝文献的标准做法是扫描剪枝比例，而且分数方法在高比例下常常崩溃（Ayed & Hayou, 2023——作者已引用）。单位之间的差异在 r = 10% 时主要由"该单位能否集中屏蔽离群目标"决定（见缺点 10），当 r 增大时 window 单位的覆盖也会上升，排序可能改变甚至反转。

**建议。** 至少在 iTransformer × 3 个数据集上，对五个单位扫描 r ∈ {1%, 5%, 10%, 20%, 30%, 50%}，画出 MAE/MSE/CVaR 随 r 的曲线；报告 Tukey 自适应预算下实际的 r 值分布（按数据集）。

### 缺点 6：打分时刻只有一个（10% epoch），且 TSLib 默认 10 个 epoch 意味着分数来自第 1 个 epoch 结束时

**问题描述。** §3.3、附录 A 说打分在"10% of the full number of epochs"；TSLib 官方 recipe 默认 train_epochs = 10 且 early stopping patience = 3（这是审稿人基于 TSLib 公开配置的合理推断，论文没有写明实际 epoch 数）。如果属实，分数来自一个仅训练一个 epoch 的模型，而 §7.2 的"average loss over 100%"、TDDS 的比较也基于这样的轨迹。论文没有报告分数读取时刻的敏感性（0%、即未训练模型的残差；50%；100%；收敛时）。

**建议。** (1) 写明每个 backbone/dataset 实际的 epoch 数、early-stopping 规则、验证损失是否为未屏蔽 MSE；(2) 增加分数读取时刻的扫描；(3) 报告 10% 时刻的残差排序与收敛时残差排序的 Spearman 相关，以及与"untrained residual"的相关（后者在 §7.1 已算过，但只给了 explained error 的相关 0.22）。

### 缺点 7：§7.1 的 "unexplained error" 分析在 10% epoch 处读取，Spearman 0.998 很可能是套套逻辑

**问题描述。** §7.1 (p.8)："train one run per fold with that fold's supervision masked, read at the same early point of training as the score"。在训练仅 10% 时，模型的函数几乎完全由全局模式决定，单个目标是否被监督对它自己残差的影响微乎其微，于是"masked-out 残差 ≈ 全量残差 = 分数"，相关 0.998 几乎是必然的，并不能说明分数衡量"其余序列无法解释的部分"。更有说服力的是在**收敛时**读取 fold-out 残差（此时记忆效应才会出现），以及比较 fold-out 残差与全量残差之差。

**建议。** 在收敛点重做 5-fold 分析；报告 "unexplained − full-data residual" 的分布；给出 explained error 的定义公式（目前只有文字）。同时请解释为什么该分析去掉 solar（p.8 仅说"the decomposition below also leaves out solar"，无理由）。

### 缺点 8：与 Selective Learning 的比较不是原方法设置；Table 3 的结论在文字中被弱化

**问题描述。** 附录 B："its nominal rates calibrated until the realised share of masked elements matches the 10% budget within half a point"。这改变了 Fu et al. (2025) 方法的核心超参数，所得是"被约束到本文预算的 SL"，而非 SL 本身；摘要中"the element mask is ahead of Selective Learning in MAE on all six datasets tested"没有标明这一限制。另外 Table 3 中 MC-dropout 在 MSE 和 MAE 上都是加粗最优，且 §7.2 承认在 MSE 上显著（p = 0.045），而摘要说"comes close to scores that cost far more"——MC-dropout 的成本是 10% + 5 次前向，并不算"far more"。

**建议。** (1) 额外报告 SL 在其论文推荐超参数（无预算校准）下的结果；(2) 摘要/结论中对 SL 的比较加上"at a matched 10% budget"；(3) 对 MC-dropout 的措辞改为"在 MSE 上略优、MAE 上持平"。

### 缺点 9：§7.3 的"等间距删窗口"结论证据不足，且三条未定义的按分数选窗规则

**问题描述。** §7.3："None of the three rules that select windows by their score does better than random pruning at any rate from 10% to 90%"——这三条规则在正文和附录中都没有定义。同时"+0.30 points, p = 0.58, n = 6"被解读为"the data cannot show that it is zero"，这是正确的谨慎表述，但 §1 和结论却把它写成可以"save optimiser steps"的正面结果。

**建议。** 定义三条规则；给出 10%–90% 的完整曲线（MSE、MAE）；用等价检验（TOST）而非不显著来支持"无损"；扩展到至少一个非 Transformer 骨干。

### 缺点 10：缺少对"mask 落在哪里"的描述性分析——时间序列特有的混杂因素（非平稳、异常、昼夜结构、通道集中）未被检查

**问题描述。** 论文从未展示被屏蔽的元素在时间轴、通道、时段上的分布。这对解释至关重要：
- solar 夜间为零、晨昏为陡坡，高残差项几乎必然集中在晨昏；occurrence 在 solar 上 −8.81% MAE（Table 1）可能只是"放弃晨昏陡坡"的结果。
- ETTh2 含有已知的尖峰段，DLinear 在 ETTh2 上 −17.76% MSE（Table 5）与"屏蔽尖峰"一致。
- ECL 有 321 个通道，Table 1 显示 finer units 在 ECL 上提升 MSE（+0.05% 到 +1.75%），可能因为高残差项集中在少数高方差通道，10% 预算把它们整通道抹掉；这也是作者在 §7 把 ECL 排除的隐含原因，但没有验证。
- 分布漂移：若训练末段（最接近验证/测试）的残差系统性偏高（regime shift），mask 会优先屏蔽最"新"的数据，这与测试集最相关——这是一个与"离群"不同的机制。
- 另一个隐藏假设是残差在哪个尺度计算：iTransformer/TimeMixer 使用 instance normalisation，不同窗口对同一元素的残差若在各自归一化后的尺度上计算则不可比（Eq. 3 对它们求平均）。

**建议。** (1) 报告每个数据集上被屏蔽元素的时间直方图、通道直方图、时段（hour-of-day）直方图，以及它们与训练集最后 10% 的重叠率；(2) 报告一个"离群覆盖率"诊断：各单位在 r = 10% 时屏蔽了残差最高的 1% 的项的比例——我的假设是它的顺序是 window < time column ≈ channel row < element < occurrence，并且与 Table 2 的中位数增益单调相关，这将把"单位"的效应归结为"离群覆盖"；(3) 明确残差是在原始标准化尺度还是实例归一化后的尺度上计算的；(4) 做一个"时间屏蔽"对照：随机屏蔽训练集最后 10% 的目标，检验漂移机制。

### 缺点 11：统计推断的若干细节

- 7 个数据集被当作可交换样本，但四个 ETT 来自同一来源（两台变压器的两种采样率），独立性比作者假设的更弱；建议补充"以数据源为单位"（ETT、weather、solar、ECL，n = 4）的敏感性分析或至少讨论。
- "tested with one-sided tests at ±0.3 points"（§4）表述不清：这是 TOST 等价检验还是单侧非劣检验？0.3 点的边界由何而来？
- 多重性未校正（作者已声明），但摘要中的"never helps"、"at every unit"等全称判断建立在十几个未校正检验之上，措辞应相应放松。
- Fig. 2(a) 的 95 分位数"between −0.05% and −0.91%"被称为"no longer significant"，但未给出该检验的 p 值。

### 缺点 12：相关工作遗漏与定位

- 缺少 robust regression / trimmed loss（Rousseeuw, 1984）、Huber loss、以及异常鲁棒预测（作者引用了 RobustTSF 作为分数，但没有作为训练方法比较）。
- 分类剪枝中的覆盖类方法（CCS, Zheng et al., 2023；Moderate coreset, Xia et al., 2023）主张在高剪枝率下应保留中等难度样本——与本文"屏蔽最高端"有概念呼应，值得讨论（本文 r = 10% 下尚未触及这一区域）。
- Rho-1（Lin et al., 2024）被引用，但其核心是"excess loss against a reference model"，而本文 §7.1 的 unexplained error 实际上也是一种 reference-model 思路；可以更直接地对比。
- InfoBatch（Qin et al., 2024）的"丢弃低损失样本 + 重缩放梯度"与本文低端屏蔽直接对应，应作为低端基线的参照而非仅作引用。

---

## 5. 领域层面的分析

**研究脉络。** 数据剪枝的主线（EL2N/GraNd → forgetting → 动态不确定性 → TDDS/InfoBatch → 覆盖类 CCS/Moderate）始终以"样本"为单位，并以计算效率为目标。时间序列预测把"样本 = 窗口"照搬进来（You et al., 2024；Fu et al., 2025 是例外）。本文的贡献是把单位从窗口拆到损失项并指出跨窗口的 element 单位，这在概念上是对的，但把它称为剪枝会和主线的效率目标产生张力。

**隐藏联系。** (1) 固定 mask 的高端屏蔽 ≡ 离线 trimmed loss；在线版本 ≡ loss truncation / Co-teaching 的回归版。(2) 单位之间的差异在低预算下可能主要由"离群覆盖率"决定：window 单位每屏蔽一个窗口只去掉一个离群目标 1/T 的监督，而 occurrence 可以精确去掉它的全部高残差出现。这说明"window 最差"不是因为窗口是"错误的抽象"，而是因为 stride-1 重叠使得窗口级决策对单个目标几乎没有杠杆——这一机制论文在 §3.2 末尾一句话带过，值得作为主要解释并量化。(3) MAE 改善而 MSE 不变、尾部不变，是鲁棒损失的教科书式指纹，而非"剪枝只作用于误差主体"的新现象。

**研究空白。** (a) 时间序列中"难样本"与"异常"的区分：在分类里标签噪声与难例可以靠多次训练分开（forgetting），而预测里一个高残差目标可能是异常、regime shift 或仅是可预测性低；本文的 explained/unexplained 分解是向这个方向的一步，但需要在收敛点重做。(b) 真正省计算的 supervision pruning：若 mask 可以在 epoch 早期固定，能否把被完全屏蔽的目标附近的窗口合并/跳过？(c) 与基础模型/预训练的关系（Wu et al., 2026 被引用）：在多数据集预训练时 element 级的 mask 是否迁移？

**概念对比表（审稿人整理）**

| 方法族 | 单位 | 分数 | 是否省计算 | 本文的对应 |
|---|---|---|---|---|
| EL2N/GraNd, forgetting, TDDS | 样本 | 早期损失/梯度/轨迹 | 是 | window 单位（但不省计算） |
| InfoBatch | 样本（低损失） | 当前损失 | 是 | low-end window（结果：无益） |
| Loss truncation / Co-teaching | 样本/序列，在线 | 当前损失 | 否 | 最接近 occurrence，但本文为离线固定 mask（无此基线） |
| Rho-1 | token | 超额损失 vs 参考模型 | 否 | §7.1 的 unexplained error |
| Selective Learning | 目标（element） | 残差熵 + 异常估计器 | 否 | element 单位 |
| Trimmed / Huber loss | 损失项 | 当前残差 | 否 | 缺失的关键基线 |

---

## 6. 给作者的问题

1. 用 L1 loss、Huber loss 或在线 per-batch trimmed loss 训练同样的骨干，MAE/中位数误差的改善是多少？与 occurrence 单位相比如何？
2. 每个 backbone/dataset 的实际 epoch 数、early-stopping 规则是什么？打分发生在第几个 epoch 结束？验证损失是否为未屏蔽的 MSE？
3. 剔除了多少发散运行？分布在哪些配置？是否 masked 与 full 不对称？
4. Eq. 4 的归一化是在整个数据集上还是每个 mini-batch 上计算？若按 batch，被屏蔽项多的 batch 会放大其余项的有效学习率，这是否影响结果？
5. 残差 r_i(t,d) 在哪个尺度计算（数据集标准化尺度还是实例归一化尺度）？对 iTransformer/TimeMixer 的 RevIN 式归一化，Eq. 3 对不同窗口残差求平均是否可比？
6. 被屏蔽元素在时间、通道、时段上的分布如何？在 solar 上是否集中在晨昏？在 ECL 上是否集中在少数通道？
7. §7.3 的"three rules that select windows by their score"是什么？
8. §7 为什么去掉 ECL（我理解是"finer units raise MSE"），为什么 §7.1 的分解又去掉 solar？
9. Table 3 中 Selective Learning 若使用其论文推荐的超参数（不校准到 10%），结果如何？
10. 元素级 mask 下 10% 的目标完全失去监督；在 r 更大时（如 30%），element 单位是否因为整段目标缺失而崩溃？这会改变推荐默认单位吗？

---

## 7. 次要问题（排版、笔误、符号）

- p.2 §1："masking the low end never gains more than 0.3 percentage points"与 p.5 "+0.51%" 的表述并列时读者易混淆（前者说"增益"后者说"变化"），建议统一为"变化不低于 −0.3 点"。
- Eq. 2 中 element 行的基数 c(t,d) 在 Fig. 1 中写作 "≈ T terms"，但 §3.1 说"T of them away from the ends of the series"，建议在图中注明边界效应。
- §3.3 "⌊rm + 1/2⌋" 与附录 A 一致，但对 occurrence 单位 m = n·T·D，而对 element 单位 m = N·D，建议在 Eq. 2 旁列出各单位的 m。
- §4 "Readout"：RMSE 被列出但只在 Fig. 2(b) 使用；Table 2 没有 RMSE 行。
- Table 1 标题 "the last three rows give each backbone, averaged over the seven datasets"，但 "Mean" 与 "Datasets better" 也是独立两行，建议用横线分组更清晰（目前已有横线，可加说明）。
- Fig. 2(a) 与 Fig. 3 的 y 轴标签 "change against full data (%)" 应注明是绝对误差分位数的变化。
- 参考文献：Prayogo et al. (2020) 给出的是 URL 而非 DOI；Camporeale (2025)、Liu et al. (2025) 等 arXiv 条目缺少发表状态说明；格式总体一致。
- "Fu et al., 2025" 在 §2 中被引三次，建议首次给出方法全称及其单位（element）以便与 Eq. 2 对应。
- 摘要较长且包含过多数字，可将 §7 的三个次级结论压缩为一句。

---

## 8. TMLR 标准评估与总体推荐

**标准 (a) 主张是否有准确、令人信服、清晰的证据支持。** 部分满足。在"固定分数与预算、r = 10%"的前提下，五个单位之间的排序（C3）和增益位于误差主体（C4）有相当规模的实验和合适的统计检验支撑。但 (i) 没有方差与绝对数值、剔除发散运行未说明，使"每个数据集都改善"这类主张不可核验；(ii) 最重要的是缺少鲁棒损失 / 在线 trimmed loss 基线，使论文对"增益来自决策单位"的因果解释与一个更简单的解释无法区分；(iii) §7.1 的 0.998 相关性在当前读取时刻下几乎是套套逻辑；(iv) 与 Selective Learning 的比较不是原方法设置。这些都可以通过补充实验而非改变方法来解决。

**标准 (b) 是否有 TMLR 读者感兴趣的内容。** 满足。"决策单位"的视角、element 单位、以及分位数诊断对时间序列与数据选择两个社区都有价值；负结果（低端无益、尾部不变、按分数选窗不优于随机）同样有信息量。但目前的"data pruning"框架会让一部分读者失望（不省计算）。

**总体推荐：Major revision。**
论文有一个清楚的问题和干净的控制实验，但当前版本的核心解释存在一个未排除的、更简单的替代解释（trimmed/robust loss），且缺少方差与绝对数值。若作者补充 (1) L1/Huber/在线 trimmed loss 基线，(2) 带方差的绝对结果与发散运行说明，(3) 预算与打分时刻扫描，(4) mask 分布的描述性分析，并相应调整框架与措辞，我预期可以接收。

**置信度：4/5。** 我仔细阅读了全文与所有表格；对 TSLib 默认 epoch 数的推断来自公开配置而非论文；对 Fu et al. (2025) 原方法超参数的细节不作断言。

---

## 9. 参考文献（Harvard 格式；仅列出审稿人确信存在的文献）

- Ayed, F. and Hayou, S. (2023) 'Data pruning and neural scaling laws: fundamental limitations of score-based algorithms', *Transactions on Machine Learning Research*.
- Han, B., Yao, Q., Yu, X., Niu, G., Xu, M., Hu, W., Tsang, I.W. and Sugiyama, M. (2018) 'Co-teaching: Robust training of deep neural networks with extremely noisy labels', *Advances in Neural Information Processing Systems*, 31.
- Kang, D. and Hashimoto, T. (2020) 'Improved natural language generation via loss truncation', *Proceedings of the 58th Annual Meeting of the Association for Computational Linguistics*.
- Lin, Z., Gou, Z., Gong, Y., Liu, X., Shen, Y., Xu, R., Lin, C., Yang, Y., Jiao, J., Duan, N. and Chen, W. (2024) 'Not all tokens are what you need for pretraining', *Advances in Neural Information Processing Systems*, 37.
- Liu, Y., Hu, T., Zhang, H., Wu, H., Wang, S., Ma, L. and Long, M. (2024) 'iTransformer: Inverted Transformers are effective for time series forecasting', *International Conference on Learning Representations*.
- Mindermann, S., Brauner, J., Razzak, M., Sharma, M., Kirsch, A., Xu, W., Höltgen, B., Gomez, A.N., Morisot, A., Farquhar, S. and Gal, Y. (2022) 'Prioritized training on points that are learnable, worth learning, and not yet learnt', *International Conference on Machine Learning*.
- Okanovic, P., Waleffe, R., Mageirakos, V., Nikolakakis, K.E., Karbasi, A., Kalogerias, D., Gürel, N.M. and Rekatsinas, T. (2024) 'Repeated random sampling for minimizing the time-to-accuracy of learning', *International Conference on Learning Representations*.
- Paul, M., Ganguli, S. and Dziugaite, G.K. (2021) 'Deep learning on a data diet: Finding important examples early in training', *Advances in Neural Information Processing Systems*, 34.
- Qin, Z., Wang, K., Zheng, Z., Gu, J., Peng, X., Xu, Z., Zhou, D., Shang, L., Sun, B., Xie, X. and You, Y. (2024) 'InfoBatch: Lossless training speed up by unbiased dynamic data pruning', *International Conference on Learning Representations*.
- Rousseeuw, P.J. (1984) 'Least median of squares regression', *Journal of the American Statistical Association*, 79(388), pp. 871–880.
- Sorscher, B., Geirhos, R., Shekhar, S., Ganguli, S. and Morcos, A.S. (2022) 'Beyond neural scaling laws: beating power law scaling via data pruning', *Advances in Neural Information Processing Systems*, 35.
- Toneva, M., Sordoni, A., Tachet des Combes, R., Trischler, A., Bengio, Y. and Gordon, G.J. (2019) 'An empirical study of example forgetting during deep neural network learning', *International Conference on Learning Representations*.
- Wang, S., Wu, H., Shi, X., Hu, T., Luo, H., Ma, L., Zhang, J.Y. and Zhou, J. (2024) 'TimeMixer: Decomposable multiscale mixing for time series forecasting', *International Conference on Learning Representations*.
- Xia, X., Liu, J., Yu, J., Shen, X., Han, B. and Liu, T. (2023) 'Moderate coreset: A universal method of data selection for real-world data-efficient deep learning', *International Conference on Learning Representations*.
- Zeng, A., Chen, M., Zhang, L. and Xu, Q. (2023) 'Are Transformers effective for time series forecasting?', *AAAI Conference on Artificial Intelligence*.
- Zhang, X., Du, J., Li, Y., Xie, W. and Zhou, J.T. (2024) 'Spanning training progress: Temporal dual-depth scoring (TDDS) for enhanced dataset pruning', *IEEE/CVF Conference on Computer Vision and Pattern Recognition*.
- Zheng, H., Liu, R., Lai, F. and Prakash, A. (2023) 'Coverage-centric coreset selection for high pruning rates', *International Conference on Learning Representations*.
- Fu, Y., Shao, Z., Yu, C., Li, Y., An, Z., Wang, Q., Xu, Y. and Wang, F. (2025) 'Selective learning for long time series forecasting', *Advances in Neural Information Processing Systems* (as cited by the paper; arXiv:2510.25207).
