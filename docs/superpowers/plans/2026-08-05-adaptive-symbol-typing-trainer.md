# Adaptive Symbol Typing Trainer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `Symbols` Typing Trainer mode that derives printable characters from the connected keyboard’s layers and adaptively repeats characters with more mistakes.

**Architecture:** Keep keymap parsing and weighted target generation in a new pure `typing_trainer_symbols` module. Extend the existing trainer state only with mode, target length and transient symbol pool; store the global per-character counters in `AppSettings`. The existing `ui/typing_trainer.rs` owns the bridge to the connected `KeyboardLayout`, records outcomes and renders the new controls.

**Tech Stack:** Rust 2021, egui/eframe, serde/serde_json, existing Vial/RMK keymap model, Cargo test.

## Global Constraints

- Do not alter keyboard firmware or write any keymap data to a device.
- Derive symbols only from the already loaded current `KeyboardLayout`; do not issue HID reads when an exercise starts.
- Include direct printable Vial keycodes and their supported Shift variants; skip transparent, no-op, control, macro, AltGr, Compose and multi-key actions.
- Keep character statistics global to the Entropy application, not per device, layer or language.
- Preserve deserialization of all existing settings and history files with `#[serde(default)]`.
- Give every available character a nonzero base weight even after adaptive weighting.
- UI strings must be added in both `i18n/en.toml` and `i18n/ru.toml`.
- Commit messages must be in Russian and must not contain vendor footers or `Co-Authored-By` lines.

---

## File structure

- Create: `src/typing_trainer_symbols.rs` — pure conversion of `KeyboardLayout` bindings into symbols, deterministic weighted selection, and per-character counter helpers with unit tests.
- Modify: `src/app.rs` — register the new focused module before `app_state` so state and UI can import it.
- Modify: `src/app_state.rs` — add `TypingTrainerMode::Symbols`, persisted symbol length and global statistics, extend state transitions and record extraction.
- Modify: `src/ui/typing_trainer.rs` — expose the Symbols mode, derive/update the current pool, show unavailable state, record outcomes and format history.
- Modify: `src/ui/settings_shell.rs` — pass the current `KeyboardLayout` into the trainer renderer.
- Modify: `i18n/en.toml` — English labels and unavailable-state explanation.
- Modify: `i18n/ru.toml` — Russian labels and unavailable-state explanation.

## Interfaces

`src/typing_trainer_symbols.rs` provides these interfaces to state and UI:

```rust
pub(crate) const TYPING_TRAINER_SYMBOL_COUNTS: [usize; 3] = [25, 50, 100];

#[derive(Clone, Copy, Debug, Default, PartialEq, Eq, serde::Serialize, serde::Deserialize)]
pub(crate) struct TypingTrainerCharacterStats {
    pub(crate) attempts: u64,
    pub(crate) errors: u64,
}

pub(crate) type TypingTrainerCharacterStatsMap = std::collections::BTreeMap<char, TypingTrainerCharacterStats>;

pub(crate) fn printable_symbols_from_layout(layout: &crate::keyboard::KeyboardLayout) -> Vec<char>;
pub(crate) fn weighted_symbol_text(
    symbols: &[char],
    count: usize,
    stats: &TypingTrainerCharacterStatsMap,
    seed: usize,
) -> String;
pub(crate) fn record_symbol_attempt(
    stats: &mut TypingTrainerCharacterStatsMap,
    expected: char,
    was_error: bool,
);
```

`TypingTrainerState` provides the UI with:

```rust
pub(crate) fn set_symbol_pool(&mut self, symbols: Vec<char>);
pub(crate) fn symbol_pool_available(&self) -> bool;
pub(crate) fn set_symbol_count(&mut self, count: usize);
pub(crate) fn type_char(&mut self, ch: char, now: Instant) -> Option<(char, bool)>;
```

The tuple is `(expected_character, was_error)` and is returned only for an input character accepted by an active Symbols run. Existing callers ignore it in non-Symbols modes.

### Task 1: Pure symbol pool and adaptive generator

**Files:**
- Create: `src/typing_trainer_symbols.rs`
- Modify: `src/app.rs:15-19`

**Consumes:** `KeyboardLayout`, `KeyBinding::Vial`, and Vial basic keycodes already represented in `src/keycode.rs`.

**Produces:** `printable_symbols_from_layout`, `weighted_symbol_text`, `TypingTrainerCharacterStats`, and `record_symbol_attempt` for Tasks 2 and 3.

- [ ] **Step 1: Write failing extraction tests**

Add a `#[cfg(test)]` module that constructs a minimal `KeyboardLayout` with three layers and asserts this exact result:

```rust
assert_eq!(
    printable_symbols_from_layout(&layout),
    vec!['1', '!', 'a', 'A', '/', '?']
);
```

Use Vial keycodes `0x001E` (`1`), `0x0004` (`A`), and `0x0038` (`/`); include `0x0000`, `0x0001`, and `0x0028` (`Enter`) in the fixture and assert none leaks into the output. Add a second test proving duplicate keycodes across layers create a single character pair.

- [ ] **Step 2: Run the extraction test and verify it fails**

Run:

```bash
cargo test typing_trainer_symbols::tests::printable_symbols -- --exact
```

Expected: compilation failure because `typing_trainer_symbols` and `printable_symbols_from_layout` do not yet exist.

- [ ] **Step 3: Implement the smallest symbol extractor**

Create `src/typing_trainer_symbols.rs` using `BTreeSet<char>` for deterministic deduplication. Match only direct Vial basic keycodes:

```rust
fn direct_and_shifted_symbols(keycode: u16) -> Option<(char, Option<char>)>
```

Map `0x0004..=0x001d` to lowercase/uppercase ASCII letters, `0x001e..=0x0027` to `1..0` and `!@#$%^&*()`, and the US punctuation keycodes `0x002d..=0x0038` to their unshifted/shifted pairs (`-/_`, `=/+`, `[/ {`, `]/}`, `\\/|`, `;/ :`, `' / \"`, `` `/~``, `,/<`, `./>`, `/?`). Return `None` for all other bindings and for every `KeyBinding::Rmk` in this release. Iterate all `layout.layers`, flatten bindings, then return the sorted set.

Register the module in `src/app.rs` immediately after `typing_trainer_words`.

- [ ] **Step 4: Run the extraction tests and verify they pass**

Run:

```bash
cargo test typing_trainer_symbols::tests::printable_symbols -- --exact
```

Expected: both extraction tests pass.

- [ ] **Step 5: Write failing weighting and recording tests**

Add tests with `symbols = &['a', 'b', 'c']` that assert:

```rust
assert_eq!(weighted_symbol_text(symbols, 0, &stats, 9), "");
assert_eq!(weighted_symbol_text(&['x'], 4, &stats, 9), "xxxx");
assert!(weighted_symbol_text(symbols, 24, &stats, 9).chars().collect::<BTreeSet<_>>().len() > 1);
```

Then record ten attempts and five errors for `b`, produce a fixed-seed 3,000-character sample, and assert `b` occurs more often than both `a` and `c`. Add a test that `record_symbol_attempt` increments `attempts` every time and increments `errors` only when `was_error` is true.

- [ ] **Step 6: Run the weighting tests and verify they fail**

Run:

```bash
cargo test typing_trainer_symbols::tests::weighted -- --exact
```

Expected: compilation failure because the generator and counter helper are absent.

- [ ] **Step 7: Implement deterministic weighted generation**

Implement a small deterministic PRNG local to the module (a wrapping `u64` xorshift) so tests and `Retry` are reproducible without a dependency. Calculate an integer weight as `100 + (100 * errors / attempts.max(1))`, giving every symbol a base weight and limiting the error bonus to 100%. Draw proportionally from cumulative weights. If `symbols.len() > 1` and a target would contain one unique symbol, replace its last character with the next pool character. Implement `record_symbol_attempt` by updating the entry produced by `or_default()`.

- [ ] **Step 8: Run the focused module tests and format**

Run:

```bash
cargo fmt --check
cargo test typing_trainer_symbols::tests -- --nocapture
```

Expected: format check succeeds and all module tests pass.

- [ ] **Step 9: Commit the pure model**

```bash
git add src/app.rs src/typing_trainer_symbols.rs
git commit -m "feat: добавить генератор символьной тренировки"
```

### Task 2: Persisted trainer mode and per-character outcomes

**Files:**
- Modify: `src/app_state.rs:80-90, 2590-3080`
- Test: `src/app_state.rs` existing `typing_trainer_tests` and `app_settings_tests` modules

**Consumes:** Task 1’s `TypingTrainerCharacterStatsMap`, symbol count constant and text generator.

**Produces:** compatible settings, a Symbols state machine, and a per-keystroke result for Task 3.

- [ ] **Step 1: Write failing settings compatibility tests**

Add tests that deserialize this pre-feature JSON without failure and with empty statistics:

```rust
let settings: AppSettings = serde_json::from_str(
    r#"{\"typing_trainer\":{\"mode\":\"words\",\"word_count\":50}}"#,
).unwrap();
assert!(settings.typing_trainer_character_stats.is_empty());
```

Add a settings round-trip test for `TypingTrainerMode::Symbols` and `symbol_count: 50`, plus a normalization test that replaces `symbol_count: 17` with the default `50`.

- [ ] **Step 2: Run the settings tests and verify they fail**

Run:

```bash
cargo test app_settings_tests::typing_trainer_symbol -- --exact
```

Expected: compilation failure because the Symbols enum variant and new persisted fields are absent.

- [ ] **Step 3: Add compatible data fields and normalization**

Add `Symbols` to `TypingTrainerMode`; add `symbol_count` with `#[serde(default = "default_typing_trainer_symbol_count")]` to `TypingTrainerSettings`; add `typing_trainer_character_stats` with `#[serde(default)]` to `AppSettings`; initialise it in `Default`. Add `default_typing_trainer_symbol_count() -> usize { 50 }` and normalize against `TYPING_TRAINER_SYMBOL_COUNTS`. Extend `TypingTrainerRunRecord::matches_settings` so Symbols compares `symbol_count`, not duration or word count.

- [ ] **Step 4: Run settings tests and verify they pass**

Run:

```bash
cargo test app_settings_tests::typing_trainer_symbol -- --exact
```

Expected: the legacy, round-trip and normalization tests pass.

- [ ] **Step 5: Write failing trainer-state tests**

Add tests that configure `Symbols`, call `set_symbol_pool(vec!['a', 'A', '!'])`, and assert a 25-character target uses only that pool. Add a test that `set_symbol_pool(Vec::new())` makes `symbol_pool_available()` false and leaves target text empty. Add this outcome test:

```rust
let outcome = state.type_char('x', now);
assert_eq!(outcome, Some(('a', true)));
```

Then call `backspace()` and assert the already returned outcome is not retroactively changed; the UI will use it to retain the counter update.

- [ ] **Step 6: Run the trainer-state tests and verify they fail**

Run:

```bash
cargo test typing_trainer_tests::symbol -- --exact
```

Expected: compilation failure because no symbol pool, count setter or outcome is implemented.

- [ ] **Step 7: Implement Symbols state transitions**

Add transient `symbol_pool: Vec<char>` to `TypingTrainerState`; initialise it empty and expose `set_symbol_pool`, `symbol_pool_available`, and `set_symbol_count`. Extend `from_settings`, `new_target_text`, `remaining_secs_at`, `pause_if_running`, `word_progress`, `typing_trainer_focus_status` callers and `extend_target_text` logic so Symbols behaves as a finite-count exercise like Words, never appends word text and finishes on the configured target length. Let `type_char` capture the expected target character at the current index before pushing the input, then return `Some((expected, ch != expected))` only in Symbols mode. Use Task 1’s generator to create the target text from the transient pool and the state seed; no statistics are mutated in state.

- [ ] **Step 8: Run state tests and the existing trainer suite**

Run:

```bash
cargo test typing_trainer_tests -- --nocapture
cargo test app_settings_tests -- --nocapture
```

Expected: new and existing tests pass, including Time and Words regression cases.

- [ ] **Step 9: Commit state and persistence**

```bash
git add src/app_state.rs
git commit -m "feat: сохранить статистику ошибок тренажёра"
```

### Task 3: Connect the live layout, controls and localisation

**Files:**
- Modify: `src/ui/settings_shell.rs:10-80`
- Modify: `src/ui/typing_trainer.rs:32-335, 375-395, 1050-1130`
- Modify: `i18n/en.toml`
- Modify: `i18n/ru.toml`

**Consumes:** Task 1’s symbol extraction and recording helpers plus Task 2’s state API.

**Produces:** a usable Symbols-mode screen and persisted adaptive input outcomes.

- [ ] **Step 1: Write failing UI-adjacent tests for formatting and mode labels**

Extend existing `typing_trainer_history_run_label` tests with a `Symbols` record and assert it renders the translated Symbols label plus `50`. Add pure helper tests for a new `typing_trainer_symbols_unavailable(layout)` condition: a layout containing only `KC_NO` and `KC_TRNS` returns true; a layout with `KC_A` returns false.

- [ ] **Step 2: Run the tests and verify they fail**

Run:

```bash
cargo test typing_trainer -- --nocapture
```

Expected: compilation failure or assertion failure until Symbols labels and availability helper exist.

- [ ] **Step 3: Pass the current layout into the trainer page**

Change `draw_typing_trainer_page` to accept `layout: &KeyboardLayout`; update its sole `SettingsTab::TypingTrainer` call in `src/ui/settings_shell.rs`. At the start of the page, obtain `printable_symbols_from_layout(layout)` and call `self.typing_trainer.set_symbol_pool(pool)` only when the pool changed, preserving the current run when it has not.

- [ ] **Step 4: Add the Symbols control path and unavailable state**

Extend the mode labels and selector mapping with `TypingTrainerMode::Symbols`. In Symbols mode, show a `25 / 50 / 100` selector bound to `set_symbol_count`; hide language, punctuation and numbers controls. If no pool is available, render `typing_trainer.symbols_unavailable`, disable Restart/Next/Retry as applicable, and do not call the text renderer with an empty target. Continue to call `save_typing_trainer_settings()` after user-controlled changes.

- [ ] **Step 5: Record outcomes at the input boundary**

Replace the ignored result of `self.typing_trainer.type_char(ch, now)` in `handle_typing_trainer_input` with:

```rust
if let Some((expected, was_error)) = self.typing_trainer.type_char(ch, now) {
    crate::typing_trainer_symbols::record_symbol_attempt(
        &mut self.app_settings.typing_trainer_character_stats,
        expected,
        was_error,
    );
    save_app_settings(&self.app_settings);
}
```

Keep existing focus-mode behavior unchanged. This intentionally commits an outcome before Backspace, matching the approved requirement.

- [ ] **Step 6: Localise every new string and update history/focus labels**

Add `typing_trainer.symbols` and `typing_trainer.symbols_unavailable` to both catalog files. Extend `typing_trainer_focus_status` to show `typed/target` for Symbols and `typing_trainer_history_run_label` to show the Symbols count without language, punctuation or number suffixes. Update any exhaustive `match` over `TypingTrainerMode` in this file.

- [ ] **Step 7: Run focused tests and format**

Run:

```bash
cargo fmt --check
cargo test typing_trainer -- --nocapture
cargo test i18n -- --nocapture
```

Expected: all tests pass and no `TypingTrainerMode` match is non-exhaustive.

- [ ] **Step 8: Perform a manual K:04 smoke test**

Run Entropy with K:04 connected. Open Advanced → Typing Trainer → Symbols and verify:

1. the selector offers 25, 50 and 100;
2. the target contains letters, shifted letters, numbers and punctuation present on the loaded layers;
3. an intentional wrong character increments the displayed run error count;
4. after completing a run with repeated mistakes on `?`, the next Symbols run contains `?` more often than an untouched punctuation character;
5. unplugging the keyboard produces the unavailable explanation rather than a panic or empty exercise;
6. Time and Words still start and finish normally.

- [ ] **Step 9: Commit the UI integration**

```bash
git add src/ui/settings_shell.rs src/ui/typing_trainer.rs i18n/en.toml i18n/ru.toml
git commit -m "feat: добавить символьный режим тренажёра"
```

### Task 4: Full regression verification and delivery preparation

**Files:**
- Modify only if a verification failure identifies a concrete defect in Tasks 1–3.

**Consumes:** completed model, state and UI work from Tasks 1–3.

**Produces:** verified branch ready for upstream pull request review.

- [ ] **Step 1: Run the complete automated suite**

Run:

```bash
cargo fmt --check
cargo test --all-targets
```

Expected: both commands exit 0.

- [ ] **Step 2: Build the release binary**

Run:

```bash
cargo build --release
```

Expected: exit 0 and create `target/release/entropy` (or platform equivalent) without warnings promoted to errors.

- [ ] **Step 3: Inspect the change scope**

Run:

```bash
git diff upstream/main...HEAD --check
git diff --stat upstream/main...HEAD
git status --short
```

Expected: no whitespace errors, only the files in this plan plus intentionally added tests, and a clean worktree.

- [ ] **Step 4: Commit a concrete regression fix only if verification required one**

If and only if Steps 1–3 required a source change, add the exact changed files and commit:

```bash
git add -u
git commit -m "fix: устранить регрессию символьного тренажёра"
```

If no verification change was needed, do not make an empty commit.

- [ ] **Step 5: Push the finished branch and prepare the pull request summary**

Run:

```bash
git push origin main
```

Prepare a PR description that names the new `Symbols` mode, global adaptive statistics, legacy-settings compatibility and the exact commands from Steps 1–2. Do not add a vendor signature or `Co-Authored-By` footer.

## Plan self-review

- Spec coverage: Tasks 1–3 cover all printable supported direct/Shift symbols, all layers, deduplication, global persisted counters, compatibility, weighted selection, empty pool behavior, UI mode, error retention, history and localization. Task 4 covers the K:04 manual verification and regression gate.
- Scope: manual symbol lists, per-device profiles, advanced modifier synthesis, dashboards and firmware changes are explicitly omitted.
- Type consistency: Task 1 defines all external model names; Task 2 consumes them and exposes the state methods consumed by Task 3. `Symbols`, `symbol_count`, `symbol_pool`, and `TypingTrainerCharacterStatsMap` use the same spelling throughout.
- Placeholder scan: no unresolved requirements, generic test instructions or undefined interfaces remain.
