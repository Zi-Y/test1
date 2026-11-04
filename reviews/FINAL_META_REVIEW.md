# 最终汇总报告（Meta-Review）

**论文**：*Prune the Supervision, Not the Windows: Decision Units for Data Pruning in Time-Series Forecasting*（TMLR 投稿，14 页）
**目标**：在投稿前把 TMLR 的录取概率尽可能提高
**依据**：6 份互不可见的审稿意见（Sonnet / Fable / Opus × medium / high，原文见本目录下 `review_*.md`）。我自己也逐页读了原文，审稿中引用的论文数字和位置都对照原文核实过。
**去重原则**：同一模型的 medium 和 high 两份意见高度相关（见 §1.5），所以**共识一律按"模型家族"计数（x/3）**，评审份数（y/6）只作参考。有效的独立意见约为 3 份，而不是 6 份。
**配套文件**：`EXPERIMENT_PLAN.md`（按优先级排序的补充实验计划）

> 记号说明：**[已核实]** 表示我在原文中找到了对应内容；**[部分属实]** 表示方向正确但细节需要修正；**[有误]** 表示审稿人说错了；**[推断]** 表示合理推论，作者需用实验确认。

---

## 0. 执行摘要

1. **3 个模型家族（6 份评审）全部给出 Major revision，置信度都是 4/5。** 同一家族的两份意见要算作约一份独立意见，所以这是"3 个独立来源一致"，而不是"6 票一致"。即便如此，3 个不同模型在互不可见的情况下都把同一个问题排在第一位，这个信号仍然很强。而且这个问题**并不在我给审稿人的提示词里**（见 §1.5），说明它是审稿人自己发现的，而不是被提示词引导出来的。真实的 TMLR 审稿人几乎一定会提出它。

2. **决定录取与否的问题只有一个**：论文的主现象是"屏蔽残差最高的 10% 损失项 → MAE、中位数误差显著下降，MSE 和尾部基本不变"。这与*鲁棒损失 / 截断损失（trimmed loss）*的典型特征几乎一样，而论文**没有任何 L1、Huber 或在线截断的对照**。3 个家族的全部 6 份评审都把它列为第一或第二关键问题。如果不补这组对照，"决策单元重要"这一核心因果主张就无法与"任何鲁棒化都有效"区分开，直接违反 TMLR 的第一条标准（claims 要有证据支撑）。

3. **第二梯队有三个问题**，每个都被 3/3 家族独立提出：
   - (i) 单元比较被"移除的残差质量 / 选择的尖锐度"混杂，且缺少每个单元的**同预算随机屏蔽**对照；
   - (ii) 只测了 r=10% 一个预算；
   - (iii) 统计报告不完整：没有种子方差、没有绝对数值、没有交代被剔除的发散运行、ETT 数据集之间不独立、尾部"无变化"没有做等价检验。

4. **TMLR 流程带来的策略含义**：TMLR 的最终决定只有 Accept / Accept with minor revision / Reject 三种，没有正式的"Major revision"。讨论期允许上传修订稿，但时间短（约两周）。6 个 Major revision 落到实际结果上，**最可能是 Reject**，除非作者在讨论期内补完关键实验。**所以 P0 实验必须在投稿前完成**，不能留到 rebuttal。

5. **好消息**：
   - 所有审稿人都认可论文**满足 TMLR 第二条标准**（对读者有兴趣）。
   - 所有审稿人都核对了表格数字，**没有发现任何计算错误**，我自己的复核结果也一样。
   - 所有审稿人都称赞统计推断意识高于领域平均水平。
   - 多数问题可以通过**改写措辞**（零成本）或**利用已有运行做分析**（几乎零成本）来解决。
   - 即使鲁棒损失对照的结果不利，论文也有可行的重新定位路线（见 §6.2）。

6. **最重要的五项行动**：
   - ① 加入 L1、Huber、在线截断、mask+L1 对照（E1）；
   - ② 为每个单元加入随机屏蔽对照（E2）；
   - ③ 利用已有运行补齐方差、绝对值、残差质量、掩码结构和分 horizon 结果（E0，零训练成本）；
   - ④ 做预算扫描并按残差质量匹配比较（E3/E4）；
   - ⑤ 按 §5 的措辞审计表收窄或改写约 15 处表述，并把论文定位从"data pruning（省算力）"调整为"supervision masking"。

---

## 1. 方法：如何得到这份汇总

- 6 个审稿代理各自独立读 PDF，各自只写自己的文件，彼此不可见。
- 我完整读过原文后，**逐条核对**了审稿意见中的每个数字、位置和事实性断言（例如表 1 均值、表 5 中 DLinear 窗口的 MSE 为 3/7、表 6 中 ECL element 的 MAE 为 +0.39、§7.1 的随机掩码结果等）。
- 每个问题都按以下标准评估：
  - (a) 是否属实；
  - (b) 有几位审稿人独立提出（共识度）；
  - (c) **如果不解决，对 TMLR 录取的影响**；
  - (d) 解决成本：仅改措辞 / 用已有数据分析 / 需要新实验。

### 1.5 去重：同一模型两份评审的相关性，以及提示词带来的偏差

**(1) 同家族的两份评审有多相似？**（我逐条对比了两份的"缺点"清单）

| 家族 | 重合程度 | 证据 | high 比 medium 多出的独有内容 | medium 比 high 多出的独有内容 |
|---|---|---|---|---|
| Opus | **约 80%，最高** | W1（鲁棒损失）和 W2（残差质量混杂）在两份中几乎逐字对应，前三条的排序也一样 | 无法核实的 "every dataset of every backbone"；掩码在种子间共享；2×2 输入/目标分解；CCS/Moderate 的连续谱改写；附录 D 措辞 | **lead time 混杂**（最清楚的版本）；Sorscher 数据稀缺区的解释；内存估算；验证集是否屏蔽 |
| Fable | **约 75%** | 第一条（鲁棒损失）相同；预算、§7.1 同义反复、SL、§7.3、掩码分布都在两份中出现 | **窗口得分自相关 → 窗口掩码相当于整段删除**；每个 horizon、每个通道的预算上限；最佳 epoch 分布；修复对照的输入污染 | 离群覆盖率诊断及其预测排序；实例归一化下的残差尺度；"屏蔽最后 10%"漂移对照；"never" 的措辞 |
| Sonnet | **约 70%，最低** | 鲁棒损失、随机掩码、预算、channel row、SL、§7.3 两份都有；但 **SH 把"残差质量混杂"列为第一条，SM 完全没有提** | 残差质量混杂；**在 MSE 下 low end 无效几乎是必然的**；平凡得分基线；"得分 = 回归版 EL2N"；ECL 与 solar 维度的矛盾 | ECL 通道子采样；按步长对得分做标准化；尾部风险伦理声明 |

**(2) 去重后的计数规则**
- **家族共识 = 至少有一份该家族的评审提出**，取值 0–3。分级主要看家族数：3/3 为强信号，2/3 为中等信号，1/3 主要依靠我自己的核实来判断。
- 同家族两份都提出的问题**只算一次**。因此，一条看起来是"6/6"的问题，实际证据强度与"3/3"相同；"4/6"也可能已经是 3/3 家族（例如 lead time），而"2/6"可能只来自一个家族（例如按 horizon 拆分只来自 Opus）。
- 只来自单一家族的意见，我不会因为票数少就降级，而是**用原文核实结果决定**。例如"按 horizon 拆分"只有 Opus 提出，但它直接检验论文的核心论点，而且成本为零，所以保留在 A 级。

**(3) 跨家族的共识也不完全独立：提示词偏差**
- 我给 6 个审稿代理的提示词里**点名提到**了 EL2N、GraNd、forgetting、CCS、Moderate、TDDS、InfoBatch，以及"滑窗样本重叠、非平稳性、分布漂移"。所以以下共识**有一部分是提示词带出来的**，作为"真实审稿人会提出"的证据要打折扣：#16（掩码分布、非平稳）、#26（相关工作缺 CCS、Moderate 等）。不过这些问题我在原文中都核实属实，结论不变，只是降低了它们作为"必然被提出"的预测强度。
- 以下共识**不在提示词中**，是 3 个家族各自独立发现的，**可信度最高**：#1 鲁棒损失替代解释、#3 残差质量混杂、#13 §7.1 同义反复、#11 尾部需要等价检验、#14 SL 被改造、#5 §7.3 缺证据、#12 channel row 推荐依据不足、#17 lead time 混杂。这些也正是 S 级和 A 级的核心。
- 6 个代理都是 Claude 系列模型，训练来源相近，可能有共同的盲区。所以我额外补充了自己的独立判断（§7）。
- 最终分为四类：**S 必须解决（否则大概率被拒）**、**A 应该解决（会显著影响审稿人判断）**、**B 建议解决（加分项）**、**X 可以忽略或降级**。

---

## 2. 共识矩阵（问题 × 审稿人）

缩写：SM=Sonnet·medium，SH=Sonnet·high，FM=Fable·medium，FH=Fable·high，OM=Opus·medium，OH=Opus·high。✓ 表示提出，★ 表示该审稿人把它列为最关键问题。

| # | 问题 | SM | SH | FM | FH | OM | OH | 家族共识（评审数） | 核实 | 等级 |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 缺 L1/Huber/在线截断对照，收益可能是鲁棒损失效应 | ★ | ★ | ★ | ★ | ★ | ★ | **3/3**（6/6） | 已核实（全文无任何损失函数对照） | **S** |
| 2 | 缺每单元的同预算随机屏蔽；low end 作为对照太弱 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（6/6） | 已核实（随机对照只在 §7.1 iTransformer×element 做过） | **S** |
| 3 | 单元效应与"移除的残差质量 / 选择尖锐度"混杂 | – | ★ | ✓ | ✓ | ✓ | ✓ | **3/3**（5/6） | 已核实（预算按项数而非残差质量统一，§3.3） | **S** |
| 4 | "data pruning"定位与"不省算力"不符 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（6/6） | 已核实（§3.4 "does not shorten an epoch"） | **S**（靠改写解决） |
| 5 | §7.3 无表无图，"three rules"未定义 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（6/6） | 已核实 | **S**（证据缺失） |
| 6 | 只有 r=10% 一个预算 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（6/6） | 已核实 | **A** |
| 7 | 无种子方差、CI、绝对数值 | ✓ | ✓ | ★ | ✓ | ✓ | ✓ | **3/3**（6/6） | 已核实 | **A** |
| 8 | 被剔除的发散运行没有计数 | – | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（5/6） | 已核实（§4 "Runs that diverge are excluded"） | **A** |
| 9 | ETTh1/ETTm1、ETTh2/ETTm2 同源，有效 n≈5 | 部分属实 | ✓ | ✓ | ✓ | ✓ | – | **3/3**（5/6） | 已核实（SM 说"四个 ETT 同源"有误，实际是 2 个源） | **A** |
| 10 | 多重比较未校正，多个关键 p 落在 0.02–0.05 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（6/6） | 已核实（p=0.045/0.040/0.036/0.027） | **A** |
| 11 | "最差 10% 无变化"只是不显著，没有等价检验 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（6/6） | 已核实（low end 做了等价检验，CVaR 没做；occurrence 的 CVaR 为 +0.47，3/7） | **A** |
| 12 | 推荐 channel row 依据不显著的差异（p=0.525） | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（6/6） | 已核实 | **A**（靠改写解决） |
| 13 | §7.1 的 Spearman 0.998 近乎同义反复 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（6/6） | 已核实（"read at the same early point of training"） | **A** |
| 14 | Selective Learning 被强制校准到 10%、投影到 element，公平性存疑 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（6/6） | 已核实（附录 B） | **A** |
| 15 | 插值修复同时改动了输入，对照不干净 | – | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（5/6） | 已核实（"an interpolated value enters both"） | **A** |
| 16 | 被屏蔽项在时间、通道、时段上的分布没有展示（非平稳、漂移、异常段） | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（6/6） | 已核实 | **A** |
| 17 | occurrence 掩码可能集中在远步长位置（lead time 混杂） | ✓ | – | – | ✓ | ★ | ✓ | **3/3**（4/6） | 推断，但机理可信 | **A** |
| 18 | 没有按 horizon 拆分结果 | – | – | – | – | ✓ | ✓ | **1/3**（2/6） | 已核实 | **A**（成本几乎为零） |
| 19 | 实现细节：epoch 数、式 4 按 batch 还是全局归一化、验证集是否屏蔽、残差尺度 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（6/6） | 已核实（均未写明） | **A**（靠改写解决） |
| 20 | §7 只用 iTransformer，排除 ECL/solar 未说明理由，"nine combinations"未定义 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（6/6） | 已核实 | **B+** |
| 21 | 打分时刻只有一个（10% epoch） | – | ✓ | ✓ | ✓ | ✓ | – | **3/3**（4/6） | 已核实 | **B+** |
| 22 | 缺平凡得分基线（幅值、局部方差、季节朴素残差） | – | ✓ | – | ✓ | – | – | **2/3**（2/6） | — | **B** |
| 23 | 合成或注入异常的受控实验 | ✓ | ✓ | – | – | ✓ | ✓ | **2/3**（4/6） | — | **B+**（机理价值高） |
| 24 | MC-dropout 显著更好，与"cheap score"的表述有张力 | ✓ | – | ✓ | – | – | ✓ | **3/3**（3/6） | 已核实（p=0.045） | **B**（靠改写解决） |
| 25 | 缺 PatchTST、Traffic、L≠96 | ✓ | ✓ | – | ✓ | ✓ | ✓ | **3/3**（5/6） | 已核实 | **B** |
| 26 | 相关工作缺鲁棒回归、LTS、CCS、Moderate；"我们的得分 = 回归版 EL2N" | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（6/6） | 已核实（CCS、Moderate 未引用） | **A**（靠改写解决） |
| 27 | 与分类文献"极性相反"的叙述有误导性（Sorscher 的数据稀缺区、CCS/Moderate） | – | ✓ | ✓ | ✓ | ✓ | ✓ | **3/3**（5/6） | 已核实（§2 "the opposite end pays"） | **A**（靠改写解决） |
| 28 | 掩码在 5 个种子间共享，不同打分种子下掩码是否稳定 | – | ✓ | – | – | – | ✓ | **2/3**（2/6） | 已核实（附录 A：每个配置"one ordinary full-data run"） | **B** |
| 29 | occurrence 打分的存储开销（ECL、T=720 时约 4e9 项，约 16 GB） | – | – | – | ✓ | ✓ | – | **2/3**（2/6） | 推断，算术正确 | **B**（加一句说明即可） |
| 30 | 软重加权（逆方差）对照 | – | ✓ | – | ✓ | – | ✓ | **3/3**（3/6） | — | **B−** |

**结论**（按家族计数）：S 级和 A 级问题除 #18 外都是 **3/3 家族**，说明它们不依赖某一个模型的偏好，而是论文本身的结构性缺口。#18（按 horizon 拆分）只来自 Opus 一个家族，保留在 A 级是因为我核实它直接检验核心论点、且成本为零。#22、#23、#28、#29 是 2/3 家族，所以放在 B 级。
注意：表中 "3/3（3/6）" 和 "3/3（6/6）" 的证据强度**基本相同**，因为差别只在同家族的另一档是否重复提出，而这种重复带来的独立信息很少。

---

## 3. 逐位审稿人的质量评估（哪份意见最值得信）

| 审稿人 | 长度/深度 | 独有且有价值的贡献 | 事实错误或弱论点 | 总体可用度 |
|---|---|---|---|---|
| **Opus·high** | 很深，最完整、最精确 | ① 指出 §5 "occurrence is ahead on every dataset of every backbone" **无法核实**，且 iTransformer 上 element 的 MAE 实际优于 occurrence（−3.69 vs −3.67，表 4）[已核实]；② 掩码在种子间共享，方差被低估；③ 用 CCS/Moderate 把"极性相反"改写为"连续谱"；④ 目标×输入的 2×2 分解；⑤ 按异常结构匹配单元结构的注入实验；⑥ DLinear 没有 dropout，MC-dropout 无法推广 [已核实]；⑦ 附录 D "fall at every unit"措辞过强（window 的 q90 只有 5/6）[已核实]；⑧ TimeMixer 文献缺 arXiv 号 [已核实]；⑨ RHO-Loss 式得分 | 无实质错误 | ★★★★★ |
| **Opus·medium** | 深 | ① **lead time 混杂**讲得最清楚，并给出可检验的对照；② Sorscher 数据稀缺区的重新解释，以及训练集截断实验；③ 内存估算；④ 验证集是否屏蔽；⑤ 数值复核完整（例如 ECL element MSE=(0.44+1.47+3.35)/3=1.75）[已核实] | 无实质错误 | ★★★★★ |
| **Fable·high** | 很深 | ① **窗口得分高度自相关，窗口掩码相当于整段删除**，从而把"覆盖度"引入时间序列剪枝，这是最有洞察力的机制解释；② 每个 horizon、每个通道设预算上限的对照；③ element 劣于 occurrence 说明"是否值得学"取决于上下文，与"按目标排序"的叙事冲突；④ 屏蔽是否只是改变了早停时机（最佳 epoch 分布）；⑤ 用 Exchange 做压力测试 | 小问题：说 §7.1 随机掩码的 MAE 没给，其实可以推出（见 §7） | ★★★★★ |
| **Fable·medium** | 中深 | ① "离群覆盖率"诊断，并预测了单元排序（可证伪）；② **残差在实例归一化下的尺度问题**；③ 用"屏蔽训练集最后 10%"做漂移对照；④ 在未训练模型上打分（epoch 0）的对照 | 参考文献里 Selective Learning 的标题写错（审稿人自身的错误，不影响论文） | ★★★★ |
| **Sonnet·high** | 最长（47 KB），覆盖广 | ① **在 MSE 下 low end 无效几乎是必然的**（低残差项梯度小），所以 low end 不能证明"收益来自屏蔽了哪些项、而不是屏蔽本身"，这一逻辑点很关键；② 平凡得分基线；③ "我们的得分 = 回归版 EL2N"；④ 窗口列并不是"已有方法"；⑤ "ECL 是唯一高维数据集"与 solar 有 137 个通道矛盾 [已核实]；⑥ DLinear 窗口 MSE 只有 3/7 [已核实] | 推测性论点：MAE 是事后选的（无证据）；solar 平局按数组下标打破会带来偏差（浮点数几乎不会平局，可忽略）；"low end 才是真正的剪枝方向"（掩码本身不省算力，逻辑不成立） | ★★★★ |
| **Sonnet·medium** | 中 | ① 对 ECL 做通道子采样，检验 D 的作用；② 按步长对得分做标准化；③ 尾部风险的伦理声明 | **若干事实错误**：说"四个 ETT 同源"（实际是 2 台变压器，2 个源）；说 PDF 红框表示"引用未解析"（那是 hyperref 默认的链接框，不是错误）；说 Okanovic 缺年份（文中写了 ICLR 2024）；说表 1 中 ECL 的 MSE 最佳单元是 occurrence（实际是 window，−0.64）；"尾部不变由定义保证"（论证错误） | ★★★ |

**附带观察（去重之后）**：
- 每个家族的两份意见约 70–80% 重合（§1.5）。真正有增量的是每份的"独有内容"，上表只列这部分。
- **按家族看信息贡献**：Opus 家族贡献最大（lead time、残差质量、可核实性问题、2×2 分解）；Fable 家族贡献了最有洞察力的机制解释（窗口自相关 → 段删除；离群覆盖率）；Sonnet 家族的独有贡献集中在 Sonnet·high（low end 必然无效的逻辑、得分 = EL2N），而 Sonnet·medium 的独有内容最少、事实错误最多。
- 下面的分级只采信经过原文核实的意见，共识度按家族计算，不按评审份数计算。

---

## 4. 问题清单（按对 TMLR 录取的影响排序）

每条的格式是：**问题 → 原文证据 → 为什么影响录取 → 应对方式（改写 R# / 分析 E0 / 实验 E#）**。实验编号对应 `EXPERIMENT_PLAN.md`。

### S 级：不解决，大概率被拒

#### S1. 鲁棒损失 / 截断损失这一替代解释（3/3 家族，均列为最关键）

- **原文证据** [已核实]：
  - 训练损失是 MSE（式 1/4）。收益集中在 median |e|（−4.42% 至 −12.45%，表 2），q95 时已经消失；MAE 显著，MSE 不显著（p=0.06–0.13）。
  - solar 上 occurrence 的 MAE 为 −8.81%，MSE 却是 **+0.91%**；time column 的 MAE 为 −7.83%，MSE 为 +0.66%（表 1）。
  - 收益最大的是线性模型 DLinear 在 ETTh2/ETTm2 上（表 5，MSE −8.7% 至 −17.8%）。
  - loss truncation（Kang & Hashimoto 2020）和 Co-teaching 在 §2 有引用，但没有作为基线比较。
- **我对审稿意见的修正（审稿人的说法过强）**：有几位审稿人说"均值→中位数偏移可以**完全**解释"，这句话过头了。鲁棒统计至少给出两种机制：
  - (i) **降低离群点的影响**：在重尾或被污染的噪声下降低拟合方差，这在对称噪声下也成立；
  - (ii) **位置偏移**：只在条件分布有偏时出现，solar 很可能属于这一类。

  两种机制都预测"MAE 和中位数收益大于 MSE 收益"，但都没有被检验过。所以准确的说法是：**论文现有证据无法把"单元或分数的特异性"与"一般的鲁棒化"区分开**，而不是"已经被鲁棒化解释掉了"。rebuttal 中应采用这种更精确的说法。
- **为什么影响录取**：TMLR 第一条标准就是 claims 要有证据支撑。论文的因果主张有三个："决策单元重要"、"收益来自屏蔽了哪些项"、"分数选出了无法解释的目标"。只要缺少这组对照，这三个主张都不成立。
- **应对**：E1（L1、Huber、在线截断 occurrence 级和 window 级、mask+L1、mask+Huber）。
- **潜在的反转机会（审稿人没有点透）**：L1/Huber 训练通常会**牺牲 MSE**来换 MAE，而论文的掩码在 MSE 上平均也是改善的（−1.7% 至 −2.8%）。如果 E1 显示"L1 的 MAE 收益相近但 MSE 变差，而掩码两项都改善"，这就会成为论文**最有力的卖点**："保持 MSE 目标、却拿到接近 L1 的 MAE 收益"。另外，如果固定的早期掩码优于在线截断，就可以得出新的正面结论："一次性早期决定优于逐批截断"。在线截断在训练早期会误删难但可学的项，所以这个结果有可能出现。详见 §6.2 的决策树。

#### S2. "单元效应"被残差质量和选择尖锐度混杂，且缺少随机对照（3/3 家族）

- **原文证据** [已核实]：
  - §3.3 写 "every unit removes about the same share r of the loss terms"，预算按**项数**统一。在同样 10% 的项数下，occurrence 的 top-10% 按构造就是移除平方残差总量最大的那组项；更粗的单元按均值打分，会混入大量正常项。所以 "window < … < occurrence" 的排序**很大程度上是构造使然**。
  - §5 用 low end 来论证 "the gain … comes from which loss terms are masked, not from masking as such"。但正如 Sonnet·high 所指出的，在 MSE 训练下，低残差项本来对梯度贡献就很小，**low end 无效几乎是必然的**，这个论证不成立，**必须用随机对照**。
- **已有证据（审稿人没有充分利用，作者可以立刻使用）**：§7.1 的匹配随机掩码在 iTransformer 上的 MSE 为 −0.16%。从 "element mask is ahead … by 2.66 points of MSE and 3.59 points of MAE" 可以反推出它的 **MAE 约为 −0.38%**（−3.97+3.59）。这说明在 iTransformer 的 element 单元上，随机屏蔽几乎没有收益，是有利证据，但只覆盖了 1 个骨干 × 1 个单元。
- **Fable·high 的补充机制**：stride=1 时相邻窗口共享 T−1 个目标，窗口得分高度自相关，top-10% 的窗口会聚成少数连续段，**窗口掩码实际上是"删除整段序列"**。这为"窗口收益最小"提供了一个比"窗口是错误的抽象"更具体、更可检验的解释，作者可以主动把它写成论文的机制分析，化被动为主动。
- **应对**：E0 中的残差质量计算（零训练成本）、E2（每单元随机掩码）、E3/E4（预算扫描 + 按质量匹配），以及在 E0 中报告窗口掩码的连通段长度。
- **结果怎么写**：
  - 如果各单元在"收益 vs 移除的残差质量"图上**落在同一条曲线**，就把主张改写为"在滑动窗口中，单元越细，越能精确定位高残差监督；窗口因重叠而定位精度最差"。这仍然是有价值的结论，并且与标题一致。
  - 如果**不落在同一条曲线**（例如 element 在相同质量下更好），那么"单元本身重要"就得到了强证据。

#### S3. 定位与证据不一致："Data Pruning" 但不省算力；§7.3 没有证据（3/3 家族）

- **原文证据** [已核实]：
  - §3.4 写 "it does not shorten an epoch"，并且还额外需要一次打分运行。
  - §7.3 的所有数字只出现在正文里，没有表也没有图；"three rules that select windows by their score" 和 Tukey 自适应预算的实际屏蔽比例都没有报告。
  - 标题是 "Prune the Supervision, **Not the Windows**"，但 §7.3 的省算力建议恰恰是**删窗口**（Fable·high 指出的张力）。
- **为什么影响录取**：这是"claims 与证据不符"中最容易被一眼看出的一类。审稿人读到标题会期待效率收益，实际拿到的是精度收益，第一印象会受损。
- **应对**：
  - R1：重新定位（见 §6.1 的标题和摘要方案）；
  - E7：为 §7.3 补完整表格，定义三条规则，报告 wall-clock 和总成本（包括打分）；
  - 否则就把 §7.3 降为附录中的探索性结果，并删去相关的强表述。

### A 级：会显著影响审稿人判断

#### A1. 统计报告的完整性（3/3 家族）

| 子问题 | 原文 | 应对 | 成本 |
|---|---|---|---|
| 没有种子方差、CI、绝对数值 | 全部 7 张表只有相对变化的均值 | 附录给出所有配置的绝对 MSE/MAE ± 种子标准差，并与 TSLib 或原论文的公开数值对照 | E0（零训练） |
| 发散运行未计数 | §4 | 按配置列出发散计数表，并做敏感性分析（发散运行按最差值计入） | E0 |
| ETT 同源 | §4 把 7 个数据集当作独立 | 合并 ETTh1+ETTm1、ETTh2+ETTm2 后重做主检验（n=5），或做 block bootstrap | E0 |
| 多重比较 | §4 声明不校正 | 预先指定"主张族"，对主张族做 Holm 校正，正文标出校正后仍成立的结论；单元间两两比较用 Wilcoxon+Holm（不要用 Nemenyi，见 Benavoli et al. 2016） | E0 |
| 尾部"无变化" | §6；摘要 "does not change measurably" | 对 CVaR@90%、q95、MSE 做与 low end 相同的等价检验（TOST），并说明 ±δ 的来源（例如用全数据跨种子标准差作为尺度） | E0 |
| 百分比均值被 DLinear-ETTh2 等极端值主导 | 表 5 | 同时报告中位数、log-ratio，或做 Wilcoxon 检验 | E0 |
| 掩码在 5 个种子间共享 | 附录 A "one ordinary full-data run" | 用 3 个不同的打分种子生成掩码，报告 Jaccard 和下游方差 | E12 |

#### A2. 只有一个预算 r=10%（3/3 家族）

- 数据剪枝文献的核心教训是：方法排序会随剪枝率翻转（Sorscher et al. 2022；Ayed & Hayou 2023；Zheng et al. 2023）。单点结论很容易被质疑。
- **应对**：E3（预算扫描）。另外要在主文报告 Tukey 自适应预算实际屏蔽了多少比例（数据很可能已经有）。

#### A3. 时间序列特有的结构混杂：lead time、时间位置、通道集中、分 horizon（3/3 家族）

- **lead time**（OM、FH、SM、OH）：远步长的残差系统性更大，所以 occurrence 和 time column 的 high end 可能集中在远步长，**掩码实际上近似于"降低远步长的权重"**，也就是 Hounie et al. (2024) 的 loss shaping（论文已引用）。这也能顺带解释为什么 element 不如 occurrence：element 在所有步长上取平均，丢掉了这层信息。**这是对论文核心叙事威胁最大的时间序列特有混杂。**
- **时间位置 / 漂移**：如果被屏蔽的 element 集中在训练段末尾（最接近测试期）或某个 regime，那么机制就是"去掉某个 regime"，而不是"去掉无法解释的噪声"。
- **通道集中**：在 ECL 这种 321 通道的数据上，channel row 或 element 掩码可能集中在少数高方差通道，这与表 1 中 ECL 的 MSE 上升（+0.05 至 +1.75）一致 [推断]。
- **分 horizon**：论文的核心论点是"每个目标被监督 T 次"，但所有结果都把 4 个 horizon 平均在一起。按 horizon 拆分是**对这一论点最直接的检验**，而且只需要重新读一遍日志。
- **应对**：E0（掩码结构直方图、分 horizon 表）、E5（按 lead time / 通道设预算上限的掩码、无得分的远步长降权、"屏蔽最后 10% 训练段"对照）。

#### A4. §7.1 "得分 = 未解释误差" 近乎同义反复（3/3 家族）

- **原文** [已核实]："read at the same early point of training as the score"。训练到 10% 时，单个目标几乎还没有被"记住"，所以折内残差和折外残差几乎一样，Spearman 0.998 是可以预期的。而 explained error 的相关只有 0.22，这恰恰说明此时还谈不上"解释"。作者自己写了 "neither says that a target is unpredictable in principle"，但摘要和结论的措辞远比这强。
- **应对**：
  - R：把摘要和结论改写为"与训练早期的折外残差高度一致"；
  - 给出 explained 和 unexplained 的公式定义；
  - 说明"nine combinations"具体是哪九个；
  - E8：在收敛点重做分解；
  - E11：平凡得分（幅值、季节朴素残差）与本文得分的相关性。

#### A5. 对比的公平性：Selective Learning 和插值修复（3/3 家族）

- **SL**（附录 B）：被投影到 element 单元，名义比例被校准到 10%，与原方法设置不同；MSE 上差距不显著（p=0.07/0.46）。摘要只写了 MAE 上的优势。→ R：摘要改为 "at a matched 10% budget, in MAE; MSE differences n.s."；E9：用官方代码、官方设置跑一遍，并与 Fu et al. (2025) 报告的数值对照。
- **插值**：同时改动了输入和目标，而测试时输入仍包含原始的大残差点。→ E10：只修复目标、保留原输入的变体（2×2 分解：目标保留或屏蔽 × 输入原值或插值）。

#### A6. 默认推荐 channel row 的依据不足（3/3 家族）

- 原文自己的 Friedman 检验 p=0.525（MSE），channel row 在 ECL 上的 MSE 为 +0.05；在 iTransformer 上 MSE 最好的是 element（−2.33，表 4）。§7 的机理分析全部针对 element，而不是推荐的 channel row。
- **应对（零成本）**：把推荐改为条件式。关心 MAE 或典型误差时用 occurrence；希望实现简单、尾部稳定时用 channel row；并明确写出"MSE 上没有任何单元有显著优势"。如果 E1–E5 之后 channel row 在 MSE 或 CVaR 上有显著优势，再恢复为默认推荐。

#### A7. 相关工作与概念定位（3/3 家族，零成本）

- 补充以下文献脉络：鲁棒回归（Huber 1964；Rousseeuw 1984 的 LTS）、迭代截断损失（Shen & Sanghavi 2019）、self-paced learning、CCS（Zheng et al. 2023）、Moderate coreset（Xia et al. 2023）、RHO-Loss 的可学习性视角（已引用）、Hounie et al. 2024 的步长加权关系（已引用，但要讨论）。
- **主动承认"本文得分是 EL2N 在回归中的对应物"**（Sonnet·high）。主动说出来比被审稿人指出要好得多。
- **重写 "In our setting the opposite end pays"**：这不是与分类文献对立，而是落在一个连续谱上。在低剪枝率、含噪回归、MSE 损失的条件下，剔除最难项正好对应 Sorscher et al. (2022) 的数据稀缺或含噪区，以及 CCS/Moderate 的"先去掉最难的一部分"。
- Opus·medium 建议的可选实验：截断训练集，检验最优端是否翻转（E16，P3）。

#### A8. 可复现性细节（3/3 家族，零成本，但缺了会被扣分）

需要写明：
- 每个骨干 × 数据集的 epoch 数、patience、学习率调度、batch size；
- "10% of full epochs" 具体对应第几个 epoch（多位审稿人推断 TSLib 默认 10 个 epoch，即第 1 个 epoch 结束时，需要确认）；
- **式 4 的分母在实现中是 batch 内的 Σw 还是全局常数**；
- 验证集损失是否屏蔽；
- 残差在哪个尺度上计算（数据集标准化尺度，还是 RevIN 实例归一化之后）；
- TSLib 的 commit，测试 loader 是否 drop_last（Qiu et al. 2024 的 TFB 指出过这个问题）；
- 匿名代码链接；
- 算法框和超参数表。

### B 级：加分项，有余力就做

| # | 内容 | 来源 | 应对 |
|---|---|---|---|
| B1 | 合成或注入异常的受控实验：异常结构（点、单通道段、跨通道同一时刻、水平漂移）× 单元结构；同时注入"罕见但可学"的模式 | OH、OM、SH、SM | E6（机理价值高、成本低，**建议提到 P1**） |
| B2 | 打分时刻扫描（epoch 0 / 10% / 30% / 收敛） | FM、FH、OM、SH | E8 |
| B3 | 平凡得分基线 | SH、FH | E11 |
| B4 | PatchTST、L=336、Traffic | 3/3 家族 | E13（PatchTST 和 L=336 为 P2，Traffic 为 P3） |
| B5 | 按通道标准化的得分；ECL 通道子采样（D=7/21/64/321） | OH、SM | E14 |
| B6 | MC-dropout 措辞："comes close to scores that cost far more" 应改为 "MC-dropout 在 MSE 上小幅但显著更好（p=0.045），只多 5 次前向；但它不适用于没有 dropout 的 DLinear" | OH、FM | R（零成本） |
| B7 | 存储开销：occurrence 打分在 ECL、T=720 时约 4×10⁹ 项 | OM、FH | 加一句说明流式 top-k 或分块实现 |
| B8 | 最佳 epoch 分布：屏蔽是否只是改变了早停时机 | FH | E0（零成本） |
| B9 | 加 Limitations 小节，以及尾部风险的 Broader Impact（屏蔽高残差可能削弱对罕见极端事件的学习） | SM、OH | R（零成本） |
| B10 | 软重加权（逆方差或按残差分位数连续降权） | SH、FH、OH | E1 的可选附加项 |

### C 级：次要修正（零成本，全部照改）

1. 记号冲突：$r_i(t,d)$ 表示残差，$r$ 又表示预算。建议把预算改为 $\rho$ 或 $b$（OH）。
2. $c(t,d)$ 与 $d$ 无关，可以记为 $c(t)$；给出 $n=N-L-T+1$；前 $L$ 步不作为目标（OH、FH、OM）。
3. 说明 element 单元"about the same share"的实际份额，因为序列两端的 $c<T$。
4. "ECL is the only high-dimensional dataset" 与 solar 有 137 个通道矛盾，应改为"通道数最多"（SH）[已核实]。
5. 附录 D "fall at every unit" 应改为 "fall on average at every unit"（window 的 q90 只有 5/6）（OH）[已核实]。
6. §5 "the occurrence is ahead on every dataset of every backbone" 需要补一张分骨干的 median |e| 表来支撑（OH）。
7. 表 6 中 ECL 的 MSE 那一格把 +0.02 加粗为最佳，建议注明"没有单元改善"（OH）。
8. TimeMixer 参考文献缺 arXiv 号（OH）[已核实]。
9. "points"（相对百分比之差）与 "%" 容易混淆，应在每个表注中重复定义。
10. 摘要中数字太多，压缩到 3–4 个核心发现（3/3 家族）。
11. 图 1 只画了 MAE，标题却提到 MSE，应补一个 MSE 子图或误差棒。
12. 注明 1/64 是双侧值（FH）。
13. §7.3 中 "+0.30 points" 与 "within ±0.3 points" 是两个不同的比较，要写清楚各自的比较对象（OH）。
14. 摘要中 "dataset as the unit of inference" 对一般读者不好懂，应改写。
15. "RobustTSF residual, which needs no model" 应改为 "needs no trained model"（FH）。

---

## 5. 可以忽略或降级的建议（附理由）

| 建议 | 来源 | 判断 | 理由 |
|---|---|---|---|
| PDF 中的红框交叉引用说明"引用未解析" | SM | **忽略（有误）** | 那是 hyperref 默认的链接边框，链接本身正常 |
| Okanovic et al. 缺年份或会议 | SM | **忽略（有误）** | 参考文献已写明 ICLR 2024 |
| "四个 ETT 数据集是同一来源" | SM | **修正后采纳** | 实际是 2 台变压器、2 个源，有效 n≈5，而不是 4 合 1 |
| "尾部不变由定义保证" | SM | **忽略（论证错误）** | MAE 对尾部不敏感，并不意味着尾部指标本身不会变 |
| 表 1 中 ECL 的 MSE 最佳单元是 occurrence | SM | **忽略（有误）** | 实际是 window（−0.64） |
| "MAE 是在看到结果后才选的" | SH | **忽略这一指控，采纳它背后的建议** | 没有证据；但应在摘要中给 MSE 与 MAE 同等的显著位置（零成本） |
| solar 的平局按数组下标打破会产生偏差 | SH | **忽略** | 浮点残差几乎不会出现平局 |
| "low end 近乎无损，所以它才是真正的剪枝方向" | SH | **大部分忽略** | 掩码不省算力，这个逻辑不成立。只有把 low end 做成真正的窗口删除时才有意义，这部分已并入可选的 E15 |
| Nemenyi 事后检验 | SH | **不采纳** | Benavoli et al. (2016) 指出基于平均秩的事后检验有问题，应改用两两 Wilcoxon + Holm |
| 广泛调参全数据基线 | SH | **降级** | 成本高，也不是领域惯例。用"与 TSLib 公开数值对照的绝对值表"来回应基线偏弱的质疑就够了（E0） |
| 时间序列基础模型（Chronos、TimesFM）的微调或预训练 | SH、SM、OM、OH | **放进讨论或未来工作** | 超出本文范围，投入产出比低（E17，P3） |
| Exchange、ILI、PEMS、M4 等更多数据集 | SM、SH、FH | **降级** | 优先级低于 Traffic 和 PatchTST。只做 Exchange 作为"不可预测"的压力测试是可选项 |
| 截断偏差和方差的形式化理论 | SM | **可选** | TMLR 不要求理论。如果 E1 结果有利，可以加一个小命题，说明 trimmed MSE 是鲁棒位置估计 |
| 更强的"修复"方法（STL、模型预测替换） | SH | **降级** | E10（只修复目标）已经能消除主要混杂 |
| 对数比率聚合 | SH | **可选** | 作为敏感性分析即可 |
| 图 3 的纵轴范围 | SH | **可选** | 纯排版 |

---

## 6. 关于 TMLR 录取的策略建议

### 6.1 重新定位（零成本，最高性价比）

**问题**：标题和摘要承诺的是"data pruning"（让读者期待效率），证据给出的是"监督重加权带来的精度收益"。

**推荐的标题方案**（按改动程度由小到大）：
1. *Prune the Supervision, Not the Windows: Decision Units for **Supervision Masking** in Sliding-Window Forecasting*（改动最小）
2. *Which Loss Terms to Drop? Decision Units for Supervision Pruning in Time-Series Forecasting*
3. （E1 结果不利时）*How Fine Should Trimming Be? Decision Units for Robust Supervision in Sliding-Window Forecasting*

**摘要的结构建议**（3–4 个核心发现，每个都附证据强度）：
1. 问题：stride=1 时每个目标被监督约 T 次，窗口只是最粗的决策单元。
2. 设计：固定得分和预算，只改单元；对照包括 high、low、random，以及 L1、Huber、在线截断（E1、E2 完成后）。
3. 发现 ①：屏蔽 high end 改善 MAE，MSE 改善不显著。用一句话同时给出两者，并说明"按骨干平均"。
4. 发现 ②：单元越细收益越大，并给出机制（窗口重叠 → 窗口得分自相关 → 窗口掩码相当于整段删除；或"定位精度"）。
5. 发现 ③：收益集中在误差分布主体；尾部经等价检验后无可测变化（如果 TOST 通过的话）。
6. 明确声明：掩码本身**不缩短 epoch**；如果需要省算力，可以叠加等间距删窗（要有 E7 的证据）。

### 6.2 E1 结果的决策树（决定论文最终怎么写）

```
E1：在 3 个骨干 × 7 个数据集上，比较 L1、Huber、在线截断、mask+L1
│
├─ 情况 A：掩码的 MAE ≥ L1/Huber，且掩码不损害 MSE，而 L1/Huber 损害 MSE
│     → 最强叙事："targeted masking 拿到鲁棒损失的 MAE 收益，又不牺牲 MSE"
│       写进摘要的第一个发现；保留 data pruning 或 supervision masking 的框架
│
├─ 情况 B：L1 的 MAE 与掩码相当，但掩码在 MSE、CVaR 上更好
│     → 定位为"保留 MSE 目标的鲁棒训练替代方案"；单元分析作为第二贡献
│
├─ 情况 C：mask+L1 优于 L1（两者互补），且 L1 下单元排序依然成立
│     → "单元重要"的主张不依赖损失函数，核心叙事得以保留，这是最理想的情况
│
├─ 情况 D：固定的早期掩码优于在线截断
│     → 新的正面结论："一次性早期决定优于逐批截断"，进摘要
│
└─ 情况 E（最坏）：L1/Huber 在 MAE 和 MSE 上都 ≥ 掩码，且 mask+L1 ≈ L1
      → 必须重新定位为"滑动窗口中截断粒度的系统研究"（标题方案 3）：
        - 删除"推荐默认单元"这类实践建议，改为解释性贡献
          （为什么窗口最差：重叠 + 自相关 → 定位精度；element 与 occurrence 的差距 → 上下文依赖）
        - 强调这是对社区有价值的澄清：已有的窗口级 TS 剪枝方法可能只是在做粗粒度的鲁棒化
        - TMLR 不要求 SOTA 或新颖性，这样的论文仍然可以被接收，但需要在 claims 上非常克制
```

### 6.3 Claims 审计表：每条主张应如何处理

| # | 原文主张（位置） | 现有证据 | 处理 |
|---|---|---|---|
| 1 | "masking the highest-residual 10% … lowers test MAE at every unit and on all seven datasets"（摘要） | 表 1，按骨干平均后成立 | **保留，补充限定**："averaged over backbones"；同时给出 MSE 结果 |
| 2 | "masking the lowest-residual 10% never helps"（摘要、§5） | 只有 10% 一个预算，且在 MSE 下几乎是必然结果 | **弱化**为 "gave no detectable gain at r=10%"；E3 后可再加强 |
| 3 | "the gain … comes from which loss terms are masked, not from masking as such"（§5） | low end 不能证明这一点 | **在 E2 之前删除**；E2 结果有利再恢复 |
| 4 | "The window gains least"（§1、§5、§8） | 存在残差质量混杂 | **补上机制**（E0 和 E4）；改写为"窗口在相同项预算下定位精度最差" |
| 5 | "the worst 10% of the errors does not change measurably"（摘要、§6） | 只是不显著 | **TOST 通过后保留**；否则改为 "no significant change (n=7, not tested for equivalence)" |
| 6 | "MSE therefore improves on average … but not significantly"（摘要） | 表 2 | **保留，并在摘要中提升到与 MAE 同等的位置** |
| 7 | "As a default we recommend the channel row"（§6、§8） | p=0.525 | **改为条件式推荐** |
| 8 | "the score ranks targets by the error that the rest of the series leaves unexplained"（摘要、§2、§7.1） | 早期读取，近乎同义反复 | **弱化**为 "…closely matches the early-training cross-fitted residual"；E8 结果有利再加强 |
| 9 | "a score read once early … comes close to scores that cost far more"（摘要） | MC-dropout 在 MSE 上显著更好 | **改写**（见 B6） |
| 10 | "the element mask is ahead of Selective Learning in MAE on all six datasets"（摘要） | SL 被改造过；MSE 不显著 | **加限定**："at a matched 10% budget; MSE n.s."；E9 后再定 |
| 11 | "Masking is also better than repairing"（§7.2） | 插值改动了输入 | **E10 之前改为** "masking beats interpolating the training series" |
| 12 | "windows can then be pruned at equal spacing, without a score, to save optimiser steps"（摘要、§7.3） | 没有表、没有 TOST、没有 wall-clock | **E7 之前降级到附录**，或删除 |
| 13 | "the occurrence is ahead on every dataset of every backbone"（§5，指 median） | 无法核实 | **补充表格** |
| 14 | "Selective Learning … is the one prior forecasting method that masks supervision below the window"（§2） | 过于绝对 | **弱化**："the closest prior forecasting method" |
| 15 | "In our setting the opposite end pays"（§2） | 叙述有误导性 | **改写**为连续谱（见 A7） |
| 16 | "ECL, the only high-dimensional dataset"（§5、附录 D） | solar 有 137 个通道 | **改为** "the dataset with the most channels" |

### 6.4 投稿前检查清单

- [ ] E0、E1、E2 全部完成（P0），结果写进主表（表 1 和表 2 增加 random、L1、Huber、online-trim 行）
- [ ] 表 1、2、4–7 加上种子标准差或 95% CI；附录加绝对值表
- [ ] 发散运行计数表
- [ ] 尾部的 TOST；Holm 校正；ETT 合并后的敏感性分析
- [ ] 掩码结构诊断图：lead time、通道、时段、绝对时间、窗口连通段
- [ ] 分 horizon 的表 1、表 2
- [ ] §7.3 完整表格（E7），或降级处理
- [ ] 标题和摘要按 §6.1 改写；按 §6.3 逐条处理 claims
- [ ] 相关工作补充鲁棒回归、LTS、CCS、Moderate；承认"得分 = 回归版 EL2N"
- [ ] Limitations 小节 + 尾部风险说明
- [ ] 算法框、超参数表、式 4 的实现说明、验证集说明、残差尺度说明
- [ ] 匿名代码仓库（TMLR 审稿人很看重）
- [ ] C 级的 15 条小修正

### 6.5 rebuttal 预案（如果审稿人仍提出以下问题）

- **"这就是 trimmed loss"** → 引用 E1 的决策树结果。强调本文的贡献是"截断的**粒度**和**跨窗口单元**"，并且"保持 MSE 目标"（如果是情况 A 或 B）。
- **"不省算力"** → 已在标题和摘要中声明；给出 E7 的 Pareto 曲线。
- **"只有 7 个数据集，而且不独立"** → 给出 ETT 合并后的检验和 Traffic、PatchTST 的复现（E13）。
- **"SL 比较不公平"** → 给出 E9 的官方设置结果。

---

## 7. 审稿人遗漏、由我补充的要点

1. **§7.1 已经隐含了随机掩码的 MAE**：−3.97+3.59 ≈ **−0.38%**，MSE 为 −0.16%。可以马上写进正文，作为"随机掩码几乎无收益"的部分证据（目前只覆盖 iTransformer×element）。
2. **S1 的精确表述**：鲁棒统计给出了"降低影响"和"位置偏移"两种机制，审稿人说"均值→中位数可完全解释"过于绝对（见 S1）。rebuttal 中用精确说法会更有说服力。
3. **L1 的 MSE 代价是论文潜在的最大卖点**（见 §6.2 情况 A），没有一位审稿人明确点出。E1 必须同时报告 MSE 和 CVaR，不能只看 MAE。
4. **TMLR 没有正式的 Major revision 决定**：3 个家族一致给出 Major revision，意味着关键实验必须**在投稿前**完成。讨论期只有约两周，不足以跑完 E1 加 E2（约 6–8C 的计算量，见实验计划）。
5. **数字一致性已核实**：表 3 中 "ours" 一行（MSE −2.81、MAE −3.97）与表 4 中 iTransformer 去掉 ECL 后 element 的均值（−2.80、−3.975）一致；表 1 均值、Fig. 1 柱值、§5 和 §7 中引用的差值也都一致。**这一点可以在 rebuttal 中作为"结果可信"的背书**，但同时说明论文的问题不在数字，而在解释和对照。
6. **概念上的悖论应主动写进 Discussion，化被动为主动**：element 是论文强调的"只有滑动窗口才有"的单元，但它不如 occurrence（低 0.91 点 MAE）。Fable·high 的解释是"目标是否值得学取决于上下文"。这其实是一个有趣的发现，应该正面讨论，而不是只在 §5 一句带过。

---

## 8. 录取前景评估（主观判断，仅供参考）

| 状态 | 预计审稿结果 | 主要风险 |
|---|---|---|
| 当前版本直接投稿 | 很可能拿到 2–3 个"需大改"意见，最终 **Reject** 的可能性较大 | S1–S3 几乎一定会被提出，讨论期内来不及补 |
| 完成 P0（E0+E1+E2）+ 全部改写（R） | 有较大机会进入 **Accept with minor revision** | 取决于 E1 的结果；如果是情况 E，需要重新定位 |
| 完成 P0+P1（加 E3–E7） | 录取把握明显更高 | 剩余风险主要在泛化性（PatchTST、Traffic），可在讨论期补 |

> 说明：以上是定性判断，不是统计预测。依据有两点：3 个模型家族（有效独立意见约 3 份，而不是 6 份，见 §1.5）在不在提示词中的核心问题上高度一致；以及 TMLR"claims 与证据一致"的标准。6 个审稿代理都是 Claude 系列模型，可能有共同盲区，真实审稿人还可能提出这里没有覆盖的问题。

---

## 附：6 份原始审稿意见

- `review_sonnet_medium.md`，`review_sonnet_high.md`
- `review_fable_medium.md`，`review_fable_high.md`
- `review_opus_medium.md`，`review_opus_high.md`

## 参考文献（Harvard；只列本报告新提到、且确信存在的文献）

- Ayed, F. and Hayou, S. (2023) 'Data pruning and neural scaling laws: fundamental limitations of score-based algorithms', *Transactions on Machine Learning Research*.
- Benavoli, A., Corani, G. and Mangili, F. (2016) 'Should we really use post-hoc tests based on mean-ranks?', *Journal of Machine Learning Research*, 17(5), pp. 1–10.
- Demšar, J. (2006) 'Statistical comparisons of classifiers over multiple data sets', *Journal of Machine Learning Research*, 7, pp. 1–30.
- Holm, S. (1979) 'A simple sequentially rejective multiple test procedure', *Scandinavian Journal of Statistics*, 6(2), pp. 65–70.
- Hounie, I., Porras-Valenzuela, J. and Ribeiro, A. (2024) 'Loss shaping constraints for long-term time series forecasting', *ICML*.
- Huber, P.J. (1964) 'Robust estimation of a location parameter', *The Annals of Mathematical Statistics*, 35(1), pp. 73–101.
- Kang, D. and Hashimoto, T. (2020) 'Improved natural language generation via loss truncation', *ACL*.
- Lakens, D. (2017) 'Equivalence tests: a practical primer for t tests, correlations, and meta-analyses', *Social Psychological and Personality Science*, 8(4), pp. 355–362.
- Paul, M., Ganguli, S. and Dziugaite, G.K. (2021) 'Deep learning on a data diet: finding important examples early in training', *NeurIPS*.
- Qiu, X. et al. (2024) 'TFB: towards comprehensive and fair benchmarking of time series forecasting methods', *Proceedings of the VLDB Endowment*, 17(9).
- Rousseeuw, P.J. (1984) 'Least median of squares regression', *Journal of the American Statistical Association*, 79(388), pp. 871–880.
- Shen, Y. and Sanghavi, S. (2019) 'Learning with bad training data via iterative trimmed loss minimization', *ICML*.
- Sorscher, B., Geirhos, R., Shekhar, S., Ganguli, S. and Morcos, A.S. (2022) 'Beyond neural scaling laws: beating power law scaling via data pruning', *NeurIPS*.
- Xia, X. et al. (2023) 'Moderate coreset: a universal method of data selection for real-world data-efficient deep learning', *ICLR*.
- Zheng, H., Liu, R., Lai, F. and Prakash, A. (2023) 'Coverage-centric coreset selection for high pruning rates', *ICLR*.
