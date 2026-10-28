# Regras de Operação dos Agentes — Projeto FERRO

**Versão:** 1.0  
**Data:** 28/10/2026  
**Aplicável a:** Todos os agentes despachados via `task` tool (planmaster, backdev, frontdev, engai, kalimaster, opsmaster, qamaster, etc.)

---

## Regra Principal: Execução Contínua Sem Interrupções Desnecessárias

### ✅ O que FAZER
- **Executar tarefas completas** sem parar para perguntar "posso continuar?" ou "próximo passo?"
- **Assumir decisões técnicas** baseadas nos docs (SRS, ADRs, benchmarks) já aprovados
- **Fazer escolhas de implementação** (nomes de arquivos, estrutura, libs) seguindo convenções do projeto
- **Comitar progresso** incrementalmente em `develop` com Conventional Commits
- **Reportar resultado final** com resumo executivo + arquivos alterados + próximos passos sugeridos

### ❌ O que NÃO FAZER
- Perguntar "posso prosseguir?" após cada passo
- Perguntar confirmação para decisões já documentadas no SRS/ADRs
- Parar execução aguardando input para escolhas triviais (nomes de variáveis, imports, formatação)
- Pedir aprovação para commits de desenvolvimento em `develop`

---

## Quando PERGUNTAR (question tool) — Apenas Decisões Críticas

Use `question` **somente** quando:
1. **Ambiguidade real no SRS/ADR** — requisito não especificado ou contraditório
2. **Trade-off arquitetural não documentado** — ex: "SQLite vs Postgres para sessions?"
3. **Mudança de escopo** — feature nova não prevista no backlog
4. **Risco de segurança/legal** — ex: "expor API key no client-side?"

### Formato Obrigatório das Perguntas
```json
{
  "questions": [{
    "question": "Descrição clara do dilema",
    "header": "Rótulo curto (≤30 chars)",
    "options": [
      {"label": "Opção A (Recomendada)", "description": "Justificativa técnica"},
      {"label": "Opção B", "description": "Justificativa técnica"},
      {"label": "Opção C", "description": "Justificativa técnica"}
    ],
    "multiple": false
  }]
}
```
- **Mínimo 2 opções**, idealmente 3-4
- **Marcar "(Recomendada)"** na melhor opção baseada nos docs
- **Nunca** incluir "Outro" ou "Decida você" — o agente deve recomendar

---

## Instrução de Injeção para Subagentes

Ao despachar qualquer agente via `task`, **sempre incluir no prompt**:

```
IMPORTANTE: Siga as regras em docs/AGENT_OPERATING_RULES.md
- Execução contínua sem perguntar "posso continuar?"
- question tool APENAS para decisões críticas com múltiplas opções
- Assuma decisões técnicas baseadas em SRS/ADRs/benchmarks aprovados
- Commit incremental em develop, reporte resultado final
```

---

## Workflow Padrão por Fase

| Fase | Agente Responsável | Entregável | Commits Esperados |
|------|-------------------|------------|-------------------|
| FASE 3 — SRS | planmaster | `docs/SRS.md` | `docs(srs): ...` |
| FASE 4 — UML/ADR | planmaster | `docs/03-arquitetura-e-uml.md`, `docs/04-ADRs/` | `docs(uml): ...`, `docs(adr): ...` |
| FASE 5 — Scrum | planmaster | `docs/05-backlog.md` | `docs(scrum): ...` |
| FASE 6 — Entrega | planmaster + opsmaster | Cargo workspace, CI, runbook | `chore(init): ...`, `ci: ...` |
| Sprint 1-6 | backdev/frontdev/engai/kalimaster/qamaster | Código + testes + docs | `feat(...): ...`, `fix(...): ...`, `test(...): ...` |

---

## Checklist Anti-Retrabalho (DoD por Entrega)

Antes de reportar "concluído", o agente deve verificar:

- [ ] Todos os ACs do SRS/Story atendidos (Gherkin passing)
- [ ] `cargo test` + `cargo clippy -- -D warnings` + `cargo fmt --check` passam
- [ ] `kalimaster-sast` limpo (rodar skill se disponível)
- [ ] Docs atualizados (README, CHANGELOG, ADRs se decisão nova)
- [ ] Commit em `develop` com mensagem Conventional Commits
- [ ] Resumo executivo: o que foi feito, arquivos, métricas, riscos, próximos passos

---

## Escalação

Se agente encontrar **bloqueio real** (dependência externa, bug em crate, decisão de produto):
1. Documentar bloqueio em `docs/BLOCKERS.md` com contexto
2. Fazer **uma** pergunta crítica via `question` com opções de workaround
3. Continuar em paralelo no que não estiver bloqueado

---

> **Princípio:** "Melhor pedir perdão que permissão" — desde que dentro do escopo documentado e com commits reversíveis em `develop`.