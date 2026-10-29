# Backlog Scrum — Agente TUI Rust (FERRO)

**Versão:** 1.0  
**Data:** 28/10/2026  
**Autor:** PLANMASTER  
**Branch:** develop  
**Base:** `docs/SRS.md`, `docs/03-arquitetura-e-uml.md`, `docs/04-ADRs/`

---

## 1. Product Vision

> **Elevator Pitch:** FERRO é um agente TUI nativo em Rust que embute 19 agentes especializados do squad (planmaster, backdev, frontdev, engai, kalimaster, etc.) como módulos internos, eliminando latência de subprocessos, provendo UX nível "Obra" (diff inline, LSP, syntax highlight, git UI, command palette), sincronizando configuração via dotfiles Git multi-PC, e usando CLI wrappers (opencode/claude/gh copilot/ollama) apenas para chamadas LLM — tudo em um single binary < 15MB, cold start < 2s, 60fps.

---

## 2. Épicos

| Épico ID | Nome | Descrição | Prioridade | Sprint Alvo |
|----------|------|-----------|------------|-------------|
| **EPIC-001** | **Core TUI Foundation** | Event loop, layout splits/tabs, vim keys, command palette, themes, mouse, hot reload config | Must | Sprint 1 |
| **EPIC-002** | **Agent System** | Agent trait, registry dinâmico, 7 agentes MVP (planmaster, backdev, frontdev, engai, kalimaster, opsmaster, qamaster), auto-seleção | Must | Sprint 2-3 |
| **EPIC-003** | **LLM Gateway** | CliWrapperGateway (opencode→claude→gh→ollama), streaming JSONL, cache semântico (candle), trace.json, fallback chain | Must | Sprint 2 |
| **EPIC-004** | **Rich Widgets** | Syntax highlight (tree-sitter), diff inline (Myers), LSP diagnostics (tower-lsp), file explorer, git UI (git2) | Must | Sprint 3-4 |
| **EPIC-005** | **Profile & Sync** | Dotfiles git clone/pull/push, figment layering, hot reload, auto-pull idle > 2h, anti-colisão branch derivação | Must | Sprint 3 |
| **EPIC-006** | **Persistence & MCP** | SQLite (sqlx) sessions + FTS5 + JSONL export, MCP client JSON-RPC 2.0, session history | Should | Sprint 4 |
| **EPIC-007** | **Plugin System** | dylib nativo (libloading) hot reload, wasmtime WASM sandbox (v1.1), capability-based security | Could | Sprint 5 |
| **EPIC-008** | **Security & Polish** | Secret redaction, kalimaster-sast integrado, acessibilidade (axe-core), CI/CD, release automation, docs | Must | Sprint 5-6 |

---

## 3. User Stories

### EPIC-001: Core TUI Foundation

#### US-001: Inicialização TUI + Dashboard
**Como** Usuário Primário  
**Quero** executar `ferro` e ver um dashboard funcional em < 2s  
**Para** começar a trabalhar imediatamente  

**Critérios de Aceite:**
- [ ] AC-001: `ferro` abre TUI em < 2s (cold start)
- [ ] AC-002: Dashboard mostra 3 painéis: Agents, Sessions, Files
- [ ] AC-003: Header com tabs navegáveis (Tab/Shift-Tab)
- [ ] AC-004: Footer com hints contextuais + clock
- [ ] AC-005: `ferro --version` imprime versão e sai em < 100ms

**DoR:** RF-001, RNF-001, ADR-006 aprovados  
**Estimativa:** 8 SP  
**DoD:** Código + testes unitários (layout, timing) + `cargo clippy` clean + doc

---

#### US-002: Layout Splits Redimensionáveis + Vim Keys
**Como** Usuário Primário  
**Quero** redimensionar splits com Ctrl+h/j/k/l e navegar com vim keys  
**Para** organizar workspace como no Neovim/tmux  

**Critérios de Aceite:**
- [ ] AC-001: Ctrl+h/j/k/l redimensiona split focado (mínimo 10 colunas/linhas)
- [ ] AC-002: h/j/k/l/navega entre widgets dentro do split
- [ ] AC-003: gg/G vai para topo/fim de lista
- [ ] AC-004: / abre busca no widget focado
- [ ] AC-005: Mouse scroll/click funciona (opcional, config)

**DoR:** US-001 Done  
**Estimativa:** 5 SP

---

#### US-003: Command Palette (Ctrl+P) com Fuzzy Search
**Como** Usuário Primário  
**Quero** pressionar Ctrl+P e buscar ações/agentes/arquivos/config fuzzy  
**Para** acessar qualquer funcionalidade sem memorizar atalhos  

**Critérios de Aceite:**
- [ ] AC-001: Ctrl+P abre palette centralizada com animação slide+fade
- [ ] AC-002: Busca fuzzy (skim) em: ações, agentes, arquivos, themes, keybindings, git, LSP
- [ ] AC-003: Scoring: frequência + recência + match quality
- [ ] AC-004: Enter executa ação; ESC fecha; Tab completa
- [ ] AC-005: Categorias com ícones: 🎯 Ações, 🤖 Agentes, 📁 Arquivos, ⚙️ Config, 🔧 Git, 💡 LSP

**DoR:** US-001 Done, ADR-006 (skim)  
**Estimativa:** 5 SP

---

#### US-004: Theming (opaline) + Hot Reload
**Como** Usuário Primário  
**Quero** alternar themes base16 via command palette e ver mudança instantânea  
**Para** personalizar aparência sem reiniciar  

**Critérios de Aceite:**
- [ ] AC-001: 20+ themes base16 disponíveis (dracula, gruvbox, catppuccin, nord, tokyo-night, etc.)
- [ ] AC-002: Theme selector na command palette (`theme: dracula`)
- [ ] AC-003: Mudança aplica em < 50ms sem restart (hot reload)
- [ ] AC-004: Custom tokens FERRO (agent.active, git.added, etc.) funcionam
- [ ] AC-005: `reduced_motion = true` desabilita animações tachyonfx

**DoR:** US-001 Done, ADR-006 (opaline, tachyonfx)  
**Estimativa:** 3 SP

---

#### US-005: Config Hot Reload (figment + notify)
**Como** Usuário Primário  
**Quero** editar `~/.config/ferro/profile.toml` e ver mudanças aplicadas instantaneamente  
**Para** iterar configuração sem reiniciar  

**Critérios de Aceite:**
- [ ] AC-001: Editar profile.toml → config recarrega em < 100ms
- [ ] AC-002: Layering: defaults → file → env (FERRO_*) → CLI args
- [ ] AC-003: Validação schema: campos inválidos = erro claro no log + notificação TUI
- [ ] AC-004: Mudanças de keybindings/theme/agents aplicam imediatamente
- [ ] AC-005: Log `.ferro-sync.log` registra reload com timestamp

**DoR:** ADR-005 aprovado, RF-002  
**Estimativa:** 5 SP

---

### EPIC-002: Agent System

#### US-006: Agent Trait + Registry Dinâmico
**Como** Sistema  
**Quero** trait `Agent` unificado e registry que descobre agentes via Cargo features  
**Para** carregar apenas agentes necessários (binary size, compile time)  

**Critérios de Aceite:**
- [ ] AC-001: Trait `Agent` com `id()`, `name()`, `description()`, `capabilities()`, `handle()`, `configure()`
- [ ] AC-002: `AgentRegistry` lista agentes das features habilitadas
- [ ] AC-003: `cargo build --features planmaster,backdev` → registry tem 2 agentes
- [ ] AC-004: `cargo build --features all` → registry tem 19 agentes
- [ ] AC-005: Switch entre agentes < 50ms (benchmark)

**DoR:** ADR-007 aprovado, RF-003  
**Estimativa:** 8 SP

---

#### US-007: 7 Agentes MVP (planmaster, backdev, frontdev, engai, kalimaster, opsmaster, qamaster)
**Como** Usuário Primário  
**Quero** 7 agentes essenciais funcionando nativamente  
**Para** cobrir planejamento, implementação, IA, segurança, DevOps, QA  

**Critérios de Aceite:**
- [ ] AC-001: `planmaster` gera SRS + UML + backlog em `docs/` + commit
- [ ] AC-002: `backdev` implementa feature: código + testes + commit develop
- [ ] AC-003: `frontdev` cria componente React/TS/Tailwind/shadcn + storybook
- [ ] AC-004: `engai` configura RAG + MCP + gateway + guardrails
- [ ] AC-005: `kalimaster` roda SAST (cargo-audit, clippy, gitleaks) + laudo
- [ ] AC-006: `opsmaster` gera Dockerfile + compose + CI + runbook + custo
- [ ] AC-007: `qamaster` gera plano testes Gherkin + automação Playwright + cobertura
- [ ] AC-008: Cada agente carrega skill/prompt/template do dotfiles repo

**DoR:** US-006 Done, RF-004, skills disponíveis  
**Estimativa:** 21 SP (3 SP/agente)

---

#### US-008: Auto-Seleção de Agente por Intent
**Como** Usuário Primário  
**Quero** digitar `ferro chat "auditoria segurança"` e o sistema escolher `kalimaster` automaticamente  
**Para** não precisar especificar agente manualmente  

**Critérios de Aceite:**
- [ ] AC-001: Embedding-based selector (candle) classifica intent
- [ ] AC-002: Keywords fallback se embedding falhar
- [ ] AC-003: `ferro chat "planeja feature"` → planmaster
- [ ] AC-004: `ferro chat "implementa API"` → backdev
- [ ] AC-005: Command palette `> agent:` permite override manual

**DoR:** US-007 Done, ADR-007 (selector)  
**Estimativa:** 5 SP

---

### EPIC-003: LLM Gateway

#### US-009: CliWrapperGateway com Fallback Chain
**Como** Agente (planmaster, backdev, etc.)  
**Quero** chamar `LlmGateway::stream_chat()` e ter fallback automático opencode→claude→gh→ollama  
**Para** não falhar se um provider cai  

**Critérios de Aceite:**
- [ ] AC-001: Primary = opencode (router OpenRouter/Ollama, cache, guardrail, trace.json)
- [ ] AC-002: Fallback 1 = claude (Anthropic SDK wrapper)
- [ ] AC-003: Fallback 2 = gh copilot (GitHub Models)
- [ ] AC-004: Fallback 3 = ollama local (sem internet)
- [ ] AC-005: Timeout 30s + retry 2x + circuit breaker 5min por provider
- [ ] AC-006: Health check prévio (`opencode --version`) antes de chamar

**DoR:** ADR-002 aprovado, RF-007, EngAI Global Regra #1  
**Estimativa:** 8 SP

---

#### US-010: Streaming JSONL + Render Tempo Real no TUI
**Como** Usuário Primário  
**Quero** ver resposta do LLM token a token no TUI enquanto stream chega  
**Para** feedback imediato, sem esperar resposta completa  

**Critérios de Aceite:**
- [ ] AC-001: Parser JSONL incremental (skip linhas inválidas)
- [ ] AC-002: Widget chat renderiza tokens conforme chegam (60fps)
- [ ] AC-003: Tool calls aparecem como blocos colapsáveis
- [ ] AC-004: Cancelamento (Ctrl+C) para stream graciosamente
- [ ] AC-005: Latência overhead < 100ms vs CLI direto

**DoR:** US-009 Done, RF-007  
**Estimativa:** 5 SP

---

#### US-011: Cache Semântico (SQLite + candle embeddings)
**Como** Sistema  
**Quero** deduplicar prompts idênticos/semanticamente similares (cosine > 0.95)  
**Para** reduzir custo/latência LLM  

**Critérios de Aceite:**
- [ ] AC-001: Cache hit < 10ms (SQLite query)
- [ ] AC-002: Embeddings locais via candle (sem rede)
- [ ] AC-003: TTL 24h configurável
- [ ] AC-004: Trace.json mostra `cache_hit: true` + custo/latência = 0
- [ ] AC-005: Comando `ferro cache stats` mostra hit rate, size

**DoR:** US-009 Done, ADR-002, RF-008  
**Estimativa:** 5 SP

---

#### US-012: Trace.json Collection (FinOps)
**Como** Tech Lead  
**Quero** todo chamado LLM gerar `trace.json` com modelo, tokens, custo, latência, cache_hit, agent_id, session_id  
**Para** auditoria de custo e performance  

**Critérios de Aceite:**
- [ ] AC-001: trace.json salvo em SQLite (tabela `traces`) + arquivo `.ferro-traces.jsonl`
- [ ] AC-002: Campos: timestamp, model, provider, tokens_in/out, cost_usd, latency_ms, cache_hit, agent_id, session_id
- [ ] AC-003: Request/response JSON redacted (secrets)
- [ ] AC-004: `ferro traces export --since 1d --format csv` para análise
- [ ] AC-005: EngAI Global Regra #6 cumprida

**DoR:** US-009 Done, RN-006, EngAI Global  
**Estimativa:** 3 SP

---

### EPIC-004: Rich Widgets

#### US-013: Syntax Highlight (tree-sitter + syntect)
**Como** Usuário Primário  
**Quero** syntax highlight correto em editor, diff, preview para Rust, TS, Python, Go, TOML, JSON, YAML, MD  
**Para** ler/escrever código com cores semânticas  

**Critérios de Aceite:**
- [ ] AC-001: `ratatui-code-editor` + tree-sitter grammars (features opcionais por lang)
- [ ] AC-002: Scrolling 60fps em arquivos 10k+ linhas
- [ ] AC-003: Incremental parsing não bloqueia UI
- [ ] AC-004: Highlight aplica em: editor widget, diff view, file preview
- [ ] AC-005: Grammars bundladas no binary (features) + download on-demand

**DoR:** ADR-006 (syntect, tree-sitter), RF-009  
**Estimativa:** 8 SP

---

#### US-014: Diff Inline (Myers Algorithm + Staging Linha)
**Como** Usuário Primário  
**Quero** ver diff unified/inline/side-by-side com syntax highlight e fazer staging linha a linha  
**Para** revisar mudanças como `git add -p` mas visual  

**Critérios de Aceite:**
- [ ] AC-001: `diffrs` (Myers O(ND)) + syntax highlight no diff
- [ ] AC-002: 3 modos: unified (default), side-by-side (Ctrl+S), inline (Ctrl+I)
- [ ] AC-003: Navegação: `n`/`p` next/prev hunk, `Enter` abre arquivo na linha
- [ ] AC-004: Staging: `Space` toggle linha, `s` stage hunk, `u` unstage
- [ ] AC-005: Integra com GitWidget (US-018)

**DoR:** US-013 Done, ADR-006 (diffrs), RF-010  
**Estimativa:** 8 SP

---

#### US-015: LSP Diagnostics Inline + Code Actions
**Como** Usuário Primário  
**Quero** ver erros/warnings inline (virtual text) + painel lateral + code actions via palette  
**Para** corrigir código sem sair do TUI  

**Critérios de Aceite:**
- [ ] AC-001: `tower-lsp` + `lsp-types` inicia rust-analyzer via stdio
- [ ] AC-002: Diagnostics inline: vermelho (error), amarelo (warning), azul (info)
- [ ] AC-003: Painel lateral lista todos diagnostics do workspace (filtravel por severidade)
- [ ] AC-004: Code actions via command palette `> code actions` (quick-fixes, refactors)
- [ ] AC-005: Go-to-definition: Ctrl+Click ou `> goto definition`
- [ ] AC-006: Completion/hover: fase 2 (Could)

**DoR:** ADR-006 (tower-lsp), RF-011  
**Estimativa:** 13 SP

---

#### US-016: File Explorer (Git Status + CRUD)
**Como** Usuário Primário  
**Quero** navegar arquivos (Ctrl+E), ver git status (M/A/D/??), criar/renomear/deletar/mover  
**Para** gerenciar projeto sem sair do TUI  

**Critérios de Aceite:**
- [ ] AC-001: Tree view expansível/colapsável, fuzzy filter `/`
- [ ] AC-002: Git status icons: M (modified), A (added), D (deleted), ?? (untracked)
- [ ] AC-003: F2 renomeia, Del deleta, Ctrl+N cria, Ctrl+M move
- [ ] AC-004: Enter abre no editor widget; `g l` revela no git log
- [ ] AC-005: Integra com DiffView (US-014) e GitWidget (US-018)

**DoR:** US-013 Done, RF-012  
**Estimativa:** 5 SP

---

#### US-017: Git Widget (Status, Log, Stage, Commit, Push)
**Como** Usuário Primário  
**Quero** git status/log/stage/commit/push/branch/stash tudo no TUI  
**Para** fluxo git completo sem terminal separado  

**Critérios de Aceite:**
- [ ] AC-001: Status: staged/unstaged/untracked com cores + diff preview (Tab)
- [ ] AC-002: Log: graph (--oneline --graph), autor, data, mensagem, navegação
- [ ] AC-003: Stage: file (s), hunk (Shift+S), linha (Space) — integra DiffView
- [ ] AC-004: Commit: editor com template Conventional Commits + validação
- [ ] AC-005: Push/Pull: fetch + rebase/merge conforme config
- [ ] AC-006: Branch: list local/remote, checkout (c), create (C), delete (D)
- [ ] AC-007: Stash: list, pop, drop, apply

**DoR:** US-014 Done, ADR-006 (git2), RF-013  
**Estimativa:** 13 SP

---

### EPIC-005: Profile & Sync

#### US-018: Dotfiles Git Clone/Pull/Push (`ferro --sync-profile`)
**Como** Usuário Primário  
**Quero** `ferro --sync-profile pull` clonar/pullar dotfiles repo e sincronizar config  
**Para** onboarding em máquina nova em segundos  

**Critérios de Aceite:**
- [ ] AC-001: Primeira execução: `git clone <url> ~/.config/ferro`
- [ ] AC-002: Pull: fetch + rebase local; push: commit + push alterações locais
- [ ] AC-003: Status: mostra ahead/behind vs origin
- [ ] AC-004: Hot reload config automático após pull (US-005)
- [ ] AC-005: Log `.ferro-sync.log` registra operação

**DoR:** ADR-004, ADR-005, RF-006, RF-020  
**Estimativa:** 5 SP

---

#### US-019: Auto Pull Idle > 2h
**Como** Sistema  
**Quero** antes de primeira task após >2h idle, fazer `git pull --rebase` automático  
**Para** manter base atualizada sem intervenção manual  

**Critérios de Aceite:**
- [ ] AC-001: Verifica `git log -1 --format=%ct` vs `now`; se delta > 7200s → pull
- [ ] AC-002: Stash automático se uncommitted changes + `stash_before_pull = true`
- [ ] AC-003: Rebase (config `rebase_instead_of_merge = true`)
- [ ] AC-004: Restore stash pós-pull
- [ ] AC-005: Log sync com `action: AutoPullRebase`, `behind`, `ahead`, `result`

**DoR:** US-018 Done, RN-008, ADR-004  
**Estimativa:** 3 SP

---

#### US-020: Anti-Colisão Branch Derivação
**Como** Sistema  
**Quero** se dois PCs cometem na mesma branch, derivar branch `<tipo>-<original>-<hostname>-<timestamp>`  
**Para** evitar force push e perda de histórico  

**Critérios de Aceite:**
- [ ] AC-001: Detecta `BEHIND > 0 && AHEAD > 0` vs origin
- [ ] AC-002: Deriva branch: `DEV-sync-pc-b-20261028-143000`
- [ ] AC-003: Checkout automático na branch derivada
- [ ] AC-004: Notificação TUI: "⚠️ Colisão detectada. Branch: DEV-sync-..."
- [ ] AC-005: Branch original intacta; merge manual via PR posterior

**DoR:** US-018 Done, RN-007, ADR-004  
**Estimativa:** 5 SP

---

### EPIC-006: Persistence & MCP

#### US-021: SQLite Sessions + FTS5 + JSONL Export
**Como** Usuário Primário  
**Quero** sessões persistidas, busca full-text, export/import JSONL  
**Para** histórico recuperável e portável  

**Critérios de Aceite:**
- [ ] AC-001: `sqlx` + SQLite WAL mode + FTS5 (porter unicode61)
- [ ] AC-002: Tabelas: sessions, messages, traces + triggers FTS sync
- [ ] AC-003: Busca: `ferro search "planmaster"` → resultados rankeados < 50ms
- [ ] AC-004: Export: `ferro export --session <id> --format jsonl` → arquivo válido
- [ ] AC-005: Import: `ferro import <file.jsonl>` → restaura sessão
- [ ] AC-006: Backup hourly automático + retenção 10 backups

**DoR:** ADR-008 aprovado, RF-014  
**Estimativa:** 8 SP

---

#### US-022: MCP Client (JSON-RPC 2.0 Nativo)
**Como** Agente  
**Quero** chamar `mcp.call_tool("git", "log", {"limit": 10})` e receber resultado tipado  
**Para** integrar ferramentas externas sem subprocesso  

**Critérios de Aceite:**
- [ ] AC-001: JSON-RPC 2.0 over stdio/TCP nativo (sem CLI wrapper)
- [ ] AC-002: Discovery servers via `profile.toml [mcp.servers]`
- [ ] AC-003: Tool calling: `call_tool(server, tool, args)` → `Result<Value>`
- [ ] AC-004: Resources: `read_resource(uri)` + subscriptions
- [ ] AC-005: Integração: agentes usam MCP para git, fs, db, etc.

**DoR:** ADR-002 (gateway pattern), RF-015  
**Estimativa:** 8 SP

---

### EPIC-007: Plugin System

#### US-023: Plugin dylib Nativo (libloading) + Hot Reload
**Como** Contribuidor/Time Confiável  
**Quero** compilar agente como `cdylib`, colocar em `~/.config/ferro/plugins/`, hot reload via command palette  
**Para** estender FERRO sem recompilar core  

**Critérios de Aceite:**
- [ ] AC-001: `cargo build --features dylib` compila PluginManager + libloading
- [ ] AC-002: Plugin implementa `Plugin` trait + `as_agent()` → `Agent`
- [ ] AC-003: `ferro plugin load <path>` carrega `.so`/`.dll`/`.dylib`
- [ ] AC-004: `ferro plugin reload <id>` hot reload (unload + load)
- [ ] AC-005: Apenas plugins assinados (`signature_required = true`)

**DoR:** ADR-001, ADR-003, RF-016  
**Estimativa:** 8 SP

---

#### US-024: Plugin WASM Sandbox (wasmtime) — v1.1
**Como** Contribuidor Externo  
**Quero** compilar agente para WASM (WASI), declarar capabilities, rodar sandboxed  
**Para** execução segura sem confiança  

**Critérios de Aceite:**
- [ ] AC-001: `cargo build --features wasm` compila wasmtime runtime
- [ ] AC-002: Plugin declara capabilities no WIT: fs-read, fs-write, net-connect, env-get, mcp-call, llm-chat
- [ ] AC-003: Validação estática imports vs capabilities declaradas
- [ ] AC-004: Runtime: fuel limit (instruções), memory limit, timeout
- [ ] AC-005: Violação capability = trap controlado + erro claro

**DoR:** ADR-001, ADR-003, RF-016 (Could)  
**Estimativa:** 13 SP (v1.1)

---

### EPIC-008: Security & Polish

#### US-025: Secret Redaction (Logs, Traces, Prompts, Diffs)
**Como** Sistema  
**Quero** qualquer secret (API keys, tokens, passwords) redacted automaticamente  
**Para** nunca vazar credenciais  

**Critérios de Aceite:**
- [ ] AC-001: Patterns: `sk-...`, `ghp_...`, `gho_...`, `xoxb-...`, `api_key = "..."`
- [ ] AC-002: Redaction em: tracing logs, trace.json, LLM prompts (pré-process), diff view, file preview
- [ ] AC-003: Flag `--no-redact` apenas para debug local (opt-in)
- [ ] AC-004: Testes com corpus secrets conhecidos (gitleaks patterns)
- [ ] AC-005: `kalimaster-sast` valida redaction no CI

**DoR:** RF-017, RN-004, EngAI Global  
**Estimativa:** 5 SP

---

#### US-026: Kalimaster-SAST Integrado (Build + On-Demand)
**Como** Usuário Primário  
**Quero** `ferro audit --sast` rodar cargo-audit + clippy + gitleaks + dependency check + gerar laudo  
**Para** auditoria de segurança integrada no workflow  

**Critérios de Aceite:**
- [ ] AC-001: Pre-commit hook: `cargo clippy -- -D warnings` + `cargo audit` + `gitleaks detect`
- [ ] AC-002: `ferro audit --sast` roda tudo + laudo `docs/07-security/LAUDO.md`
- [ ] AC-003: Laudo: severidade, evidência (arquivo:linha), correção sugerida
- [ ] AC-004: TUI mostra findings com navegação para código (Ctrl+Click)
- [ ] AC-005: CI falha se vulnerabilidades críticas/altas

**DoR:** RF-017, skill `kalimaster-sast`  
**Estimativa:** 5 SP

---

#### US-027: Acessibilidade (axe-core 0 violações AA)
**Como** Usuário com deficiência  
**Quero** TUI navegável por teclado, screen reader friendly, high contrast, reduced motion  
**Para** usar FERRO sem barreiras  

**Critérios de Aceite:**
- [ ] AC-001: 100% operações via teclado (zero mouse required)
- [ ] AC-002: Screen reader events (ratatui+crossterm acessibilidade)
- [ ] AC-003: High contrast themes (WCAG AA) — `theme: high-contrast`
- [ ] AC-004: `prefers-reduced-motion` respeitado (desabilita tachyonfx)
- [ ] AC-005: CI: `axe-core` scan em TUI renderizado = 0 violações

**DoR:** RNF-004, RF-001  
**Estimativa:** 5 SP

---

#### US-028: CI/CD + Release Automation
**Como** DevOps  
**Quero** GitHub Actions matrix build (6 targets), test, clippy, fmt, audit, release binary  
**Para** entrega contínua confiável  

**Critérios de Aceite:**
- [ ] AC-001: Matrix: Linux x86_64/aarch64, macOS x86_64/arm64, Windows x86_64
- [ ] AC-002: Jobs: build, test, clippy, fmt, cargo-audit, cargo-deny, bench
- [ ] AC-003: Release: tag `v*` → build artifacts → GitHub Release + `cargo publish` (opcional)
- [ ] AC-004: Binary size < 15MB (strip + LTO + panic=abort)
- [ ] AC-005: Changelog automático (conventional commits)

**DoR:** RNF-006, RNF-007, opsmaster-cloud skill  
**Estimativa:** 8 SP

---

#### US-029: Documentação + Onboarding
**Como** Novo Usuário  
**Quero** `ferro --help` completo, wizard primeiro run, docs arquitetura/decisões  
**Para** aprender e usar sem ajuda externa  

**Critérios de Aceite:**
- [ ] AC-001: `ferro --help` mostra todos subcomandos + exemplos
- [ ] AC-002: Primeiro run: wizard configura dotfiles repo + theme + agents
- [ ] AC-003: `docs/` completo: SRS, UML, ADRs, backlog, runbook
- [ ] AC-004: `cargo doc --no-deps` gera docs API publicadas
- [ ] AC-005: README com quickstart, arquitetura, contribuição

**DoR:** Todas US Done  
**Estimativa:** 5 SP

---

## 4. Sprint Planning

### Sprint 1 (Semanas 1-2) — **Goal: Core TUI Foundation**
**Capacidade:** 30 SP  
**Stories Comprometidas:**
- US-001: Inicialização TUI + Dashboard (8 SP)
- US-002: Layout Splits + Vim Keys (5 SP)
- US-003: Command Palette Fuzzy (5 SP)
- US-004: Theming + Hot Reload (3 SP)
- US-005: Config Hot Reload (5 SP)
- **Tech Debt/Setup:** Cargo workspace init, CI básico, clippy/fmt gates (4 SP)

**Riscos Sprint 1:**
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| ratatui 0.30 API changes | Média | Alto | Pin versão exata; PoC semana 1 |
| tachyonfx performance | Baixa | Médio | Testar em terminais lentos (WSL, SSH) |
| Config hot reload race | Média | Médio | RwLock + debounce 50ms; testar concorrência |

---

### Sprint 2 (Semanas 3-4) — **Goal: Agent System + LLM Gateway**
**Capacidade:** 30 SP  
**Stories Comprometidas:**
- US-006: Agent Trait + Registry (8 SP)
- US-007: 3 agentes core (planmaster, backdev, engai) (9 SP)
- US-009: CliWrapperGateway + Fallback Chain (8 SP)
- US-010: Streaming JSONL + TUI Render (5 SP)

**Riscos Sprint 2:**
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| CLI wrapper flakiness | Média | Alto | Health check + timeout + retry + circuit breaker |
| opencode JSONL parsing | Média | Médio | Parser tolerante; testar com output real |
| Agent trait overhead | Baixa | Baixo | Benchmark `async_trait` vs manual `BoxFuture` |

---

### Sprint 3 (Semanas 5-6) — **Goal: Remaining MVP Agents + Profile Sync**
**Capacidade:** 30 SP  
**Stories Comprometidas:**
- US-007: 4 agentes restantes (frontdev, kalimaster, opsmaster, qamaster) (12 SP)
- US-008: Auto-seleção agente (5 SP)
- US-011: Cache Semântico (5 SP)
- US-018: Dotfiles Sync (5 SP)
- US-012: Trace.json Collection (3 SP)

**Riscos Sprint 3:**
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| 7 agents = compile time | Média | Médio | Cargo features; sccache CI |
| Dotfiles sync conflicts | Média | Alto | ADR-004 anti-colisão implementado |
| Embeddings candle performance | Baixa | Médio | Cache embeddings; quantização INT8 |

---

### Sprint 4 (Semanas 7-8) — **Goal: Rich Widgets (Editor, Diff, LSP, Git, Files)**
**Capacidade:** 30 SP  
**Stories Comprometidas:**
- US-013: Syntax Highlight (8 SP)
- US-014: Diff Inline + Staging (8 SP)
- US-015: LSP Diagnostics (13 SP) — *pode spill para Sprint 5*
- US-016: File Explorer (5 SP)

**Riscos Sprint 4:**
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| LSP client complexity | Alta | Médio | MVP: só rust-analyzer + diagnostics; completion depois |
| tree-sitter grammars size | Média | Médio | Features opcionais; download on-demand |
| git2-rs segfaults | Baixa | Alto | `vendored-libgit2`; fallback `git` CLI |

---

### Sprint 5 (Semanas 9-10) — **Goal: Git Widget + Persistence + MCP + Plugin dylib**
**Capacidade:** 30 SP  
**Stories Comprometidas:**
- US-017: Git Widget (13 SP)
- US-019: Auto Pull Idle (3 SP)
- US-020: Anti-Colisão Branch (5 SP)
- US-021: SQLite Sessions + FTS5 + JSONL (8 SP)
- US-022: MCP Client (8 SP) — *pode spill*
- US-023: Plugin dylib (8 SP) — *pode spill para Sprint 6*

**Riscos Sprint 5:**
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| Scope creep widgets | Alta | Médio | Priorizar Git Widget (core); MCP/Plugin Could |
| SQLite WAL concurrency | Baixa | Médio | Testes stress concorrentes |
| libloading hot reload | Média | Médio | Testar unload/load cycle; memory leaks |

---

### Sprint 6 (Semanas 11-12) — **Goal: Security + Polish + Release + Docs**
**Capacidade:** 30 SP  
**Stories Comprometidas:**
- US-015: LSP Completion/Hover (se spill) (5 SP)
- US-022: MCP Client (se spill) (5 SP)
- US-023: Plugin dylib (se spill) (5 SP)
- US-024: Plugin WASM (v1.1 — opcional) (0 SP MVP)
- US-025: Secret Redaction (5 SP)
- US-026: Kalimaster-SAST Integrado (5 SP)
- US-027: Acessibilidade (5 SP)
- US-028: CI/CD Release (8 SP)
- US-029: Docs + Onboarding (5 SP)

**Riscos Sprint 6:**
| Risco | Prob. | Impacto | Mitigação |
|-------|-------|---------|-----------|
| Binary size > 15MB | Média | Médio | `strip`, `lto`, `panic=abort`, features seletivas |
| Acessibilidade gaps | Média | Baixo | Testar com NVDA/Orca; axe-core CI |
| Release automation bugs | Baixa | Alto | Testar release em branch `release/*` antes de main |

---

## 5. Roadmap

| Release | Data Alvo | Épicos/Features Principais | Marco |
|---------|-----------|---------------------------|-------|
| **v0.1 (MVP)** | Sprint 4 (Semana 8) | EPIC-001, 002 (3 agents), 003, 004 (syntax, diff), 005 | `ferro` roda: chat + 3 agents + TUI Obra básico |
| **v0.2 (Beta)** | Sprint 5 (Semana 10) | EPIC-002 (7 agents), 004 (LSP, git, files), 005 (sync), 006 (persistence) | 7 agents + git UI + sync multi-PC + history |
| **v0.3 (RC)** | Sprint 6 (Semana 12) | EPIC-006 (MCP), 007 (plugin dylib), 008 (security, a11y, CI) | Plugin system + security + CI/CD |
| **v1.0 (GA)** | Sprint 6 + 2 semanas | Hardening, bug fixes, performance, docs completas | Production ready: `cargo install --locked ferro` |

---

## 6. Riscos de Sprint (Consolidado)

| Sprint | Risco Principal | Prob. | Impacto | Mitigação |
|--------|-----------------|-------|---------|-----------|
| 1 | ratatui ecosystem integration | Média | Alto | PoC semana 1; travar versões |
| 2 | CLI wrapper reliability | Média | Alto | Health checks, fallback chain, timeouts |
| 3 | Multi-agent compilation | Média | Médio | Features seletivas, sccache |
| 4 | LSP + tree-sitter complexity | Alta | Médio | Fasear: diagnostics only MVP |
| 5 | Git + SQLite + MCP concurrency | Baixa | Alto | Testes stress; WAL mode |
| 6 | Binary size + release | Média | Médio | Otimizações build; release dry-run |

---

## 7. Métricas

| Métrica | Target | Ferramenta | Frequência |
|---------|--------|------------|------------|
| **Velocity** | 25-30 SP/sprint | GitHub Projects | Sprint |
| **Bug Escape Rate** | < 5% | Issues (bug label) | Sprint |
| **Lead Time** | < 3 dias | Cycle time (PR opened → merged) | Contínuo |
| **Deploy Frequency** | 1/sprint (staging) | GitHub Actions | Sprint |
| **Cold Start** | < 2s | `hyperfine` | CI nightly |
| **Memory Idle** | < 50MB RSS | `ps_record` | CI nightly |
| **Test Coverage** | > 80% unit, 100% critical | `cargo llvm-cov` | PR |
| **Accessibility** | 0 axe-core violations | CI axe-core | PR |
| **Binary Size** | < 15MB | `cargo bloat` | Release |
| **CLI Wrapper Overhead** | < 100ms | `trace.json` | CI integration |

---

## 8. DoR / DoD Globais

### Definition of Ready (Global) — Story pronta para Sprint Planning
- [ ] Requisito claro e testável (RF/SRS vinculado)
- [ ] ACs em Gherkin (Given/When/Then)
- [ ] Dependências identificadas (outras stories, ADRs, libs)
- [ ] UX/UI definido (se envolve TUI) — wireframe ou descrição
- [ ] Estimado em Story Points pelo time (Planning Poker)
- [ ] Disponível no backlog priorizado (GitHub Project)
- [ ] ADR relacionado aprovado (se decisão arquitetural)

### Definition of Done (Global) — Story considerada "Done"
- [ ] Código implementado + revisado (PR aprovado por 1+ reviewer)
- [ ] Testes unitários > 80% cobertura (linhas) — `cargo llvm-cov`
- [ ] Testes integração passando (critical paths: sync, gateway, agent registry)
- [ ] `cargo clippy -- -D warnings` clean
- [ ] `cargo fmt --check` pass
- [ ] `kalimaster-sast` limpo (cargo-audit, clippy, gitleaks)
- [ ] Documentação atualizada: ADR (se decisão nova), README, CHANGELOG, doc comments
- [ ] Deploy staging validado (GitHub Actions matrix build + test pass)
- [ ] Acessibilidade: axe-core 0 violações AA (CI)
- [ ] Performance dentro do target (benchmarks CI não regrediram > 10%)
- [ ] Commit em `develop` com Conventional Commits (`feat(agent): ...`)
- [ ] Resumo executivo no PR: o que foi feito, métricas, riscos, follow-ups

---

## 9. Backlog Priorizado (Resumo para GitHub Issues)

| Prioridade | Issue ID | Título | Épico | SP | Sprint |
|------------|----------|--------|-------|----|--------|
| Must | #1 | Cargo workspace init + CI básico | EPIC-001 | 4 | 1 |
| Must | #2 | TUI bootstrap: ratatui + crossterm + layout | EPIC-001 | 8 | 1 |
| Must | #3 | Splits redimensionáveis + vim keys | EPIC-001 | 5 | 1 |
| Must | #4 | Command palette (skim) | EPIC-001 | 5 | 1 |
| Must | #5 | Theming (opaline) + tachyonfx | EPIC-001 | 3 | 1 |
| Must | #6 | Config hot reload (figment + notify) | EPIC-001 | 5 | 1 |
| Must | #7 | Agent trait + registry + Cargo features | EPIC-002 | 8 | 2 |
| Must | #8 | Planmaster agent (SRS/UML/backlog) | EPIC-002 | 3 | 2 |
| Must | #9 | Backdev agent (impl + test + commit) | EPIC-002 | 3 | 2 |
| Must | #10 | Engai agent (RAG/MCP/gateway) | EPIC-002 | 3 | 2 |
| Must | #11 | CliWrapperGateway (opencode primary) | EPIC-003 | 8 | 2 |
| Must | #12 | Fallback chain (claude/gh/ollama) | EPIC-003 | 3 | 2 |
| Must | #13 | Streaming JSONL parser + TUI render | EPIC-003 | 5 | 2 |
| Must | #14 | Frontdev agent (React/TS/Tailwind) | EPIC-002 | 3 | 3 |
| Must | #15 | Kalimaster agent (SAST/DAST) | EPIC-002 | 3 | 3 |
| Must | #16 | Opsmaster agent (Docker/CI/cloud) | EPIC-002 | 3 | 3 |
| Must | #17 | Qamaster agent (test plan/automation) | EPIC-002 | 3 | 3 |
| Must | #18 | Auto-seleção agente (embeddings) | EPIC-002 | 5 | 3 |
| Must | #19 | Cache semântico (candle + SQLite) | EPIC-003 | 5 | 3 |
| Must | #20 | Trace.json collection (FinOps) | EPIC-003 | 3 | 3 |
| Must | #21 | Dotfiles sync (clone/pull/push) | EPIC-005 | 5 | 3 |
| Must | #22 | Syntax highlight (tree-sitter + syntect) | EPIC-004 | 8 | 4 |
| Must | #23 | Diff inline (diffrs + staging linha) | EPIC-004 | 8 | 4 |
| Must | #24 | LSP diagnostics inline (tower-lsp) | EPIC-004 | 13 | 4 |
| Must | #25 | File explorer (git status + CRUD) | EPIC-004 | 5 | 4 |
| Must | #26 | Auto pull idle > 2h | EPIC-005 | 3 | 5 |
| Must | #27 | Anti-colisão branch derivação | EPIC-005 | 5 | 5 |
| Must | #28 | Git widget (status/log/stage/commit/push) | EPIC-004 | 13 | 5 |
| Must | #29 | SQLite sessions + FTS5 + JSONL | EPIC-006 | 8 | 5 |
| Must | #30 | MCP client (JSON-RPC 2.0) | EPIC-006 | 8 | 5 |
| Should | #31 | Plugin dylib (libloading + hot reload) | EPIC-007 | 8 | 5-6 |
| Must | #32 | Secret redaction (logs/traces/prompts) | EPIC-008 | 5 | 6 |
| Must | #33 | Kalimaster-SAST integrado (pre-commit + audit) | EPIC-008 | 5 | 6 |
| Must | #34 | Acessibilidade (axe-core + high contrast) | EPIC-008 | 5 | 6 |
| Must | #35 | CI/CD matrix + release automation | EPIC-008 | 8 | 6 |
| Must | #36 | Docs + onboarding wizard | EPIC-008 | 5 | 6 |
| Could | #37 | Plugin WASM (wasmtime sandbox) | EPIC-007 | 13 | v1.1 |
| Could | #38 | LSP completion/hover | EPIC-004 | 8 | v1.1 |
| Could | #39 | 12 agentes restantes | EPIC-002 | 36 | v1.1 |

---

> **Próximo passo:** FASE 6 — Plano de Entrega (`docs/06-plano-de-entrega.md`): estrutura Cargo workspace detalhada, comandos setup, estratégia branches, CI/CD, runbook, checklist anti-retrabalho.