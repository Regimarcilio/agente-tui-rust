# ADR-006: TUI Framework — ratatui + Ecosystem (tachyonfx, opaline, syntect)

**Status:** Accepted  
**Data:** 28/10/2026  
**Autor:** PLANMASTER  
**Relacionado:** RF-001, RF-009, RF-010, RF-011, RF-012, RF-013, RF-018, RNF-001, RNF-004, `docs/02-benchmarks.md` Seção 1

---

## Contexto

O FERRO precisa de TUI "nível Obra" (diff inline, LSP, syntax highlight, git UI, command palette, animações, theming). Frameworks avaliados (ver benchmarks):

| Critério | **ratatui** | crossterm+tui-rs | cursive | iocraft | egui |
|----------|-------------|------------------|---------|---------|------|
| Stars | 23.1k | 8.2k (archived) | 3.8k | 1.2k | 24.5k |
| Widgets ricos | ✅ (ecossistema) | Básico | Rico | Componentes | Rico |
| Syntax highlight | `ratatui-code-editor` | Manual | Manual | Manual | `egui_extras` |
| Diff inline | Custom (giff, Livediff) | Manual | Manual | Manual | Manual |
| LSP client | `tower-lsp` + widget | Não | Não | Não | `egui_lsp` exp. |
| Animações | `tachyonfx` (50+) | Básico | Básico | Limitado | `egui_animate` |
| Theming | `opaline` (token-based) | Manual | Básico | CSS-like | Rico |
| Apps referência | gitui, Yazi, borg | Legado | - | - | - |
| no_std | ✅ (core) | ❌ | ❌ | ❌ | ❌ |

---

## Decisão

**ratatui 0.30+ como framework TUI core + ecossistema complementar:**

### Stack TUI Definitiva

| Camada | Crate | Versão | Propósito |
|--------|-------|--------|-----------|
| **Core** | `ratatui` | 0.30+ | Layout, widgets base, backend abstraction |
| **Backend** | `ratatui-crossterm` | 0.30+ | Cross-platform terminal (Win/Mac/Linux) |
| **Animações** | `tachyonfx` | 0.6+ | 50+ effects, DSL, WASM, no_std, 60fps |
| **Theming** | `opaline` | 0.3+ | Token-based, 20 themes, selector widget |
| **Syntax Highlight** | `syntect` + `tree-sitter` | latest | Highlight incremental, grammars bundladas |
| **Code Editor** | `ratatui-code-editor` | latest | Editor widget com tree-sitter, LSP integration |
| **Diff** | `diffrs` | latest | Myers O(ND), inline/side-by-side/unified |
| **LSP** | `tower-lsp` + `lsp-types` | latest | Client LSP 3.17, usado por helix/lapce |
| **Git** | `git2` | 0.18+ | libgit2 bindings, vendored, ssh/https |
| **Fuzzy Search** | `skim` | 0.12+ | Command palette, file finder |
| **File Explorer** | `ratatui-explorer` / custom | latest | Tree view, git status, CRUD |

### Arquitetura de Widgets Compostos

```
TUI Runtime (ratatui + crossterm)
├── LayoutManager (splits, tabs, focus)
├── WidgetRegistry (dynamic widgets)
│   ├── AgentPanel (lista 19 agentes, status)
│   ├── SessionPanel (histórico, busca, export)
│   ├── FileExplorer (tree, git status, CRUD)
│   ├── EditorTab (ratatui-code-editor + syntect)
│   ├── DiffView (diffrs + syntax highlight)
│   ├── GitWidget (git2 + log graph + staging)
│   ├── LspWidget (tower-lsp + diagnostics inline)
│   ├── CommandPalette (skim + ações categorizadas)
│   └── StatusBar (mode, git branch, LSP status, clock)
└── ThemeEngine (opaline + base16 + custom)
```

### Integração tachyonfx (Animações Nível Obra)

```rust
// ferro-tui/src/animations.rs
use tachyonfx::{Effect, EffectTimer, Interpolation, fx};

pub fn build_effects() -> Vec<Effect> {
    vec![
        // Transições de tab/painel
        fx::slide_in::left(Duration::from_millis(200), Interpolation::EaseOut),
        fx::fade_in::from(0.0, Duration::from_millis(150)),
        
        // Command palette
        fx::scale::from(0.9, Duration::from_millis(100), Interpolation::EaseOut),
        fx::fade_in::from(0.0, Duration::from_millis(80)),
        
        // Diff hunk highlight
        fx::highlight::bg(Color::Rgb(60, 60, 40), Duration::from_millis(500)),
        
        // LSP diagnostic pulse
        fx::pulse::bg(Color::Red, Duration::from_secs(2), 3),
        
        // Git status change
        fx::shake::horizontal(2.0, Duration::from_millis(300)),
    ]
}
```

### Theming com opaline (Token-based)

```rust
// ferro-tui/src/theme.rs
use opaline::{Theme, Token, tokens};

pub fn load_theme(name: &str) -> Theme {
    match name {
        "dracula" => opaline::themes::DRACULA.clone(),
        "gruvbox" => opaline::themes::GRUVBOX_DARK.clone(),
        "catppuccin" => opaline::themes::CATPPUCCIN_MOCHA.clone(),
        "nord" => opaline::themes::NORD.clone(),
        "tokyo-night" => opaline::themes::TOKYO_NIGHT.clone(),
        _ => opaline::themes::DEFAULT.clone(),
    }
    // Custom tokens para FERRO
    .with_tokens([
        (tokens::BG, Token::rgb(0x1e, 0x1e, 0x2e)),
        (tokens::FG, Token::rgb(0xcd, 0xd6, 0xf4)),
        (tokens::ACCENT, Token::rgb(0xf5, 0xc2, 0xe7)),
        (tokens::SUCCESS, Token::rgb(0xa6, 0xe3, 0xa1)),
        (tokens::WARNING, Token::rgb(0xf9, 0xe2, 0xaf)),
        (tokens::ERROR, Token::rgb(0xf3, 0x8b, 0xa8)),
        // FERRO-specific
        (Token::new("ferro.agent.active"), Token::rgb(0x89, 0xb4, 0xfa)),
        (Token::new("ferro.agent.idle"), Token::rgb(0x6c, 0x70, 0x86)),
        (Token::new("ferro.git.added"), Token::rgb(0xa6, 0xe3, 0xa1)),
        (Token::new("ferro.git.modified"), Token::rgb(0xf9, 0xe2, 0xaf)),
        (Token::new("ferro.git.deleted"), Token::rgb(0xf3, 0x8b, 0xa8)),
    ])
}
```

---

## Consequências

### Positivas
- **Ecossistema maduro:** 23k stars, apps "Obra" comprovados (gitui, Yazi, borg, claude-code-rust)
- **Modular:** Workspace (core/widgets/backends) = bundle size controlado
- **Performance:** 60fps fácil, layout cache, no_std core
- **Extensível:** Widgets custom, animações DSL, theming token-based
- **Cross-platform:** crossterm backend = Win/Mac/Linux sem código específico

### Negativas
- **Curva aprendizado:** Média (ratatui + tachyonfx + opaline + syntect)
- **Bundle size:** ~2MB core + widgets + deps
- **API 0.30 breaking:** Mudanças recentes; travar versão exata

### Riscos Mitigados
| Risco | Mitigação |
|-------|-----------|
| ratatui 0.30 breaking changes | Pin `ratatui = "=0.30.0"` no Cargo.lock; testar migração isolada |
| tree-sitter grammars size | Features opcionais por linguagem; download on-demand via `tree-sitter-cli` |
| tachyonfx performance em terminais lentos | Fallback: `reduced_motion = true` desabilita animações |
| syntect vs tree-sitter conflict | Usar `tree-sitter-highlight` para editor; `syntect` só para diff/preview |

---

## Validação

- [ ] `cargo build --features tui` compila < 30s
- [ ] Cold start `ferro --version` < 2s (hyperfine)
- [ ] TUI 60fps sustentado com 3 splits + editor + git log (10k commits)
- [ ] Syntax highlight Rust/TS/Python/Go/TOML/JSON/YAML/MD funciona
- [ ] Diff inline Myers: side-by-side + unified + inline modes
- [ ] LSP diagnostics inline + painel lateral + code actions via palette
- [ ] Command palette Ctrl+P: fuzzy search agentes, arquivos, ações, config
- [ ] Themes: 20+ base16 + custom tokens FERRO aplicam em < 50ms
- [ ] Animações tachyonfx: tab switch, palette open, diff highlight suaves
- [ ] Acessibilidade: navegação teclado-only, screen reader events, high contrast

---

## Referências
- ratatui: https://github.com/ratatui/ratatui (ARCHITECTURE.md)
- awesome-ratatui: https://github.com/ratatui/awesome-ratatui
- tachyonfx: https://github.com/tachyonfx/tachyonfx
- opaline: https://github.com/opaline-rs/opaline
- ratatui-code-editor: https://github.com/ratatui/ratatui-code-editor
- syntect: https://github.com/trishume/syntect
- tree-sitter: https://tree-sitter.github.io/tree-sitter/