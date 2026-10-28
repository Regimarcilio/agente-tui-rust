# ADR-004: Multi-PC Sync — Auto Pull Idle + Anti-Colisão Branch Derivação

**Status:** Accepted  
**Data:** 28/10/2026  
**Autor:** PLANMASTER  
**Relacionado:** RF-006, RF-020, RN-007, RN-008, `docs/AGENT_OPERATING_RULES.md` Seção 10

---

## Contexto

O FERRO é compartilhado entre múltiplas máquinas (laptop casa, laptop trabalho, desktop, servidor). Problemas:
- **Config drift:** Mudanças em uma máquina não propagam para outras
- **Branch collision:** Duas máquinas cometem na mesma branch (`develop`) antes de pull
- **Idle stale:** Máquina parada por horas tem base desatualizada
- **Force push perigoso:** Perde histórico, quebra CI, causa conflitos

Requisitos (J4 - Sincronização Multi-Máquina):
- `ferro --sync-profile pull` em máquina B puxa mudanças de máquina A
- Hot reload config automático após pull
- Detecção automática de colisão + derivação de branch segura
- Log auditável de todas sincronizações

---

## Decisão

**Duas mecanismos automáticos no `GitSync` service:**

### 1. Auto Pull em Idle > 2h (RN-008)

```rust
// ferro-git/src/sync.rs
impl GitSync {
    pub async fn auto_pull_if_idle(&self) -> Result<()> {
        let idle_seconds = self.get_idle_seconds().await?;
        if idle_seconds < self.config.auto_pull_idle_hours * 3600 {
            return Ok(()); // Não idle o suficiente
        }
        
        let branch = self.git.current_branch()?;
        let behind = self.git.count_behind(&branch, "origin")?;
        
        if behind > 0 {
            // Stash se houver mudanças locais não commitadas
            let had_changes = self.git.has_uncommitted_changes()?;
            if had_changes && self.config.stash_before_pull {
                self.git.stash_push(&format!("auto-stash pre-pull {}", Utc::now()))?;
            }
            
            // Pull com rebase (configurável)
            self.git.pull_rebase("origin", &branch)?;
            
            // Restore stash
            if had_changes && self.config.stash_before_pull {
                self.git.stash_pop()?;
            }
            
            // Hot reload config
            self.config_reloader.trigger_reload()?;
        }
        
        self.log_sync(SyncLogEntry {
            timestamp: Utc::now(),
            branch,
            idle_seconds,
            behind,
            ahead: 0,
            action: SyncAction::AutoPullRebase,
            result: SyncResult::Success,
        }).await?;
        
        Ok(())
    }
}
```

### 2. Anti-Colisão Branch Derivação (RN-007)

```rust
impl GitSync {
    pub async fn check_and_handle_collision(&self) -> Result<CollisionResult> {
        let branch = self.git.current_branch()?;
        let (behind, ahead) = self.git.count_divergence(&branch, "origin")?;
        
        match (behind, ahead) {
            (0, 0) => Ok(CollisionResult::Clean),           // Sincronizado
            (b, 0) if b > 0 => {                            // Só remoto tem novidades
                self.git.pull_rebase("origin", &branch)?;
                Ok(CollisionResult::Pulled { behind: b })
            }
            (b, a) if b > 0 && a > 0 => {                   // COLISÃO: ambos têm commits
                let derived = self.derive_branch(&branch)?;
                self.git.checkout(&derived)?;
                self.log_sync(SyncLogEntry {
                    timestamp: Utc::now(),
                    branch: branch.clone(),
                    idle_seconds: 0,
                    behind: b,
                    ahead: a,
                    action: SyncAction::DerivedBranch { original: branch, derived: derived.clone() },
                    result: SyncResult::Success,
                }).await?;
                Ok(CollisionResult::DerivedBranch { original: branch, derived })
            }
            (0, a) if a > 0 => Ok(CollisionResult::AheadOnly { ahead: a }), // Só local tem commits
            _ => Ok(CollisionResult::Clean),
        }
    }
    
    fn derive_branch(&self, original: &str) -> Result<String> {
        let hostname = self.config.host_id.clone()
            .unwrap_or_else(|| hostname::get()?.to_string_lossy().into_owned());
        let timestamp = Utc::now().format("%Y%m%d-%H%M%S");
        let prefix = self.branch_type_prefix(original);
        Ok(format!("{}-{}-{}", prefix, hostname, timestamp))
    }
    
    fn branch_type_prefix(&self, branch: &str) -> &str {
        if branch == "develop" { "DEV-sync" }
        else if branch.starts_with("ISSUE-") { "ISSUE-sync" }
        else if branch.starts_with("DEV-") { "DEV-sync" }
        else if branch.starts_with("SEG-") { "SEG-sync" }
        else if branch.starts_with("DOC-") { "DOC-sync" }
        else { "SYNC" }
    }
}
```

### Configuração (profile.toml)

```toml
[git.sync]
auto_pull_idle_hours = 2           # Padrão: 2h
anti_collision_enabled = true      # Padrão: true
host_id = "pc-casa"                # Override hostname (opcional)
preferred_remote = "origin"        # Remote padrão
rebase_instead_of_merge = true     # Sempre rebase
stash_before_pull = true           # Stash automático se divergência
log_file = ".ferro-sync.log"       # Log local de sincronizações
```

---

## Consequências

### Positivas
- **Zero config manual:** Funciona out-of-the-box após primeiro `ferro --sync-profile pull`
- **Seguro:** Nunca force push, nunca merge automático sem revisão
- **Auditável:** `.ferro-sync.log` registra todas ações com timestamp
- **Não bloqueia:** Derivação de branch permite continuar trabalhando imediatamente
- **Recuperável:** Branch original intacta; merge manual posterior via PR

### Negativas
- **Branch proliferation:** Muitas branches derivadas se colisões frequentes
- **Merge manual necessário:** Branch derivada precisa PR para integrar
- **Idle detection imperfeito:** Baseado em último commit, não última atividade real

### Riscos Mitigados
| Risco | Mitigação |
|-------|-----------|
| Stash conflitos no pop | `git stash pop` pode conflitar → logar erro + abrir issue manual (question tool) |
| Rebase falha (conflitos complexos) | Fallback: `git merge` + commit de merge; log para revisão manual |
| Hostname collision (mesmo hostname) | `host_id` no profile.toml permite override único |
| Log file growth | Rotate log > 10MB; manter últimos 1000 entries |

---

## Implementação (Resumo)

### Integração no Agent Loop
```rust
// ferro-core/src/session.rs
impl SessionManager {
    pub async fn start_session(&mut self, agent_id: AgentId) -> Result<Session> {
        // 1. Auto-sync antes de qualquer operação
        self.git_sync.auto_pull_if_idle().await?;
        
        // 2. Verificar colisão
        let collision = self.git_sync.check_and_handle_collision().await?;
        if let CollisionResult::DerivedBranch { derived, .. } = collision {
            // Notificar TUI
            self.tui.notify(&format!("⚠️ Colisão detectada. Branch: {}", derived))?;
        }
        
        // 3. Criar sessão
        self.session_store.create_session(agent_id).await
    }
}
```

### Log Format (`.ferro-sync.log`)
```json
{"timestamp":"2026-10-28T14:30:00Z","branch":"develop","idle_seconds":10800,"behind":3,"ahead":0,"action":"AutoPullRebase","result":"Success"}
{"timestamp":"2026-10-28T14:35:00Z","branch":"develop","idle_seconds":0,"behind":1,"ahead":1,"action":"DerivedBranch","original":"develop","derived":"DEV-sync-pc-b-20261028-143500","result":"Success"}
```

---

## Validação

- [ ] Máquina A push develop → Máquina B idle 3h → primeira task faz auto-pull rebase
- [ ] Máquina A e B cometem em develop → B detecta colisão → deriva `DEV-sync-pc-b-...`
- [ ] Stash automático: mudanças não commitadas preservadas após pull
- [ ] Hot reload: config alterada no dotfiles repo aplica após pull sem restart
- [ ] `.ferro-sync.log` registra todas ações com timestamps corretos
- [ ] Config `auto_pull_idle_hours = 0` desabilita auto-pull
- [ ] Config `anti_collision_enabled = false` desabilita derivação (apenas log warning)

---

## Referências
- `docs/AGENT_OPERATING_RULES.md` Seção 10 (regras completas)
- `docs/01-visao-e-escopo.md` Jornada J4 (Sincronização Multi-Máquina)
- git2-rs docs: https://docs.rs/git2/
- Conventional Commits: https://www.conventionalcommits.org/