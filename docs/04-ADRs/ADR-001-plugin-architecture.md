# ADR-001: Plugin Architecture — dylib nativo (MVP) → WASM (v1.1+)

**Status:** Accepted  
**Data:** 28/10/2026  
**Autor:** PLANMASTER  
**Relacionado:** RF-016, RNF-003, RNF-006

---

## Contexto

O FERRO precisa suportar plugins/agentes de terceiros sem recompilar o core. Requisitos:
- **Performance nativa** para plugins do time confiável
- **Sandbox seguro** para plugins não assinados/terceiros
- **Hot reload** sem reiniciar TUI
- **Interface estável** (semver) para ecossistema
- **Distribuição single binary** — plugins carregados dinamicamente

Opções avaliadas (ver `docs/02-benchmarks.md` Seção 4):
1. **Native dylib (`cdylib` + `libloading`)** — Performance nativa, hot reload simples, sem sandbox
2. **wasmtime (WASM/WASI)** — Sandbox capability-based, WASI, Component Model, ~8MB overhead
3. **wasmer** — Similar wasmtime, menos maduro em Rust
4. **wasmi** — Interpretador, lento, sem JIT
5. **Process isolation** — Overhead IPC, complexidade

---

## Decisão

**Fase 1 (MVP - Sprints 1-3): Native dylib (`cdylib` + `libloading`)**
- Plugins = `cdylib` Rust compilados separadamente
- Carregados via `libloading::Library` em `~/.config/ferro/plugins/`
- Implementam trait `Agent` (mesmo que agentes nativos)
- Hot reload: `PluginManager::hot_reload(plugin_id)` → unload + reload
- Apenas para time confiável (assinados, auditados)

**Fase 2 (v1.1+ - Sprint 4+): wasmtime para sandbox**
- Plugins terceiros compilados para WASM (WASI preview 1/2)
- `wasmtime` com capability-based security: `fs:read`, `fs:write`, `net:http`, `env:read`
- Component Model (WIT) para interface estável cross-language
- Fuel metering + memory limits para DoS protection
- Fallback: dylib ainda suportado para performance crítica

**Interface Comum (ambas fases):**
```rust
// Trait unificado (dylib) ou WIT (WASM)
pub trait Plugin: Send + Sync {
    fn metadata(&self) -> PluginMetadata;
    fn init(&mut self, config: &Config) -> Result<()>;
    fn shutdown(&mut self) -> Result<()>;
    // Para agentes: downcast para Agent trait
    fn as_agent(&self) -> Option<&dyn Agent>;
}
```

---

## Consequências

### Positivas
- **MVP rápido:** dylib é Rust nativo, zero overhead, hot reload trivial
- **Segurança progressiva:** WASM só quando necessário (plugins terceiros)
- **Ecossistema:** Preparação para Component Model (futuro multi-language)
- **Performance:** Agentes core + plugins confiáveis = velocidade nativa

### Negativas
- **Duas implementações:** PluginManager precisa suportar ambos loaders
- **WASM complexity:** wasmtime API instável até 1.0; travar versão
- **dylib sem sandbox:** Plugins nativos têm acesso total ao processo

### Riscos Mitigados
| Risco | Mitigação |
|-------|-----------|
| wasmtime breaking changes | Travar versão exata no `Cargo.lock`; testar upgrade isolado |
| dylib ABI instability | Usar `cdylib` com `#[no_mangle]` C API estável; trait via `Any` downcast |
| Plugin malicious code | Fase 1: apenas plugins assinados; Fase 2: WASM sandbox obrigatório para não-assinados |

---

## Implementação (Resumo)

### Cargo Workspace
```toml
# ferro-plugins/Cargo.toml
[features]
default = ["dylib"]
dylib = ["libloading"]
wasm = ["wasmtime", "wasi"]
```

### PluginManager
```rust
pub struct PluginManager {
    dylib_loader: Option<LibLoadingLoader>,
    wasm_runtime: Option<WasmtimeRuntime>,
    loaded: HashMap<PluginId, LoadedPlugin>,
}

impl PluginManager {
    pub fn load_dylib(&mut self, path: &Path) -> Result<PluginId> { ... }
    pub fn load_wasm(&mut self, path: &Path, caps: Capabilities) -> Result<PluginId> { ... }
    pub fn hot_reload(&mut self, id: &PluginId) -> Result<()> { ... }
}
```

### Capability System (WASM)
```rust
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct Capabilities {
    pub fs_read: Vec<PathBuf>,      // paths permitidos leitura
    pub fs_write: Vec<PathBuf>,     // paths permitidos escrita
    pub net_http: Vec<String>,      // domínios permitidos
    pub env_read: Vec<String>,      // vars permitidas
    pub fuel_limit: u64,            // instruções WASM max
    pub memory_limit: usize,        // bytes max
}
```

---

## Validação

- [ ] MVP: `cargo build --features dylib` carrega plugin `cdylib` e executa `Agent::handle()`
- [ ] Hot reload: modificar plugin `.so` → `ferro plugin reload <id>` → nova versão ativa
- [ ] v1.1: `cargo build --features wasm` carrega `.wasm` com capabilities declaradas
- [ ] Security: plugin WASM sem `fs_write` falha ao tentar escrever fora do permitido
- [ ] Binary size: `--features wasm` adiciona < 8MB (wasmtime)

---

## Referências
- `docs/02-benchmarks.md` Seção 4 (WASM Runtime comparativo)
- wasmtime docs: https://docs.wasmtime.dev/
- Component Model: https://github.com/WebAssembly/component-model
- libloading: https://docs.rs/libloading/