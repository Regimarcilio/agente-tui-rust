# ADR-008: Persistence — SQLite (sqlx) + FTS5 + JSONL Export

**Status:** Accepted  
**Data:** 28/10/2026  
**Autor:** PLANMASTER  
**Relacionado:** RF-014, RNF-005, RNF-006, RF-006 (trace.json)

---

## Contexto

O FERRO precisa persistir:
- **Sessões:** Múltiplas, paralelas, por agente/workspace
- **Mensagens:** Role (user/assistant/system/tool), conteúdo, tool calls, tokens, custo, latência
- **Traces:** LLM calls completos (model, provider, tokens, cost, latency, cache_hit, agent_id)
- **Busca:** Full-text search em histórico (FTS5)
- **Export/Import:** JSONL portável para backup/migração/análise
- **Confiabilidade:** Crash recovery, WAL mode, backup automático

Opções:
- **SQLite (sqlx)** — Embedded, ACID, WAL, FTS5, JSON1, zero config, single file
- **PostgreSQL** — Overkill, precisa server, não single-binary
- **Redis** — Não persistente o suficiente, sem FTS nativo
- **Custom file format** — Reinventar roda, sem query/índice
- **Sled/RocksDB** — Key-value, sem SQL/FTS, complexidade

---

## Decisão

**SQLite com `sqlx` (compile-time checked queries) + FTS5 para busca + JSONL export/import.**

### Schema (ferro-session)

```sql
-- ferro-session/migrations/001_init.sql

-- Sessões
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,                    -- UUID v7 (timestamp + random)
    created_at INTEGER NOT NULL,            -- Unix ms (Utc::now().timestamp_millis())
    updated_at INTEGER NOT NULL,
    agent_id TEXT NOT NULL,                 -- AgentId (planmaster, backdev, ...)
    workspace TEXT NOT NULL,                -- Path absoluto
    metadata TEXT DEFAULT '{}',             -- JSON: tags, project, etc.
    status TEXT DEFAULT 'active',           -- active, archived, deleted
    message_count INTEGER DEFAULT 0
);

-- Mensagens (tabela principal)
CREATE TABLE messages (
    id TEXT PRIMARY KEY,                    -- UUID v7
    session_id TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    created_at INTEGER NOT NULL,
    role TEXT NOT NULL,                     -- user, assistant, system, tool
    content TEXT NOT NULL,                  -- Conteúdo principal
    tool_calls TEXT DEFAULT '[]',           -- JSON array: [{name, args, id}]
    tool_results TEXT DEFAULT '[]',         -- JSON array: [{id, result, error}]
    tokens_in INTEGER DEFAULT 0,            -- Input tokens
    tokens_out INTEGER DEFAULT 0,           -- Output tokens
    cost_usd REAL DEFAULT 0.0,              -- Custo em USD
    latency_ms INTEGER DEFAULT 0,           -- Latência total ms
    cache_hit BOOLEAN DEFAULT 0,            -- Cache semântico hit
    model TEXT,                             -- Modelo usado (se assistant)
    provider TEXT,                          -- Provider (opencode, claude, ...)
    metadata TEXT DEFAULT '{}',             -- JSON extra
    
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

-- Índices para queries comuns
CREATE INDEX idx_messages_session_created ON messages(session_id, created_at);
CREATE INDEX idx_messages_role ON messages(role);
CREATE INDEX idx_messages_model ON messages(model);

-- FTS5 Virtual Table para busca full-text
CREATE VIRTUAL TABLE messages_fts USING fts5(
    content,                                -- Coluna indexada
    content_rowid='id',                     -- Rowid = messages.id
    tokenize='porter unicode61'             -- Stemming + Unicode
);

-- Triggers para manter FTS sincronizado
CREATE TRIGGER messages_ai AFTER INSERT ON messages BEGIN
    INSERT INTO messages_fts(rowid, content) VALUES (new.id, new.content);
END;

CREATE TRIGGER messages_ad AFTER DELETE ON messages BEGIN
    INSERT INTO messages_fts(messages_fts, rowid, content) VALUES ('delete', old.id, old.content);
END;

CREATE TRIGGER messages_au AFTER UPDATE ON messages BEGIN
    INSERT INTO messages_fts(messages_fts, rowid, content) VALUES ('delete', old.id, old.content);
    INSERT INTO messages_fts(rowid, content) VALUES (new.id, new.content);
END;

-- Traces detalhados (para FinOps/debug)
CREATE TABLE traces (
    id TEXT PRIMARY KEY,                    -- UUID v7
    message_id TEXT NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    created_at INTEGER NOT NULL,
    model TEXT NOT NULL,
    provider TEXT NOT NULL,
    prompt_tokens INTEGER NOT NULL,
    completion_tokens INTEGER NOT NULL,
    total_tokens INTEGER NOT NULL,
    cost_usd REAL NOT NULL,
    latency_ms INTEGER NOT NULL,
    cache_hit BOOLEAN NOT NULL,
    agent_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    request_json TEXT NOT NULL,             -- Request completo (redacted)
    response_json TEXT NOT NULL,            -- Response completo (redacted)
    error TEXT                              -- Se erro
);

CREATE INDEX idx_traces_session ON traces(session_id);
CREATE INDEX idx_traces_agent ON traces(agent_id);
CREATE INDEX idx_traces_model ON traces(model);

-- Configuração de backup
CREATE TABLE meta (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

INSERT INTO meta(key, value) VALUES ('schema_version', '1');
INSERT INTO meta(key, value) VALUES ('last_backup', '0');
```

### SessionStore (sqlx + compile-time queries)

```rust
// ferro-session/src/store.rs
use sqlx::{SqlitePool, Row, query, query_as};
use sqlx::sqlite::{SqliteConnectOptions, SqliteJournalMode, SqliteSynchronous};
use std::path::Path;

pub struct SessionStore {
    pool: SqlitePool,
}

impl SessionStore {
    pub async fn new(path: &Path) -> Result<Self> {
        let options = SqliteConnectOptions::new()
            .filename(path)
            .create_if_missing(true)
            .journal_mode(SqliteJournalMode::Wal)  // WAL mode para concorrência
            .synchronous(SqliteSynchronous::Normal) // Balance safety/performance
            .busy_timeout(std::time::Duration::from_secs(30))
            .pragma("mmap_size", "268435456")      // 256MB mmap
            .pragma("page_size", "4096")
            .pragma("cache_size", "-32768");       // 32MB cache
        
        let pool = SqlitePool::connect_with(options).await?;
        
        // Run migrations
        sqlx::migrate!("./migrations").run(&pool).await?;
        
        Ok(Self { pool })
    }
    
    pub async fn create_session(&self, agent_id: &str, workspace: &Path) -> Result<Session> {
        let id = Uuid::now_v7().to_string();
        let now = Utc::now().timestamp_millis();
        
        query!(
            r#"INSERT INTO sessions (id, created_at, updated_at, agent_id, workspace) VALUES (?, ?, ?, ?, ?)"#,
            id, now, now, agent_id, workspace.to_string_lossy()
        ).execute(&self.pool).await?;
        
        Ok(Session { id, created_at: now, updated_at: now, agent_id: agent_id.into(), workspace: workspace.into(), ..Default::default() })
    }
    
    pub async fn save_message(&self, session_id: &str, msg: Message) -> Result<()> {
        let now = Utc::now().timestamp_millis();
        
        // Update session updated_at + message_count
        query!(
            r#"UPDATE sessions SET updated_at = ?, message_count = message_count + 1 WHERE id = ?"#,
            now, session_id
        ).execute(&self.pool).await?;
        
        // Insert message
        query!(
            r#"INSERT INTO messages (id, session_id, created_at, role, content, tool_calls, tool_results, tokens_in, tokens_out, cost_usd, latency_ms, cache_hit, model, provider, metadata) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)"#,
            msg.id, session_id, now, msg.role, msg.content,
            serde_json::to_string(&msg.tool_calls)?,
            serde_json::to_string(&msg.tool_results)?,
            msg.tokens_in, msg.tokens_out, msg.cost_usd, msg.latency_ms, msg.cache_hit,
            msg.model, msg.provider, serde_json::to_string(&msg.metadata)?
        ).execute(&self.pool).await?;
        
        Ok(())
    }
    
    pub async fn search_fts(&self, session_id: Option<&str>, query: &str, limit: usize) -> Result<Vec<Message>> {
        let sql = if let Some(sid) = session_id {
            r#"SELECT m.* FROM messages m JOIN messages_fts f ON m.id = f.rowid WHERE m.session_id = ? AND messages_fts MATCH ? ORDER BY rank LIMIT ?"#
        } else {
            r#"SELECT m.* FROM messages m JOIN messages_fts f ON m.id = f.rowid WHERE messages_fts MATCH ? ORDER BY rank LIMIT ?"#
        };
        
        // sqlx não suporta query_as dinâmico fácil, usar query + map
        // ... implementação completa
    }
    
    pub async fn export_jsonl(&self, session_id: &str, path: &Path) -> Result<()> {
        let mut file = tokio::fs::File::create(path).await?;
        
        let rows = query!(
            r#"SELECT * FROM messages WHERE session_id = ? ORDER BY created_at"#,
            session_id
        ).fetch_all(&self.pool).await?;
        
        for row in rows {
            let msg = Message {
                id: row.id,
                session_id: row.session_id,
                created_at: row.created_at,
                role: row.role,
                content: row.content,
                tool_calls: serde_json::from_str(&row.tool_calls)?,
                tool_results: serde_json::from_str(&row.tool_results)?,
                tokens_in: row.tokens_in,
                tokens_out: row.tokens_out,
                cost_usd: row.cost_usd,
                latency_ms: row.latency_ms,
                cache_hit: row.cache_hit,
                model: row.model,
                provider: row.provider,
                metadata: serde_json::from_str(&row.metadata)?,
            };
            
            let line = serde_json::to_string(&msg)?;
            file.write_all(line.as_bytes()).await?;
            file.write_all(b"\n").await?;
        }
        
        file.flush().await?;
        Ok(())
    }
    
    pub async fn import_jsonl(&self, path: &Path) -> Result<Session> {
        let mut file = tokio::fs::File::open(path).await?;
        let mut contents = String::new();
        file.read_to_string(&mut contents).await?;
        
        let session_id = Uuid::now_v7().to_string();
        let now = Utc::now().timestamp_millis();
        
        // Criar sessão
        query!(
            r#"INSERT INTO sessions (id, created_at, updated_at, agent_id, workspace) VALUES (?, ?, ?, ?, ?)"#,
            session_id, now, now, "imported", "imported"
        ).execute(&self.pool).await?;
        
        // Importar mensagens
        for line in contents.lines() {
            let msg: Message = serde_json::from_str(line)?;
            self.save_message(&session_id, msg).await?;
        }
        
        Ok(Session { id: session_id, ..Default::default() })
    }
}
```

### Backup Automático

```rust
// ferro-session/src/backup.rs
impl SessionStore {
    pub async fn backup(&self, backup_dir: &Path) -> Result<PathBuf> {
        let timestamp = Utc::now().format("%Y%m%d-%H%M%S");
        let backup_path = backup_dir.join(format!("sessions-{}.db", timestamp));
        
        // SQLite online backup API (não bloqueia writers)
        let backup = self.pool.backup(&backup_path).await?;
        backup.run_to_completion().await?;
        
        // Atualizar meta
        query!("UPDATE meta SET value = ? WHERE key = 'last_backup'", timestamp.to_string())
            .execute(&self.pool).await?;
        
        // Manter apenas últimos 10 backups
        self.cleanup_old_backups(backup_dir, 10).await?;
        
        Ok(backup_path)
    }
    
    pub async fn start_backup_scheduler(&self, interval: Duration, backup_dir: PathBuf) {
        tokio::spawn(async move {
            let mut interval = tokio::time::interval(interval);
            loop {
                interval.tick().await;
                if let Err(e) = self.backup(&backup_dir).await {
                    tracing::error!("Backup failed: {}", e);
                }
            }
        });
    }
}
```

---

## Consequências

### Positivas
- **Zero config:** Single file `~/.config/ferro/sessions.db`
- **ACID + WAL:** Concorrência leitores/escritores, crash recovery automático
- **FTS5 nativo:** Busca full-text rápida (porter stemming + unicode)
- **Compile-time safety:** `sqlx` verifica queries no build (`cargo sqlx prepare`)
- **JSONL portável:** Backup, migração, análise externa (jq, Python, etc.)
- **Embedded:** Sem server, single binary, cross-platform

### Negativas
- **Write throughput:** SQLite serializa writes (~10k ops/s) — OK para chat
- **Schema migration:** Cuidado com breaking changes (sqlx migrate gerencia)
- **File locking:** WAL ajuda, mas NFS/shared storage problemático

### Riscos Mitigados
| Risco | Mitigação |
|-------|-----------|
| Corrupção DB | WAL mode + backup hourly + `PRAGMA integrity_check` no startup |
| FTS5 sync drift | Triggers automáticos + `INSERT INTO messages_fts(messages_fts) VALUES('rebuild')` periódico |
| Backup falha silencioso | Verificar `last_backup` meta no startup; alertar se > 25h |
| Query performance | Índices em session_id, created_at, role, model; `EXPLAIN QUERY PLAN` em CI |

---

## Validação

- [ ] `cargo sqlx prepare` passa (queries compiladas verificadas)
- [ ] `SessionStore::new()` cria schema + FTS + triggers
- [ ] CRUD sessões + mensagens funciona com concorrência (múltiplas tasks)
- [ ] FTS search: "planmaster" retorna mensagens relevantes em < 50ms
- [ ] Export JSONL: arquivo válido, import restaura sessão idêntica
- [ ] Backup: arquivo `.db` copiado, integridade verificada, restore funciona
- [ ] WAL mode: leitores não bloqueiam escritores, crash recovery OK
- [ ] Retenção: `DELETE FROM sessions WHERE created_at < ?` + vacuum periódico

---

## Referências
- sqlx docs: https://github.com/launchbadge/sqlx
- SQLite FTS5: https://www.sqlite.org/fts5.html
- SQLite WAL: https://www.sqlite.org/wal.html
- UUID v7: https://datatracker.ietf.org/doc/draft-peabody-dispatch-new-uuid-format/