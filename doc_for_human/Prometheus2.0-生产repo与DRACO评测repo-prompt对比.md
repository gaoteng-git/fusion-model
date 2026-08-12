# Prometheus 2.0：生产repo与DRACO评测repo的Prompt原文对比

逐字核对了面板/判官/合成三个阶段的prompt，两边仓库都翻了原文，一字不改地摆出来对比。

- 生产repo：`quill-cloud-proxy`（真实用户调用`trustedrouter/prometheus-2.0`时实际执行的代码）
- DRACO评测repo：`TrustedRouter-Fusion-Draco`（研究团队用来跑出77.4等分数的评测代码）

---

## 面板阶段（Panel）

**生产repo（`quill-cloud-proxy/fusion.go`，第2424-2431行）**：
```
basePrompt := ""
if len(builtInPrompt) > 0 {
    basePrompt = strings.TrimSpace(builtInPrompt[0])
}
if basePrompt == "" {
    basePrompt = "Answer the request independently and return only the visible answer."
}
system := fmt.Sprintf("You are TrustedRouter Synth panel member %d.\n\n%s", index+1, basePrompt)
```
https://github.com/Lore-Hex/quill-cloud-proxy/blob/129c54b01ce478540dbb4c81f64cc4943c847d98/enclave-go/cmd/enclave/fusion.go#L2424-L2431

**DRACO评测repo（`TrustedRouter-Fusion-Draco/src/trusted_router/evals/agentic_tools.py`，第138-148行，`DRACO_AGENTIC_SYSTEM_PROMPT`）**：
```
"You are a deep research analyst. Answer the user's research task with a
complete, source-grounded report. You have three tools: web_search (find
sources), web_fetch (read a specific URL in full), and bash (run shell /
python for any calculation or data manipulation). Search iteratively: start
broad, then fetch the most authoritative primary sources and verify key
figures with bash. Cite source URLs inline. Show quantitative work explicitly
and state uncertainty plainly. Do not mention benchmark rubrics. When you have
enough evidence, write the final report as plain text with no further tool calls.
Your final report must contain only the report itself — no planning, reasoning
narration, or scratchpad text."
```
https://github.com/Lore-Hex/TrustedRouter-Fusion-Draco/blob/2c3ff11f3287dbd5d8f4d37a491e18024906ae75/src/trusted_router/evals/agentic_tools.py#L138-L148

---

## 判官阶段（Judge）

**生产repo（`fusion.go`第2457行）**：
```
"You are the TrustedRouter Synth judge. Compare panel responses and return
compact JSON with keys consensus, contradictions, partial_coverage,
unique_insights, blind_spots, and final_guidance. Do not write the final
answer. Return only JSON; do not include chain-of-thought, hidden reasoning,
or <think> blocks."
```
https://github.com/Lore-Hex/quill-cloud-proxy/blob/129c54b01ce478540dbb4c81f64cc4943c847d98/enclave-go/cmd/enclave/fusion.go#L2457

**DRACO评测repo（`draco_client_fusion.py`第34-40行，`JUDGE_SYSTEM`）**：
```
"You are the TrustedRouter Fusion judge. Compare panel responses and return
compact JSON with keys consensus, contradictions, partial_coverage,
unique_insights, blind_spots, and final_guidance. Do not write the final
answer. Return only JSON; do not include chain-of-thought, hidden reasoning,
or <think> blocks."
```
https://github.com/Lore-Hex/TrustedRouter-Fusion-Draco/blob/db99682172e7b788b76754b70ecdb049646728b5/scripts/draco_client_fusion.py#L34-L40

---

## 合成阶段（Synthesis）

**生产repo（`fusion.go`第2477-2479行，默认值）**：
```
instruction := strings.TrimSpace(config.BuiltInFinalPrompt)
if instruction == "" {
    instruction = "Use the panel answers as evidence and the judge analysis as guidance to answer the original request. Return only the final visible answer."
}
```
https://github.com/Lore-Hex/quill-cloud-proxy/blob/129c54b01ce478540dbb4c81f64cc4943c847d98/enclave-go/cmd/enclave/fusion.go#L2477-L2479

**DRACO评测repo（`draco_client_fusion.py`第41-47行，`FINAL_INSTRUCTION`）**：
```
"TrustedRouter Fusion panel answers and judge analysis follow. Use the panel
answers as the primary evidence and the judge analysis as guidance to write
the final answer for the original request. Return only the final visible
answer. Do not include chain-of-thought, hidden reasoning, analysis,
scratchpad text, <think> blocks, or internal model names unless the user
asked for methodology."
```
https://github.com/Lore-Hex/TrustedRouter-Fusion-Draco/blob/db99682172e7b788b76754b70ecdb049646728b5/scripts/draco_client_fusion.py#L41-L47

---

## 对比分析

| 阶段 | 是否一致 | 具体差异 |
|---|---|---|
| **面板** | ❌ **完全不同**，不是措辞差异，是两套不同的东西 | 生产版是通用兜底句，不预设研究场景、不提工具；DRACO版整段是"深度研究分析师+三个具体工具（web_search/web_fetch/bash）+怎么用工具"的完整方法论指令。**生产环境的面板阶段本来就不会拿到这段prompt**——除非调用方自己在请求里传入等效的`builtInPrompt`，否则用户真实调用`trustedrouter/prometheus-2.0`时，面板成员看到的就是那句通用兜底句。 |
| **判官** | ✅ **几乎逐字一致** | 唯一区别是"TrustedRouter **Synth** judge"（生产）vs "TrustedRouter **Fusion** judge"（DRACO评测），一个改名遗留的词，JSON字段、指令措辞、"不要输出思维链"这条都完全一样。 |
| **合成** | ⚠️ **思路一致，但不是逐字复制** | 生产版是一句短的兜底指令；DRACO评测版是重写过的加长版，多了"primary evidence"的措辞、多了一整段关于不要输出思维链/scratchpad/内部模型名的补充指令。核心意思（面板答案当证据、判官分析当参考、只出最终可见答案）没变，但字面上不是同一份文本。 |

## 结论

**三段里只有判官阶段基本对得上，面板阶段完全是两套东西，合成阶段介于两者之间**。最关键的是面板阶段这条——DRACO评测用来产生77.4分（以及其它所有DRACO分数）的面板prompt，跟真实用户调用`trustedrouter/prometheus-2.0`时面板实际收到的prompt，压根不是同一份文本，一个专门为"deep research + 三工具agentic loop"设计，一个是什么场景都能用的兜底句。这进一步印证了此前的结论：**DRACO评测衡量的是"照着生产代码的判官/合成prompt、但换了一套完全不同面板方法论"跑出来的流程，不是"用户真实调用产品会得到什么"。**

（补充说明：`quill-router`仓库里还有第三个版本的面板prompt，在`src/trusted_router/evals/fusion_live.py`的`panel_messages`函数里，是"面板成员+深度研究基准测试+已提供检索资料"的框架，假设模型拿到的是预先抓取好的资料而非实时工具——但这个文件从未被生产代码或`draco_client_fusion.py`调用过，是仓库里第三份"存在但未被使用"的prompt，如果老板问起，可以说明这一点，避免以为只有两个版本。）
