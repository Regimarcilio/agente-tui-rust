# Benchmarks Referenciais — Agente TUI Rust (FERRO)

**Versão:** 0.1  
**Data:** 28/10/2026  
**Autor:** PLANMASTER  
**Branch:** develop

---

## 1. Frameworks TUI (Rust)

### 1.1 Comparativo Principal

| Critério | **ratatui** (escolhido) | crossterm + tui-rs (legacy) | cursive | iocraft | egui (GUI/TUI hybrid) |
|----------|------------------------|----------------------------|---------|---------|----------------------|
| **Stars GitHub** | 23.1k | 8.2k (tui-rs, archived) | 3.8k | 1.2k | 24.5k |
| **Manutenção** | Ativa (2024+) | Arquivado (fork → ratatui) | Ativa | Ativa | Ativa |
| **Arquitetura** | Workspace modular (core/widgets/backends) | Monolítico | Declarativo (React-like) | Declarativo (componentes) | Immediate mode |
| **Backends** | crossterm, termion, termina, termwiz | crossterm, termion | crossterm, termion, pancurses | crossterm | winit, web, raw |
| **no_std** | ✅ (ratatui-core) | ❌ | ❌ | ❌ | ❌ |
| **Widgets ricos** | ✅ (widgets crate + ecossistema) | Básico | Rico (views) | Componentes compostos | Rico |
| **Syntax highlight** | Via `ratatui-code-editor` (tree-sitter) | Manual | Manual | Manual | Via `egui_extras` |
| **Diff inline** | Custom (ex: `giff`, `Livediff`) | Manual | Manual | Manual | Manual |
| **LSP client** | Via `tower-lsp` + custom widget | Não nativo | Não nativo | Não nativo | `egui_lsp` experimental |
| **Animation/Motion** | `tachyonfx` (50+ effects, WASM, no_std) | Básico | Básico | Limitado | `egui_animate` |
| **Theming** | `opaline` (token-based, 20 themes) | Manual | Temas básicos | CSS-like | Temas ricos |
| **File explorer** | `ratatui-explorer`, `Yazi` (app) | Não | Não | Não | Não |
| **Git UI** | `gitui`, `gitu`, `gitty` (apps) | Não | Não | Não | Não |
| **Performance** | 60fps fácil, layout cache | 60fps | 30-60fps | 60fps | 60fps+ (GPU) |
| **Bundle size** | ~2MB (core) + widgets | ~1.5MB | ~3MB | ~2MB | ~5MB+ |
| **Curva aprendizado** | Média | Baixa | Média | Média-alta | Alta |
| **Comunidade** | Discord ativo, Forum, Matrix | Legado | Discord | Pequena | Discord grande |
| **Licença** | MIT | MIT | MIT | MIT | MIT |

### 1.2 Decisão: **ratatui**

**Justificativa:**
- **Ecossistema maduro**: 23k stars, workspace modular (core/widgets/backends), compatível no_std
- **Widgets ricos prontos**: `ratatui-code-editor` (tree-sitter), `ratatui-explorer`, `rat-widget` (forms/tables), `tachyonfx` (animações nível Obra)
- **Apps de referência**: `gitui` (git), `Yazi` (files), `opencode-stats`, `claude-code-rust`, `borg` (AI agents) — provam viabilidade "Obra"
- **Backends multiplataforma**: crossterm (Win/Mac/Linux), termwiz (advanced), termion (Unix)
- **Extensibilidade**: WASM via `ratatui-wasm`, plugins via `ratatui-wasm` (host-controlled capabilities)
- **Theming profissional**: `opaline` (token-based, gradients, theme selector widget)

### 1.3 Crates Complementares Escolhidos

| Necessidade | Crate | Versão | Notas |
|-------------|-------|--------|-------|
| **Core TUI** | `ratatui` | 0.30+ | Workspace modular |
| **Backend** | `ratatui-crossterm` | 0.30+ | Cross-platform |
| **Animações** | `tachyonfx` | 0.6+ | 50+ effects, DSL, WASM, no_std |
| **Theming** | `opaline` | 0.3+ | Token-based, 20 themes, selector widget |
| **Syntax Highlight** | `ratatui-code-editor` / `tui-syntax-highlight` / `syntect` | latest | tree-sitter based |
| **Diff** | `diffrs` / `similar` | latest | Myers diff, inline |
| **LSP Client** | `tower-lsp` + `lsp-types` | latest | rust-analyzer, typescript, etc. |
| **Git** | `git2` (libgit2) | 0.18+ | Vendored libgit2, ssh/https features |
| **WASM Runtime** | `wasmtime` | 20+ | Sandbox agents, WASI |
| **Config** | `figment` / `config` | latest | TOML, env, layering, hot-reload |
| **FS Watch** | `notify` / `watchexec` | latest | Cross-platform file watch |
| **SQLite** | `sqlx` / `rusqlite` | latest | Sessions, cache, history |
| **Embeddings** | `candle` / `onnxruntime` | latest | Semantic cache local |
| **Fuzzy Search** | `skim` / `fzf` Rust bindings | latest | Command palette |
| **Logging** | `tracing` + `tracing-subscriber` | latest | Structured, multi-target |
| **CLI Args** | `clap` (derive) | 4.5+ | Subcommands, completions |
| **Async** | `tokio` (full) | 1.40+ | Runtime, sync, fs, net |
| **Serialization** | `serde` + `serde_json` + `toml` | latest | Config, messages, traces |

---

## 2. Agent Frameworks / Orquestração (Rust)

### 2.1 Comparativo

| Critério | **Custom Trait + Module System (escolhido)** | rig | llm-chain | async-openai |_agentic_ | langchain-rust |
|----------|----------------------------------------------|-----|-----------|--------------|----------|----------------|
| **Abordagem** | Traits nativos + módulos Rust | Framework completo | Chain-based | OpenAI SDK | Framework | Port do LangChain |
| **Controle total** | ✅ Máximo | Médio | Médio | Alto (só OpenAI) | Médio | Baixo (abstração) |
| **Multi-LLM** | Via CLI wrapper | ✅ Providers | ✅ Providers | ❌ Só OpenAI | ✅ Providers | ✅ Providers |
| **Tools/MCP** | Nativo (JSON-RPC) | ✅ | ✅ | Manual | ✅ | ✅ |
| **Streaming** | Nativo (async stream) | ✅ | ✅ | ✅ | ✅ | ✅ |
| **State management** | Custom (SQLite + memory) | Built-in | Built-in | Manual | Built-in | Built-in |
| **WASM agents** | ✅ wasmtime | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Hot reload agents** | ✅ Dynamic libs | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Overhead** | Zero (monomorphizado) | Médio | Médio | Baixo | Alto | Alto |
| **Curva aprendizado** | Baixa (Rust nativo) | Média | Média | Baixa | Alta | Alta |
| **Manutenção** | Própria | Ativa | Ativa | Ativa | Baixa | Baixa |

### 2.2 Decisão: **Trait Nativo + Módulos Rust**

**Justificativa:**
- **Zero overhead**: Monomorfização, sem boxing desnecessário, `async fn` nativo
- **Controle total**: Loop de agente, tool calling, state, permissions — tudo em Rust
- **WASM nativo**: `wasmtime` para sandbox de agentes terceiros (capability-based)
- **Hot reload**: `libloading` para dylib nativo + `wasmtime` para WASM
- **Integração TUI**: Agents como widgets ratatui, streaming direto no buffer
- **19 agentes = 19 módulos**: Cargo features para compilação seletiva (`--features planmaster,backdev`)

**Arquitetura do Agent Trait:**
```rust
pub trait Agent: Send + Sync + 'static {
    fn id(&self) -> AgentId;
    fn name(&self) -> &'static str;
    fn description(&self) -> &'static str;
    fn capabilities(&self) -> AgentCapabilities;
    
    async fn handle(
        &self, 
        ctx: AgentContext, 
        input: AgentInput
    ) -> Result<AgentOutputStream, AgentError>;
    
    fn configure(&mut self, config: AgentConfig) -> Result<(), ConfigError>;
}
```

---

## 3. CLI Wrappers LLM (opencode/claude/gh copilot)

### 3.1 Comparativo de Estratégias

| Estratégia | **Subprocess + JSONL Streaming (escolhida)** | HTTP Direto (reqwest) | SDK Oficial (async-openai, anthropic-sdk) | Gateway Próprio (Rust) |
|------------|----------------------------------------------|----------------------|------------------------------------------|------------------------|
| **Provedores** | Todos (CLI abstrai) | Manual por provider | 1-2 por SDK | Todos (custom) |
| **Auth** | CLI gerencia (login, tokens) | Manual (env, keyring) | SDK gerencia | Custom (keyring, OAuth) |
| **Model routing** | CLI (opencode router) | Manual | SDK | Custom (OpenRouter) |
| **Cache semântico** | Local (SQLite + embeddings) | Local | Não nativo | Custom |
| **Cost tracking** | Parse JSONL output | Manual headers | SDK response | Custom |
| **Latência overhead** | ~50ms spawn + pipe | ~10ms HTTP | ~10ms HTTP | ~5ms (in-process) |
| **Manutenção** | Baixa (CLI update) | Média (API changes) | Média (SDK updates) | Alta (próprio) |
| **Privacidade** | Local (sem telemetry CLI) | Total | Telemetry SDK | Total |
| **Fallback chain** | Fácil (try opencode → claude → gh) | Manual | Manual | Custom |

### 3.2 Decisão: **Subprocess Manager com JSONL Streaming**

**Justificativa:**
- **Reusa seu gateway atual**: `src/ai/gateway` já existe no workspace (EngAI Global)
- **OpenCode/CLI já fazem**: router (OpenRouter/Ollama), cache semântico, RAG, guardrail, trace.json
- **Fallback chain nativo**: `opencode` → `claude` → `gh copilot` → `ollama` local
- **Streaming JSONL**: Parse incremental, render no TUI em tempo real
- **Custo/latência**: `trace.json` já gerado pelo gateway — reaproveitar

**Interface Abstrata (Gateway Trait):**
```rust
pub trait LlmGateway: Send + Sync {
    async fn stream_chat(
        &self,
        req: ChatRequest,
    ) -> Result<BoxStream<ChatEvent>, GatewayError>;
    
    async fn get_models(&self) -> Result<Vec<ModelInfo>, GatewayError>;
    
    fn provider_name(&self) -> &'static str;
}

pub struct CliWrapperGateway {
    primary: CliProvider,      // opencode
    fallbacks: Vec<CliProvider>, // claude, gh copilot, ollama
    cache: SemanticCache,
    tracer: TraceCollector,
}
```

---

## 4. WASM Runtime para Plugins/Agentes

### 4.1 Comparativo

| Critério | **wasmtime (escolhido)** | wasmer | wasmi | wamr | native dylib (cdylib) |
|----------|--------------------------|--------|-------|------|----------------------|
| **Performance** | Cranelift JIT/AOT (rápido) | Cranelift/LLVM | Interpretador (lento) | AOT (rápido) | Nativo (mais rápido) |
| **WASI Support** | ✅ Completo (preview 1/2) | ✅ Completo | Parcial | ✅ Completo | N/A |
| **Component Model** | ✅ Preview | ✅ Preview | ❌ | ❌ | N/A |
| **Sandbox/Security** | ✅ Capability-based | ✅ Capability-based | Básico | ✅ | ❌ (full access) |
| **Embedding Rust** | ✅ Excelente | ✅ Boa | ✅ Boa | C API | Native |
| **Hot reload** | ✅ Instance recreation | ✅ | ✅ | ❌ | `libloading` |
| **Fuel metering** | ✅ Built-in | ✅ | ✅ | ❌ | Manual |
| **Memory limits** | ✅ Configurável | ✅ | ✅ | ✅ | Process-level |
| **Tamanho binary** | ~8MB | ~12MB | ~2MB | ~4MB | N/A |
| **MSRV** | 1.70+ | 1.70+ | 1.60+ | C toolchain | 1.60+ |

### 4.2 Decisão: **wasmtime + native dylib (hibrido)**

- **Fase 1 (MVP)**: Native dylib (`cdylib`) via `libloading` — performance máxima, hot reload simples
- **Fase 2 (v1.1+)**: wasmtime para agentes terceiros/sandbox — capability-based security, WASI
- **Component Model**: Preparar interfaces WIT para agentes WASM futuros

---

## 5. LSP Client Integration

### 5.1 Comparativo

| Critério | **tower-lsp + lsp-types (escolhido)** | lsp-client | rust-analyzer LSP lib | code-intel |
|----------|--------------------------------------|------------|----------------------|------------|
| **Maturity** | Usado por helix, lapce, vscode-rust | Básico | Interno (não público) | Experimental |
| **Async** | ✅ tower (tokio) | ✅ | N/A | ✅ |
| **Features** | Completo (LSP 3.17) | Básico | Completo | Parcial |
| **Integration** | Widget custom ratatui | Manual | Não exposto | Manual |
| **Diagnostics UI** | Custom (inline, panel) | Não | Não | Não |

### 5.2 Decisão: **tower-lsp + widget ratatui custom**

- Integrar como widget `LspWidget` no layout
- Diagnostics inline (virtual text) + panel lateral
- Code actions via command palette
- Go-to-definition, hover, completion (future)

---

## 6. Diff Inline / Git UI

### 6.1 Referências Existentes (ratatui ecosystem)

| App | Feature | Crate Base |
|-----|---------|------------|
| `gitui` | Git staging, diff, log, blame | `git2`, `ratatui` |
| `gitu` | Magit-like, diff inline | `git2`, `ratatui` |
| `gitty` | GitHub Desktop-like, line staging | `git2`, `ratatui` |
| `giff` | Interactive rebase, diff | `git2`, `ratatui` |
| `Livediff` | Real-time file diff monitoring | `notify`, `diffrs`, `ratatui` |
| `declutter` | AI code diff viewer (hide comments/tests) | `similar`, `ratatui` |

### 6.2 Decisão: **git2 + diffrs + widget custom**

- `git2` para operações Git (stage, commit, push, log)
- `diffrs` (Myers) ou `similar` para diff algorithm
- Widget `DiffView` com: side-by-side, unified, inline, syntax highlight
- Integrar com `ratatui-code-editor` para navegação no diff

---

## 7. Tabela Resumo — Stack Tecnológico Definitivo

| Camada | Escolha | Versão Mínima | Cargo Features |
|--------|---------|---------------|----------------|
| **Runtime** | `tokio` | 1.40 | `full` |
| **TUI Core** | `ratatui` | 0.30 | `crossterm`, `macros` |
| **Backend** | `ratatui-crossterm` | 0.30 | - |
| **Animações** | `tachyonfx` | 0.6 | `dsl`, `std` |
| **Theming** | `opaline` | 0.3 | - |
| **Syntax Highlight** | `syntect` + `tree-sitter` | latest | `syntect` features |
| **Code Editor** | `ratatui-code-editor` | latest | - |
| **Diff** | `diffrs` | latest | - |
| **LSP** | `tower-lsp` + `lsp-types` | latest | - |
| **Git** | `git2` | 0.18 | `https`, `ssh`, `vendored-libgit2` |
| **WASM** | `wasmtime` | 20 | `wasi`, `component-model` |
| **Dynamic Libs** | `libloading` | 0.8 | - |
| **Config** | `figment` | 0.10 | `toml`, `env`, `serde` |
| **FS Watch** | `notify` | 6.0 | - |
| **Database** | `sqlx` | 0.7 | `sqlite`, `runtime-tokio`, `migrate` |
| **Embeddings** | `candle` | 0.5 | `cuda` (opcional) |
| **Fuzzy** | `skim` | 0.12 | - |
| **Logging** | `tracing` + `tracing-subscriber` | 0.1 | `env-filter`, `json` |
| **CLI** | `clap` | 4.5 | `derive`, `env` |
| **Serialization** | `serde` + `serde_json` + `toml` | latest | `derive` |
| **Error Handling** | `color-eyre` / `anyhow` | latest | - |
| **Testing** | `ratatui-test` / `tokio-test` | latest | - |

---

## 8. Benchmarks de Performance (Targets)

| Métrica | Target | Ferramenta de Medição |
|---------|--------|----------------------|
| **Cold start** | < 2s | `hyperfine 'ferro --version'` |
| **Memory idle** | < 50MB RSS | `ps_record` / `memusage` |
| **TUI frame time** | < 16ms (60fps) | `tracing` spans + `tachyonfx` |
| **Agent switch** | < 50ms | Benchmark interno |
| **LLM first token** | < 500ms (local) / < 2s (remote) | `trace.json` do gateway |
| **LLM streaming** | > 50 tok/s | `trace.json` |
| **Semantic cache hit** | < 10ms | SQLite query time |
| **Git status** | < 100ms | `git2` benchmark |
| **LSP startup** | < 1s (rust-analyzer) | `tower-lsp` init |
| **Config hot-reload** | < 100ms | `notify` + `figment` reload |

---

## 9. Riscos Técnicos Identificados

| Risco | Mitigação |
|-------|-----------|
| **ratatui 0.30 breaking changes** | Pin versão exata no `Cargo.lock`, testar migração 0.29→0.30 |
| **git2-rs segfaults (libgit2)** | Feature `vendored-libgit2`, fallback para `git` CLI subprocess |
| **wasmtime 20+ API instável** | Travar versão, usar `wasmtime` apenas v1.1+, testar extensivamente |
| **tower-lsp complexidade** | Começar só com rust-analyzer, diagnostics inline; completion depois |
| **tree-sitter grammars bundling** | `tree-sitter` grammars como features opcionais, download on-demand |
| **CLI wrapper flakiness** | Timeout + retry + fallback chain, health check prévio |
| **Bundle size > 15MB** | Cargo features por agente, `strip = true`, `lto = true`, `panic = "abort"` |

---

## 10. Próximos Passos

1. **FASE 3** — SRS completo com requisitos funcionais (RF) e não-funcionais (NFR)
2. **FASE 4** — UML (Casos de Uso, Classes, Sequência) + ADRs (arquitetura de plugin, gateway LLM, sandbox)
3. **FASE 5** — Backlog Scrum: Épicos → Stories → Sprints (6 sprints de 2 semanas)
4. **FASE 6** — Plano de entrega: estrutura Cargo workspace, CI, release, handoff

---

> **Referências consultadas:**
> - ratatui: https://github.com/ratatui/ratatui (23.1k ⭐, workspace modular)
> - ratatui ARCHITECTURE.md: modular crates, backend abstraction
> - awesome-ratatui: 100+ apps/libs (gitui, Yazi, borg, claude-code-rust, opencode-stats)
> - tachyonfx: 1.3k ⭐, 50+ effects, DSL, WASM, no_std
> - opaline: token-based theming, 20 themes, selector widget
> - tokio: 33.5k ⭐, runtime padrão Rust
> - git2-rs: 2.1k ⭐, libgit2 bindings, vendored libgit2
> - wasmtime: 18.8k ⭐, Bytecode Alliance, WASI, Component Model
> - tree-sitter: 27.3k ⭐, incremental parsing, Rust bindings
> - tower-lsp: LSP implementation usado por helix, lapce