# TMLR 审稿意见

**论文**：Prune the Supervision, Not the Windows: Decision Units for Data Pruning in Time-Series Forecasting（匿名投稿，14 页）
**审稿人**：独立审稿人（本审稿完全基于论文 PDF 本身，未参考任何其他审稿材料）

> 标注约定：下文中 **[事实]** 表示论文中可直接定位的内容（附页码/章节/表格/公式）；**[推断]** 表示由论文内容出发的合理推论；**[假设]** 表示需要作者补充实验才能确认或证伪的猜测。

---

## 1. 论文摘要（审稿人复述）

论文研究长序列时间序列预测（LTSF）中"数据剪枝"的**决策单元（decision unit）**问题。核心观察是 [事实, §1, §3.1]：滑动窗口训练（stride = 1）中，窗口 $i$ 的损失是 $T\times D$ 个平方残差项之和（Eq. 1），且同一目标值 $y_{t,d}$ 在 $c(t,d)=|\mathcal{W}(t)|\approx T$ 个窗口中反复被监督；因此从图像分类继承的"以样本（窗口）为剪枝单元"并不是最小的单元。作者固定打分函数（在全量训练 10% epoch 处、关闭 dropout、对所有训练窗口做一次前向，记录每个损失项的平方残差 $r_i(t,d)$，§3.3）与预算（$r=10\%$ 的损失项），只改变决策单元，比较五种单元（Eq. 2）：窗口（$T\times D$ 项）、时间列（$D$ 项）、通道行（$T$ 项）、单个 occurrence（1 项）、以及跨窗口聚合同一目标值的 element（$c(t,d)$ 项，得分见 Eq. 3）。"剪枝"实现为损失加权掩码（Eq. 4），不删除任何窗口或输入值。

实验 [事实, §4]：三个骨干（iTransformer、DLinear、TimeMixer）、七个数据集（ETTh1/h2/m1/m2、weather、solar、ECL）、四个预测长度、五个随机种子，使用 TSLib 官方配置；统计推断以数据集为单位（配对 t 检验、符号翻转、Friedman）。主要结论 [事实, 摘要/§5/§6]：(i) 掩码高残差 10% 在所有单元上都降低测试 MAE（窗口 −2.03% 到 occurrence −4.72%），掩码低残差 10% 从不带来收益；(ii) 窗口单元收益最小；(iii) 收益集中在误差分布主体（中位数 −4.42% 至 −12.45%），95 分位以后消失，CVaR@90% 无可测变化，MSE 平均改善 1.7%–2.8% 但不显著。§7 在 iTransformer 上进一步表明：早期残差得分与"序列其余部分无法解释的误差"的 Spearman 相关为 0.998；与形状匹配的随机掩码只有 6% 的增益；早期得分与 MC-dropout、TDDS 等更昂贵的得分效果接近；element 掩码在 MAE 上全部六个数据集优于 Selective Learning；掩码优于用插值"修复"；在 element 掩码之上以等间距删除 30% 窗口可节省优化步数而 MSE 变化 +0.30 点（不显著）。

---

## 2. 主要主张与证据充分性

| # | 主张（论文原话/概述） | 位置 | 证据 | 充分性评估 |
|---|---|---|---|---|
| C1 | 掩码最高残差 10% 的损失项在每个单元、每个数据集上降低测试 MAE（−2.03% 到 −4.72%） | 摘要；§5；表 1 | 表 1（7/7 datasets better，配对 t 检验 $p\le 0.022$）；表 4–6 按骨干拆分 | **基本充分**，但：无种子方差/CI；ETTh1 窗口 −0.11% 几乎为零仍计入"7/7"；MSE 不显著；未报告绝对基线数值；缺少**随机掩码**与**鲁棒损失**对照（见缺点 1、2） |
| C2 | 掩码最低残差 10% 从不带来收益 | 摘要；§5 | 图 1 灰柱；一侧 TOST 检验 $p\le 0.004$ | 充分，但该结论近乎必然（低损失项对梯度贡献本就很小），其作为"对照"的信息量有限（见缺点 2） |
| C3 | 窗口单元收益最小；通道行与 occurrence 在每个数据集上优于窗口 | §5；表 1、2 | Friedman $p=0.002$；平均秩 4.71/5 | **部分充分**。替代解释：窗口得分是 $T\times D$ 项的均值、相邻窗口得分高度自相关 → 窗口掩码等价于整段删除；细粒度单元的优势可能只是"掩掉的残差质量更集中"（见缺点 3） |
| C4 | 收益位于误差分布主体，尾部不变；MSE 因此动得少 | §6；表 2；图 2 | 分位数曲线、CVaR@90% 无显著变化 | 充分（描述层面），但解释层面有强替代解释：等价于从均值回归向中位数回归的偏移，L1/Huber 训练可能复现（见缺点 1） |
| C5 | 得分对目标的排序等于"序列其余部分无法解释的误差"（Spearman 0.998） | 摘要；§7.1 | 五折交叉掩码，在**同一早期训练点**读取 | **不充分/被高估**。在 10% epoch 处，看过 80% 监督与看过 100% 监督的模型预测几乎相同，相关 0.998 近乎同义反复（见缺点 5） |
| C6 | 早期一次性得分接近昂贵得分；element 掩码优于 Selective Learning | §7.2；表 3 | 表 3，六个数据集 | MAE 上充分；MSE 上 SL 差距 0.68/0.40 点不显著（$p=0.07/0.46$），摘要中只说 MAE 是审慎的。但 SL 被改造到 element 单元与 10% 预算，原方法的设置未复现（见缺点 7） |
| C7 | 掩码优于修复（插值） | §7.2 | 表 3 末行 | **不充分**：插值同时改变了输入与目标（作者自己在附录 B 承认），不是"仅对监督的修复"，比较混淆 |
| C8 | 等间距删窗 30% 可节省步数且不损失 | §7.3 | 文字叙述 | **证据不足**：无表格；"三条按得分选窗的规则"从未被定义；缺少"不加 element 掩码的等间距删窗"对照；自适应 Tukey 预算从未在主文报告其掩码比例与结果 |
| C9 | 推荐通道行为默认单元 | §6 末；§8 | 表 1、2 | 推荐依据是"MSE 最低但不显著"（Friedman $p=0.52$），与 occurrence 在 MAE 上更优的结论存在内部张力（见缺点 9） |

---

## 3. 优点

1. **问题设定新颖而重要** [事实, §1, §3]：指出滑动窗口训练中"窗口不是最小监督单元"、同一目标被监督 $\approx T$ 次，并由此系统地定义五种决策单元，这是对时间序列数据剪枝文献的一个清晰、有启发的概念贡献。element 单元（Eq. 3）确实是只有滑动窗口预测才有的决策。
2. **实验设计的控制变量意识强** [事实, §3.3–3.4]：得分、预算、训练配置全部固定，只改变决策单元；掩码不删窗口、不缩短 epoch，这样各单元之间确实"只在决策捆绑方式上不同"。
3. **统计推断相对谨慎** [事实, §4]：以数据集为推断单位、明确说明符号翻转检验的下界 1/64、TOST 式等价检验、Friedman 检验、不做多重性校正但报告数据集个数。这在 LTSF 文献中少见且值得肯定。
4. **规模可观** [事实, §4]：3 骨干 × 7 数据集 × 4 horizon × 5 种子 × (1 + 5 单元 × 2 端)，并附按骨干拆分的表 4–6 和图 3。
5. **对负面/弱结果的诚实报告** [事实, §5–§6]：明确指出 MSE 不显著、ECL 上细粒度单元反而提高 MSE、CVaR 无变化、element 比 occurrence 损失约四分之一收益。
6. **§7.2 的得分对比**（表 3）很有信息量：RobustTSF 的无模型残差几乎无效，说明掩码并不简单等于"去异常值（相对于趋势）"；MC-dropout/TDDS/平均损失只略好或不如一次早期得分。
7. 写作紧凑、术语一致（occurrence / element / high end / low end），表格数值与正文引用经我核对一致（如表 1 均值行、§5 中的 1.43/2.69 点、§7.2 中的 0.24/0.07/0.54/2.93/3.63 点等均可由表格复算）。

---

## 4. 缺点与修改建议（按重要性排序）

### 缺点 1（最关键）：核心现象存在强替代解释——"掩掉高残差项"≈ 固定掩码版的鲁棒损失 / 损失截断，论文缺少这一类最直接的基线

- **问题描述**：论文的主效应是"在 MSE 训练下去掉 10% 最大残差项 → 测试 MAE 与中位误差下降、尾部与 MSE 不变"。这与鲁棒回归中早已熟知的现象一致：减少大残差项的权重使模型从条件均值向条件中位数靠拢，于是 L1 型指标改善、L2 型指标不变或轻微变差（Huber, 1964；Rousseeuw, 1984 的最小截断二乘；Shen & Sanghavi, 2019 的迭代截断损失；Kang & Hashimoto, 2020 的 loss truncation——后者论文已引用但未比较）。
- **论文位置/证据** [事实]：§3.3 "All units rank by the same residuals"；§6 "The gain is largest for ordinary errors and fades towards the tail"、"Why MSE moves less"；表 2 CVaR@90% 行。§2 "Scores and their limits" 中承认"In our setting the opposite end pays"，但没有把它与鲁棒损失联系起来。
- **为何重要**：若简单地用 L1 损失、Huber 损失、或每个 batch 在线丢弃 top-10% 残差项（无需任何打分、无需额外 10% 训练）就能得到同量级的 MAE 增益，那么"决策单元"这一主轴的实际意义将大幅缩水，论文标题中的"data pruning"框架也需重新定位。TSLib 中切换损失函数是一行配置。
- **具体建议/补充实验**：
  1. 在相同协议下加入：(a) L1 损失训练；(b) Huber/SmoothL1 训练；(c) **在线损失截断**：每个 batch 内按当前残差丢弃最高 10% 的 occurrence（动态，零额外成本）；(d) 按 $r_i(t,d)$ 做软重加权（如 $w\propto 1/(r+\epsilon)$）而非硬掩码。报告 MAE、MSE、中位数、CVaR@90%。
  2. 若固定早期掩码仍优于在线截断，这反而会是更强的论点（说明"早期一次性决定"有信息优势），请把它作为主结果之一。
  3. 在 §2 增加一段"鲁棒回归/损失截断"相关工作，并明确本文与之的关系：本文的新内容是"固定掩码 + 跨窗口的决策单元"，而不是"去掉大残差有益"。

### 缺点 2：缺少随机掩码这一真正的零假设对照；"低端"对照信息量太弱

- **问题描述**：主表（表 1）的对照是全量训练与低端掩码。低端掩码去掉的是梯度几乎为零的项，"从不带来收益"几乎是必然的 [推断]。真正能区分"选对了项"与"任意减少 10% 监督的正则化效应"的对照是**同预算随机掩码**（对每个单元各做一次）。
- **论文位置** [事实]：§7.1 只在 iTransformer、六个数据集、element 单元上做了形状匹配随机掩码（MSE −0.16%），未报告其 MAE；表 1/表 2 没有任何随机对照。
- **为何重要**：若随机掩码在某些单元（尤其窗口）上也给出 −1% 左右的 MAE，则"窗口单元 −2.03%"中大半是非特异性效应。另外，随机掩码还能校准 ETTh1 窗口 −0.11% 这类接近零的"改善"。
- **建议**：为五个单元各加一列"random 10%"，纳入表 1/表 2 与 Friedman 检验；在 §7.1 同时报告随机掩码的 MAE。

### 缺点 3：五个单元之间的比较混淆了"决策捆绑"与"得分平滑度/掩码空间结构"，窗口单元劣势的机制未被分析

- **问题描述** [推断]：
  - 窗口得分是 $T\times D$ 项的均值，方差极小；在 stride = 1 下相邻窗口共享 $T-1$ 个目标时间步，得分高度自相关，于是 top-10% 窗口会聚成少数**连续时间段**——窗口掩码实际上是"删除整段序列"，这在非平稳序列上可能恰好删掉一个 regime。细粒度单元（occurrence/时间列）则可以只去掉这些段中真正异常的项。
  - occurrence 的残差随 horizon 位置系统增大（远端预测误差更大），因此 occurrence 掩码很可能**系统性地集中在远 horizon 位置**；时间列单元同理。这意味着 occurrence 的额外收益可能部分可由"无得分的 horizon 位置加权"复现——这正是论文引用的 Hounie et al. (2024) 的做法，但论文没有检验。
  - 通道行与 element 掩码很可能集中在少数高方差通道上；在 ECL（321 通道）上这可能使某些通道几乎失去全部监督，这与表 1 中 ECL 细粒度单元 MSE 上升（+0.05% 到 +1.75%）一致 [假设]。
- **论文位置** [事实]：§5 "Since the two ends use the same unit, the same score, and the same budget, the gain from masking the high end comes from which loss terms are masked"；§7.1 只控制了"stretch 长度与一天中的时刻"，未控制 horizon 位置与通道分布。
- **建议**：
  1. 对每个单元报告掩码的**结构诊断**：按 horizon 位置、通道、绝对时间（训练集前/中/后段）的直方图；窗口掩码的连通段长度分布。
  2. 增加无得分基线：(a) 掩掉最后 10% horizon 步；(b) 掩掉训练残差最高的 $\lceil 0.1D\rceil$ 个通道；(c) 带每通道/每 horizon 位置预算上限的 occurrence 掩码。
  3. 增加"按去除的残差质量匹配"而非"按去除的项数匹配"的比较，以区分"单元"与"去除残差集中度"两个因素。
  4. 讨论 ECL 异常并补充 Traffic（862 通道）以确认是否为高维数据集的共性。

### 缺点 4：只考察了单一预算（10%）与单一打分时刻（10% epoch），结论的稳健范围未知

- **论文位置** [事实]：§3.3 "The main comparison uses $r=10\%$"；自适应 Tukey 预算只在 §7.3 使用且从未报告其实际掩码比例与性能；§7.3 的窗口删除率 10%–90% 只针对窗口。
- **为何重要**：数据剪枝文献的核心教训之一是方法排序随剪枝率翻转（Sorscher et al., 2022；Zheng et al., 2023 的 CCS；Ayed & Hayou, 2023——后两者思想在本文中未被实验覆盖）。单元之间的排序在 1%、5%、20%、30%、50% 时可能完全不同（例如 element 在高预算下可能因整体删掉目标而迅速恶化）。
- **建议**：至少在 iTransformer 上做预算扫描（1/2/5/10/20/30/50%）与打分时刻扫描（1 epoch / 10% / 30% / 100% / 最优验证 epoch），画出每个单元的曲线；主文报告 Tukey 自适应预算的掩码比例与结果。

### 缺点 5：§7.1 "得分 = 未解释误差"（Spearman 0.998）的论证近乎同义反复，摘要中的表述被高估

- **论文位置** [事实]：§7.1 "train one run per fold with that fold's supervision masked, read at the same early point of training as the score"；摘要 "the score ranks targets by the error that the rest of the series leaves unexplained"。
- **问题** [推断]：在训练的 10% 处，用 80% 监督与用 100% 监督训练的模型几乎相同（每个 element 作为输入仍出现在 $L$ 个窗口中，且窗口损失平均了 $T\times D$ 项，单个目标在 1 个 epoch 内几乎不会被"记住"）。因此"折外早期残差"与"折内早期残差"的秩相关接近 1 是必然的，并不能说明得分衡量的是"序列其余部分**无法**解释"的误差——它只说明"在训练早期尚未被解释"。作者自己也写道 "neither says that a target is unpredictable in principle"，但摘要与 §8 的措辞远强于此。
- **建议**：(1) 用**训练至收敛**的折外模型计算 unexplained error，重新报告相关性；(2) 报告得分与简单统计量（|y − 通道滚动均值|、|Δy|、RobustTSF 残差）的相关性，以说明得分并非只是幅值代理；(3) 把摘要/结论中的表述改为"与训练早期折外残差高度一致"。

### 缺点 6：统计推断的几个隐藏假设与遗漏

- **数据集独立性被高估** [事实+推断]：§4 "we therefore test every claim with the dataset as the unit"——但 ETTh1 与 ETTm1 是同一台变压器（15 分钟与小时采样）的同一原始序列，ETTh2/ETTm2 同理（Zhou et al., 2021）。因此 7 个"独立"数据集实际上约为 5 个独立来源，配对 t 检验的自由度被高估。建议：报告把 ETTh/ETTm 合并后的检验结果，或至少在 §4 指出这一点。
- **无种子方差** [事实]：全文没有任何标准差、CI 或种子级分布；表 1 中 ETTh1 窗口 −0.11%、ECL 多个格 < 1% 的变化很可能在种子噪声内。建议每格附标准误，或至少给出各数据集全量基线的种子变异系数。
- **多重检验** [事实]：§4 承认 p 值未校正；§5–§7 报告了几十个 p 值，其中不少在 0.02–0.05 之间（如 MC-dropout $p=0.045$、SL $p=0.040$、插值 MAE $p=0.058$）。建议至少对每节内的检验做 Holm (1979) 校正，并把 Demšar (2006) 推荐的 Friedman + 事后检验流程完整使用（目前只有 Friedman 没有事后配对比较）。
- **被剔除的发散运行** [事实]：§4 "Runs that diverge are excluded"，没有数量、没有分布（发散是否在某个单元/骨干更常见）。这可能产生选择偏差。请给出计数表并说明处理方式（例如记为最差值）。
- **对 t 检验的重尾敏感**：DLinear 在 ETTh2/ETTm2 上的 −10% 到 −18% MSE（表 5）是明显的离群效应，会主导均值；建议并列报告 Wilcoxon 符号秩检验与中位数效应。

### 缺点 7：与 Selective Learning 的比较公平性存疑，原方法的设置未被复现

- **论文位置** [事实]：附录 B "Every alternative masks elements exactly as the element mask does, at the same 10% budget"；"Its form scored once ... with its nominal rates calibrated until the realised share of masked elements matches the 10% budget"。
- **问题** [推断]：SL（Fu et al., 2025，按本文描述）以 timestep 为掩码单位并每 epoch 重打分，其掩码率是方法的一部分。本文把它投影到 element 单元并把掩码率标定到 10%，这已是作者改造后的变体；表 3 没有原始设置（原掩码率、原单元）下的 SL 结果，也没有说明是否复现了 SL 论文报告的数值。此外 SL 的 MSE 差距（0.68/0.40 点）不显著，而摘要只强调 MAE。
- **建议**：增加 SL 在其官方默认设置下的结果（并报告其实际掩码比例）；在摘要中同时说明 MSE 上差异不显著。

### 缺点 8：§7.3 的证据不完整，且与标题叙事存在张力

- **论文位置** [事实]：§7.3 全部为文字；"None of the three rules that select windows by their score does better than random pruning" ——这三条规则在全文任何地方都未定义；没有"仅等间距删窗、不加 element 掩码"的对照；自适应 Tukey 预算的掩码比例未知。
- **为何重要**：等间距删窗等价于把 stride 从 1 改为 ~1.4，是业内常用且无需得分的技巧，论文将其作为贡献的一部分需要完整证据；而"Prune the Supervision, Not the Windows"的标题与"最终还是要删窗口来省算力"的实际建议之间需要更清楚地调和。
- **建议**：补一张表（删除率 × {等间距, 随机, 三条得分规则} × {有/无 element 掩码}），定义三条规则，报告 Tukey 预算的实际比例。

### 缺点 9：默认单元的推荐依据与论文自身的统计结论冲突

- **论文位置** [事实]：§6 "As a default we recommend the channel row: ... has the lowest MSE of the five units, though not significantly so"；表 2 MSE 行 Friedman $p=0.525$；而 occurrence 在 MAE/中位数上显著最优（Friedman $p=0.002$，7/7）。
- **问题**：推荐建立在作者自己说"无法区分"的差异上。若读者关心 MSE（LTSF 的标准头号指标），论文的诚实结论应是"任何高端单元在 MSE 上的改善均不显著"。
- **建议**：把推荐改写为条件式（关心 MAE/典型误差 → occurrence；担心尾部/实现简单 → 通道行），并明确 MSE 上无显著证据支持任何选择。

### 缺点 10：可复现性信息不足

- [事实] 全文无代码链接、无 TSLib 版本/commit、无随机种子列表、无超参表（仅"official per-dataset recipes"）、无任何**绝对** MSE/MAE 基线值（读者无法核对全量运行是否达到各骨干论文报告的水平）；solar 在 TSLib 中并非所有骨干都有官方脚本（尤其 DLinear），需说明；TimeMixer 的"官方配置"是 TimeMixer 仓库还是 TSLib 仓库也需说明。
- [推断] occurrence 单元需要存储 $n\times T\times D$ 的残差张量（ECL, $T=720$ 时约 $4\times10^9$ 个浮点数，float32 约 16 GB），论文未讨论存储/实现成本。
- **建议**：发布代码与掩码文件；附录增加绝对指标表、超参表、每个单元的额外存储与时间开销。

### 缺点 11：验证集读取与早停的交互未说明

- [事实] §4 "Every run is read at its epoch of lowest validation loss"。未说明验证损失是全损失还是掩码损失、是否使用 TSLib 默认 patience=3 的早停、"10% of the full number of epochs"在早停下指的是什么（名义 epoch 数的 10%，即 TSLib 默认 10 epoch 下的第 1 个 epoch？）。
- **建议**：明确说明；并报告掩码运行与全量运行的最佳 epoch 分布（掩码是否只是改变了早停时机？）。

### 缺点 12："修复 vs 掩码"的比较被输入污染混淆

- [事实] 附录 B "because inputs and targets are cut from the same series, an interpolated value enters both"。
- **建议**：增加"只替换目标、输入保持原值"的修复变体；否则 C7 只能说明"修改训练序列本身有害"。

### 缺点 13：对非平稳性/分布漂移的讨论缺失

- [推断] 早期得分在非平稳序列上可能系统性地标记训练集末段（最接近测试分布的部分）或 regime 转换处，这恰是对测试最有价值的监督。论文没有报告掩码在时间轴上的分布，也没有在漂移明显的数据集（Exchange、Traffic）上测试，所用数据集均经 TSLib 的全局标准化。
- **建议**：报告掩码随绝对时间的分布；加入 Exchange（近随机游走，是"不可解释误差"的天然压力测试）与 Traffic；讨论与 RevIN（Kim et al., 2022）/非平稳 Transformer（Liu et al., 2022）等归一化方法的交互。

---

## 5. 领域层面的分析

### 5.1 研究脉络
- **图像分类的样本级剪枝**：EL2N/GraNd（Paul et al., 2021）、forgetting（Toneva et al., 2019）、动态不确定性（He et al., 2024）、TDDS（Zhang et al., 2024）、InfoBatch（Qin et al., 2024）、CCS（Zheng et al., 2023）、Moderate（Xia et al., 2023）。共同前提：一个样本 = 一个损失项。主要教训：保留难样本只在数据充足时有效（Sorscher et al., 2022），纯难度得分在高剪枝率下劣于随机（Ayed & Hayou, 2023；Zheng et al., 2023），因此覆盖度/多样性约束很重要——本文完全没有覆盖度维度，而窗口单元的"整段删除"问题正是覆盖度问题的一个时间序列版本。
- **噪声标签 / 鲁棒损失**：Co-teaching（Han et al., 2018）、MentorNet（Jiang et al., 2018）、loss truncation（Kang & Hashimoto, 2020）、迭代截断损失（Shen & Sanghavi, 2019）、经典鲁棒回归（Huber, 1964；Rousseeuw, 1984）。本文"去高端有益"属于这一脉络，而非数据剪枝脉络；论文应明确承认。
- **异方差/不确定性视角**：若高残差目标是高 aleatoric 噪声目标，则最优策略是逆方差加权而非硬掩码（Nix & Weigend, 1994；Kendall & Gal, 2017）。本文的 MC-dropout 得分（表 3 最优）已经暗示不确定性信息有用——软加权是自然的下一步。
- **时间序列特有的监督选择**：RobustTSF（Cheng et al., 2024）、Selective Learning（Fu et al., 2025）、horizon 损失整形（Hounie et al., 2024）、通道影响（Wang et al., 2025）、数据估值（TimeLAVA, Liu et al., 2026；Data Shapley, Ghorbani & Zou, 2019）。本文的贡献是把这些"沿不同轴"的做法统一在"决策单元"框架下，这是真正有价值的概念贡献。
- **基础模型时代**：Chronos（Ansari et al., 2024）等预训练模型的训练语料也由滑动窗口构成，"同一目标被监督 $T$ 次"的问题在那里更严重；本文结论能否迁移到预训练/微调场景是显而易见的后续问题，值得在讨论中提出。

### 5.2 隐藏联系与悖论
1. **分类 vs 回归的悖论**：分类中去掉易样本有益，本文中去掉难项有益。统一解释：在分类中"难"= 决策边界附近的信息量；在平方损失回归中"难"= 大残差 = 大梯度，且在 MSE 下大残差往往由噪声/异常主导。这其实是"损失函数决定了哪一端值得剪"的问题，而非时间序列的特性；建议论文明确这一点。
2. **MAE 改善而 MSE 不变的悖论**可由"均值→中位数"偏移完全解释（缺点 1），无需援引决策单元。
3. **窗口重叠 ⇒ 得分自相关 ⇒ 窗口剪枝 = 段删除**（缺点 3）是本文最有趣却未被挖掘的时间序列特有机制，它把"覆盖度"概念自然地引入时间序列剪枝。
4. **element 比 occurrence 差**本身是一个有意思的发现：它意味着"一个目标是否值得学"不是目标的内在属性，而取决于预测它的上下文（窗口）——这与论文摘要"按目标排序"的叙事略有冲突，值得正面讨论。

### 5.3 研究空白
- 预算与打分时刻的敏感性；覆盖度约束；软加权；与鲁棒损失的边界；非平稳/漂移数据；高维数据集（ECL 异常）；预训练基础模型；概率预测（本文仅点预测）；不同 lookback $L$。

### 5.4 对比表（审稿人整理）

| 方法/脉络 | 决策单元 | 得分来源 | 是否减少算力 | 是否处理跨窗口重复监督 |
|---|---|---|---|---|
| EL2N/GraNd/forgetting/TDDS | 样本 | 训练动态 | 是 | 否 |
| InfoBatch | 样本（动态） | 当前损失 | 是 | 否 |
| Loss truncation / 截断损失 | 单个损失项（动态） | 当前损失 | 否 | 否 |
| RobustTSF | 时间点 | 无模型趋势残差 | 否 | 部分 |
| Selective Learning | 目标 timestep | 残差熵 + 异常模型 | 否 | 部分 |
| Hounie et al. 2024 | horizon 位置 | 约束优化 | 否 | 否 |
| **本文** | 五种（含 element） | 早期残差，固定 | 否（§7.3 除外） | 是（element） |

---

## 6. 给作者的问题

1. 在相同协议下，L1/Huber 损失训练与在线 top-10% 损失截断分别能达到多少 MAE/MSE 改善？（缺点 1）
2. 各单元的同预算随机掩码结果（MAE 与 MSE）是多少？（缺点 2）
3. occurrence/时间列掩码在 horizon 位置上的分布如何？通道行/element 掩码在通道上的分布如何？窗口掩码的连通段长度分布如何？（缺点 3）
4. 预算从 1% 到 50% 时五个单元的排序是否保持？打分时刻改变时呢？（缺点 4）
5. 用收敛后的折外模型计算 unexplained error，Spearman 相关还是 0.998 吗？"nine combinations"（§7.1）指哪九个？（缺点 5）
6. 有多少运行因发散被剔除？分布在哪些骨干/单元？（缺点 6）
7. 验证损失是全损失还是掩码损失？是否使用早停？"10% of the full number of epochs"在早停下如何定义？（缺点 11）
8. Selective Learning 在其官方设置下的结果与实际掩码率是多少？（缺点 7）
9. §7.3 中"三条按得分选窗的规则"是什么？Tukey 自适应预算实际掩掉了多少比例？不加 element 掩码的等间距删窗结果如何？（缺点 8）
10. 各数据集全量基线的绝对 MSE/MAE 是多少？与各骨干原论文的报告值是否一致？（缺点 10）
11. 是否将 ETTh/ETTm 视作独立来源？若合并，p 值如何变化？（缺点 6）
12. occurrence 单元在 ECL、$T=720$ 时残差张量的存储与计算成本是多少？

---

## 7. 次要问题（排版、笔误、符号）

- §3.1：$c(t,d)=|\mathcal{W}(t)|$ 与 $d$ 无关，记作 $c(t)$ 更一致；$n$ 的定义（$n=N-L-T+1$）未给出。
- Eq. 4 的归一化 $\sum w$ 是按 batch 还是全局？按 batch 时掩码比例高的 batch 会被相对放大，请说明。
- §4 "exact sign-flip test cannot fall below 1/64" 为双侧值，请注明单/双侧。
- 图 1 顶部示意图在打印尺寸下很难辨认（尤其 occurrence 与 element 的区别），建议放大或拆分。
- 表 2 与表 7 的 Friedman p 在正文中被四舍五入（0.525→0.52），无妨，但建议统一位数。
- §7.1 "nine combinations" 未定义；§7.3 "three rules" 未定义。
- 表 3 标题说 "Bold: the best row of each column" 但正文将"ours"作为参照，建议同时标出"ours"行。
- 摘要中"the dataset as the unit of inference"对不熟悉的读者不易理解，建议改写。
- 参考文献中多条为 arXiv 预印本（如 Camporeale 2025、Chen et al. 2026、Wu et al. 2026），请核对正式出版信息；"Lin et al., 2024"（Rho-1）在正文 §2 中描述准确。
- "RobustTSF residual, which needs no model"——它需要趋势过滤器，建议改为"needs no trained model"。
- 标题与 §7.3 的"删窗口"建议之间的张力，建议在 §8 一句话调和。

---

## 8. TMLR 标准评估与推荐

**(a) 主张是否有准确、令人信服、清晰的证据支持？** 部分满足。描述性主张（C1–C4 的"发生了什么"）证据充分、统计处理审慎。但解释性主张（为什么、以及"决策单元"是否是关键因素）面临未被排除的强替代解释：鲁棒损失/在线截断（缺点 1）、非特异性正则化（缺点 2）、掩码的空间结构与 horizon/通道集中（缺点 3）；C5 的论证近乎同义反复；C8 证据不完整；推断的独立性假设（ETTh/ETTm）与种子方差缺失削弱了小样本检验的可信度。

**(b) 是否有 TMLR 读者感兴趣的内容？** 是。"窗口不是最小监督单元"的视角、element 单元的定义、以及"收益在分布主体而非尾部"的分位数分析，对时间序列与数据剪枝两个社区都有价值。

**总体推荐：Major revision（主要修改）**。
若作者补充：(i) L1/Huber/在线截断基线；(ii) 同预算随机掩码；(iii) 掩码结构诊断与 horizon/通道无得分基线；(iv) 预算与打分时刻扫描；(v) 种子方差与绝对基线；(vi) §7.3 完整表格——并相应地收敛摘要/结论中的因果措辞，我认为论文可以达到 TMLR 的接收标准，即便补充实验表明收益主要来自鲁棒损失效应（那本身也是对社区有价值的澄清）。

**置信度**：4/5。我仔细阅读了全文并核对了表格与正文数值的一致性；对 TSLib 实现细节与 Selective Learning 原始设置的判断基于论文描述与我对该领域的了解，未运行代码。

---

## 参考文献（Harvard 格式）

- Ansari, A.F. et al. (2024) 'Chronos: Learning the language of time series', *Transactions on Machine Learning Research*.
- Arpit, D. et al. (2017) 'A closer look at memorization in deep networks', *Proceedings of ICML*.
- Ayed, F. and Hayou, S. (2023) 'Data pruning and neural scaling laws: fundamental limitations of score-based algorithms', *Transactions on Machine Learning Research*.
- Benjamini, Y. and Hochberg, Y. (1995) 'Controlling the false discovery rate: a practical and powerful approach to multiple testing', *Journal of the Royal Statistical Society: Series B*, 57(1), pp. 289–300.
- Cheng, H., Wen, Q., Liu, Y. and Sun, L. (2024) 'RobustTSF: Towards theory and design of robust time series forecasting with anomalies', *Proceedings of ICLR*.
- Demšar, J. (2006) 'Statistical comparisons of classifiers over multiple data sets', *Journal of Machine Learning Research*, 7, pp. 1–30.
- Fu, Y. et al. (2025) 'Selective learning for deep time series forecasting', *Advances in Neural Information Processing Systems* (as cited in the submission, arXiv:2510.25207).
- Ghorbani, A. and Zou, J. (2019) 'Data Shapley: Equitable valuation of data for machine learning', *Proceedings of ICML*.
- Guo, C., Zhao, B. and Bai, Y. (2022) 'DeepCore: A comprehensive library for coreset selection in deep learning', *Proceedings of DEXA*.
- Han, B. et al. (2018) 'Co-teaching: Robust training of deep neural networks with extremely noisy labels', *Advances in Neural Information Processing Systems*, 31.
- He, M., Yang, S., Huang, T. and Zhao, B. (2024) 'Large-scale dataset pruning with dynamic uncertainty', *CVPR Workshops*.
- Holm, S. (1979) 'A simple sequentially rejective multiple test procedure', *Scandinavian Journal of Statistics*, 6(2), pp. 65–70.
- Hounie, I., Porras-Valenzuela, J. and Ribeiro, A. (2024) 'Loss shaping constraints for long-term time series forecasting', *Proceedings of ICML*.
- Huber, P.J. (1964) 'Robust estimation of a location parameter', *Annals of Mathematical Statistics*, 35(1), pp. 73–101.
- Jiang, L., Zhou, Z., Leung, T., Li, L.-J. and Fei-Fei, L. (2018) 'MentorNet: Learning data-driven curriculum for very deep neural networks on corrupted labels', *Proceedings of ICML*.
- Kang, D. and Hashimoto, T. (2020) 'Improved natural language generation via loss truncation', *Proceedings of ACL*.
- Katharopoulos, A. and Fleuret, F. (2018) 'Not all samples are created equal: Deep learning with importance sampling', *Proceedings of ICML*.
- Kendall, A. and Gal, Y. (2017) 'What uncertainties do we need in Bayesian deep learning for computer vision?', *Advances in Neural Information Processing Systems*, 30.
- Kim, T. et al. (2022) 'Reversible instance normalization for accurate time-series forecasting against distribution shift', *Proceedings of ICLR*.
- Liu, Y. et al. (2022) 'Non-stationary Transformers: Exploring the stationarity in time series forecasting', *Advances in Neural Information Processing Systems*, 35.
- Liu, Y. et al. (2024) 'iTransformer: Inverted Transformers are effective for time series forecasting', *Proceedings of ICLR*.
- Nie, Y., Nguyen, N.H., Sinthong, P. and Kalagnanam, J. (2023) 'A time series is worth 64 words: Long-term forecasting with Transformers', *Proceedings of ICLR*.
- Nix, D.A. and Weigend, A.S. (1994) 'Estimating the mean and variance of the target probability distribution', *Proceedings of the IEEE International Conference on Neural Networks*.
- Paul, M., Ganguli, S. and Dziugaite, G.K. (2021) 'Deep learning on a data diet: Finding important examples early in training', *Advances in Neural Information Processing Systems*, 34.
- Qin, Z. et al. (2024) 'InfoBatch: Lossless training speed up by unbiased dynamic data pruning', *Proceedings of ICLR*.
- Rousseeuw, P.J. (1984) 'Least median of squares regression', *Journal of the American Statistical Association*, 79(388), pp. 871–880.
- Shen, Y. and Sanghavi, S. (2019) 'Learning with bad training data via iterative trimmed loss minimization', *Proceedings of ICML*.
- Sorscher, B., Geirhos, R., Shekhar, S., Ganguli, S. and Morcos, A.S. (2022) 'Beyond neural scaling laws: Beating power law scaling via data pruning', *Advances in Neural Information Processing Systems*, 35.
- Toneva, M. et al. (2019) 'An empirical study of example forgetting during deep neural network learning', *Proceedings of ICLR*.
- Wang, S. et al. (2024) 'TimeMixer: Decomposable multiscale mixing for time series forecasting', *Proceedings of ICLR*.
- Wu, H. et al. (2023) 'TimesNet: Temporal 2D-variation modeling for general time series analysis', *Proceedings of ICLR* (Time-Series Library).
- Xia, X. et al. (2023) 'Moderate coreset: A universal method of data selection for real-world data-efficient deep learning', *Proceedings of ICLR*.
- Zeng, A., Chen, M., Zhang, L. and Xu, Q. (2023) 'Are Transformers effective for time series forecasting?', *Proceedings of AAAI*.
- Zhang, X., Du, J., Li, Y., Xie, W. and Zhou, J.T. (2024) 'Spanning training progress: Temporal dual-depth scoring (TDDS) for enhanced dataset pruning', *Proceedings of CVPR*.
- Zheng, H., Liu, R., Lai, F. and Prakash, A. (2023) 'Coverage-centric coreset selection for high pruning rates', *Proceedings of ICLR*.
- Zhou, H. et al. (2021) 'Informer: Beyond efficient Transformer for long sequence time-series forecasting', *Proceedings of AAAI*.
