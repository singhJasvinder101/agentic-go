# agentic-go low-level design

A library, not a process. The caller constructs providers and composes them. There is no service registry and no factory that wires the graph at startup.

| 6 | 4 | 5 | 100 |
|---|---|---|---|
| Ports the rest of the library programs against | LLM adapters: OpenAI, Claude, Gemini, Ollama | Vector stores: memory, Chroma, Qdrant, Pinecone, Weaviate | Default graph step cap before a cycle is aborted |

> **Dependency rule**
>
> Orchestration packages depend on interfaces in llm, memory, template, and vectorstore. graph depends on none of them — a node function is free to call an agent or a chain, but the executor does not import those packages.

## Package composition

Arrows point from the caller to the type it depends on. Adapters sit behind the two ports at the bottom.

```mermaid
flowchart TB
  app["Caller"]
  agent["agent.Agent"]
  chain["chain.Prompt"]
  rag["retriever.RAGChain"]
  graph["graph.Runnable[S]"]
  config["init/config"]
  provider["llm.Provider"]
  memory["memory.Memory"]
  template["template.Registry"]
  retriever["retriever.Retriever"]
  store["vectorstore.VectorStore"]
  embedder["embedder.Embedder"]

  app --> agent
  app --> chain
  app --> rag
  app --> graph
  app --> config
  agent --> provider
  agent --> memory
  chain --> provider
  chain --> template
  rag --> provider
  rag --> retriever
  retriever --> store
  store --> embedder

  class provider,store,embedder port
  classDef port stroke-width:2px
```

## When to use what

Pick the smallest piece that can express the control flow. A straight prompt stays on Provider or Chain. Tool choice belongs on Agent. Branching and retries belong on Graph.

| You need | Use | Skip it when |
|---|---|---|
| One completion you fully specify | `llm.Provider.Generate` | The model must call tools or you need tokens live |
| Tokens as they arrive | `llm.NewStream` | You only need the final string |
| A Go struct, not free text | `llm.GenerateStructured[T]` | The answer is prose for a human |
| The model chooses tools and loops | `agent.Agent` | The steps are fixed in advance |
| Summarize, then translate, then format | `chain.Sequential` | The next step depends on a category or a retry |
| An answer grounded in your documents | `retriever.RAGChain` | You only want the matching documents |
| Documents, no generated answer | `retriever.Retriever` | You want the model to write the reply |
| History across user turns | `memory.Buffer` on the agent | You are storing a document corpus |
| Branch, cycle, or retry until valid | `graph.Runnable[S]` | The path is a straight line of prompts |
| One prompt string | `chain.NewPromptTemplate` | Many chains share the same templates |
| Templates shared by several chains | `template.Registry` | The template is used once |

*Source: public constructors in llm, agent, chain, retriever, memory, graph, template.*

<details open>
<summary><strong>Provider.Generate</strong></summary>

Single question, classification, or rewrite. You own the messages. Swap openai.New, claude.New, gemini.New(ctx), or ollama.New(ctx) without changing the call.

```go
provider, err := gemini.New(ctx) // or openai.New(), claude.New()
resp, err := provider.Generate(ctx, &llm.GenerateRequest{
    Messages: []llm.Message{
        llm.SystemMessage(llm.TextPart("Answer in one sentence.")),
        llm.UserMessage(llm.TextPart("Capital of France?")),
    },
})
fmt.Println(resp.Text())
```

</details>

<details>
<summary><strong>Streaming</strong></summary>

A chat UI that should print tokens as they arrive. Same request shape as Generate.

```go
stream := llm.NewStream(provider.GenerateStream(ctx, req))
for stream.Next() {
    fmt.Print(stream.Text())
}
if err := stream.Err(); err != nil { /* stream failed */ }
```

</details>

<details>
<summary><strong>GenerateStructured</strong></summary>

The next line of Go needs fields, not a paragraph. Any tool-calling provider works. No extra method on Provider.

```go
type City struct {
    Name    string `json:"name"`
    Country string `json:"country"`
}
city, err := llm.GenerateStructured[City](ctx, provider, &llm.GenerateRequest{
    Messages: []llm.Message{
        llm.UserMessage(llm.TextPart("Capital of France?")),
    },
})
// city.Name == "Paris"
```

</details>

<details open>
<summary><strong>Agent</strong></summary>

The model must decide which function to call, possibly more than once, before it answers. Handler errors stay in the loop as text the model can read.

```go
type WeatherArgs struct {
    City string `json:"city" jsonschema_description:"City name"`
}
weather := agent.NewTypedTool("get_weather", "Current weather",
    func(ctx context.Context, args WeatherArgs) (string, error) {
        return `{"temp_c":18,"city":"` + args.City + `"}`, nil
    })

a := agent.New(provider,
    agent.WithTools(weather),
    agent.WithSystemPrompt("Use tools for live data."),
    agent.WithMemory(memory.NewBuffer(memory.WithMaxMessages(50))),
    agent.WithMaxIterations(10),
)
result, err := a.Run(ctx, "Weather in Paris?")
fmt.Println(result.Text, result.Steps)
```

</details>

<details open>
<summary><strong>Chain</strong></summary>

The steps are known before the first call. NewPromptTemplate for one prompt. Sequential when the next prompt consumes the previous text. Set Engine for Jinja.

```go
summarize := chain.NewPromptTemplate(provider,
    "Summarize: {{.Text}}")
summarize.SystemPrompt = "Be concise."

translate := chain.NewPromptTemplate(provider,
    "Translate to French: {{.input}}")

pipeline := chain.Sequential([]chain.Chain{summarize, translate})
out, err := pipeline.Run(ctx, map[string]any{"Text": "..."})
// translate sees the summary under "input"
```

</details>

<details>
<summary><strong>Registry, only if templates are shared</strong></summary>

Several chains reuse the same prompt text. One-off prompts should stay on NewPromptTemplate.

```go
registry := template.NewRegistry()
registry.MustRegisterTemplate("summary", template.FormatterNative,
    "Summarize: {{.Text}}")
summarize := chain.NewPrompt(provider, registry, "summary")
```

</details>

<details open>
<summary><strong>RAG</strong></summary>

The answer must come from your documents. Pass a vector store through FromVectorStore. Drop the RAGChain into Sequential when a later chain should rewrite the answer.

```go
emb, err := geminiembedder.New(ctx)
store, err := memorystore.New(emb)
store.AddDocuments(ctx, []vectorstore.Document{
    {ID: "france", Content: "Paris is the capital of France."},
})

rag := retriever.NewRAGChain(
    retriever.FromVectorStore(store), provider, 4)
answer, err := rag.Run(ctx, map[string]any{
    "question": "Capital of France?",
})
```

</details>

<details>
<summary><strong>Retriever without generation</strong></summary>

Search UI, citations, or a graph node that only needs documents. Do not use RAGChain if nothing should be sent to the model.

```go
docs, err := retriever.FromVectorStore(store).
    Retrieve(ctx, "capital of France", 3)
```

</details>

<details>
<summary><strong>Memory</strong></summary>

Multi-turn chat. Attach the same Buffer to the agent so the next Run sees prior turns. This is not a vector store.

```go
mem := memory.NewBuffer(memory.WithMaxMessages(20))
a := agent.New(provider, agent.WithMemory(mem))
a.Run(ctx, "My name is Ada.")
a.Run(ctx, "What is my name?") // history is on mem
```

</details>

<details open>
<summary><strong>Graph</strong></summary>

The next node depends on state, or a node must run again until a condition holds. Put an agent or a chain inside the node. Do not use Graph for a fixed two-step prompt.

```go
type SupportState struct {
    Input, Category, Result string
}
g := graph.New[SupportState]()
g.AddNode("classify", classifyNode)
g.AddNode("billing", billingNode)
g.AddNode("support", supportNode)
g.AddConditionalEdge("classify", func(s SupportState) string {
    return s.Category
}, map[string]string{
    "billing": "billing",
    "support": "support",
})
g.AddEdge("billing", graph.End)
g.AddEdge("support", graph.End)
g.SetEntryPoint("classify")

runnable, err := g.Compile()
final, err := runnable.Run(ctx, SupportState{Input: "charged twice"})
```

</details>

> **How the pieces stack**
>
> Index documents with an embedder and a vector store. Answer from them with RAGChain. If that answer must be translated, wrap RAGChain and a Prompt in Sequential. If the user can also ask for live weather, that path is an Agent, not a longer chain. If billing and support are different agents, the router is a Graph node, and each branch calls the agent it needs.

## Ports

| Interface | Package | Contract |
|---|---|---|
| `Provider` | llm | Generate, GenerateStream, Close. Shared request is GenerateRequest. |
| `Embedder` | embedder | EmbedDocuments for batches, EmbedQuery for a single search string. |
| `VectorStore` | vectorstore | AddDocuments, SimilaritySearch, Delete. Store owns the embedder. |
| `Memory` | memory | Messages, Add, Clear. Buffer is the in-process implementation. |
| `Chain` | chain | Run(ctx, vars map[string]any) (string, error). |
| `Retriever` | retriever | Retrieve(ctx, query, k) returns documents, dropping scores. |

*Source: public interfaces in llm, embedder, vectorstore, memory, chain, retriever.*

## Runtime flows

### agent.Run `loop`

One user turn. Default cap is 10 generate calls (ErrMaxIterations).

| Step | What happens |
|---|---|
| 1 | Build tools[] and name → Handler map. |
| 2 | Seed messages: system, then Memory.Messages(), then user. |
| 3 | Provider.Generate with messages and tools. |
| 4 | Append assistant message. If ToolCalls() is empty, Memory.Add(user, assistant) and return Result. |
| 5 | For each call, run Handler. Errors become the string "error: …" and are sent back as a tool message. |
| 6 | Loop. Unknown tool names are also tool results, not a hard stop. |

### chain.Sequential `chain`

Each child Chain.Run receives a copied vars map. The previous text is written under outputKey, default "input". The sequence returns the last child's text. A failure is wrapped with the child index.

### RAGChain.Run `rag`

Requires vars["question"]. Retrieves K documents (default 4), joins Content with "---", puts that block in the system message, and calls Provider.Generate once. It satisfies chain.Chain by method shape and does not import chain.

### graph.Runnable.RunWithLimit `cycles`

Compile copies nodes and edges, then checks the entry point and every edge target. End is the sentinel "__end__". A node with no outgoing edge also stops the run.

| Check | Failure |
|---|---|
| Entry set and registered | ErrEntryRequired or ErrNodeNotFound |
| Static edge target exists or is End | ErrNodeNotFound |
| Conditional route target exists or is End | ErrNodeNotFound |
| Router key missing from routes at runtime | no route for key |
| Steps reach maxSteps (default 100) | exceeded N steps |
| Node returns an error | state before that node is returned with the error |

## Shared message model

### llm.Message

Role is system, user, assistant, or tool. Body is []ContentPart. Providers convert this shape in their own messages.go. Validate is opt-in; Generate calls PrepareRequest instead.

| Part | Required fields |
|---|---|
| text | Text |
| image_url | URL |
| image / file | Data, MIMEType |
| tool_call | ToolCall (ID, Name, Arguments JSON) |
| tool_result | ToolCallID, Name, Text |

### GenerateResponse

Choices[0] is the assistant turn. Text() and ToolCalls() are derived from parts. Usage is optional. Raw keeps the provider payload. Stream is a pair of channels: chunks and errors.

| FinishReason | Meaning |
|---|---|
| stop | Model finished the turn |
| length | Hit the token limit |
| tool_calls | Model asked to call tools |
| content_filter | Provider blocked the output |

## Structured output and typed tools

Both paths reflect a Go struct into a flat JSON schema (jsonschema, DoNotReference, no $schema) via llm.SchemaFor[T]. Tags: json for names, jsonschema_description for field text.

### llm.GenerateStructured[T]

Appends a synthetic tool named emit_result and sets ToolChoice required to that name. Decodes the matching ToolCall.Arguments into T. Any provider that already implements tool calling gets structured output with no extra method on Provider.

### agent.NewTypedTool[Args]

Builds llm.Tool.Parameters from Args, then wraps the typed handler. The inner Handler unmarshals json.RawMessage. A schema that cannot be reflected panics at construction, same idea as MustRegisterTemplate.

## Adapters

Each New reads keys from the process-wide YAML config. Importing a subpackage is what pulls in that SDK.

| Adapter | Constructs | Config keys | Notes |
|---|---|---|---|
| llm/openai | OpenAIProvider | openai.api_key, openai.model | PrepareRequest, then go-openai chat completion |
| llm/claude | Claude provider | claude.* | No embedder counterpart |
| llm/gemini | Gemini provider | gemini.* | LLM and embedder |
| llm/ollama | Ollama provider | ollama.* | Shares internal/ollama client with the embedder |
| vectorstore/memory | in-process store | none | Tests and local search |
| vectorstore/chroma, qdrant, pinecone, weaviate | remote stores | per-backend keys | Embedder injected at New |

---

## Bootstrap and what is intentionally absent

### init.Init

Sets log to info/json, then config.MustInit loads YAML into a map once (sync.Once). Log is reconfigured from log.level and log.format. Providers call config.GetString themselves. Init does not construct an LLM, a store, or an agent.

### Not in this design

MCP, request timeouts, guardrails, graph checkpointing, and parallel branches are still open. cmd/agentic-go is a placeholder. Memory is process-local. Tool handler failures stay inside the loop as text the model can read.
