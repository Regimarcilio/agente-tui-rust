# Plano de Entrega — Agente TUI Rust (FERRO)

**Versão:** 1.0  
**Data:** 28/10/2026  
**Autor:** PLANMASTER  
**Branch:** develop  
**Base:** `docs/SRS.md`, `docs/03-arquitetura-e-uml.md`, `docs/04-ADRs/`, `docs/05-backlog.md`

---

## 1. Estrutura do Cargo Workspace

```
ferro/
├── Cargo.toml                    # Workspace root
├── Cargo.lock
├── .github/
│   ├── workflows/
│   │   ├── ci.yml               # CI matrix build + test + clippy + fmt + audit
│   │   ├── release.yml          # Release automation (tag v* → build → GH Release)
│   │   ├── bench.yml            # Nightly benchmarks (cold start, memory, binary size)
│   │   └── docs.yml             # Deploy docs (cargo doc → gh-pages)
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.yml
│   │   ├── feature_request.yml
│   │   └── security_report.yml
│   └── dependabot.yml
├── .vscode/
│   ├── settings.json            # rust-analyzer, clippy, format on save
│   └── tasks.json               # Cargo tasks (build, test, check, doc)
├── .gitignore
├── .rustfmt.toml
├── clippy.toml
├── deny.toml                    # cargo-deny config (licenses, bans, advisories)
├── LICENSE                      # MIT OR Apache-2.0
├── README.md
├── CHANGELOG.md
├── docs/                        # Documentação (já existe)
│   ├── 01-visao-e-escopo.md
│   ├── 02-benchmarks.md
│   ├── 03-arquitetura-e-uml.md
│   ├── 04-ADRs/
│   ├── 05-backlog.md
│   └── 06-plano-de-entrega.md   # Este arquivo
├── scripts/
│   ├── install.sh               # Install script (curl | sh)
│   ├── dev-setup.sh             # Dev environment setup
│   ├── bench.sh                 # Benchmark runner
│   └── release.sh               # Release helper
├── crates/
│   ├── ferro-core/              # Config, traits, errors, logging, utils
│   ├── ferro-tui/               # TUI runtime, widgets, layout, themes
│   ├── ferro-agents/            # AgentRegistry + 19 agent crates
│   ├── ferro-gateway/           # LlmGateway, CliWrapperGateway, cache, traces
│   ├── ferro-plugins/           # PluginManager, dylib loader, wasmtime runtime
│   ├── ferro-mcp/               # MCP client JSON-RPC 2.0
│   ├── ferro-git/               # GitService, GitSync, DiffView
│   ├── ferro-lsp/               # LSP client (tower-lsp)
│   ├── ferro-session/           # SessionStore (SQLite + FTS5 + JSONL)
│   └── ferro-cli/               # CLI entrypoint (clap), subcommands
├── plugins/                     # Example plugins (builtin)
│   ├── ferro-agent-planmaster/
│   ├── ferro-agent-backdev/
│   └── ...
└── tests/
    ├── integration/             # Integration tests (binary + fixtures)
    ├── e2e/                     # E2E tests (assert_cmd + ferro binary)
    └── fixtures/                # Test data (sample repos, configs)
```

### Cargo.toml (Workspace Root)

```toml
[workspace]
resolver = "2"
members = [
    "crates/ferro-core",
    "crates/ferro-tui",
    "crates/ferro-agents",
    "crates/ferro-gateway",
    "crates/ferro-plugins",
    "crates/ferro-mcp",
    "crates/ferro-git",
    "crates/ferro-lsp",
    "crates/ferro-session",
    "crates/ferro-cli",
]
exclude = ["plugins/*"]  # Plugins built separately

[workspace.dependencies]
# Core
tokio = { version = "1.40", features = ["full", "tracing"] }
tracing = { version = "0.1", features = ["std"] }
tracing-subscriber = { version = "0.3", features = ["env-filter", "json", "fmt"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
toml = "0.8"
figment = { version = "0.10", features = ["toml", "env", "serde"] }
anyhow = "1.0"
thiserror = "1.0"
uuid = { version = "1.8", features = ["v7", "serde", "fast-rng"] }
clap = { version = "4.5", features = ["derive", "env", "string"] }

# TUI
ratatui = { version = "0.30", features = ["crossterm", "macros"] }
crossterm = "0.28"
tachyonfx = { version = "0.6", features = ["dsl", "std"] }
opaline = "0.3"
syntect = { version = "5.0", features = ["default"] }
tree-sitter = "0.22"
ratatui-code-editor = "0.2"
skim = "0.12"
diffrs = "0.2"

# Git
git2 = { version = "0.18", features = ["https", "ssh", "vendored-libgit2"] }

# LSP
tower-lsp = "0.21"
lsp-types = "0.95"

# Database
sqlx = { version = "0.7", features = ["sqlite", "runtime-tokio", "migrate", "uuid", "chrono", "json"] }

# Embeddings / Cache
candle = { version = "0.5", features = ["cuda"] }  # Optional CUDA

# WASM
wasmtime = { version = "20", features = ["wasi", "component-model"] }
libloading = "0.8"

# MCP
serde_json = "1.0"
tokio-util = "0.7"

# Testing
mockall = { version = "0.13", features = ["nightly"] }
proptest = "1.0"
assert_cmd = "2.0"
predicates = "3.0"

[workspace.lints]
rust_2024 = true

[workspace.lints.rust]
unused_crate_dependencies = "warn"
unused_imports = "warn"
dead_code = "allow"  # Allow in agent crates for unused variants

[workspace.lints.clippy]
all = "warn"
pedantic = "warn"
nursery = "warn"
cargo = "warn"
```

---

## 2. Comandos de Setup por Stack

### 2.1 Desenvolvimento (Linux/macOS/Windows)

```bash
# 1. Instalar Rust (se não tiver)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
rustup component add rustfmt clippy

# 2. Instalar dependências sistema
# Ubuntu/Debian:
sudo apt update && sudo apt install -y libssl-dev pkg-config libclang-dev
# macOS:
brew install openssl llvm
# Windows: rustup default stable-msvc (ou gnu)

# 3. Clone e build
git clone https://github.com/Regimarcilio/agente-tui-rust.git ferro
cd ferro
git checkout develop

# 4. Build desenvolvimento (com todos agentes)
cargo build --workspace --features "all"

# 5. Testes
cargo test --workspace --features "all"
cargo clippy --workspace --features "all" -- -D warnings
cargo fmt --check

# 6. Run
cargo run --bin ferro --features "all" -- --help
cargo run --bin ferro --features "all"  # Abre TUI
```

### 2.2 Build Otimizado (Release)

```bash
# Build release com otimizações máximas
cargo build --release --workspace --features "all" \
  --config 'profile.release.strip = "debuginfo"' \
  --config 'profile.release.lto = true' \
  --config 'profile.release.panic = "abort"' \
  --config 'profile.release.codegen-units = 1'

# Binary final: target/release/ferro (~12-15MB)
```

### 2.3 Instalação Usuário Final

```bash
# Via cargo install (crates.io - quando publicado)
cargo install --locked ferro --features "all"

# Via script (GitHub Releases)
curl -fsSL https://github.com/Regimarcilio/agente-tui-rust/releases/latest/download/install.sh | sh

# Via binário direto
# Download ferro-x86_64-unknown-linux-gnu.tar.gz do GitHub Releases
tar -xzf ferro-*.tar.gz
sudo mv ferro /usr/local/bin/
```

### 2.4 Onboarding Primeiro Run

```bash
# 1. Configurar dotfiles repo (uma vez)
ferro --sync-profile init  # Wizard interativo
# Ou manual:
git clone https://github.com/seu-usuario/dotfiles-ferro ~/.config/ferro

# 2. Sincronizar profile
ferro --sync-profile pull

# 3. Iniciar
ferro  # Abre TUI com dashboard
```

---

## 3. Estratégia de Branches

### 3.1 Branch Model (Develop-First)

```
main (protegida, apenas tags v*)
  ↑
release/v0.1.0  ←── merge develop → tag v0.1.0
  ↑
develop (branch principal, CI passing obrigatório)
  ↑
  ├── ISSUE-123  (fecha issue #123, PR com "Closes #123")
  ├── DEV-auth   (feature nova: autenticação)
  ├── SEG-cve    (fix segurança)
  └── DOC-srs    (documentação)
```

### 3.2 Regras Obrigatórias (planmaster-git + AGENT_OPERATING_RULES)

| Regra | Descrição |
|-------|-----------|
| **Nunca commit direto em main** | Apenas via `release/*` branch + tag semver |
| **Develop = integration branch** | Todo trabalho merge em `develop` via PR |
| **Branch naming EXATO** | `ISSUE-*`, `DEV-*`, `SEG-*`, `DOC-*` (recriar após merge) |
| **Conventional Commits** | `type(scope): descrição` — types: `feat`, `fix`, `docs`, `plan`, `chore`, `refactor`, `test`, `ci` |
| **PR Template** | Descrição, ACs, testes, screenshots (se UI), breaking changes |
| **Review obrigatório** | 1+ aprovação + CI passing + `kalimaster-sast` limpo |
| **Merge strategy** | Squash and merge (histórico limpo) |
| **Push --force PROIBIDO** | Exceto force-push em própria branch de feature (com coordenação) |

### 3.3 Fluxo de Trabalho Padrão

```bash
# 1. Iniciar feature/fix
git checkout develop
git pull origin develop
git checkout -b DEV-minha-feature  # ou ISSUE-42, SEG-cve, DOC-readme

# 2. Desenvolver + commitar incrementalmente
git add -p  # Stage seletivo
git commit -m "feat(agent): adiciona planmaster SRS generation"

# 3. Push + PR
git push -u origin DEV-minha-feature
gh pr create --base develop --title "feat(agent): planmaster SRS generation" --body "..."
# Preencher: Descrição, ACs (link SRS), Testes, Screenshots, Breaking changes

# 4. CI roda: build matrix + test + clippy + fmt + audit + bench
# 5. Review + aprovação
# 6. Merge (squash) → develop
# 7. Limpar branch local/remota
git checkout develop && git pull && git branch -d DEV-minha-feature
```

### 3.4 Release Process

```bash
# 1. Preparar release branch
git checkout develop
git pull
git checkout -b release/v0.2.0

# 2. Atualizar versão + changelog
cargo set-version 0.2.0  # Atualiza todos Cargo.toml do workspace
# Editar CHANGELOG.md (gerado via conventional commits)
git commit -m "chore(release): v0.2.0"

# 3. Tag + push
git tag -a v0.2.0 -m "Release v0.2.0"
git push origin release/v0.2.0 --tags

# 4. GitHub Actions release.yml faz:
#    - Build matrix (6 targets)
#    - Test + clippy + fmt + audit
#    - Cargo build --release (strip + LTO)
#    - Create GitHub Release com artifacts
#    - (Opcional) cargo publish

# 5. Merge release → main
git checkout main
git merge release/v0.2.0 --ff-only
git push origin main

# 6. Cleanup
git branch -d release/v0.2.0
git push origin --delete release/v0.2.0
```

---

## 4. CI/CD Pipeline (GitHub Actions)

### 4.1 `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [develop]
  pull_request:
    branches: [develop, release/*]

env:
  CARGO_TERM_COLOR: always
  RUSTFLAGS: "-D warnings"

jobs:
  # 1. Check formatting
  fmt:
    name: Format Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo fmt --all --check

  # 2. Clippy
  clippy:
    name: Clippy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with: components: [clippy]
      - run: cargo clippy --workspace --features all -- -D warnings

  # 3. Build matrix (6 targets)
  build:
    name: Build ${{ matrix.os }} ${{ matrix.target }}
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        include:
          - os: ubuntu-latest
            target: x86_64-unknown-linux-gnu
          - os: ubuntu-latest
            target: aarch64-unknown-linux-gnu
          - os: macos-latest
            target: x86_64-apple-darwin
          - os: macos-latest
            target: aarch64-apple-darwin
          - os: windows-latest
            target: x86_64-pc-windows-msvc
          - os: windows-latest
            target: x86_64-pc-windows-gnu
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with: targets: ${{ matrix.target }}
      - name: Cache
        uses: Swatinem/rust-cache@v2
      - run: cargo build --workspace --features all --target ${{ matrix.target }}

  # 4. Testes
  test:
    name: Test
    runs-on: ubuntu-latest
    needs: [fmt, clippy]
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo test --workspace --features all -- --test-threads=4
      - run: cargo test --workspace --features all --test integration -- --test-threads=1

  # 5. Security audit
  audit:
    name: Security Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo install cargo-audit cargo-deny
      - run: cargo audit
      - run: cargo deny check

  # 6. Binary size check
  size:
    name: Binary Size
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo build --release --workspace --features all
      - run: |
          SIZE=$(stat -c%s target/release/ferro)
          echo "Binary size: $SIZE bytes ($((SIZE/1024/1024)) MB)"
          if [ $SIZE -gt 15728640 ]; then  # 15MB
            echo "::error::Binary size exceeds 15MB limit"
            exit 1
          fi

  # 7. Benchmarks (nightly only)
  bench:
    name: Benchmarks
    if: github.event_name == 'schedule' || github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@nightly
      - uses: Swatinem/rust-cache@v2
      - run: cargo bench --workspace --features all
      - uses: benchmark-action/github-action-benchmark@v1
        with:
          tool: 'cargo'
          output-file-path: bench-results.json
          github-token: ${{ secrets.GITHUB_TOKEN }}
          auto-push: true
          alert-threshold: '10%'

  # 8. Accessibility (axe-core)
  a11y:
    name: Accessibility
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo build --release --workspace --features all
      - run: |
          # Run ferro with axe-core test (headless)
          # Requires custom test binary that renders TUI and runs axe-core
          cargo test --test a11y --features all
```

### 4.2 `.github/workflows/release.yml`

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write
  packages: write

jobs:
  release:
    name: Release ${{ matrix.target }}
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        include:
          - os: ubuntu-latest
            target: x86_64-unknown-linux-gnu
            asset: ferro-x86_64-unknown-linux-gnu.tar.gz
          - os: ubuntu-latest
            target: aarch64-unknown-linux-gnu
            asset: ferro-aarch64-unknown-linux-gnu.tar.gz
          - os: macos-latest
            target: x86_64-apple-darwin
            asset: ferro-x86_64-apple-darwin.tar.gz
          - os: macos-latest
            target: aarch64-apple-darwin
            asset: ferro-aarch64-apple-darwin.tar.gz
          - os: windows-latest
            target: x86_64-pc-windows-msvc
            asset: ferro-x86_64-pc-windows-msvc.zip
          - os: windows-latest
            target: x86_64-pc-windows-gnu
            asset: ferro-x86_64-pc-windows-gnu.zip
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with: targets: ${{ matrix.target }}
      - uses: Swatinem/rust-cache@v2
      - name: Build release
        run: |
          cargo build --release --workspace --features all --target ${{ matrix.target }} \
            --config 'profile.release.strip = "debuginfo"' \
            --config 'profile.release.lto = true' \
            --config 'profile.release.panic = "abort"' \
            --config 'profile.release.codegen-units = 1'
      - name: Package
        run: |
          BINARY=target/${{ matrix.target }}/release/ferro${{ matrix.target == 'x86_64-pc-windows-msvc' || matrix.target == 'x86_64-pc-windows-gnu' && '.exe' || '' }}
          if [[ "${{ matrix.asset }}" == *.zip ]]; then
            7z a ${{ matrix.asset }} $BINARY
          else
            tar -czf ${{ matrix.asset }} -C $(dirname $BINARY) $(basename $BINARY)
          fi
      - name: Upload to Release
        uses: softprops/action-gh-release@v1
        with:
          files: ${{ matrix.asset }}
          generate_release_notes: true
```

---

## 5. Runbook Operacional

### 5.1 Comandos Diários

| Operação | Comando |
|----------|---------|
| **Iniciar trabalho** | `git checkout develop && git pull` |
| **Nova feature** | `git checkout -b DEV-minha-feature` |
| **Commit** | `git add -p && git commit -m "feat(scope): descrição"` |
| **Push + PR** | `git push -u origin DEV-minha-feature && gh pr create --base develop` |
| **Sync profile (multi-PC)** | `ferro --sync-profile pull` (início dia) / `push` (fim dia) |
| **Rodar testes** | `cargo test --workspace --features all` |
| **Lint local** | `cargo clippy --workspace --features all -- -D warnings && cargo fmt --check` |
| **Build release local** | `cargo build --release --workspace --features all` |
| **Ver traces** | `ferro traces export --since 1d --format csv > traces.csv` |

### 5.2 Troubleshooting Comum

| Problema | Diagnóstico | Solução |
|----------|-------------|---------|
| **Cold start > 2s** | `hyperfine 'ferro --version'` | Profile: `cargo build --release --features all`, check binary size, strip |
| **TUI lag (< 60fps)** | `RUST_LOG=trace ferro` → check frame times | Reduzir widgets ativos; `reduced_motion=true`; profile `tachyonfx` |
| **Git sync falha** | `ferro --sync-profile status` | Verificar `.ferro-sync.log`; manual `git -C ~/.config/ferro status` |
| **LLM gateway timeout** | `trace.json` → `latency_ms` alto | Health check CLIs; aumentar timeout; verificar rede |
| **Cache miss excessivo** | `ferro cache stats` → hit rate baixo | Aumentar threshold cosine; verificar embeddings |
| **LSP não inicia** | `ferro --lsp-debug` | Verificar `rust-analyzer` no PATH; logs `tower-lsp` |
| **Plugin não carrega** | `ferro plugin list` | Verificar assinatura (dylib); capabilities (WASM); logs |
| **Binary > 15MB** | `cargo bloat --release --features all` | Features seletivas; `strip`, `lto`, `panic=abort` |

### 5.3 Incident Response

```bash
# 1. Segurança (vazamento secret)
ferro audit --sast --dast
# → Laudo em docs/07-security/LAUDO.md
# → Rotacionar credenciais expostas
# → Verificar redaction patterns

# 2. CI falhando em main
git checkout main
git log --oneline -5
# → Reverter commit problemático: git revert <sha>
# → Push fix em develop → release branch

# 3. Performance regression (bench CI)
# → Comparar bench results: `gh run view <run-id> --log`
# → Bisect: `git bisect start HEAD HEAD~20` + `cargo bench`
```

---

## 6. Checklist Anti-Retrabalho (Por Entrega)

### 6.1 Pre-Commit (Local)
- [ ] `cargo test --workspace --features all` passa
- [ ] `cargo clippy --workspace --features all -- -D warnings` clean
- [ ] `cargo fmt --check` passa
- [ ] `cargo audit` clean (ou vulnerabilidades conhecidas documentadas)
- [ ] `cargo deny check` clean (licenças, bans)
- [ ] Binary size `< 15MB` (`cargo build --release`)
- [ ] Conventional Commit message válida

### 6.2 Pull Request (CI Gates)
- [ ] CI matrix build passa (6 targets)
- [ ] Todos testes unit + integração passam
- [ ] Coverage > 80% (`cargo llvm-cov --workspace --features all`)
- [ ] `kalimaster-sast` limpo (SAST no PR)
- [ ] Acessibilidade: axe-core 0 violações
- [ ] Benchmarks não regrediram > 10% (cold start, memory, binary size)
- [ ] Docs atualizadas (ADR se decisão nova, README, CHANGELOG)
- [ ] 1+ reviewer aprovou
- [ ] Breaking changes documentados + migration guide

### 6.3 Post-Merge (Develop)
- [ ] Branch de feature deletada (local + remota)
- [ ] `develop` atualizado localmente
- [ ] Changelog entry presente (conventional commit)
- [ ] Métricas registradas (velocity, lead time, bug escape)
- [ ] Retrospectiva se sprint finalizado

### 6.4 Release (Tag v*)
- [ ] Todos checks CI passam em `release/*` branch
- [ ] Binary artifacts gerados para 6 targets
- [ ] GitHub Release criado com notes (auto-generated)
- [ ] `main` atualizada com tag
- [ ] Changelog finalizado
- [ ] (Opcional) `cargo publish` para crates.io
- [ ] Anúncio: Discord, Twitter, newsletter

---

## 7. Métricas de Sucesso (KPIs)

| Métrica | Target MVP (v0.1) | Target GA (v1.0) | Medição |
|---------|-------------------|------------------|---------|
| **Cold start** | < 3s | < 2s | `hyperfine` CI nightly |
| **Memory idle** | < 100MB | < 50MB | `ps_record` CI nightly |
| **TUI frame time** | < 33ms (30fps) | < 16ms (60fps) | `tracing` spans |
| **Agent switch** | < 100ms | < 50ms | Benchmark interno |
| **LLM first token** | < 2s (remote) | < 500ms (local) / < 2s (remote) | `trace.json` |
| **LLM streaming** | > 20 tok/s | > 50 tok/s | `trace.json` |
| **Semantic cache hit** | > 30% | > 60% | `ferro cache stats` |
| **Git sync reliability** | 95% | 100% | `.ferro-sync.log` analysis |
| **Test coverage** | > 60% unit | > 80% unit + 100% critical | `cargo llvm-cov` |
| **Accessibility** | 0 critical | 0 AA violations | axe-core CI |
| **Binary size** | < 20MB | < 15MB | `cargo bloat` release |
| **Velocity** | 20 SP/sprint | 30 SP/sprint | GitHub Projects |
| **Bug escape rate** | < 10% | < 5% | Issues labeled bug |
| **Lead time** | < 7 dias | < 3 dias | PR opened → merged |

---

## 8. Próximos Passos Imediatos (Kickoff Sprint 1)

### Semana 1 (Setup + PoC)
- [ ] **Day 1:** Criar workspace Cargo + CI básico (fmt, clippy, build matrix)
- [ ] **Day 2:** PoC TUI: `ratatui` + `crossterm` + layout splits + vim keys (validar 60fps)
- [ ] **Day 3:** Command palette (skim) + theming (opaline) + hot reload config (figment+notify)
- [ ] **Day 4:** Agent trait + registry vazio + Cargo features skeleton
- [ ] **Day 5:** Retrospectiva + planejamento Sprint 1 backlog refinement

### Semana 2 (Core TUI Complete)
- [ ] Dashboard com 3 painéis (Agents, Sessions, Files)
- [ ] Config validation + error handling
- [ ] Unit tests: layout, palette, theme, config reload
- [ ] Integration test: `ferro --version`, `ferro --help`, cold start benchmark
- [ ] PR + merge para `develop`

---

## 9. Comunicação e Artefatos

| Artefato | Local | Frequência | Responsável |
|----------|-------|------------|-------------|
| **Sprint Planning** | GitHub Projects | Início sprint | PLANMASTER + Team |
| **Daily Standup** | Discord/Async | Diário | Team |
| **Sprint Review** | GitHub Projects | Fim sprint | Team + Stakeholders |
| **Retrospectiva** | docs/retrospectives/ | Fim sprint | Team |
| **Architecture Decisions** | docs/04-ADRs/ | Conforme necessário | PLANMASTER |
| **Incident Reports** | docs/incidents/ | Conforme necessário | Kalimaster/Opsmaster |
| **Release Notes** | GitHub Releases / CHANGELOG.md | Por release | Opsmaster |
| **Benchmarks** | CI artifacts / docs/benchmarks/ | Nightly + Release | PLANMASTER |

---

## 10. Aprovação e Handoff

| Role | Nome | Assinatura | Data |
|------|------|------------|------|
| **PLANMASTER** | | | 28/10/2026 |
| **Tech Lead (Você)** | | | |
| **Backdev (Implementação)** | | | |
| **Frontdev (TUI/Widgets)** | | | |
| **Engai (Gateway/IA)** | | | |
| **Kalimaster (Segurança)** | | | |
| **Opsmaster (CI/CD/Infra)** | | | |
| **Qamaster (Testes)** | | | |

---

> **Status:** Planejamento completo (FASE 0-6). Pronto para **iniciar Sprint 1** com `backdev`/`frontdev` implementando `crates/ferro-core` + `crates/ferro-tui`.
>
> **Comando para iniciar desenvolvimento:**
> ```bash
> cd /home/oficina/PROJETOS/agente-tui-rust
> git checkout develop
> # Criar branch para Sprint 1
> git checkout -b DEV-sprint1-core-tui
> # Iniciar implementação...
> ```