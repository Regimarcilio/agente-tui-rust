# ADR-007: Agent Model — Native Rust Trait + Dynamic Registry + Cargo Features

**Status:** Accepted  
**Data:** 28/10/2026  
**Autor:** PLANMASTER  
**Relacionado:** RF-003, RF-004, RF-005, RF-016, RNF-006, `docs/02-benchmarks.md` Seção 2

---

## Contexto

O FERRO embute 19 agentes especializados como módulos nativos. Requisitos:
- **Zero overhead:** Monomorfização, sem boxing desnecessário
- **Controle total:** Loop de agente, tool calling, state, permissions em Rust
- **WASM nativo:** wasmtime para sandbox plugins terceiros
- **Hot reload:** dylib nativo + WASM
- **Compilação seletiva:** Cargo features por agente
- **Registry dinâmico:** Descoberta runtime, auto-seleção por intent

Frameworks avaliados (ver benchmarks):
- **Custom Trait + Modules (escolhido)** — Rust nativo, zero overhead, controle total
- **rig** — Framework completo, overhead médio, sem WASM/hot reload
- **llm-chain** — Chain-based, overhead médio
- **async-openai** — Só OpenAI
- **agentic** — Framework, overhead alto, manutenção baixa
- **langchain-rust** — Port do LangChain, abstração pesada, overhead alto

---

## Decisão

**Agent Trait nativo + Registry dinâmico + Cargo features para compilação seletiva.**

### Agent Trait (ferro-core)

```rust
// ferro-core/src/agent.rs
use async_trait::async_trait;
use futures::Stream;
use std::fmt::Debug;

pub type AgentId = String;
pub type AgentOutputStream = BoxStream<Result<AgentOutput, AgentError>>;

#[async_trait]
pub trait Agent: Send + Sync + Debug + 'static {
    fn id(&self) -> AgentId;
    fn name(&self) -> &'static str;
    fn description(&self) -> &'static str;
    fn capabilities(&self) -> AgentCapabilities;
    
    async fn handle(
        &self,
        ctx: AgentContext,
        input: AgentInput,
    ) -> Result<AgentOutputStream, AgentError>;
    
    fn configure(&mut self, config: AgentConfig) -> Result<(), ConfigError>;
}

#[derive(Debug, Clone, Default)]
pub struct AgentCapabilities {
    pub can_write_code: bool,
    pub can_read_files: bool,
    pub can_run_tests: bool,
    pub can_git_commit: bool,
    pub can_call_llm: bool,
    pub can_call_mcp: bool,
    pub can_spawn_agents: bool,
    pub requires_network: bool,
    pub max_concurrent_tasks: usize,
}

#[derive(Debug, Clone)]
pub struct AgentContext {
    pub session_id: SessionId,
    pub workspace: PathBuf,
    pub config: Arc<Config>,
    pub llm_gateway: Arc<dyn LlmGateway>,
    pub mcp_client: Arc<McpClient>,
    pub git: Arc<GitService>,
    pub fs: Arc<FsService>,
    pub cancellation: CancellationToken,
}

#[derive(Debug, Clone)]
pub struct AgentInput {
    pub text: String,
    pub attachments: Vec<Attachment>,
    pub metadata: HashMap<String, Value>,
}

#[derive(Debug, Clone)]
pub enum AgentOutput {
    Text(String),
    ToolCall(ToolCall),
    ToolResult(ToolResult),
    SpawnAgent { agent_id: AgentId, input: AgentInput },
    Done(Result<(), AgentError>),
}
```

### Agent Registry (ferro-agents)

```rust
// ferro-agents/src/registry.rs
use inventory; // Para registro automático via macro

pub struct AgentRegistry {
    agents: HashMap<AgentId, Box<dyn Agent>>,
    metadata: HashMap<AgentId, AgentMetadata>,
    selector: AgentSelector,
}

#[derive(Debug, Clone)]
pub struct AgentMetadata {
    pub id: AgentId,
    pub name: &'static str,
    pub description: &'static str,
    pub capabilities: AgentCapabilities,
    pub category: AgentCategory,
    pub keywords: Vec<&'static str>, // Para auto-seleção
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum AgentCategory {
    Planning,
    Backend,
    Frontend,
    AI,
    Security,
    DevOps,
    QA,
    Mobile,
    Payments,
    Offers,
    Social,
    Design,
    Prototype,
    Issues,
    Knowledge,
    Data,
    CodeInspection,
    Debugging,
    TDD,
}

impl AgentRegistry {
    pub fn new() -> Self {
        let mut registry = Self {
            agents: HashMap::new(),
            metadata: HashMap::new(),
            selector: AgentSelector::new(),
        };
        
        // Registrar agentes built-in (via inventory ou macro)
        registry.register_builtin_agents();
        registry
    }
    
    fn register_builtin_agents(&mut self) {
        // Macro registra automaticamente agentes com #[agent]
        // Ou registro manual condicional a Cargo features
        #[cfg(feature = "planmaster")]
        self.register(Box::new(PlanmasterAgent::new()), AgentMetadata { ... });
        
        #[cfg(feature = "backdev")]
        self.register(Box::new(BackdevAgent::new()), AgentMetadata { ... });
        
        // ... 17 outros agentes
    }
    
    pub fn register(&mut self, agent: Box<dyn Agent>, metadata: AgentMetadata) {
        let id = agent.id();
        self.metadata.insert(id.clone(), metadata);
        self.agents.insert(id, agent);
    }
    
    pub fn get(&self, id: &AgentId) -> Option<&dyn Agent> {
        self.agents.get(id).map(|b| b.as_ref())
    }
    
    pub fn list(&self) -> Vec<AgentMetadata> {
        self.metadata.values().cloned().collect()
    }
    
    pub async fn auto_select(&self, intent: &str) -> Result<AgentId> {
        self.selector.select(intent, &self.metadata).await
    }
}
```

### Cargo Features por Agente

```toml
# ferro-agents/Cargo.toml
[features]
default = [
    "planmaster", "backdev", "frontdev", "engai", "kalimaster",
    "opsmaster", "qamaster"  # MVP: 7 agentes essenciais
]

# Agentes individuais
planmaster = ["ferro-agents-planmaster"]
backdev = ["ferro-agents-backdev"]
frontdev = ["ferro-agents-frontdev"]
engai = ["ferro-agents-engai"]
kalimaster = ["ferro-agents-kalimaster"]
opsmaster = ["ferro-agents-opsmaster"]
qamaster = ["ferro-agents-qamaster"]
mobilemaster = ["ferro-agents-mobilemaster"]
paymaster = ["ferro-agents-paymaster"]
offermaster = ["ferro-agents-offermaster"]
socialmaster = ["ferro-agents-socialmaster"]
stylemaster = ["ferro-agents-stylemaster"]
protomaster = ["ferro-agents-protomaster"]
issuemaster = ["ferro-agents-issuemaster"]
aprendiz = ["ferro-agents-aprendiz"]
datamaster = ["ferro-agents-datamaster"]
code-inspection = ["ferro-agents-code-inspection"]
debugging = ["ferro-agents-debugging"]
tdd = ["ferro-agents-tdd"]

# Meta-features
all = [
    "planmaster", "backdev", "frontdev", "engai", "kalimaster",
    "opsmaster", "qamaster", "mobilemaster", "paymaster", "offermaster",
    "socialmaster", "stylemaster", "protomaster", "issuemaster",
    "aprendiz", "datamaster", "code-inspection", "debugging", "tdd"
]

minimal = []  # Apenas core, sem agentes built-in
```

### Auto-Seleção por Intent (Embedding-based)

```rust
// ferro-agents/src/selector.rs
pub struct AgentSelector {
    embedder: CandleEmbedder,
    agent_embeddings: HashMap<AgentId, Vec<f32>>,
}

impl AgentSelector {
    pub fn new() -> Self {
        // Embeddings pré-computados das descriptions + keywords
        let embeddings = compute_agent_embeddings();
        Self { embedder: CandleEmbedder::new(), agent_embeddings: embeddings }
    }
    
    pub async fn select(&self, intent: &str, metadata: &HashMap<AgentId, AgentMetadata>) -> Result<AgentId> {
        let intent_emb = self.embedder.embed(intent).await?;
        
        let best = metadata.iter()
            .filter(|(_, m)| m.capabilities.can_call_llm) // Só agentes que usam LLM
            .max_by_key(|(id, _)| {
                let agent_emb = &self.agent_embeddings[id];
                cosine_similarity(&intent_emb, agent_emb) as u32
            });
        
        best.map(|(id, _)| id.clone())
            .ok_or_else(|| AgentError::NoSuitableAgent)
    }
}
```

---

## Consequências

### Positivas
- **Zero overhead:** Monomorfização, `async_trait` com `Send + Sync`, sem `Box<dyn Fn>` desnecessário
- **Controle total:** Loop, tools, state, permissions, cancellation — tudo em Rust
- **Compilação seletiva:** `cargo build --features planmaster,backdev` = binary menor
- **Hot reload nativo:** `libloading` para dylib; wasmtime para WASM (ADR-001)
- **Auto-seleção inteligente:** Embeddings semânticos > keyword matching
- **Testabilidade:** Trait mockável com `mockall`; unit tests por agente

### Negativas
- **19 crates de agentes:** Mais manutenção, mas isolamento claro
- **Compile time:** Features ajudam, mas full build = 19 crates
- **Registry complexity:** Dynamic loading + features + WASM = código de bootstrap

### Riscos Mitigados
| Risco | Mitigação |
|-------|-----------|
| Code bloat 19 agentes | Cargo features; `minimal` feature para core apenas; dynamic loading preferido |
| Compile time alto | Workspace modular; `cargo check` incremental; `sccache` em CI |
| Trait object overhead | `async_trait` usa `BoxFuture` internamente; aceitável para boundary agente |
| Auto-seleção errada | Fallback: command palette manual; logs de seleção para debug |

---

## Validação

- [ ] `cargo build --features minimal` compila apenas core (sem agentes)
- [ ] `cargo build --features planmaster,backdev` compila 2 agentes + registry
- [ ] `cargo build --features all` compila 19 agentes < 5 min
- [ ] Registry lista apenas agents das features habilitadas
- [ ] `registry.auto_select("planeja feature")` → retorna `planmaster`
- [ ] `registry.auto_select("auditoria segurança")` → retorna `kalimaster`
- [ ] Hot reload dylib: `ferro plugin reload <id>` recarrega agente sem restart
- [ ] WASM agent: `ferro plugin load-wasm <path>` carrega sandboxed
- [ ] Unit tests: cada agente testável isoladamente com `mockall`

---

## Referências
- `docs/02-benchmarks.md` Seção 2 (Agent Frameworks comparativo)
- async-trait: https://docs.rs/async-trait/
- inventory: https://docs.rs/inventory/ (registro automático)
- candle: https://github.com/huggingface/candle (embeddings locais)