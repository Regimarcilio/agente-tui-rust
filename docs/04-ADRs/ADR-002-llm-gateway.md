# ADR-002: LLM Gateway — CLI Wrapper Subprocess + JSONL Streaming

**Status:** Accepted  
**Data:** 28/10/2026  
**Autor:** PLANMASTER  
**Relacionado:** RF-007, RF-008, RNF-001, RNF-003, EngAI Global

---

## Contexto

Agentes nativos precisam chamar LLMs. Opções (ver `docs/02-benchmarks.md` Seção 3):
1. **HTTP Direto (reqwest)** — Controle total, mas: auth manual por provider, model routing manual, sem cache nativo, manutenção alta
2. **SDK Oficial (async-openai, anthropic-sdk)** — Só 1-2 providers, telemetry SDK, vendor lock-in
3. **Gateway Próprio (Rust)** — Máximo controle, mas: reimplementar router, cache, guardrail, trace — reinventar roda
4. **Subprocess CLI Wrapper (escolhido)** — Reusa `opencode`/`claude`/`gh copilot`/`ollama` existentes

**EngAI Global Regra #1:** "Toda feature com IA passa por `src/ai/gateway` com router (OpenRouter/Ollama), cache semântico, RAG e guardrail. Registrar `trace.json`."

---

## Decisão

**Implementar `CliWrapperGateway` que gerencia subprocessos CLI com fallback chain:**

```rust
pub struct CliWrapperGateway {
    primary: CliProvider,      // opencode (router + cache + guardrail + trace)
    fallbacks: Vec<CliProvider>, // claude → gh copilot → ollama
    cache: SemanticCache,      // SQLite + embeddings (candle) - cache local adicional
    tracer: TraceCollector,    // Agrega trace.json do CLI + métricas próprias
}
```

**Fluxo:**
1. Agente chama `LlmGateway::stream_chat(ChatRequest)`
2. `CliWrapperGateway` tenta `primary` (opencode) com timeout 30s
3. Streaming JSONL parse incremental → `ChatEvent` stream para agente
4. Se falha (timeout, erro, exit code ≠ 0): retry 2x → próximo fallback
5. `trace.json` gerado pelo opencode é lido + enriquecido (agent_id, session_id)
6. Cache semântico local (candle embeddings) para deduplicar prompts idênticos

**Providers Suportados:**
| Provider | Comando | Auth | Router | Cache | Guardrail | Trace |
|----------|---------|------|--------|-------|-----------|-------|
| opencode | `opencode` | Login CLI | OpenRouter/Ollama | Semântico nativo | Sim | trace.json |
| claude | `claude` | Login CLI | Anthropic | Não nativo | Básico | JSONL |
| gh copilot | `gh copilot` | gh auth | GitHub Models | Não | Não | JSONL |
| ollama | `ollama run` | Local | Local only | Não | Não | JSONL |

---

## Consequências

### Positivas
- **Zero manutenção de providers:** CLI updates = novos modelos/features automaticamente
- **Reusa EngAI Gateway:** opencode já faz router, cache, RAG, guardrail, trace.json
- **Fallback chain robusto:** 4 providers, 3 fallbacks, circuit breaker
- **Privacidade:** ollama local como último recurso (dados não saem da máquina)
- **Observabilidade nativa:** trace.json já existe, só agregar metadados

### Negativas
- **Overhead subprocess:** ~50ms spawn + pipe vs ~10ms HTTP direto
- **Dependência externa:** CLIs devem estar instalados no PATH
- **Parsing JSONL:** Precisa parser incremental robusto (streaming)

### Riscos Mitigados
| Risco | Mitigação |
|-------|-----------|
| CLI flakiness | Health check prévio (`opencode --version`); timeout 30s; retry 2x; circuit breaker 5 min |
| JSONL parsing errors | Parser tolerante (skip linhas inválidas); fallback para resposta completa no final |
| Auth expirada | Detectar erro 401 → sugerir `opencode login` / `gh auth login` via TUI notification |
| Version mismatch | Pin versões mínimas no README; testar em CI com versões fixas |

---

## Implementação (Resumo)

### Traits
```rust
// ferro-gateway/src/lib.rs
pub trait LlmGateway: Send + Sync {
    async fn stream_chat(&self, req: ChatRequest) -> Result<BoxStream<ChatEvent>, GatewayError>;
    async fn get_models(&self) -> Result<Vec<ModelInfo>, GatewayError>;
    fn provider_name(&self) -> &'static str;
}

pub struct ChatRequest {
    pub messages: Vec<ChatMessage>,
    pub model: Option<String>,
    pub temperature: Option<f32>,
    pub max_tokens: Option<u32>,
    pub tools: Vec<ToolDefinition>,
    pub stream: bool, // sempre true para TUI
}

pub enum ChatEvent {
    Text(String),
    ToolCall(ToolCall),
    ToolResult(ToolResult),
    Done(FinishReason),
    Error(GatewayError),
}
```

### CliWrapperGateway
```rust
impl LlmGateway for CliWrapperGateway {
    async fn stream_chat(&self, req: ChatRequest) -> Result<BoxStream<ChatEvent>, GatewayError> {
        // 1. Check semantic cache
        if let Some(cached) = self.cache.get(&cache_key(&req)).await {
            return Ok(cached.into_stream());
        }
        
        // 2. Try providers in order
        let providers = std::iter::once(&self.primary).chain(&self.fallbacks);
        for provider in providers {
            match self.call_provider(provider, &req).await {
                Ok(stream) => {
                    // 3. Tee stream: cache + trace + return
                    return Ok(self.tee_stream(stream, &req).await);
                }
                Err(e) if e.is_retryable() => {
                    // retry logic + next provider
                    continue;
                }
                Err(e) => return Err(e),
            }
        }
        Err(GatewayError::AllProvidersFailed)
    }
}
```

### Semantic Cache
```rust
// SQLite + candle embeddings
pub struct SemanticCache {
    pool: SqlitePool,
    embedder: CandleEmbedder, // local, no network
    threshold: f32, // 0.95 default
}

impl SemanticCache {
    pub async fn get(&self, prompt: &str, model: &str) -> Option<CachedResponse> {
        let embedding = self.embedder.embed(prompt).await?;
        let rows = sqlx::query!(
            "SELECT response, embedding FROM cache WHERE model = ? AND embedding <-> ? < ?",
            model, embedding, 1.0 - self.threshold
        ).fetch_all(&self.pool).await?;
        // cosine similarity > threshold
        rows.into_iter().max_by_key(|r| cosine_sim(&r.embedding, &embedding))
    }
}
```

---

## Validação

- [ ] `ferro chat "teste"` → usa opencode, stream JSONL renderiza no TUI
- [ ] Kill opencode mid-stream → fallback para claude automaticamente
- [ ] Mesmo prompt 2x → cache hit < 10ms, trace mostra `cache_hit: true`
- [ ] `trace.json` contém: model, tokens, cost, latency, cache_hit, agent_id, session_id
- [ ] ollama local funciona sem internet (fallback final)
- [ ] Benchmark: overhead < 100ms vs CLI direto

---

## Referências
- `docs/02-benchmarks.md` Seção 3 (CLI Wrappers comparativo)
- EngAI Global Instructions: `~/.config/opencode/instructions/engai-global.md` Regra #1
- opencode gateway: `src/ai/gateway` (repo interno)
- JSONL streaming spec: https://github.com/opencode-ai/opencode/blob/main/docs/streaming.md