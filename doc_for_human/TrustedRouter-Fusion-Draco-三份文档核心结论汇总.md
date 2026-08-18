# TrustedRouter-Fusion-Draco 仓库：README / FINDINGS / LESSONS 三份文档核心结论汇总

本文档整合仓库里三份最重要的说明文档的主要结论：`README.md`（面向读者的概览+复现命令）、`docs/FINDINGS.md`（详细实验记录）、`docs/LESSONS.md`（方法论教训）。按文档分章节，最后附一节三份材料互相对照时发现的几处值得注意的地方。

---

# 一、README.md 核心结论（对外呈现的概览版本）

## 1.1 头条结论

> "a diverse panel — frontier *and* open-weights — reaches **71.6** on the full 100 tasks with MiniMax-M3 synthesizing... state of the art, above OpenRouter's best published fusion (Fable 5 + GPT-5.5, 69.0)."

面板 = `gpt-5.5 + opus-4.8 + gemini-3-flash + kimi-k2.6 + deepseek-v4-pro`（3闭源+2开源混合的frontier面板），判官固定`google/gemini-3.1-pro-preview`，**合成器换成MiniMax-M3拿到71.6分**，超过OpenRouter自己发布过的最佳融合成绩。

## 1.2 完整solo对照表（含一处诚实披露）

| solo | TrustedRouter | OpenRouter |
|---|---:|---:|
| GPT-5.5 | 63.0 | 60.0 |
| Claude Opus 4.8 | 60.7 | 58.8 |
| DeepSeek V4 Pro | 59.9 | 60.3 |
| Kimi K2.6 | 50.1 | 53.7 |
| Gemini 3.1 Pro | 47.4 | 45.4 |
| Gemini 3 Flash | 41.1 | 43.1 |
| **Claude Fable 5** | **(not run)** | 65.3 |

**Fable-5这一行明确标注"(not run)"**——TrustedRouter自己从未独立跑过Fable-5的solo成绩，文档里所有引用的65.3分，都是直接照抄OpenRouter自己发布的数字，不是TrustedRouter自己复现出来的。这是一处很坦诚的披露，值得注意。

## 1.3 完整融合配置对照表

| fusion config | TrustedRouter | OpenRouter |
|---|---:|---:|
| frontier面板 + MiniMax-M3合成 *(自己最佳)* | **71.6** | — |
| frontier面板 + GLM-5.2合成 | 71.1 † | — |
| frontier面板 + Opus合成 | 70.6 | — |
| OR — Fable 5 + GPT-5.5 *(对方最佳)* | — | 69.0 |
| OR — Opus + GPT-5.5 + Gemini | — | 68.3 |
| OR — Opus + GPT-5.5 | — | 67.6 |
| OR — Opus + Opus | — | 65.5 |
| budget面板 + Opus合成 | 62.6 | **64.7** |
| frontier面板 + GPT-5.5合成 | 62.2 | — |

**"budget面板+Opus合成"这一行，是TrustedRouter唯一一处自己的分数反而低于OpenRouter对应分数的地方（62.6 vs 64.7）**——README原话强调这一点，说明"如果真有什么系统性作弊/灌水，这条便宜配置应该也会表现出同样的偏高，但它没有"，作为反驳"分数虚高"质疑的论据之一。

## 1.4 GLM-5.2审查问题的完整披露

GLM-5.2在100题里有1题返回空内容——不是上下文超限（该题输入只有~19k token），是**政治审查**：面板报告涉及"大中华区"基金的中国/香港/台湾归属描述，GLM-5.2静默拒答（吐出一个停止符，零输出）。把"Taiwan"/"Hong Kong"换成中性词后，同一个面板能正常融合。这一题记0分，其余99题平均分71.8。**因此默认用MiniMax-M3而不是GLM-5.2**，尽管两者分数相近。

## 1.5 自我融合部分（README的精简版本）

- Opus 4.8：solo 60.7 → 2次自融合67.6（+6.9）
- MiniMax-M3：solo 66.2 → 2次自融合66.2（+0.0，没用）→ 10次自融合69.4
- 完整爬坡曲线（N=1到10）：66.2/66.1/67.7/68.1/68.1/68.2/69.5/69.2/68.4/69.4——2次没用，4次开始爬升并超过Fable-5(65.3)，7次左右到顶
- 成本：4次$37（~7倍便宜于Fable-5模型化的~$250），10次$87（~3倍便宜）

## 1.6 Haiku/Sonnet自融合对比（README版本，比FINDINGS.md的早期数字更新）

| self-fusion | solo | fused | gain | n tasks |
|---|---:|---:|---:|---:|
| Claude Sonnet 4.6 | 73 | ~79 | **+4.4** | 4 |
| Claude Haiku 4.5 | 60.5 | ~62 | **+1.5**（不显著） | 26 |

（这里的数字对应FINDINGS.md第8节里"n=26"那一版结果，不是最终"n=44/n=23"那一版——说明README在这处更新滞后于FINDINGS.md，细节见本文第四节。）

## 1.7 关于"是否作弊/泄题"的正面回应

- **无法完全解释跟OpenRouter的分数差异**：因为不知道OpenRouter自己harness的具体工具预算、抓取大小、合成步骤、判官轮次这些细节，"没法说清楚为什么某个具体数字不一样"。
- **差异是双向的、混合的，不是系统性偏向自己**：solo成绩有的比OpenRouter高（GPT-5.5+3.0、Gemini 3.1 Pro+2.0），有的比OpenRouter低（Kimi−3.6、Gemini Flash−2.0），"看起来像普通的跑分噪声，不是暗中动了手脚"。
- **泄题审计**：**12,704次web_search + 5,390次web_fetch，零命中DRACO/Perplexity/HuggingFace/评分细则/答案相关的域名**（这个数字比FINDINGS.md第2节写的"568次搜索+155次抓取"大得多，两者不是同一批统计口径，详见本文第四节）。

## 1.8 仓库文件地图（`Layout`一节）

```
src/trusted_router/evals/tr_sdk.py          网关传输层——所有模型调用都走这个SDK
src/trusted_router/evals/agentic_tools.py   agentic的web_search/web_fetch/bash循环
src/trusted_router/evals/draco_replay.py    replay格式 + 逐条评分细则打分
src/trusted_router/evals/{fusion_live,exa,draco,fusion_micro}.py   客户端、Exa、题目、裁判
scripts/draco_agentic_solo.py               跑一个模型的agentic研究（主要harness）
scripts/draco_client_fusion.py              客户端编排的融合（面板→判官→合成）
scripts/finance_parser_ablation.py          财务文档解析器对比实验
scripts/draco_native_fusion_gen.py          生成"原生trustedrouter/fusion"的replay
scripts/draco_rejudge.py                    用DRACO评分细则重新打分
scripts/draco_report.py                     并排分数报告
scripts/selffusion_gen_workflow.py          生成Claude Code Workflow脚本用于自融合实验
scripts/selffusion_analyze.py               从保存的输出重新推导自融合数字和图表
data/draco-{full-100,non-financial-80,financial-20}.manifest.json   题目+评分细则
replays/                                     原始agentic运行记录
results/                                     打好分的结果
artifacts/haiku-selffusion/                  自融合subagent实验的完整素材
docs/FINDINGS.md, docs/LESSONS.md           分析结论与经验教训
```

## 1.9 结尾"Notes"一节的四条要点

1. **"SOTA来自面板，不是来自单模型"**——solo层面自己的成绩跟OpenRouter持平（有高有低），budget融合甚至低于对方（62.6 vs 64.7）；真正拉开差距的是面板设计本身（把DeepSeek、Kimi这类开源frontier模型也拉进闭源面板里）+ 合成器选对了（MiniMax-M3）。
2. **泄题被三重核实过**。
3. **"合成器比面板本身更重要"**——同一个面板，换合成器分数能从54.0(Gemma-4)一路到71.6(MiniMax-M3)。
4. **"过程本身都公开了，不只是分数"**——每一次搜索、抓取、最终报告都在`replays/`里，`results/`是对应的打分结果，欢迎自己重新打分/审计。

---

# 二、FINDINGS.md 核心结论（详细实验记录，按原文档章节顺序）

## 第1节：差距的根源是"有没有实时工具"

- frozen context vs live tools，同模型分数能差20分（Kimi K2.6: 47.0→64.0；DeepSeek: 45.2→68.3）。
- 全100题solo成绩：GPT-5.5(63.3)最高，Opus-4.8(60.3)、DeepSeek(57.5)、Gemini-3.1-Pro(47.1)、Kimi K2.6(46.3)、Gemini-3-Flash(40.4)。
- "Budget Fusion"复现：60.8分，没超过自己的GPT-5.5 solo(63.3)，差距归因于财务文档解析能力。
- Fuser消融：MiniMax-M3(71.6)最好，**GPT-5.5(62.2)是最差的合成器**，Gemma-4-31b(54.0)垫底。
- GLM-5.2静默拒答根因确认为政治审查（涉台涉港内容触发）。
- 全开源面板：69.9分。

## 第2节：为什么自己的分数普遍比OpenRouter高

排除泄题；判官模型一致；真实原因是"自己的harness工具预算更慷慨"（16次工具调用、更大抓取字符数、强制首次调用工具、专门合成轮）——决定保留这套配置，但要求任何对比都要公开披露这些具体参数。

## 第3节：财务题验证为"确实更难"

财务20题比非财务80题普遍掉12-17分，原因是文档解析工具（markitdown）能力不如OpenRouter自己的方案。

## 第4-5节：其它诊断 + harness设计经验

Opus合成器截断bug（外层max_tokens导致）；"前N题更难"假设被推翻；工具硬截断会导致模型泄漏原生工具调用格式，修复方式是"充裕预算+专门合成轮+清洗泄漏格式"。

## 第6节：自我融合

Opus两次自融合+6.9（错误不相关，能互补）；MiniMax-M3两次自融合+0.0（错误高度相关）；但M3跑到10次能涨到69.4——"数量可以替代跨模型多样性"；爬坡曲线2次没用、4次开始爬升、7次到顶；成本4次$37/10次$87，对比Fable-5模型化~$250。

## 第7节：融合是两份工作（判官 vs 合成）——73.4分怎么来的

- **73.4分SOTA**：judge×synthesizer全排列网格，最佳格=Kimi-K2.6判官→GLM-5.2合成，超过之前"固定gemini判官、只换合成器"测出的71.6。网格跨度48.7-73.4。
- GLM-5.2最会写但最不会判自己写的东西（系统性低估自己）。
- 全开源委员会+最佳合成器=69.2，打平Fable-5+GPT-5.5(69.0)。
- 开源面板上合成器几乎不影响分数（都在~69附近），frontier面板上差8分——"合成器只有在面板足够多样时才有价值"。
- 消融：单独去掉任何一个成员，只有MiniMax-M3(-3.9)和DeepSeek(-2.1)显著掉分；但同时去掉两个"看似多余"的成员（Kimi+GLM）反而显著掉分(-3.4)——"冗余是共享的，不是某个成员多余"。
- 相关性矩阵：5个面板成员两两相关系数集中在0.47-0.71（均值0.56），没有哪个模型特别"独立"，多样性是弥散在整个面板里的，不集中在某个"英雄模型"身上。
- 评分基建：Sonnet-4.6 chunk-of-3可以当gemini-3.1-pro的免费替代裁判（偏差≈0，相关系数0.92）。

## 第8节：Haiku/Sonnet自融合——合成器强弱决定自融合是否有效（含自我修正过程）

- 最早8题试点：Haiku自融合完全没用甚至更差（-3.5，Needle-in-Haystack任务从87.1暴跌到63.0）。
- 换Sonnet-4.6当合成器：明显有效（+4.4，Needle任务只轻微降到82.2，没崩）。
- **样本量扩大后的自我修正**：Haiku的收益从8题时的-3.5，修正到26题的+1.5，再到44题的+2.6（勉强不显著，95%CI [-0.31, +5.43]）；**Sonnet在23题上是显著的+8.02（CI [+4.60, +11.24]，明确排除0）**。
- 最终结论：**自融合对两个模型都有帮助，但强合成器的收益约是弱合成器的10倍，且只有强合成器这条收益是统计显著确立的**；文档自己承认"最早Haiku没用"是小样本误判。

---

# 三、LESSONS.md 核心结论（方法论教训，按原文档分类）

## 3.1 方法论

- **Agentic评测必须用真实工具**：一次性给冻结资料会低估~20分，深度研究类评测必须让模型自己跑`web_search`/`web_fetch`/`bash`循环。
- **裁判模型和打分轮次都要完全对齐才能比较**：1次打分比多次打分噪声更大，重新打分的每个配置必须用完全相同的设置，不能拿"新跑出来的分数"直接比"用不同方式打分的已发布数字"。
- **小样本切片不等于完整数据集的结论**：10-15题的切片不代表已发布的100题数字；曾经错误地把一个切片判断为"更难"，实际上并不是。
- **每次跑完都要审计泄题**：把所有搜索词和抓取URL倒出来，搜数据集/评分细则/答案相关的关键词；按评分细则本身的内容（criterion id、每条要求的前几个词、禁用词）过滤工具结果，屏蔽数据集/论文所在域名——但不要把合法的信源域名（比如arxiv）也一并屏蔽。

## 3.2 网关/供应商的坑（多轮工具调用要逐个供应商测试！）

- **DeepSeek（OpenAI兼容）在工具结果之后返回空内容**——根源是网关做了OpenAI→Anthropic→OpenAI的转译，把Anthropic的`tool_use`/`tool_result`格式原样传给了OpenAI接口，需要反向转译。
- **Vertex/Gemini完全不支持工具调用**——工具没有被发送、工具历史也没有被转译，需要做OpenAI↔Gemini的`functionDeclarations`/`functionCall`/`functionResponse`双向转译。
- **Gemini 3要求`thought_signature`原样回传**——每次`functionCall`都会返回一个不透明的签名，下一轮必须原样带回去，不然报400错误；OpenAI的`tool_calls`格式里没有专门字段装这个签名，只能藏在`tool_call`的`id`字段里。
- **结论**：正式信任任何多轮工具调用结果之前，一定要针对每个供应商单独跑一次"2轮工具对话"测试。

## 3.3 Harness设计

- 不要"硬性截断工具调用次数、然后突然逼模型回答"——会导致模型把原生工具调用格式泄漏进正文内容（DeepSeek的`<｜｜DSML｜｜>`、Kimi/Claude的`<invoke>`），或者反复说"让我继续"，或者干脆截断——解决方案是"充裕的研究预算+专门的合成轮（不给工具，明确说"现在就写最终报告"）+把泄漏的格式清洗掉当兜底"。
- 对"不爱主动用工具"的模型（比如gemini-flash，倾向于凭记忆直接回答），要在第一轮强制`tool_choice:"required"`。
- bash工具：本地Docker容器（`python:3.12-slim`，`--network none`）就够用了。
- **每一次replay和报告都要存下来，生成过程本身也要写成可保存复用的脚本**——曾经有一次产出关键结果的一次性操作没有保存下来，导致后来要花一整套流程重新推导一遍。

## 3.4 大规模打分（chunk-of-3、免费替代裁判、subagent并发）

- **打分时的分块大小和顺序，会实质性改变最终分数，不只是影响成本**：按3条一批（chunk-of-3，每批单独一次裁判调用）vs 一次性全部评分细则一起打分，数字会变——chunk-all平均虚高约+7分，最坏情况+15分；先给答案再给评分标准（顺序颠倒）会虚高+4-5分。**对任何要拿去跟参考分数比较的结果，必须固定用chunk-of-3、且评分标准要放在答案之前**——没有省钱的捷径，正确的打分方式本身就是贵的。
- **换裁判模型之前必须先验证校准度**：付费额度用完后，换成不计费的Claude subagent当裁判，换之前先在一部分共享数据上验证——Sonnet-4.6用chunk-of-3打分，跟gemini偏差≈0、相关系数0.92，可用；Opus用chunk-all是+4.3偏差/相关系数只有0.59；Haiku用chunk-of-3是-5.2偏差/相关系数0.83——**同样的数据，换个裁判方式，结果差异很大**。换裁判时要完整复刻参考裁判的原始prompt（系统提示文字+消息顺序），不能只是意思差不多的改写。
- **皮尔逊相关系数不受"每个评委固定偏移量"的影响**——所以做相关性/结构分析时，混用两个裁判是安全的，即便其中一个有固定的小偏移；但**如果是要比较绝对分数高低（排行榜类比较），就不能混用裁判**，必须用同一个裁判重新打分。
- **subagent并发打分要设上限、要控制节奏**：踩过两个坑——①一个用`budget.remaining()`做循环条件、但没设token预算的循环，遇到预算是`Infinity`时会一直跑到系统硬性上限（1000个agent），白白烧掉~900万token；②即便设了上限（约685个agent、约14并发），也会被服务端限流（"临时限制请求"，是临时性的，不是用量超限）、丢失大量调用结果——**解决方式是分成更小批次、按顺序（不要并行）跑完一批再跑下一批，并且循环重跑那些没跑完的，直到全部打完分**。
- **Workflow工具返回结果的解析细节**：输出文件结构是`{summary, agentCount, logs, result}`，`result`字段可能是**双重编码**的JSON字符串，需要用`while isinstance(res, str): res = json.loads(res)`这种循环解码方式；每个agent最好只返回精简的结果行，用"输入文件名"而不是"agent自己返回的内容"去重新对应任务身份（比如任务序号、config_id）。

---

# 四、三份材料交叉对照，几处值得注意的地方

**1. 泄题审计的数字，README.md和FINDINGS.md对不上，不是同一批统计口径**：
- `FINDINGS.md`第2节写的是"568次web_search + 155次web_fetch"；
- `README.md`写的是"12,704次web_searches + 5,390次web_fetches"。
- 两者相差约22倍（搜索次数）和35倍（抓取次数）。**大概率是统计的实验范围不一样**——`FINDINGS.md`第2节写于项目较早期，可能只统计了当时已完成的那几组实验（budget fusion、fuser消融）；`README.md`作为面向读者的最终概览，很可能统计了后续陆续追加的全部实验（自我融合、开源委员会、judge×synth网格等）累积下来的总量。**这一点文档本身没有解释，是我们对照出来的差异，不是原文自己标注过的。**

**2. Fable-5从未被独立跑过**——`README.md`solo对照表明确标注"(not run)"，这一点`FINDINGS.md`正文没有专门强调过，是仅在`README.md`里才看到的披露。之前几轮汇报里提到的"Fable-5 solo 65.3"，任何场合下都应该说明这是"引用OpenRouter自己的数字，不是TrustedRouter自己复现的"。

**3. Haiku/Sonnet自融合的数字，README.md版本落后于FINDINGS.md最终版本**——`README.md`里写的是"n=26时Haiku+1.5、Sonnet n=4时+4.4"，这对应`FINDINGS.md`第8节里"Update"那一版（不是最终版）；`FINDINGS.md`后面还有"Update 2"，把Haiku扩到n=44（+2.62，勉强不显著）、Sonnet扩到n=23（+8.02，显著），**这是更新更全的数据，`README.md`没有同步更新到这一版**。引用这组数字时，应该以`FINDINGS.md`的"Update 2"为准，不是`README.md`表格里的数字。

**4. 73.4分这个后来居上的SOTA数字，没有出现在README.md的头条结论里**——`README.md`的头条只提到71.6分（"state of the art"），完全没提73.4分（judge×synth网格测出的更高分，来自`FINDINGS.md`第7.1节）。**这说明`README.md`这份面向读者的概览文档，写作时间早于`FINDINGS.md`第7节的判官×合成网格实验，两份文档没有完全同步更新**，如果只看`README.md`会误以为71.6分才是最高分。
