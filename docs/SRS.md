# SRS — Agente TUI Rust Personalizado (Codename: FERRO)

**Versão:** 1.0  
**Data:** 28/10/2026  
**Autor:** PLANMASTER  
**Branch:** develop  
**Status:** Aprovado para Implementação

---

## 1. Introdução

### 1.1 Propósito
Este documento especifica os requisitos funcionais e não-funcionais do **FERRO**, um agente TUI nativo em Rust que embute 19 agentes especializados do squad (planmaster, backdev, frontdev, engai, kalimaster, etc.) como módulos internos, eliminando latência de subprocessos e provendo UX unificada nível "Obra" (diff inline, LSP, syntax highlight, git UI, command palette).

### 1.2 Escopo
**Produto:** FERRO (Ferro = Iron/Fe, base do aço/Rust)  
**Contexto:** Substitui orquestração via OpenCode/CLI + múltiplos terminais por single binary Rust com agentes nativos, perfil via dotfiles Git, gateway LLM via CLI wrappers (opencode/claude/gh copilot/ollama), sandbox WASM para plugins terceiros.  
**Limites:** 
- IN: Core TUI, 19 agentes nativos, dotfiles sync, LLM gateway, WASM/dylib plugins, MCP client, SQLite persistence, security audit integrado
- OUT: GUI complementar, fine-tuning local, cloud sync sessões, marketplace agentes, multiplayer CRDT (pós-v1.0)

### 1.3 Definições, Acrônimos, Abreviações

| Termo | Definição |
|-------|-----------|
| **FERRO** | Codename do agente TUI |
| **Agent Trait** | Interface Rust unificada: `fn handle(&self, ctx: AgentCtx, input: Input) -> Stream<Output>` |
| **Dotfiles Repo** | Git repo com `profile.toml`, `agents/`, `themes/`, `keybindings/`, `mcp/` |
| **Semantic Cache** | SQLite + embeddings (local) para deduplicar chamadas LLM idênticas |
| **MCP Client** | Implementação nativa do Model Context Protocol (JSON-RPC 2.0 over stdio/TCP) |
| **Kalimaster-SAST** | Skill de auditoria estática integrada no build/CI |
| **Command Palette** | Fuzzy search (skim/fzf-like) para ações, agentes, arquivos, config |
| **TUI Nível Obra** | ratatui + tachyonfx + opaline + syntect + tower-lsp + git2 + diffrs |

### 1.4 Referências
- `docs/01-visao-e-escopo.md` — Visão, OKRs, jornadas, riscos
- `docs/02-benchmarks.md` — Benchmarks TUI, agent frameworks, CLI wrappers, WASM, LSP, diff
- `docs/AGENT_OPERATING_RULES.md` — Regras de operação contínua, sync multi-PC, anti-colisão
- EngAI Global Instructions (`~/.config/opencode/instructions/engai-global.md`)
- ratatui Architecture: https://github.com/ratatui/ratatui/blob/main/ARCHITECTURE.md
- awesome-ratatui: https://github.com/ratatui/awesome-ratatui

### 1.5 Visão Geral
- **Seção 2:** Descrição geral (perspectiva, usuários, restrições, premissas)
- **Seção 3:** Requisitos Funcionais (RF-001 a RF-XXX) com prioridade MoSCoW + AC Gherkin
- **Seção 4:** Requisitos Não-Funcionais (RNF-001 a RNF-XXX) com métricas quantificadas
- **Seção 5:** Requisitos de Interface (UI, APIs, MCP, Comunicação)
- **Seção 6:** Regras de Negócio (RN-001 a RN-XXX)
- **Seção 7:** Matriz de Rastreabilidade RF × RNF × RN × Testes
- **Seção 8:** Riscos e Mitigações
- **Seção 9:** Glossário
- **Seção 10:** Aprovações

---

## 2. Descrição Geral

### 2.1 Perspectiva do Produto
Sistema **independente, single-binary, local-first**, distribuível via `cargo install --locked ferro`. Integra-se com ecossistema existente (OpenCode, Claude Code, GitHub Copilot, Ollama) apenas como **CLI wrappers** para chamadas LLM — todo loop de agente, estado, tool calling, sandbox, persistência roda nativamente em Rust.

### 2.2 Funções do Produto (Resumo)
1. **Core TUI** — Event loop async, layout splits/tab, vim keys, command palette, themes, mouse
2. **19 Agentes Nativos** — Módulos Rust com trait `Agent`, registry dinâmico, Cargo features
3. **Perfil/Dotfiles Sync** — Git clone/pull/push, hot-reload config (notify), schema TOML validado
4. **Widgets Ricos** — Syntax highlight (tree-sitter), diff inline (Myers), LSP client, git UI, file explorer
5. **LLM Gateway** — Subprocess manager (opencode/claude/gh copilot/ollama), streaming JSONL, cache semântico, cost tracking
6. **Session/History** — SQLite persistência (sessões, mensagens, tool calls, traces), export/import JSONL, full-text search
7. **MCP Integration** — Cliente JSON-RPC 2.0 nativo, discovery servers, tool calling unificado
8. **Segurança** — Sandbox WASM (wasmtime), capability-based permissions, secret redaction, audit log

### 2.3 Características dos Usuários

| Perfil | Primário/Secundário | Necessidades |
|--------|---------------------|--------------|
| **Arquiteto Rustacean (Você)** | Primário | Controle total, privacidade, extensibilidade, velocidade, integração nativa 19 agentes |
| **Colegas de Squad** | Secundário | Onboarding `git clone dotfiles && ferro`, mesmo setup em segundos |
| **Contribuidores OSS** | Terciário | Arquitetura plugin WASM/dylib para adicionar agentes sem recompilar core |

### 2.4 Restrições

| Tipo | Detalhe |
|------|---------|
| **Técnica** | Rust 2024 edition (MSRV 1.80+), single binary, zero-config default, cross-platform (Linux/macOS/Windows) |
| **Performance** | Cold start < 2s, RAM idle < 50MB, 60fps TUI, LLM streaming overhead < 100ms |
| **Legal** | LGPD/GDPR by design (dados locais, opt-in telemetry), licenças deps MIT/Apache-2.0 |
| **Acessibilidade** | Screen reader friendly (ratatui+crossterm), high contrast themes, reduced motion, keyboard-only |
| **Orçamento** | Zero custo infra (local-first), deps open-source apenas |
| **Prazo** | MVP em 6 sprints (2 semanas cada) = ~12 semanas |

### 2.5 Premissas e Dependências
- **Premissa:** ratatui 0.30+ mantém API estável; tokio 1.40+ como runtime padrão
- **Dependência Externa:** `opencode`/`claude`/`gh copilot`/`ollama` instalados no PATH para LLM gateway
- **Dependência Sistema:** libgit2 (vendored via `git2` crate), OpenSSL (para git2 ssh), tree-sitter grammars
- **Premissa:** WASM Component Model estabiliza até v1.1; MVP usa dylib nativo (`cdylib` + `libloading`)

---

## 3. Requisitos Funcionais (RF)

### RF-001: Core TUI - Event Loop e Layout Base
**Prioridade:** Must  
**Ator:** Usuário Primário  
**Descrição:** Inicializar TUI com ratatui + crossterm, event loop async (tokio), layout com splits redimensionáveis (horizontal/vertical), tabs navegáveis (Tab/Shift-Tab), vim keybindings (h/j/k/l, gg, G, /, ?), command palette (Ctrl+P) com fuzzy search (skim), themes base16 via opaline, suporte a mouse.  
**Dependências:** RNF-001 (performance), RNF-007 (acessibilidade)  
**Critério de Aceite (Gherkin):**
```gherkin
Given o usuário executa `ferro` pela primeira vez
When o TUI carrega
Then a interface abre em < 2s (cold start)
And mostra dashboard com 3 painéis: agents, sessions, files
And splits são redimensionáveis com Ctrl+h/j/k/l
And command palette abre com Ctrl+P e busca fuzzy
And themes base16 são alternáveis via command palette
And navegação vim funciona em todos os painéis
```

### RF-002: Core TUI - Hot Reload de Configuração
**Prioridade:** Must  
**Ator:** Usuário Primário  
**Descrição:** Monitorar alterações em `~/.config/ferro/` (profile.toml, themes/, keybindings/, agents/) via `notify` e recarregar configuração sem reiniciar TUI (figment layering: defaults → file → env).  
**Dependências:** RF-001, RF-015 (dotfiles sync)  
**Critério de Aceite (Gherkin):**
```gherkin
Given o TUI está rodando
When o usuário edita ~/.config/ferro/profile.toml e salva
Then a configuração é recarregada em < 100ms
And mudanças de theme/keybindings aplicam imediatamente
And nenhum restart é necessário
And log mostra "config reloaded" com timestamp
```

### RF-003: Agentes Nativos - Registry e Trait Unificado
**Prioridade:** Must  
**Ator:** Sistema / Usuário  
**Descrição:** Implementar trait `Agent` unificado e registry dinâmico que descobre agentes via Cargo features (`--features planmaster,backdev,engai,...`) e/ou dynamic loading (dylib/WASM). Cada agente expõe: `id()`, `name()`, `description()`, `capabilities()`, `handle()`, `configure()`.  
**Dependências:** RF-004 (agentes específicos), RNF-002 (memory), RNF-006 (manutenibilidade)  
**Critério de Aceite (Gherkin):**
```gherkin
Given o binary é compilado com --features planmaster,backdev
When o TUI inicia
Then o registry lista 2 agentes disponíveis
And cada agente implementa Agent trait completo
And `ferro --agent planmaster` carrega apenas planmaster
And switch entre agentes leva < 50ms
```

### RF-004: Agentes Nativos - 19 Agentes do Squad
**Prioridade:** Must (3 no MVP, 19 no v1.0)  
**Ator:** Usuário Primário  
**Descrição:** Implementar 19 agentes como módulos Rust em `agents/`: planmaster, backdev, frontdev, engai, kalimaster, opsmaster, qamaster, mobilemaster, paymaster, offermaster, socialmaster, stylemaster, protomaster, issuemaster, aprendiz, datamaster, mh-codebase-inspection, mh-systematic-debugging, mh-test-driven-development. Cada agente carrega sua skill/prompt/template do dotfiles repo.  
**Dependências:** RF-003, RF-015 (dotfiles), RF-007 (LLM gateway)  
**Critério de Aceite (Gherkin):**
```gherkin
Given o usuário executa `ferro chat "planeja feature X"`
When o sistema seleciona planmaster (auto ou manual)
Then planmaster gera SRS + UML + backlog em docs/
And salva artefatos em docs/01-visao-e-escopo.md, docs/03-arquitetura-e-uml.md, docs/05-backlog.md
And commita em develop com mensagem "docs(plan): ..."
```

### RF-005: Agentes Nativos - Seleção Automática/Manual
**Prioridade:** Should  
**Ator:** Usuário Primário  
**Descrição:** Permitir seleção de agente via: (a) auto — LLM classifica intent e escolhe agente; (b) manual — command palette `> agent: planmaster`; (c) CLI `--agent planmaster`.  
**Dependências:** RF-003, RF-007  
**Critério de Aceite (Gherkin):**
```gherkin
Given o usuário digita `ferro chat "auditoria segurança"`
When o sistema processa
Then seleciona automaticamente kalimaster
And executa kalimaster-sast no código atual
And mostra findings no TUI com navegação para código
```

### RF-006: Perfil/Dotfiles - Git Clone/Pull/Push
**Prioridade:** Must  
**Ator:** Usuário Primário  
**Descrição:** Implementar `ferro --sync-profile pull|push|status` usando `git2-rs` (libgit2 vendored). Clone inicial: `git clone <dotfiles-url> ~/.config/ferro`. Pull: fetch + rebase local. Push: commit alterações locais (config, themes, agents) + push. Status: mostra ahead/behind.  
**Dependências:** RF-002 (hot reload), RNF-005 (confiabilidade)  
**Critério de Aceite (Gherkin):**
```gherkin
Given o usuário executa `ferro --sync-profile pull` em máquina nova
When o comando roda
Then clona dotfiles repo se não existe
And faz pull --rebase se existe
And dispara hot reload da config (RF-002)
And loga em .ferro-sync.log com timestamp
```

### RF-007: LLM Gateway - Subprocess Manager com Fallback Chain
**Prioridade:** Must  
**Ator:** Sistema (Agentes)  
**Descrição:** Implementar `LlmGateway` trait com implementação `CliWrapperGateway` que gerencia subprocessos: `opencode` (primary) → `claude` → `gh copilot` → `ollama` (fallback chain). Streaming JSONL parsing incremental, render no TUI tempo real. Cache semântico local (SQLite + embeddings via candle). Trace.json collection (modelo, tokens, custo, latência). Timeout 30s + retry 2x + circuit breaker.  
**Dependências:** RF-004 (agentes consomem), RNF-001 (latência), RNF-003 (segurança)  
**Critério de Aceite (Gherkin):**
```gherkin
Given um agente chama LlmGateway::stream_chat()
When opencode está disponível
Then usa opencode com router OpenRouter/Ollama
And stream JSONL renderiza no TUI token a token
And trace.json é salvo com modelo, tokens, custo, latência
When opencode falha (timeout/erro)
Then tenta claude automaticamente
And depois gh copilot
And por fim ollama local
And loga fallback no trace
```

### RF-008: LLM Gateway - Cache Semântico
**Prioridade:** Should  
**Ator:** Sistema  
**Descrição:** Cache SQLite com embeddings (candle/onnxruntime) para deduplicar prompts idênticos/semanticamente similares (threshold cosine > 0.95). Key: hash(prompt + model + params). Value: resposta + metadata. TTL configurável (default 24h). Hit < 10ms.  
**Dependências:** RF-007, RNF-001  
**Critério de Aceite (Gherkin):**
```gherkin
Given o mesmo prompt é enviado duas vezes ao mesmo modelo
When a segunda chamada ocorre
Then cache hit retorna resposta em < 10ms
And trace.json mostra cache_hit: true
And custo/latência reportados como zero
```

### RF-009: Widgets Ricos - Syntax Highlight (tree-sitter)
**Prioridade:** Must  
**Ator:** Usuário Primário  
**Descrição:** Integrar `syntect` + `tree-sitter` via `ratatui-code-editor` para syntax highlight em: code editor widget, diff view, file preview. Grammars bundled como features opcionais (rust, python, js, ts, go, java, toml, json, yaml, md).  
**Dependências:** RF-001, RNF-001 (60fps)  
**Critério de Aceite (Gherkin):"
```gherkin
Given o usuário abre arquivo .rs no file explorer
When o código é renderizado
Then syntax highlight aplica cores corretas (keywords, types, macros)
And scrolling 60fps mantido em arquivos 10k+ linhas
And tree-sitter parsing incremental não bloqueia UI
```

### RF-010: Widgets Ricos - Diff Inline (Myers Algorithm)
**Prioridade:** Must  
**Ator:** Usuário Primário  
**Descrição:** Implementar `DiffView` widget com `diffrs` (Myers O(ND)): side-by-side, unified, inline modes. Syntax highlight no diff. Navegação: `n`/`p` next/prev hunk, `Enter` abre arquivo no editor. Line staging (git add -p style).  
**Dependências:** RF-009, RF-013 (git UI)  
**Critério de Aceite (Gherkin):**
```gherkin
Given o usuário abre diff de um commit no git log
When o DiffView renderiza
Then mostra unified diff com syntax highlight
And navegação n/p move entre hunks
And Enter no hunk abre arquivo na linha correspondente
And staging linha a linha funciona (space para toggle)
```

### RF-011: Widgets Ricos - LSP Client (Diagnostics Inline)
**Prioridade:** Must (diagnostics) / Could (completion/hover)  
**Ator:** Usuário Primário  
**Descrição:** Integrar `tower-lsp` + `lsp-types` como widget `LspWidget`. Iniciar language server (rust-analyzer, typescript-language-server, etc.) via stdio. Mostrar diagnostics inline (virtual text vermelho/amarelo) + painel lateral com lista. Code actions via command palette. Go-to-definition (Ctrl+Click). Completion/hover: fase 2.  
**Dependências:** RF-001, RF-009, RNF-001  
**Critério de Aceite (Gherkin):**
```gherkin
Given o usuário edita arquivo .rs com erro de tipo
When rust-analyzer processa
Then diagnostic inline aparece na linha com erro
And painel lateral lista todos diagnostics do workspace
And command palette > "code actions" mostra quick-fixes
And Ctrl+Click em função vai para definição
```

### RF-012: Widgets Ricos - File Explorer
**Prioridade:** Should  
**Ator:** Usuário Primário  
**Descrição:** File tree navegável (ratatui-explorer style): expand/collapse, fuzzy filter (/), create/delete/rename/move (F2, Del, Ctrl+N), git status indicators (M, A, D, ??), open no editor (Enter), reveal in git log (g l).  
**Dependências:** RF-001, RF-013  
**Critério de Aceite (Gherkin):"
```gherkin
Given o usuário pressiona Ctrl+E
When file explorer abre
Then mostra árvore do workspace com git status icons
And fuzzy filter / funciona em tempo real
And F2 renomeia, Del deleta, Ctrl+N cria arquivo
And Enter abre no editor widget
```

### RF-013: Git UI - Status, Log, Stage, Commit, Push
**Prioridade:** Must  
**Ator:** Usuário Primário  
**Descrição:** Git widget usando `git2-rs`: status (staged/unstaged/untracked), log (graph, author, date, message), stage/unstage file/hunk/line, commit (editor mensagem convencional), push/pull, branch list/checkout/create/delete, stash list/pop/drop. Integra com DiffView (RF-010).  
**Dependências:** RF-010, RF-012, RNF-005  
**Critério de Aceite (Gherkin):**
```gherkin
Given o usuário abre git widget (Ctrl+G)
When visualiza status
Then mostra staged/unstaged/untracked com cores
And stage file (s) / unstage (u) / stage hunk (Shift+S) funcionam
And commit abre editor com template Conventional Commits
And push/pull faz fetch + rebase/merge conforme config
And branch list mostra local/remote com ahead/behind
```

### RF-014: Session/History - Persistência SQLite
**Prioridade:** Must  
**Ator:** Sistema / Usuário  
**Descrição:** SQLite (sqlx) para persistir: sessões (id, created_at, agent, workspace), mensagens (role, content, tool_calls, tokens, cost), tool calls (name, args, result, duration), traces (model, latency, cache_hit). Full-text search (FTS5) em mensagens. Export/import JSONL. Auto-save a cada mensagem.  
**Dependências:** RF-007 (traces), RNF-005 (confiabilidade)  
**Critério de Aceite (Gherkin):**
```gherkin
Given o usuário tem sessão ativa com 50 mensagens
When fecha e reabre ferro
Then sessão anterior é restaurada com histórico completo
And busca full-text "planmaster" retorna mensagens relevantes
And export JSONL gera arquivo válido com todos campos
```

### RF-015: MCP Client - JSON-RPC 2.0 Nativo
**Prioridade:** Should  
**Ator:** Agentes / Usuário  
**Descrição:** Cliente MCP nativo (sem subprocesso): JSON-RPC 2.0 over stdio/TCP. Discovery de servers via `mcp/` config no dotfiles. Tool calling unificado: agente chama `mcp.call_tool("server", "tool", args)` → gateway roteia → resposta async. Suporte a resources, prompts, sampling.  
**Dependências:** RF-003 (agentes usam), RF-007 (gateway pattern)  
**Critério de Aceite (Gherkin):**
```gherkin
Given profile.toml tem mcp.server.git = { command = "mcp-server-git" }
When agente chama mcp.call_tool("git", "log", {"limit": 10})
Then MCP client inicia server via stdio
And envia request JSON-RPC
And recebe response assíncrona
And retorna resultado tipado para agente
```

### RF-016: Plugin System - Dynamic Loading (dylib + WASM)
**Prioridade:** Must (dylib MVP) / Should (WASM v1.1)  
**Ator:** Contribuidores / Usuário Avançado  
**Descrição:** Sistema de plugins híbrido: Fase 1 — `cdylib` nativo via `libloading` (hot reload, performance nativo). Fase 2 — `wasmtime` para sandbox WASI + capability-based (fs:read, net:http, env:read). Interface comum via `Agent` trait (dylib) ou WIT (WASM). Cargo features para compilação seletiva.  
**Dependências:** RF-003, RNF-003 (segurança), RNF-006  
**Critério de Aceite (Gherkin):"
```gherkin
Given o usuário compila agente custom como cdylib
When coloca .so/.dll em ~/.config/ferro/plugins/
And reinicia ferro (ou hot reload)
Then agente aparece no registry com nome customizado
And handle() executa com performance nativa
When v1.1: agente WASM com capability fs:read-only
Then wasmtime instancia com sandbox
And acesso a fs fora do permitido falha com trap
```

### RF-017: Segurança - Secret Redaction + Kalimaster-SAST Integrado
**Prioridade:** Must  
**Ator:** Sistema / Kalimaster  
**Descrição:** Redação automática de secrets (API keys, tokens, passwords) em: logs, traces, diff views, LLM prompts (pré-processamento). Integrar `kalimaster-sast` skill no build (pre-commit hook) e on-demand (`ferro audit --sast`). SAST roda: cargo-audit, clippy, secrets scan (gitleaks), dependency check.  
**Dependências:** RF-007, RF-013, RNF-003  
**Critério de Aceite (Gherkin):"
```gherkin
Given código contém `api_key = "sk-1234567890abcdef"`
When diff é mostrado no TUI
Then secret aparece como `api_key = "***REDACTED***"`
When trace.json é salvo
Then prompt enviado ao LLM tem secret redacted
When `ferro audit --sast` roda
Then gitleaks + cargo-audit + clippy rodam
And laudo gerado em docs/07-security/LAUDO.md
```

### RF-018: Command Palette - Fuzzy Search Universal
**Prioridade:** Must  
**Ator:** Usuário Primário  
**Descrição:** Command palette (Ctrl+P) com `skim` backend: ações (new session, sync profile, audit), agentes (switch, list), arquivos (open, create), config (theme, keybindings), git (commit, push, log), LSP (code actions, goto def). Scoring: frequência + recência + match quality.  
**Dependências:** RF-001, RF-003, RF-012, RF-013  
**Critério de Aceite (Gherkin):"
```gherkin
Given o usuário pressiona Ctrl+P
When digita "pla"
Then planmaster aparece no topo (frequência alta)
And "git: push" aparece
And "theme: dracula" aparece
And Enter executa ação selecionada
And ESC fecha palette
```

### RF-019: CLI Interface - Subcomandos Principais
**Prioridade:** Must  
**Ator:** Usuário Primário / Scripts  
**Descrição:** CLI via `clap` derive: `ferro` (abre TUI), `ferro chat "prompt"` (non-interactive), `ferro --agent <id> "prompt"`, `ferro --sync-profile pull|push|status`, `ferro audit --sast|--dast`, `ferro --version`, `ferro --help`. Completions bash/zsh/fish.  
**Dependências:** RF-004, RF-006, RF-007  
**Critério de Aceite (Gherkin):"
```gherkin
Given o usuário executa `ferro chat "resuma main.rs" --agent backdev`
When comando roda non-interactive
Then backdev processa prompt
And resposta é impressa no stdout (sem TUI)
And exit code 0 em sucesso
And `ferro --help` mostra todos subcomandos com descrições
```

### RF-020: Sincronização Multi-PC - Auto Pull + Anti-Colisão
**Prioridade:** Must  
**Ator:** Sistema (Agentes)  
**Descrição:** Implementar regras da Seção 10 do `AGENT_OPERATING_RULES.md`: auto pull se idle > 2h (verifica `git log -1 --format=%ct`), anti-colisão deriva branch se `BEHIND > 0 && AHEAD > 0`, log em `.ferro-sync.log`. Configurável via `profile.toml [git.sync]`.  
**Dependências:** RF-006, RF-013, RNF-005  
**Critério de Aceite (Gherkin):"
```gherkin
Given máquina A fez push em develop 3h atrás
And máquina B está idle há 3h
When usuário em máquina B executa primeira task
Then auto pull --rebase ocorre silenciosamente
And .ferro-sync.log registra sync com timestamp
Given máquina A e B têm commits locais diferentes em develop
When máquina B inicia trabalho
Then branch derivada DEV-sync-pc-b-20261028-143000 é criada
And trabalho continua na branch derivada
```

---

## 4. Requisitos Não-Funcionais (RNF)

### RNF-001: Performance - Cold Start e Throughput
**Categoria:** Performance  
**Prioridade:** Must  
**Descrição:** Cold start < 2s (binary load + config + TUI init). TUI frame time < 16ms (60fps sustentado). Agent switch < 50ms. LLM first token < 500ms (local) / < 2s (remote). LLM streaming > 50 tok/s. Semantic cache hit < 10ms.  
**Métrica/Target:** `hyperfine 'ferro --version'` < 2s; `tracing` spans frame time p95 < 16ms; benchmark interno agent switch; `trace.json` LLM metrics  
**Validação:** CI benchmark job semanal + `cargo bench` suite

### RNF-002: Performance - Memória
**Categoria:** Performance  
**Prioridade:** Must  
**Descrição:** RSS idle < 50MB. Com 3 agentes carregados < 100MB. Com LSP ativo (rust-analyzer) < 300MB. Sem memory leaks em sessão 8h (valgrind/heaptrack).  
**Métrica/Target:** `ps_record` / `memusage` monitoring contínuo  
**Validação:** Teste de soak 8h em CI

### RNF-003: Segurança - Sandbox e Dados
**Categoria:** Security  
**Prioridade:** Must  
**Descrição:** WASM sandbox (wasmtime) com capability-based: `fs:read`, `fs:write`, `net:http`, `env:read` — default deny. Native dylib: isolamento via processo separado (opcional v1.1). Secrets: redaction em logs/traces/prompts (RF-017). LGPD/GDPR: dados locais, opt-in telemetry, right to delete (apaga `~/.config/ferro/sessions.db`).  
**Métrica/Target:** 0 vulnerabilidades críticas em `cargo audit`; 0 secrets vazados em logs (teste automatizado)  
**Validação:** `kalimaster-sast` em todo PR; pen test anual

### RNF-004: Usabilidade - Acessibilidade e UX
**Categoria:** Usability  
**Prioridade:** Must  
**Descrição:** Screen reader friendly (ratatui+crossterm emite eventos acessíveis). High contrast themes (WCAG AA). Reduced motion (respecta `prefers-reduced-motion`). Keyboard-only operation (zero mouse required). Onboarding: `ferro --help` + primeiro run wizard.  
**Métrica/Target:** 0 violações axe-core em CI; 100% operações via teclado  
**Validação:** CI axe-core check; teste manual com NVDA/Orca

### RNF-005: Confiabilidade - Persistência e Sync
**Categoria:** Reliability  
**Prioridade:** Must  
**Descrição:** SQLite WAL mode + backup automático (cópia a cada 1h). Git sync reliability 100% (roundtrip pull→edit→push). Crash recovery: sessão restaurada ao reabrir. Config hot-reload sem perda estado.  
**Métrica/Target:** 0 corrupção SQLite em 10k ciclos teste; sync success rate 100%  
**Validação:** Chaos testing (kill -9 aleatório + recover)

### RNF-006: Manutenibilidade - Arquitetura Modular
**Categoria:** Maintainability  
**Prioridade:** Must  
**Descrição:** Cargo workspace: `ferro-core`, `ferro-tui`, `ferro-agents`, `ferro-gateway`, `ferro-plugins`, `ferro-mcp`, `ferro-cli`. Cada agente = crate independente. Plugin API estável (semver). Documentação: `cargo doc` + ADRs em `docs/04-ADRs/`. Test coverage > 80% unit + 100% integration critical paths.  
**Métrica/Target:** `cargo llvm-cov` > 80%; `cargo clippy -- -D warnings` clean; `cargo fmt --check` pass  
**Validação:** CI em todo PR

### RNF-007: Portabilidade - Cross-Platform
**Categoria:** Portability  
**Prioridade:** Should  
**Descrição:** Build e teste em Linux (x86_64, aarch64), macOS (x86_64, arm64), Windows (x86_64). Single binary via `cargo install`. Crossterm backend abstrai terminal. Git2 vendored libgit2 (sem system deps).  
**Métrica/Target:** CI matrix 6 targets passing; binary size < 15MB (strip + LTO)  
**Validação:** GitHub Actions matrix build

### RNF-008: Interoperabilidade - MCP e CLI Wrappers
**Categoria:** Interoperability  
**Prioridade:** Should  
**Descrição:** MCP client compatível spec 2024-11-05 (resources, tools, prompts, sampling). CLI wrappers: opencode ≥ 0.5, claude ≥ 1.0, gh copilot ≥ 1.0, ollama ≥ 0.1. JSONL streaming format estável.  
**Métrica/Target:** Integração testada com 3+ MCP servers; fallback chain funcional  
**Validação:** Testes de contrato MCP + integração nightly

---

## 5. Requisitos de Interface

### 5.1 Interfaces de Usuário (UI)
- **TUI Principal:** ratatui + crossterm, layout grid (header: tabs + status; body: splits 3 painéis; footer: command line + hints)
- **Painel Agentes:** Lista 19 agentes, status (loaded/active/idle), atalhos (1-9, a-z)
- **Painel Sessões:** Histórico, busca, export, nova sessão
- **Painel Arquivos:** Explorer + editor + diff + git + LSP (tabs inferiores)
- **Command Palette:** Overlay central, fuzzy, ações categorizadas
- **Themes:** 20+ base16 via opaline, customizável via `themes/*.toml`
- **Keybindings:** Vim default, customizável via `keybindings/*.toml`, leader key `<Space>`

### 5.2 Interfaces de Hardware
- Terminal: qualquer compatível crossterm (xterm-256color, tmux, alacritty, kitty, wezterm, Windows Terminal)
- GPU: opcional (tachyonfx usa CPU rendering; futuro: wgpu backend)
- Rede: apenas para LLM gateway (HTTPS) e MCP servers (stdio/TCP local)

### 5.3 Interfaces de Software / APIs
| Interface | Protocolo | Schema | Consumidores |
|-----------|-----------|--------|--------------|
| **Agent Trait** | Rust trait (in-process) | `Agent`, `AgentContext`, `AgentInput`, `AgentOutputStream` | Core TUI, plugins |
| **LLM Gateway** | Rust trait + subprocess JSONL | `ChatRequest`, `ChatEvent` (stream), `ModelInfo` | Agentes |
| **MCP Client** | JSON-RPC 2.0 (stdio/TCP) | MCP 2024-11-05 spec | Agentes, user tools |
| **Config** | TOML (figment layering) | `profile.toml`, `theme.toml`, `keybindings.toml` | Core, hot reload |
| **Sessions DB** | SQLite (sqlx) | `sessions`, `messages`, `tool_calls`, `traces` (FTS5) | Core, export/import |
| **Git** | libgit2 (git2-rs) | Git objects (commit, tree, index, ref) | Git UI, sync |

### 5.4 Interfaces de Comunicação
- **LLM Providers:** HTTPS (OpenRouter, Anthropic, OpenAI, Ollama local) — via CLI wrappers
- **MCP Servers:** stdio (local processes) ou TCP (remote) — JSON-RPC 2.0
- **Git Remotes:** SSH (git@), HTTPS — via libgit2
- **Telemetry (opt-in):** HTTPS POST para endpoint configurado — apenas métricas agregadas anônimas

---

## 6. Regras de Negócio (RN)

### RN-001: Agente Nativo Prioritário sobre CLI Wrapper
**Descrição:** Se agente nativo existe para a tarefa (ex: planmaster para planejamento), ele **deve** ser usado em vez de delegar para LLM via gateway. CLI wrapper só para tarefas sem agente nativo ou quando agente explicitamente chama `LlmGateway`.  
**Origem:** Arquitetura (controle total, latência zero, privacidade)  
**Validação:** Lint rule: `ferro lint --agent-usage` verifica chamadas diretas a gateway vs agente

### RN-002: Configuração via Dotfiles Git (Single Source of Truth)
**Descrição:** Toda configuração (profile, agents, themes, keybindings, MCP servers) vive no dotfiles repo. `ferro --sync-profile` é única via de sincronização. Config local (`~/.config/ferro/`) é **cache** do repo — nunca editado manualmente em produção.  
**Origem:** Requisito onboarding multi-máquina (J4)  
**Validação:** `ferro --sync-profile status` mostra clean se sync; CI testa roundtrip

### RN-003: Sandbox WASM para Plugins Terceiros
**Descrição:** Plugins de terceiros (não assinados pelo core team) **devem** rodar em WASM sandbox (wasmtime) com capabilities mínimas. Plugins nativos (dylib) só para time confiável (assinados).  
**Origem:** Segurança (supply chain, execução código não confiável)  
**Validação:** `ferro plugin verify <path>` checa assinatura/capabilities; CI bloqueia WASM sem capabilities declaradas

### RN-004: Redaction Obrigatória de Secrets
**Descrição:** Qualquer string que match regex de secret (API keys, JWT, passwords, tokens) **deve** ser redacted antes de: log, trace.json, LLM prompt, diff view, file preview. Exceção: usuário explicitamente autoriza via `--no-redact` (apenas debug local).  
**Origem:** LGPD/GDPR, segurança operacional  
**Validação:** Testes unitários com corpus de secrets conhecidos; `kalimaster-sast` valida

### RN-005: Git Workflow Develop-First Obrigatório
**Descrição:** Todo desenvolvimento ocorre em `develop` ou branches derivadas (`ISSUE-*`, `DEV-*`, `SEG-*`, `DOC-*`). **Nunca** commit direto em `main`. Merge para `main` só via `release/*` branch + tag semver. Conventional Commits obrigatórios.  
**Origem:** `planmaster-git` skill, `AGENT_OPERATING_RULES.md`  
**Validação:** Git hook `commit-msg` valida formato; CI bloqueia push para `main` sem PR

### RN-006: Observabilidade Obrigatória (Trace.json)
**Descrição:** Toda chamada LLM **deve** gerar `trace.json` com: timestamp, model, provider, tokens_in/out, cost_usd, latency_ms, cache_hit, agent_id, session_id. Armazenado em SQLite + exportável.  
**Origem:** EngAI Global (FinOps, avaliação)  
**Validação:** Schema JSON validado; CI checa presença em testes de integração

### RN-007: Anti-Colisão Branch Automática
**Descrição:** Se detecção de colisão (outro PC pushou na mesma branch + commits locais), **sempre** derivar branch `<tipo>-<original>-<hostname>-<timestamp>`. Nunca force push, nunca merge automático sem revisão.  
**Origem:** Requisito multi-PC (Seção 10 AGENT_OPERATING_RULES)  
**Validação:** Teste de integração simula 2 workspaces concorrentes

### RN-008: Auto Pull em Idle > 2h
**Descrição:** Antes de qualquer operação escrita (commit, push, sync), se última atividade > 2h: executar `git fetch` + `pull --rebase` automático. Stash automático se divergência. Log em `.ferro-sync.log`.  
**Origem:** Requisito multi-PC  
**Validação:** Teste de integração com clock mockado

---

## 7. Matriz de Rastreabilidade

| RF/RNF/RN | User Story (Fase 5) | Critério Aceite | Teste Unitário | Teste Integração | Teste E2E |
|-----------|---------------------|-----------------|----------------|------------------|-----------|
| RF-001 | US-001 | AC-001 | TU-001 | TI-001 | TE-001 |
| RF-002 | US-002 | AC-002 | TU-002 | TI-002 | TE-001 |
| RF-003 | US-003 | AC-003 | TU-003 | TI-003 | TE-002 |
| RF-004 | US-004 | AC-004 | TU-004 | TI-004 | TE-002 |
| RF-005 | US-005 | AC-005 | TU-005 | TI-005 | TE-002 |
| RF-006 | US-006 | AC-006 | TU-006 | TI-006 | TE-003 |
| RF-007 | US-007 | AC-007 | TU-007 | TI-007 | TE-003 |
| RF-008 | US-008 | AC-008 | TU-008 | TI-008 | TE-003 |
| RF-009 | US-009 | AC-009 | TU-009 | TI-009 | TE-004 |
| RF-010 | US-010 | AC-010 | TU-010 | TI-010 | TE-004 |
| RF-011 | US-011 | AC-011 | TU-011 | TI-011 | TE-004 |
| RF-012 | US-012 | AC-012 | TU-012 | TI-012 | TE-004 |
| RF-013 | US-013 | AC-013 | TU-013 | TI-013 | TE-005 |
| RF-014 | US-014 | AC-014 | TU-014 | TI-014 | TE-005 |
| RF-015 | US-015 | AC-015 | TU-015 | TI-015 | TE-005 |
| RF-016 | US-016 | AC-016 | TU-016 | TI-016 | TE-006 |
| RF-017 | US-017 | AC-017 | TU-017 | TI-017 | TE-006 |
| RF-018 | US-018 | AC-018 | TU-018 | TI-018 | TE-001 |
| RF-019 | US-019 | AC-019 | TU-019 | TI-019 | TE-001 |
| RF-020 | US-020 | AC-020 | TU-020 | TI-020 | TE-003 |
| RNF-001 | US-PERF | AC-PERF | TU-PERF | TI-PERF | TE-PERF |
| RNF-002 | US-MEM | AC-MEM | TU-MEM | TI-MEM | TE-PERF |
| RNF-003 | US-SEC | AC-SEC | TU-SEC | TI-SEC | TE-SEC |
| RNF-004 | US-A11Y | AC-A11Y | TU-A11Y | TI-A11Y | TE-A11Y |
| RNF-005 | US-REL | AC-REL | TU-REL | TI-REL | TE-REL |
| RNF-006 | US-MAINT | AC-MAINT | TU-MAINT | TI-MAINT | - |
| RNF-007 | US-PORT | AC-PORT | - | TI-PORT | TE-PORT |
| RNF-008 | US-INTEROP | AC-INTEROP | TU-INTEROP | TI-INTEROP | TE-INTEROP |
| RN-001 | US-004 | AC-004 | TU-004 | TI-004 | TE-002 |
| RN-002 | US-006 | AC-006 | TU-006 | TI-006 | TE-003 |
| RN-003 | US-016 | AC-016 | TU-016 | TI-016 | TE-006 |
| RN-004 | US-017 | AC-017 | TU-017 | TI-017 | TE-006 |
| RN-005 | All | All | - | TI-GIT | TE-GIT |
| RN-006 | US-007 | AC-007 | TU-007 | TI-007 | TE-003 |
| RN-007 | US-020 | AC-020 | TU-020 | TI-020 | TE-003 |
| RN-008 | US-020 | AC-020 | TU-020 | TI-020 | TE-003 |

---

## 8. Riscos e Mitigações

| ID | Risco | Prob. | Impacto | Mitigação | Responsável | Status |
|----|-------|-------|---------|-----------|-------------|--------|
| RSK-001 | Complexidade TUI "Obra" | Alta | Alto | Fasear MVP→Essencial→Intermediário→Obra; ratatui widgets compostos; PoC Sprint 1 | backdev/frontdev | Planejado |
| RSK-002 | WASM sandbox instável (wasmtime API) | Média | Alto | MVP: dylib nativo apenas; WASM v1.1+; travar versão wasmtime 20+ | engai/opsmaster | Planejado |
| RSK-003 | git2-rs segfaults (libgit2) | Baixa | Alto | Feature `vendored-libgit2`; fallback `git` CLI subprocess para ops complexas | backdev | Planejado |
| RSK-004 | LSP client complexidade (tower-lsp) | Alta | Médio | MVP: só rust-analyzer + diagnostics inline; completion/hover Sprint 4+ | frontdev | Planejado |
| RSK-005 | tree-sitter grammars bundling | Média | Médio | Grammars como features opcionais; download on-demand via `tree-sitter` CLI | backdev | Planejado |
| RSK-006 | CLI wrapper flakiness (opencode/claude) | Média | Médio | Timeout 30s + retry 2x + circuit breaker; health check prévio; fallback chain | engai | Planejado |
| RSK-007 | Bundle size > 15MB | Média | Médio | Cargo features por agente; `strip = true`, `lto = true`, `panic = "abort"`; `cargo bloat` CI | opsmaster | Planejado |
| RSK-008 | 19 agentes = code bloat / compile time | Média | Médio | Cargo features seletivas; dynamic loading preferido; workspace modular | planmaster | Planejado |
| RSK-009 | Config sync conflitos multi-PC | Média | Alto | Anti-colisão branch derivação (RN-007); auto-pull rebase (RN-008); log auditável | backdev | Implementado |
| RSK-010 | Semantic cache false positives | Baixa | Médio | Threshold cosine 0.95 ajustável; TTL 24h; métricas hit/miss no trace | engai | Planejado |

---

## 9. Glossário

| Termo | Definição |
|-------|-----------|
| **FERRO** | Codename do agente TUI (Ferro = Iron/Fe, base do aço/Rust) |
| **Agent Trait** | Interface Rust unificada: `fn handle(&self, ctx: AgentCtx, input: Input) -> Stream<Output>` |
| **Dotfiles Repo** | Git repo com `profile.toml`, `agents/`, `themes/`, `keybindings/`, `mcp/` |
| **Semantic Cache** | SQLite + embeddings (local) para deduplicar chamadas LLM idênticas |
| **MCP Client** | Implementação nativa do Model Context Protocol (JSON-RPC 2.0) |
| **Kalimaster-SAST** | Skill de auditoria estática integrada no build/CI |
| **Command Palette** | Fuzzy search (skim/fzf-like) para ações, agentes, arquivos, config |
| **TUI Nível Obra** | ratatui + tachyonfx + opaline + syntect + tower-lsp + git2 + diffrs |
| **MoSCoW** | Priorização: Must (obrigatório), Should (importante), Could (desejável), Won't (fora escopo) |
| **Gherkin** | Formato Given/When/Then para critérios de aceite testáveis |
| **ADR** | Architecture Decision Record — decisão arquitetural documentada |

---

## 10. Aprovações

| Role | Nome | Assinatura | Data |
|------|------|------------|------|
| **PLANMASTER** | | | 28/10/2026 |
| **Tech Lead (Você)** | | | |
| **Arquiteto Segurança (Kalimaster)** | | | |
| **DevOps (Opsmaster)** | | | |

---

> **Próximo passo:** FASE 4 — UML (Casos de Uso, Classes, Sequência) + ADRs (Plugin Architecture, LLM Gateway, Sandbox, Sync Multi-PC)