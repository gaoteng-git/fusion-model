# quill-cloud-proxy：Prometheus 2.0 用户请求的入口代码逐段讲解

本文档回答的问题：**当一个真实用户请求打到 TrustedRouter 的线上网关（`quill-cloud-proxy`）、要求使用 `trustedrouter/prometheus-2.0` 时，代码从 TCP 连接建立开始，是怎样一步步路由到 fusion 子系统的？**

写法与之前的《DRACO评测代码-逐段讲解.md》一致：按代码执行顺序、逐段贴代码、解释关键变量。本文档覆盖完整链路：从 `main()` 的 TCP accept 循环，到 `maybeServeFusion` 把请求移交进 fusion 子系统，再到 `serveFusionNonStreaming` 内部面板并发调度、判官JSON分析、合成三阶段的完整代码（第5节），是一份自包含的端到端讲解（另有《生产环境Fusion代码-逐段讲解.md》从配置构建角度切入、覆盖同一段代码，两份文档可互相参照）。

- 仓库：`Lore-Hex/quill-cloud-proxy`（已在上一任务中核实为 TrustedRouter 真实线上 attested gateway，见下方"背景核实"一节）
- 语言/位置：Go，`enclave-go/cmd/enclave/`
- 主要文件：`main.go`（HTTP 解析与路由分发）、`fusion.go`（fusion 配置解析与最终分发）

---

## 0. 背景核实（简述，细节见上一任务的对话结论）

`quill-cloud-proxy` 的 `NOTICE` 文件明确写道：

> "This product is part of TrustedRouter (https://trustedrouter.com)... The attested-enclave + parent vsock-proxy + multi-cloud bootstrap code in this repository implements the trust surface published at https://trust.trustedrouter.com."

实时抓取 `https://trust.trustedrouter.com/trust/gcp-release.json` 可以看到该 attestation 文件直接点名 `https://github.com/Lore-Hex/quill-cloud-proxy`，并带有具体的 commit（`2e8302c`）和镜像 digest（`sha256:2793d5ae...`）——这是一份可由任何人独立验证的、外部可访问的密码学证明，指向"这个 repo 的这个 commit 就是正在跑的线上网关镜像"。这是本次调查中拿到的最强证据（不是"看起来像"，而是线上系统自证）。

`README.md` 也说明了这个仓库产出两个二进制：
- `enclave-go/`：跑在 Nitro/CSP 机密计算 workload 里的 Go 程序——"Authenticates bearer hashes, calls the configured LLM provider... streams OpenAI-format chunks back"。**这就是本文档要追踪的代码。**
- `parent/`：跑在 EC2 宿主机上的 Python 运维工具，与请求处理无关。

---

## 1. `main()` —— 进程启动、监听端口、进入 accept 循环

文件：`enclave-go/cmd/enclave/main.go`，`func main()` 从第 66 行开始。

跳过大量初始化细节（加载证书、初始化 `auth.Registry`、初始化 `llm.Client`、`trustedrouter.Client` 等——这些是网关自身的基础设施，和 fusion 逻辑无关），核心是第 232 行附近建立监听、以及第 398 行开始的 accept 循环：

```go
var listener net.Listener = rawListener
...
for {
    conn, err := listener.Accept()
    if err != nil {
        ...
    }
    go serveOne(ctx, conn, registry, br, tlsServer, deviceBlob, trGateway, byokSecrets)
}
```

- **关键变量**：`listener` 是网关对外暴露的 TCP 监听端口（enclave 内部服务的实际端口，由 vsock-proxy/CSP 网络转发过来）；`br` 是 `llm.Client`，即调用各家上游模型 API（MiniMax、Moonshot/Kimi、Zhipu/GLM、DeepSeek、Xiaomi/MiMo 等）的统一客户端，后面 fusion 面板真正打模型请求都靠它；`trGateway` 是 `*trustedrouter.Client`，代表和 TrustedRouter 控制面（control plane）的连接，fusion 子系统要求这个连接必须存在且 enabled（下面会看到）；`byokSecrets` 是 BYOK（bring-your-own-key）密钥缓存。
- **执行逻辑**：每 accept 到一条新连接，就 `go serveOne(...)` 启一个 goroutine 去服务它——也就是说，一条 TCP 连接对应一个 goroutine，这是标准的"每连接一协程"模型，不是每请求一协程（因为要支持 HTTP keep-alive，一条连接上可能有多个请求，见下一节）。

另外第 413 行有一个 `startHealthListener(port string)`，是独立的健康检查端口（给 K8s/CSP 探活用），和用户请求路径完全无关，跳过。

---

## 2. `serveOne` —— 单条连接的请求循环（支持 keep-alive）

文件同上，`func serveOne(...)` 第 434 行：

```go
func serveOne(
    ctx context.Context,
    conn net.Conn,
    reg *auth.Registry,
    br llm.Client,
    tlsServer *enclavetls.Server,
    deviceBlob []byte,
    trGateway *trustedrouter.Client,
    byokSecrets *byokcache.Cache,
) {
    statsConn := &responseStatsConn{Conn: conn}
    conn = statsConn
    defer conn.Close()

    requestReader := bufio.NewReaderSize(conn, maxHTTPHeaderLineBytes+1)
    attestationCount := 0
    healthRequestCount := 0
    for serveOneRequest(ctx, conn, statsConn, requestReader, reg, br, deviceBlob, trGateway, byokSecrets, &attestationCount, &healthRequestCount) {
    }
}
```

- **关键变量**：`statsConn` 是对原始 `net.Conn` 的一层包装，用于统计这条连接上写出去的响应字节数等（计费/可观测性用，和业务逻辑无关）；`requestReader` 是带缓冲的 reader，供后面手写的 HTTP 请求行/请求头解析使用（这个网关没有用 Go 标准库的 `net/http` Server，而是自己手动解析 HTTP，因为它要在 attested enclave 里做一些标准库不支持的定制，比如流式转发、逐字节计费等）；`attestationCount`/`healthRequestCount` 是这条连接生命周期内的计数器，用于限制同一条连接上 attestation/health 请求的滥用次数。
- **执行逻辑**：`for serveOneRequest(...) {}` —— `serveOneRequest` 每处理完一个 HTTP 请求，返回一个 bool：`true` 表示"这条连接还能继续复用、请再读下一个请求"（HTTP keep-alive），`false` 表示"该关闭连接了"（比如 `Connection: close`、协议错误、或者已经把响应写完且不打算复用）。这个 for 循环就是 keep-alive 的实现方式。

---

## 3. `serveOneRequest` —— 核心路由函数（HTTP 解析 + 分发）

文件同上，`func serveOneRequest(...)` 第 455 行开始。这是整个入口链路里最重要、最长的函数。以下按执行顺序分段讲解（省略纯 HTTP 协议解析的样板代码，如读请求行、读 header、读 body、处理 chunked encoding 等，只讲和 fusion 路由相关的部分）。

### 3.1 `/v1/messages` 分支（第 650~663 行附近）

```go
if routePath == "/v1/messages" {
    ... // Anthropic Messages API 透传逻辑
}
```

这是 Anthropic 原生 Messages API 的透传路由，和 OpenAI 兼容的 `/v1/chat/completions` 是平行的两条路径。**Prometheus 2.0 走的是下面的 `/v1/chat/completions` 分支**（TrustedRouter 的产品线是 OpenAI 兼容 API 风格，`trustedrouter/prometheus-2.0` 作为 `model` 字段的值传入）。这里跳过不展开。

### 3.2 请求体解析分支（第 665~750 行附近）

```go
var req types.OpenAIChatRequest
...
} else if routePath == "/v1/chat/completions" {
    if method != "POST" {
        writeError(conn, 404, "route not found")
        return
    }
    if err := json.Unmarshal(body, &req); err != nil {
        ...
    }
    req.NormalizeMaxTokens()
    originalInput = req.Messages
} else {
    writeError(conn, 404, "route not found")
    return
}
```

- **关键变量**：`req` 是 `types.OpenAIChatRequest` 类型——这就是承载整条请求的核心结构体，`req.Model` 字段此时的值就是调用方传入的 `"trustedrouter/prometheus-2.0"`，`req.Messages` 是用户的对话历史，`req.Tools` 是调用方声明的工具（如果调用方想开搜索工具，就在这里传）。`originalInput = req.Messages` 保存了一份原始输入的引用，供后面 fusion 流程需要"最初的用户消息"时使用（比如面板成员各自需要拿到同一份原始输入）。
- 在这之前，代码分支还处理了 `/v1/conversations`（返回 501 不支持）、`/v1/responses/input_tokens`、以及其他不支持的 `/v1/responses/*` 子路径，还有 `/v1/responses`（通过 `parseResponsesRequest` + `adapter.ResponsesToChat` 把 Responses API 格式转换成内部统一的 `OpenAIChatRequest`）。这说明网关对外同时暴露 Chat Completions 和 Responses 两套 API 风格，内部统一转换成同一个 `req` 结构处理。

### 3.3 请求元数据处理（紧接着，约第 750~780 行）

```go
applyAttributionHeaders(...)
validateOrObserveRequestMetadata(...)
adapter.RejectUnsupportedN(&req)
```

这几步是通用的请求校验/计费归因逻辑（打租户标签、拒绝不支持的 `n>1` 参数等），对所有模型（不只 fusion）都会执行，跳过不展开。

### 3.4 自定义模型解析

```go
if isCustomModelID(req.Model) {
    resolvedCustomModel, err = maybeResolveCustomModelForOrchestration(...)
    ...
}
```

这是处理"自定义模型别名"的逻辑（用户可以在 TrustedRouter 控制台配一个自定义模型 ID，指向某个具体上游模型或某个 fusion 预设）。`trustedrouter/prometheus-2.0` 本身不是自定义模型 ID，是内置的固定模型名（见下面 `isFusionModel` 的判断），所以这一步对本文的场景通常不生效，但代码结构上会先检查一次。

### 3.5 Responses 专属的 web_search 分支（约第 798~802 行）

```go
if routeType == "responses" && !isUserProvidedCustomModel(resolvedCustomModel) && maybeServeResponsesWebSearch(ctx, conn, &req, br, trGateway, byokSecrets, bearer, requestLogID) {
    return
}
```

这一行是上次已经确认过的关键证据：`maybeServeResponsesWebSearch` 只在 `routeType == "responses"` 时才会被调用。也就是说，这个"网关自己执行真实 Exa 搜索"的能力，**架构上被限定死只服务于 Responses API 这条独立路径**，和 `/v1/chat/completions`（Prometheus 2.0 走的路径）完全不共享调用路径。这就是之前结论"生产环境 Prometheus 2.0 面板阶段不会自动拿到真实搜索结果"的代码级根据之一。

### 3.6 顺序分发链：Advisor → Subagent → **Fusion**（约第 804~836 行）

```go
if handled, err := maybeServeAdvisor(...); handled {
    ...
    return
}

if handled, err := maybeServeSubagent(...); handled {
    ...
    return
}

if handled, err := maybeServeFusion(ctx, conn, br, &req, trGateway, byokSecrets, bearer, originalInput, requestLogID); handled {
    if err != nil {
        var aerr *adapter.AdapterError
        if asAdapterErr(err, &aerr) {
            writeError(conn, aerr.Status, aerr.Message)
            return
        }
        writeError(conn, 500, "fusion error")
    }
    return
}
```

- **执行逻辑**：这是一条典型的"责任链"（chain of responsibility）模式——依次问每个子系统"这个请求归你管吗（`handled`）"，谁先说 `true` 就把请求交给谁处理并 `return`，都不认领就往下走普通模型路径（非 fusion 的常规模型请求最终会走到 `userModelAnthropicReq` 等逻辑，这部分和 fusion 无关，本文不展开）。
- 对于 `req.Model == "trustedrouter/prometheus-2.0"` 的请求：`maybeServeAdvisor` 和 `maybeServeSubagent` 内部各自会检查模型名是否匹配自己负责的模型集合，都不匹配，返回 `handled=false`；轮到 `maybeServeFusion` 时才会命中。**这一行就是本文档要找的"入口点"：请求从通用 HTTP 路由代码，正式移交进 fusion 子系统的确切位置。**

---

## 4. `maybeServeFusion` —— fusion 子系统的守门与分发函数

文件：`enclave-go/cmd/enclave/fusion.go`，`func maybeServeFusion(...)` 第 689 行开始。这是连接"入口链路"和"已经写过的面板/裁判/合成主线"的桥梁函数，逐段讲解如下。

### 4.1 判断这是不是一个 fusion 请求

```go
config, requested, err := fusionConfigForRequest(req)
if err != nil {
    return true, err
}
if !requested {
    return false, nil
}
```

`fusionConfigForRequest` 的定义（第 830 行）揭示了判断逻辑：

```go
func fusionConfigForRequest(req *types.OpenAIChatRequest) (fusionConfig, bool, error) {
    config := fusionConfig{Enabled: true}
    requested := isFusionModel(req.Model)
    ...
    cleanTools, toolConfig, toolRequested, err := fusionConfigFromTools(req.Tools)
    ...
    if toolRequested {
        config = mergeFusionConfig(config, toolConfig)
        requested = true
        req.Tools = cleanTools
    }
    return config, requested, nil
}
```

- **关键变量**：`requested` 最基础的赋值就是 `isFusionModel(req.Model)`——也就是说，**只要 `req.Model` 字段的值是网关内置认识的 fusion 模型名之一（`trustedrouter/prometheus-2.0` 正是其中之一），`requested` 就直接为 `true`**，不需要调用方额外传任何 plugin 或 tool 配置。这对应上面 `isFusionModel` 函数（第 255 行）的 `switch` 语句，里面列出了包括 `trustedRouterPrometheus20Model`（值为 `"trustedrouter/prometheus-2.0"`，见第 33 行常量定义）在内的一长串内置模型名常量。
- 另外两条路径（`fusionConfigFromPlugins`、`fusionConfigFromTools`）是给"通用 fusion 原语"（比如用户在一个普通模型请求里通过 `plugins:[{id:"synth",...}]` 或工具数组里塞 `trustedrouter:synth` 配置来临时启用 fusion）用的，和 Prometheus 2.0 这种"预设产品模型"场景不是同一条路径——Prometheus 2.0 用户根本不需要碰 plugins/tools 字段，只需要把 `model` 设成 `"trustedrouter/prometheus-2.0"`。
- 回到 `maybeServeFusion`：如果 `requested == false`（既不是内置 fusion 模型名，也没人塞 plugin/tool 配置），直接 `return false, nil`，把请求交还给上层继续走普通模型路径。对 Prometheus 2.0 请求而言，这里 `requested == true`，继续往下走。

### 4.2 校验与控制面连通性检查

```go
if !config.Enabled {
    if isFusionModel(req.Model) {
        return true, &adapter.AdapterError{Status: 400, Message: "trustedrouter/synth cannot be disabled without selecting a concrete model", Context: "plugins.synth.enabled"}
    }
    return false, nil
}
if trGateway == nil || !trGateway.Enabled() {
    return true, &adapter.AdapterError{Status: 503, Message: "trustedrouter/synth requires the TrustedRouter control plane", Context: "trustedrouter/synth"}
}
forceProviderJurisdiction(req, config.ProviderJurisdiction)
config.Mode = fusionModeForRequest(req.Model, config.Mode)
```

- `trGateway` 就是第 1 节 `main()` 里传下来的、和 TrustedRouter 控制面的连接——这里做了一次硬性检查：**fusion 功能强依赖控制面在线**，控制面挂了直接 503，不会退化成"裸模型直连"。
- `config.Mode` 在这里被 `fusionModeForRequest` 根据模型名重新赋值——对于 `trustedrouter/prometheus-2.0` 这种预设产品模型，走的是普通的"面板 fusion 模式"（不是下面 `fusionModeMapReduce` 那条 map-reduce 特殊模式，那是给另一类模型用的）。

### 4.3 预设面板解析——Prometheus 2.0 具体拿到哪 5 个模型

```go
if len(config.AnalysisModels) == 0 {
    if preset, panel, ok := fusionPresetPanelForModel(req.Model); ok {
        config.Preset = preset
        config.AnalysisModels = panel
    } else if !isGenericFusionPrimitive(req.Model) {
        config.AnalysisModels = append([]string(nil), fusionQualityPanel...)
    }
}
```

因为调用方没有显式传 `analysis_models`（Prometheus 2.0 用户不需要、也通常不会自己指定面板），`config.AnalysisModels` 此时是空的，于是走 `fusionPresetPanelForModel(req.Model)` 查表。这个函数（第 309 行）里明确写着：

```go
case trustedRouterPrometheus20Model:
    return "quality-2.0", append([]string(nil), fusionPrometheus20Panel...), true
```

也就是说，`config.Preset` 被设为字符串 `"quality-2.0"`，`config.AnalysisModels` 被设为 `fusionPrometheus20Panel` 这个包变量的内容——这正是之前在《生产环境Fusion代码-逐段讲解.md》里已经确认过的 5 模型面板（MiniMax M3、Kimi K3、GLM 5.2、DeepSeek V4 Pro、MiMo V2.5 Pro）。**这一行就是"Prometheus 2.0"这个产品名字第一次在代码里被翻译成具体模型列表的地方。**

### 4.4 后续校验、模型 ID 解析、裁判/合成模型确定

```go
if err := validateGenericFusionConfig(config, req.Model); err != nil { ... }
if len(config.AnalysisModels) > 8 { ... }
if config.SelectionStrategy == "" { ... }
switch config.SelectionStrategy { ... }
if config.MaxToolCalls < 0 || config.MaxToolCalls > 16 { ... }

for i, model := range config.AnalysisModels {
    config.AnalysisModels[i] = resolveFusionModelID(model)
}
finalModels, err := fusionFinalModels(config, req.Model, config.AnalysisModels[0])
judgeModels, err := fusionJudgeModels(config, req.Model)
selectorModels := judgeModels
```

- `MaxToolCalls` 的取值范围校验（`0~16`）就是之前讨论过的"面板模型最多迭代 16 轮"这个上限的代码出处之一（默认值/来源在 `fusionConfig` 结构体别处定义，这里只做边界校验）。
- `resolveFusionModelID` 把面板里的逻辑模型名（如 `"minimax/m3"`）解析成实际调用上游 API 时用的具体 provider/模型 ID。
- `fusionFinalModels` / `fusionJudgeModels` 分别确定"合成阶段"和"裁判阶段"要用的模型链（含 primary/fallback），这部分和之前文档里说的"判官 MiniMax M3 主用/Kimi K3 兜底，合成 Kimi K3 主用/GLM 5.2 兜底/MiniMax M3 兜底"完全对应。

### 4.5 fusion-code 变体的模型置换（与 Prometheus 2.0 无关，跳过）

```go
codeModel := isFusionCodeModel(req.Model)
config.CodeModel = codeModel
if codeModel {
    config.AnalysisModels = applyFusionCodeSwap(config.AnalysisModels)
    judgeModels = applyFusionCodeSwap(judgeModels)
}
config.BuiltInPanelPrompt, config.BuiltInFinalPrompt = fusionBuiltInPrompts(codeModel)
```

`trustedrouter/prometheus-2.0` 不是 code 变体（`isFusionCodeModel` 返回 `false`），这一段对它不生效，只是顺带把内置的面板 prompt / 合成 prompt 模板取出来（`fusionBuiltInPrompts`——这正是之前文档里逐句对比过的生产环境 prompt 的来源函数）。

### 4.6 最终分发：streaming / non-streaming

```go
if req.Stream {
    if config.SelectionStrategy == fusionSelectorSelectionStrategy {
        serveSelectorStreaming(...)
    } else {
        serveFusionStreaming(ctx, conn, br, req, config, finalModels, judgeModels, trGateway, secretCache, bearer, originalInput, requestLogID)
    }
} else {
    if config.SelectionStrategy == fusionSelectorSelectionStrategy {
        serveSelectorNonStreaming(...)
    } else {
        serveFusionNonStreaming(ctx, conn, br, req, config, finalModels, judgeModels, trGateway, secretCache, bearer, originalInput, requestLogID)
    }
}
return true, nil
```

这里根据 `req.Stream`（用户请求里是否要求流式返回）以及 `config.SelectionStrategy` 是否是特殊的 "selector" 策略（Prometheus 2.0 默认走普通的 synthesize 策略，不是 selector），最终调用到 `serveFusionNonStreaming` 或 `serveFusionStreaming`。

`maybeServeFusion` 最后 `return true, nil`：`true` 告诉 `serveOneRequest`"这个请求我已经处理完了、响应也写完了"，`serveOneRequest` 就此 `return`，回到 `serveOne` 的 for 循环，等待这条连接上的下一个请求（如果是 keep-alive）。

`serveFusionNonStreaming`（`fusion.go` 第 1221 行）就是 `maybeServeFusion` 最终落到的执行函数——下面第 5 节逐段展开面板/判官/合成三阶段的完整代码。

---

## 5. `serveFusionNonStreaming` —— 面板/判官/合成三阶段逐段讲解

文件：`enclave-go/cmd/enclave/fusion.go`，`func serveFusionNonStreaming(...)` 第 1221 行开始。整体流程：

```
runFusionPanel(...)                       # 1. 面板：5个模型并发跑一次
  → 若 SelectionStrategy 不是 synthesize/synthesize_non_refusals：
      selectFusionPanelResult(...) → 直接返回某个面板成员的答案（select类策略，不走判官/合成）
  → 否则（Prometheus 2.0 默认走这条）：
runFusionJudge(...)                       # 2. 判官：读5份面板答案，出JSON分析
runFusionFinal(...)                       # 3. 合成：读判官JSON+5份面板答案，出最终可见答案
writeFusionChatCompletionResponse(...)    # 4. 组装成OpenAI格式响应写回
```

### 5.1 面板阶段：`runFusionPanelObserved`（第 1442 行）——并发调度，每个成员只跑一次

```go
func runFusionPanelObserved(ctx, br, req, config, ...) ([]fusionCallResult, error) {
    panel := make([]fusionCallResult, len(config.AnalysisModels))
    var wg sync.WaitGroup
    for i, model := range config.AnalysisModels {
        wg.Add(1)
        go func(i int, model string) {
            defer wg.Done()
            panelReq := fusionPanelRequest(req, model, i, config.MaxCompletionTokens, config.PanelPrompt, config.BuiltInPanelPrompt)
            result, err := runFusionCallObserved(ctx, br, panelReq, ...)   // ← 每个成员只调用这一次
            if err != nil {
                panel[i] = fusionCallResult{Result: adapter.StreamResult{
                    Text: fmt.Sprintf("[panel member %d, model %s failed before producing an answer: %s]", i+1, model, err.Error()),
                    FinishReason: "error",
                }, Model: model}
                return
            }
            if strings.TrimSpace(result.Result.Text) == "" && len(result.Result.ToolCalls) == 0 {
                result.Result.Text = fmt.Sprintf("[panel member %d, model %s returned an empty answer; finish_reason=%s]", ...)
                result.Result.FinishReason = "empty"
            }
            panel[i] = result
        }(i, model)
    }
    wg.Wait()
    return panel, nil
}
```

- **`go func(i int, model string) { ... }(i, model)`**：Go语言goroutine——5个面板成员并发起飞、互不等待，这一点跟DRACO评测脚本里的多线程`ThreadPoolExecutor`本质是同一个意思（都是并行调度），但**这只是"并行"，不是"迭代"**，两者不要混为一谈。
- **`wg.Add(1)` / `wg.Wait()`**：`sync.WaitGroup`，等所有goroutine都跑完才继续往下走——必须5个面板成员全部有结果（或失败兜底文案），才能进入下一阶段。
- **每个面板成员只调用了一次`runFusionCallObserved`**——这就是"单轮"的直接证据：没有循环，没有根据返回结果决定要不要再调一次。如果模型返回的是`ToolCalls`（想调用工具），这里也不会去执行它、拿结果喂回去，只是原样记录进`panel[i]`，交给后面的判官/合成阶段当"证据"去看。
- **失败/空输出兜底**：调用出错，或者返回文本为空且没有工具调用，都会生成一段人类可读的占位文案，保证判官阶段总能拿到点什么，不会因为某个面板成员挂了就让整个流程崩掉。

**面板请求的构造（`fusionPanelRequest`，第 2530 行）——工具怎么传、prompt怎么拼：**

```go
func fusionPanelRequest(req *types.OpenAIChatRequest, model string, index int, maxCompletionTokens int, panelPrompt string, builtInPrompt ...string) *types.OpenAIChatRequest {
    out := cloneChatRequest(req)
    out.Model = model
    out.Stream = false
    // Give each panel member the caller's function tools (minus the
    // trustedrouter:synth config entry) so they can actually propose tool calls.
    // Without this the panel was tool-blind and only ever produced text, so the
    // select strategies (first_non_refusal / first_success) could never surface a
    // tool call — a tool-use request silently lost its tools. The first
    // non-refusal panel member still wins as before; now that answer may itself
    // be a tool call, which passes straight through.
    out.Tools = stripFusionToolEntries(req.Tools)
    if len(out.Tools) == 0 {
        out.ToolChoice = nil
    }
    out.MaxTokens = fusionInnerMaxTokens(req, maxCompletionTokens)
    basePrompt := ""
    if len(builtInPrompt) > 0 {
        basePrompt = strings.TrimSpace(builtInPrompt[0])
    }
    if basePrompt == "" {
        basePrompt = "Answer the request independently and return only the visible answer."
    }
    system := fmt.Sprintf("You are TrustedRouter Synth panel member %d.\n\n%s", index+1, basePrompt)
    if len(out.Tools) > 0 {
        system += "\n\nIf the next correct step is a provided function call, emit the tool call directly instead of describing it."
    }
    if custom := strings.TrimSpace(panelPrompt); custom != "" {
        system += "\n\nAdditional caller panel instructions:\n" + custom
    }
    out.Messages = prependSystem(req.Messages, system)
    return out
}
```

- **`out.Tools = stripFusionToolEntries(req.Tools)`**：面板成员能拿到的工具，**100%来自调用方自己请求里带的`req.Tools`**，生产系统不会自动挂载任何搜索/联网工具；`stripFusionToolEntries`只是把内部控制用的`trustedrouter:synth`配置项摘掉，不影响调用方自己定义的真实工具。源码原文这段注释交代了历史动机：早期版本面板是"工具盲"的，会导致`first_success`/`first_non_refusal`这类选优策略下，调用方自己传的工具被悄悄丢掉——这是后来修复的。
- **`basePrompt`**：调用方不传自定义`builtInPrompt`时的兜底话——**没有任何"Search context"或研究框架内容**，跟DRACO评测的`DRACO_AGENTIC_SYSTEM_PROMPT`完全不是一回事。
- **`system := ...`**：最终系统prompt = "你是第几号面板成员" + 兜底/自定义指令，**没有任何检索资料被拼进去**。
- **`out.Messages = prependSystem(req.Messages, system)`**：把系统prompt插到用户原始对话最前面。

**面板调用最终落地：单轮的证据链（`runFusionCallValidatedObservedAttempt`，第 1801 行）**

`runFusionCallObserved`一路调用到`runFusionCallValidatedObservedAttempt`，核心执行只有两行：

```go
go invokeProviderStream(invokeCtx, br, req, anthropicReq, pw, options, ...)
result, err := adapter.CollectAnthropicTextWithObserver(pr, collectObserver)
```

拿到`result`直接返回——**没有代码检查`result.ToolCalls`是否非空、如果非空就执行工具再发起第二轮**。哪怕模型这一轮吐出`finish_reason=tool_calls`，这个"我要调用工具"的请求本身就被当作这个面板成员的最终产出，原样记录进`fusionCallResult`。

需要澄清一点避免和"重试"混淆：`invokeProviderStream`（`provider_stream.go` 第 21 行）内部确实有`for tryN := 0; ; tryN++`和`for i, option := range options`两层循环，但语义分别是"同一endpoint瞬时网络错误重试"和"多个候选路由/BYOK endpoint故障转移"——是基础设施可靠性重试，**不是**"模型调用工具→执行→喂回结果→再问一次"的agentic循环。生产Prometheus 2.0面板阶段只有前者，没有后者。

### 5.2 判官阶段：`fusionJudgeRequest` / `runFusionJudge`（第 1551、2569 行）

```go
func fusionJudgeRequest(req *types.OpenAIChatRequest, model string, panel []fusionCallResult, maxCompletionTokens int) *types.OpenAIChatRequest {
    out := cloneChatRequest(req)
    out.Model = model
    out.Stream = false
    out.Tools = nil
    out.ToolChoice = nil
    out.ResponseFormat = map[string]any{"type": "json_object"}
    out.MaxTokens = fusionInnerMaxTokens(req, maxCompletionTokens)
    out.Messages = []types.OpenAIChatMessage{
        {Role: "system", Content: "You are the TrustedRouter Synth judge. Compare panel responses and return compact JSON with keys consensus, contradictions, partial_coverage, unique_insights, blind_spots, and final_guidance. Do not write the final answer. Return only JSON; do not include chain-of-thought, hidden reasoning, or <think> blocks."},
        {Role: "user", Content: fusionJudgePrompt(req, panel)},
    }
    return out
}
```

- **`out.Tools = nil`**：判官阶段**永远没有工具**，纯粹读文本做分析——这是三个阶段里唯一被完全禁用工具的一环。
- **`out.ResponseFormat = map[string]any{"type": "json_object"}`**：强制JSON模式，接口层面保证输出是合法JSON。
- **`fusionJudgePrompt(req, panel)`**：把用户原始请求 + 5份面板答案拼成user消息。
- 判官只有**一次调用**（`runFusionJudge`内部按`judgeModels`候选列表依次尝试，Prometheus 2.0是`["minimax/minimax-m3", "moonshotai/kimi-k3"]`，第一个失败才换下一个——这是"模型级别的失败转移"，不是"同一个模型多轮迭代"）。

### 5.3 合成阶段：`fusionFinalRequest` / `runFusionFinal`（第 1652、2594 行）

```go
func fusionFinalRequest(req *types.OpenAIChatRequest, model string, judgeJSON string, panel []fusionCallResult, config fusionConfig) *types.OpenAIChatRequest {
    out := cloneChatRequest(req)
    out.Model = model
    out.Stream = false
    out.Tools = stripFusionToolEntries(out.Tools)
    out.Messages = append([]types.OpenAIChatMessage{}, req.Messages...)
    instruction := strings.TrimSpace(config.BuiltInFinalPrompt)
    if instruction == "" {
        instruction = "Use the panel answers as evidence and the judge analysis as guidance to answer the original request. Return only the final visible answer."
    }
    if len(out.Tools) > 0 {
        instruction += "\n\nIf the next correct action is a provided function call, emit the tool call directly instead of describing it in text. Return visible text only when no tool call is needed."
    }
    if custom := strings.TrimSpace(config.SynthesisPrompt); custom != "" {
        instruction += "\n\nAdditional caller synthesis instructions:\n" + custom
    }
    out.Messages = append(out.Messages, types.OpenAIChatMessage{
        Role: "user",
        Content: instruction + "\n\n" + fusionPanelEvidence(panel) + "\n\nJudge analysis JSON:\n" + judgeJSON,
    })
    return out
}

func fusionPanelEvidence(panel []fusionCallResult) string {
    var b strings.Builder
    b.WriteString("Panel answers:\n")
    for i, item := range panel {
        text := strings.TrimSpace(item.Result.Text)
        if tc := fusionToolCallsText(item.Result.ToolCalls); tc != "" {
            text += "\n" + tc      // 面板成员的工具调用请求，转成可读文字塞进证据
        }
        fmt.Fprintf(&b, "\n[%d] model=%s\n%s\n", i+1, model, text)
    }
    return b.String()
}
```

- **`out.Tools = stripFusionToolEntries(out.Tools)`**：合成阶段同样跟随调用方原始`req.Tools`，不像判官阶段那样被强制清空。
- **`instruction`**：默认兜底合成指令，调用方可通过`config.SynthesisPrompt`追加"额外合成要求"。
- **`fusionPanelEvidence`**：把5个面板结果格式化成编号列表——**如果某个面板成员的`Result`里有`ToolCalls`而不是文本，`fusionToolCallsText`会把这个工具调用请求转成一段可读文字**（比如"想调用web_search，参数是..."），当作这个面板成员的"答案"塞进证据里，**而不是真的去执行、把执行结果放进去**——这再次印证：面板成员想调用的工具，从来没有被真正执行过，只是被转述成文字证据。
- **最终user消息** = 合成指令 + `Panel answers:`（5份面板证据）+ 判官JSON，一次性发给合成模型，同样只调用一次（`runFusionFinal`内部`finalModels`候选列表`[Kimi-K3, GLM-5.2, MiniMax-M3]`依次失败转移，同样是模型级兜底，不是多轮迭代）。

### 5.4 响应组装：工具调用请求的两种最终归宿

```go
if err := writeFusionChatCompletionResponse(
    &body, requestID, responseModel, final.Result.Text, adapter.JoinThinking(final.Result.Thinking),
    final.Result.ToolCalls,     // ← 合成阶段模型自己吐出的工具调用，会原样透传进最终响应
    totalIn, totalOut, ...,
); err != nil { ... }
writeJSONResponse(conn, 200, body.Bytes())
```

这里能看到两种"工具调用请求"完全不同的下场：
1. **面板成员**的工具调用请求 → 被`fusionToolCallsText`转成内部文字证据，喂给判官/合成阶段参考，**外部调用方永远看不到**，也从未被执行。
2. **合成（final）阶段**模型自己的工具调用请求 → 原样写进`writeFusionChatCompletionResponse`的`toolCalls`参数，**结构化透传回外部调用方**（标准OpenAI function-calling格式：`finish_reason: "tool_calls"` + `tool_calls: [...]`），由调用方自己执行、自己发起下一轮请求——网关自己从不执行。

---

## 6. 全链路小结（一句话版本）

```
main() 建监听、accept循环
  → go serveOne(conn)                         # 每条 TCP 连接一个 goroutine
    → for serveOneRequest(...) {}             # 支持 keep-alive，一条连接多个请求
      → 解析 HTTP，识别 /v1/chat/completions，json.Unmarshal 出 req(含 req.Model="trustedrouter/prometheus-2.0")
      → 依次尝试 maybeServeAdvisor / maybeServeSubagent（均不认领）
      → maybeServeFusion(req)                 # ← 用户请求真正进入 fusion 子系统的入口点
        → fusionConfigForRequest(req)         # isFusionModel(req.Model) → requested=true
        → 校验 trGateway 在线、设置 fusion 模式
        → fusionPresetPanelForModel(req.Model) → preset="quality-2.0", AnalysisModels=fusionPrometheus20Panel（5模型面板）
        → resolveFusionModelID / fusionFinalModels / fusionJudgeModels  # 确定合成/裁判模型链
        → serveFusionNonStreaming(...) 或 serveFusionStreaming(...)   # ← 见第5节：面板并发调度→判官JSON分析→合成，逐段展开
```

## 7. 与本次调查其它结论的关系

1. `maybeServeResponsesWebSearch` 只挂在 `routeType == "responses"` 分支（3.5 节），而 `maybeServeFusion` 挂在通用分发链上、对 `/v1/chat/completions` 传入的 Prometheus 2.0 请求生效——这是"生产环境搜索工具与 fusion 面板阶段架构上互不相通"这一结论在入口代码层面的又一处直接证据（此前已从 `fusion.go` 内部的 `fusionConfigFromTools` 不注入新工具、以及两个文件间零功能耦合这两个角度证实过；这是第三个独立的代码位置）。
2. `isFusionModel`/`fusionPresetPanelForModel` 这两个函数是**纯粹的字符串匹配/查表**，不含任何"评测模式"或"是否来自 DRACO 复现代码"的判断逻辑——换句话说，**生产网关代码本身完全无法区分一个 `model="trustedrouter/prometheus-2.0"` 的请求，究竟是真实付费用户发的，还是 DRACO 评测脚本（`draco_agentic_solo.py`/`draco_client_fusion.py`）在测试时发的**。这与此前结论一致：77.4 分对应的评测方法论（Path B）走的是完全独立的 Python 评测代码库（`TrustedRouter-Fusion-Draco`），并非直接调用这条生产 HTTP 入口；生产入口只是"被评测脚本当作面板成员的调用后端之一"（评测里单个模型的 solo 调用可能经过这个网关，但完整的 Prometheus 2.0 融合流程本身在评测里是评测脚本自己实现的编排，不是靠一次性调用 `trustedrouter/prometheus-2.0` 这个融合模型名跑出来的）。这一点此前文档已反复强调，这里从入口代码角度再次印证：**这条 `maybeServeFusion` 链路是"产品线上用户真实调用 Prometheus 2.0"的入口，不是 77.4 分评测跑分时代码实际执行的入口**——评测的入口是 `draco_agentic_solo.py`/`draco_client_fusion.py`（已在《DRACO评测代码-逐段讲解.md》中详述）。
