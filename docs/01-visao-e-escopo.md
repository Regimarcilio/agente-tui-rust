# Visão e Escopo — Agente TUI Rust Personalizado (Codename: FERRO)

**Versão:** 0.1  
**Data:** 28/10/2026  
**Autor:** PLANMASTER  
**Branch:** develop  
**Status:** Em elaboração

---

## 1. Problema

Desenvolvedores que usam múltiplos agentes especializados (planmaster, backdev, frontdev, engai, kalimaster, etc.) hoje dependem de:
- **OpenCode/CLI** como orquestrador externo → latência de subprocesso, falta de UX unificada, sem estado compartilhado nativo
- **Múltiplos terminais/tmux** para rodar agentes em paralelo → contexto fragmentado, difícil correlacionar decisões
- **Configuração dispersa** (`.opencode/`, `~/.config/`, dotfiles git) → drift entre máquinas, onboarding lento

Não existe hoje um **agente TUI nativo em Rust** que:
1. Embuta todos os 19 agentes do squad como módulos internos (zero subprocess spawn para lógica de agente)
2. Carregue perfil/diretrizes de um **repo Git de dotfiles** com hot-reload
3. Ofereça TUI nível "Obra" (diff inline, LSP, syntax highlight, git UI, command palette)
4. Use **CLI wrappers** (opencode/claude/gh copilot) apenas para chamadas LLM, mantendo controle total do loop de agente

---

## 2. Público-Alvo

| Perfil | Primário/Secundário | Necessidades |
|--------|---------------------|--------------|
| **Você (autor)** | Primário | Controle total, privacidade, extensibilidade, velocidade, integração nativa dos seus 19 agentes |
| **Colegas de squad** | Secundário | Onboarding via `git clone dotfiles && ferro`, mesmo setup em segundos |
| **Contribuidores open-source** | Terciário | Arquitetura plugin (WASM/dylib) para adicionar agentes sem recompilar core |

---

## 3. Objetivos (OKRs)

| Objetivo | Key Result | Target |
|----------|------------|--------|
| **O1: Agente TUI funcional** | MVP roda `ferro` com chat + 3 agentes embutidos | Sprint 2 |
| **O2: 19 agentes nativos** | Todos agentes do squad carregados como módulos Rust | Sprint 5 |
| **O3: Perfil via dotfiles git** | `ferro --sync-profile` pull/push dotfiles repo | Sprint 3 |
| **O4: TUI nível Obra** | Diff inline, LSP, syntax highlight, git staging | Sprint 4 |
| **O5: Gateway LLM via CLI wrapper** | `ferro chat` usa `opencode`/`claude`/`gh copilot` transparente | Sprint 2 |
| **O6: Distribuível single-binary** | `cargo install --locked ferro` < 15MB, < 2s cold start | Sprint 6 |

---

## 4. Escopo

### 4.1 IN (MVP → v1.0)

| Área | Itens |
|------|-------|
| **Core** | Event loop async (tokio), config hot-reload, plugin system (WASM + dylib), logging estruturado (tracing) |
| **Agentes Nativos** | 19 agentes como módulos Rust (`agents/planmaster`, `agents/backdev`, ...), trait `Agent` unificado, registry dinâmico |
| **Perfil/Dotfiles** | Git2-rs para clone/pull/push, watchman/notify para hot-reload, schema TOML validado (serde) |
| **TUI (ratatui + crossterm)** | Layout: splits redimensionáveis, tabs, vim keybindings, command palette (fuzzy), themes (base16), mouse support |
| **Widgets Ricos** | Syntax highlight (syntect/tree-sitter), diff inline (diffrs), LSP client (lsp-types + tower-lsp), git UI (git2), file explorer |
| **LLM Gateway** | Subprocess manager para `opencode`/`claude`/`gh copilot`/`ollama`, streaming JSONL, cache semântico local (sqlite + embeddings), cost/latency tracking |
| **Session/History** | Persistência SQLite (sessões, mensagens, tool calls, traces), export/import JSONL, search full-text |
| **MCP Integration** | Cliente MCP nativo (JSON-RPC 2.0 over stdio/TCP), discovery de servers, tool calling unificado |
| **Segurança** | Sandbox WASM (wasmtime), capability-based permissions, secret redaction (kalimaster-sast integrado), audit log |

### 4.2 OUT (Pós-v1.0 / Backlog)

- GUI complementar (egui/tauri) — foco permanece TUI
- Fine-tuning/LoRA local — fora de escopo (regra EngAI: RAG > fine-tuning)
- Cloud sync de sessões — opcional, via plugin
- Marketplace de agentes — fase 2
- Colaboração multiplayer (CRDT) — fase 3

---

## 5. Restrições

| Tipo | Detalhe |
|------|---------|
| **Técnica** | Rust 2024 edition (MSRV: 1.80+), single binary, zero-config default, cross-platform (Linux/macOS/Windows) |
| **Performance** | Cold start < 2s, RAM idle < 50MB, 60fps TUI, LLM streaming latency < 100ms overhead |
| **Legal** | LGPD/GDPR by design (dados locais, opt-in telemetry), licenças deps compatíveis (MIT/Apache-2.0) |
| **Acessibilidade** | Screen reader friendly (ratatui + crossterm), high contrast themes, reduced motion, keyboard-only |
| **Orçamento** | Zero custo infra (local-first), deps open-source apenas |
| **Prazo** | MVP em 6 sprints (2 semanas cada) = ~12 semanas |

---

## 6. Critérios de Sucesso

| Métrica | Target | Medição |
|---------|--------|---------|
| **Time-to-first-agent** | < 5s | `ferro --agent planmaster` pronto |
| **Cold start** | < 2s | `hyperfine 'ferro --version'` |
| **Memory baseline** | < 50MB RSS | `ps` / `memusage` |
| **Agent switch latency** | < 50ms | TUI frame timing |
| **LLM streaming throughput** | > 50 tokens/s | benchmark vs CLI direto |
| **Config sync reliability** | 100% | git push/pull roundtrip |
| **Test coverage** | > 80% (unit) + 100% (integration critical paths) | `cargo llvm-cov` |
| **Accessibility score** | 0 axe-core violations | CI check |

---

## 7. Stakeholders & Personas

| Stakeholder | Papel | Contato |
|-------------|-------|---------|
| **Você** | Product Owner + Tech Lead + Usuário Primário | — |
| **Aprendiz** | Guardião do conhecimento (consulta pré-impl) | `aprendiz` agent |
| **Kalimaster** | Auditoria de segurança (SAST/DAST) | `kalimaster-sast` skill |
| **Opsmaster** | CI/CD, release, observabilidade | `opsmaster-cloud` skill |

**Persona Principal — "Arquiteto Rustacean"**
- Desenvolve em Rust diariamente
- Usa Neovim + tmux + zsh
- Valoriza: velocidade, controle, privacidade, extensibilidade
- Dor: context switching entre terminais, config drift, agentes como black boxes

---

## 8. Jornadas Principais

### J1: Primeira Execução (Onboarding)
```
git clone ~/dotfiles ~/.config/ferro
ferro --sync-profile        # puxa profile.toml + agents/ do dotfiles
ferro                       # abre TUI, carrega 19 agentes, mostra dashboard
```

### J2: Coding com Agente
```
ferro chat "planeja feature X"
→ Seleciona planmaster (auto ou manual)
→ planmaster gera SRS + UML + backlog em docs/
→ User aprova → ferro spawn backdev para implementar
→ backdev escreve código, roda testes, commita em develop
→ ferro mostra diff inline, user aprova PR
```

### J3: Auditoria de Segurança
```
ferro audit --sast --dast
→ kalimaster roda SAST no código atual
→ kalimaster roda DAST no staging local (docker compose up)
→ Gera laudo em docs/07-security/LAUDO.md
→ TUI mostra findings com navegação para código
```

### J4: Sincronização Multi-Máquina
```
# Machine A
ferro --sync-profile push   # commit + push dotfiles

# Machine B
ferro --sync-profile pull   # pull + hot-reload config
→ Mesmos agentes, atalhos, themes, histórico
```

---

## 9. Riscos e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| **Complexidade TUI "Obra"** | Alta | Alto | Fasear: MVP → Essencial → Intermediário → Obra; usar ratatui widgets compostos |
| **WASM sandbox instável** | Média | Alto | Começar com dylib nativo (cdylib); WASM como opção v1.1; testar wasmtime 20+ |
| **CLI wrapper flakiness** | Média | Médio | Abstrair `LlmGateway` trait; fallback chain (opencode → claude → gh copilot → ollama); timeout + retry |
| **Git2-rs segfaults** | Baixa | Alto | Pin git2 versão estável; fallback para `git` CLI via subprocess para operações complexas |
| **LSP client complexity** | Alta | Médio | Integrar `tower-lsp` + `lsp-types`; começar com rust-analyzer apenas; language servers como plugins |
| **19 agentes = code bloat** | Média | Médio | Cargo features por agente (`--features planmaster,backdev`); dynamic loading preferido |

---

## 10. Glossário

| Termo | Definição |
|-------|-----------|
| **FERRO** | Codename do agente TUI (Ferro = Iron/Fe, base do aço/Rust) |
| **Agent Trait** | Interface Rust unificada: `fn handle(&self, ctx: AgentCtx, input: Input) -> Stream<Output>` |
| **Dotfiles Repo** | Git repo com `profile.toml`, `agents/`, `themes/`, `keybindings/`, `mcp/` |
| **Semantic Cache** | SQLite + embeddings (local) para deduplicar chamadas LLM idênticas |
| **MCP Client** | Implementação nativa do Model Context Protocol (JSON-RPC 2.0) |
| **Kalimaster-SAST** | Skill de auditoria estática integrada no build/CI |
| **Command Palette** | Fuzzy search (skim/fzf-like) para ações, agentes, arquivos, config |

---

## 11. Aprovação

| Role | Nome | Assinatura | Data |
|------|------|------------|------|
| **PLANMASTER** | | | |
| **Tech Lead (Você)** | | | |

---

> **Próximo passo:** FASE 2 — Benchmarks de frameworks TUI, agent frameworks Rust, CLI wrappers LLM, WASM runtimes, LSP clients.