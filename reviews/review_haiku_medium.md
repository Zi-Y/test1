# 独立审稿意见

**论文**：Prune the Supervision, Not the Windows: Decision Units for Data Pruning in Time-Series Forecasting（TMLR 匿名投稿，14 页）
**审稿人**：独立审稿人（Haiku 5.5, medium）
**阅读范围**：全文 14 页逐页阅读（第 1–5、6–10、11–14 页），包括所有表格、图、公式、附录与参考文献。页码均指 PDF 页码。
**独立性声明**：本审稿仅基于论文 PDF 本身完成，未访问 `/home/user/test1/reviews/` 下除本文件以外的任何内容。

---

## 0. 结论速览

论文的核心对照设计（打分函数与预算固定，只改变"决策单元"）是清晰且有价值的，在 iTransformer / DLinear / TimeMixer 上对 7 个数据集的高残差端掩码提升 MAE 的结论在统计上较稳。但论文目前存在若干实质性问题：（1）比较没有计算量匹配，且"无需第二次训练"的表述与方法流程不符；（2）随机掩码与稳健损失基线只在局部（iTransformer、element、MSE、6 个数据集）出现，无法证明"打分"本身的贡献；（3）horizon 位置与 occurrence 多重性的混杂没有排除；（4）"高残差 = 未被序列其余部分解释的误差"这一解释缺少替代解释的检验；（5）存在一处与表格直接矛盾的断言；（6）多个"从不有效""尾部不变""无法区分"的零结果以不显著作为依据，表述过强；（7）只有一个预算（10%）、一个打分时点（10% 训练进度）、一个回看长度（L=96），敏感性缺失。

**推荐**：Major revision（大修）。**置信度**：4/5。

---

## 1. 论文摘要（用自己的话）

作者指出，滑动窗口预测中一个数据窗口包含 T×D 个损失项，同一个目标值 (t,d) 会在 c(t,d)=|W(t)| 个窗口中各被监督一次。既有的预测数据剪枝方法（大多来自分类）以"窗口"为剪枝单位。作者将剪枝单位在五个层级之间比较：整窗口（T×D 项）、时间列 (i,t)（D 项）、通道行 (i,d)（T 项）、单个 occurrence（1 项）、以及跨窗口的 element (t,d)（同一目标的全部 occurrence）。打分统一为：在完整训练 10% 进度时读取一次训练集上每个 occurrence 的平方残差，每个单元的分数为其成员残差的均值；预算统一为每个单元屏蔽其成员中得分最高（或最低）的 10%（式 2–4）。实验在 ETTh1/ETTh2/ETTm1/ETTm2/weather/solar/ECL 七个数据集、L=96、四个预测长度、五个随机种子上进行，并以"相对于同种子全数据训练的百分比变化"报告结果，以数据集为推断单位。主要发现：（a）屏蔽高残差端在所有单元上都降低 MAE，幅度从 window 的 2.03% 到 occurrence 的 4.72%；屏蔽低残差端几乎无益；（b）增益主要来自误差分布的中位数附近，而非尾部；MSE 改善较小且不显著；（c）在 iTransformer 上，element 掩码的收益与"未被序列其余部分解释的误差"相关，单次早期打分接近代价高得多的分数，并优于 Selective Learning 的 MAE 结果；（d）在窗口级剪枝上，基于分数的规则不优于随机剪枝。

---

## 2. 主要主张与证据评估

| # | 主张 | 位置 | 证据 | 评估 |
|---|---|---|---|---|
| C1 | 屏蔽高残差 10% 在所有单元上降低 MAE，且 7/7 数据集一致（window −2.03% 至 occurrence −4.72%） | 摘要；p.2；Table 1（p.6）；Fig. 1 | 表 1 的 MAE 均值可复算（已核对 window、time col、element、occ 的均值）；每单元 7/7 为负；p≤0.022 | **基本支持**，但仅在单一预算、单一打分时点、无随机对照（跨单元）条件下成立 |
| C2 | 屏蔽低残差端"从不有益" | 摘要；p.5 第 5 节；Fig. 1（p.2） | 低端 MAE 均值为 +0.09% 至 +0.49%；"28/30 个单 backbone 案例增益 <0.3"；但低端数值除 Fig. 1 外无表格 | **部分支持，表述过强**：论文自己的例外（TimeMixer MSE 中 chan row −0.06%、element −0.19%）说明并非"从不"；应改为"未显示超过 0.3 点的增益" |
| C3 | window 增益最小，occurrence 增益最大，排序不是偶然 | p.5 | 平均秩 4.71 与 1.29 已核对；Friedman p=0.002 | **支持**（仅限 MAE 和 10% 预算） |
| C4 | 增益位于误差分布主体，随分位数增大而消失 | p.6–7；Table 2；Fig. 2、3 | 中位数与 MAE 的 Friedman p=0.002；q90 的 p=0.066 | **方向支持**；但"尾部不变"的结论基于不显著性（见 W5） |
| C5 | MSE 改善较小是因为其对大误差加权，尾部增益消失 | p.7 "Why MSE moves less" | RMSE 变化与 MSE 变化在数学上相互对应（RMSE 变化约为 MSE 变化的一半）；论证合理但未直接检验 | **合理但未验证**（见 W16） |
| C6 | "occurrence 在每个数据集、每个 backbone 上都优于 element" | p.6 "Deciding per target" | **Table 4（p.13）与此矛盾**：iTransformer 上 solar（occ −5.64 vs element −7.57）与 ECL（occ −1.86 vs element −2.01）中 element 更优 | **被表格反驳**（见 W11） |
| C7 | 默认推荐通道行（channel row） | p.7 "Which unit to use" | MAE 均值 −3.46 在五个单元中排第三（occurrence −4.72，element −3.81）；MSE 最低但 Friedman p=0.52 | **证据不足**（见 W8） |
| C8 | 分数 ≈ "其余序列未解释的误差"；Spearman 0.998，与"已解释误差"仅 0.22 | p.8 第 7.1 节 | 仅在 iTransformer、5 个数据集（不含 solar）、20 个组合上检验；与替代解释未比较 | **部分支持**：统计上一致，但"未解释"的含义存在循环性与替代解释（见 W4） |
| C9 | 随机掩码（同长度、同时段）几乎无增益，element 掩码显著优于随机 | p.8 第 7.1 节 | 仅 iTransformer、6 个数据集、MSE 报告；MAE 未报告 | **部分支持**：控制实验范围太窄（见 W2） |
| C10 | "屏蔽优于修复"（插值替代） | p.8–9；Table 3 | 插值同时改变输入与标签（附录 B，p.12） | **混淆**，不是纯损失层面的比较（见 W12） |
| C11 | 单次早期打分接近代价高得多的分数，并优于 Selective Learning（MAE） | p.8–9；Table 3 | 仅 iTransformer；MC-dropout 在 MSE 上显著优于本文分数（p=0.045） | **基本支持**，但仅在一个模型、一个打分时点、一个预算上检验 |
| C12 | 打分"无需第二次训练"、成本可忽略（10%） | p.4 第 3.3 节；Table 3 "cost" 列 | 方法包含 10% 的部分完整训练 + 从零开始的完整掩码训练 | **与流程不符**（见 W1） |
| C13 | 基于分数的窗口剪枝不优于随机剪枝（10%–90%）；等间距窗口在 50%、70% 更优 | p.9 第 7.3 节 | 单点比较，p 值未做多重校正 | **支持**，但这是一个对论文"剪枝"框架不利的负结果（见 W9） |
| C14 | 结论在三个 backbone、七个数据集上成立 | 全文 | 三个 backbone，其中 DLinear 为线性、TimeMixer 为多尺度混合，设计上差异较大 | **在所测范围内支持**；推广到其他回看长度、预算、模型族尚未证明 |

---

## 3. 优点

1. **控制变量的设计合理**：打分函数与预算固定、只改变决策单元（式 2），使"单元"的比较有较清楚的因果含义。这一点是本文最有价值的贡献，也是 TMLR 读者会感兴趣的部分。
2. **配对的相对变化度量**：每个结果以同种子的全数据运行为基准（第 4 节），并以数据集为推断单位（而非 run 或 dataset-horizon 组合），且明确说明了 horizon 与 backbone 的非独立性。这一推断单位的选择是合理且诚实的。
3. **多个分位数与多种度量并报**：从中位数到 CVaR@90 与 MSE，表 2 与图 2 的"增益在分布主体"论证比单纯报告均值更有说服力。
4. **在 Section 7 中做了较深入的诊断**：留出（hold-out）检验、随机掩码匹配、与 Selective Learning 和 MC-dropout 的对照，且作者主动报告了对自身方法不利的结果（MC-dropout 在 MSE 上更优；随机窗口剪枝不劣于分数规则）。
5. **附录给出了较完整的训练与打分细节**（附录 A、B），包括分数的并列处理方式。
6. **写作清晰**，式（1）–（4）的符号基本一致（已核对 W(t)、c(t,d)、T_i 的定义），图 1 对五个单元的示意直观。

---

## 4. 缺点与修改建议（按重要性排序）

### W1（最高优先级）：比较未做计算量匹配；"无需第二次训练"的表述与流程不符

**位置**：p.4 第 3.3 节（"The score needs no auxiliary model, no ensemble, and no second training run"）；p.5 第 4 节（"every masked run trains from scratch"）；Table 3（p.8）"cost of the score" 列；p.9 第 7.3 节（"saving 30% of the optimiser steps"）。

**问题**：
- 打分需要在完整训练的 10% 进度处停止并读取残差（p.4）。随后，掩码运行从零开始，以相同 epoch 数完整训练（p.5、附录 A p.11）。因此方法的总训练量约为基线的 **110%**，并且掩码不缩短任何 epoch（作者在 p.9 也承认 "a mask does not shorten an epoch"）。
- "no second training run" 的说法不准确：打分本身是对同一模型的一次 10% 前缀训练，之后被丢弃，掩码运行又是完整的第二次训练。
- 因此，相对于"全数据、同 epoch 数"的基线，本方法多消耗约 10% 的计算。论文的增益（MAE 2–5%）中，有多少来自额外的优化步数，没有被排除。

**为何重要**：TMLR 读者关心的是在相同计算预算下的表现。当前的基线不公平地对方法有利（它比方法少用 10% 的训练），而 Table 3 中的"cost"列只在打分之间比较，没有与基线比较。

**建议**：
1. 增加"计算量匹配"的基线：(a) 全数据运行，epoch 数增加 10%（或与掩码运行相同的总优化步数）；(b) 随机 10% 掩码（同预算、同单元），两者都以同种子报告。
2. 报告实际的墙钟时间与优化步数，而不是仅报告"10%"。
3. 修正 p.4 "no second training run" 的表述，明确写出"10% 前缀 + 完整掩码运行"的总成本。
4. 在标题与摘要中，避免"剪枝"的效率含义，除非能证明在等计算量下仍有优势。

---

### W2：缺少关键控制——随机掩码与稳健损失基线未在全部单元、两个主要指标上给出

**位置**：p.8 第 7.1 节仅对 iTransformer、element 单元、MSE、6 个数据集做了随机掩码对照（"−0.16% in MSE ... not significant"）；MAE 未报告。第 2 节（p.3）引用了 Kang & Hashimoto（2020）的损失截断和 Han et al.（2018）的 Co-teaching，但没有任何与之的对比实验。没有 Huber 损失、trimmed loss 或 per-batch top-k 截断等稳健损失基线。

**问题**：
- 论文的核心主张是"高残差端有效、低残差端无效"。这要求证明：相同数量的随机掩码不产生同样的 MAE 增益。目前这一点只在 iTransformer、element、MSE 上检验。若随机 10% 掩码在 MAE 上也带来 1–2% 的改善（这在 Fig. 1 与 Table 1 中并不能排除），则"决策单元"的结论会大为不同。
- 屏蔽高残差 10% 在功能上近似于对高残差项做**稳健的截断/修剪**（trimmed loss），这是一个已有的、可直接比较的方法族。论文没有证明屏蔽优于简单的稳健损失。

**建议**：
1. 对 **所有五个单元、三个 backbone、七个数据集、两个指标（MAE、MSE）** 增加随机掩码（同预算、同单元结构、同长度分布）的对照。
2. 增加稳健损失基线：Huber 损失（Huber, 1964）；批内 top-k 损失截断（Kang & Hashimoto, 2020 的思路）；以及在相同预算下的"按打分删除"与"按打分降权"两种版本。
3. 增加分类/预测中常用的分数基线在相同预算下的对比（如 EL2N、遗忘分数），以说明"分数"是否比"任意 10% 掩码"更好。论文第 7.2 节仅在 iTransformer 的 element 上比较了分数，不足以支撑"分数有效"这一结论。

---

### W3：混杂：horizon 位置与 occurrence 的多重性未被排除

**位置**：p.4 第 3.2 节（"each element is supervised between 96 and 720 times per epoch"，即 c(t,d)=T）；p.5 第 4 节（四个 horizon 在表中被平均）；表 1–7 均为跨 horizon 的平均。

**问题**：
- 对于 element 与 occurrence 单元，occurrence 的多重性由预测长度 T 决定，而 occurrence 在窗口内的位置（horizon step k = t−i−L+1）决定了它的预测误差大小。误差通常随 k 增大而增大。
- 因此，高残差端的 occurrence 掩码很可能**主要落在长 horizon 步**上，等价于对长 horizon 的预测降权。这与 Hounie et al.（2024）的 loss shaping（按 horizon 位置加权）机制相近，但论文没有讨论。
- 论文的所有表格都对 horizon 取平均，因此无法判断增益是否主要来自某些 horizon，也无法判断在短 horizon 上是否仍然成立。

**建议**：
1. 报告被掩码的 occurrence 在 horizon step 上的分布（按 k 分箱），与全体分布相比。
2. 按 T∈{96,192,336,720} 分别报告 Table 1 的结果（至少 MAE）。
3. 增加 horizon 匹配的随机掩码（每个 k 上的被掩码比例与高端掩码相同），以检验增益是否仅来自 horizon 位置的重新加权。
4. 增加直接的 horizon 降权基线（按 k 的误差均值加权），作为"等价的简单机制"的对照。

---

### W4：对"高残差 = 其余序列未解释的误差"的解释未排除替代解释

**位置**：p.8 第 7.1 节（median Spearman 0.998；与已解释误差相关 0.22）；摘要；第 8 节结论（p.9）。

**问题**：
- 作者定义的"未解释误差"是在将该元素监督屏蔽后、同一早期时点读取的残差。该残差与用于打分的残差（同一早期时点、几乎相同的模型）几乎必然高度相关。Spearman 0.998 因此在很大程度上是**结构性的**，并不能证明"其余序列未能解释"这一语义。作者在 p.8 也承认"neither says that a target is unpredictable in principle"，但摘要与结论仍使用了较强的表述。
- 更简单的替代解释是：早期难以拟合的目标，往往是幅值大、局部波动大、处于峰值或变化点的目标，也可能是分布偏移区段（见 W13）。论文未检验分数是否只是目标幅值或局部方差的单调函数。
- 第 7 节的分析使用的数据集子集在不同小节之间不一致：7.1 不含 solar（五个数据集、20 个组合），7.2 与 7.3 不含 ECL（六个数据集），而 Table 4 含 ECL。两个数据集的排除理由未给出。

**建议**：
1. 在标准化后的目标空间中，检验分数与 |y|、局部方差、一阶差分幅值、通道尺度、时刻（time-of-day）、horizon step 的偏相关，并报告在控制这些变量后分数的排序能力。
2. 检验分数排序在训练后期（而非仅 10%）是否仍成立，或其与最终残差的关系如何。
3. 统一数据集子集，或为每个排除给出理由并在 7 个数据集上报告核心结果。
4. 将摘要中的"the error that the rest of the series leaves unexplained"改为更谨慎的说法，直到上述检验完成。

---

### W5：多处零结果以"不显著"作为依据，表述过强

**位置**：摘要（"masking the lowest-residual 10% never helps"）；p.5 第 5 节（"28 of the 30 single-backbone cases"，以及 "the two exceptions ... −0.06% and −0.19%"）；p.6–7 第 6 节（"The worst 10% of errors does not change measurably"，基于 Friedman p=0.27）；p.8 第 7.2 节（TDDS 与分数 "cannot be told apart"，p=0.45 与 0.16）。

**问题**：
- 摘要称低端掩码"从不有益"。但 p.5 的两个例外（TimeMixer 上 MSE 的 chan row −0.06%、element −0.19%）本身就是降低误差的方向，只是区间不足以排除增益。正确表述应为"未显示超过 0.3 点的增益"，而非"从不有益"。
- 低端掩码的结果只在 Fig. 1 中给出了 MAE 均值，未在任何表中给出每个数据集、每个 backbone 的数值；读者无法核对"28/30"这一断言。
- "尾部不变"的结论基于 Friedman 检验的不显著性（p=0.27）。这是"无法检出差异"，不是"无差异"。论文第 5 节对"在某个带宽内"的断言使用了 ±0.3 点的单侧检验（p.5），但在第 6 节对 CVaR 没有使用同一标准。事实上，CVaR@90 中 chan row 的 −0.52% 与 occurrence 的 +0.47% 都超出 ±0.3 的带宽，因此"尾部不变"并未按论文自己的标准得到证明。
- 同样，TDDS 与分数"无法区分"的说法应改用等效性检验（TOST）或置信区间。

**建议**：
1. 将摘要中的"never helps"改为具体的、可检验的表述（如"低端掩码的 MAE 增益不超过 0.3 点"），并在表格中给出低端结果（每个单元、每个数据集、每个 backbone）。
2. 对"尾部不变"与"无法区分"的断言，统一使用 TOST 或置信区间，并给出预先设定的等效带宽（与 ±0.3 一致或给出理由）。
3. 对 p.5 中"28/30"的断言，给出完整列表。

---

### W6：敏感性缺失——只有一个预算（10%）、一个打分时点（10% 训练进度）、一个回看长度（L=96）

**位置**：p.4 第 3.3 节（"At a budget r ... The main comparison uses r=10%"；打分"at 10% of the full number of epochs"）；p.5 第 4 节（L=96）；p.9 第 7.3 节（自适应预算只在窗口+element 的组合中使用，未给出与固定预算的比较）。

**问题**：
- "低端不有效""高端有效"等结论都在 r=10% 这一个点上得到。若增益在更高或更低预算下改变符号，结论会完全不同。
- 打分时点只取 10%，而"早期读取、接近代价高的分数"是论文的核心实用主张。如果在 1%、5%、25%、50% 或训练末期打分的结果显著不同，这一主张就只适用于特定的时点。
- L=96 固定。occurrence 的多重性与输入长度的交互没有考察。
- 第 3.3 节定义的自适应预算（Tukey 栅栏）在第 7.3 节被使用，但其结果（与固定预算相比）没有给出。

**建议**：
1. 对预算 r∈{5,10,20,30,50}% 扫描，至少对 high-end 与 low-end、五个单元报告 MAE。
2. 对打分时点扫描（如 1%、5%、10%、25%、50%、终点），报告核心单元的结果。
3. 增加至少一个更长的回看长度（如 L=336）的结果，以检验结论对输入长度的依赖。
4. 报告自适应预算的结果，并与固定预算比较。

---

### W7：统计设计与报告存在多处缺口

**位置**：p.5 第 4 节（"Inference"）；附录 A（p.11）；全文 p 值。

**问题**：
- 推断以七个数据集为单位，样本量很小。作者已承认这一点，但仍有多个 p 值（如 p ≤ 0.004、0.022、0.001、0.008、0.045、0.040 等）被并列报告，且"not adjusted for multiplicity"（p.5）。读者难以判断哪些结论在多重检验下仍成立。
- 表格中所有单元格只给出均值，不给出种子间标准差或置信区间。读者无法判断 1–2% 的差异是否超出种子噪声。
- 附录 A（p.11）写道"one ordinary full-data run is evaluated ... for every backbone, dataset, and horizon"，并未说明打分运行是否每个种子各做一次。如果一次打分被五个种子共享，掩码本身的随机性（打分噪声）就没有被评估；这是一个重要的未知项。
- "Runs that diverge are excluded"（p.5），但未报告被排除的运行数量与所在单元，可能造成选择偏差。
- "sign-flip test cannot fall below 1/64"（p.5）：七个数据集的双侧精确符号翻转检验的最小 p 值确为 1/64，应明确写为"双侧"。
- 第 4 节说"Every run is read at its epoch of lowest validation loss"，而打分时点定义为"10% of the full number of epochs"。若使用早停，"full number of epochs"的含义不明确。

**建议**：
1. 报告全数据基线的绝对误差（MAE、MSE）及种子标准差，并在关键比较上给出配对的种子级差异及其 95% 置信区间。
2. 明确打分是否按种子重做；若否，增加种子级的掩码重采样，以评估掩码本身的方差。
3. 报告被排除运行的数量与分布。
4. 对主要结论使用多重检验校正（如 Holm），或明确哪些结论是探索性的。
5. 明确"full number of epochs"的定义（最大 epoch 数或早停后的实际 epoch 数）。

---

### W8：推荐的默认单元（通道行）与论文自身的证据不一致

**位置**：p.7 "Which unit to use"："As a default we recommend the channel row: it lowers MAE on every dataset (3.46%), has the lowest MSE of the five units, though not significantly so, and shows no detectable change in the worst 10%."；结论 p.9。

**问题**：
- 论文的主要指标 MAE 与中位数误差上，通道行并不是最优单元：MAE 均值 −3.46%，在五个单元中排第三，明显弱于 occurrence（−4.72%）与 element（−3.81%）；中位数上 chan row −7.24% 远小于 occurrence −12.45%。
- 推荐的理由是"MSE 最低但不显著"（Friedman p=0.52），而"尾部不变"是一个不显著性的零结果（见 W5）。
- MSE 均值 −4.66%（DLinear）主要由 ETTh2（−17.76%）与 ETTm2（−14.55%）驱动，并非跨数据集的一致优势。
- 推荐本身是在同一批数据上得出的事后选择，没有独立的验证。

**建议**：
1. 将推荐按指标拆分：若关注典型误差，推荐 occurrence（或 element）；若关注 MSE，报告 chan row 的优势并说明其不显著。
2. 若要保留"默认单元"的推荐，应在独立的数据集或种子上验证，或明确说明为事后观察。
3. 修正 p.9 结论中的"channel rows by default"表述，使之与 Table 1–2 一致。

---

### W9：窗口级负结果与"剪枝"框架的张力

**位置**：标题；摘要末句；p.9 第 7.3 节（"None of the three rules that select windows by their score does better than random pruning at any rate from 10% to 90%"；等间距在 50%、70% 更优）。

**问题**：
- 论文的标题与摘要强调"剪枝"单元，但在唯一能真正节省计算量的操作（窗口剪枝，即减少优化步数）上，分数规则不优于随机剪枝，而等间距甚至更优。这一负结果在摘要中没有被提及。
- 因而，论文的实际贡献更接近于"损失层面的重加权/截断"，而非"数据剪枝"。在不节省计算的前提下，论文的价值取决于 W1 与 W2 的结果。

**建议**：
1. 在摘要与结论中明确报告窗口级结果为负（与随机剪枝无差异）。
2. 考虑将标题与框架改为"loss-level masking/trimming for forecasting"或类似表述，使之与证据一致。
3. 对窗口剪枝的 p 值做多重校正（论文 p.9 的多个 p 值在同一组比较中未校正）。

---

### W10：声称"每个单元移除相同比例 r 的损失项"未经验证

**位置**：p.4（"so every unit removes about the same share r of the loss terms"）；p.4 第 3.3 节（"At a budget r, each unit masks its ⌊rm+1/2⌋ highest-scoring members"）。

**问题**：
- 对于 window、time column、channel row、occurrence，每个成员包含相同数量的损失项（T×D、D、T、1），因此屏蔽 10% 成员恰好等于屏蔽 10% 的损失项。
- 但 element 单元的成员是目标 (t,d)，其包含的 occurrence 数 c(t,d)=|W(t)| 在序列边界处较少（附录 A 与 p.4）。因此屏蔽 10% 的 element 不等于屏蔽 10% 的 occurrence。若 element 的高残差与边界位置相关，则 element 的预算与 occurrence 的预算并不相同，直接比较（如 Table 1 中 element −3.81% vs occurrence −4.72%）会受到影响。

**建议**：报告每个单元实际屏蔽的 occurrence 比例（按数据集、backbone、horizon），并在 element 与 occurrence 的比较中对齐实际屏蔽比例，或给出按比例校正的比较。

---

### W11：文字断言与 Table 4 直接矛盾

**位置**：p.6 "Deciding per target gives up part of the gain"：原文 "the occurrence is ahead on every dataset of every backbone"。

**问题**：Table 4（p.13，iTransformer，MAE）中：
- solar：occurrence −5.64，element −7.57（element 更优，且表中以粗体标出 element）；
- ECL：occurrence −1.86，element −2.01（element 更优）。

因此"在每个数据集、每个 backbone 上 occurrence 都更优"的断言在 iTransformer 上不成立（至少两个数据集）。

**建议**：修正该断言（例如"在 DLinear 与 TimeMixer 的全部数据集上，以及 iTransformer 的 5/7 个数据集上"），并核对文中其他"every dataset"类断言（如 p.2、p.5、p.6 中的 "on every dataset"）。

---

### W12：Selective Learning 与"插值修复"的比较混淆了输入与标签

**位置**：p.8–9 第 7.2 节；附录 B（p.12）"the elements selected by the element score are not masked but replaced in the training split by linear interpolation along their channel ... an interpolated value enters both [inputs and targets]"；Table 3。

**问题**：
- 插值修复同时修改了输入与标签。因此它不是"同一损失层面的修复"，而是对输入分布的改变。在高残差（通常是峰值或突变）处，线性插值会平滑峰值，这一结果预期会使误差增大，因此"屏蔽优于修复"的结论并不能说明屏蔽本身的优越性。
- Selective Learning 的两种形式在打分与掩码定义上也与本文并不完全相同（附录 B 描述了 calibrated rates 与 re-scoring），对照的公平性需要更清楚的说明。

**建议**：(a) 增加"仅修改标签、不修改输入"的修复版本（若可行）；(b) 增加"用模型预测替代"（self-distillation 式）的修复基线；(c) 在文中明确说明插值会同时改变输入，并对结论作相应的限定。

---

### W13：分布偏移与时间位置的替代解释

**位置**：全文；第 7.1 节未涉及；附录 A 未说明归一化方式。

**问题**：
- ETT、weather、solar、ECL 等序列存在显著的非平稳性与季节性。训练序列中早期时段的窗口，可能与测试期分布不同。若高残差端恰好集中在训练序列的早期时段，则"屏蔽高残差"实际上是一种**时间重加权**（相对强调近期窗口），而非"剪枝"。
- 论文未报告被屏蔽项在训练序列中的时间位置分布，也未说明是否使用了可逆归一化（如 RevIN）或每通道标准化。归一化方式直接影响通道行单元的选择以及 MSE 的尺度依赖（见 W16）。

**建议**：(a) 报告被屏蔽项在训练序列中的时间位置分布；(b) 检验增益是否依赖于时间位置（如只屏蔽同一时段内的高残差项）；(c) 明确说明归一化方式，并在有/无 RevIN（或等效归一化）的情况下检验结论。

---

### W14：相关工作的定位需要补充与修正

**位置**：第 2 节（p.3）。

**问题**：
- 缺少若干与"按样本/按损失项选择"直接相关的经典工作：CCS（Zheng et al., 2023）、Moderate coreset（Xia et al., 2023）、DeepCore 基准（Guo et al., 2022）、MentorNet（Jiang et al., 2018）等。它们对"在相同预算下比较分数"的标准实践有参考价值。
- InfoBatch（Qin et al., 2024）仅以"dynamic re-scoring"一笔带过。据我对该方法的理解，InfoBatch 倾向于按损失随机删除**低损失**样本并对剩余样本重新缩放，其方向与本文"低端掩码无益"的发现相反。作者应讨论这一差异（例如分类与预测中的样本分布、重缩放的作用、时间依赖结构），这是一个对本文结论很重要的对照点。
- "Selective Learning ... is the one prior forecasting method that masks supervision below the window"（p.3）的表述过于绝对。Hounie et al.（2024）的 loss shaping 按 horizon 位置对损失加权，与本文的 occurrence 单元在机制上最接近；应明确限定"屏蔽"与"加权"的区别。Wu et al.（2026）虽在时间点级计算影响，但选择的仍是整个样本，因此不构成"窗口以下的选择"的先例。
- 第 2 节与 p.2 的"从分类继承窗口单元"的历史叙述较为笼统，可引用更直接的文献（如分类中关于"保留困难样本 vs 删除困难样本"的讨论，以及噪声标签文献中的损失截断）。

**建议**：补充上述文献，讨论 InfoBatch 的方向差异，并限定"唯一"类表述。

---

### W15：可复现性与实验范围

**位置**：附录 A（p.11）；全文。

**问题**：
- 未说明代码、掩码文件、打分脚本是否公开（匿名投稿阶段可以说明将公开）。
- 未逐一列出各 backbone 的 TSLib 训练配方（学习率、batch size、dropout、epoch 上限、早停耐心），也未说明归一化与早停的具体设置。
- 实验范围有限：三个 backbone（其中两个为通道独立或线性结构）、七个数据集、L=96、固定的预测长度集合，未包含时间序列基础模型。论文的一般性主张（如"the gain lives in the bulk"）应在范围内明确限定。
- 未报告墙钟时间（与 W1 相关）。

**建议**：公开代码与掩码，列出完整超参表，并在结论中明确实验范围。

---

### W16：MSE 解释的论证与尺度依赖

**位置**：p.7 "Why MSE moves less"。

**问题**：
- RMSE 的变化在数学上约为 MSE 变化的一半（图 2b 与表 1 中的数值与之一致），因此"MSE 改善较小"在一定程度上是度量的代数关系，而非独立证据。
- MSE 对数据集的尺度敏感。若数据未按通道标准化，solar 与 ECL 的 MSE 变化将主要反映高幅值通道的行为（论文中 ECL 上 MSE 上升，可能与此有关）。
- "尾部"解释的直接检验缺失：例如比较被屏蔽/未屏蔽项对误差平方和前 1%、前 5% 的贡献变化。

**建议**：报告尺度归一化后的 MSE（或 NMSE），并直接检验尾部贡献的变化。

---

## 5. 领域层面的分析

**研究脉络**：数据选择/剪枝在分类中的主线是"按分数（EL2N、GraNd、遗忘、预测方差）排序后保留或删除样本"，并发现"高分数（困难）样本"的取舍依赖预算与数据冗余程度（Sorscher et al., 2022；Ayed & Hayou, 2023）。时间序列预测借用了这一框架，但将样本定义为滑动窗口（You et al., 2024；Liu et al., 2025）。本文的贡献是把"剪枝单元"从窗口拆分为五种粒度，并指出：在预测中，窗口之间的重叠使同一目标被多次监督，因此单元的选择会改变"被删除的是什么"。

**隐藏联系**：
- 本文的 element 单元与 Selective Learning 的"按目标掩码"和 Rho-1（Lin et al., 2024）的"按 token 选择"在结构上相似：三者都把监督的粒度降到"目标"或"token"层面。Rho-1 的成功依赖于参考模型的超额损失，而本文不需要参考模型，这一点值得讨论。
- 高端掩码的有效性与 Co-teaching（Han et al., 2018）、损失截断（Kang & Hashimoto, 2020）的噪声标签直觉相关，但方向相反：噪声标签文献通常删除高损失样本，本文在预测中也如此，而分类中的"保留困难样本"直觉被颠覆。论文应更深入地解释这一差异是否来自预测目标的噪声、分布偏移或 horizon 结构（见 W3、W13）。
- 等间距窗口与随机剪枝同样有效，暗示在滑动窗口预测中，窗口本身的冗余度远高于样本分数能提供的信息。这一点对实践有直接意义，但论文只将其作为一个附带结果。

**研究空白**：
1. 缺少在相同计算量下的剪枝/重加权比较（W1）。
2. 缺少在预测任务中系统比较"分数规则"与"随机规则"的研究（本文仅在窗口级做了）。
3. 缺少对预算、打分时点、回看长度的系统敏感性分析（W6）。
4. 缺少对"分布偏移下的剪枝"的研究（W13）。

**可选的对比表（建议作者补充）**：

| 方法 | 单元 | 是否需要额外模型/运行 | 方向（删高/删低） | 在预测中的已知结果 |
|---|---|---|---|---|
| EL2N / GraNd（Paul et al., 2021） | 样本 | 早期训练 | 删低（保留困难） | 分类中有效；本文未在预测中比较 |
| 遗忘分数（Toneva et al., 2019） | 样本 | 完整训练历史 | 删低 | 分类 |
| InfoBatch（Qin et al., 2024） | 样本 | 无 | 删低 + 重缩放 | 分类；与本文方向相反 |
| 损失截断（Kang & Hashimoto, 2020） | 样本/批内 | 无 | 删高 | 语言模型 |
| Selective Learning（Fu et al., 2025） | 目标 | 辅助估计器 | 删高（熵+异常掩码） | 预测；本文对照 |
| 本文 occurrence/element/window | 损失项 | 10% 前缀运行 | 删高 | 预测（本文） |

---

## 6. 给作者的问题清单

1. 在相同总优化步数（或墙钟时间）下，全数据基线与本方法的差异是多少？（W1）
2. 随机 10% 掩码在所有五个单元、三个 backbone、MAE 与 MSE 上的结果是什么？（W2）
3. 按 horizon step k 分箱，被屏蔽 occurrence 的分布如何？与 horizon 匹配的随机掩码相比结果如何？（W3）
4. 在控制 |y|、局部方差、通道尺度与时刻之后，分数的排序能力是否仍然存在？（W4）
5. 低端掩码在每个数据集、backbone、单元上的具体数值是多少？"28/30"的断言请给出完整列表。（W5）
6. 在 r∈{5,10,20,30,50}% 与不同打分时点（1%–终点）下，高端与低端的结论是否稳定？（W6）
7. 打分是否按种子重做？若否，掩码的随机性如何影响结果？（W7）
8. 被排除的发散运行有多少，分布在哪些单元与数据集中？（W7）
9. 为何 ECL 在第 7 节被排除而 solar 仅在 7.1 中被排除？（W4）
10. 在 iTransformer 的 solar 与 ECL 上，element 优于 occurrence 的事实（见 Table 4）是否与正文"every dataset of every backbone"的断言一致？（W11）
11. 数据是否做了按通道标准化或 RevIN？这如何影响通道行单元与 MSE？（W13、W16）
12. 为什么推荐通道行作为默认单元，而 MAE 与中位数指标上它明显不是最优？（W8）
13. 对于 InfoBatch 的"删低损失样本 + 重缩放"，本文的低端结果为何与之相反？（W14）

---

## 7. 次要问题（排版、笔误、符号）

1. **符号 r 的重载**：r 既表示屏蔽预算（r=10%，p.4），又在 r_i(t,d) 中表示残差。建议将残差改为 e_i(t,d) 或将预算改为 ρ。
2. **命名不统一**：摘要中称"a single loss term"与"a single term"，正文称"occurrence"。建议在摘要中统一用语。
3. **式 (4) 的归一化**：分母为 Σ_i Σ_{t∈T_i} Σ_d w，但式 (1) 中的 1/(nTD) 在式 (4) 中消失。建议注明式 (4) 是归一化的加权平均，并说明其与式 (1) 在全部 w=1 时的一致性。
4. **Figure 2(a) 与 Table 2 的分位数不一致**：图 2(a) 含 q75 与 q95，表 2 含 q90 与 CVaR，但 p.6 文字引用了 q95 的数值（"between −0.05% and −0.91%"）而表中没有。建议将 q95 数值补入表中。
5. **Table 3 的 "ours" 行**的标签为"mean residual at 10% of training (ours)"，但其数值（−2.81 MSE，−3.97 MAE）是 iTransformer 上六个数据集的 element 单元结果。建议在标题中明确"element unit, six datasets"，以免与表 1 的 channel row MSE（−2.81）混淆。
6. **p.5 "sign-flip test"**：应明确写为"two-sided exact sign-flip test"，并说明其在七个数据集上的最小 p 值。
7. **附录 A "Ties are broken by array index"**：应说明这一规则对 window、time column 单元是否可能产生实际的平局。
8. **Figure 1 caption**：括号中 "(Table 1 and Section 5)" 的指代不够清楚，建议改为 "(see Table 1 and Section 5)"。
9. **参考文献格式**：部分条目为 arXiv 与会议版本混合，建议统一为 TMLR 风格；Li et al.（2026）的标题应核对是否与 TOIS 版本一致。
10. **p.4 第 3.1 节**："consecutive windows are one step apart, and there are n of them"：建议注明 n = N − L − T + 1，以便读者核对。
11. **p.2 "Three findings emerge"** 与第 6 节标题的"bulk of the error distribution"表述可统一，便于读者对应。

---

## 8. TMLR 两个标准下的评估与推荐

**标准 (a)：主张是否有准确、令人信服、清晰的证据支持？**
- 核心方向性结论（高残差端屏蔽在 10% 预算下降低 MAE，窗口增益最小，occurrence 增益最大）**有较好的证据支持**，且统计检验的设计（配对、以数据集为单位）总体合理。
- 以下主张的证据**不足或表述过强**：低端"从不有益"（W5）；"尾部不变"（W5）；"occurrence 在每个数据集上优于 element"（被 Table 4 反驳，W11）；"无需第二次训练、成本可忽略"（W1）；"分数衡量的是其余序列未解释的误差"（W4）；"默认推荐通道行"（W8）。
- 核心的"分数有效"主张缺少随机掩码与稳健损失基线的全面对照（W2），目前无法区分"分数"与"任意掩码/截断"的贡献。

**标准 (b)：是否有 TMLR 读者会感兴趣的内容？**
- 有。滑动窗口预测中"决策单元"的概念是清晰的，并且有可能影响后续的数据选择研究。
- 但论文的实际效用受限：掩码不节省计算（W1），且唯一能节省计算的窗口剪枝在分数上没有优势（W9）。若作者能证明在计算量匹配下，高端掩码或其替代形式仍有稳定优势，论文的价值会大为提升。

**总体推荐**：**Major revision（大修）**。

**理由**：论文的对照设计与核心观察有价值，但在计算量匹配、关键基线、混杂排除与若干断言的正确性上存在实质性缺口。上述缺口需要新的实验（而非仅改写），因此不宜仅以 minor revision 处理。若作者能完成 W1–W4 与 W11 的修正，论文将达到接收水准；若 W1 的计算量匹配结果显示增益主要来自额外训练，结论需要根本性改写。

**置信度**：4/5。

---

## 9. 参考文献（Harvard 格式）

以下列出本审稿意见中引用的文献。其中论文已引用的条目按论文中的信息整理；本审稿新增的文献（标注 *）仅列出我有把握存在的条目。

Ayed, F. and Hayou, S. (2023) 'Data pruning and neural scaling laws: fundamental limitations of score-based algorithms', *Transactions on Machine Learning Research*. arXiv:2302.06960.

Chen, W., Chen, J., Wu, Y., Liang, Y. and Zhou, X. (2026) 'Learning from complexity: exploring dynamic sample pruning of spatio-temporal training', arXiv:2602.19113.

Cheng, H., Wen, Q., Liu, Y. and Sun, L. (2024) 'RobustTSF: towards theory and design of robust time series forecasting with anomalies', in *International Conference on Learning Representations*. arXiv:2402.02032.

Fu, Y., Shao, Z., Yu, C., Li, Y., An, Z., Wang, Q., Xu, Y. and Wang, F. (2025) 'Selective learning for deep time series forecasting', in *Advances in Neural Information Processing Systems*. arXiv:2510.25207.

Gal, Y. and Ghahramani, Z. (2016) 'Dropout as a Bayesian approximation: representing model uncertainty in deep learning', in *International Conference on Machine Learning*. arXiv:1506.02142.

Grosz, S., Zhao, R., Ranjan, R., Wang, H., Aggarwal, M., Medioni, G. and Jain, A. (2024) 'Data pruning via separability, integrity, and model uncertainty-aware importance sampling', in *International Conference on Pattern Recognition*. arXiv:2409.13915.

Han, B., Yao, Q., Yu, X., Niu, G., Xu, M., Hu, W., Tsang, I. W. and Sugiyama, M. (2018) 'Co-teaching: robust training of deep neural networks with extremely noisy labels', in *Advances in Neural Information Processing Systems*. arXiv:1804.06872.

Hounie, I., Porras-Valenzuela, J. and Ribeiro, A. (2024) 'Loss shaping constraints for long-term time series forecasting', in *International Conference on Machine Learning*. arXiv:2402.09373.

Kang, D. and Hashimoto, T. B. (2020) 'Improved natural language generation via loss truncation', in *Proceedings of the 58th Annual Meeting of the Association for Computational Linguistics*. doi:10.18653/v1/2020.acl-main.66.

Lin, Z., Gou, Z., Gong, Y., Liu, X., Shen, Y., Xu, R., Lin, C., Yang, Y., Jiao, J., Duan, N. and Chen, W. (2024) 'Not all tokens are what you need for pretraining', in *Advances in Neural Information Processing Systems*. arXiv:2404.07965.

Liu, Y., Hu, T., Zhang, H., Wu, H., Wang, S., Ma, L. and Long, M. (2024) 'iTransformer: inverted transformers are effective for time series forecasting', in *International Conference on Learning Representations*. arXiv:2310.06625.

Liu, T., Wan, M., Wang, J. and Nie, N. (2025) 'Enlightenment period improving DNN performance', arXiv:2504.01737.

Mindermann, S., Brauner, J., Razzak, M., Sharma, M., Kirsch, A., Xu, W., Höltgen, B., Gomez, A. N., Morisot, A., Farquhar, S. and Gal, Y. (2022) 'Prioritized training on points that are learnable, worth learning, and not yet learnt', in *International Conference on Machine Learning*. arXiv:2206.07137.

Okanovic, P., Waleffe, R., Mageirakos, V., Nikolakakis, K. E., Karbasi, A., Kalogerias, D., Gürel, N. M. and Rekatsinas, T. (2024) 'Repeated random sampling for minimizing the time-to-accuracy of learning', in *International Conference on Learning Representations*. arXiv:2305.18424.

Paul, M., Ganguli, S. and Dziugaite, G. K. (2021) 'Deep learning on a data diet: finding important examples early in training', in *Advances in Neural Information Processing Systems*. arXiv:2107.07075.

Qin, Z., Wang, K., Zheng, Z., Gu, J., Peng, X., Xu, Z., Zhou, D., Shang, L., Sun, B., Xie, X. and You, Y. (2024) 'InfoBatch: lossless training speed up by unbiased dynamic data pruning', in *International Conference on Learning Representations*. arXiv:2303.04947.

Sorscher, B., Geirhos, R., Shekhar, S., Ganguli, S. and Morcos, A. S. (2022) 'Beyond neural scaling laws: beating power law scaling via data pruning', in *Advances in Neural Information Processing Systems*. arXiv:2206.14486.

Toneva, M., Sordoni, A., Tachet des Combes, R., Trischler, A., Bengio, Y. and Gordon, G. J. (2019) 'An empirical study of example forgetting during deep neural network learning', in *International Conference on Learning Representations*. arXiv:1812.05159.

Wang, S., Wu, H., Shi, X., Hu, T., Luo, H., Ma, L., Zhang, J. Y. and Zhou, J. (2024) 'TimeMixer: decomposable multiscale mixing for time series forecasting', in *International Conference on Learning Representations*.

Wu, S., Li, T., Leng, Y., Suo, J., Lou, J., Li, D. and Ng, S.-K. (2026) 'Lightweight time series data valuation on time series foundation models via in-context finetuning', in *International Conference on Database Systems for Advanced Applications*. arXiv:2511.12988.

Zeng, A., Chen, M., Zhang, L. and Xu, Q. (2023) 'Are transformers effective for time series forecasting?', in *Proceedings of the AAAI Conference on Artificial Intelligence*.

Zhang, X., Du, J., Li, Y., Xie, W. and Zhou, J. T. (2024) 'Spanning training progress: temporal dual-depth scoring (TDDS) for enhanced dataset pruning', in *IEEE/CVF Conference on Computer Vision and Pattern Recognition*. arXiv:2311.13613.

You, J., Cela, A., Natowicz, R., Ouanounou, J. and Siarry, P. (2024) 'Robust time series forecasting with non-heavy-tailed Gaussian loss-weighted sampler', arXiv:2406.13871.

*Guo, C., Zhao, B., Bai, Y. (2022) 'DeepCore: a comprehensive library for coreset selection in deep learning', in *International Conference on Database Systems for Advanced Applications*. arXiv:2106.08970.

*Huber, P. J. (1964) 'Robust estimation of a location parameter', *The Annals of Mathematical Statistics*, 35(1), pp. 73–101.

*Jiang, L., Zhou, Z., Leung, T., Li, L.-J. and Fei-Fei, L. (2018) 'MentorNet: learning data-driven curriculum for very deep neural networks on corrupted labels', in *International Conference on Machine Learning*.

*Xia, X., Liu, J., Zhang, S., Wu, Q., Wei, H. and Liu, T. (2023) 'Moderate coreset: a universal method of data selection for real-world data-efficient deep learning', in *International Conference on Learning Representations*.

*Zheng, H., Liu, R., Lai, F. and Prakash, A. (2023) 'Coverage-centric coreset selection for high pruning rates', in *International Conference on Learning Representations*.

（注：带 * 的条目为本审稿新增引用，作者应核对其具体版本与页码；论文自身的参考文献条目（如 Li et al. 2026、Pennino & Gabbrielli 2026、Liu et al. 2026 TimeLAVA）本审稿未逐一核对外部版本信息，仅按论文原文引用。）
