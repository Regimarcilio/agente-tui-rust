# ADR-005: Config Management — Dotfiles Git + Figment Layering + Hot Reload

**Status:** Accepted  
**Data:** 28/10/2026  
**Autor:** PLANMASTER  
**Relacionado:** RF-002, RF-006, RF-020, RN-002, `docs/02-benchmarks.md` Seção 1.3

---

## Contexto

Configuração do FERRO deve ser:
- **Single Source of Truth:** Dotfiles Git repo (versionado, compartilhado multi-PC)
- **Layered:** Defaults → File (dotfiles) → Env vars → CLI args
- **Hot Reload:** Mudanças aplicadas sem restart TUI
- **Validada:** Schema TOML com serde, erros claros
- **Extensível:** Plugins/agentes adicionam suas próprias seções

Opções (ver benchmarks):
- **figment** — Layered config (TOML, JSON, YAML, Env, CLI), merge strategy, validação serde
- **config-rs** — Similar, mas menos flexível em layering
- **confy** — Simples, sem layering avançado
- **Custom** — Reimplementar roda

---

## Decisão

**Usar `figment` como config engine com layering explícito:**

### Layering Order (Priority: High to Low)
```rust
// ferro-core/src/config.rs
use figment::{Figment, providers::{Toml, Env, Format}};

pub fn load_config() -> Result<Config> {
    let figment = Figment::new()
        // 1. Defaults (embedded no binary)
        .merge(ferro_core::defaults())
        // 2. Dotfiles repo: ~/.config/ferro/profile.toml (source of truth)
        .merge(Toml::file(dotfiles_path().join("profile.toml")).required(false))
        // 3. Theme file: ~/.config/ferro/themes/{theme}.toml
        .merge(Toml::file(theme_path()).required(false))
        // 4. Keybindings: ~/.config/ferro/keybindings.toml
        .merge(Toml::file(keybindings_path()).required(false))
        // 5. MCP servers: ~/.config/ferro/mcp.toml
        .merge(Toml::file(mcp_path()).required(false))
        // 6. Environment variables: FERRO_*
        .merge(Env::prefixed("FERRO_").global())
        // 7. CLI args (aplicados no main, override final)
        ;
    
    figment.extract::<Config>()
}
```

### Schema TOML (profile.toml)

```toml
# ~/.config/ferro/profile.toml (source of truth no dotfiles repo)
[user]
name = "Arquiteto Rustacean"
email = "dev@empresa.com"
host_id = "pc-casa"           # Override para anti-colisão

[ui]
theme = "dracula"             # base16 theme name
language = "pt-BR"
animations = true
reduced_motion = false
mouse_enabled = true

[keybindings]
leader = "Space"
# Vim defaults + custom overrides
custom = { "Ctrl+P" = "command_palette", "Ctrl+E" = "file_explorer" }

[agents]
# Quais agentes compilar/carregar (Cargo features)
enabled = ["planmaster", "backdev", "frontdev", "engai", "kalimaster", "opsmaster", "qamaster"]
auto_select = true           # LLM classifica intent
default_agent = "planmaster"

[network]
llm_timeout_secs = 30
llm_retry_count = 2
fallback_chain = ["opencode", "claude", "gh-copilot", "ollama"]
ollama_host = "http://localhost:11434"

[git.sync]
auto_pull_idle_hours = 2
anti_collision_enabled = true
rebase_instead_of_merge = true
stash_before_pull = true
preferred_remote = "origin"
log_file = ".ferro-sync.log"

[mcp.servers]
git = { command = "mcp-server-git", args = [] }
fs = { command = "mcp-server-fs", args = ["/workspace"] }
sqlite = { command = "mcp-server-sqlite", args = ["~/.config/ferro/sessions.db"] }

[plugins]
directory = "~/.config/ferro/plugins"
wasm_enabled = false         # v1.1+
dylib_enabled = true
signature_required = true    # Apenas plugins assinados (dylib)

[security]
secret_redaction = true
redaction_patterns = [
    "sk-[a-zA-Z0-9]{32,}",           # OpenAI
    "ghp_[a-zA-Z0-9]{36}",           # GitHub PAT
    "gho_[a-zA-Z0-9]{36}",           # GitHub OAuth
    "xoxb-[0-9]{12}-[a-zA-Z0-9]{24}", # Slack Bot
    "api_key\\s*=\\s*[\"'][^\"']+[\"']" # Generic
]
audit_on_save = true

[session]
auto_save = true
export_on_exit = false
retention_days = 90
max_sessions = 1000
```

### Hot Reload Implementation

```rust
// ferro-core/src/config.rs
use notify::{RecommendedWatcher, Watcher, RecursiveMode, EventKind};
use std::sync::Arc;
use tokio::sync::broadcast;

pub struct ConfigReloader {
    watcher: RecommendedWatcher,
    tx: broadcast::Sender<ConfigChange>,
    config: Arc<RwLock<Config>>,
}

impl ConfigReloader {
    pub fn new(config: Arc<RwLock<Config>>) -> Result<Self> {
        let (tx, _rx) = broadcast::channel(100);
        let tx_clone = tx.clone();
        
        let watcher = RecommendedWatcher::new(
            move |res: Result<Event, Error>| {
                if let Ok(event) = res {
                    if matches!(event.kind, EventKind::Modify(_) | EventKind::Create(_)) {
                        let _ = tx_clone.send(ConfigChange::FileChanged(event.paths));
                    }
                }
            },
            notify::Config::default()
        )?;
        
        // Watch dotfiles directory
        watcher.watch(&dotfiles_path(), RecursiveMode::Recursive)?;
        
        Ok(Self { watcher, tx, config })
    }
    
    pub async fn run(&mut self) -> Result<()> {
        let mut rx = self.tx.subscribe();
        while let Ok(change) = rx.recv().await {
            // Debounce 50ms
            tokio::time::sleep(Duration::from_millis(50)).await;
            
            // Reload config
            match load_config() {
                Ok(new_config) => {
                    let mut guard = self.config.write().await;
                    *guard = new_config;
                    // Notify subscribers (TUI, agents, etc.)
                    let _ = self.tx.send(ConfigChange::Reloaded);
                }
                Err(e) => {
                    tracing::error!("Config reload failed: {}", e);
                    let _ = self.tx.send(ConfigChange::Error(e));
                }
            }
        }
        Ok(())
    }
}
```

### Validação Schema (serde + custom)

```rust
#[derive(Debug, Deserialize, Serialize, Clone)]
#[serde(deny_unknown_fields)] // Erro em campos desconhecidos
pub struct Config {
    #[serde(default)]
    pub user: UserConfig,
    
    #[serde(default)]
    pub ui: UiConfig,
    
    #[serde(default)]
    pub agents: AgentsConfig,
    
    #[serde(default)]
    pub network: NetworkConfig,
    
    #[serde(default)]
    pub git: GitConfig,
    
    #[serde(default)]
    pub mcp: McpConfig,
    
    #[serde(default)]
    pub plugins: PluginsConfig,
    
    #[serde(default)]
    pub security: SecurityConfig,
    
    #[serde(default)]
    pub session: SessionConfig,
}

impl Config {
    pub fn validate(&self) -> Result<()> {
        // Validar theme existe
        if !THEMES.contains(&self.ui.theme.as_str()) {
            return Err(ConfigError::InvalidTheme(self.ui.theme.clone()));
        }
        // Validar agents enabled são features válidas
        for agent in &self.agents.enabled {
            if !VALID_AGENTS.contains(&agent.as_str()) {
                return Err(ConfigError::InvalidAgent(agent.clone()));
            }
        }
        // Validar fallback chain
        for provider in &self.network.fallback_chain {
            if !VALID_PROVIDERS.contains(&provider.as_str()) {
                return Err(ConfigError::InvalidProvider(provider.clone()));
            }
        }
        Ok(())
    }
}
```

---

## Consequências

### Positivas
- **Single source of truth:** Dotfiles Git = config versionada, compartilhada, auditável
- **Layering explícito:** Prioridade clara, override previsível (Env > File > Defaults)
- **Hot reload real:** `notify` + `figment` reload + broadcast → TUI/agents reagem
- **Validação forte:** `deny_unknown_fields` + validação custom = erros claros no startup
- **Extensível:** Plugins adicionam seções `[plugins.meu-plugin]` no profile.toml

### Negativas
- **Complexidade:** Figment + notify + broadcast + RwLock = moving parts
- **Race conditions:** Reload concorrente com uso → RwLock resolve, mas latência
- **Schema evolution:** Campos novos = breaking para configs antigas (mitigado: defaults)

### Riscos Mitigados
| Risco | Mitigação |
|-------|-----------|
| Config inválida trava TUI | Validação no load; se erro → log + manter config anterior + notificar TUI |
| Hot reload loop (config muda → salva → reload) | Debounce 50ms + ignorar escritas próprias (pid check) |
| Env vars sensíveis no log | `tracing` filtra `FERRO_*` secrets; redaction patterns |
| Dotfiles repo corrompido | `ferro --sync-profile status` detecta; `pull` recupera |

---

## Implementação (Resumo)

### Integração no FerroApp
```rust
// ferro-core/src/app.rs
pub struct FerroApp {
    config: Arc<RwLock<Config>>,
    reloader: ConfigReloader,
    // ... outros campos
}

impl FerroApp {
    pub async fn new() -> Result<Self> {
        let config = Arc::new(RwLock::new(load_config()?));
        config.read().await.validate()?;
        
        let mut reloader = ConfigReloader::new(config.clone())?;
        tokio::spawn(async move { reloader.run().await });
        
        Ok(Self { config, reloader, ... })
    }
}
```

### TUI Reage a Mudanças
```rust
// ferro-tui/src/runtime.rs
impl TuiRuntime {
    pub async fn handle_config_change(&mut self, change: ConfigChange) -> Result<()> {
        match change {
            ConfigChange::Reloaded => {
                let config = self.app.config.read().await;
                self.apply_theme(&config.ui.theme)?;
                self.apply_keybindings(&config.keybindings)?;
                self.widgets.reload_config(&config)?;
            }
            ConfigChange::Error(e) => {
                self.show_notification(&format!("Config error: {}", e), Level::Error)?;
            }
            _ => {}
        }
        Ok(())
    }
}
```

---

## Validação

- [ ] `ferro` carrega config do dotfiles repo + env vars + CLI args (prioridade correta)
- [ ] Editar `profile.toml` → TUI aplica theme/keybindings em < 100ms sem restart
- [ ] Campo inválido no TOML → erro claro no startup com linha/coluna
- [ ] `FERRO_UI_THEME=gruvbox` override funciona
- [ ] `ferro --theme=dracula` CLI override funciona (priority > env > file)
- [ ] Plugin adiciona `[plugins.custom]` → carregado e validado
- [ ] Secret redaction patterns funcionam em logs/traces

---

## Referências
- figment docs: https://docs.rs/figment/
- notify docs: https://docs.rs/notify/
- `docs/02-benchmarks.md` Seção 1.3 (Config crates comparativo)
- 12-factor app config: https://12factor.net/config