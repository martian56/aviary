# aviary

[![CI](https://github.com/martian56/aviary/actions/workflows/ci.yml/badge.svg)](https://github.com/martian56/aviary/actions/workflows/ci.yml)

One chat-completion API across many LLM providers, for
[Raven](https://github.com/martian56/raven). Write your code once against a
neutral request and reply; switch providers by changing the model string.

```raven
import "github.com/martian56/aviary" { complete }
import "github.com/martian56/aviary/request" { ChatRequest }

fun main() {
    let reply = complete(
        ChatRequest.new("openai/gpt-4o-mini")
            .system("You are terse.")
            .user("Name three members of the crow family.")
    )
    match reply {
        Ok(r) -> print(r.content),
        Err(e) -> print("error: ${e.message}"),
    }
}
```

`complete` and `ask` come from the package root; the request, reply, and
message types come from their sub-modules (`aviary/request` and
`aviary/message`), the same way Raven libraries expose types defined outside
the entry file.

Point the same code at another provider by changing one string:

```raven
complete(ChatRequest.new("anthropic/claude-3-5-sonnet-latest").user("..."))
complete(ChatRequest.new("groq/llama-3.3-70b-versatile").user("..."))
complete(ChatRequest.new("ollama/llama3").user("..."))
```

## Install

```
rvpm add github.com/martian56/aviary@v0.2.0
```

No native dependencies; aviary is pure Raven over `std/http` and `std/json`.
Streaming and per-request timeouts use `std/http` APIs added in Raven
2.21.0, so build with that toolchain or newer.

## The model string

Every request names its model as `provider/model`. The provider before the
slash selects the endpoint, the credential, and the wire format; everything
after the first slash is the model name, passed through untouched (so
`openrouter/anthropic/claude-3.5-sonnet` works).

## Providers

Most providers speak the OpenAI chat-completions format and share one code
path. Anthropic, Gemini, and Cohere have native adapters.

| Provider prefix | Endpoint | API key variable |
|---|---|---|
| `openai` | api.openai.com | `OPENAI_API_KEY` |
| `anthropic` (or `claude`) | api.anthropic.com | `ANTHROPIC_API_KEY` |
| `gemini` (or `google`) | generativelanguage.googleapis.com | `GEMINI_API_KEY` |
| `cohere` | api.cohere.com | `COHERE_API_KEY` |
| `groq` | api.groq.com | `GROQ_API_KEY` |
| `mistral` | api.mistral.ai | `MISTRAL_API_KEY` |
| `deepseek` | api.deepseek.com | `DEEPSEEK_API_KEY` |
| `xai` (or `grok`) | api.x.ai | `XAI_API_KEY` |
| `together` | api.together.xyz | `TOGETHER_API_KEY` |
| `fireworks` | api.fireworks.ai | `FIREWORKS_API_KEY` |
| `perplexity` | api.perplexity.ai | `PERPLEXITY_API_KEY` |
| `openrouter` | openrouter.ai | `OPENROUTER_API_KEY` |
| `deepinfra` | api.deepinfra.com | `DEEPINFRA_API_KEY` |
| `nebius` | api.studio.nebius.ai | `NEBIUS_API_KEY` |
| `moonshot` | api.moonshot.cn | `MOONSHOT_API_KEY` |
| `azure` | `AZURE_API_BASE` | `AZURE_API_KEY` |
| `ollama` | localhost:11434 (or `OLLAMA_HOST`) | none |
| `custom` | `AVIARY_BASE_URL` (or `.base(...)`) | `AVIARY_API_KEY` |

The `custom` provider reaches any other OpenAI-compatible service: set the
base URL and key and use `custom/<model>`. You can also override per request
with `.key(...)` and `.base(...)`, which win over the environment.

Providers that need request signing (AWS Bedrock, Google Vertex) are out of
scope for now.

## Building a request

`ChatRequest` is a builder. Options you do not set are left out of the
request, so a minimal call is a model and a message.

```raven
ChatRequest.new("openai/gpt-4o")
    .system("You are a careful editor.")
    .user("Tighten this sentence: ...")
    .temperature(0.3)
    .max_tokens(500)
    .top_p(0.9)
    .timeout(180000)      // optional deadline in ms; default 120s
    .retries(0)           // optional; default 2 retries on 429/5xx/transport
    .key(my_key)          // optional; overrides the env var
    .base(my_base_url)    // optional; overrides the default endpoint
    .header("X-Title: my app")   // optional extra header line
```

Transient failures (429, 500, 502, 503, 504, and transport errors) are
retried with exponential backoff; anything else fails immediately.

A one-liner when you just want text (no type imports needed):

```raven
import "github.com/martian56/aviary" { ask }
match ask("openai/gpt-4o-mini", "Say hi.") {
    Ok(text) -> print(text),
    Err(e) -> print(e.message),
}
```

## The reply

```raven
struct ChatReply {
    content: String,        // the text
    model: String,          // the model the provider reported
    finish_reason: String,  // "stop", "tool_calls", "end_turn", ...
    tool_calls: List<ToolCall>,
    usage: Usage,           // prompt_tokens, completion_tokens, total_tokens
    raw: String,            // the provider's untouched JSON, for anything else
}
```

## Streaming

`complete_stream` delivers the reply as it is produced: your callback gets
each text delta the moment it arrives, and the assembled `ChatReply` comes
back at the end. Works on every provider aviary supports.

```raven
import "github.com/martian56/aviary" { complete_stream }
import "github.com/martian56/aviary/request" { ChatRequest }

fun show(delta: String) {
    print(delta)
}

fun main() {
    let reply = complete_stream(
        ChatRequest.new("anthropic/claude-3-5-sonnet-latest").user("Tell me about rooks."),
        show,
    )
}
```

Tool calls stream too: fragments assemble internally and arrive complete in
`reply.tool_calls`. On a stream, `.timeout(ms)` bounds the connect and each
read rather than the whole response, so long generations keep flowing as
long as the provider keeps sending. Retries apply only to the connection;
once text has flowed, a failure surfaces as an error rather than a retry
that would repeat output. Token counts are whatever the provider includes
while streaming, which for some is nothing.

## Tool calling

Declare tools with a JSON Schema for the arguments, then feed results back:

```raven
import "github.com/martian56/aviary" { complete }
import "github.com/martian56/aviary/request" { ChatRequest }
import "github.com/martian56/aviary/message" { Message, ToolDef }
import "github.com/martian56/aviary/jsonutil" { Obj }

let weather = ToolDef.new(
    "get_weather",
    "Look up the current weather for a city.",
    Obj.new()
        .str("type", "object")
        .set("properties", Obj.new().set("city", Obj.new().str("type", "string").build()).build())
        .build()
)

let first = complete(ChatRequest.new("openai/gpt-4o").user("Weather in Baku?").tool(weather))
// If the model asked for a tool, run it and send the result back:
//   .message(Message.assistant_calls("", reply.tool_calls))
//   .message(Message.tool_result(call.id, call.name, your_result))
```

Tool calling is supported on the OpenAI-compatible providers, Anthropic, and
Gemini. On each, `tool_calls` on the reply carries the calls the model made,
with `arguments` as the raw JSON string the model produced.

## Layout

Raven turns a package's file paths into its public import paths, and the
entry file must be `lib.rv` at the repo root, so the public surface lives at
the root and the internals sit under `providers/`:

```
lib.rv            complete, complete_stream, ask   (import aviary)
request.rv        ChatRequest, ChatReply   (import aviary/request)
message.rv        Role, Message, ToolDef   (import aviary/message)
jsonutil.rv       Obj and JSON helpers     (import aviary/jsonutil)
providers/        internal, not imported directly
  registry.rv       provider table and resolution
  transport.rv      the one HTTP call
  openai.rv         the OpenAI-compatible adapter
  sse.rv            the server-sent-events parser
  accum.rv          the streamed-reply accumulator
  anthropic.rv, gemini.rv, cohere.rv   native adapters
src/main.rv       the demo
```

## Development

```
rvpm test     # 88 tests: request building, response and SSE parsing per
              # provider, plus end-to-end runs against an in-process server
rvpm build    # builds the demo in src/main.rv
rvpm run      # prints the request it would send, or calls out if a key is set
rvpm fmt
```

## License

MIT
