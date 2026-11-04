# TMLR 审稿报告：*Prune the Supervision, Not the Windows: Decision Units for Data Pruning in Time-Series Forecasting*

（独立审稿，仅基于论文正文 14 页：正文 1–9 页、参考文献 9–11 页、附录 11–14 页。）

## 1. 论文摘要

滑窗训练时，同一个目标值 $y_{t,d}$ 在每个 epoch 中被约 $T$ 个不同窗口各监督一次（"occurrence"，式(1)）。作者认为，沿用分类领域的做法、以整个窗口作为剪枝单元，是粗粒度且次优的。

论文在固定"分数"和固定预算（10%）的条件下，只改变"决策单元"（decision unit，式(2)）：
- window（$T\times D$ 项）
- time column（$D$ 项）
- channel row（$T$ 项）
- occurrence（单个 loss 项）
- element（同一目标值在所有窗口中的 $c(t,d)$ 个 occurrence）

分数取完整数据训练到 10% epoch 处，对所有训练窗口做一次 eval 前向得到的平方残差，再按单元取平均（式(3)）。"剪枝"实际上是把得分最高的 10% loss 项权重置零（式(4)）。所有输入窗口都保留，因此一个 epoch 的计算量不变。

实验覆盖 3 个骨干（iTransformer、DLinear、TimeMixer）、7 个数据集（ETT 四个、weather、solar、ECL）、4 个预测步长、5 个随机种子。统计推断以"数据集"为单位（n=7，配对 t 检验、符号翻转检验、Friedman 检验）。

主要结论：
1. 只有 mask 高残差端才有收益：MAE 下降 2.03%（window）到 4.72%（occurrence），mask 低残差端无收益。
2. window 收益最小；收益集中在误差分布的中部（中位数误差下降 4.4%–12.5%），在 95 分位数以上和 CVaR@90% 上不可测，因此 MSE 的改善更小且不显著。
3. 在 iTransformer 上（第 7 节），该分数基本等于"系统其余部分无法解释的残差"；一次早期打分就接近 MC-dropout、TDDS 等更贵的分数；element mask 优于 Selective Learning（Fu et al., 2025）以及"mask 后用插值修复"；若要省优化步数，等间隔去掉窗口不比随机差。

最终建议：默认用 channel row 单元，仅关心典型误差时用 occurrence，并且永远不要 mask 低端。

## 2. 主要主张与证据充分性

| # | 主张 | 位置 | 证据 | 我的评估 |
|---|------|------|------|----------|
| C1 | 只有高端 mask 有益，低端从不有益 | §5, 图1, 表1 | 10% 预算，3 骨干×7 数据集×4 步长×5 种子；低端变化在 −0.05%~+0.51% | 经验现象扎实。但解释为"决策单元/剪枝"而非"鲁棒/截断损失"缺少关键对照（见 W1） |
| C2 | 高端 mask 在所有单元、所有数据集上降低 MAE | 表1 | 7/7 数据集 | 与基线（全数据 MSE 训练）比较成立；但未与"同等鲁棒损失/随机 mask"对照，难以归因（W1、W2） |
| C3 | window 收益最小，channel row/occurrence 显著优于 window | §5 | Friedman p=0.002，平均秩 4.71/5 | 在 MAE 上证据较强；MSE 上单元不可区分（p=0.52），这削弱了"单元选择重要"这一总体叙事（W4） |
| C4 | 收益在误差分布中部，尾部不变 | §6, 表2, 图2 | 分位数、CVaR@90% | 描述性证据好；"最坏 10% 无变化"仅为"检验无力拒绝"，且是一个与 MSE-vs-MAE 训练目标不匹配相一致的解释（W1） |
| C5 | channel row 是推荐默认 | §5/§8 | MAE 7/7，MSE 均值最低但不显著 | 推荐依据偏弱：MSE 最佳单元在各骨干/数据集间不稳定，ECL 上 MSE 为 +0.05%（W4） |
| C6 | 分数 ≈ "未被其余序列解释的误差"，gain 取决于"mask 了哪些目标"而非 mask 形状 | §7.1 | 中位 Spearman 0.998；匹配随机 mask 只得到其收益的 6% | 相关性接近同义反复（W5）；仅在 iTransformer、6 数据集上；"unexplained"的解释未与"噪声/异常/离群"区分 |
| C7 | 早期单次分数接近昂贵分数；element mask 优于 Selective Learning；mask 优于修复 | §7.2, 表3 | 6 数据集，iTransformer | 比较的是作者重实现的、重新校准过阈值的 Selective Learning；且基线集合不含任何鲁棒损失/简单 loss-clipping（W2、W6） |
| C8 | 等间隔去窗口省 30% 步数不损失 | §7.3 | +0.30 MSE，p=0.58 | 属"未能显著"≠"等价"；只有一个骨干、较短的论证（W7） |

## 3. 优点

1. **问题切分清晰、概念贡献有价值。** 指出滑窗训练中"同一目标被监督 $T$ 次、窗口只是最粗的决策单元"，把 sample/time column/channel row/occurrence/element 统一到式(2)，是对 TS 数据剪枝文献的有益澄清。
2. **控制变量干净。** 分数、预算、骨干配方、种子均固定，只改单元；"mask 而非删除"（式(4)）让输入完整，避免了数据量变化与输入分布变化的混杂。
3. **统计推断意识强。** 明确指出四个步长共享训练序列、三个骨干共享数据集，因此以数据集为推断单位，并报告符号翻转检验的最小 p 值（1/64）、±0.3 点的等价性单侧检验、Friedman 检验。这在 TS 论文中少见，值得肯定。
4. **报告了误差分布而非仅均值**（中位数、q75/q90/q95、CVaR@90%、RMSE，图2、表2），揭示 MAE 与 MSE 的不一致，这是诚实且有信息量的结果。
5. **低端对照与负结果。** 同时报告 mask 低端（无益）、RobustTSF 残差（几乎无益）、"随机 mask 匹配形状"（几乎无益），以及 ECL 上 MSE 变差，附录给出分骨干表与去掉 ECL 的版本，透明度较好。
6. 计算开销被诚实地提出：mask 不缩短 epoch，并在 §7.3 单独讨论节省优化步数。

## 4. 缺点与修改建议（按重要性排序）

### W1（最关键）：所观察到的"收益"很可能是鲁棒/截断损失效应，而非"数据剪枝"；缺少把二者区分开的对照

- **证据。** 训练损失是 MSE（式(1)(4)），而最主要的收益体现在 MAE 与中位数绝对误差（表1：MAE −2.03%~−4.72%；表2：median |e| −4.42%~−12.45%），尾部（q95、CVaR@90%）无变化，MSE 提升小且不显著。这是"以 MSE 训练、按 MAE 评测"时，剔除大残差项（trimmed/truncated MSE）使估计从均值向中位数靠拢的典型特征。作者将该分数解读为 §7.1 的"未被解释的误差"，也与"异常点/离群点/噪声"一致。论文引用了 loss truncation（Kang and Hashimoto, 2020）和 Co-teaching（Han et al., 2018），但在实验中未纳入。
- **为何重要。** 论文标题与摘要主张的是"决策单元对数据剪枝很重要"。如果同样的增益能由 Huber/L1 损失、固定阈值 loss clipping 或动态 trimmed loss 免费获得，那么这里的发现更接近"TS 训练应使用鲁棒损失"，而不是数据剪枝的新规律；"never prune the low end"也同样是 trimmed loss 的平凡推论。
- **建议的补充实验：**
  1. 用 L1 损失、Huber（δ 扫描）及 MAE 评测直接对比全数据训练，报告全数据 L1 训练相对全数据 MSE 训练的增益，作为"天花板/基线"。
  2. 加入 online trimmed MSE（每个 batch 丢弃最大 10% 残差项）和 loss clipping 作为额外"单元=occurrence、动态分数"的基线。
  3. 在 L1/Huber 训练条件下重做表1，检验 element/channel row mask 是否还有额外收益；若消失，则说明收益来自损失函数错配。
  4. 评测同时给出 MSE-训练-MSE-评测、L1-训练-MAE-评测两个"对齐"设置，报告各自结论。

### W2：缺少最基本的随机 mask 与 window-level 之外的现有剪枝基线；"相对于全数据"的比较无法给出剪枝方法的相对位置

- **证据。** 主表（表1、2）只报告相对 full-data 的变化，各单元之间比较而无随机 mask 基线（§7.1 仅有"形状匹配的随机 mask"，且仅 iTransformer、6 个数据集）。论文的 §2 列举了 EL2N 类早期误差、GraNd、forgetting、TDDS、dynamic uncertainty、InfoBatch 等，但除 TDDS、MC-dropout、Selective Learning 外都没有跑；CCS（Zheng et al., 2023）、Moderate（Xia et al., 2023）、InfoBatch（Qin et al., 2024）、DUAL 等也没有。引言明确说"这些方法为图像分类设计"，却没有用它们的 window 版本作为公平基线。
- **建议：** 每个预算（10/30/50/70/90%）、每个单元都加上均匀随机 mask 基线，并报告"相对随机"的增益；至少加入 window 单元上的 InfoBatch 式动态剪枝、EL2N 风格分数、CCS 的分层覆盖选择，以及时间区间均匀/按时间分层采样，用作"classification-origin"代表。

### W3：预算范围过窄，"剪枝"主张缺少大预算与随预算变化的证据

- **证据。** 主比较仅 $r=10\%$（§3.3）；高预算（50%/70%）只在 §7.3 的窗口剪枝里出现；自适应 Tukey fence 预算虽然提到，却没有展示 fence 实际 mask 比例。
- **为何重要。** 数据剪枝文献的核心现象（Sorscher et al., 2022；Ayed and Hayou, 2023）是"高/低端的最优选择随保留率反转"。仅看 10% 无法判断"never the low end"是否普遍成立；在更高预算下，高端 mask 终究会剔除有信息的难例，预期会出现收益转负的拐点，这恰是 TMLR 读者关心的。
- **建议：** 画出 MAE/MSE 变化随 $r\in\{5,10,20,30,50,70\}\%$ 的曲线，对每个单元报告"拐点"；报告自适应预算实际落在哪个比例。

### W4：单元比较的结论在 MSE 上不成立，"推荐 channel row"的依据脆弱；多重检验与显著性解读偏宽松

- **证据。** 表2：MSE 上 Friedman p=0.525（去掉 ECL 后 0.171，表7），CVaR@90% p=0.273；channel row 的 MSE 提升 2.81% 被说明为"not significantly"优于其他。表1：MSE 逐数据集最佳单元在 channel row（4 个数据集）、occurrence（ETTm1、ECL 等）、element（weather、solar）间分散；ECL 上 channel row 对 MSE 为 +0.05%，TimeMixer ECL window 为 +0.02%。作者所称"一些单元在 MSE 上优于全数据"在 §5 里仅为 5/7~7/7 数据集。§4 声明"p 值不做多重校正"，而全文报告了数十个检验（含五个单元×五个度量×三个骨干、等价检验、Friedman、配对 t），其中不少 p 在 0.01~0.06 之间（如 §7.2 的 p=0.045、0.036、0.040、0.027）。
- **为何重要。** n=7 的数据集中 ETTh1/ETTh2/ETTm1/ETTm2 本质上是同一来源（同一变压器站的不同变量/采样率），并非独立样本，Weather/ECL/Solar 才较独立。以数据集为单位已经偏保守，但 ETT 四数据集内部相关使有效 n 更小；同时，边缘显著的 p 值若不做校正，"优于"陈述需要弱化。
- **建议：** (a) 对每一族主张预先指定一个主检验，其余作为探索性并作 Holm/BH 校正；(b) 以 bootstrap 报告每个比较的置信区间而非仅 p 值；(c) 把"channel row 是推荐默认"改为"在 MAE 上稳健、MSE 上与其他单元无差别"，或补充更多非 ETT 数据集（Traffic、Exchange、Electricity 以外的 Illness、PEMS 等）后再做建议；(d) 在摘要里同样如实反映 MSE 结果不显著。

### W5：§7.1 的"分数 = 未被解释的误差"几乎是同义反复，且"是否为噪声/异常"这一更自然的替代解释没有被排除

- **证据。** 分数是训练 10% epoch 时的残差；"unexplained error"是该元素在其监督被 mask 的另一次训练中的残差。二者的 Spearman 为 0.998（中位）。一个在训练早期尚未拟合的元素，在从未见过它的模型里仍然拟合不好，这几乎是由"此元素本身是难预测的值（尖峰、异常、突变）"导致的，并不提供机制洞见；而"explained error"（untrained 到 trained 的下降）相关仅 0.22，说明 mask 选出的是**本来就很难学**的点，而非"可学但未学会"的点。作者也承认"neither says that a target is unpredictable in principle"。
- **混杂/替代解释：** mask 的目标集中在若干连续"stretch"（§7.1 提到长度与一天中时间匹配），极可能是异常段、尖峰、分布漂移段或 solar 的日出日落过渡段。RobustTSF 残差基线（Cheng et al., 2024）基本无效，但该基线仅使用了其趋势滤波器实现；这并不能证明"异常"解释不成立。
- **建议：** (1) 可视化被 mask 元素在时间轴与通道上的分布，并统计它们占各通道的比例，是否集中于少数通道/时段；(2) 与基于异常检测（Isolation Forest、Hampel、鲁棒 z-score）的 mask 对照；(3) 在训练序列中人工注入已知异常（脉冲/平移）后检验 mask 是否捕获，并比较 mask 前后的鲁棒性；(4) 检查测试集误差的位置：被 mask 的训练元素是否对应测试期同样"困难"的时段。

### W6：与 Selective Learning 的比较公平性存疑，且 element mask 的优势幅度有限

- **证据。** 表3：element（ours）MSE −2.81/MAE −3.97；Selective Learning "每 epoch 重打分" −2.13/−1.94，"打一次" −2.41/−2.21；MSE 差距 0.68 和 0.40 点，p=0.07、0.46，不显著；MAE 差距 2.03 和 1.76 点，p=0.013、0.040。附录 B 说明"Selective Learning 的两种形式的名义阈值被校准到实现的 mask 比例匹配 10%（半个点内）"，原方法的异常 mask、不确定性 mask 是被改造过的。MC-dropout（需 5 次前向）反而比我们的分数在 MSE 与 MAE 上更好（−3.05/−4.04）。
- **为何重要。** 在 MSE 上论文并没有证明优于 Selective Learning；校准阈值、不使用其官方超参和损失，可能使对手低于其论文性能。另外，Selective Learning 针对的是 MSE 训练的鲁棒性，与 W1 的"鲁棒损失"问题直接相关。
- **建议：** 使用作者发布代码在其原始设置下复现，并报告其论文里的相同数据集上的数字；增加骨干（至少 DLinear/TimeMixer）而不只 iTransformer；报告 mask 比例的实际分布；摘要中"ahead of Selective Learning in MAE on all six datasets"应补充"MSE 上无显著差异"。

### W7：§7.3（节省优化步数）论证过薄，这才是"数据剪枝"的实际价值所在

- **证据。** "Removing 30% of windows at equal spacing changes MSE by +0.30 points ... p=0.58" 被解读为"可以节省 30% 优化步数"；p=0.58 只是未能检出差异，作者自己也承认"the data cannot show that it is zero"。仅 iTransformer、6 个数据集；未报告墙钟时间；"自适应预算"（Tukey fence）下，实际 mask 多少、窗口剪枝在其之上叠加是什么比例并不清楚。窗口剪枝的"等间隔"在 stride-1 滑窗下本质上是增大 stride，与"滑窗冗余"文献（Li et al., 2026）的结论相近，新颖性有限。
- **建议：** 报告 wall-clock/FLOPs，对窗口保留率 10–100% 画权衡曲线，加入 DLinear、TimeMixer；把"等间隔 ≈ 随机"作为明确结论但不要作为"节省"的证据，除非使用 TOST 等价检验（论文已有 ±0.3 点的框架，应在此处统一使用）；与 InfoBatch 等真正省算力的方法比较。

### W8：全数据基线与"配方"相关的公平性/超参问题

- **证据。** §4 和附录 A：所有运行使用 TSLib 官方配方；masked run "differs only in the loss weights"。式(4)对掩码后的权重和归一化，改变了有效梯度尺度；mask 后每步的有效样本数变化，而学习率、epoch、早停耐心不变；每个 run 按验证集最低 loss 读出，但未说明验证集损失是否也被 mask（如果验证用全数据，则是各 run 一致；如果被 mask 则不可比）。"10% of the full number of epochs" 在默认 TSLib（如 10 epoch）时可能仅为 1 个 epoch，且早停可能使"full number of epochs"含义不明。
- **替代解释：** 掩码改变了有效学习率/正则（类似 dropout 式噪声注入），在没有对 masked run 调参的情况下，其收益可能反映"原配方并非最优"，而非 mask 的内容。
- **建议：** 明确验证损失、epoch 数、早停规则；加入"随机 mask 同比例"与"重新缩放学习率"的对照；报告 full-data 在小幅超参调整下的方差，确认 2–5% 的 MAE 变化相对于超参敏感度确实可观。

### W9：数据集/骨干覆盖与规模

- **证据。** 7 个数据集中 4 个是 ETT（高度相关）；缺少 Traffic、Exchange、PEMS、M4 等；没有 PatchTST、基础模型（Chronos、Moirai、TimesFM 等）或大规模预训练场景，而数据剪枝在大规模预训练语料上才最有实际意义。仅 $L=96$ 的回看窗口；ECL 上细粒度单元的 MSE 为正（表1），作者归因于高维但未验证。
- **建议：** 加入至少一个大数据集（Traffic）、不同 $L$（192/336/512），并验证 ECL 现象是否与通道数 $D$ 有关（例如对 ECL 做通道子采样 D=7/21/64/321 的扫描）。

### W10：理论/形式化与符号

- 论文基本没有理论；"element 与 occurrence 的差异"、"为何 window 收益最小"都是经验性解释。至少应形式化：在 MSE 目标下，mask 高残差项等价于对条件分布的截断估计，其偏差/方差如何随 $r$ 变化；这样能同时解释 MAE 收益与 MSE 收益弱，且与 W1 相呼应。
- 式(3)中 $s(t,d)$ 对 occurrence 取平均，但 occurrence 的残差来自不同窗口位置（即不同的预测步长 $t-i$ 与不同输入上下文），早期预测步长与后期预测步长误差尺度不同；分数未按步长归一化，可能使"高端"系统性偏向长步长 occurrence。建议按 horizon 位置分层或标准化后对比。
- 式(2)中 element 的描述 "$c(t,d)$ terms" 在序列两端有边界效应（$c(t,d)<T$），正文已说明但选择 $\lfloor rm+1/2\rfloor$ 的取整与"ties broken by array index"可能引入系统性偏向某些时间位置。

## 5. 领域层面的分析

**研究脉络。** 分类领域的数据剪枝（EL2N/GraNd，Paul et al., 2021；forgetting，Toneva et al., 2019；TDDS，Zhang et al., 2024；InfoBatch，Qin et al., 2024；Sorscher et al., 2022 的"数据多时保留难例"理论）在 TS 中被直接移植，剪枝单元固定为样本/窗口。另一条平行脉络是"loss-level 选择"：Co-teaching（Han et al., 2018）、loss truncation（Kang and Hashimoto, 2020）、Rho-1（Lin et al., 2024）、RHO-Loss（Mindermann et al., 2022），及 TS 里的 Selective Learning 与 RobustTSF。本文实际上站在这两条线的交汇点上，但在叙述中强调前者，实验上与后者的关系（尤其是 trimmed/robust loss）几乎未被检验，这是我认为最大的概念缺口。

**隐藏联系与悖论。**
1. "高残差端 mask 有益而低端无益"与分类中"数据少时保留易例/数据多时保留难例"并不矛盾：这里的训练集相对于一个 TS 骨干属于"数据多但含噪"的体制，高损失项更可能是噪声/异常（类似标签噪声下的 Co-teaching 逻辑）。作者在 §2 触及此点，但没有测试"噪声体制 vs 难例体制"。
2. 悖论：mask 越细越好（window < time column ≈ channel row < element < occurrence），但 occurrence 最优而"element"是论文声称"只有滑窗才有"的特色单元，其表现却不如 occurrence。这意味着，"跨窗口一致地决定一个目标"的卖点（贡献 1）反而在数据上是劣势（element 比 occurrence 少 0.91 点 MAE，§5）。论文对此有说明（高均值残差目标也有被良好预测的 occurrence），但这削弱了"element 是特色单元"的叙事，也与 §7 以 element 为核心的论证不太一致。
3. 评估指标选择的循环：以"MAE/中位数"作为"典型误差"，并据此宣称"gain lives in bulk"，而这类指标本来就对尾部不敏感；"tail 不变"几乎由定义保证。需要用在 held-out 异常/漂移测试段上的指标来真正检验尾部行为。

**研究空白。**（i）非平稳/分布漂移下，静态一次性 mask 的有效性；（ii）窗口重叠导致的 train/val/test 泄漏效应与 mask 的相互作用；（iii）与基础模型预训练数据选择的联系；（iv）mask 对概率预测/分位数输出的影响；（v）高维通道数下的行为（ECL）。

**概念对比表（建议作者补充的维度）。**

| 方法族 | 作用单位 | 分数来源 | 目标 | 是否省算力 |
|---|---|---|---|---|
| EL2N/GraNd/forgetting/TDDS | 样本（window） | 早期误差/梯度/轨迹 | 剪枝 | 是 |
| InfoBatch | 样本 | 损失 + 重缩放 | 无损加速 | 是 |
| Co-teaching/loss truncation | loss 项 | 小损失/截断 | 鲁棒 | 否 |
| Selective Learning | 目标（timestep） | 熵 + 异常 mask | 鲁棒 | 否 |
| 本文 | 5 种单元 | 早期残差 | "剪枝"（实为 loss mask） | 否（§7.3 另议） |

## 6. 给作者的问题

1. L1/Huber 损失或在线 trimmed MSE 训练，相对全数据 MSE 训练能达到多少 MAE 收益？element/channel row mask 在其上是否还有增益？
2. 同样预算的均匀随机 mask 在 5 个单元下分别是多少？为什么主表中不给？
3. 当 $r$ 增至 30–70% 时曲线如何变化，何处开始变差？
4. §4 中验证损失是否也被 mask？"10% of the full number of epochs"对应具体几个 epoch？
5. 被 mask 的元素在时间/通道上的分布如何？是否对应已知异常、solar 的日出日落、ETT 的突变？
6. 为什么 element（跨窗口一致）反而不如 occurrence？这是否意味着"只有滑窗才有的单元"并非关键？
7. Selective Learning 的复现是否使用官方代码和超参？其在原论文数据集上的数字能否复现？
8. 评测是在标准化空间还是原空间？文中没有明确说明。
9. 是否在更长 $L$、更多数据集（Traffic、Exchange、PEMS）、PatchTST 上验证过？
10. 去掉 ECL 后 Friedman 在 MSE 上仍不显著，是否应当调整摘要中"MSE 随单元变化"的措辞？

## 7. 次要问题

- 摘要/引言重复使用"high end/low end"、"decision unit"等自造术语，初读较难；建议在图1处用一句话定义"high end = 残差最高的 10%"。
- "Pruning" 与 "masking" 混用：本文的"剪枝"不减少计算（§3.4），建议在标题和摘要中明确"loss masking"，避免与数据剪枝读者的期望相冲突。
- 图2(a)/图3 缺少误差条/置信区间；图1 的数字很小，可读性有限。
- 表1 的"Datasets better"与粗体"best unit"仅按均值，建议用淡化字体标出统计上与最佳不可区分的单元。
- 表3 的 "cost of the score" 对 Selective Learning 写作"10% + estimator"，估计器成本没有量化，成本比较不完整。
- 参考文献中多项标注为 2026 年的 arXiv 编号（如 TimeLAVA、UNSEEN、PARIS、CLMFormer 等），我无法独立确认其存在与细节，建议作者核对；同时 Okanovic et al. 的 ICLR 年份、Zhu et al. 2024 的会议信息等条目缺少出版场所，宜补全。
- 文中多处交叉引用（Table 7、Section 7.3 等）在 PDF 中显示为红色高亮，说明引用链接可能未正确解析，建议检查。
- 未提供代码/配置/随机种子列表，也未给出各数据集的训练/验证/测试划分；可复现性声明缺失。
- 伦理/更广影响：论文没有相关声明；建议至少说明"mask 高残差样本可能系统性地忽略罕见事件（如极端天气、故障）"所带来的安全风险，特别是对关乎尾部风险的应用。

## 8. TMLR 标准下的评估与总体推荐

**标准 1：主张是否有准确、令人信服、清晰的证据支持？** 部分满足。实验设计在统计推断（以数据集为单位、配对检验、等价性检验）上优于同类工作，描述性发现（高端 vs 低端、误差分布的中部 vs 尾部）证据扎实。但核心叙事——"决策单元是数据剪枝的关键维度"——被 W1（鲁棒/截断损失的替代解释）、W2（缺随机与现有剪枝基线）、W3（仅 10% 预算）所削弱；"MSE 上单元不可区分"以及"推荐 channel row"的依据脆弱；§7 仅在单个骨干上成立。部分主张（"节省 30% 步数"、"优于 Selective Learning"）与统计证据不匹配。

**标准 2：对 TMLR 读者是否有趣？** 是。把滑窗训练的"重复监督"形式化并系统比较单元粒度，以及"只有高端 mask 有益、增益集中在误差分布中部"这一发现，对 TS 训练与数据选择社区有价值；统计方法学上的严谨尝试也值得鼓励。

**总体推荐：Major revision。** 实验工作量大、写作清晰、统计意识强，但需要补充 L1/Huber/trimmed-loss 对照、随机 mask 与经典剪枝基线、预算扫描、更广的数据集/骨干，并收紧结论措辞（特别是 MSE 与 Selective Learning 的比较）。若补充实验表明收益并非可由简单鲁棒损失复现，则论文价值会显著提升；反之，应把论文重新定位为"滑窗训练中的重复监督与鲁棒 loss masking"的研究。

**置信度：** 4/5（我完整阅读了全文，对 TS 预测与数据剪枝文献较熟悉；无法核验 2026 年的若干引用）。

## 9. 参考文献（Harvard 格式，仅列我确信存在者及文中被我引用者）

- Ayed, F. and Hayou, S. (2023) 'Data pruning and neural scaling laws: fundamental limitations of score-based algorithms', *Transactions on Machine Learning Research*.
- Cheng, H., Wen, Q., Liu, Y. and Sun, L. (2024) 'RobustTSF: Towards theory and design of robust time series forecasting with anomalies', *ICLR*.
- Fu, Y. et al. (2025) 'Selective learning for deep time series forecasting', *NeurIPS* (as cited by the submission).
- Han, B. et al. (2018) 'Co-teaching: Robust training of deep neural networks with extremely noisy labels', *NeurIPS*.
- Kang, D. and Hashimoto, T. (2020) 'Improved natural language generation via loss truncation', *ACL*.
- Li, M. et al. (2026) 'Mitigating data redundancy to revitalize Transformer-based long-term time series forecasting system', *ACM TIST* (as cited by the submission).
- Lin, Z. et al. (2024) 'Not all tokens are what you need for pretraining', *NeurIPS*.
- Liu, Y. et al. (2024) 'iTransformer: Inverted Transformers are effective for time series forecasting', *ICLR*.
- Mindermann, S. et al. (2022) 'Prioritized training on points that are learnable, worth learning, and not yet learnt', *ICML*.
- Okanovic, P. et al. (2024) 'Repeated random sampling for minimizing the time-to-accuracy of learning', *ICLR*.
- Paul, M., Ganguli, S. and Dziugaite, G.K. (2021) 'Deep learning on a data diet: Finding important examples early in training', *NeurIPS*.
- Qin, Z. et al. (2024) 'InfoBatch: Lossless training speed up by unbiased dynamic data pruning', *ICLR*.
- Sorscher, B. et al. (2022) 'Beyond neural scaling laws: beating power law scaling via data pruning', *NeurIPS*.
- Toneva, M. et al. (2019) 'An empirical study of example forgetting during deep neural network learning', *ICLR*.
- Wang, S. et al. (2024) 'TimeMixer: Decomposable multiscale mixing for time series forecasting', *ICLR*.
- Xia, X. et al. (2023) 'Moderate coreset: A universal method of data selection for real-world data-efficient deep learning', *ICLR*.
- Zeng, A. et al. (2023) 'Are Transformers effective for time series forecasting?', *AAAI*.
- Zhang, X. et al. (2024) 'Spanning training progress: Temporal dual-depth scoring (TDDS) for enhanced dataset pruning', *CVPR*.
- Zheng, H. et al. (2023) 'Coverage-centric coreset selection for high pruning rates', *ICLR*.
