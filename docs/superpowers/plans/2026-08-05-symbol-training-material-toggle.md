# Symbol Training Material Toggle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move symbol training from the pacing dropdown into a material toggle beside punctuation and digits.

**Architecture:** Keep `TypingTrainerMode::{Time, Words}` for pacing and add `symbols_enabled` as independent material state. The legacy serialized `Symbols` value maps to `Words + symbols_enabled` during normalization. The egui row decides its controls and labels using the material flag.

**Tech Stack:** Rust 2021, egui, serde, TOML i18n catalogs, Cargo tests.

## Global Constraints

- Symbols use the current keyboard layout and the existing adaptive statistics.
- Language, punctuation and digits are hidden in symbols but keep their text-mode values.
- Symbols support time or count; count values are 25, 50 and 100.
- No firmware writes, device-specific persistence, or dependencies are added.

---

### Task 1: Persist material independently of pacing

**Files:**
- Modify: `src/app_state.rs:2590-3100`
- Modify: `src/app_storage.rs:300-322`
- Test: `src/app_state.rs:typing_trainer_tests`

**Interfaces:** Add `symbols_enabled: bool` to `TypingTrainerSettings`, `TypingTrainerState`, and `TypingTrainerRunRecord`; add `is_symbol_training(&self) -> bool` and `set_symbols_enabled(&mut self, enabled: bool)` to state.

- [ ] **Step 1: Write failing migration and round-trip tests**

```rust
#[test]
fn legacy_symbol_mode_normalizes_to_symbol_material_with_count_pacing() {
    let settings = TypingTrainerSettings {
        mode: TypingTrainerMode::Symbols,
        ..TypingTrainerSettings::default()
    }.normalized();
    assert_eq!(settings.mode, TypingTrainerMode::Words);
    assert!(settings.symbols_enabled);
}

#[test]
fn disabling_symbols_restores_saved_text_options() {
    let mut state = TypingTrainerState::from_settings(TypingTrainerSettings {
        punctuation_enabled: true,
        numbers_enabled: true,
        ..TypingTrainerSettings::default()
    });
    state.set_symbols_enabled(true);
    state.set_symbols_enabled(false);
    assert!(state.punctuation_enabled);
    assert!(state.numbers_enabled);
}
```

- [ ] **Step 2: Run the red tests**

```bash
nix shell nixpkgs#rustc nixpkgs#cargo nixpkgs#gcc --command cargo test legacy_symbol_mode_normalizes_to_symbol_material_with_count_pacing disabling_symbols_restores_saved_text_options -- --nocapture
```

Expected: compilation failure because the material flag and setter do not exist.

- [ ] **Step 3: Implement normalization and state**

```rust
let (mode, symbols_enabled) = match self.mode {
    TypingTrainerMode::Symbols => (TypingTrainerMode::Words, true),
    mode => (mode, self.symbols_enabled),
};
```

Mirror the flag in `TypingTrainerState` and `settings()`. Replace active
`TypingTrainerMode::Symbols` branches in target generation, finish conditions,
visible-text extension, statistics recording, history labels and matching with
`is_symbol_training()`. Do not mutate punctuation/digits when enabling symbols;
the symbol generator ignores them so their saved text-mode values return later.
Normalize legacy records before history truncation by changing `Symbols` to
`Words` and setting `symbols_enabled = true`.

- [ ] **Step 4: Run green state tests and commit**

```bash
nix shell nixpkgs#rustc nixpkgs#cargo nixpkgs#gcc --command cargo test typing_trainer -- --nocapture
git add src/app_state.rs src/app_storage.rs
git commit -m "refactor: отделить материал символьной тренировки"
```

### Task 2: Render the symbols material pill

**Files:**
- Modify: `src/ui/typing_trainer.rs:163-340`
- Modify: `i18n/en.toml:[typing_trainer]`
- Modify: `i18n/ru.toml:[typing_trainer]`
- Test: `src/ui/typing_trainer.rs:typing_trainer_ui_tests`

**Interfaces:** Add `typing_trainer_pacing_labels(language, symbols_enabled) -> [String; 2]`, consuming Task 1 material state.

- [ ] **Step 1: Write failing contextual-label test**

```rust
#[test]
fn symbol_material_uses_count_instead_of_words() {
    assert_eq!(
        typing_trainer_pacing_labels(crate::i18n::Language::English, true),
        ["time".to_owned(), "count".to_owned()]
    );
}
```

- [ ] **Step 2: Run the red test**

```bash
nix shell nixpkgs#rustc nixpkgs#cargo nixpkgs#gcc --command cargo test symbol_material_uses_count_instead_of_words -- --nocapture
```

Expected: compilation failure because the helper and `typing_trainer.count` key do not exist.

- [ ] **Step 3: Implement the two row variants**

Add `count = "count"` to `i18n/en.toml` and `count = "количество"` to
`i18n/ru.toml`. Use `modern_toggle_pill` for `symbols`, rendering:

```text
text:    [language] [time | words] [value] [punct] [digits] [symbols]
symbols: [time | count] [value] [symbols]
```

In symbol material, count retains internal `TypingTrainerMode::Words` but uses
`TYPING_TRAINER_SYMBOL_COUNTS`; do not draw language, punctuation or digits.

- [ ] **Step 4: Verify, commit, and push**

```bash
nix shell nixpkgs#rustc nixpkgs#cargo nixpkgs#gcc --command cargo test --all-targets
python3 scripts/check_i18n.py
nix shell nixpkgs#rustfmt --command rustfmt --check src/app_state.rs src/app_storage.rs src/ui/typing_trainer.rs
git add src/app_state.rs src/app_storage.rs src/ui/typing_trainer.rs i18n/en.toml i18n/ru.toml
git commit -m "feat: вынести символы в материал тренировки"
git push origin feat/adaptive-symbol-trainer
```

Expected: automated checks pass; manual K:04 validation shows `time | count`, a wrapped 100-symbol target, and restoration of language/punctuation/digits after symbols is disabled.
