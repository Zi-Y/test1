# TMLR 审稿意见

**论文**：Prune the Supervision, Not the Windows: Decision Units for Data Pruning in Time-Series Forecasting（匿名投稿，14 页，含附录 A–D）

---

## 1. 论文摘要（审稿人复述）

论文指出，在 stride=1 的滑动窗口预测训练中，一个窗口的损失由 T×D 个平方误差项组成（式 1），而同一个目标值 y_{t,d} 会被 c(t,d)=|W(t)|（内部点为 T 个）个窗口重复监督。已有的时间序列数据剪枝方法沿用了图像分类里"以样本为单位"的做法，即以窗口为剪枝单位。作者把"打分函数"和"预算"固定，只改变**决策单元**（decision unit，即一次决策同时保留或丢弃的损失项集合），比较了五种单元（式 2）：window（T×D 项）、time column（D 项）、channel row（T 项）、occurrence（1 项），以及跨窗口的 element（某个目标值的全部 c(t,d) 次出现）。

打分方法（§3.3）：在一次普通全数据训练进行到总 epoch 数 10% 时，对所有训练窗口做一次评估，记录每个 occurrence 的平方残差 r_i(t,d)，单元得分取其所含项的残差均值。在预算 r=10% 下，屏蔽得分最高的 ⌊rm+1/2⌋ 个单元（"high end"）或得分最低的（"low end"）。屏蔽只是把损失权重置 0 并重新归一化（式 4），输入和窗口都保留，所以 epoch 不会变短。

实验覆盖 iTransformer、DLinear、TimeMixer 三个 backbone，ETTh1/h2/m1/m2、weather、solar、ECL 七个数据集，L=96，T∈{96,192,336,720}，5 个 seed。统计检验以数据集为单位（n=7）：配对 t 检验、符号计数、±0.3 点的单侧等价检验，以及五个单元之间的 Friedman 检验。主要发现如下：
(i) 屏蔽 high end 在所有单元、所有数据集上都降低 MAE（window −2.03%，occurrence −4.72%），屏蔽 low end 没有收益；
(ii) window 单元收益最小（Friedman p=0.002）；
(iii) 收益集中在误差分布的主体部分（中位数 −4.42% 至 −12.45%），越往尾部越弱，CVaR@90% 与 MSE 都没有显著变化；
(iv) 在 iTransformer 上（§7），得分几乎就是"留一折交叉拟合下的不可解释误差"的重标记（Spearman 0.998），廉价的早期得分接近 MC-dropout/TDDS 这类更贵的得分；element mask 的 MAE 优于 Selective Learning（Fu et al., 2025），"屏蔽"优于"插值修复"；在 mask 之上按等间距剪掉窗口可以节省优化步数。

---

## 2. 主要主张与证据充分性

| # | 主张（原文位置） | 证据 | 评估 |
|---|---|---|---|
| C1 | 屏蔽 high-end 10% 在每个单元、7/7 数据集上降低 MAE（摘要；§5；表 1） | 表 1 均值行与 "Datasets better" 7/7；p≤0.022 | **结论本身有数据支撑**，但只是对三个 backbone 取平均后的结果（TimeMixer 上 window 与 element 只有 6/7，表 6）；没有给出 seed 方差。**机制解释不充分**，见 W1 |
| C2 | 屏蔽 low end "never helps"（摘要；§5） | 10 个均值都通过 ±0.3 点单侧检验（p≤0.004）；30 个单 backbone 情形中有 28 个 | 在 10% 预算下基本成立；±0.3 点的等价带为何这样取没有论证；仅测了单一预算 |
| C3 | window 单元收益最小，单元选择有影响（§5，Friedman p=0.002） | 表 1、表 2 | 排序可信，但"单元"的作用与"被屏蔽的残差总量"这一混杂因素尚未分离（W2） |
| C4 | 收益在误差分布主体，尾部无可测变化（§6；表 2；图 2–3） | 分位数曲线；CVaR p=0.27 | "无可测变化"是 n=7 下的不显著，**不等于等价**；未对 CVaR/MSE 做等价检验（W5） |
| C5 | 推荐默认使用 channel row（§1 贡献 2；§6；§8） | MSE 最低但不显著（Friedman p=0.525） | **证据不足**：推荐依据的是作者自己判定为不可区分的差异（W7） |
| C6 | 得分 ≈ "其余序列无法解释的误差"（§7.1） | 5 折交叉拟合，Spearman 0.998 / 0.22；strict hold-out 0.990 | 相关性数值可信，但解释近乎同义反复：只训练了 10% 的模型尚未记住单个目标（W6） |
| C7 | 廉价得分接近昂贵得分；element mask 的 MAE 优于 Selective Learning（§7.2；表 3） | iTransformer、6 个数据集 | 仅限单一 backbone；SL 被改造成固定 10% 预算，公平性存疑（W8） |
| C8 | 屏蔽优于插值修复（§7.2） | 表 3 末行 | 存在混杂：插值同时改动了输入（训练/测试分布不一致）（W9） |
| C9 | 等间距剪窗口可省 30% 步数且无显著损失；三种基于得分的窗口规则都不优于随机（§7.3） | **正文与附录均无表格或图**；"three rules" 没有定义 | **证据不可核查**（W10） |

---

## 3. 优点

1. **问题提出清晰且有洞察**：把"剪枝单元"从"样本"中剥离出来，并指出时间序列滑动窗口天然具有 T 倍重复监督（§1，§3.1）。这是对"沿用分类任务剪枝范式"的有价值反思，element 这一跨窗口单元的概念也很新颖。
2. **控制变量的设计思路好**：固定得分和预算，只改变单元（式 2，§3.4："the five units differ in what a decision ties together and in nothing else"）。low-end 作为对照，可以排除"重新归一化导致有效学习率变化"这类机械性解释。
3. **统计推断比领域平均水平更严谨**：明确以数据集为推断单位，避免把 horizon×seed 当作独立样本（§4 "Inference"）；同时报告符号计数、Friedman 检验和等价检验；也坦承未做多重比较校正。
4. **误差分布分析有信息量**：分位数、CVaR 和 MSE/RMSE 的对照（表 2、图 2–3）说明收益集中在"典型误差"上。这是 TSF 文献中少见、而且对实践者很重要的刻画。
5. **§7.1 的对照实验设计巧妙**：长度和时段都匹配的随机 mask（−0.16% MSE）表明"选哪些目标"比"mask 的形状"更重要。
6. 数值内部一致性良好。我抽查了表 1 的均值行与 backbone 平均（如 window MAE −14.21/7=−2.03；ETTh1 window = (−0.28−0.19+0.14)/3=−0.11；ECL element MSE = (0.44+1.47+3.35)/3=1.75）、表 3 首行与表 4 去掉 ECL 后的均值（MAE −3.97，MSE −2.80/−2.81），以及 §5 中 DLinear 的 MSE 区间（8.66–17.76；−1.93 至 +2.32），均与正文吻合。

---

## 4. 缺点与修改建议（按重要性排序）

### W1【最关键】"剪枝收益"很可能主要是**稳健损失 / 损失截断**效应，而论文没有与稳健损失基线比较

- **问题**：训练目标是 MSE（式 1/4），得分是早期平方残差，屏蔽的是残差最大的 10% 项。这在统计上接近 least trimmed squares（Rousseeuw, 1984）或迭代截断损失（Shen and Sanghavi, 2019），作用是把估计量从"条件均值"推向更稳健、更接近"条件中位数"的位置。此时可以预期 MAE 与中位数绝对误差大幅下降，而 MSE（训练目标本身）与尾部误差几乎不变。这正是论文的核心发现（§6，表 2：median −4.42% 至 −12.45%，MSE/CVaR 不显著）。换言之，论文观察到的"收益在误差主体"可能并非剪枝单元的性质，而是"训练损失（L2）与评估指标（L1）不匹配"被部分纠正的结果。
- **证据**：§6 "MSE, which weights the largest errors most, therefore improves on average … but not significantly"；表 2 CVaR@90% 在 occurrence 上是 +0.47%（只有 3/7 改善）。§2 引用了 loss truncation（Kang and Hashimoto, 2020）和 Co-teaching（Han et al., 2018），但没有作为基线。
- **为何重要**：如果直接用 MAE/Huber 损失训练，或者在每个 mini-batch 内在线丢弃残差最大的 10% 项（无需打分运行），就能获得同等或更大的 MAE 收益，那么论文的实践建议和"决策单元"叙事都需要重写。这关系到 TMLR 标准 (a)，即主张与证据是否一致。
- **建议实验**：
  1. 全数据 + L1 损失；全数据 + Huber（δ 网格）；全数据 + 在线 batch 内 trimmed loss（occurrence 粒度，10%）。
  2. 在 L1 训练的模型上重复 high-end/low-end 屏蔽：如果收益消失，说明主要是损失不匹配效应。
  3. 把这些结果加入表 1/表 2，并报告 MAE、中位数、CVaR、MSE 四个指标。

### W2【关键】"单元"效应与"被屏蔽的残差质量 / 打分分辨率"存在混杂

- **问题**：在相同的 10% 项数预算下，单元越细，选出的项越接近真正残差最大的那 10%，因此被移除的平方残差总量越大。occurrence 在定义上就是"按项截断"的最优形式，window 则把大残差项与大量正常项捆绑在一起。表 1/表 2 中 window < time col ≈ channel row < element < occurrence 的排序，可能只是"移除的残差总量"的单调函数，而不是单元结构本身的作用。
- **证据**：§3.3 用项数来统一预算（"every unit removes about the same share r of the loss terms"），但没有报告每个单元移除的残差质量。
- **建议**：(a) 报告每个单元在 10% 预算下移除的 Σr 占比，以及被屏蔽项与 occurrence top-10% 的重叠率（Jaccard）。(b) 加一组按"移除残差质量匹配"的对照，例如提高 window 的预算，使其移除的 Σr 与 occurrence 相同，再比较测试误差。(c) 画出"移除残差质量 vs MAE 变化"的散点图（单元 × 数据集 × backbone）。如果所有单元落在同一条曲线上，那么"单元重要"的主张应改述为"截断的精度重要"。

### W3【关键】缺少每个单元的**随机屏蔽**基线和**预算扫描**

- **问题**：数据剪枝文献的标准做法是同时报告随机剪枝，并给出多个剪枝率下的曲线（Paul et al., 2021；Sorscher et al., 2022；Zheng et al., 2023；Okanovic et al., 2024）。本文主结果只有 r=10%，且只有 high/low 两端。随机 mask 只在 §7.1 针对 iTransformer/element 做过一次。不同粒度的随机屏蔽本身就是不同强度的正则化（类似 label dropout），收益可能随单元变化。
- **为何重要**："only the high end is worth masking"和"the window gains least"这两个结论都只在单一预算下验证。分类文献已知最优端会随剪枝率和数据量翻转（Sorscher et al., 2022），本文没有检验这一点。
- **建议**：对每个单元报告 random 基线；在 r∈{1, 5, 10, 20, 30, 50}% 上给出 high/low/random 三条曲线，至少覆盖一个 backbone 和全部数据集，并报告最优 r。

### W4【关键】时间序列特有的结构混杂没有分析：lead time（预测步长位置）、时间位置与非平稳性

- **问题 1（lead time）**：同一目标的不同 occurrence 对应不同的 lead time（t−i−L）。远步长的残差系统性更大，所以 occurrence 的 high end 很可能集中在远步长位置。这样一来，occurrence 屏蔽实际上近似于"降低远步长的损失权重"，与 Hounie et al.（2024）讨论的 loss shaping across horizon 直接相关。论文提到了这一工作（§2），但没有检验这种解释。这也可以解释"element 比 occurrence 差"（§5）：element 对所有 lead time 取平均，从而丢失了这一信息。
- **问题 2（时间位置 / 分布漂移）**：被屏蔽的 element 在训练序列中处于什么位置？如果集中在某段异常时期（如 ETT 中的传感器异常）或训练集末尾（最接近测试期），解释会完全不同：前者是去除异常，后者则可能伤害对近期分布的适应。
- **问题 3（per-horizon）**：结果按四个 horizon 取平均（§4）。但单元结构依赖于 T（window 有 T×D 项，element 的 c=T），单元效应很可能随 T 变化。
- **建议**：(a) 给出被屏蔽项在 lead time 上的直方图（每个单元）；(b) 加一个基线：按 lead time 标准化残差后再打分（即在每个 lead time 内部取 top-10%），或者直接按 lead time 设权重；(c) 画出被屏蔽 element 在时间轴上的分布，并与已知异常段对照；(d) 补充按 horizon 分开的表 1。

### W5【重要】"尾部无可测变化"的证据是"不显著"，不是"等价"；统计独立性假设也过于乐观

- **问题 1**：§6 写道 "The worst 10% of errors does not change measurably"，摘要中也说 "does not change measurably"。在 n=7 下，Friedman p=0.27、t 检验不显著都不能证明没有变化。表 2 中 occurrence 的 CVaR 为 +0.47%（3/7），图 3 中 iTransformer 的 occurrence 在 q95 上略高于 0。作者对 low end 用了等价检验，对尾部却没有用。
- **问题 2**：ETTh1 与 ETTm1、ETTh2 与 ETTm2 是同一变压器数据在不同采样率下的版本（ETT-small 的来源结构，这一点论文没有讨论）。因此 7 个数据集并非 7 个独立来源，有效 n 更接近 5，而 "7/7" 的符号论证会被高估。
- **问题 3**：没有报告任何 seed 间方差或置信区间；"Runs that diverge are excluded"（§4），但没有说明排除了多少、分别属于哪些条件。如果排除集中在某些单元或某种 mask，会造成选择偏差。
- **建议**：对 CVaR@90% 和 MSE 做与 low end 相同的 ±δ 单侧等价检验，并论证 δ 的选取；把 ETT 按源合并后重做主要检验作为敏感性分析；在附录表格中给出 seed 标准差；报告发散运行的数量和分布。另外，多重比较至少应给出 Holm 校正后的结论是否改变。

### W6【重要】§7.1"不可解释误差"的解读有循环论证之嫌

- **问题**：得分是在 10% epoch 时读出的残差。此时模型几乎还没有拟合单个目标（论文给出的 "explained error" 与得分的相关只有 0.22，本身就说明了这一点）。所以"在见过监督的模型上的残差"与"留出该目标监督的模型上的残差"高度一致（0.998）几乎是必然的，因为两者都约等于"早期模型对该目标的拟合误差"。这说明得分捕捉的是"早期难拟合的目标"，但并不能说明这些目标是"噪声/异常"还是"困难但可学"。论文自己也承认 "neither says that a target is unpredictable in principle"，可是摘要和结论仍把它表述为机制性解释。
- **建议**：(a) 在不同的打分时刻（例如 5%、10%、30%、100%）重复交叉拟合；如果在训练末期 Spearman 仍很高，解释才更有力。(b) 用合成数据做验证：注入已知的点异常或噪声段，以及已知的可学习但困难的模式（如稀有周期），检验 high-end mask 分别移除了哪一类，以及移除后对这两类测试误差的影响。这是区分"去噪"与"丢弃困难样本"的直接实验，也与 RHO-Loss（Mindermann et al., 2022）中 "learnable vs noisy" 的区分直接对应。

### W7【重要】"推荐默认使用 channel row"与论文自身的推断标准不一致

- **问题**：§6 "As a default we recommend the channel row: … has the lowest MSE of the five units, though not significantly so"。但作者自己的 Friedman 检验显示 MSE 上各单元不可区分（p=0.525）；MAE 上 channel row（−3.46）与 time column（−3.44）几乎相同，且明显不如 occurrence；在 ECL 上 channel row 的 MSE 为 +0.05；在 iTransformer 上 channel row 的 MSE 也不是最好（表 4，element −2.33 更好）。
- **建议**：要么撤回明确的单元推荐，改为"window 以外的任一细粒度单元"；要么给出支持 channel row 的显著证据，例如 channel row 相对 occurrence 在 MSE 或 CVaR 上的配对检验。

### W8【重要】基线范围与公平性：标准数据剪枝得分只在一个 backbone 上比较；Selective Learning 被改造过

- **问题 1**：标题和引言把工作定位为"数据剪枝"，但除表 3（仅 iTransformer、element 单元、6 个数据集）之外，没有在其他单元或 backbone 上比较 EL2N/GraNd（Paul et al., 2021）、forgetting（Toneva et al., 2019）、Moderate（Xia et al., 2023）、CCS（Zheng et al., 2023）、InfoBatch（Qin et al., 2024）等方法。因此"得分 × 单元"的交互作用未知：在其他得分下，window 是否仍然最差？
- **问题 2**：Selective Learning（Fu et al., 2025）在附录 B 中被校准到固定 10% 的屏蔽率，其原始方法的屏蔽率是自适应的；而且 SL 只在 iTransformer 上比较。"element mask is ahead of Selective Learning in MAE on all six datasets"（摘要）这一结论可能受校准方式影响。
- **问题 3**：§7 排除了 ECL，而 ECL 恰好是细粒度单元使 MSE 变差的数据集（表 1）；§7.1 的分解又排除了 solar，原因都没有说明。这容易被理解为选择性报告。
- **建议**：(a) 至少在 DLinear 上补充"得分 × 单元"的小规模网格；(b) 报告 SL 原始设置（原始超参、自适应比例）下的结果；(c) 说明排除 ECL 和 solar 的原因（例如计算或内存限制），并在可行时补上。

### W9【中等】"屏蔽优于修复"的对比存在输入分布不匹配的混杂

- **问题**：附录 B 中插值处理直接修改了训练序列，因此插值同时进入了**输入**和**目标**（"an interpolated value enters both"）。但测试时输入仍是原始序列，包含同类的大残差点，这会造成训练/测试输入分布不一致。表 3 中 +4.32% 的 MSE 可能主要来自这种不一致，而不是"修复目标"本身。
- **建议**：增加"只修复目标、保留原始输入"的变体（即仅替换监督信号），才能真正比较 mask 与 repair。

### W10【中等】§7.3 的结论无数据支持，且 "three rules" 未定义

- **问题**：§7.3 给出了多个数值结论（+0.30 点、p=0.58；50%/70% 时 p=0.04/0.006；"None of the three rules that select windows by their score does better than random pruning at any rate from 10% to 90%"），但正文和附录都没有对应的表格或图，"three rules" 也没有定义。
- **建议**：补充一张表或图，内容包括剪枝率 10–90% × {equal spacing, random, 三种 score 规则} 的测试误差和步数/墙钟时间，并给出三种规则的定义。

### W11【中等】计算开销、可扩展性与"数据剪枝"定位

- **问题**：mask 方法不减少 epoch 长度（§3.4），还需要额外 10% 的打分运行，所以它本质上是**提升精度的监督重加权**，而不是传统意义上提升效率的数据剪枝。论文没有报告总计算量。按实验设计估算，主实验约需 3×7×4×5×(1 + 5 单元×2 端) ≈ 4620 次运行（审稿人推算）。另外，occurrence 单元需要存储每个 (i,t,d) 的残差。以 ECL、T=720 为例：训练长度约 1.8 万步，窗口约 1.76 万个，残差项约 1.76e4×720×321 ≈ 4e9 个，float32 约 16 GB（审稿人推算，需作者核实）。论文没有讨论这一内存和 IO 成本。
- **建议**：报告打分和 mask 的时间与内存开销，并讨论大 D、大 T 下的实现方式（如流式 top-k、分块计算）；在标题、摘要和引言中更明确地将方法定位为"监督选择/重加权"，并与"效率型剪枝"区分开。

### W12【中等】训练协议细节不足，影响可复现性与结论的稳健性

- **问题**：(a) "10% of the full number of epochs" 的具体 epoch 数没有给出。TSLib 官方脚本常用的 epoch 数较少，并配合 early stopping 和逐 epoch 衰减的学习率，所以 10% 可能只相当于 1 个 epoch 甚至更少（这是推断，需作者确认）；结论也可能只在欠训练的条件下成立。(b) 式 4 是对整个数据集归一化，而实际训练使用 mini-batch：实现上是 batch 内按未屏蔽项归一化，还是全局常数？两者给出的有效权重不同。(c) 选择 epoch 用的验证损失是全量的还是屏蔽后的？验证集是否也做了屏蔽？(d) 数据是否标准化、指标是否在标准化尺度上计算、测试 loader 是否 drop_last（TSLib 存在这一已知问题，见 Qiu et al., 2024）。(e) 没有代码链接，也没有超参数表。
- **建议**：在附录中给出每个 backbone × 数据集的 epoch、batch size、学习率、patience、打分 epoch，以及式 4 的实际实现；报告训练到收敛（关闭 early stopping 或延长 epoch）时结论是否保持；投稿时提供匿名代码。

### W13【次要偏中等】外部效度：仅 L=96、stride=1、三种 backbone，没有覆盖其他设置

- **建议**：至少补充 L∈{336, 512} 中的一个（重叠程度和 element 的出现次数会随之变化）、一个 channel-independent 的 Transformer（如 PatchTST；Nie et al., 2023），并讨论在时间序列基础模型微调场景（如 Chronos：Ansari et al., 2024；TimesFM：Das et al., 2024）中这些单元是否仍有意义。stride>1 会改变 c(t,d)，也值得讨论。

---

## 5. 领域层面的分析

**研究脉络**：数据剪枝主要有三条线。(1) 静态打分（EL2N/GraNd：Paul et al., 2021；forgetting：Toneva et al., 2019；TDDS：Zhang et al., 2024；CCS：Zheng et al., 2023；Moderate：Xia et al., 2023）；(2) 动态/在线选择（InfoBatch：Qin et al., 2024；repeated random sampling：Okanovic et al., 2024；online batch selection：Loshchilov and Hutter, 2015；Katharopoulos and Fleuret, 2018）；(3) "可学习 vs 噪声"的区分（RHO-Loss：Mindermann et al., 2022；Co-teaching：Han et al., 2018；loss truncation：Kang and Hashimoto, 2020）。时间序列方向主要沿用 (1)。本文实际上把 (3) 的思想放到了更细的监督粒度上。

**隐藏联系**：

1. **与稳健统计的联系**（W1）：occurrence 粒度的 high-end mask ≈ 一次性的 trimmed least squares；element 粒度 ≈ 按目标的 trimmed loss；window 粒度 ≈ 按样本的 trimmed loss。按这个框架，可以预测细粒度单元在 MAE/中位数上收益更大、在 MSE/尾部上收益小，与论文观测一致。
2. **与 Sorscher et al.（2022）的联系**：论文称 "In our setting the opposite end pays"（§2）。但 Sorscher et al. 指出，在数据稀缺的情况下应保留简单样本、去掉困难样本。ETT 等基准规模很小，本文的发现可能正好落在这一理论预测的区域内，而不是"与分类相反"。建议作者据此重新表述，并通过训练集截断实验（例如只用 25%/50% 的训练序列）检验最优端是否翻转。
3. **与多步预测 loss shaping 的联系**（W4）：occurrence 的选择可能隐式改变了各 lead time 的权重（Hounie et al., 2024）。
4. **与 Rho-1 的联系**（Lin et al., 2024）：在 token 级别屏蔽损失、同时保留输入，与本文的"保留输入、屏蔽目标"同构。作者已经引用，但可以进一步讨论 Rho-1 中"参考模型的超额损失"在时间序列中的对应物，即 §7.1 的交叉拟合残差。

**研究空白与悖论**：

- **悖论**：训练目标是 MSE，屏蔽却主要改善 MAE，MSE 不显著。这说明"剪枝改进了训练目标本身"的直觉不成立，或者说它是在损失和指标不匹配的条件下才成立的。
- **空白**：没有一种方法同时考虑粒度、预算和打分时刻的联合选择；也没有给出 TS 剪枝在"保留输入时，被屏蔽目标的信息仍经由输入通道进入模型"这一点上的理论刻画。

**概念对比表**：

| 单元 | 项数 | 跨窗口？ | 稳健统计对应 | 预期 MAE 收益 | 预期尾部影响 |
|---|---|---|---|---|---|
| window | T×D | 否 | 样本级截断 | 小 | 小 |
| time col / channel row | D / T | 否 | 块级截断 | 中 | 小 |
| occurrence | 1 | 否 | 逐项 LTS | 大 | 可能变差（表 2：CVaR +0.47） |
| element | c(t,d) | 是 | 按目标截断 | 中–大 | 不确定 |

---

## 6. 给作者的问题

1. 全数据 L1 或 Huber 训练，以及在线 batch 内 trimmed loss，相对本文的 mask 表现如何？在 L1 训练下 high-end mask 是否仍然有效？
2. 每个单元在 10% 预算下移除的平方残差总量占比是多少？与 occurrence top-10% 的重叠率是多少？
3. 被屏蔽的 occurrence 在 lead time 上如何分布？被屏蔽的 element 在时间轴上如何分布，是否与已知异常段或训练集末段重合？
4. 打分时的 10% epoch 具体对应几个 epoch？式 4 在 mini-batch 中如何归一化？验证集是否也做屏蔽？
5. 共有多少次运行因发散被排除，分布在哪些条件？
6. §7 为什么排除 ECL，§7.1 的分解为什么排除 solar？
7. §7.3 中的 "three rules" 分别是什么？对应结果在哪里？
8. Selective Learning 在原始自适应设置下的结果如何？
9. 剪枝率改变（如 1%、30%、50%）时，"low end never helps" 与 "window gains least" 是否仍成立？
10. 结论在按 horizon 拆分后是否一致？
11. occurrence 单元在 ECL/T=720 下的打分内存与时间开销是多少？

---

## 7. 次要问题

- §3.1：没有给出 n 与 N、L、T 的关系（n=N−L−T+1）；前 L 个时间步的 c(t,d)=0，此时式 3 无定义，应注明 element 的取值范围。
- §3.3/附录 A："⌊rm+1/2⌋" 是四舍五入，可以直接写成 round(rm)。element 的成员大小不等（序列边界处 c<T），所以"移除约 r 份额"只是近似，应报告实际份额。
- 式 2 中 window 写作 "{(i,t,d)}_{t∈T_i, d}"，下标风格与其他行不一致。
- 图 1 中 element 单元标注为 "≈T terms"，与式 2 中的 c(t,d) 一致，但建议在图注中说明边界情况。
- 引用歧义：同时有 "Liu et al., 2024"（iTransformer）、"Liu et al., 2025"（Enlightenment Period）和 "Liu et al., 2026"（TimeLAVA），正文中容易混淆，建议加 a/b 或方法名。
- 附录 B 标题写 "Scores and treatments of Section 7.2"，但插值处理也被 §7.2 末段使用，可以保留；不过 MC-dropout 的 K=5 只出现在表 3，正文应同时说明。
- 表 1 至表 7 都没有 seed 标准差；"Datasets better" 可以加上显著性标记。
- 摘要句子较为密集，"the score ranks targets by the error that the rest of the series leaves unexplained" 在读完 §7.1 之前难以理解，建议改写。
- §5 中 "never gains more than 0.3 percentage points"（§1）与 "between −0.05% and +0.51%"（§5）的表述层次不同（前者是等价检验结论，后者是点估计），建议统一。
- 图 2(b) 只展示 RMSE，而正文表格用 MSE。建议在图注中注明 RMSE 的变化 ≈ MSE 变化的一半（平方根关系），避免读者误以为是两个独立的发现。

---

## 8. TMLR 标准评估与总体推荐

**(a) 主张是否有准确、令人信服、清晰的证据支持**：**部分满足**。实验规模和以数据集为单位的推断方式值得肯定，表格数值内部一致。但存在以下问题：
- 核心机制主张（剪枝单元重要、收益来自屏蔽不可解释的误差）存在一个强有力的替代解释，即稳健损失或损失截断（W1）以及残差质量混杂（W2），而论文没有排除；
- 仅测试了单一预算，也缺少随机基线（W3）；
- 时间序列特有的混杂（lead time、时间位置）没有分析（W4）；
- 部分结论只有"不显著"而没有"等价"的证据（W5），推荐与自身统计结论不一致（W7）；
- §7.3 的结论没有展示数据（W10）。

**(b) TMLR 读者是否感兴趣**：**是**。"决策单元"这一视角、element 这一跨窗口单元，以及误差分布层面的刻画，对时间序列和数据选择两个社区都有启发；即使机制最终被归结为稳健截断，系统性的粒度对比本身也有价值。

**总体推荐**：**Major revision**。
如果补充 W1–W4 的实验（稳健损失与在线截断基线、残差质量匹配、随机基线与预算扫描、lead time 与时间位置分析），并修正 W5、W7、W10，论文有望达到接收标准。若 W1 的实验显示 L1/Huber/在线截断能达到同等效果，作者需要相应地重新定位贡献（例如定位为"监督粒度上的稳健训练的系统研究"）。

**置信度**：4/5。审稿人熟悉数据剪枝与时间序列预测文献。部分推断（TSLib 的 epoch 设置、打分内存估算）已标注为推断，需作者确认。

---

## 参考文献（Harvard 格式）

- Ansari, A.F. et al. (2024) 'Chronos: Learning the language of time series', *Transactions on Machine Learning Research*.
- Das, A., Kong, W., Sen, R. and Zhou, Y. (2024) 'A decoder-only foundation model for time-series forecasting', *ICML*.
- Fu, Y. et al. (2025) 'Selective learning for deep time series forecasting', *NeurIPS*.（论文引用）
- Han, B. et al. (2018) 'Co-teaching: Robust training of deep neural networks with extremely noisy labels', *NeurIPS*.
- Hounie, I., Porras-Valenzuela, J. and Ribeiro, A. (2024) 'Loss shaping constraints for long-term time series forecasting', *ICML*.
- Huber, P.J. (1964) 'Robust estimation of a location parameter', *The Annals of Mathematical Statistics*, 35(1), pp. 73–101.
- Kang, D. and Hashimoto, T. (2020) 'Improved natural language generation via loss truncation', *ACL*.
- Katharopoulos, A. and Fleuret, F. (2018) 'Not all samples are created equal: Deep learning with importance sampling', *ICML*.
- Lin, Z. et al. (2024) 'Not all tokens are what you need for pretraining', *NeurIPS*.
- Loshchilov, I. and Hutter, F. (2015) 'Online batch selection for faster training of neural networks', arXiv:1511.06343.
- Mindermann, S. et al. (2022) 'Prioritized training on points that are learnable, worth learning, and not yet learnt', *ICML*.
- Nie, Y., Nguyen, N.H., Sinthong, P. and Kalagnanam, J. (2023) 'A time series is worth 64 words: Long-term forecasting with transformers', *ICLR*.
- Okanovic, P. et al. (2024) 'Repeated random sampling for minimizing the time-to-accuracy of learning', *ICLR*.
- Paul, M., Ganguli, S. and Dziugaite, G.K. (2021) 'Deep learning on a data diet: Finding important examples early in training', *NeurIPS*.
- Qin, Z. et al. (2024) 'InfoBatch: Lossless training speed up by unbiased dynamic data pruning', *ICLR*.
- Qiu, X. et al. (2024) 'TFB: Towards comprehensive and fair benchmarking of time series forecasting methods', *Proceedings of the VLDB Endowment*.
- Rousseeuw, P.J. (1984) 'Least median of squares regression', *Journal of the American Statistical Association*, 79(388), pp. 871–880.
- Shen, Y. and Sanghavi, S. (2019) 'Learning with bad training data via iterative trimmed loss minimization', *ICML*.
- Sorscher, B. et al. (2022) 'Beyond neural scaling laws: Beating power law scaling via data pruning', *NeurIPS*.
- Toneva, M. et al. (2019) 'An empirical study of example forgetting during deep neural network learning', *ICLR*.
- Xia, X. et al. (2023) 'Moderate coreset: A universal method of data selection for real-world data-efficient deep learning', *ICLR*.
- Zhang, X. et al. (2024) 'Spanning training progress: Temporal dual-depth scoring (TDDS) for enhanced dataset pruning', *CVPR*.
- Zheng, H., Liu, R., Lai, F. and Prakash, A. (2023) 'Coverage-centric coreset selection for high pruning rates', *ICLR*.
