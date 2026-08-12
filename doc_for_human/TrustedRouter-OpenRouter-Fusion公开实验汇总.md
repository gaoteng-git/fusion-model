# TrustedRouter / OpenRouter 公开Fusion实验汇总：模型组合、solo对照、跑分（更新版）

**说明**：本文收录能查到具体面板+分析(judge)+合成(synthesizer)模型组成、且同一篇材料里给出了跟GPT-5.5或其他solo模型对照的实验。每条给出①**发布日期**②官网博客URL（可直接打开）③本地克隆仓库源码文件的行号，并附commit锁定的GitHub链接，方便逐行核对。**本次更新新增发布日期，各小节内部、以及第三节汇总表，均按时间从远到近排列。**

**⚠️ 贯穿全文的方法论保留意见**：DRACO评测的"面板作答"这一步，**从未真正走生产环境的接口**——生产环境真实服务`trustedrouter/prometheus-2.0`等模型的，是`quill-cloud-proxy`仓库里的Go代码（`enclave-go/cmd/enclave/fusion.go`），面板阶段用的是通用prompt（"Answer the request independently and return only the visible answer."，第2429行），不给实时联网工具；而DRACO评测团队自己另外写了一个客户端脚本（`TrustedRouter-Fusion-Draco/scripts/draco_client_fusion.py`），**绕开生产的面板调度机制，自己单独调用面板模型并给它们塞实时联网工具**（脚本自己的文档字符串写明："The native gateway /fusion endpoint cannot give panels live tools...so we reproduce Fusion client-side"）。只有判官和合成这两步，脚本才尝试"照抄生产代码的prompt"——核对下来判官那步几乎逐字一致（只差一个产品改名的词："Fusion"→"Synth"），合成那步则是重写过的加长版，不是逐字复制。**这意味着本文列出的所有DRACO分数，衡量的都是"这套模仿出来的流程"，不是"真实调用一次对应模型接口会得到什么"**，这条对本文一、二两节里所有来自`TrustedRouter-Fusion-Draco`的分数同样适用，不是Prometheus 2.0一家的问题。

源码文件的GitHub链接（锁定commit）：
- `blog.py`（quill-router，博客正文数据源）：https://github.com/Lore-Hex/quill-router/blob/838283f3956ea2b641b05eb4785d8bd3d0e6a06c/src/trusted_router/content/blog.py
- `fusion.html`（quill-router，产品页）：https://github.com/Lore-Hex/quill-router/blob/973599749d3d3602a46f203a92d024a87c5bfba4/src/trusted_router/templates/public/fusion.html
- `catalog_data.py`（quill-router，模型/面板常量定义）：https://github.com/Lore-Hex/quill-router/blob/75c8fc0bc7c86143d0fc15f7c665b4252a5c3bc3/src/trusted_router/catalog_data.py
- `fusion.go`（quill-cloud-proxy，**真实生产环境执行引擎**）：https://github.com/Lore-Hex/quill-cloud-proxy/blob/129c54b01ce478540dbb4c81f64cc4943c847d98/enclave-go/cmd/enclave/fusion.go
- `FINDINGS.md`（TrustedRouter-Fusion-Draco，评测原始记录）：https://github.com/Lore-Hex/TrustedRouter-Fusion-Draco/blob/3d40af9c63c8d9237b9a01a51349d6c8f63c93d9/docs/FINDINGS.md
- `draco_client_fusion.py`（TrustedRouter-Fusion-Draco，评测客户端脚本）：https://github.com/Lore-Hex/TrustedRouter-Fusion-Draco/blob/db99682172e7b788b76754b70ecdb049646728b5/scripts/draco_client_fusion.py

按fusion配置里是否混入闭源模型，分两大类；每类内部按发布时间从远到近排列。

---

## 一、纯开源Fusion配置（面板 + 分析 + 合成，全部是开源模型）

### 1.1 Iris 1.0（budget，3个开源模型）—— 2026-06-17（后于06-24重新包装为"Iris 1.0"）

| 项目 | 内容 |
|---|---|
| 发布日期 | **2026-06-17**（首次以"Budget panel"面貌出现）；**2026-06-24**（重新包装为"Iris 1.0"产品名） |
| 面板 | MiniMax M3、Kimi K2.6、DeepSeek V4 Pro（3个开源模型） |
| 分析(judge) | Kimi K2.6 |
| 合成(synthesizer) | GLM 5.2 |
| 分数 | **62.6**，约$20/100题 |
| 对照solo模型 | Fable-5 solo 65.3、GPT-5.5 solo 63.0-63.3、Opus-4.8 solo 60.3-60.7 |
| 原文 | `synth-iris-prometheus-zeus`（06-24）<br>https://trustedrouter.com/blog/synth-iris-prometheus-zeus |
| 行号 | `blog.py` 第754-780行（面板/分数描述在第776行） |

**⚠️ 一个数据矛盾，未解决**：同一个62.6分，在更早发布（06-17）的`fusion-evals-open-source`一文（https://trustedrouter.com/blog/fusion-evals-open-source ，第1357-1367行）里，图表标注的是"Budget panel → Opus synthesizer"（面板是Gemini-3-Flash+Kimi-K2.6+DeepSeek-V4-Pro，**合成器是闭源的Opus-4.8**），跟06-24发布的`synth-iris-prometheus-zeus`一文里"Iris 1.0"的描述（面板换成MiniMax M3+Kimi K2.6+DeepSeek V4 Pro，**合成器是开源的Kimi判官+GLM合成**）不是同一套配置，却给出了完全相同的62.6分，两篇官方文章对不上。

### 1.2 Prometheus 1.0 / 全开源委员会（5个开源模型）—— 2026-06-17（后于06-24重新包装为"Prometheus 1.0"）

| 项目 | 内容 |
|---|---|
| 发布日期 | **2026-06-17**（`open-fusion-beats-fable-5`首次发布69.2分）；**2026-06-24**（重新包装为"Prometheus 1.0"产品名） |
| 面板 | MiniMax M3、Kimi K2.6、DeepSeek V4 Pro、Gemma 4、GLM 5.2（5个开源模型） |
| 分析(judge) | Kimi K2.6 |
| 合成(synthesizer) | GLM 5.2 |
| 分数 | **69.2**，约$34-80/100题 |
| 对照solo模型 | Opus 4.8 solo **60.7**、GPT-5.5 solo **63.0**、Fable 5 solo **65.3** |
| 原文① | `open-fusion-beats-fable-5`（06-17）<br>https://trustedrouter.com/blog/open-fusion-beats-fable-5 |
| 行号① | `blog.py` 第1133-1164行 |
| 原文② | `synth-iris-prometheus-zeus`（06-24）<br>https://trustedrouter.com/blog/synth-iris-prometheus-zeus |
| 行号② | `blog.py` 第754-780行 |
| 原始数据 | `FINDINGS.md` §7.2，第245-260行<br>https://github.com/Lore-Hex/TrustedRouter-Fusion-Draco/blob/3d40af9c63c8d9237b9a01a51349d6c8f63c93d9/docs/FINDINGS.md |
| **生产代码交叉验证** | `fusion.go`第95-101行`fusionQualityPanel`常量：minimax-m3 + 通用Kimi + glm-5.2 + gemma-4-31b-it + deepseek-v4-pro，跟博客描述的面板基本一致（对应`trustedrouter/prometheus-1.0`等模型） |

超过Fable-5 solo(65.3分)+3.9分，超过GPT-5.5 solo(63.0分)+6.2分。

### 1.3 MiniMax-M3 自我融合（1个开源模型，自己融合自己）—— 2026-06-18

| 项目 | 内容 |
|---|---|
| 发布日期 | **2026-06-18** |
| 配置 | 1个开源模型（MiniMax-M3）独立跑N次，自己融合自己 |
| 分数 | N=4：**68.1**（~$37）；N=10：**69.4**（~$87） |
| 对照solo模型 | MiniMax-M3自己solo **66.2**；Fable-5 solo **65.3**（~$250模型化推算价） |
| 原文 | `ten-cheap-runs-beat-the-frontier`<br>https://trustedrouter.com/blog/ten-cheap-runs-beat-the-frontier |
| 行号 | `blog.py` 第1047-1066行 |
| 原始数据 | `FINDINGS.md` §6，第161-218行 |

四次自融合($37)已超过Fable-5 solo，约1/7价格；十次自融合($87)达69.4，约1/3价格。**不含闭源模型，但不是"5面板"，是单模型重复采样。**

### 1.4 Prometheus 2.0（77.4分）—— 2026-07-18，本文时间线上最新的一条

| 项目 | 内容 |
|---|---|
| 发布日期 | **2026-07-18**，比其它所有条目晚了近一个月，是本文时间线上最新的一条 |
| 面板（5个，全部开源） | MiniMax M3、**Kimi K3**（非1.2里的K2.6）、GLM 5.2、DeepSeek V4 Pro、**MiMo V2.5 Pro**（1.2里没有这个模型） |
| 分析(judge) | MiniMax M3（主），失败降级Kimi K3 |
| 合成(synthesizer) | Kimi K3（主），失败降级GLM 5.2，再降级MiniMax M3 |
| 对外宣传分数 | **77.4**，95%置信区间[74.5, 80.2] |
| 对照solo模型（同文章给出） | Fable-5 solo **65.3**、GPT-5.5 solo **63.3**、Claude Opus 4.8 solo **60.3** |
| 博客原文 | `prometheus-2-new-draco-state-of-the-art`<br>https://trustedrouter.com/blog/prometheus-2-new-draco-state-of-the-art |
| 行号 | `blog.py` 第130-183行，关键数字在第179-180行 |
| **模型清单证据①**（代码常量） | `catalog_data.py`第1443-1449行`SYNTH_PROMETHEUS_2_MODEL_ORDER`<br>https://github.com/Lore-Hex/quill-router/blob/75c8fc0bc7c86143d0fc15f7c665b4252a5c3bc3/src/trusted_router/catalog_data.py#L1443-L1449 |
| **模型清单证据②**（产品页文案，含判官/合成分工） | `fusion.html`第81-83行<br>https://github.com/Lore-Hex/quill-router/blob/973599749d3d3602a46f203a92d024a87c5bfba4/src/trusted_router/templates/public/fusion.html#L81-L83 |
| **模型清单证据③**（测试断言） | `tests/test_public_revenue_pages.py`第423-430行<br>https://github.com/Lore-Hex/quill-router/blob/e7c68c05ccb5cc7079923ffd75d71d720491b24c/tests/test_public_revenue_pages.py#L423-L430 |
| **模型清单证据④**（生产环境真实执行引擎，最权威） | `fusion.go`：面板第110-116行`fusionPrometheus20Panel`；判官分工第316-318行；合成分工第295-297行<br>https://github.com/Lore-Hex/quill-cloud-proxy/blob/129c54b01ce478540dbb4c81f64cc4943c847d98/enclave-go/cmd/enclave/fusion.go#L110-L116 |

**四份独立来源完全一致，模型清单本身可信度很高，且5个角色对应的都是开源模型——如果77.4分成立，这会是本文全部纯开源配置里分数最高的一组，比1.2的69.2还高8.2分。**

**但有两条硬伤，导致这个分数本身无法采信，跟"模型是谁"是两件独立的事**：
1. **发布这篇博客的commit本身承认没写清楚**——commit `b83a6cdf`（2026-07-18，作者Joseph Perla）message原文："Lead chart with 95% bootstrap CI whiskers; links to the DRACO/Synth/self-fusion posts; **no composition details, just the model id and the attested gateway.**"——作者自己承认这篇没公开面板构成，博客正文的"面板未公开"不是巧合。
2. **`TrustedRouter-Fusion-Draco`仓库里找不到这个组合的任何运行记录**——全仓库、全部39条commit、两个分支，搜"kimi-k3"和"mimo"均无匹配到实际运行过的实验（`mimo`唯一命中在`catalog.py`第685-695行，只是SDK层面"小米是个可选供应商"的通用登记，型号还是`mimo-v2-flash`而非`mimo-v2.5-pro`，从未出现在任何`replays/*.jsonl`的实际运行记录里）；且该仓库最后一次commit是**2026-06-24**，**比这篇博客发布日期（07-18）早了近一个月**——该仓库从时间线上就没有机会记录这次跑分。**这条时间差本身就是本文所有条目里最能说明问题的一处：仓库停更→产品换面板（K2.6+Gemma4 → K3+MiMo）→博客发新分数，三件事按时间顺序连起来，公开仓库没跟上产品迭代的节奏。**

**结论**：模型清单可信，跑分数字目前查无实据。

---

## 二、含闭源模型的Fusion配置

### 2.1 OpenRouter自己发布的最佳融合（Fable-5 + GPT-5.5，2个闭源模型）—— 约2026年5月，本文时间线上最早的一条

| 项目 | 内容 |
|---|---|
| 发布日期 | **约2026年5月**（估算，未独立核实——TrustedRouter 06-23发布的`fusion-works-now-even-self-fusion`一文原话"Last month OpenRouter showed..."，倒推大约一个月前，即上个月） |
| 配置 | Fable-5起草（闭源）+ GPT-5.5融合（闭源），2模型 |
| 分数 | **69.0**，约$450模型化推算价 |
| 对照solo | Fable-5 solo 65.3、GPT-5.5 solo 63.0-63.3 |
| 原文 | OpenRouter原始博客：https://openrouter.ai/blog/announcements/fusion-beats-frontier/（未独立核实，日期未验证）；TrustedRouter转引见`open-fusion-beats-fable-5`第1151-1154行（06-17），https://trustedrouter.com/blog/open-fusion-beats-fable-5 |

OpenRouter一手数据，非5面板结构。**是本文里唯一日期未经独立核实的条目**，排在最前只是估算意义上的"最早"，不代表可信度最高。

### 2.2 Zeus 1.0 / Frontier panel（5模型面板，含2-3个闭源）—— judge×synthesizer全排列，2026-06-17 首发，2026-06-19 补完整网格

面板固定为：**GPT-5.5（闭源）+ Claude Opus-4.8（闭源）+ Gemini-3-Flash（闭源）+ Kimi-K2.6（开源）+ DeepSeek-V4-Pro（开源）**。

| 发布日期 | 分析(judge) | 合成(synthesizer) | 分数 | 备注 |
|---|---|---|---:|---|
| 06-19 | Kimi K2.6 | GLM-5.2 | **73.4** | 全场最高分（SOTA） |
| 06-19 | MiniMax-M3 | GLM-5.2 | 72.3 | |
| 06-19 | GLM-5.2 | GLM-5.2 | 70.3 | 自己判自己 |
| 06-19 | MiniMax-M3 | MiniMax-M3 | 67.1 | |
| 06-19 | Kimi K2.6 | MiniMax-M3 | 67.1 | |
| 06-19 | GLM-5.2 | MiniMax-M3 | 68.0 | |
| 06-19 | GLM-5.2 | Kimi K2.6 | 66.9 | |
| 06-19 | MiniMax-M3 | Kimi K2.6 | 64.7 | |
| 06-19 | Kimi K2.6 | Kimi K2.6 | **48.7** | 全场最低分 |
| 06-17 | （固定gemini-3.1-pro judge） | Opus-4.8 | 70.6 | 最早一版 |
| 06-17 | （固定gemini-3.1-pro judge） | **GPT-5.5** | **62.2** | 全场倒数第二，⚠️该行judge也被换成GPT-5.5自己，非单变量对照 |
| 06-17 | （固定gemini-3.1-pro judge） | Gemma-4-31b | 54.0 | 垫底 |

| 对照solo模型 | 分数 |
|---|---:|
| GPT-5.5 solo | 63.0（另一篇写63.3） |
| Claude Opus-4.8 solo | 60.3-60.7 |
| DeepSeek V4 Pro solo | 57.5-59.9 |
| Kimi K2.6 solo | 46.3-50.1 |
| Gemini 3.1 Pro solo | 47.1-47.4 |
| Gemini 3 Flash solo | 40.4-41.1 |

| 原文① | `fusion-evals-open-source`（06-17，首发70.6/62.2/54.0这几行）<br>https://trustedrouter.com/blog/fusion-evals-open-source |
|---|---|
| 行号① | `blog.py` 第1314-1397行 |
| 原文② | `the-best-synthesizers`（06-17）<br>https://trustedrouter.com/blog/the-best-synthesizers |
| 行号② | `blog.py` 第1119-1130行 |
| 原文③ | `fusion-is-two-jobs`（06-19，补完整3×3网格，含73.4/48.7）<br>https://trustedrouter.com/blog/fusion-is-two-jobs |
| 行号③ | `blog.py` 第1026-1044行 |
| 原始数据 | `FINDINGS.md` §7.1，第229-244行 |

### 2.3 Budget panel（3个开源面板成员 + 闭源Opus-4.8合成器）—— OpenRouter原始实验复现，2026-06-17

| 项目 | 内容 |
|---|---|
| 发布日期 | **2026-06-17** |
| 面板 | Gemini-3-Flash（闭源）+ Kimi-K2.6（开源）+ DeepSeek-V4-Pro（开源） |
| 分析(judge) | Gemini-3.1-Pro（闭源） |
| 合成(synthesizer) | **Opus-4.8（闭源）** |
| 分数 | TrustedRouter复现：**62.6**；`FINDINGS.md`另给60.8（全100）/63.2（非财务80） |
| OpenRouter原始发布分数 | **64.7** |
| 原文 | `fusion-evals-open-source`<br>https://trustedrouter.com/blog/fusion-evals-open-source |
| 行号 | `blog.py` 第1357-1359行、第1364-1367行 |
| 原始数据 | `FINDINGS.md` §1，第44-103行 |

未超过GPT-5.5 solo（63.0-63.3）。

### 2.4 Claude Sonnet 4.6 / Haiku 4.5 自我融合（全闭源，judge×synthesizer 2×2）—— 2026-06-23 首发，2026-06-24 拆解

| 项目 | 内容 |
|---|---|
| 发布日期 | **2026-06-23**（首次公布+8.0/+2.6）；**2026-06-24**（拆解judge/synthesizer各自贡献） |
| 配置 | 单一闭源模型自己融合自己，Sonnet 4.6 / Haiku 4.5各自测试 |
| Sonnet 4.6 | solo 66 → 自融合 **~74**（+8.0，显著） |
| Haiku 4.5 | solo 55 → 自融合 **~58**（+2.6，不太显著） |
| 裁判 | Sonnet-4.6自己评分，非标准gemini-3.1-pro |
| 原文① | `fusion-works-now-even-self-fusion`（06-23）<br>https://trustedrouter.com/blog/fusion-works-now-even-self-fusion |
| 行号① | `blog.py` 第812-832行 |
| 原文② | `self-fusion-gain-lives-in-the-synthesizer`（06-24）<br>https://trustedrouter.com/blog/self-fusion-gain-lives-in-the-synthesizer |
| 行号② | `blog.py` 第783-810行 |

全闭源，非5模型面板结构。**是本节（含闭源类）里发布最晚的一条**，仅次于第一节里07-18发布的Prometheus 2.0。

---

## 三、汇总对照表（按发布日期从远到近排列）

| 发布日期 | 分类 | 配置 | 面板是否含闭源 | 分数 | vs GPT-5.5 solo(~63) | 模型清单可信度 |
|---|---|---|---|---:|---:|---|
| ~2026-05（估算，未核实） | 含闭源 | OpenRouter: Fable-5+GPT-5.5 | 是（全闭源，2模型） | 69.0 | **+6** | 中（未独立核实一手来源） |
| 2026-06-17 | 纯开源 | Iris 1.0（3开源） | 否 | 62.6 | 略低 | 中（跟另一篇矛盾） |
| 2026-06-17 | 纯开源 | Prometheus 1.0/开源委员会（5开源） | 否 | 69.2 | **+6.2** | 高 |
| 2026-06-17 | 含闭源 | Zeus 1.0/Frontier panel 最差格 | 是 | 62.2 | 略低 | 高（但judge也被换了） |
| 2026-06-17 | 含闭源 | Budget panel（3开源+闭源合成） | 是 | 60.8-63.2 | 略低 | 高 |
| 2026-06-18 | 纯开源 | MiniMax-M3自融合×4 | 否 | 68.1 | **+5.1** | 高 |
| 2026-06-18 | 纯开源 | MiniMax-M3自融合×10 | 否 | 69.4 | **+6.4** | 高 |
| 2026-06-19 | 含闭源 | Zeus 1.0/Frontier panel 最佳格 | 是 | 73.4 | **+10.4** | 高 |
| 2026-06-19 | 含闭源 | Zeus 1.0/Frontier panel 垫底格 | 是 | 48.7 | 低 | 高 |
| 2026-06-23 | 含闭源 | Sonnet 4.6自融合 | 是（全闭源） | ~74 | **+11** | 高 |
| 2026-06-23 | 含闭源 | Haiku 4.5自融合 | 是（全闭源） | ~58 | 略低 | 高 |
| 2026-06-24 | 纯开源 | Iris 1.0（重新包装） | 否 | 62.6 | 略低 | 中（跟前一篇矛盾） |
| 2026-06-24 | 纯开源 | Prometheus 1.0（重新包装） | 否 | 69.2 | **+6.2** | 高 |
| **2026-07-18** | **纯开源** | **Prometheus 2.0（5开源）** | **否** | **77.4** | **+14.1** | **模型清单高（4处独立证据）；分数本身查无实据** |

**这张表按时间轴排出来，能看出一条明显的时间线**：2026年5月OpenRouter先发布69.0分挑起话题 → TrustedRouter 6月17-24日一周多的时间里密集发布了8篇文章、逐步把分数从62.6/69.2推高到73.4 → 之后**近一个月的空窗期**（6月24日到7月18日，正好是`TrustedRouter-Fusion-Draco`仓库停止更新的那段时间）→ 7月18日突然发布77.4分，且这条是全表**唯一一条前后没有渐进过程、没有中间数据支撑、直接跳到最高分**的记录。

---

## 四、分析

**1. "融合赢单模型"这件事，两家公司反复验证过，方向是稳的。** 纯开源委员会（69.2）、开源自融合（68.1-69.4）、混闭源的frontier面板最佳格（73.4）、OpenRouter自己的最佳融合（69.0），全都跑赢了各自的最强solo基线。这不是孤证，是多篇独立发布的文章反复验证出来的结果。

**2. "融合赢多少"，取决于要不要混入闭源模型。** 纯开源配置赢GPT-5.5 solo的幅度是+5到+6分；混入闭源模型能冲到73.4分（+10分）。如果0G的方案定位是"纯开源面板"，该参照+5~6分这个量级，不是73.4或77.4这两个数字——**尤其77.4，虽然模型清单确认全开源，但分数本身缺乏可信证据支撑，不能直接当作"纯开源配置能达到的效果"去引用**。

**3. "最强闭源模型"这个基准点，两家公司测的都是几个月前的模型（Fable-5、GPT-5.5、Opus-4.8），不是当前最强的。** 今天市面上更新的闭源模型（GPT-5.6系列、Claude Opus 5/Sonnet 5）从未出现在这些对照实验里。

**4. DRACO评测的方法论本身存在一个贯穿全文的缺口：面板作答从未走生产代码路径。** 生产环境（`fusion.go`）的面板prompt是通用模板，不带实时联网工具；评测团队自己写的客户端脚本给面板模型额外塞了联网工具，绕开了生产的面板调度逻辑，理由是生产面板不给工具直接测分数只有40分左右。只有判官、合成两步的prompt号称"照抄生产代码"——判官那步核实下来几乎逐字一致（差一个产品改名的词），合成那步实际是重写的加长版，不是逐字复制。**这意味着本文列出的所有分数（不管纯开源还是含闭源），衡量的都是"这套模仿出来的流程"，跟"真实调用一次对应模型接口"存在结构性差距，不是单个数字的问题。**

**5. 官方材料里两处尚未解释清楚的不一致：**
   - Iris 1.0（62.6分）在两篇官方文章里对应两个不同的面板构成（一处合成器是闭源Opus-4.8，一处是开源Kimi判官+GLM合成），无法确认哪个是真实来源；
   - "GPT-5.5当合成器只得62.2分"这条，原始数据显示该行同时把判官也换成了GPT-5.5自己，不是干净的单变量对照。

**6. Prometheus 2.0是本文里"模型清单可信度"和"分数可信度"分离得最彻底的一条。** 四份独立代码证据（产品常量、产品页文案、测试断言、生产Go引擎）一致确认了5个开源模型+判官/合成分工，可信度很高；但分数本身——博客发布commit自称"未公开构成"，且专门的复现仓库里从未出现过这个具体组合、且仓库停止更新的时间还早于博客发布日期——两条独立线索都指向"77.4分目前没有可核实的原始记录"。**这两件事必须分开表述，不能因为模型清单查实了就反过来认为分数也可信，也不能因为分数查无实据就怀疑模型清单本身。**

**7. 【本次新增，按时间排序后才看出来的一条】77.4分是本文时间线上唯一一个"断层式跳跃"的数字。** 从5月的69.0到6月17-24日密集迭代出的73.4，中间每一步都有可查的过程数据（网格、消融、自融合曲线）；但73.4到77.4这一步，跳过了整整近一个月、且这一个月正好是评测仓库停止更新的窗口，中间没有任何一份公开材料记录过介于73.4和77.4之间的过渡数据。**其它每个数字的提升都能在本文里找到"上一版是多少、改了什么、涨了几分"的完整链条，只有77.4断在半空。**
