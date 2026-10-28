# ADR-003: Sandbox Strategy — wasmtime Capability-based (WASI)

**Status:** Accepted  
**Data:** 28/10/2026  
**Autor:** PLANMASTER  
**Relacionado:** ADR-001, RF-016, RNF-003, RF-017

---

## Contexto

Plugins de terceiros (não assinados pelo core team) precisam de isolamento. Requisitos:
- **Execução não confiável:** Código arbitrário WASM
- **Capability-based security:** Princípio menor privilégio
- **WASI support:** Acesso controlado a FS, rede, env, clock
- **Resource limits:** Fuel (instruções), memory, tempo
- **Determinismo:** Mesma WASM = mesmo comportamento
- **Performance:** JIT/AOT (Cranelift), não interpretador

Opções (ver `docs/02-benchmarks.md` Seção 4):
- **wasmtime** — Bytecode Alliance, Cranelift JIT/AOT, WASI completo, Component Model, capability-based, fuel metering
- **wasmer** — Similar, mas menos integração Rust nativa
- **wasmi** — Interpretador puro, lento, sem JIT
- **wamr** — C API, AOT, menos ergonomic em Rust
- **Process isolation** — Overhead IPC, não granular

---

## Decisão

**Usar wasmtime como runtime WASM exclusivo para plugins sandboxed.**

### Configuração de Sandbox por Plugin

```rust
// ferro-plugins/src/wasm_sandbox.rs
use wasmtime::{Engine, Store, Module, Instance, Linker, Config};
use wasmtime_wasi::{WasiCtx, WasiCtxBuilder, WasiView};

pub struct WasmSandbox {
    engine: Engine,
    linker: Linker<WasmState>,
}

pub struct WasmState {
    ctx: WasiCtx,
    caps: Capabilities,
    fuel_consumed: u64,
}

impl WasmSandbox {
    pub fn new(caps: Capabilities) -> Result<Self> {
        let mut config = Config::new();
        config.consume_fuel(true);           // Fuel metering
        config.memory_limit(caps.memory_limit); // Memory limit
        config.wasm_component_model(true);   // Component Model
        
        let engine = Engine::new(&config)?;
        let mut linker = Linker::new(&engine);
        wasmtime_wasi::add_to_linker(&mut linker, |s: &mut WasmState| &mut s.ctx)?;
        
        // Host functions customizadas (ex: MCP calls, LLM gateway)
        linker.func_wrap("ferro", "mcp_call", |caller: Caller<WasmState>, ...| { ... })?;
        linker.func_wrap("ferro", "llm_stream", |caller: Caller<WasmState>, ...| { ... })?;
        
        Ok(Self { engine, linker })
    }
    
    pub fn instantiate(&self, wasm_bytes: &[u8], caps: Capabilities) -> Result<Instance> {
        let mut store = Store::new(&self.engine, WasmState::new(caps)?);
        store.add_fuel(caps.fuel_limit)?; // Limite de instruções
        
        let module = Module::new(&self.engine, wasm_bytes)?;
        let instance = self.linker.instantiate(&mut store, &module)?;
        
        // Verificar capabilities declaradas vs usadas (static analysis)
        self.validate_capabilities(&module, &caps)?;
        
        Ok(instance)
    }
}
```

### Capability Declaration (WIT + Runtime)

```wit
// ferro-plugin.wit - Interface do plugin
package ferro:plugin;

interface agent {
    // Metadata
    record metadata {
        name: string,
        version: string,
        description: string,
        capabilities: list<capability>,
    }
    
    // Capabilities solicitadas
    variant capability {
        fs-read(path: string),
        fs-write(path: string),
        net-connect(host: string, port: u16),
        env-get(var: string),
        clock-now,
        mcp-call(server: string, tool: string),
        llm-chat(model: string),
    }
    
    // Entry point
    handle: func(ctx: agent-context, input: agent-input) -> stream<agent-output>;
}

record agent-context {
    session-id: string,
    workspace: string,
    config: config,
}

record agent-input {
    text: string,
    attachments: list<attachment>,
    metadata: map<string, string>,
}

variant agent-output {
    text(string),
    tool-call(tool-call),
    tool-result(tool-result),
    done(result<_, error>),
}
```

### Validação Estática + Runtime

```rust
// Build-time: wasm-tools component wit --world plugin ferro-plugin.wasm
// Runtime: verificar imports vs capabilities declaradas
fn validate_capabilities(module: &Module, caps: &Capabilities) -> Result<()> {
    for import in module.imports() {
        match (import.module(), import.name()) {
            ("wasi_snapshot_preview1", "fd_write") | 
            ("wasi_snapshot_preview1", "path_open") => {
                if !caps.fs_write.iter().any(|p| import_has_path(p)) {
                    return Err(Error::CapabilityViolation("fs-write not declared"));
                }
            }
            ("wasi_snapshot_preview1", "sock_connect") => {
                if !caps.net_connect.iter().any(|(h,p)| import_matches(h,p)) {
                    return Err(Error::CapabilityViolation("net-connect not declared"));
                }
            }
            ("ferro", "mcp_call") => {
                if !caps.mcp_servers.contains(&imported_server) {
                    return Err(Error::CapabilityViolation("mcp server not declared"));
                }
            }
            _ => {}
        }
    }
    Ok(())
}
```

---

## Consequências

### Positivas
- **Segurança granular:** Capabilities declaradas no WIT + validadas em runtime
- **Performance:** Cranelift JIT ~80% velocidade nativa; AOT para produção
- **Ecossistema:** WASI padrão emergente; Component Model para multi-language
- **Resource control:** Fuel metering previne DoS; memory limit previne OOM
- **Determinismo:** WASM puro = reproduzível across platforms

### Negativas
- **Overhead ~8MB** no binary (wasmtime)
- **API instável** até wasmtime 1.0 (travar versão)
- **WASI preview 1/2 transição** — breaking changes possíveis
- **Component Model** ainda em preview; tooling imaturo

### Riscos Mitigados
| Risco | Mitigação |
|-------|-----------|
| wasmtime breaking changes | Travar versão exata (`wasmtime = "=20.0.0"`); testar upgrade em branch isolada |
| WASI incompatibilidade | Target `wasi-preview1` + `wasi-preview2` condicional; testar com wasi-testsuite |
| Plugin malicious fuel exhaustion | Fuel limit default 10M instruções; config por plugin; reset fuel em cada call |
| Memory leak host functions | `Store` limitado por plugin; drop = cleanup; `WasiView` gerencia recursos |

---

## Implementação (Resumo)

### Cargo Features
```toml
# ferro-plugins/Cargo.toml
[features]
default = ["dylib"]
dylib = ["libloading"]
wasm = ["wasmtime", "wasmtime-wasi", "wasm-tools"]
```

### Plugin Manifest (TOML + WIT)
```toml
# plugin.toml (bundled no .wasm ou separado)
[plugin]
name = "meu-agente-custom"
version = "0.1.0"
wasm = "plugin.wasm"
wit = "ferro-plugin.wit"

[capabilities]
fs_read = ["/workspace", "/home/user/.config/ferro"]
fs_write = ["/workspace/output"]
net_connect = ["api.github.com:443", "api.openai.com:443"]
env_get = ["GITHUB_TOKEN", "OPENAI_API_KEY"]
mcp_servers = ["git", "fs"]
fuel_limit = 50_000_000
memory_limit = 67_108_864  # 64MB
```

### Integração no Agent Registry
```rust
// ferro-agents/src/registry.rs
impl AgentRegistry {
    pub async fn load_wasm_agent(&mut self, path: &Path, manifest: PluginManifest) -> Result<AgentId> {
        let sandbox = WasmSandbox::new(manifest.capabilities.clone())?;
        let instance = sandbox.instantiate(&wasm_bytes, manifest.capabilities)?;
        
        // Criar wrapper que implementa Agent trait
        let agent = WasmAgentWrapper::new(instance, manifest);
        let id = AgentId::new(&manifest.name);
        
        self.register(Box::new(agent));
        Ok(id)
    }
}
```

---

## Validação

- [ ] Plugin WASM com `fs_read` apenas lê arquivos permitidos; escrita falha com trap
- [ ] Plugin sem `net_connect` falha ao tentar HTTP; erro claro no log
- [ ] Fuel limit: loop infinito WASM termina após N instruções; retorna erro controlado
- [ ] Memory limit: alocação > 64MB falha com `MemoryLimitExceeded`
- [ ] Component Model: plugin Rust + plugin Go (via tinygo) ambos carregam
- [ ] Binary size: `cargo build --features wasm` adiciona < 10MB

---

## Referências
- wasmtime docs: https://docs.wasmtime.dev/
- WASI preview 2: https://github.com/WebAssembly/WASI/blob/main/preview2/README.md
- Component Model: https://github.com/WebAssembly/component-model
- wasmtime-wasi: https://docs.rs/wasmtime-wasi/
- Capability-based security: https://en.wikipedia.org/wiki/Capability-based_security