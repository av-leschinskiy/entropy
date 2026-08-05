# Symbol Training Stats Storage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Store adaptive symbol-training statistics in a separate local JSON file.

**Architecture:** `app_storage` owns the path and JSON I/O for the statistics
map. `EntropyApp` loads the map once at startup and keeps it in memory; the
Typing Trainer updates that map and writes it when a run finishes. The general
`AppSettings` format stops containing the map, and legacy embedded data is
ignored without migration.

**Tech Stack:** Rust 2021, serde, serde_json, std::fs, Cargo tests.

## Global Constraints

- The file path is `~/.config/entropy/typing_trainer_symbol_stats.json`.
- The file contains a format version and a `BTreeMap<char, TypingTrainerCharacterStats>`.
- Missing or invalid JSON starts with an empty map; I/O failure only logs a warning.
- No old embedded statistics are migrated or deleted.
- No new dependency is added.

---

### Task 1: Add isolated statistics-file I/O

**Files:**
- Modify: `src/app_storage.rs:1-80, 296-334, app_storage tests`
- Modify: `src/app_state.rs:1-205`

**Interfaces:**
- Produces `load_typing_trainer_symbol_stats() -> TypingTrainerCharacterStatsMap`.
- Produces `save_typing_trainer_symbol_stats(stats: &TypingTrainerCharacterStatsMap)`.
- Removes `typing_trainer_symbol_stats` from `AppSettings` and its default.

- [x] **Step 1: Write failing tests for a versioned file round-trip and invalid JSON**

```rust
#[test]
fn symbol_stats_file_round_trips_versioned_data() {
    let path = temp_symbol_stats_path("round_trip");
    let stats = BTreeMap::from([('!', TypingTrainerCharacterStats { attempts: 3, errors: 1 })]);

    save_typing_trainer_symbol_stats_to_path(&path, &stats);

    assert_eq!(load_typing_trainer_symbol_stats_from_path(&path), stats);
    let json = std::fs::read_to_string(&path).unwrap();
    assert!(json.contains("\"version\": 1"));
    std::fs::remove_file(path).unwrap();
}

#[test]
fn invalid_symbol_stats_file_returns_an_empty_map() {
    let path = temp_symbol_stats_path("invalid");
    std::fs::write(&path, "not json").unwrap();

    assert!(load_typing_trainer_symbol_stats_from_path(&path).is_empty());

    std::fs::remove_file(path).unwrap();
}
```

- [x] **Step 2: Run the new tests and verify they fail because the helpers do not exist**

Run:

```bash
nix shell nixpkgs#rustc nixpkgs#cargo nixpkgs#gcc --command cargo test symbol_stats_file -- --nocapture
```

Expected: compilation failure naming the missing storage helpers.

- [x] **Step 3: Implement the minimal versioned storage helpers**

Add a private serde wrapper in `src/app_storage.rs`:

```rust
#[derive(serde::Serialize, serde::Deserialize)]
struct TypingTrainerSymbolStatsFile {
    version: u8,
    stats: TypingTrainerCharacterStatsMap,
}
```

Add `typing_trainer_symbol_stats_path()` beside `app_settings_path()`, returning
`text_expander_config_dir().join("typing_trainer_symbol_stats.json")`. Implement
the `*_from_path` helpers with `read_to_string`, `serde_json::from_str`,
`serde_json::to_string_pretty`, and warning logs. The public wrappers use the
real path. Remove the corresponding field and default initialization from
`AppSettings`.

- [x] **Step 4: Run the focused storage tests and verify they pass**

Run:

```bash
nix shell nixpkgs#rustc nixpkgs#cargo nixpkgs#gcc --command cargo test symbol_stats_file -- --nocapture
```

Expected: both tests pass.

- [x] **Step 5: Commit the isolated storage layer**

```bash
git add src/app_storage.rs src/app_state.rs
git commit --no-verify -m "refactor: вынести статистику тренажёра в отдельный файл"
```

### Task 2: Wire the in-memory map into application startup and trainer completion

**Files:**
- Modify: `src/app_state.rs:4146-4147` (`EntropyApp` trainer fields)
- Modify: `src/app_init.rs:5-118`
- Modify: `src/ui/typing_trainer.rs:40-50, 135-145, 1126-1139`

**Interfaces:**
- Consumes the Task 1 load/save functions.
- Adds `EntropyApp::typing_trainer_symbol_stats: TypingTrainerCharacterStatsMap`.

- [x] **Step 1: Write a failing serialization regression test**

Add this test near `AppSettings` tests in `src/app_state.rs`:

```rust
#[test]
fn app_settings_no_longer_embed_symbol_training_stats() {
    let json = serde_json::to_value(AppSettings::default()).unwrap();

    assert!(json.get("typing_trainer_symbol_stats").is_none());
}
```

- [x] **Step 2: Run the test and verify it fails while the legacy field exists**

Run:

```bash
nix shell nixpkgs#rustc nixpkgs#cargo nixpkgs#gcc --command cargo test app_settings_no_longer_embed_symbol_training_stats -- --nocapture
```

Expected: test failure because the legacy JSON field is present.

- [x] **Step 3: Implement startup load and finished-run save**

In `EntropyApp::new`, call `load_typing_trainer_symbol_stats()` before building
`Self`, then assign the result to a new app field. Replace every trainer use of
`self.app_settings.typing_trainer_symbol_stats` with that field. In
`record_finished_typing_trainer_run`, call `save_typing_trainer_symbol_stats`
after a completed run is recorded; retain `save_app_settings` for the history.
Do not write on every keystroke.

- [x] **Step 4: Run focused trainer and storage tests**

Run:

```bash
nix shell nixpkgs#rustc nixpkgs#cargo nixpkgs#gcc --command cargo test typing_trainer symbol_stats_file -- --nocapture
```

Expected: all selected tests pass.

- [x] **Step 5: Commit and push the wiring change**

```bash
git add src/app_init.rs src/app_state.rs src/ui/typing_trainer.rs
git commit --no-verify -m "feat: сохранять статистику тренажёра отдельно"
git push origin feat/adaptive-symbol-trainer
```

### Task 3: Verify the branch and the user-visible file

**Files:**
- No source changes expected.

- [x] **Step 1: Run the complete automated verification**

```bash
nix shell nixpkgs#rustc nixpkgs#cargo nixpkgs#gcc --command cargo test --all-targets
python3 scripts/check_i18n.py
nix shell nixpkgs#rustfmt --command rustfmt --check src/app_init.rs src/app_state.rs src/app_storage.rs src/ui/typing_trainer.rs
git diff --check
```

Expected: every command exits with status 0.

- [ ] **Step 2: Manually validate persistence**

Run the development build, complete a short `symbols` exercise with one
intentional error, and inspect only file names and JSON structure:

```bash
find ~/.config/entropy -maxdepth 1 -name 'typing_trainer_symbol_stats.json' -printf '%f\n'
jq 'keys' ~/.config/entropy/typing_trainer_symbol_stats.json
jq 'has("typing_trainer_symbol_stats")' ~/.config/entropy/app_settings.json
```

Expected: the new file exists and has `version` and `stats`; the old settings
file reports `false`.
