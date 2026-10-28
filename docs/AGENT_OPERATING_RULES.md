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

---

## 10. Sincronização Multi-PC / Anti-Colisão (Obrigatório)

### 10.1 Auto Pull em Idle (> 2h)
**Regra:** Antes da **primeira requisição** após período de inatividade ≥ 2 horas no mesmo workspace, o agente **deve executar automaticamente**:
```bash
git fetch origin
git status -sb
# Se houver commits remotos na branch atual:
git pull --rebase origin <branch-atual>
# Se houver divergência (local commits não empurrados):
git stash push -m "auto-stash pre-pull $(date -Iseconds)" && git pull --rebase origin <branch-atual> && git stash pop
```

**Implementação:** Verificar `git log -1 --format=%ct` vs `date +%s`; se delta > 7200s → executar pull.

### 10.2 Detecção e Desvio de Colisão de Branch
**Regra:** Antes de iniciar trabalho em branch compartilhada (`develop`, `ISSUE-*`, `DEV-*`, `SEG-*`, `DOC-*`), o agente **deve verificar**:
```bash
# 1. Verificar se branch existe remoto e tem commits novos
git fetch origin <branch>
git rev-list --left-right --count HEAD...origin/<branch>

# 2. Se origin/<branch> tem commits que não temos (outro PC empurrou):
#    → Criar branch de trabalho derivada: <tipo>-<original>-<hostname>-<timestamp>
#    Ex: DEV-1234-pc-casa-20261028-143000
#    → Fazer checkout na nova branch
#    → Continuar trabalho normalmente

# 3. Se temos commits locais não empurrados E origin tem commits novos:
#    → Stash local → pull rebase → pop stash → continuar
#    → Se conflito: abrir issue de merge manual (question tool)
```

**Nomenclatura de branch derivada (anti-colisão):**
| Tipo Original | Derivada (exemplo) |
|---------------|---------------------|
| `develop` | `DEV-sync-<hostname>-<YYYYMMDD-HHMMSS>` |
| `ISSUE-42` | `ISSUE-42-<hostname>-<YYYYMMDD-HHMMSS>` |
| `DEV-auth` | `DEV-auth-<hostname>-<YYYYMMDD-HHMMSS>` |
| `SEG-cve` | `SEG-cve-<hostname>-<YYYYMMDD-HHMMSS>` |
| `DOC-srs` | `DOC-srs-<hostname>-<YYYYMMDD-HHMMSS>` |

**Hostname:** Usar `hostname -s` (short) ou variável `FERRO_HOST_ID` configurada no profile.toml.

### 10.3 Checklist Obrigatório Início de Sessão
Todo agente, ao receber **primeira task** da sessão, deve executar e registrar:

```bash
# 1. Verificar idle time
LAST_COMMIT=$(git log -1 --format=%ct 2>/dev/null || echo 0)
NOW=$(date +%s)
IDLE=$((NOW - LAST_COMMIT))

# 2. Se IDLE > 7200 (2h): auto pull
if [ $IDLE -gt 7200 ]; then
  git fetch origin
  git pull --rebase origin $(git branch --show-current) 2>/dev/null || true
fi

# 3. Verificar colisão na branch atual
BRANCH=$(git branch --show-current)
git fetch origin $BRANCH 2>/dev/null
BEHIND=$(git rev-list --count HEAD..origin/$BRANCH 2>/dev/null || echo 0)
AHEAD=$(git rev-list --count origin/$BRANCH..HEAD 2>/dev/null || echo 0)

if [ $BEHIND -gt 0 ] && [ $AHEAD -eq 0 ]; then
  # Só remoto tem novidades: pull rebase direto
  git pull --rebase origin $BRANCH
elif [ $BEHIND -gt 0 ] && [ $AHEAD -gt 0 ]; then
  # Ambos têm commits: derivar branch anti-colisão
  NEW_BRANCH="${BRANCH}-$(hostname -s)-$(date +%Y%m%d-%H%M%S)"
  git checkout -b "$NEW_BRANCH"
  echo "⚠️ Colisão detectada: mudado para branch derivada $NEW_BRANCH"
fi

# 4. Log da sincronização
echo "[$(date -Iseconds)] Sync: branch=$BRANCH, idle=${IDLE}s, behind=$BEHIND, ahead=$AHEAD" >> .ferro-sync.log
```

### 10.4 Configuração no Profile (dotfiles)
Adicionar ao `profile.toml` (sync via `ferro --sync-profile`):

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

> **Nota:** Estas regras valem para **todos os agentes** (planmaster, backdev, frontdev, engai, kalimaster, opsmaster, qamaster, etc.) e devem ser injetadas no prompt de despacho via `task`.