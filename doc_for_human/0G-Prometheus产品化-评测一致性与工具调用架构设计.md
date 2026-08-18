# 0G Prometheus 产品化：评测一致性问题与工具调用架构设计

本文档汇总一次连续讨论的思考、分析与结论，主题是：**TrustedRouter"评测代码"与"生产代码"不一致的问题**，以及基于此教训，**0G自研fusion模型产品化时应该如何设计评测方法论和工具调用架构**。

---

## 第一部分：TrustedRouter的问题——评测与生产代码不一致

### 1.1 问题描述

通过对`quill-cloud-proxy`（TrustedRouter真实线上attested gateway，已通过`trust.trustedrouter.com`实时attestation验证）和`TrustedRouter-Fusion-Draco`（DRACO评测复现仓库）两份代码的逐段代码级核查，确认了以下事实：

**生产环境`trustedrouter/prometheus-2.0`（`quill-cloud-proxy`，`enclave-go/cmd/enclave/fusion.go`）的执行流程：**

```
用户请求 → maybeServeFusion → serveFusionNonStreaming
  → 面板阶段：5个模型（MiniMax M3、Kimi K3、GLM 5.2、DeepSeek V4 Pro、MiMo V2.5 Pro）
             并发起飞，每个成员只调用一次（runFusionPanelObserved）
             工具：100%来自调用方自己的req.Tools，网关不自动注入任何工具；
             即使模型这一轮吐出tool_calls，代码也不会执行、不会有第二轮
  → 判官阶段：1次调用，无工具（out.Tools = nil），强制JSON输出
  → 合成阶段：1次调用，工具跟随调用方req.Tools（同面板），但同样只调用一次
  → 组装响应返回给调用方
```

关键代码证据（`fusion.go`）：
- `runFusionPanelObserved`：`go func(...) { runFusionCallObserved(...) }`——每个面板成员的goroutine里只有一次模型调用，没有"看结果决定要不要再调一次"的循环。
- `runFusionCallValidatedObservedAttempt`：核心执行只有`invokeProviderStream` + `CollectAnthropicTextWithObserver`两行，不检查`result.ToolCalls`是否非空、不执行工具、不发起第二轮。
- `fusionJudgeRequest`：`out.Tools = nil`，判官阶段永远无工具。
- `fusionPanelEvidence`/`fusionToolCallsText`：面板成员的工具调用请求会被转成一段可读文字塞进证据，**从未被真正执行**。

**DRACO评测77.4/73.4等发布分数对应的执行流程（Path B：`draco_agentic_solo.py`×5 + `draco_client_fusion.py`）：**

```
5个面板模型，各自独立通过 draco_agentic_solo.py 运行
  → 每个模型真正执行工具（web_search / web_fetch），可以多轮迭代
     （看到搜索结果 → 决定继续搜索还是收尾 → ... 直到主动停止或撞到轮数上限）
  → 5份"深度研究"最终答案 → draco_client_fusion.py：判官分析1次、合成1次
```

**核心结论：判官/合成阶段的"拓扑"其实两边一样（各跑一次），真正的、根本性的差异全部发生在"5个面板成员的答案是怎么产生的"这一层——生产是单轮、无工具执行；评测是多轮、真实工具执行。** 这不是配置调优的差异，是代码路径级别的根本不同（`fusionPanelRequest`的单轮prompt vs `draco_agentic_solo.py`的agentic循环）。

### 1.2 为什么这个不一致是有问题的

这里需要分两层看，两层的答案并不一样：

**层面一：评测配置比生产配置更强，这件事本身合理吗？**

这是行业里非常普遍的做法，不算孤例，也不构成本身层面的诚信问题。几乎所有大模型厂商报出的SOTA分数，用的都是"能拿到的最强配置"（更长推理预算、工具访问权限、best-of-N采样等），业内对此已经见怪不怪，只是要求"报分数时说清楚用的什么配置"。单纯"评测配置强于默认产品配置"，不是这里要指出的核心问题。

**层面二：但具体到TrustedRouter这里，问题比行业惯例更严重一层**

普通行业惯例的问题模式是"产品A（默认配置）vs 产品A（评测专用增强配置）"，好歹存在某种可以区分的上下文。而TrustedRouter的具体情况更值得较真的地方在于：**`trustedrouter/prometheus-2.0`同时是（a）营销/论文里的产品名，也是（b）客户实际发起API调用时必须传入的、字面意义上的`model`字段值**。这不是"同一品牌用了两套配置"这么简单的表述问题，而是**同一个可调用的API标识符，客户实际调用后拿到的系统，和公开宣传分数所对应的系统，在代码路径级别根本不是一回事**——这不是调参数能达到的差异。

进一步验证：**生产环境的judge/synthesis阶段架构是封闭的**——`fusionConfig`结构体里没有任何"外部注入面板答案"的字段。这意味着即便客户自己写代码，先分别调用5个基础模型（绕开`prometheus-2.0`融合入口）、自己实现真正的agentic工具循环、拿到5份高质量深度研究答案，**客户也没有任何API参数能把这5份答案喂给`trustedrouter/prometheus-2.0`的判官/合成阶段**——只能自己再写一套判官+合成逻辑。也就是说，客户通过任何合法的API调用方式，都**无法**让自己实际拿到的产品体验，逼近77.4分对应的执行方式。

**这个问题可以从四个角度分别指出：**
- **不客观**：对外声称"Prometheus 2.0 拿到77.4分"，掩盖了产品API与评测配置在架构上的根本差异，呈现的信息与实际情况不符。
- **不准确**：77.4分不能代表客户调用`trustedrouter/prometheus-2.0`这个API实际会拿到的分数——两者是不同的系统，用同一个数字去描述两者，是不准确的。
- **不科学**：科学的可复现性要求"同样的配置产生同样的结果"，而这里客户用同样的model字符串、同样的合法API参数，永远无法复现宣传分数对应的执行方式——不具备可复现性。
- **不公正**：客户基于"Prometheus 2.0 拿到77.4分"这一宣传信息做采购/使用决策，实际得到的是一个架构上完全不同、能力显著弱化（无工具执行、单轮）的系统，这对客户的知情权和合理预期是不公正的。

**需要澄清的是**：TrustedRouter内部技术文档（`TrustedRouter-Fusion-Draco`仓库的README/FINDINGS/LESSONS）对方法论本身写得相当详细，研究团队并没有在内部文档里刻意隐瞒执行方式——**问题出在对外的"77.4分"这个headline宣传语境下，没有同步、显著地说明该分数对应的执行方式与生产API默认行为不等价**，这是披露缺失的问题，不是方法论本身造假的问题。

---

## 第二部分：0G提出的评测方案——完全调用产品API，多轮迭代+本地执行工具

### 2.1 方案描述

为避免重蹈"评测分数代表不了产品体验"的覆辙，0G提出的评测方案核心原则是：**评测用的就是真实生产API，不存在任何评测专用的、产品客户拿不到的执行方式**。具体协议：

1. 将0G自研的fusion模型（记为API A）和baseline模型（如gpt-5.5，记为API B）都部署为可实际调用的生产API。
2. 对每一道DRACO题目：
   - 调用API（标准OpenAI-compatible chat completions协议，带`tools`声明），得到响应；
   - 如果响应是最终文本答案 → 结束，记录该答案；
   - 如果响应是`tool_calls` → 在客户端本地真实执行该工具调用，将执行结果以标准`role: "tool"`消息追加进**结构化的多轮messages数组**（而非拼接成一段flat文本重新提问），再次调用同一个API，重复上述判断；
   - 直到拿到最终文本答案，或迭代轮数达到预设上限为止。
3. 用这套协议，把100道DRACO题目，先完整跑一遍API A，再完整跑一遍API B。
4. 用同一个判分模型、同一套评分设置（chunk-of-3、pass次数一致）分别给A、B的答案打分，取平均分对比。

### 2.2 相对TrustedRouter原方法论的核心优势

**A、B两边走的是完全相同的外部协议**——这个协议本身就是"真实用户会怎么用这个产品"，不是另外造一套评测专用harness。分数天然就是"用户真实这么用能拿到的分数"，不存在任何"分数虚高、产品拿不到"的落差。这直接消除了第一部分指出的核心问题。

### 2.3 需要在实施前钉死的技术细节（风险与建议）

1. **必须用标准结构化多轮messages，不能拼flat文本prompt**。原因两点：
   - 信息丢失风险：如果每轮只拼"题目+上一轮输出+这一轮工具结果"，模型会看不到更早几轮的历史，深度研究任务经常需要横跨多轮的信息，容易系统性压低分数，且可能对A、B造成不对称伤害。
   - Provider兼容性风险：TrustedRouter自己的LESSONS.md记录过真实踩坑——"Gemini 3 requires `thought_signature` round-trip — every `functionCall` returns an opaque signature that must be echoed back on the next turn (400 otherwise)"。如果harness不维护标准的结构化`tool_calls`/`tool_result` message block，遇到这类模型会直接出错。
2. **A、B两边的工具实现、最大迭代轮数上限、判官模型与判分设置必须完全一致**，否则会引入新的混杂变量，破坏对比的科学性。
3. **需要同时按"轮数对齐"和"成本/延迟对齐"两个口径分别报告**：fusion每一轮的成本远高于单模型baseline每一轮（面板5模型+判官+合成 vs 单模型1次调用），如果只给两边设同样的最大轮数上限，等价于允许fusion多花数倍的钱去够到同样的轮数——质量对比没问题，但如果不同时报告成本倍数，容易得出"fusion更强"却掩盖"但贵数倍"这个前提。
4. **必须做benchmark leakage审计**（照搬LESSONS.md的经验）：真实联网搜索有可能直接搜到DRACO题目原文、答案要点甚至评分rubric，需要过滤搜索结果/URL against数据集host、rubric关键词等，否则分数可能因为"抄答案"而失真。
5. **统计噪声控制**：单次跑一遍的平均分可能有几分噪声，如果A、B分差不大，建议多跑几次取平均或报告置信区间，不要凭一次跑分下结论。

---

## 第三部分：0G提出的2-API拆分设计——面向两类用户群体

### 3.1 方案描述

将"支持工具调用作为输出"和"不支持"，拆分成面向两类用户群体的两个产品身份：

- **开发者/高级用户版**（如`0g/prometheus-2.0-agent`）：json分析模型（判官）、融合模型（合成）都加入对工具调用的透传指令，允许最终响应里出现结构化`tool_calls`，供有能力自己执行工具、搭建agent的开发者/专业用户使用。
- **小白用户版**（如`0g/prometheus-2.0`）：完全不支持工具调用输出，目标用户是不懂agent、不懂工具执行的普通用户，只想调用模型做写稿、分析等纯文本任务，只使用模型的内化能力。

### 3.2 这个设计如何解决第一部分指出的"同名不同物"问题

这是本方案里价值最高的一点：**只要给这两个产品配置两个清晰不同的model ID，并分别独立评测、独立公布分数、不混用同一个benchmark结论去代表两者**，第一部分指出的"同一个API标识符代表两个不等价系统"问题就从产品设计的源头上被解决了——不需要再纠结"该不该改名"，因为它们本来就是两个名字、两个产品、两条独立的评测记录。

### 3.3 工程实现建议：一套引擎+一个硬性配置开关，而非两套物理部署

底层可以（也应该）共享同一套面板/判官/合成代码，差异收敛为配置层一个硬开关：

```go
type fusionConfig struct {
    ...
    AllowToolCallOutput bool   // 由 req.Model 决定，不受调用方任何参数影响
}

// 面板/合成阶段构造请求时：
if !config.AllowToolCallOutput {
    out.Tools = nil          // 硬性清空，不管调用方传没传tools，一律无视
    out.ToolChoice = nil
}
```

沿用现有代码里`isFusionModel`/`fusionPresetPanelForModel`"按模型ID字符串查表拿配置"的模式即可，工程上只多了一个bool字段和几处判断，不需要维护两套独立部署、两套独立监控，但对外呈现为两个清晰独立的产品身份。

### 3.4 小白版需要"硬拦截"而非"默认不给"

不建议假设"小白用户不会主动传`tools`字段所以不用管"——很多SDK/agent框架会默认给请求挂一些工具schema，小白用户自己可能都不知道。建议小白版：
- 不管调用方有没有传`tools`，一律清空；
- 如果检测到调用方明确传了`tools`参数，直接返回清晰的400错误（"this model does not support tool-calling output, use the agent variant instead"），而不是静默忽略——静默忽略会让用户误以为工具被使用了、实际上没有，属于隐藏的行为不一致。

### 3.5 两个产品必须用不同的评测集，不能都套用DRACO

**DRACO本质上是一个"深度研究"benchmark，题目设计前提就是需要真实检索/工具能力**（评测里solo baseline和面板成员都要走agentic多轮工具执行才能拿到发布分数）。拿DRACO评测"小白版"（完全不支持工具输出），结果必然大幅落后，但这不代表小白版做得差，而是DRACO这把尺子对没有工具的模型天然不公平。

- **开发者版**：用DRACO，配合第二部分的"真实API+client执行工具+多轮迭代"评测方式。
- **小白版**：应该换一套不需要联网/工具的评测集（写作质量、常识问答、纯推理类benchmark），公平衡量它"内化能力"这个卖点。

---

## 第四部分：工具透传的具体实现——面板/判官/合成三阶段的prompt与代码设计

本节仅适用于**开发者版**（`AllowToolCallOutput = true`）。核心原则：**不在服务端执行任何工具**，网关角色始终是"传话+翻译"，真正执行工具、决定要不要继续，是调用方（客户端/开发者自己的agent代码）的责任——与生产环境现有的架构原则保持一致，只是在"判官/合成该如何处理面板成员的工具调用信息"这一点上做针对性增强。

### 4.1 面板阶段（沿用现有单轮设计，不新增执行逻辑）

```go
func fusionPanelRequest(req *types.OpenAIChatRequest, model string, index int, ...) *types.OpenAIChatRequest {
    out := cloneChatRequest(req)
    out.Tools = stripFusionToolEntries(req.Tools)   // 工具100%来自调用方，不自动注入
    system := fmt.Sprintf("You are TrustedRouter Synth panel member %d.\n\n%s", index+1, basePrompt)
    if len(out.Tools) > 0 {
        system += "\n\nIf the next correct step is a provided function call, emit the tool call directly instead of describing it."
    }
    out.Messages = prependSystem(req.Messages, system)
    return out
}
```

- 面板阶段维持**单轮、无服务端执行**：每个成员依然只调用一次上游模型，如果模型这一轮判断"应该调用工具"，就直接吐出结构化`tool_calls`，代码不执行、不追问，原样记录进`fusionCallResult`。
- 不新增任何多轮/agentic循环——这是与第一部分"评测配置"（`draco_agentic_solo.py`）刻意保持区别的地方：产品API的面板阶段始终轻量、可预测、成本可控，深度研究能力靠"客户端多轮调用整个API"（第二部分方案）来实现，不在面板内部实现。

### 4.2 判官阶段（新增：让判官"看懂"面板成员的工具调用意图，并给出采纳建议）

现有判官JSON schema（`consensus`/`contradictions`/`partial_coverage`/`unique_insights`/`blind_spots`/`final_guidance`）需要扩展一个新字段：

```json
{
  "consensus": "...",
  "contradictions": "...",
  "partial_coverage": "...",
  "unique_insights": "...",
  "blind_spots": "...",
  "final_guidance": "...",
  "recommended_tool_call_index": null   // 新增：1~N的整数，或null
}
```

判官prompt新增指令：*"如果5个面板成员中，有人认为当前最优的下一步行动是调用某个工具，且你认为这个判断是合理的、应当被采纳为最终输出，请在`recommended_tool_call_index`里填写该面板成员的编号；否则填null，交由合成阶段正常综合出文本答案。"*

- **判官读到的证据格式也需要相应更新**：`fusionPanelEvidence`/`fusionToolCallsText`目前只是把工具调用转述成一段自然语言文字（"想调用工具X，参数Y"）喂给判官/合成——这个转述本身可以保留（判官需要"读懂"每个面板成员想干什么），但**判官对某个面板成员的"采纳"决策，不能靠judge自己重新生成/复刻这个工具调用**（模型复述参数容易出错、走样），而是**只输出一个索引号**。
- **调度层收到判官的`recommended_tool_call_index`后，直接从原始`panel[]`数组里按索引原样取回该面板成员未经改写的`fusionCallResult`**（包括其原始、完整、未被复述过的`ToolCalls`结构体），作为最终响应返回——这个"按索引原样透传"的机制，正是复用了production代码里现有的`selector`策略（`combo.go`的`runSelectorDecision`：`selected := panel[parsed.SelectedIndex-1]`）已经验证过的做法，保真度100%，不存在"模型转述工具调用参数时手抖出错"的风险。
- 判官如果推荐了某个索引，调度层仍需做一次有效性校验（复用`fusionPanelResultUsable`这类兜底逻辑）——确认该面板成员的结果不是失败/空占位文案、且确实包含非空`ToolCalls`，校验不通过则回退到正常合成路径，不能盲目采纳。

### 4.3 合成阶段（两条分支：原样采纳 vs 正常文本合成）

```go
// 调度层伪代码
if judge.RecommendedToolCallIndex != nil {
    candidate := panel[*judge.RecommendedToolCallIndex - 1]
    if fusionPanelResultUsable(candidate) && len(candidate.Result.ToolCalls) > 0 {
        return candidate   // 分支一：原样透传该面板成员的工具调用，不再进入合成阶段
    }
}
// 分支二：正常走 runFusionFinal 文本合成
```

**分支二（正常合成）的prompt需要新增格式校验指令**：

```
Use the panel answers as evidence and the judge analysis as guidance to answer the original request.
If you decide to output a tool call, ensure the tool call is valid and schema-compliant,
and that any accompanying visible text does not conflict with the tool call
(return visible text only when no tool call is needed).
```

- 这条prompt指令能降低"模型同时输出矛盾的文本+工具调用"的概率，但**不能只靠prompt自觉**——LLM在"文本与结构化输出语义一致"这类复合指令上的遵循度并不稳定。建议**同时在代码层面加确定性兜底**：

```go
// 代码层面的确定性规则，不完全依赖模型自觉
if len(result.ToolCalls) > 0 && strings.TrimSpace(result.Text) != "" {
    // 明确规则：有tool_calls时，finish_reason强制标记为tool_calls，
    // text字段仅作为可选的辅助说明保留，不能让客户端把text误认为是最终答案
}
```

- **一次响应只能采纳一个工具调用**：如果5个面板成员提出了不同的工具意图（比如一个想搜索，一个想读取某URL），判官/合成也只能二选一（或都不选、走文本合成），这个机制不能"合并"多个不同的工具调用意图，这是需要向内部/文档说明清楚的能力边界。

### 4.4 小白版的对应处理（第三部分3.4节的具体落地）

小白版（`AllowToolCallOutput = false`）在4.1~4.3的每一处请求构造函数里，统一加上硬性清空：

```go
if !config.AllowToolCallOutput {
    out.Tools = nil
    out.ToolChoice = nil
}
```

且judge schema不启用`recommended_tool_call_index`字段（面板阶段既然没有工具，也就不可能产生任何`ToolCalls`，这个字段没有存在意义）——小白版本质上是开发者版在"工具能力"维度上的一个纯净子集裁剪，不需要维护两套不同的判官/合成核心逻辑。

---

## 总结：一张表看清四部分的关系

| 部分 | 核心结论 |
|---|---|
| 第一部分 | TrustedRouter同一API model字符串代表两个架构不等价的系统，且未披露方法论差异——不客观、不准确、不科学、不公正 |
| 第二部分 | 0G评测方案：完全用生产API+client执行工具+多轮迭代，消除"评测分数≠产品体验"的落差 |
| 第三部分 | 0G产品设计：拆成开发者版（支持工具透传）+ 小白版（纯内化能力），从源头解决"同名不同物"问题，且需配套不同评测集 |
| 第四部分 | 工具透传的具体实现：面板单轮不执行、判官新增"推荐采纳索引"字段（复用selector的按索引原样透传机制保真度100%）、合成阶段prompt指令+代码层确定性兜底双重保障 |
