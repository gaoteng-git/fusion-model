# DRACO评测代码逐段讲解（阶段1-4）

本文档对应仓库：`TrustedRouter-Fusion-Draco`（评测主体代码）+ `quill-router`（任务加载/常量定义，评测代码依赖它）。按实际执行顺序，分四个阶段讲解，每段贴代码原文+逐句解释+关键变量含义。

---

## 入口函数：没有统一入口，是4个独立的CLI脚本，需要人工依次手动执行

**跟生产代码不一样，这份评测代码不存在一个"发一次请求，内部自动跑完全流程"的单一入口函数**。核实过仓库里没有Makefile、没有shell脚本把下面几步自动串起来——**是4个独立的Python命令行脚本，对应阶段2-4，每个脚本自己的`main()`就是各自的入口**，需要研究者手动一步一步跑，把上一步的输出文件路径当参数传给下一步：

| 阶段 | 入口函数 | 文件 | 用途 |
|---|---|---|---|
| 2 | `main()`，第41行；`if __name__ == "__main__": raise SystemExit(main())`，第254行 | `scripts/draco_agentic_solo.py` | 对某一个面板/solo模型，批量跑完100道题的agentic研究循环，写出`replays/solo-<模型>.jsonl` |
| 3 | `main()`，第92行；入口第319行 | `scripts/draco_client_fusion.py` | 读取上一步产出的若干份replay当"面板"，跑判官+合成，写出融合后的replay文件 |
| 4a | `main()`，第27行；入口第271行 | `scripts/draco_rejudge.py` | 读取上一步的融合结果，用固定裁判模型打分 |
| 4b | `main()`，第11行；入口第55行 | `scripts/draco_report.py` | 汇总打分结果，算出最终分数、跟OpenRouter数字对比，出报告 |

**这一点本身也是"77.4分能不能复现"这条线上的一个佐证**：既然连TrustedRouter自己的评测代码都没有一键跑全流程的入口，需要研究者手动敲4次命令、每次都要正确传对`--model`/`--panel`/`--manifest`这些参数，那么"这次跑Kimi-K3+MiMo-V2.5-Pro这个新面板到底具体敲了哪几条命令、传了什么参数"，本来就没有代码层面的强制记录，全靠研究者自己在博客/文档里说清楚——而这恰恰是`TrustedRouter-Fusion-Draco`仓库和77.4分博客都没做的事。

---

## 阶段1：任务加载

**文件**：`quill-router/src/trusted_router/evals/draco.py`

```python
DRACO_DATASET = "perplexity-ai/draco"
DRACO_CONFIG = "default"
DRACO_SPLIT = "test"
DRACO_TASK_COUNT = 100
DRACO_DATASETS_SERVER_URL = "https://datasets-server.huggingface.co/rows"
```
- **`DRACO_DATASET`**：DRACO数据集在HuggingFace上的路径，`perplexity-ai/draco`——说明DRACO这个评测集是Perplexity AI发布的，不是TrustedRouter或OpenRouter自己的。
- **`DRACO_TASK_COUNT = 100`**：固定100道题，全文档反复出现的"100题"就是这个常量。
- **`DRACO_DATASETS_SERVER_URL`**：通过HuggingFace官方的datasets-server API拉取数据，不是直接下载文件。

```python
@dataclass(frozen=True)
class DracoTask:
    id: str
    domain: str
    problem: str
    rubric: dict[str, Any]
```
- **`DracoTask`**：一道题的数据结构。`id`是题目唯一标识；`domain`是题目所属领域（比如"finance"、"law"，之前提过的"排除财务题"就是按这个字段过滤）；`problem`是题目正文；`rubric`是评分细则（一个JSON，包含~39条加权评分标准，模型和面板永远看不到这个字段，只有打分阶段的裁判模型能看到）。

```python
def filter_draco_tasks(
    tasks: tuple[DracoTask, ...], *, task_filter: DracoTaskFilter
) -> tuple[DracoTask, ...]:
    if task_filter == "all":
        return tasks
    if task_filter == "non-financial":
        return tuple(task for task in tasks if task.domain.lower() != "finance")
```
- **`filter_draco_tasks`**：这就是之前提过的"排除财务题"开关。`task_filter="non-financial"`时，过滤掉`domain=="finance"`的题目，得到80题子集（100题里20题是财务类）。官方操作指引推荐高成本实验先用这个子集。

```python
DRACO_EXCLUDED_SEARCH_DOMAINS: tuple[str, ...] = (
    "huggingface.co",
    "datasets-server.huggingface.co",
    "openrouter.ai",
)
```
- **防作弊域名黑名单**：面板模型搜索/抓取时，这几个域名会被屏蔽，防止模型直接搜到DRACO数据集本身或OpenRouter的评测文章，从而"抄答案"。

---

## 阶段2：面板成员各自研究（agentic多轮循环）——**这是实际产出所有DRACO分数所用的代码**

### 2.0 一个容易搞混的地方：产出"面板"其实有两条完全不同的路，融合模型（Prometheus 2.0这种）走的是"路径B"

**"面板"（5个模型各自的答案）不是靠一个专门的"面板生成脚本"一次跑出来的，而是有两条截然不同的路径可以产出，仓库里都存在，不要混：**

**路径A——真调用生产的融合接口**（`scripts/draco_native_fusion_gen.py`）：
```python
"""Generate native trustedrouter/fusion DRACO replays with explicit token caps.
...It POSTs the native ``trustedrouter/fusion`` tool (panel + judge + synthesizer
run server-side in the attested gateway)..."""
...
from ...fusion_live import (
    panel_messages,
    ...
)
```
- 这个脚本是**真的把请求发给生产环境**，面板+判官+合成都在`fusion.go`里服务器端一次性跑完，跟真实用户调用`trustedrouter/prometheus-2.0`走的是同一套生产代码。
- 但它用的面板prompt是从`fusion_live.py`导入的`panel_messages`——**就是之前确认过"已废弃、从未产出过真实高分"的那个"统一检索一次、共享给全部面板"的死代码路径**。
- 这条路径对应的正是"~40分"那个低分结果，**不是**71.6/73.4/69.2/77.4这些数字的来源。

**路径B——`draco_agentic_solo.py` × N次 + `draco_client_fusion.py`**（这才是产出所有已发布高分的方法）：
- `draco_client_fusion.py`自己的docstring原话："the panel == our validated tooled solos (gemini-flash + kimi + deepseek, full-100)"——**"面板"直接等于好几份"solo"跑出来的结果拼在一起**。
- 具体做法：**对5个面板模型，各自单独跑一次`draco_agentic_solo.py`（只改`--model`参数），产出5份独立的`replays/solo-<模型>.jsonl`文件**，再把这5个文件路径传给`draco_client_fusion.py`的`--panel model_label=replay_path`参数（可重复传，每传一次对应一个面板成员）。
- **脚本名字里的"solo"，指的是"这一次调用只跑一个模型自己走agentic循环"（不是5个模型混在一次调用里协作），不是"这个脚本只能用来产出单模型对照组"**——同一份"solo"输出，下游既可以直接报成独立对照组分数，也可以跟另外4份拼在一起走判官+合成，机制上是完全一样的东西，区别只在"写完之后拿去干什么"。
- **Prometheus 2.0这种融合模型，如果要用这份代码复现，走的是路径B**：5个面板模型（MiniMax-M3、Kimi-K3、GLM-5.2、DeepSeek-V4-Pro、MiMo-V2.5-Pro）各跑一次`draco_agentic_solo.py`，产出5份solo文件，再喂给`draco_client_fusion.py`。

### 2.1 三个工具的实现

**文件**：`TrustedRouter-Fusion-Draco/src/trusted_router/evals/agentic_tools.py`

```python
def make_web_search(task: DracoTask, exa_client: ExaSearchClient) -> Callable[[dict[str, Any]], str]:
    def run(args: dict[str, Any]) -> str:
        query = str(args.get("query") or "").strip()
        ...
        bundle: ExaSearchBundle = exa_client.search_with_contents(
            query, exclude_domains=DRACO_BLOCKED_DOMAINS, num_results=num_results,
        )
        kept = [r for r in bundle.results if not _draco_search_result_leak_reason(task, r)]
        ...
    return run
```
- **`make_web_search`**：真正调用Exa搜索API（一个第三方搜索引擎API，不是Google）。`exclude_domains=DRACO_BLOCKED_DOMAINS`屏蔽泄题域名。`_draco_search_result_leak_reason`是内容级别的二次过滤——即使搜索结果没被域名黑名单挡住，也会再扫一遍内容里有没有"评分细则关键词/题目原文片段"这类泄题特征，命中就丢弃这条结果。这是`make_web_search`返回的一个**闭包函数**（closure），绑定了当前这道`task`，所以每次搜索都能拿当前题目去做泄题检查。

```python
def make_web_fetch(task: DracoTask, *, doc_parser: str = "llamaparse") -> Callable[[dict[str, Any]], str]:
    def run(args: dict[str, Any]) -> str:
        url = str(args.get("url") or "").strip()
        if _url_is_blocked(url):
            return "Error: that domain is blocked for this task."
        text: str | None = None
        if doc_parser == "llamaparse":
            text = _llamaparse_fetch(url, DEFAULT_FETCH_CHARS)
        if not text and doc_parser in ("llamaparse", "markitdown"):
            text = _markitdown_fetch(url, DEFAULT_FETCH_CHARS)
        if not text:
            text = fetch_result_text(url, max_chars=DEFAULT_FETCH_CHARS)
        ...
    return run
```
- **`make_web_fetch`**：抓取一个具体网页/文件的正文。**`doc_parser`参数决定用哪种文档解析链路**——`llamaparse`（付费高精度表格解析，先查本地缓存`LLAMAPARSE_CACHE_DIR`避免重复计费）→失败则退到`markitdown`（开源库，把PDF/表格转成markdown）→再失败退到最朴素的纯文本抓取（`fetch_result_text`）。**这条解析链路的选择，直接影响财务类题目（大量PDF/SEC文件）的表现**——`FINDINGS.md`里提过财务题分数普遍低15-17分，跟文档解析能力不够强有关。

```python
def make_bash(*, image: str = DEFAULT_BASH_IMAGE, timeout_seconds: float = 30.0) -> Callable[[dict[str, Any]], str]:
    def run(args: dict[str, Any]) -> str:
        proc = subprocess.run(
            [docker, "run", "--rm", "--network", "none", "--memory", "512m",
             "--cpus", "1", "--pids-limit", "256", "-w", "/work", image, "bash", "-lc", command],
            capture_output=True, text=True, timeout=timeout_seconds,
        )
```
- **`make_bash`**：真的起一个Docker容器（`python:3.12-slim`镜像）跑模型给的shell命令，**`--network none`表示这个容器没有网络**——模型不能借着"跑代码"这个由头联网搜集额外信息，只能做纯计算/数据处理。

### 2.2 核心循环：`run_agentic_completion`

```python
def run_agentic_completion(..., max_tool_calls: int = DEFAULT_MAX_TOOL_CALLS, ...) -> AgenticResult:
    messages: list[dict[str, Any]] = [
        {"role": "system", "content": system_prompt},   # = DRACO_AGENTIC_SYSTEM_PROMPT
        {"role": "user", "content": user_prompt},        # = "Research task:\n{题目正文}"
    ]
    tool_calls_made = 0
    steps = 0
    while tool_calls_made < max_tool_calls:
        steps += 1
        body = _completion_body(
            model, messages, tools=tool_schemas,
            tool_choice="required" if (force_first_tool and tool_calls_made == 0) else "auto",
        )
        resp = _post_with_retry(client, url, headers, body)
        msg = choice.get("message") or {}
        tool_calls = msg.get("tool_calls")
        if not tool_calls:
            content = strip_tool_markup(msg.get("content") or "")
            if content.strip():
                return AgenticResult(content=content, ...)   # 自然写完，退出循环
            break   # 空输出，跳出循环去强制合成
        messages.append({"role": "assistant", "content": ..., "tool_calls": tool_calls})
        for tc in tool_calls:
            executor = executors.get(name)
            result = executor(args)             # 真正执行工具（web_search/web_fetch/bash其中之一）
            tool_calls_made += 1
            messages.append({"role": "tool", "tool_call_id": tc.get("id"), "content": result[:MAX_TOOL_RESULT_CHARS]})
    # --- 工具预算用完/模型自然停止但没写完，强制走一次"只准写报告"的合成轮 ---
    messages.append({"role": "user", "content": SYNTHESIS_INSTRUCTION})
    body = _completion_body(model, messages, max_tokens=synthesis_max_tokens, ...)  # 这一轮不给tools
    resp = _post_with_retry(client, url, headers, body)
    return AgenticResult(content=strip_tool_markup(msg.get("content") or ""), ..., truncated_loop=True)
```

**关键变量逐个解释**：
- **`messages`**：这是**唯一一条对话历史**，从头到尾就这一个模型在自己跟自己对话——每调用一次工具，就往`messages`里追加一轮"assistant要调用什么工具"+"tool返回了什么结果"，下一轮请求把完整历史带上再问模型。这就是"多轮"的物理实现：不是多次独立提问，是**同一个对话越滚越长**。
- **`tool_calls_made`**：全局计数器，每执行一次工具调用（不管哪个工具）就+1，直到达到`max_tool_calls`上限（默认16）。
- **`max_tool_calls=DEFAULT_MAX_TOOL_CALLS=16`**：单道题最多允许16次工具调用，注释写"Deep-research tasks want many searches, so this is generous"。
- **`while tool_calls_made < max_tool_calls`**：**这就是"跑好几轮"的判断条件**——模型自己决定要不要继续调工具，只要没到16次上限、且模型还想调工具，就一直循环。
- **`tool_choice="required" if (force_first_tool and tool_calls_made == 0) else "auto"`**：`force_first_tool`是个开关，给"不太爱主动搜索"的模型（注释提到gemini-flash）强制第一轮必须调用工具，之后改回"auto"（模型自己判断要不要调工具）。
- **循环退出的两种情况**：①模型自然写出了非空正文且没有再请求工具（`not tool_calls`且`content.strip()`非空）——正常结束；②工具预算耗尽或模型空手而归——**触发`SYNTHESIS_INSTRUCTION`这个强制合成指令**，最后再单独发一次"不给工具、必须现在就写完整报告"的请求，确保不会因为循环没走完就交白卷。
- **`truncated_loop`**：标记这次结果是"正常结束"还是"被工具预算逼着强行合成"，供后续分析用。

---

## 阶段3：判官 + 合成

**文件**：`TrustedRouter-Fusion-Draco/scripts/draco_client_fusion.py`

```python
JUDGE_SYSTEM = (
    "You are the TrustedRouter Fusion judge. Compare panel responses and return "
    "compact JSON with keys consensus, contradictions, partial_coverage, "
    "unique_insights, blind_spots, and final_guidance. ..."
)
FINAL_INSTRUCTION = (
    "TrustedRouter Fusion panel answers and judge analysis follow. Use the panel "
    "answers as the primary evidence and the judge analysis as guidance to write "
    "the final answer for the original request. ..."
)
```
- 这两个常量就是之前对比过的"判官prompt"和"合成prompt"，跟生产代码里的版本几乎一致（判官只差"Fusion"vs"Synth"这一个词）。

```python
def _load_panel(path: Path) -> dict[str, str]:
    """task_id -> final report text (last non-failed row wins)."""
    out: dict[str, str] = {}
    for line in path.read_text(encoding="utf-8").splitlines():
        r = json.loads(line)
        if r.get("status") == "failed":
            continue
        t = r.get("task_id")
        c = (r.get("final") or {}).get("content")
        if t and isinstance(c, str) and c.strip():
            out[t] = c
    return out
```
- **`_load_panel`**：**这一步证实了"面板不是这个脚本自己跑出来的，是读现成文件"**——它读的是阶段2产出的`replay`文件（比如`solo-gpt55.jsonl`），把每道题的最终报告文本取出来，做成`task_id -> 报告文本`的字典。

```python
def run_one(task) -> dict[str, Any]:
    panel = [(l, panels[l][task.id]) for l in labels]   # 拼出这道题的N份面板答案
    # 1) 判官
    jkey = f"{args.judge_model}\t{args.judge_provider or ''}\t{task.id}"
    judge_json = judge_cache.get(jkey)
    if judge_json is None:
        judge_body = {
            "model": args.judge_model, "response_format": {"type": "json_object"},
            "messages": [
                {"role": "system", "content": JUDGE_SYSTEM},
                {"role": "user", "content": _judge_user(task.problem, panel)},
            ],
        }
        judge_json = _content(_post(client, args.base_url, api_key, judge_body)).strip()
    # 2) 合成
    final_user = FINAL_INSTRUCTION + "\n\n" + _panel_evidence(panel) + "\n\nJudge analysis JSON:\n" + judge_json
    def _fuse(model: str, pin: bool = True) -> tuple[str, dict[str, Any]]:
        body = {"model": model, "messages": [
            {"role": "user", "content": task.problem},
            {"role": "user", "content": final_user},
        ]}
        resp = _post(client, args.base_url, api_key, body)
        return strip_tool_markup(_content(resp)), resp
    for attempt in range(5):
        content, fr = _fuse(args.fuser_model, pin=(attempt == 0))
        if len(content.strip()) >= 50:
            break
        time.sleep(1.5 * (attempt + 1))
    if len(content.strip()) < 50 and args.fallback_fuser_model:
        content, fr = _fuse(args.fallback_fuser_model, pin=False)
```

**关键变量逐个解释**：
- **`labels`/`panels`**：`--panel model_label=replay_path`这个命令行参数（可以重复传多次，对应有几个面板成员），`panels`是`{label: {task_id: 报告文本}}`的嵌套字典。
- **`args.judge_model`（默认`google/gemini-3.1-pro-preview`）/ `args.fuser_model`（默认`anthropic/claude-opus-4.8`）**：判官和合成用哪个模型，命令行可配置，前面查过的判官×合成9宫格实验就是靠反复改这两个参数跑出来的。
- **`judge_cache`**：**这是个复用优化**——判官分析只取决于"面板+judge_model+provider"，跟最终用哪个模型合成无关，所以做judge×synth网格实验时，同一份面板的判官分析只用算一次，缓存起来给后面所有synthesizer复用，省钱。
- **`for attempt in range(5)` + `pin`参数**：**合成阶段有重试机制**——最多试5次，第一次锁定一个具体供应商（`pin=True`），如果内容长度<50字符（判定为空/近似空输出），换成不锁定供应商的方式重试，每次间隔`1.5*(attempt+1)`秒。这是应对"GLM-5.2遇到涉政内容静默拒答"这类问题的兜底——**注意默认没有传`--fallback-fuser-model`，注释写"we do not pass one, to keep the synthesizer a pure GLM measurement"**，即测试GLM当合成器时，故意不做跨模型兜底，让空输出如实计0分，不是隐藏问题。
- **`final_user`拼接顺序**：`FINAL_INSTRUCTION`（指令）+ `_panel_evidence(panel)`（5份面板原始答案）+ 判官JSON——**合成模型看到的是"指令+原始面板证据+判官摘要"三部分**，不是只看判官摘要，这也是为什么文档里强调"panel evidence primary, judge analysis as guidance"（面板证据是主，判官分析是参考）。

---

## 阶段4：打分

**文件**：`scripts/draco_rejudge.py` + `src/trusted_router/evals/draco_replay.py` + `src/trusted_router/evals/fusion_micro.py`

```python
DRACO_JUDGE_MODEL = "google/gemini-3.1-pro-preview"   # fusion_micro.py
DRACO_JUDGE_PASSES = 3                                 # fusion_micro.py
DEFAULT_TR_CRITERION_JUDGE_CHUNK_SIZE = 3              # fusion_live.py
DEFAULT_JUDGE_REASONING_EFFORT = "high"                # fusion_live.py
```
- **`DRACO_JUDGE_MODEL`**：全部DRACO打分统一用这一个裁判模型，跟面板/合成用什么模型无关——之前反复强调"同一个judge协议"，指的就是这个常量固定不变。
- **`DRACO_JUDGE_PASSES = 3`**：整套打分流程（对全部评分细则）要**重复跑3遍，取平均**，减少裁判模型自身的随机波动。
- **`DEFAULT_TR_CRITERION_JUDGE_CHUNK_SIZE = 3`**：**这就是之前反复提到的"chunk-of-3"协议**——不是把全部~39条评分细则一次性甩给裁判模型判一次，而是**每次只给3条细则**去判，分成好几批（chunk），每批单独一次裁判调用。研究报告里提过"chunk-all会让分数虚高"，所以固定用chunk-of-3。**注意这跟`DRACO_JUDGE_PASSES=3`是两个独立的"3"，一个是"分几条一批"，一个是"整套流程重复几遍"，容易搞混。**

```python
def rejudge_replay_row(row, *, tr_client, judge_model, judge_passes, criterion_chunk_size, ...):
    ...
    for chunk in _chunks(criteria, criterion_chunk_size):
        chunk_judgments, chunk_raw_results = _judge_replay_criteria_chunk(..., chunk, ...)
```
- **`_chunks(criteria, criterion_chunk_size)`**：把这道题rubric里的全部评分细则，按`criterion_chunk_size`（默认3）切成一批批小组，每组单独发一次裁判请求。
- **`_judge_replay_criteria_chunk`**：内部还有`left_judgments`/`right_judgments`这种递归对半拆分逻辑（代码里能看到），是失败重试的兜底——一批判不动就拆成两半分别再判，提高稳健性。

**汇总**：`scripts/draco_report.py`读取所有打好分的行，调用`summarize_score_rows`算出每个config_id的平均分（`mean_score`），跟OpenRouter公开分数对比出`delta_from_openrouter`，最后`markdown_report`生成人类可读的报告——这就是最终看到的"XX分"数字的最后一步来源。

---

## 四阶段串联全景图

```
draco.py: 拉取100道题（可选filter_draco_tasks排除财务题）
    ↓
draco_agentic_solo.py（对每个面板模型各跑一次，产出N份replay文件）
    ├─ agentic_tools.py: DRACO_AGENTIC_SYSTEM_PROMPT + 三个工具 + run_agentic_completion的多轮循环
    └─ 每道题最多16轮工具调用，工具用完/模型停手就强制走SYNTHESIS_INSTRUCTION合成
    ↓
draco_client_fusion.py（读取上面产出的N份replay，当作"面板"）
    ├─ JUDGE_SYSTEM: 判官出JSON分析（consensus/contradictions/partial_coverage/unique_insights/blind_spots/final_guidance）
    └─ FINAL_INSTRUCTION: 合成模型读"指令+面板原文+判官JSON"，写最终报告（带重试+可选跨模型兜底）
    ↓
draco_rejudge.py + draco_replay.py（用google/gemini-3.1-pro-preview固定裁判）
    ├─ chunk-of-3: 评分细则每3条一批分别判
    └─ judge_passes=3: 整套打分重复3遍取平均
    ↓
draco_report.py: 汇总成最终分数、跟OpenRouter对比
```

---

## 附录：如果真要跑一次评测，完整命令行是什么样

下面给两组：①复现Prometheus 2.0（尝试跑到77.4附近）；②复现73.4分那组（frontier面板 + Kimi-K2.6判官 + GLM-5.2合成，"Synth is two jobs"里的最佳格）。都走"路径B"（`draco_agentic_solo.py` × 5 + `draco_client_fusion.py`），因为路径A（`draco_native_fusion_gen.py`）跑出来的是~40分那个量级，不是这两个分数对应的方法。

### ① 复现Prometheus 2.0

**第一步：5个面板模型，各跑一次`draco_agentic_solo.py`**

```bash
python scripts/draco_agentic_solo.py \
  --manifest data/draco-full-100.manifest.json \
  --output replays/solo-minimax-m3.jsonl \
  --model minimax/minimax-m3 \
  --config-id solo_minimax_m3_tooled \
  --execute

python scripts/draco_agentic_solo.py \
  --manifest data/draco-full-100.manifest.json \
  --output replays/solo-kimi-k3.jsonl \
  --model moonshotai/kimi-k3 \
  --config-id solo_kimi_k3_tooled \
  --execute

python scripts/draco_agentic_solo.py \
  --manifest data/draco-full-100.manifest.json \
  --output replays/solo-glm-5.2.jsonl \
  --model z-ai/glm-5.2 \
  --config-id solo_glm_5_2_tooled \
  --execute

python scripts/draco_agentic_solo.py \
  --manifest data/draco-full-100.manifest.json \
  --output replays/solo-deepseek-v4-pro.jsonl \
  --model deepseek/deepseek-v4-pro \
  --config-id solo_deepseek_v4_pro_tooled \
  --execute

python scripts/draco_agentic_solo.py \
  --manifest data/draco-full-100.manifest.json \
  --output replays/solo-mimo-v2.5-pro.jsonl \
  --model xiaomi/mimo-v2.5-pro \
  --config-id solo_mimo_v2_5_pro_tooled \
  --execute
```

**第二步：判官+合成，模型分工照抄Prometheus 2.0生产配置**

```bash
python scripts/draco_client_fusion.py \
  --manifest data/draco-full-100.manifest.json \
  --panel minimax-m3=replays/solo-minimax-m3.jsonl \
  --panel kimi-k3=replays/solo-kimi-k3.jsonl \
  --panel glm-5.2=replays/solo-glm-5.2.jsonl \
  --panel deepseek-v4-pro=replays/solo-deepseek-v4-pro.jsonl \
  --panel mimo-v2.5-pro=replays/solo-mimo-v2.5-pro.jsonl \
  --output replays/fusion-prometheus-2.0.jsonl \
  --config-id fusion_prometheus_2_0 \
  --judge-model minimax/minimax-m3 \
  --fuser-model moonshotai/kimi-k3 \
  --fallback-fuser-model z-ai/glm-5.2 \
  --execute
```

**第三步：打分**

```bash
python scripts/draco_rejudge.py \
  replays/fusion-prometheus-2.0.jsonl \
  --output results/rejudge-prometheus-2.0.jsonl \
  --execute
```

**第四步：出报告**

```bash
python scripts/draco_report.py \
  results/rejudge-prometheus-2.0.jsonl \
  --title "Prometheus 2.0 reproduction attempt" \
  --output reports/prometheus-2.0-report.md \
  --json-output reports/prometheus-2.0-report.json
```

### ② 复现73.4分（frontier面板，Kimi-K2.6判官→GLM-5.2合成）

**第一步：frontier面板5个模型（GPT-5.5+Opus-4.8+Gemini-3-Flash+Kimi-K2.6+DeepSeek-V4-Pro）**

```bash
python scripts/draco_agentic_solo.py --manifest data/draco-full-100.manifest.json --output replays/solo-gpt55.jsonl --model openai/gpt-5.5 --config-id solo_gpt55_tooled --execute
python scripts/draco_agentic_solo.py --manifest data/draco-full-100.manifest.json --output replays/solo-opus.jsonl --model anthropic/claude-opus-4.8 --config-id solo_opus_tooled --execute
python scripts/draco_agentic_solo.py --manifest data/draco-full-100.manifest.json --output replays/solo-gemini-flash.jsonl --model google/gemini-3-flash-preview --config-id solo_gemini_flash_tooled --force-first-tool --execute
python scripts/draco_agentic_solo.py --manifest data/draco-full-100.manifest.json --output replays/solo-kimi-k26.jsonl --model moonshotai/kimi-k2.6 --config-id solo_kimi_k26_tooled --execute
python scripts/draco_agentic_solo.py --manifest data/draco-full-100.manifest.json --output replays/solo-deepseek.jsonl --model deepseek/deepseek-v4-pro --config-id solo_deepseek_tooled --execute
```
（`gemini-3-flash`额外加了`--force-first-tool`——之前查过它是"不太爱主动调用工具"的模型，需要强制第一轮必须调用工具。）

**第二步：判官=Kimi-K2.6，合成=GLM-5.2（这是73.4的最佳格）**

```bash
python scripts/draco_client_fusion.py \
  --manifest data/draco-full-100.manifest.json \
  --panel gpt55=replays/solo-gpt55.jsonl \
  --panel opus=replays/solo-opus.jsonl \
  --panel gemini-flash=replays/solo-gemini-flash.jsonl \
  --panel kimi-k26=replays/solo-kimi-k26.jsonl \
  --panel deepseek=replays/solo-deepseek.jsonl \
  --output replays/fusion-frontier-kimi-glm.jsonl \
  --config-id fusion_frontier_kimi_judge_glm_synth \
  --judge-model moonshotai/kimi-k2.6 \
  --fuser-model z-ai/glm-5.2 \
  --execute
```

**第三、四步同①**，把路径换成这一组的文件即可。

### 所有参数含义（两个脚本合并列，重复的只写一次）

**`draco_agentic_solo.py`**

| 参数 | 默认值 | 含义 |
|---|---|---|
| `--manifest` | `artifacts/fusion-draco/draco-non-financial-80.manifest.json` | 题库清单文件路径，指定跑哪一批题（全100题还是排除财务的80题）。**上面命令特意换成了`draco-full-100.manifest.json`**，因为要跟77.4/73.4这类"全100题"的分数比，必须用全量清单，不能用默认的80题子集。 |
| `--output` | 必填 | 这次跑的结果写到哪个jsonl文件。 |
| `--model` | `deepseek/deepseek-v4-pro` | **这次要跑哪个模型**，改这个参数就是在切换"当前是哪个面板成员在跑"。 |
| `--config-id` | `solo_deepseek_v4_pro_tooled` | 一个自由文本标签，写进输出文件里标识"这批结果是哪个配置跑出来的"，不影响实际请求，纯粹为了后续区分。 |
| `--task-id` | 无（跑全部） | 可重复传，只跑指定的几道题（用于调试/补跑单题），不传就是跑`--manifest`里的全部题目。 |
| `--limit` | 无 | 只跑前N道题，调试用，正式跑分不要传（否则不是100题）。 |
| `--max-tool-calls` | `16` | 单道题最多允许的工具调用次数上限，就是文档正文里反复提到的那个16。 |
| `--max-tokens` | `8000` | 研究阶段每次模型调用的输出token上限。 |
| `--synthesis-max-tokens` | `12000` | 最后那次"强制写完整报告"调用的输出token上限（比研究阶段的上限更宽，因为这一轮要写完整报告）。 |
| `--force-first-tool` | 关（需要显式加这个flag开启） | 强制第一轮必须调用工具，给"不太爱主动搜索"的模型用（例子里是gemini-flash）。 |
| `--temperature` | `0.2` | 采样温度，偏低，减少随机性。 |
| `--reasoning-effort` | 无 | 部分推理模型可设`low`/`high`控制推理力度，不设就用模型默认。 |
| `--no-bash` | 关 | 加上这个flag就禁用bash工具（比如目标环境没有docker时用）。 |
| `--doc-parser` | `llamaparse` | 网页/文件解析链路：`llamaparse`（先付费高精度解析，失败退到markitdown再退到纯文本）/`markitdown`/`plain`。 |
| `--no-sec-facts` | 关 | 加上就禁用`sec_facts`这个专门查SEC财报数据的工具。 |
| `--bash-image` | `python:3.12-slim` | bash工具用的Docker镜像。 |
| `--base-url` | `https://api-us-central1.quillrouter.com/v1` | 请求发到哪个网关地址（走的是quillrouter别名域名，纯代理模式，不经过融合逻辑）。 |
| `--api-key-name` | 无（按`KEY_NAMES`列表顺序找） | 指定用哪个环境变量/密钥名去认证，不传就按内置优先级列表挨个找第一个存在的。 |
| `--workers` | `2` | 并发跑几道题（线程池大小），1-8之间。 |
| `--timeout-seconds` | `600.0` | 单次HTTP请求超时时间。 |
| `--resume` | 关 | 加上就是接着之前中断的进度续跑（会跳过`--output`文件里已经成功过的`task_id`）。 |
| `--execute` | 关 | **必须显式加这个flag才会真的发请求**——不加就是"dry run"，只打印会跑哪些题、哪个模型，不花钱。 |

**`draco_client_fusion.py`**（省略跟上表重复的`--manifest`/`--output`/`--base-url`/`--workers`/`--timeout-seconds`/`--resume`/`--execute`/`--limit`，含义相同）

| 参数 | 默认值 | 含义 |
|---|---|---|
| `--panel` | 必填，可重复 | 格式`模型标签=replay文件路径`，每传一次代表一个面板成员，标签是自己起的名字（不影响实际调用哪个模型，只是给这份证据贴的标签）。 |
| `--config-id` | `fusion_client_budget_opus` | 同样是自由标签。 |
| `--judge-model` | `google/gemini-3.1-pro-preview` | **融合流程内部"判官/分析"阶段用哪个模型**——注意这跟阶段4打分用的裁判模型是两个完全不同的角色，容易搞混。 |
| `--fuser-model` | `anthropic/claude-opus-4.8` | 合成阶段主用哪个模型。 |
| `--fallback-fuser-model` | 无 | 主合成器连续5次重试后仍输出过短（<50字符），才会换成这个兜底模型。**这个脚本只支持一层兜底**，不像生产代码的Prometheus 2.0配置有两层兜底（Kimi-K3→GLM-5.2→MiniMax-M3），如果要完全照抄生产的三级兜底链，脚本本身需要改动。 |
| `--judge-max-tokens` | `3000` | 判官这次调用的输出token上限。 |
| `--fuser-max-tokens` | `8000` | 合成这次调用的输出token上限。 |
| `--judge-provider` / `--judge-providers` | 无 | 强制判官调用走哪个/哪几个供应商（比如GLM-5.2要指定走tinfoil供应商避开Z.ai自家审查）；`-providers`可传多个，按任务id哈希分散到不同供应商上。 |
| `--fuser-provider` / `--fuser-providers` / `--fuser-provider-frac` | 无 / 无 / `1.0` | 同理，控制合成阶段走哪个供应商；`-frac`可以只让一部分任务走指定供应商，其余走默认路由（用于压测/对比）。 |
| `--judge-cache` | 无 | 判官分析结果的共享缓存文件路径——同一份面板换不同合成器测试时，判官这一步不用重复算，直接复用缓存，省钱。 |
| `--api-key-name` | `TR_FUSION_EVAL_API_KEY` | 用哪个密钥认证。 |

**⚠️ 两组命令都有一个绕不开的前提**：这份代码目前完全没有接过`moonshotai/kimi-k3`和`xiaomi/mimo-v2.5-pro`这两个模型（之前查过，全仓库、全部39条commit都搜不到），实际执行前八成会因为供应商/路由没配置好而报错，需要先确认`tr_sdk`底层能不能实际路由到这两个模型，这部分不是改改命令行参数就能解决的，可能要先补供应商接入。
