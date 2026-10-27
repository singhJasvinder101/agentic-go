# agentic-go

`agentic-go` is a Go library for building LLM-powered applications and agents — a lean, Go-native alternative to LangChain/LangGraph. Every capability lives in its own package with a direct constructor: import what you use, nothing initializes behind your back, and there's no central registry or factory to configure.

## Why agentic-go

- **Minimal surface, maximum reuse.** A handful of small interfaces (`llm.Provider`, `embedder.Embedder`, `vectorstore.VectorStore`, `memory.Memory`, `chain.Chain`) compose into everything above them — agents, chains, RAG, and graphs are all built on the same primitives you can use directly.
- **No factory, no global registry.** Each provider (`llm/openai`, `llm/claude`, ...) has its own `New()`. Only the packages you import get initialized, and swapping providers is a one-line change.
- **Idiomatic Go.** Explicit errors, `context.Context` everywhere, functional options, generics where they earn their keep (`llm.GenerateStructured[T]`) — nothing here asks you to think in another language's idioms.

## Installation

```bash
go get github.com/singhJasvinder101/agentic-go
```

Vector store backends pull in their own SDK dependencies only when you import them (for example `vectorstore/chroma` adds `chroma-go`).

## Quick start

```go
package main

import (
	"context"
	"fmt"
	"log"

	initializers "github.com/singhJasvinder101/agentic-go/init"
	"github.com/singhJasvinder101/agentic-go/llm"
	"github.com/singhJasvinder101/agentic-go/llm/gemini"
)

func main() {
	ctx := context.Background()
	initializers.Init(ctx, "configs/config.yaml")

	provider, err := gemini.New(ctx)
	if err != nil {
		log.Fatal(err)
	}

	resp, err := provider.Generate(ctx, &llm.GenerateRequest{
		Messages: []llm.Message{
			llm.UserMessage(llm.TextPart("Hello, what is the capital of France?")),
		},
	})
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(resp.Text())
}
```

See [Configuration](docs/configuration.md) for `configs/config.yaml`, and [LLM providers](docs/llm.md) for tool calling, streaming, structured output, and multimodal messages.

## Architecture

This is a library. `init.Init` loads YAML config and logging once. It does not construct a provider, a store, or an agent. Each adapter reads its own keys through `config.GetString`. Importing a subpackage is what pulls in that SDK.

Callers depend on small interfaces. Orchestration packages (`agent`, `chain`, `retriever`) program against those interfaces. `graph` imports none of them: a node function may call an agent or a chain, and the executor stays unaware of LLMs.

```
caller
  ├── agent.Agent ──────────────► llm.Provider
  │                         └──► memory.Memory
  ├── chain.Prompt / Sequential ► llm.Provider
  │                         └──► template (text or Registry)
  ├── retriever.RAGChain ───────► llm.Provider
  │                         └──► Retriever ──► VectorStore ──► Embedder
  └── graph.Runnable[S]          (nodes call whatever they need)
```

| Interface | Package | Contract |
|---|---|---|
| `Provider` | `llm` | `Generate`, `GenerateStream`, `Close` |
| `Embedder` | `embedder` | `EmbedDocuments`, `EmbedQuery` |
| `VectorStore` | `vectorstore` | `AddDocuments`, `SimilaritySearch`, `Delete`. The store holds the embedder. |
| `Memory` | `memory` | `Messages`, `Add`, `Clear`. `Buffer` is the in-process implementation. |
| `Chain` | `chain` | `Run(ctx, vars map[string]any) (string, error)` |
| `Retriever` | `retriever` | `Retrieve(ctx, query, k)` returns documents and drops scores. |

`llm.Message` is the shared turn: role `system`, `user`, `assistant`, or `tool`, body `[]ContentPart` (`text`, `image_url`, `image`, `file`, `tool_call`, `tool_result`). Every provider converts that shape in its own `messages.go`. `GenerateResponse.Text()` and `ToolCalls()` read the first choice. `Validate` is opt-in; `Generate` runs `PrepareRequest`.

`llm.GenerateStructured[T]` and `agent.NewTypedTool[Args]` both reflect a Go struct into a flat JSON schema (`json` tags for names, `jsonschema_description` for field text). Structured output appends a synthetic tool named `emit_result` and forces that tool, then decodes the arguments into `T`. Any provider that already supports tool calling gets structured output with no extra method on `Provider`.

### Runtime behavior

**Agent.** `Run` seeds messages with the system prompt, then `Memory.Messages()`, then the user turn. Each step calls `Provider.Generate`. An empty tool-call list is the final answer: the user and assistant messages are stored and `Result` is returned (`Text`, full `Messages`, `Steps`). Each tool call runs its `Handler`. A handler error or an unknown tool name becomes the string `error: …` and is sent back as a tool message, so the model can retry. The default cap is 10 generate calls (`ErrMaxIterations`).

**Chain.** `Prompt.Run` renders a template and sends one user turn. `Sequential` copies `vars` forward and writes the previous text under `outputKey` (default `"input"`). A failure is wrapped with the child index. `RAGChain` satisfies `Chain` by method shape and does not import `chain`.

**RAG.** `RAGChain.Run` requires `vars["question"]`. It retrieves `K` documents (default 4), joins `Content` with `---`, places that block in the system message, and calls `Generate` once.

**Graph.** `Compile` checks that the entry point exists and that every edge target exists or is `graph.End` (`"__end__"`). A node with no outgoing edge also stops the run. A conditional edge is a `Router[S]` whose string is looked up in the routes map. `Run` stops after 100 node executions unless you call `RunWithLimit`. On a node error, the state from before that node is returned with the error.

## When to use what

Use the smallest piece that can express the control flow.

| You need | Use |
|---|---|
| One completion you fully specify | `llm.Provider.Generate` |
| Tokens as they arrive | `llm.NewStream` |
| A Go struct, not free text | `llm.GenerateStructured[T]` |
| The model to choose tools and loop | `agent.Agent` |
| A fixed pipeline (summarize, then translate) | `chain.Sequential` |
| One prompt string | `chain.NewPromptTemplate` |
| The same prompt text shared by several chains | `template.Registry` and `chain.NewPrompt` |
| An answer grounded in your documents | `retriever.RAGChain` |
| Matching documents, with no generated answer | `retriever.Retriever` |
| History across user turns | `memory.Buffer` via `agent.WithMemory` |
| A branch, a cycle, or retry until valid | `graph.Runnable[S]` |

Index documents with an embedder and a vector store, then answer from them with `RAGChain`. Wrap that `RAGChain` and a `Prompt` in `Sequential` when a later step should rewrite the answer. Reach for `Agent` when the user can also ask for a live tool such as weather. When billing and support are different agents, the router is a graph node and each branch calls the agent it needs.

### Generate

One question, classification, or rewrite. You own the messages. `openai.New()` and `claude.New()` take no context. `gemini.New(ctx)` and `ollama.New(ctx)` do. The `Generate` call stays the same.

```go
resp, err := provider.Generate(ctx, &llm.GenerateRequest{
    Messages: []llm.Message{
        llm.SystemMessage(llm.TextPart("Answer in one sentence.")),
        llm.UserMessage(llm.TextPart("Capital of France?")),
    },
})
if err != nil {
    return err
}
fmt.Println(resp.Text())
```

### Streaming

Same request shape, pulled one chunk at a time.

```go
stream := llm.NewStream(provider.GenerateStream(ctx, req))
for stream.Next() {
    fmt.Print(stream.Text())
}
if err := stream.Err(); err != nil {
    return err
}
```

### Structured output

The next line of Go needs fields.

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
```

### Agent

The model decides which function to call, possibly more than once, before it answers. `NewTypedTool` reflects the schema from `WeatherArgs`. A schema that cannot be reflected panics at construction.

```go
type WeatherArgs struct {
    City string `json:"city" jsonschema_description:"City name"`
}

weather := agent.NewTypedTool("get_weather", "Current weather",
    func(ctx context.Context, args WeatherArgs) (string, error) {
        return fmt.Sprintf(`{"temp_c":18,"city":%q}`, args.City), nil
    })

a := agent.New(provider,
    agent.WithTools(weather),
    agent.WithSystemPrompt("Use tools for live data."),
    agent.WithMemory(memory.NewBuffer(memory.WithMaxMessages(50))),
    agent.WithMaxIterations(10),
)
result, err := a.Run(ctx, "Weather in Paris?")
```

Build the schema by hand with `agent.NewTool` when it is loaded at runtime and is not a Go type. Both return `agent.Tool`.

Attach the same `memory.Buffer` for multi-turn chat. `Buffer` is conversation history. It is not a place to store a document corpus. `WithMaxMessages` drops the oldest messages first. Omit it for an unbounded buffer.

### Chain

The steps are known before the first call. `{{.Text}}` is native Go `text/template` syntax. Set `Engine` to `template.FormatterJinja` when the template uses Jinja (`{{ text }}`).

```go
summarize := chain.NewPromptTemplate(provider, "Summarize: {{.Text}}")
summarize.SystemPrompt = "Be concise."

translate := chain.NewPromptTemplate(provider, "Translate to French: {{.input}}")

out, err := chain.Sequential([]chain.Chain{summarize, translate}).
    Run(ctx, map[string]any{"Text": "..."})
```

`translate` receives the original vars plus the summary under `"input"`. Override that key with `chain.WithOutputKey`.

Share templates by registering them once:

```go
registry := template.NewRegistry()
registry.MustRegisterTemplate("summary", template.FormatterNative, "Summarize: {{.Text}}")
summarize := chain.NewPrompt(provider, registry, "summary")
```

### RAG

The answer has to come from your documents. `vectorstore/memory` is the in-process store. Chroma, Qdrant, Pinecone, and Weaviate take an embedder plus backend options. Claude has an LLM adapter and no embedder.

```go
emb, err := geminiembedder.New(ctx)
if err != nil {
    return err
}
store, err := memorystore.New(emb)
if err != nil {
    return err
}
if err := store.AddDocuments(ctx, []vectorstore.Document{
    {ID: "france", Content: "Paris is the capital of France."},
}); err != nil {
    return err
}

rag := retriever.NewRAGChain(retriever.FromVectorStore(store), provider, 4)
answer, err := rag.Run(ctx, map[string]any{"question": "Capital of France?"})
```

`rag.SystemPrompt` replaces the default instruction. Retrieved context is still appended. `RAGChain` can sit inside `chain.Sequential` beside any other `Chain`.

When you want the documents themselves (search results, citations, a graph node that should not call the model):

```go
docs, err := retriever.FromVectorStore(store).Retrieve(ctx, "capital of France", 3)
```

### Graph

The next node depends on state, or a node must run again until a condition holds. State is a generic struct, so a wrong field is a compile error. A fixed two-step prompt stays on `chain.Sequential`.

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
if err != nil {
    return err
}
final, err := runnable.Run(ctx, SupportState{Input: "charged twice"})
```

Point a conditional route back at an earlier node to retry. `RunWithLimit` sets the step cap for that run.

## Documentation

Full usage guides live in [`docs/`](docs):

- [Configuration](docs/configuration.md) — loading `config.yaml`, reading values
- [LLM providers](docs/llm.md) — generation, messages, tool calling, structured output, streaming, the `Provider` interface
- [Embedders](docs/embedders.md) — `EmbedDocuments` / `EmbedQuery`
- [Vector stores](docs/vectorstores.md) — memory, Chroma, Qdrant, Pinecone, Weaviate
- [Templates](docs/templates.md) — native (`text/template`) and Jinja2 prompt formatting
- [Memory](docs/memory.md) — conversation history buffers
- [Agents](docs/agents.md) — the tool-execution loop
- [Chains](docs/chains.md) — composable prompt → LLM pipelines
- [Retrievers / RAG](docs/retrievers.md) — retrieval-augmented generation
- [Graphs](docs/graphs.md) — cyclic state-machine orchestration

## Supported providers

| Provider | LLM (`llm/`) | Embedder (`embedder/`) |
|---|---|---|
| Gemini | yes | yes |
| OpenAI | yes | yes |
| Claude | yes | no |
| Ollama | yes | yes |

## Example CLI

A placeholder CLI lives at `cmd/agentic-go` (not yet implemented).

## Roadmap

- [x] Tools / agentic workflows
- [x] Role-based messaging
- [x] Tool calling
- [x] Structured output
- [x] Additional LLM and vector store backends
- [x] RAG
- [x] Conversation memory
- [x] Graph / state-machine orchestration
- [ ] MCP support
- [ ] Timeouts and resource limits
- [ ] Guardrails
- [x] Stream response abstraction (like `stream.Next()`)
- [ ] Parallel branch execution and checkpointing in `graph`

## Why Go?

- Goroutines make concurrent chunking and batched embedding trivial for RAG pipelines.
- Static typing catches the request/response shape mismatches that are easy to miss in dynamically-typed equivalents.
- A single static binary with no runtime dependency, which matters for deploying agents as services.
