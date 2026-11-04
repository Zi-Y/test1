# 审稿意见

**论文**：Prune the Supervision, Not the Windows: Decision Units for Data Pruning in Time-Series Forecasting（匿名投稿，TMLR under review）
**审稿人**：独立审稿（review_haiku_high）
**日期**：2026-11-04
**阅读范围**：PDF 第 1–14 页全部内容，包括图 1–3、表 1–7、附录 A–D 与参考文献。文中 "p.X" 指 PDF 页码。

**独立性说明**：审稿仅基于论文 PDF。未读取 reviews 目录中任何其他审稿人的文件。工作过程中曾对该目录执行过一次列目录操作，只看到了文件名，未打开任何内容。

---

## 1. 论文摘要（审稿人自述）

本文关注滑动窗口式时间序列预测的数据剪枝。每个窗口包含 T×D 个监督项，而同一个目标值会被约 T 个不同窗口监督（式 (1)，p.4）。作者指出，现有预测领域的剪枝方法沿用了分类任务的"样本（窗口）"单位。

论文的做法是固定打分（训练进行到 10% 的 epoch 时，对每个监督项记录平方残差，并取其均值）和预算（屏蔽 10% 的损失项），只改变"决策单元"，即一次决策同时屏蔽的损失项集合。五种单位分别是：窗口、时间列 (i,t)、通道行 (i,d)、单个 occurrence (i,t,d)，以及 element (t,d)，后者汇总同一目标在所有窗口中的 occurrence（式 (2)，p.4）。屏蔽采用加权损失（式 (4)），所有窗口与输入都保留，屏蔽在训练前一次性确定。

实验覆盖 iTransformer、DLinear、TimeMixer 三种 backbone，7 个数据集，4 个预测长度和 5 个随机种子。推断单位为数据集（n=7），每个指标都与同种子的全量训练对比。

主要结果如下：

1. 屏蔽残差最高的 10%，平均 MAE 下降 2.03%（窗口）到 4.72%（occurrence），且在 7/7 数据集上成立（表 1，p.6）。屏蔽残差最低的 10% 几乎没有收益（图 1，p.2）。
2. 收益主要体现在误差分布主体：中位绝对误差下降 4.42%–12.45%，第 95 分位接近零，最差 10% 误差（CVaR@90%）无可检测变化（表 2，p.7）。
3. MSE 平均改善 1.73%–2.81%，但跨数据集不显著。作者推荐通道行作为默认单位（p.7）。
4. 第 7 节以 iTransformer 为对象。作者认为 element 屏蔽针对的是"序列其余部分无法解释的误差"。仅在训练早期打一次分的分数，接近代价高得多的分数（表 3，p.8）。element 屏蔽在 MAE 上优于 Selective Learning。窗口级等间距剪枝与随机剪枝持平（p.9）。

---

## 2. 主要主张清单与证据充分性

| 编号 | 主张 | 位置 | 支持证据 | 评价 |
|---|---|---|---|---|
| C1 | 高端屏蔽在所有单位上降低 MAE，且 7/7 数据集成立（2.03%–4.72%） | 摘要 p.1；表 1 p.6；表 4–6 p.13–14 | 表 1 的三 backbone 均值与表 4–6 的逐 backbone 数值一致（已核对）。逐数据集符号一致 | **基本充分**（描述性结论）。注意摘要中的 "all seven datasets" 指 backbone 平均，逐 backbone 并不全部成立（如 TimeMixer 的窗口单位在 ETTh1 为 +0.14，表 6） |
| C2 | 低端屏蔽"never helps"（摘要）；"gain never more than 0.3 points"（p.2、p.5） | 摘要 p.1；图 1；第 5 节 p.5 | 仅 MAE 的低端均值以柱状图给出（图 1）。MSE 低端的 10 个平均值只有区间描述（"−0.05% 到 +0.51%"）。逐数据集低端结果无表格。两个单 backbone 单元为 −0.06% 与 −0.19%（TimeMixer MSE），即略有改善 | **部分充分**。"never helps" 与作者自己的数据不一致，应改为"至多 0.3 点、不显著"，并补充低端完整表格 |
| C3 | 窗口单位收益最小；occurrence 排名第一（平均秩 1.29） | 第 5 节 p.5；表 1 | 平均秩与 Friedman 检验（p=0.002）有数据支持；MAE 排序一致 | **基本充分**（仅限 MAE 与中位数）。MSE 排序不显著（p=0.525） |
| C4 | 收益集中于误差分布主体，尾部不变（"the gain lives in the bulk"） | 第 6 节 p.6–7；表 2；图 2 | 中位数与 MAE 的变化有数据；CVaR 表 2 中 chan row −0.52、occ +0.47 | **部分充分**。"does not change measurably" 不等于无变化，且论文自设的 ±0.3 等价界被 CVaR 的点估计超出（见 §4 问题 4）。q95 数值只在图中给出 |
| C5 | MSE 改善较小，原因是 MSE 对大误差更敏感 | 第 6 节 p.7（"Why MSE moves less"） | RMSE 降幅约为 MSE 降幅的一半（图 2b，与数学关系一致） | **推测性**。"尾部机制"未被直接检验，应按误差分位数分解 ΔMSE 来验证 |
| C6 | 推荐通道行为默认单位 | p.7（"Which unit to use"）；结论 p.9 | MSE 均值最低（−2.81），但 Friedman p=0.525 不显著；在 ECL 上通道行的 MSE 为 +0.05（表 1） | **不充分**。依据是同一测试集上的事后选择，且 MAE 最优单位为 occurrence |
| C7 | 第 7 节：element 屏蔽针对"序列其余部分无法解释的误差"（ρ=0.998） | 第 7.1 节 p.8 | 仅在 iTransformer、5 个数据集、20 个 dataset-horizon 组合上分析 | **推测性**。高秩相关可能来自共同因素（局部波动、幅值、边界位置），需排除（见 §4 问题 12） |
| C8 | 结构化的"哪些目标被屏蔽"重要：随机同结构掩码几乎无收益（−0.16% MSE） | 第 7.1 节 p.8 | 单次随机抽样；仅 element 单位；仅报告 MSE | **部分充分**。需要五个单位的随机对照与多次抽样，并同时报告 MAE |
| C9 | 仅在训练早期打一次分的均值残差，接近代价高得多的分数 | 摘要 p.1；表 3 p.8；第 7.2 节 p.8–9 | MC-dropout 在 MSE 上领先 0.24（p=0.045，6 个数据集中的 5 个）；TDDS、平均损失、RobustTSF 表现更差 | **基本充分**，但 MC-dropout 本身成本低（10% + 5 次前向），摘要的表述容易误导 |
| C10 | 屏蔽优于修复（插值） | 第 7.2 节 p.8；附录 B p.12 | 表 3 中插值 MSE 为 +4.32%（不同于屏蔽的 −2.81%） | **不充分**。插值同时修改了输入与目标（附录 B、p.11），两者扰动不同，不构成同一变量的对照 |
| C11 | element 屏蔽在 MAE 上优于 Selective Learning（两种形式） | 摘要 p.1；第 7.2 节 p.9；表 3 | 配对差异与 p 值已列出（如 2.03 点、p=0.013） | **基本充分**，但仅限 iTransformer；摘要中的"on all six datasets"在正文中没有逐数据集的数值 |
| C12 | 窗口级等间距剪枝 30% 与随机剪枝持平，可在不打分的情况下节省优化步数 | 第 7.3 节 p.9 | 等间距与随机在 30% 下 ±0.3 点内持平（p=0.002）；50%、70% 时等间距领先 | **部分充分**。"三个基于打分的窗口规则"在全文中未定义；未与"等步数的全量训练"比较 |

---

## 3. 优点

1. **成对实验设计**。每个屏蔽运行与同种子的全量运行配对，并以百分比变化报告（p.5；附录 A，p.11）。这降低了种子方差的影响，也使得效应量的解释直接。
2. **诚实的推断单位声明**。作者明确指出，同一数据集的 4 个 horizon 共享训练序列，三个 backbone 共享数据集，因此运行与 dataset-horizon 组合都不独立，并以数据集作为推断单位（p.5）。
3. **数值一致性良好**。我核对了表 1 的三 backbone 均值与表 4–6 的逐数据集数值（如 occurrence MAE：(−3.67 −6.77 −3.73)/3 = −4.72；element MSE ECL：(0.44+1.47+3.35)/3≈1.75；iTransformer element MAE（6 数据集）≈−3.97，与表 3 一致）。图 2b 中 RMSE 降幅约为 MSE 降幅的一半，与数学关系一致。
4. **控制实验的意识**。论文设置了随机同结构掩码（p.8）、插值替代（p.8）、Selective Learning 的两种形式（p.8–9），以及窗口级等间距与随机对照（p.9）。这些都比较有价值。
5. **单位形式化清晰**。式 (2) 把五种单位统一为同一符号体系，每个单位的项数（T×D、D、T、1、≈T）也写得明确。
6. **分位数分析有启发性**。把"典型误差"（中位数、MAE、q90）与"最大误差"（CVaR、MSE）区分开，并给出了随分位数变化的图（图 2a，图 3），是本文最有价值的经验发现之一。

---

## 4. 缺点与修改建议（按重要性排序）

标记说明：【必须】指不补充则结论难以成立；【重要】指显著影响结论的强度；【建议】指提升严谨性或清晰度。

### 问题 1【必须】缺少与鲁棒损失/离群值截断的直接对照，主要收益可能只是鲁棒训练效应

**问题描述**：高残差屏蔽在数学上等价于"对高残差项赋予权重 0"，即一种 trimmed loss。论文的多个结果恰恰是鲁棒训练的典型特征：
- 收益集中在误差分布主体，中位数降幅最大（12.45%），第 95 分位接近零（表 2，图 2a）；
- MAE 收益大于 MSE 收益（图 2b）；
- CVaR 不变（表 2）；
- DLinear（线性、低容量模型）的收益最大（表 5，occurrence MAE 平均 −6.77，而 iTransformer 与 TimeMixer 均约为 −3.7）。

第 2 节已引用 loss truncation（Kang & Hashimoto, 2020）与 Co-teaching（Han et al., 2018），但全文没有任何鲁棒损失或截断的实验对照。

**为何重要**：如果全量训练使用 L1 或 Huber 损失，或使用动态 loss truncation，即可复现大部分收益，那么"决策单元"与"数据剪枝"的叙事就只剩次要地位。

**具体建议**：
- (a) 增加全量训练的 L1 损失与 Huber 损失（δ 取残差分位数）作为基线；
- (b) 增加动态 loss truncation（Kang & Hashimoto, 2020）作为基线，每 epoch 重算截断阈值，并与同预算的静态屏蔽比较；
- (c) 可选：增加基于残差的软加权（reweighting，参考 Ren et al., 2018）；
- (d) 若 L1 基线接近或超过 occurrence 屏蔽的收益，应相应修改标题、摘要与贡献。

### 问题 2【必须】"数据剪枝"的表述与实际操作不符；与数据剪枝文献的实验比较缺失

**问题描述**：
- 论文自述 "A decision removes supervision, not data"（p.4），所有屏蔽方式都保留全部窗口与输入（式 (4)）。主实验是"等 epoch、等优化步数的加权重训练"，不节省计算（p.4："Because a mask keeps every window, it does not shorten an epoch"）。这与标题 "Pruning"、引言把问题描述为"数据剪枝"（p.1–2）的表述不一致。
- 相关工作列举了 EL2N、GraNd、forgetting、CCS、Moderate、TDDS、InfoBatch 等方法（p.3），但实验中没有任何一种在窗口单位上作为基线。TDDS 与 MC-dropout 只在 element 单位上比较（表 3）。论文主张"窗口是继承自分类的粗单位"，却没有与该单位上的标准方法比较，只与论文自己的窗口行比较。

**具体建议**：
- (a) 术语上改为 "loss masking" 或 "supervision pruning"，并在摘要与标题中明确"不删除数据"；
- (b) 增加窗口级标准基线（EL2N、forgetting、InfoBatch 的窗口版本），在相同 10% 与 30% 预算、相同 epoch 数下比较；
- (c) 说明 TDDS 等图像分类方法在时间序列上的改编细节（附录 B 只描述了屏蔽方式，未描述打分的适配）。

### 问题 3【必须】计算成本与公平比较缺失

**问题描述**：
- 屏蔽运行与全量运行的 epoch 数相同（p.5）。
- 方法的实际流程是：先运行 10% 的全量训练 → 打分 → 从头训练屏蔽模型。总开销约为全量训练的 1.1 倍（评估前向推理的成本低于训练步，但论文未用墙钟时间报告）。表 3 的 "cost of the score" 列仅标注为 "10%"，容易让读者低估开销。
- 第 7.3 节声称"节省 30% 的优化步数"，但没有与"同步数的全量训练（即 70% epoch）"比较。

**具体建议**：
- (a) 报告墙钟时间与 FLOPs；
- (b) 增加计算匹配的全量基线（约 1.1× epoch，或在 early stopping 允许的同等预算下）；
- (c) 第 7.3 节增加同步数的全量训练（70% epoch）对照，并报告 early stopping 的实际 epoch 数。

### 问题 4【重要】统计与方差报告不足；部分"无变化"结论超出证据

**问题描述**：
- **种子方差未报告**。结果已对 4 个 horizon 与 5 个种子取平均，没有给出种子间标准差、配对差的标准差或置信区间。例如 iTransformer 上 ETTh1 的窗口单位 MAE 变化仅为 −0.28%（表 4），weather 为 −0.36%。这些量级可能与种子噪声相当，"7/7 数据集"的结论对种子噪声敏感。
- **排除运行未报告**。"Runs that diverge are excluded"（p.5）。排除的数量与清单未给出，可能造成选择偏差。
- **等价检验与论文自设界不一致**。作者使用 ±0.3 点的界（p.5）。但表 2 中 CVaR 的点估计分别为 chan row −0.52、occurrence +0.47，均超出 ±0.3。因此 "no unit differs measurably"、"does not change measurably"（p.6）无法由作者自己的等价检验支持。"未检出差异"不等于"没有变化"。
- **多重比较与检验方法**。作者明确未做多重比较校正（p.5）。在 n=7、k=5 时，Friedman 的卡方近似并不可靠，应改用精确或置换检验。
- **符号检验的表述**。"an exact sign-flip test cannot fall below 1/64"（p.5）指的是双侧检验；单侧的最小 p 值为 1/128。论文同时使用单侧检验，需要说明各检验的侧别。

**具体建议**：
- (a) 报告种子级配对差、按数据集与种子的两级 bootstrap 置信区间；
- (b) 对"无变化"类结论，给出 TOST 等价检验的结果，并以不超过 ±0.3 的界为准；
- (c) 对 Friedman 检验改用精确或置换检验；
- (d) 报告被排除的运行数量与原因。

### 问题 5【重要】仅在单一预算（10%）与单一打分时点（10% epoch）下评估，结论的普适性未检验

**问题描述**：
- 所有主结果都在 r=10% 下得到。高端屏蔽的收益在更大或更小的预算下是否持续、是否会反转，论文没有回答。第 7.3 节只在窗口单位上使用了 30% 的预算。
- 打分时点固定为 10% 的 epoch，但摘要的核心主张之一是"a score read once early in training"。早期与晚期打分的对比缺失。
- 第 3.3 节定义了 Tukey 自适应预算（上四分位数加 1.5 倍 IQR 以上的 element 被屏蔽，p.4），第 7.3 节称"here at the adaptive budget"（p.9）。但该预算下的实际屏蔽比例没有报告，也没有与固定 10% 的主结果对照。

**具体建议**：
- (a) 给出预算（如 1%、5%、10%、20%、30%、50%）× 单位 × 指标的曲线图；
- (b) 给出打分时点的扫描（如 1%、5%、10%、20%、50% 的 epoch）；
- (c) 报告自适应预算下每个数据集的屏蔽比例，并给出与表 1 同格式的结果。

### 问题 6【重要】机制分析覆盖的单位与最强结果不一致；关键的潜在混杂未检验

**问题描述**：
- 表 1 中 MAE 最优的单位是 occurrence（−4.72），其次是 element（−3.81）。但第 7 节的机制分析只针对 element 单位，并且只在 iTransformer 上进行（iTransformer 的 occurrence 收益与 TimeMixer 相近；DLinear 的收益最大，但未做分析）。
- occurrence 屏蔽的对象是单个（窗口，目标）对。其高残差可能来自输入上下文与目标不一致，而不是"目标本身无法由其余序列解释"。第 7.1 节的解释对它不直接适用。
- **horizon 位置混杂**：残差通常随 horizon 位置增大而增大。高残差 occurrence 可能主要位于远端 horizon，因此屏蔽效果可能部分是"对远端 horizon 的重加权"（类似 Hounie et al., 2024 的 horizon loss shaping）。论文没有报告被屏蔽 occurrence 的 horizon 位置分布。
- **实际屏蔽比例偏离名义预算**。element 的 c(t,d) 在序列两端较小（p.4，"T of them away from the ends"），因此"about the same share r"（p.4）并不精确。论文没有报告各单位的实际屏蔽比例。

**具体建议**：
- (a) 报告被屏蔽 occurrence 与 element 在 horizon 位置、时间位置（是否靠近序列两端）、通道上的分布；
- (b) 在 DLinear 上重复第 7.1 节的分解（DLinear 的收益最大，应作为主要分析对象）；
- (c) 报告每个单位的实际屏蔽比例（损失项层面）；
- (d) 检验"horizon 重加权"假说：例如，对 horizon 位置做匹配的随机屏蔽，或直接对比按 horizon 加权的损失。

### 问题 7【重要】插值对照存在混杂，"屏蔽优于修复"的结论不成立

**问题描述**：插值处理同时替换了训练切分中的输入与目标（附录 B，p.11："an interpolated value enters both"）。因此它不是"只改变目标"的对照。此外，被选中的高残差点多为尖峰，插值会系统性地抹平峰值，结果自然变差。这更说明"这些点携带了信号"，而不是"屏蔽比修复更好"。

**具体建议**：只替换损失中的目标（输入保持原值），并与"只替换输入"分别比较。在摘要中把"masking beats repairing"改为更具体的表述。

### 问题 8【重要】随机对照不足

**问题描述**：随机同结构掩码只在 element 单位上做了，且只报告了 MSE（−0.16%，p.8）。该结果来自单次随机抽样。主结论的指标是 MAE，但随机对照没有 MAE 结果。五个单位都缺少随机对照。

**具体建议**：
- (a) 对五个单位分别做同预算随机屏蔽；
- (b) 多次抽样（如 20 次），报告分布，而不是单次结果；
- (c) 同时报告 MAE 与 MSE；
- (d) 增加"中间残差段"的屏蔽（既非最高端也非最低端），检验收益是否随残差单调变化。

### 问题 9【重要】低端"never helps"的表述与数据不一致，低端结果缺少表格

**问题描述**：
- 摘要称"masking the lowest-residual 10% never helps"（p.1）。
- 第 5 节的平均值区间为 "−0.05% 到 +0.51%"（p.5），即 MSE 的某个平均值有 −0.05% 的改善。
- 单 backbone 的两个例外（TimeMixer 的 MSE，chan row −0.06%、element −0.19%）同样是略有改善（p.5）。
- 低端的 MSE 结果、逐数据集结果都没有表格，只有图 1 中 MAE 的均值条形图。

**具体建议**：把摘要改为"低端屏蔽的收益至多 0.3 点，且不显著"，并补充与表 1 格式一致的低端完整表格（逐数据集、逐 backbone、MAE 与 MSE）。

### 问题 10【重要】推荐默认单位（通道行）的依据不足，且与 ECL 结果冲突

**问题描述**：
- MAE 最优的单位是 occurrence（−4.72），通道行为 −3.46。
- 通道行的优势只在 MSE 均值上（−2.81 对 occurrence 的 −2.16），且不显著（Friedman p=0.525，p.7）。
- ECL 上通道行的 MSE 为 +0.05（表 1），即变差。论文只说"ECL 上 MSE 上升"（p.6），没有把这一点纳入推荐。
- 推荐依据是同一测试集上的事后比较，没有使用留出的选择集。

**具体建议**：按指标给出明确推荐（MAE 用 occurrence，MSE 用通道行并注明 ECL 的例外）；或给出基于验证集的选择规则，并在测试集上只报告一次。

### 问题 11【建议】单位大小与数据维度、horizon 混杂，ECL 的高维解释未经检验

**问题描述**：单位的大小（T×D、D、T、1、≈T）随 D 和 T 变化。ETT 的 D=7，ECL 的 D=321，solar 的 D=137，weather 的 D=21。论文把 ECL 上的反转归因于"高维"（p.6；附录 D，p.13），但没有做任何检验。因此"单位效应"与"数据集维度效应"无法分离。

**具体建议**：给出每个数据集、每个单位的平均单位大小；对 ECL 做通道子集实验（如随机抽取 7 个通道），检验"高维"假说。

### 问题 12【建议】第 7.1 节的分解需要更严格的定义与稳健性检验

**问题描述**：
- "unexplained error" 依赖于"该折监督被屏蔽的模型"（p.8）。该模型的训练监督减少了 20%，与打分模型不同。
- 打分与"未解释误差"的 Spearman 相关高达 0.998（中位数）。这一数值异常高，可能来自共同因素（局部波动、目标幅值、窗口水平、距序列边界的位置），而非"不可由其余序列解释"这一机制。
- "a strict relabelling ... (median 0.990 over nine combinations)"（p.8）中的"nine combinations"没有定义。
- 结论中"the error that the rest of the series leaves unexplained"的因果表述超出了分析所能支持的范围（作者在 p.8 自己也承认"neither says that a target is unpredictable in principle"）。

**具体建议**：
- (a) 计算打分与简单特征（局部标准差、|y − 移动平均|、目标幅值、边界位置）的相关及偏相关；
- (b) 定义 "nine combinations"，并解释 "strict relabelling" 的含义；
- (c) 在 DLinear 上复现分解；
- (d) 把因果语言降级为假设性语言。

### 问题 13【建议】第 7.3 节的窗口剪枝结论缺少必要的定义与对照

**问题描述**：
- "the three rules that select windows by their score"（p.9）在全文中没有定义。这三个规则是什么，论文中找不到。
- 50%、70% 的剪枝结果只在正文中提及，表中没有。
- 等间距剪枝等价于增大滑动窗口的步长（stride）。在相同步数下，直接以更大 stride 训练是最自然的基线，论文没有比较。
- "none of the three rules ... does better than random pruning"与"equal spacing is ahead at 50% and 70%"之间的关系需要说明：等间距优于随机，但基于打分的规则不优于随机。两种对比的对象应明确。

**具体建议**：定义三个规则；给出完整表格；增加"以更大 stride 训练"的对照；引用滑动窗口采样相关工作。

### 问题 14【建议】实现细节缺失，影响可复现性

**问题描述**：
- 式 (4) 按未屏蔽项的数量归一化。常见实现是对屏蔽后的损失再对所有项取均值。两者相差约 10% 的有效学习率，对 Adam 影响较小，但对 SGD 与学习率调度可能有影响，需要说明。
- 式 (4) 的归一化是对全数据集进行的，而 minibatch 训练是逐批归一化。两者不一致，需要说明实际实现。
- 数据的标准化方式（全局或逐通道、基于训练切分）没有说明。残差排序对通道尺度敏感，直接影响通道行与时间列单位。
- 未报告：训练 epoch 数、学习率、批大小、早停耐心值、数据切分比例、硬件与运行时间，以及代码与数据可用性声明。
- "10% of the full number of epochs"中的"full number"是名义值还是早停后的实际值？
- 早停以验证集 MSE 选择 epoch，而主指标是 MAE。这一选择对结论的影响需要敏感性分析。

**具体建议**：补充上述实现细节；给出代码（匿名仓库）与复现配置；做按验证 MAE 选择 epoch 的敏感性分析。

### 问题 15【建议】只测试了静态屏蔽，动态版本与最接近的先行工作缺少比较

**问题描述**：论文的屏蔽在训练前一次性确定，并在整个训练中固定。第 2 节引用的 Liu et al. (2025, Enlightenment period) 恰好是"早期丢弃高损失样本，之后再恢复"。它与本文最接近，但没有比较。Selective Learning 的"每 epoch 重打分"版本表现更差（表 3），说明动态版本未必有利，但论文没有测试本文打分方法的动态版本。

**具体建议**：增加 (a) 每隔 k 个 epoch 重打分的动态屏蔽；(b) 早期丢弃后恢复（Liu et al., 2025）作为对照。

### 问题 16【建议】相关工作的定位与若干引用需要核实

**问题描述**：
- 论文称 Selective Learning 是"唯一在预测中于窗口以下屏蔽监督的方法"（p.3）。这一断言过于绝对，需要更保守的措辞，并应检索时间序列中按时间步加权损失的相关工作（如 Hounie et al., 2024 的 horizon 加权；You et al., 2024 的 loss-weighted sampler）。
- 引言称 "Pennino & Gabbrielli, 2026" 为以样本为单位剪枝的例子（p.1）。该文题目为 "Data Mixture Search"，更像数据集层面的混合，需要核实其是否符合引用语境。
- 相关工作中把 "Camporeale, 2025"（PARIS）描述为预测中的闭式影响剪枝（p.3）。但其题目指向 imbalanced scenarios，需要核实它是否是预测方法。
- 2025–2026 年的多篇引用（Chen 2026、Wu 2026、Xu 2026、Li 2026、Liu 2026 TimeLAVA、Pennino & Gabbrielli 2026 等）我在本次审稿中无法核实。作者应确认标题、arXiv 编号和发表信息。
- 论文只使用三个 backbone，缺少 PatchTST（Nie et al., 2023）等广泛使用的补丁式 Transformer 模型。至少应在讨论中说明适用范围。

**具体建议**：修正上述引用与措辞；增加 PatchTST 或说明其缺席的理由。

### 问题 17【建议】对收益原因的解释超出证据，尾部风险未讨论

**问题描述**：
- "gain lives in the bulk"（p.6）中，中位数改善与尾部不变可能部分由 MAE 与 MSE 的度量定义决定（MAE 对中位数敏感，MSE 对尾部敏感），并非"高残差屏蔽只改善普通误差"的独立证据。
- "Why MSE moves less"（p.7）是推测，应注明。
- 屏蔽高残差目标，可能系统性地丢弃极端事件（如用电高峰、太阳能骤降、天气极值）的监督。这种风险在整体 CVaR 中被稀释，论文没有按事件条件化分析，也没有讨论在电力、气象预警等安全相关场景下的应用风险。

**具体建议**：
- (a) 按误差分位数分解 ΔMSE 的贡献，检验"尾部机制"；
- (b) 增加极端事件条件下的误差分析（如高于训练集 95 分位的目标上的 MAE 与召回）；
- (c) 在讨论中加入应用风险与伦理部分。

---

## 5. 领域层面的分析

### 5.1 研究脉络

本文位于三条研究线的交叉处：

1. **图像分类的样本级数据剪枝**：遗忘事件（Toneva et al., 2019）、EL2N 与 GraNd（Paul et al., 2021）、覆盖导向的核心集（CCS，Zheng et al., 2023）、Moderate（Xia et al., 2023）、InfoBatch（Qin et al., 2024）、TDDS（Zhang et al., 2024）。核心争论是"丢弃易样本还是难样本"（Sorscher et al., 2022；Ayed & Hayou, 2023）。
2. **噪声与鲁棒学习**：Co-teaching（Han et al., 2018）、loss truncation（Kang & Hashimoto, 2020）、自步学习（Kumar et al., 2010）、课程学习（Bengio et al., 2009）、样本重加权（Ren et al., 2018）。核心假设是"高损失样本可能是噪声，应当截断或降权"。
3. **时间序列预测的特定方法**：Selective Learning（Fu et al., 2025）、Enlightenment period（Liu et al., 2025）、RobustTSF（Cheng et al., 2024）、horizon 损失塑形（Hounie et al., 2024）、CLMFormer（Li et al., 2026）、通道影响（Wang et al., 2025）。

**本文的位置**：它把第 (2) 条线的"截断高损失"思路，放入第 (1) 条线的"打分-预算"框架，并引入"决策单元"这一维度。最有价值的贡献是对"单位"的系统比较，以及对重叠窗口结构的分析。

### 5.2 隐藏联系

- **重叠窗口与有效样本数**：一个 element 有约 T 次监督。窗口级剪枝只会减少 element 监督次数的约 10%，而 element 单位一次性移除该目标的全部监督。这一结构性差异是论文的核心机制假设，值得形式化，例如报告每个 element 被移除的监督比例的分布。
- **残差、horizon 与边界的潜在混杂**：残差随 horizon 增大，边界处的 c(t,d) 较小。这些都可能与屏蔽选择相关，需要在分析中控制（见问题 6）。
- **与标签噪声的类比**：在回归中，"无法由其余序列解释的目标"可被视为一种连续型标签噪声。这解释了为什么"高端"有用。与分类中"丢弃易样本"的结论相反，这一差异是回归任务的特点，值得在讨论中明确。
- **MAE 与 MSE 的分歧**：截断 MSE 的估计量会向条件中位数靠拢（经典鲁棒统计结果，Huber, 1964；Rousseeuw & Leroy, 1987）。因此 MAE 增益大于 MSE 增益是可以预期的，不需要"尾部机制"的额外解释。
- **CVaR 不变**：与上述理论一致。截断后的模型在尾部的预测并未改变。CVaR 的定义见 Rockafellar & Uryasev（2000）。

### 5.3 研究空白

- 窗口级标准数据剪枝方法在时间序列预测中的系统基准（缺失）。
- 动态与静态屏蔽的比较（缺失）。
- 计算匹配条件下的效率研究（缺失）。
- 极端事件与尾部风险的条件化评估（缺失）。
- 基础模型微调场景下的数据选择（仅引用，未研究）。

### 5.4 对比表：相关方法的设定

| 方法 | 单位 | 打分依据 | 成本 | 是否在本文中比较 |
|---|---|---|---|---|
| EL2N / GraNd（Paul et al., 2021） | 样本/窗口 | 早期损失或梯度范数 | 低（早期） | 否 |
| 遗忘事件（Toneva et al., 2019） | 样本/窗口 | 训练过程中的预测变化 | 高（全程） | 否 |
| InfoBatch（Qin et al., 2024） | 样本/窗口 | 损失阈值与重缩放 | 动态 | 否 |
| TDDS（Zhang et al., 2024） | 样本（改编为 element） | 跨训练轨迹的方差 | 100%（全程） | 是（表 3） |
| MC-dropout（Gal & Ghahramani, 2016） | element | 多次随机前向的方差 | 10% + 5 次前向 | 是（表 3） |
| Selective Learning（Fu et al., 2025） | element | 熵 + 辅助估计器 | 含辅助模型 | 是（表 3） |
| RobustTSF（Cheng et al., 2024） | element | 与趋势拟合的残差 | 无打分训练 | 是（表 3） |
| Enlightenment period（Liu et al., 2025） | 样本 | 早期高损失丢弃后恢复 | 低 | 否（最接近的先行工作） |
| Loss truncation（Kang & Hashimoto, 2020） | 样本/目标 | 动态截断 | 低 | 否 |
| L1 / Huber 损失 | 无屏蔽 | 不适用 | 无额外成本 | 否（建议增加） |

---

## 6. 给作者的问题清单

1. 全量训练的 L1 与 Huber 损失结果如何？动态 loss truncation 的结果如何？（问题 1）
2. 低端屏蔽的完整表格（逐数据集、逐 backbone、MAE 与 MSE）能否补充？（问题 9）
3. 种子级配对差及其置信区间是多少？被排除的 diverged 运行有多少，分别是哪些？（问题 4）
4. 预算扫描与打分时点扫描的结果如何？（问题 5）
5. 墙钟时间与计算匹配基线的结果如何？第 7.3 节的 70% 全量训练对照是否可以补充？（问题 3）
6. 各单位的实际屏蔽比例是多少（包括边界处）？Tukey 自适应预算下的屏蔽比例与结果如何？（问题 5、6）
7. 被屏蔽 occurrence 与 element 的 horizon 位置分布如何？（问题 6）
8. 五个单位各自的随机屏蔽对照（多次抽样，同时报告 MAE 与 MSE）结果如何？（问题 8）
9. 插值实验能否改为只替换目标？（问题 7）
10. 第 7.3 节的"三个基于打分的窗口规则"是什么？第 7.1 节的"nine combinations"是什么？（问题 12、13）
11. 数据标准化是如何进行的？式 (4) 的归一化在 minibatch 中如何实现？（问题 14）
12. 代码与配置能否公开（匿名仓库）？（问题 14）
13. ECL 上的"高维"解释能否通过通道子集实验检验？（问题 11）
14. 通道行的推荐是否基于选择集？能否给出按指标的明确推荐？（问题 10）
15. 对 Selective Learning 的基线，是否按原论文设定调优了超参数（如锚定估计器的校准）？
16. 第 7.1 节的分解能否在 DLinear 上重复？（问题 6、12）
17. 摘要中"on all six datasets"的 element 与 Selective Learning 的逐数据集比较数值是多少？
18. 在极端事件（高于训练集 95 分位的目标）上，屏蔽后的 MAE 与召回如何？（问题 17）

---

## 7. 次要问题（排版、笔误、符号）

1. **引用拼写**：正文 "Pravogo et al., 2020"（p.3）应为 "Prayogo et al., 2020"（参考文献 p.10 为 Prayogo）。
2. **摘要的 "all seven datasets"**：应注明为三 backbone 平均（表 1 说明），因为 TimeMixer 的窗口单位在 ETTh1 上为 +0.14（表 6）。
3. **摘要的 "never helps"**：与正文数据不一致，见问题 9。
4. **符号 T 的多重用法**：T 同时表示 horizon、时间列数与 terms 数（如 "T terms"、"T time columns"、"T of them"）。建议把 horizon 改为 H，或在首次出现处说明。
5. **"points" 与 "%" 的混用**：如 "by 2.03%" 与 "0.3 points" 同时使用，应统一。"points" 指百分比差值，应在首次定义时说明。
6. **p.4 "about the same share r"**：应量化实际偏差（见问题 6）。
7. **第 6 节 q95 数值**："at the 95th the five units lie between −0.05% and −0.91%"（p.6）只在图中给出，建议改为表格。
8. **符号不统一**：表 2 中写为 "median |e|"、"q90 of |e|"，图 2 与图 3 中写为 "median"、"q75"，建议统一。
9. **表 1 的 "Datasets better"**：建议改为 "# datasets improved"，并在表注中说明计数方式。
10. **表 3 的 "cost of the score" 列**：对 MC-dropout 的 "5 passes" 应说明一次前向相对一次训练步的成本。
11. **前向引用**：第 3.3 节（p.4）提到 "Section 7.3 also uses an adaptive budget"，建议在第 7.3 节首次出现时再定义。
12. **同名作者的 in-text 区分**：Liu et al. (2025) 与 Liu et al. (2026)，以及 Wang et al. (2024) 与 Wang et al. (2025)（两篇不同作者），在正文中需更清楚地区分。
13. **标题与内容**："Not the Windows" 可能误导读者，因为第 7.3 节仍研究窗口剪枝。
14. **图 1**：底部刻度 "change in test MAE (%)" 与 "lower is better" 的说明应在图注中给出。
15. **CVaR@90% 的定义**：应在第 4 节首次出现时给出（"最差 10% 绝对误差的均值"），目前只在表注中隐含。

---

## 8. TMLR 两项标准下的评估与总体推荐

### 标准 (a)：主张是否有准确、令人信服、清晰的证据支持

**部分满足**。

- 描述性结论（高端屏蔽降低 MAE、低端无收益、窗口单位收益最小、收益集中于误差分布主体）有较强的一致性证据，数值也相互一致。
- 但"决策单元"的因果解释缺少关键对照（鲁棒损失、随机对照、horizon 混杂），因此无法确认收益来源。
- 机制主张（"未解释误差"、"尾部机制"）仅有相关性分析，且集中在一个 backbone 上。
- "推荐默认单位"与"无变化"类主张的证据不充分，超出了作者自己的统计界。
- 与数据剪枝文献的比较缺失，使"数据剪枝"的主张缺少依托。

### 标准 (b)：是否有 TMLR 读者会感兴趣的内容

**有**。

- 时间序列预测中的样本选择、监督选择与损失加权，是实践者真实面临的问题。
- 对"重叠窗口中，监督的单位应如何定义"的系统比较，对该领域有参考价值。
- 实践建议（不要屏蔽低端、优先屏蔽高端、MAE 与 MSE 的选择不同）具有可操作性。
- 但论文对"数据剪枝"的定位与实际不符，且不节省计算，应用价值需要重新界定。

### 总体推荐

**Major revision（大修）**

理由：论文的核心经验发现有价值且数值一致，但缺少几个关键对照（鲁棒损失、计算匹配基线、窗口级标准基线），统计报告不足，多处定义缺失，且部分结论超出证据。若补充上述实验后结果仍然成立，论文可以达到 TMLR 的要求。

### 置信度

**4 / 5**

理由：我对论文的内部数值一致性与问题设定的判断较有把握，并逐页核对了表格。对 2025–2026 年文献的新颖性与准确性判断把握较低，因为我无法核实这些文献。

---

## 9. 参考文献（Harvard 格式）

Ayed, S. and Hayou, S. (2023) 'Data pruning and neural scaling laws: fundamental limitations of score-based algorithms', *Transactions on Machine Learning Research*. arXiv:2302.06960.

Bengio, Y., Louradour, J., Collobert, R. and Weston, J. (2009) 'Curriculum learning', in *Proceedings of the 26th International Conference on Machine Learning (ICML)*, pp. 41–48.

Cheng, H., Wen, Q., Liu, Y. and Sun, L. (2024) 'RobustTSF: Towards theory and design of robust time series forecasting with anomalies', in *International Conference on Learning Representations (ICLR)*. arXiv:2402.02032.

Friedman, M. (1937) 'The use of ranks to avoid the assumption of normality implicit in the analysis of variance', *Journal of the American Statistical Association*, 32(200), pp. 675–701.

Fu, Y., Shao, Z., Yu, C., Li, Y., An, Z., Wang, Q., Xu, Y. and Wang, F. (2025) 'Selective learning for deep time series forecasting', in *Advances in Neural Information Processing Systems (NeurIPS)*. arXiv:2510.25207.

Gal, Y. and Ghahramani, Z. (2016) 'Dropout as a Bayesian approximation: representing model uncertainty in deep learning', in *International Conference on Machine Learning (ICML)*. arXiv:1506.02142.

Han, B., Yao, Q., Yu, X., Niu, G., Xu, M., Hu, W., Tsang, I.W. and Sugiyama, M. (2018) 'Co-teaching: robust training of deep neural networks with extremely noisy labels', in *Advances in Neural Information Processing Systems (NeurIPS)*, vol. 31. arXiv:1804.06872.

Hounie, I., Porras-Valenzuela, J. and Ribeiro, A. (2024) 'Loss shaping constraints for long-term time series forecasting', in *International Conference on Machine Learning (ICML)*. arXiv:2402.09373.

Huber, P.J. (1964) 'Robust estimation of a location parameter', *The Annals of Mathematical Statistics*, 35(1), pp. 73–101.

Kang, D. and Hashimoto, T.B. (2020) 'Improved natural language generation via loss truncation', in *Proceedings of the 58th Annual Meeting of the Association for Computational Linguistics (ACL)*. doi:10.18653/v1/2020.acl-main.66.

Kumar, M.P., Packer, B. and Koller, D. (2010) 'Self-paced learning for latent variable models', in *Advances in Neural Information Processing Systems (NeurIPS)*.

Liu, T., Wan, M., Wang, J. and Nie, N. (2025) 'Enlightenment period improving DNN performance'. arXiv:2504.01737.

Liu, Y., Hu, T., Zhang, H., Wu, H., Wang, S., Ma, L. and Long, M. (2024) 'iTransformer: inverted transformers are effective for time series forecasting', in *International Conference on Learning Representations (ICLR)*. arXiv:2310.06625.

Nie, Y., Nguyen, N.H., Sinthong, P. and Kalagnanam, J. (2023) 'A time series is worth 64 words: long-term forecasting with transformers', in *International Conference on Learning Representations (ICLR)*.

Okanovic, P., Waleffe, R., Mageirakos, V., Nikolakakis, K.E., Karbasi, A., Kalogerias, D., Gürel, N.M. and Rekatsinas, T. (2024) 'Repeated random sampling for minimizing the time-to-accuracy of learning', in *International Conference on Learning Representations (ICLR)*. arXiv:2305.18424.

Paul, M., Ganguli, S. and Dziugaite, G.K. (2021) 'Deep learning on a data diet: finding important examples early in training', in *Advances in Neural Information Processing Systems (NeurIPS)*. arXiv:2107.07075.

Prayogo, N., Cevik, M. and Bodur, M. (2020) 'Time series sampling for probabilistic forecasting', in *ACM/SIGAPP Symposium on Applied Computing / CASCON*. doi:10.5555/3432601.3432621.

Qin, Z., Wang, K., Zheng, Z., Gu, J., Peng, X., Xu, Z., Zhou, D., Shang, L., Sun, B., Xie, X. and You, Y. (2024) 'InfoBatch: lossless training speed up by unbiased dynamic data pruning', in *International Conference on Learning Representations (ICLR)*. arXiv:2303.04947.

Rockafellar, R.T. and Uryasev, S. (2000) 'Optimization of conditional value-at-risk', *Journal of Risk*, 2(3), pp. 21–41.

Ren, M., Zeng, W., Yang, B. and Urtasun, R. (2018) 'Learning to reweight examples for robust deep learning', in *Proceedings of the 35th International Conference on Machine Learning (ICML)*.

Rousseeuw, P.J. and Leroy, A.M. (1987) *Robust Regression and Outlier Detection*. New York: Wiley.

Sorscher, B., Geirhos, R., Shekhar, S., Ganguli, S. and Morcos, A.S. (2022) 'Beyond neural scaling laws: beating power law scaling via data pruning', in *Advances in Neural Information Processing Systems (NeurIPS)*. arXiv:2206.14486.

Toneva, M., Sordoni, A., Tachet des Combes, R., Trischler, A., Bengio, Y. and Gordon, G.J. (2019) 'An empirical study of example forgetting during deep neural network learning', in *International Conference on Learning Representations (ICLR)*. arXiv:1812.05159.

Wang, S., Wu, H., Shi, X., Hu, T., Luo, H., Ma, L., Zhang, J.Y. and Zhou, J. (2024) 'TimeMixer: decomposable multiscale mixing for time series forecasting', in *International Conference on Learning Representations (ICLR)*.

Xia, X., Liu, J., Zhang, S., Wu, Q., Wei, H. and Liu, T. (2023) 'Moderate coreset: a universal method of data selection for real-world data-efficient deep learning', in *International Conference on Learning Representations (ICLR)*.

Zeng, A., Chen, M., Zhang, L. and Xu, Q. (2023) 'Are transformers effective for time series forecasting?', in *Proceedings of the AAAI Conference on Artificial Intelligence*.

Zhang, X., Du, J., Li, Y., Xie, W. and Zhou, J.T. (2024) 'Spanning training progress: temporal dual-depth scoring (TDDS) for enhanced dataset pruning', in *IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR)*. arXiv:2311.13613.

Zheng, H., Liu, R., Lai, F. and Prakash, A. (2023) 'Coverage-centric coreset selection for high pruning rates', in *International Conference on Learning Representations (ICLR)*.

**注**：以下论文（均出自本文的引用列表）我在本次审稿中无法核实其存在性、题目或发表信息，仅在审稿意见中提及其内容：Camporeale（2025，PARIS）、Chen 等（2026）、Li 等（2026，CLMFormer）、Pennino 与 Gabbrielli（2026）、Wu 等（2026）、Xu 等（2026，UNSEEN）、You 等（2024）、Wang 等（2025，Channel matters）、Liu 等（2026，TimeLAVA）、Zhu 等（2024）。作者应自行核对这些引用的准确性。
